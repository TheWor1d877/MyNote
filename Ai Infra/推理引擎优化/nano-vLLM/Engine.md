## llm_engine.py
暴露`generate`接口供外部调用
`generate` 函数首先执行一个 for 循环，遍历传入的 `prompts` 和 `sampling_params` 列表,用`zip`打包
 对于每一个 `prompt` 和 `sampling_params` 对，它调用 `self.llm_engine.add_request(prompt, sp)`
 在`add_request`内部将这个请求封装成一个`Sequence`对象然后交给`scheduler`
 将目前的请求加入队列中之后，`generate` 函数进入一个 while 循环，只要引擎内部还有未完成的请求（通常通过 `self.llm_engine.has_unfinished_requests()` 判断），循环就会持续运行
 
 在循环内部，每一轮都会调用一次 `self.llm_engine.step()`
在一次`step`中：
调用`Scheduler`获取当前`Batch`的执行计划，将计划传递`ModelRunner`执行前向计算

#### step函数讲解
```python
    def step(self):
        seqs, is_prefill = self.scheduler.schedule() # 1
        num_tokens = sum(seq.num_scheduled_tokens for seq in seqs) if is_prefill else -len(seqs) # 2
        token_ids = self.model_runner.call("run", seqs, is_prefill) # 3
        self.scheduler.postprocess(seqs, token_ids, is_prefill) # 4
        outputs = [(seq.seq_id, seq.completion_token_ids) for seq in seqs if seq.is_finished] # 5
        return outputs, num_tokens # 6
```
这是最核心的单步流水线：
1. 向调度器拉取一个`batch`
2. 统计这一步的`token`数量：
如果是 `Prefill`（`True`）：`num_scheduled_tokens` 是这个序列的 `Prompt` 长度。把整个 `Batch` 的长度加起来，就是这一步处理的 `Token` 总数（正数）。
如果是 `Decode`（`False`）：这里直接返回了 `-len(seqs)`（负数）。
纯粹是为了在 `llm.py` 中区分当前的吞吐量是 `Prefill` 还是 `Decode`（通过正负号判断）
3. 把 `seqs` 和 `is_prefill` 标志扔给 `ModelRunner`（底层执行器）
4. `postprocess` 是每一步推理流程中的状态收尾环节，判断是`decode`阶段需要追加新的`token`，还是只是`prefill`还没算完所有的`Token`，如果 `prompt` 还没处理完（`num_cached_tokens < num_tokens`），就只更新缓存进度，不追加 `token`，直接跳过后续逻辑
5. 看`seq.is_finished`，`postprocess`负责把该结束的序列设成 `is_finished=True`，这段代码只负责把那些刚刚被设成 `True` 的捞出来。
`seq.seq_id`  请求的唯一标识。用户发起请求时分配，用来把结果对回原来的请求
`seq.completion_token_ids` 这是序列的生成输出 `token` 列表，注意它只装"生成"的 `token`，不装 `prompt` 本身的 `token`。


## scheduler.py
`Scheduler`维护两个`deque`： `waiting`与`running`，新请求进入`waiting`
```python
self.waiting: deque[Sequence] = deque()
self.running: deque[Sequence] = deque()
```

先维护两个变量：
- `scheduled_seqs`：这一步准备送给 GPU 计算的序列列表（当前批次）。
- `num_batched_tokens`：当前批次累计占用的 `Token` 数量，用于控制不要超过 `max_num_batched_tokens` 的硬上限（防止单次计算撑爆显存或超时）。

#### 接下来进入prefill阶段：
尝试从`waiting`队列中取新请求 -> 只要 `waiting` 队列有请求，且当前批次序列数没达到上限，就取队首的请求 `seq` 尝试调度。

维护一个`remaining`变量， `remaining` 是这一步还能塞进去多少 `Token`。如果 `remaining == 0`，说明预算耗尽，直接结束 `Prefill` 调度。

尝试给请求分配物理显存块：
新序列若未分配物理块，先检查能否分配；显存不足（返回 -1）就 `break`，不再看后续请求；能分配则拿到命中的前缀缓存块数。

`Chunked Prefill`(分块预填充): 
`nano-vllm`不支持，所以如果剩余的 `Token` 预算（`remaining`）不足以把当前序列算完（`remaining < num_tokens`），并且当前批次里已经有其他序列了，那就直接 `break`
- **如果这一批还是空的（还没有任何序列被选中）**：当前序列即使超预算，也允许分块，切一块先算。
- **如果这一批已经有序列了**：当前序列放不下就 `break`，**不允许分块**。

状态流转：
- 如果当前序列的**总 `Token` 数**（已缓存 + 本步调度）等于它的**总长度**，说明这个 `Prompt` 的 `Prefill` **算完了**。
- 此时，把它从 `waiting`队列弹出（`popleft`），放到 `running`队列。状态标记为 `RUNNING`。

#### 进入decode阶段:
如果上面的 `Prefill` 循环什么都没调度到（`scheduled_seqs` 为空），说明 `waiting` 没请求，或者显存不够塞了。此时开始尝试 `Decode`。

从`running`获取一个元素，然后检查是否需要抢占:
抢占策略：如果 `running` 里还有其他序列，直接把队尾的踢掉（`self.running.pop()`）并通过 `preempt` 释放显存。如果连自己都要被踢掉，就 `self.preempt(seq)`，然后 `break`（这个序列这步不干了）。

如果有足够的显存：
设置当前序列本步调度 1 个 `Token`（`num_scheduled_tokens = 1`），调用 `may_append` 预留空间，加入批次。

循环结束、队列重排: 
断言调度到了一个序列,把刚才取出来调度的序列重新塞回 `running` 队列的开头。


#### preempt
抢占函数
```python
def preempt(self, seq: Sequence):
	seq.status = SequenceStatus.WAITING
	seq.is_prefill = True
	self.block_manager.deallocate(seq)
	self.waiting.appendleft(seq)
```
设置成`waiting`状态，需要重新`prefill`(没有设置KV Cache卸载)
然后调用`deallocate`回收显存
并加入`waiting deque`的队首

#### postprocess
后处理

将当前已经算完的KV哈希化，然后存入前缀缓存表，供未来的序列复用

把本步调度的 `Token` 数累加到 `num_cached_tokens`，清零`num_scheduled_tokens`

如果当前阶段是`prefill`且还有剩余 `Token` 没算完，`continue` 跳过，不生成新 `Token`

其余情况在`seq`中`append`一个新`token`

如果遇到 `EOS`（结束符）或达到最大生成长度，状态改为 `FINISHED`，释放显存，并从 `running` 队列移除。

## sequence.py
重点变量：
`token_ids` 整个序列的 `Token` 列表（`Prompt` + 已生成的）
作用：最终输出的数据来源。

`num_prompt_tokens` 与 `num_tokens`
`num_prompt_tokens` 是原始输入长度；`num_tokens` 是当前总长度。
作用：通过 `num_tokens - num_prompt_tokens` 算出已经生成了多少字。

`num_cached_tokens`（最关键）
已经真正算完并存进显存 `KV Cache` 的 `Token` 数量。
作用：调度器判断 `Prefill` 有没有做完，就靠它。

`num_scheduled_tokens`
本次 `step()` 分配给它算的 `Token` 数量。
作用：传给 `model_runner` 决定这次送给 GPU 多少数据。`Prefill` 可能是分块的，`Decode` 固定是 1。
批处理中： `seq.num_scheduled_tokens = min(num_tokens, remaining)`确保这条序列不会把剩下的额度撑爆。

`max_num_batched_tokens`：这是一个硬上限（配置项），代表 GPU 一次前向计算最多能处理多少个 `Token`。

`block_table`
一个列表，记录该序列占用了哪些物理显存块（`Block`）。
作用：`model_runner` 靠它去显存里把之前的 `KV Cache` 读出来。

## block_manager.py
#### 数据结构：块与管理器的模型
关键参数： 
`ref_count`：非常关键。如果多个序列共享同一个前缀块（比如大家都以 "你好" 开头），这个计数就是 2 或更多。只有计数归零，块才能被回收。

`hash` 和 `token_ids`：用于前缀缓存。`hash` 是块的唯一指纹，`token_ids` 用来防止哈希碰撞（碰撞时比对真实 `Token`）。

`free_block_ids`：空闲块的 ID 队列。分配时从左边拿（`popleft`）。

`used_block_ids`：正在被使用的块集合。

`hash_to_block_id`：哈希指纹到物理块 ID 的映射，用于前缀缓存查找。

#### 哈希计算
这是链式哈希。计算当前块的哈希时，会把前一个块的哈希（`prefix`）也混进去。
```python
@classmethod
def compute_hash(cls, token_ids: list[int], prefix: int = -1):
    h = xxhash.xxh64()
    if prefix != -1:
        h.update(prefix.to_bytes(8, "little"))
    h.update(np.array(token_ids).tobytes())
    return h.intdigest()
```
这意味着：如果前一个块变了，后面所有块的哈希都会变。这保证了“前缀完全相同”才能命中缓存。

`hash`的是`token_ids`

#### 哈希登记
这个哈希登记只在`postprocess`中被处理
登记刚刚被完整算完的块: `start` 到 `end` 之间的块
并且计算链式哈希
```python
def hash_blocks(self, seq: Sequence):
    start = seq.num_cached_tokens // self.block_size
    end = (seq.num_cached_tokens + seq.num_scheduled_tokens) // self.block_size
    if start == end: return
    h = self.blocks[seq.block_table[start - 1]].hash if start > 0 else -1
    for i in range(start, end):
        block = self.blocks[seq.block_table[i]]
        token_ids = seq.block(i)
        h = self.compute_hash(token_ids, h)
        block.update(h, token_ids)
        self.hash_to_block_id[h] = block.block_id
```

#### 分配计算
###### can_allocate: 查前缀看是否能分配
循环遍历序列的每一个块（注意只遍历到 `num_blocks - 1`，因为最后一个块通常还没填满，不参与缓存）。
计算链式哈希，去 `hash_to_block_id` 里查。
如果找到且 `Token` 完全一致，说明命中了前缀缓存，`num_cached_blocks` 加一。

关键判断：如果命中的块已经在 `used_block_ids` 里（别人正在用），那么本序列不需要额外分配新块（`num_new_blocks -= 1`）；如果不在，说明这个块是游离的缓存，需要把它“复活”。

最后检查：空闲块够不够？不够返回 -1，调度器会放弃这个序列。

###### allocate: 正式分配
把命中的缓存块挂到 `seq.block_table` 上，并处理引用计数
用`allocate_block()` 从空闲队列拿新块，重置他的状态(如果它之前存过缓存，要把它从哈希表里删掉)，将未命中的地方重新分配空间

#### 追加流程
`can_append`：判断当前序列的最后一个 `Token` 是否刚好填满一个块（`len(seq) % block_size == 1` 表示刚跨过边界，需要新块）。如果需要新块，就检查空闲块够不够。

`may_append`：如果跨边界了，分配一个新块追加到 `block_table`。

#### 释放流程
遍历序列占用的所有块，把引用计数减一。

只有减到 0 的块，才真正还给空闲队列。

注意：释放时没有从 `hash_to_block_id` 里删除哈希。这意味着这个块虽然空闲了，但它的哈希记录还在，如果将来有相同前缀的请求，可以“复活”它，跳过重算。

## model_runner.py
@torch.inference_mode() 是 PyTorch 提供的一个装饰器，专门用来标记“这一段代码是纯推理（Inference）阶段，不涉及训练”

#### 初始化与显存分配
分布式初始化：如果开启了张量并行（`world_size > 1`），初始化 `NCCL`。`Rank 0` 作为主节点，通过共享内存（`SharedMemory`）向其他 `Rank` 广播调度指令（`write_shm` / `read_shm`）。

模型加载
分配`KVCache`显存池

#### 预热
正式推理前用一批模拟序列跑一次前向，把该分配、该编译、该初始化的 CUDA 资源提前准备好
清空并重置显存统计，用模拟的数据跑一遍完整前向，触发CUDA上下文，cuBLAS/cuDNN 句柄、FlashAttention 内核、通信组、内存池等的一次性初始化
避免第一个真实请求时因懒加载和 JIT 编译出现首包延迟或显存突增

#### 主入口`run()` 与数据准备
```python
def run(self, seqs: list[Sequence], is_prefill: bool) -> list[int]:
    # 1. 构造输入 Tensor
    input_ids, positions = self.prepare_prefill(seqs) if is_prefill else self.prepare_decode(seqs)
    
    # 2. 构造采样温度参数 (仅 Rank 0)
    temperatures = self.prepare_sample(seqs) if self.rank == 0 else None
    
    # 3. 执行模型前向传播
    logits = self.run_model(input_ids, positions, is_prefill)
    
    # 4. 采样并返回 Token ID
    token_ids = self.sampler(logits, temperatures).tolist() if self.rank == 0 else None
    
    # 5. 清理全局 Context
    reset_context()
    return token_ids
```
###### prepare_prefill

`Prefill` 阶段是处理完整的 `Prompt`。由于不同的请求 `Prompt` 长度不同，为了在一批里计算，需要把它们拼接成一维张量，并记录边界信息。

`input_ids` 和 `positions`：把所有请求本次需要计算的 `Token` 和对应的位置拼成一维张量。

`cu_seqlens_q` 和 `cu_seqlens_k`：累计序列长度。比如批次里有两个请求，长度分别是 10 和 20，那么 `cu_seqlens_q` 就是`[0, 10, 30]`。这是 `FlashAttention` 需要的格式。

`slot_mapping`：核心所在。它告诉注意力计算层，当前这个 `Token` 的 KV 应该写到物理显存池的哪个位置。逻辑是：遍历 `seq.block_table`，找到分配的物理块，计算物理块内的偏移量，生成地址序列。

`block_tables`：如果存在前缀缓存命中（`cu_seqlens_k[-1] > cu_seqlens_q[-1]）`，则需要传入，用于读取已经缓存的历史 KV。
###### prepare_decode
`Decode` 阶段每个请求只计算 1 个 `Token`。
`input_ids`：就是每个序列的最后一个 `Token`（`seq.last_token`）。
`positions`：序列长度减一。
`context_lens`：当前序列的总长度（用于计算 `Attention`）。
`slot_mapping`：仅计算最新 `Token` 对应的物理地址：`seq.block_table[-1] * block_size + seq.last_block_num_tokens - 1`。
`block_tables`：因为 `Decode` 需要读取所有历史的 `KV Cache`，所以必须传入完整的物理块映射表。

所有张量都使用了 `pin_memory=True` 和 `.cuda(non_blocking=True)`，这是为了利用 PCIe 的异步传输，减少 CPU 到 GPU 的数据拷贝延迟。

#### 前向计算与CUDA Graph
`run_model` 负责执行模型。它有一个关键的分支：是否使用 `CUDA Graph`。

###### 捕获过程
在初始化阶段，如果未开启 `enforce_eager`，就会执行 `capture_cudagraph`：
生成一系列固定的批次大小（`graph_bs`），例如` [1, 2, 4, 8, 16, 32... 512]`。
为每个批次大小创建一个 `torch.cuda.CUDAGraph`。

预先分配好固定大小的输入张量（`input_ids`, `slot_mapping` 等），存入 `self.graph_vars`。
执行一次前向传播作为 `Warmup`。
用 `with torch.cuda.graph(graph, pool):` 捕获该批次的计算图。

###### 回放过程
在 `Decode` 且批次大小未超过 512 时：
根据当前的 `bs`，在 `graph_bs` 中找到最接近且大于等于 `bs` 的图（例如当前 `bs=5`，则选择 `bs=8` 的图）。
将真实数据拷贝到 `graph_vars` 对应的张量切片中。
调用 `graph.replay()` 执行。
从 `graph_vars["outputs"]` 中取出结果。
