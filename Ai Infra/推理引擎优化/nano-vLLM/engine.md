## llm_engine.py
暴露generate接口供外部调用
generate 函数首先执行一个 for 循环，遍历传入的 prompts 和 sampling_params 列表,用zip打包
 对于每一个 prompt 和 sampling_params 对，它调用 self.llm_engine.add_request(prompt, sp)
 在add_request内部将这个请求封装成一个Sequence对象然后交给scheduler
 将目前的请求加入队列中之后，generate 函数进入一个 while 循环，只要引擎内部还有未完成的请求（通常通过 self.llm_engine.has_unfinished_requests() 判断），循环就会持续运行
 
 在循环内部，每一轮都会调用一次 self.llm_engine.step()
在一次step中：
调用Scheduler获取当前Batch的执行计划，将计划传递ModelRunner执行前向计算

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
1. 向调度器拉取一个batch
2. 统计这一步的token数量：
如果是 Prefill（True）：num_scheduled_tokens 是这个序列的 Prompt 长度。把整个 Batch 的长度加起来，就是这一步处理的 Token 总数（正数）。
如果是 Decode（False）：这里直接返回了 -len(seqs)（负数）。
纯粹是为了在 llm.py 中区分当前的吞吐量是 Prefill 还是 Decode（通过正负号判断）
3. 把 seqs 和 is_prefill 标志扔给 ModelRunner（底层执行器）
4. postprocess 是每一步推理流程中的状态收尾环节，判断是decode阶段需要追加新的token，还是只是prefill还没算完所有的Token，如果 prompt 还没处理完（num_cached_tokens < num_tokens），就只更新缓存进度，不追加 token，直接跳过后续逻辑
5. 看seq.is_finished，postprocess负责把该结束的序列设成 is_finished=True，这段代码只负责把那些刚刚被设成 True 的捞出来。
seq.seq_id  请求的唯一标识。用户发起请求时分配，用来把结果对回原来的请求
seq.completion_token_ids 这是序列的生成输出 token 列表，注意它只装"生成"的 token，不装 prompt 本身的 token。


## scheduler.py
Scheduler维护两个deque： waiting与running，新请求进入waiting
```python
self.waiting: deque[Sequence] = deque()
self.running: deque[Sequence] = deque()
```

先维护两个变量：
- cheduled_seqs：这一步准备送给 GPU 计算的序列列表（当前批次）。
- num_batched_tokens：当前批次累计占用的 Token 数量，用于控制不要超过 max_num_batched_tokens 的硬上限（防止单次计算撑爆显存或超时）。

#### 接下来进入prefill阶段：
尝试从waiting队列中取新请求 -> 只要 waiting 队列有请求，且当前批次序列数没达到上限，就取队首的请求 seq 尝试调度。

维护一个remaining变量， remaining 是这一步还能塞进去多少 Token。如果 remaining == 0，说明预算耗尽，直接结束 Prefill 调度。

尝试给请求分配物理显存块：
新序列若未分配物理块，先检查能否分配；显存不足（返回 -1）就 break，不再看后续请求；能分配则拿到命中的前缀缓存块数。

Chunked Prefill(分块预填充): 
nano-vllm不支持，所以如果剩余的 Token 预算（remaining）不足以把当前序列算完（remaining < num_tokens），并且当前批次里已经有其他序列了，那就直接 break
- **如果这一批还是空的（还没有任何序列被选中）**：当前序列即使超预算，也允许分块，切一块先算。
- **如果这一批已经有序列了**：当前序列放不下就 break，**不允许分块**。

状态流转：
- 如果当前序列的**总 Token 数**（已缓存 + 本步调度）等于它的**总长度**，说明这个 Prompt 的 Prefill **算完了**。
- 此时，把它从 waiting队列弹出（popleft），放到 running队列。状态标记为 RUNNING。

#### 进入decode阶段:
如果上面的 Prefill 循环什么都没调度到（scheduled_seqs 为空），说明 waiting 没请求，或者显存不够塞了。此时开始尝试 Decode。

从running获取一个元素，然后检查是否需要抢占:
抢占策略：如果 running 里还有其他序列，直接把队尾的踢掉（self.running.pop()）并通过 preempt 释放显存。如果连自己都要被踢掉，就 self.preempt(seq)，然后 break（这个序列这步不干了）。

如果有足够的显存：
设置当前序列本步调度 1 个 Token（num_scheduled_tokens = 1），调用 may_append 预留空间，加入批次。

循环结束、队列重排: 
断言调度到了一个序列,把刚才取出来调度的序列重新塞回 running 队列的开头。


#### preempt
抢占函数
```python
def preempt(self, seq: Sequence):
	seq.status = SequenceStatus.WAITING
	seq.is_prefill = True
	self.block_manager.deallocate(seq)
	self.waiting.appendleft(seq)
```
设置成waiting状态，需要重新prefill(没有设置KV Cache卸载)
然后调用deallocate回收显存
并加入waiting deque的队首

#### postprocess
后处理

将当前已经算完的KV哈希化，然后存入前缀缓存表，供未来的序列复用

把本步调度的 Token 数累加到 num_cached_tokens，清零num_scheduled_tokens

如果当前阶段是prefill且还有剩余 Token 没算完，continue 跳过，不生成新 Token

其余情况在seq中append一个新token

如果遇到 EOS（结束符）或达到最大生成长度，状态改为 FINISHED，释放显存，并从 running 队列移除。

## sequence.py
重点变量：
token_ids 整个序列的 Token 列表（Prompt + 已生成的）
作用：最终输出的数据来源。

num_prompt_tokens 与 num_tokens
num_prompt_tokens 是原始输入长度；num_tokens 是当前总长度。
作用：通过 num_tokens - num_prompt_tokens 算出已经生成了多少字。

num_cached_tokens（最关键）
已经真正算完并存进显存 KV Cache 的 Token 数量。
作用：调度器判断 Prefill 有没有做完，就靠它。

num_scheduled_tokens
本次 step() 分配给它算的 Token 数量。
作用：传给 model_runner 决定这次送给 GPU 多少数据。Prefill 可能是分块的，Decode 固定是 1。
批处理中： seq.num_scheduled_tokens = min(num_tokens, remaining)确保这条序列不会把剩下的额度撑爆。

max_num_batched_tokens：这是一个硬上限（配置项），代表 GPU 一次前向计算最多能处理多少个 Token。

block_table
一个列表，记录该序列占用了哪些物理显存块（Block）。
作用：model_runner 靠它去显存里把之前的 KV Cache 读出来。

## block_manager.py
