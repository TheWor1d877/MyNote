## 整体路线图
核心就一个循环，三步：调度 → 执行 → 后处理
generate → step → schedule → model_runner → postprocess

Scheduler 决定这次跑哪个请求

ModelRunner执行前向，把选中的序列送入GPU，prefill的时候一次算完整的prompt，decode时候每个序列只生成一个新token

Scheduler.postprocess收尾： 把新 token 拼回序列，检查是否结束（命中 EOS 或达到 max_tokens），释放完成的序列占用的 KV 块。

## 目录
engine/  推理引擎核心代码
layers/   神经网络基础组件，各种算子
models/  具体的模型结构
utils/ 和 根目录 基础设施与入口

## 完整数据流过程
#### 引擎启动(GPU阶段)
加载权重：load_model 把硬盘上的权重文件读入，直接搬运到 GPU 显存。

分配 KV Cache：在 GPU 显存 中直接开辟一个大张量 self.kv_cache。

捕获 CUDA Graph：在 GPU 显存 里预先分配好 graph_vars（input_ids, positions, slot_mapping, block_tables 等），这些都是纯粹的 GPU 张量，生命周期贯穿整个引擎运行。

#### 请求入队 (CPU)
Sequence 对象（包含 token_ids, block_table 等）全部存在于 CPU 内存中。调度器（Scheduler）在 CPU 里分配物理块，改状态

#### Prefill 调度与执行(CPU调度，GPU计算)
核心逻辑： 调度 -> 数据准备 -> 前向计算 -> 采样 -> 后处理

调度（CPU）：Scheduler 在 CPU 端挑选出 seqs，分配好 block_table。

数据准备（CPU -> GPU）：prepare_prefill 在 CPU 内存 里构建 Python 列表，然后用 torch.tensor(...,pin_memory=True).cuda(non_blocking=True) 把它们变成 GPU 张量。

前向计算（GPU）：self.model(input_ids, positions) 在 GPU 上跑完，输出 logits，logits 存在于 GPU 显存 中。

采样（GPU -> CPU）：
- Sampler 在 GPU 上根据 logits 和 temperatures 选出 Token。
- token_ids = ... .tolist()：这个 .tolist() 是极其关键的跨界动作。它把采样出的单个整型数字，从 GPU 显存拷贝回 CPU 内存，变成一个 Python 列表。
- 为什么要拷贝回 CPU？ 因为 Scheduler 的 postprocess 需要在 CPU 端把新 Token 追加到 Sequence 里，并判断是否结束、释放显存。引擎的控制流在 CPU 里。

后处理（CPU）：Scheduler 更新 num_cached_tokens，调用 seq.append_token()。所有这些操作都在 CPU 内存 里发生。

#### Decode循环(CPU调度，GPU计算)
核心逻辑： 调度 -> 数据准备 -> 前向计算 -> 采样 -> 后处理

调度（CPU）：同样在 CPU 端准备，Scheduler 算出要算哪几个序列。

数据准备（CPU -> GPU）：prepare_decode 把最新的 last_token 和地址打包成 GPU 张量。

计算（GPU 加速）：
如果走了 CUDA Graph：run_model 会把刚刚用 CPU 准备好的新数据，拷贝进预先分配好的 GPU 静态张量（graph_vars）里，然后调用 graph.replay()。整个前向计算全在 GPU 内部的图中极速完成，中间没有任何 CPU 参与。
输出 logits 依然在 GPU 中。

采样（GPU -> CPU）：采样器算出新的 token，再次用 .tolist() 把数据从 GPU 显存拷回 CPU，交还给 Scheduler。

后处理（CPU）：在 CPU 中判断是否遇到 EOS，如果结束，调用 block_manager.deallocate() 在 CPU 端释放物理块的映射关系。

## 相比于vLLM未实现的关键功能
- KV Cache 的 Swap（换出）机制
nano-vLLM 实现了基础的抢占（Preemption），但处理方式很粗暴：显存不够时，直接释放被抢占序列占用的所有物理块，等它再次被调度时从头重算。
vLLM 则有完整的 Swap 机制：显存紧张时，把被抢占序列的 KV Cache 换出到 CPU 内存，而不是丢弃。等资源充裕时再换回来，避免了昂贵的重算开销

- Chunked Prefill的完整调度
nano-vLLM只允许对批次里的第一个序列做分块，后面的序列必须完整塞入
vLLM 的 Chunked Prefill 是完整实现的：长 prompt 可以被切成多个 chunk，与 Decode 请求混在同一个批次里执行

- Quantization
vLLM 支持 AWQ、GPTQ、FP8、INT8、INT4 等多种量化格式。量化直接决定了模型能不能塞进消费级显卡，以及推理成本能压到多低，是生产部署的刚需

- Speculative Decoding
nano-vLLM 的 generate 循环是标准的自回归生成，一步一个 Token。
vLLM 支持多种投机解码方案：用小模型“草拟”多个 Token，再用大模型并行验证，在不改变输出分布的前提下提升 Decode 速度。nano-vLLM 的 model_runner 和调度器没有为这种“草稿-验证”范式预留任何接口

- Pipeline Parallelism
nano-vLLM 的 linear.py 实现了基础的张量并行（TP），即把同一个层的权重切分到多卡上。
vLLM 还支持流水线并行（PP）：把模型的不同层组分配到不同 GPU 上，形成流水线。对于单卡放不下、但层数很多的模型，PP 是 TP 之外的关键补充。nano-vLLM 没有实现 PP 的调度和跨卡激活值传递。-8

