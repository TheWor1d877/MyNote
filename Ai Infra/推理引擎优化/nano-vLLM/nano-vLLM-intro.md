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
