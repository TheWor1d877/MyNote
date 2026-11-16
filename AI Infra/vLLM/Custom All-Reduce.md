Custom All-Reduce 是推理框架（如 vLLM、SGLang）自己实现的一个专用 CUDA Kernel，专门在 单机、NVLink 全互联、小消息（Decode阶段） 的场景下，绕过 NCCL，利用 GPU 的 P2P（Peer-to-Peer）直接访存能力，以极低延迟完成 All-Reduce 操作

它不是 NCCL 内部的算法，而是与 NCCL 平行的“快速路径”

## 需求
LLM 推理的 Decode 阶段，每生成一个 token，每层都要做 All-Reduce，消息极小（典型 16–256 KB）。
NCCL 的 Ring/Tree 算法为通用性和大消息设计，协议开销（同步、多轮通信）在小消息下完全压倒了实际数据传输时间，成为延迟瓶颈。

## 使用时机
必须**同时满足**以下条件，否则自动回退到 NCCL：
- **同节点**：所有 GPU 必须在同一台物理机内。
- **GPU 数量**：仅支持 **2, 4, 6, 8** 卡。
- **拓扑**：要求 GPU 间为**全 NVLink/P2P 互联**（部分 PCIe 场景也有优化实现，但有更严格限制）。
- **消息大小**：存在阈值（如 vLLM 默认 **8 MiB**），超过则回退。
- **非 NCCL 进程组**：代码强制断言通信组后端不能是 NCCL。
- **批次不变性**：vLLM 开启 `VLLM_BATCH_INVARIANT` 时会被强制禁用。

## 实现
```text
┌─────────────────────────────────────────────┐
│ L4  调度层（cuda_communicator.py）            │
│     每次调用判断：disabled？输入满足条件？       │
│     满足 → 走 Custom；不满足 → 回退 NCCL       │
│     ★ 唯一决定"用不用 Custom"的地方            │
├─────────────────────────────────────────────┤
│ L3  通信引擎层（CustomAllreduce 类）           │
│     初始化：检查能力，不行就 disabled=True      │
│     提供 should_custom_ar() 供 L4 查询        │
│     运行时：接收张量，调 CUDA 内核              │
│     ★ 不决定"用不用自己"，只回答"能不能用"       │
├─────────────────────────────────────────────┤
│ L2  CUDA Kernel 层                           │
│ One-shot：单内核读所有对端→归约→写回(全互联+小消息)│
│ Two-shot：Reduce-Scatter+All-Gather(非全互联或较大消息)      
│     Signal 原子计数器轮询同步                  │
├─────────────────────────────────────────────┤
│ L1  硬件层                                   │
│     CUDA IPC + P2P 直接访存（NVLink/PCIe）    │
│     无 NCCL Transport / Proxy               │
└─────────────────────────────────────────────┘
```
- L4 
决定“这次用不用 Custom”，每次调用都重新判断。

- L3
只回答“我能不能被用”（初始化时设 disabled）和“这次输入符不符合”（should_custom_ar 返回 True/False）。

- L2：
是唯一真正执行 All-Reduce 数据搬运与归约的地方
数据搬运：通过 rank_data 表中的 IPC 指针，直接读取所有对端 GPU 显存中的输入数据
规约计算：读到的多份数据（自己的 + 所有对端的），在本地寄存器或共享内存中累加
同步： 数据写回后，通过 Signal 原子计数器通知对端
接收方轮询等待所有对端的信号到齐。同步完全在 GPU 内完成，无 CPU、无 proxy 线程参与。

two-shot的实现：
1. **切分**：把完整的输入张量按 GPU 数量（`world_size`）**等分成 N 个 chunk**。
2. **分配**：GPU 0 负责归约 chunk 0，GPU 1 负责归约 chunk 1，以此类推
3. **直接读取**：为了算出自己负责的那个 chunk 的完整和，GPU 0 会**直接通过 NVLink 从 GPU 1、2、3 的显存中，把对应的 chunk 数据一次性读过来**，然后在本地累加。

- L1：
通信方式:不走 NCCL 的 transport/proxy。
控制面：用 Gloo 交换 CUDA IPC 显存句柄。
数据面：一个 CUDA Kernel 通过 IPC 指针直接读写对端显存，本地归约。
同步：GPU 内 Signal 原子计数器轮询，无 CPU 介入。

