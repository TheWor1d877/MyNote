全文围绕planner展开
## planner:核心
planner 的核心任务，是把一个逻辑上的通信请求（如“对 1GB 数据做 AllReduce”），翻译成一份物理上可执行的、最优的调度计划

`src/enqueue.cc` 在 launch 之前会做以下事情：
- 检查 stream 和 communicator 状态，
- 进入 group 语义，
- 记录是否在 CUDA graph capture 中，
- 选择算法与协议，
- 决定 channel 数和线程数，
- 计算 chunk / slice / loop 结构，
- 为需要 CPU 推进的 transport 生成 proxy 操作，
- 产出最终 launch plan。

## pattern

在 `src/enqueue.cc` 中，NCCL 会把 `(collective, algorithm)` 映射到具体执行 pattern。

代价表：存储的是 (AllReduce, Ring, Simple) 这种组合的预估耗时。planner 通过查表，选择 (algorithm, protocol)。

planner 的翻译步骤：当 planner 选定 Ring 后，它需要知道“Ring 在设备上到底长什么样”。这时，它会去查一个内部的映射表，将 (collective, algorithm) 翻译成 pattern。
- (AllReduce, Ring) → ncclPatternRingTwice（先 Reduce-Scatter，再 All-Gather）
- (AllReduce, Tree) → ncclPatternTreeUpDown（先向上归约，再向下广播）

这个 pattern 是生成最终 launch plan（包含 chunk 怎么切、循环怎么做）的直接依据。所以，代价表帮你“选算法”，pattern 帮你“执行算法”，两者是上下游的关系。

典型映射包括：

| Collective | Algorithm | Pattern |
| --- | --- | --- |
| all-reduce | ring | `ncclPatternRingTwice` |
| all-reduce | tree | `ncclPatternTreeUpDown` |
| all-reduce | CollNet direct | `ncclPatternCollnetDirect` |
| all-reduce | CollNet chain | `ncclPatternCollnetChain` |
| all-reduce | NVLS | `ncclPatternNvls` |
| all-reduce | NVLS tree | `ncclPatternNvlsTree` |
| all-gather | ring | `ncclPatternRing` |
| all-gather | PAT | `ncclPatternPatDown` |
| reduce-scatter | ring | `ncclPatternRing` |
| reduce-scatter | PAT | `ncclPatternPatUp` |
| broadcast | tree | `ncclPatternTreeDown` |
| reduce | tree | `ncclPatternTreeUp` |

## step、slice、chunk、loop、group、stream区分
把一大块数据，切成能塞进硬件 FIFO 的小块，再决定并行跑几趟
```text
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
维度 A：调用组织维度（group / stream）
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ncclGroupStart()
  │
  ├── 调用 1: ncclAllReduce(..., stream S1)
  │        └──→ [维度 B：5 层数据切分] → launch plan → 提交到 S1
  │
  ├── 调用 2: ncclBroadcast(..., stream S2)
  │        └──→ [维度 B：5 层数据切分] → launch plan → 提交到 S2
  │
  └── 调用 3: ncclAllReduce(..., stream S1)
           └──→ [维度 B：5 层数据切分] → launch plan → 提交到 S1
ncclGroupEnd()


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
维度 B：单次调用内部的数据切分
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                          第 0 层：总 payload (nBytes)
                          比如 256 MiB 待搬运数据
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                   │
                                   │  按 loopSize 切
                                   ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                    第 1 层：Loop（轮次）
                    ┌──────┬──────┬──────┬──────┬─── ... ───┬──────┐
                    │Loop 0│Loop 1│Loop 2│Loop 3│           │Loop 7│
                    └──┬───┴──────┴──────┴──────┴───────────┴──────┘
                       │  nLoops = 8（共 8 轮）
                       │
                       │  每一轮 loop 内部：所有 channel 并行
                       ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
第 2 层：Channel（并行通道，一层内所有 channel 同时跑）← 数据分块主要发生在这里
              ┌────────────┬────────────┬────────────┬────────────┐
              │ Channel 0  │ Channel 1  │ Channel 2  │ Channel 3  │
              └─────┬──────┴─────┬──────┴─────┬──────┴─────┬──────┘
                    │            │            │            │
                    │  每个 channel 在一轮里搬 nchunksPerLoop 个 chunk
                    ▼            ▼            ▼            ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
          第 3 层：Chunk（每个 channel 一次搬运的大块，如 1 MiB）
          ┌───────┬───────┬───────┬───────┬─── ... ───┬───────┐
          │chunk 0│chunk 1│chunk 2│chunk 3│           │chunk 7│
          └───┬───┴───┬───┴───┬───┴───┬───┴───────────┴───┬───┘
              │       │       │       │                   │
              │  每个 chunk 内部再切 slice
              │  （下面只展开 chunk 0 看内部）
              ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
          第 4 层：Slice（chunk 内部的数据分段，如按 rank/跳数分）
          ┌─────────┬─────────┬─────────┬─────────┬─── ... ───┐
          │ slice 0 │ slice 1 │ slice 2 │ slice 3 │           │
          └────┬────┴────┬────┴────┬────┴────┬────┴───────────┘
               │         │         │         │
               │  每个 slice 以 step 大小逐个塞进 FIFO 槽位
               ▼         ▼         ▼         ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
          第 5 层：Step（FIFO 槽位，硬件流水线最小节拍）
          ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
   FIFO → │ slot │ slot │ slot │ slot │ slot │ slot │ slot │ slot │ → 循环使用
   (环形) │  0   │  1   │  2   │  3   │  4   │  5   │  6   │  7   │
          └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
                   ↑ 一个 step = 一个槽位的大小 = stepSize
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 举例
16 个元素，4 个 rank，2 个 channel，每轮 loop 搬 8 个元素
```text
总数据（每个 rank 各有一份）:
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16]

参数:
  nBytes         = 16 个元素
  nChannels      = 2
  nchunksPerLoop = 2
  chunkSize      = 2 个元素
  sliceSteps     = 2   →  sliceSize = chunkSize / sliceSteps = 1 个元素
  stepSize       = 1 个元素（FIFO 一个槽位装 1 个元素）
```

- loop
```text
       GPU0                GPU1                GPU2                GPU3
   ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
   │Loop0[1..8]       │Loop0[1..8]       │Loop0[1..8]       │Loop0[1..8]
   │Loop1[9..16]      │Loop1[9..16]      │Loop1[9..16]      │Loop1[9..16]
   └──────────┘       └──────────┘       └──────────┘       └──────────┘
   Loop 0 先在所有 rank 间跑完一遍环（Reduce-Scatter + All-Gather），
   Loop 1 再独立跑一遍。两个 Loop 不共享数据。
```

- channel
每个 rank 的 Loop 0 数据分成两条独立环：
```text
Channel 0 环（处理 [1..4]）:
       GPU0 ──→ GPU1 ──→ GPU2 ──→ GPU3 ──→ GPU0
       [1..4]  [1..4]  [1..4]  [1..4]

Channel 1 环（处理 [5..8]）:
       GPU0 ──→ GPU1 ──→ GPU2 ──→ GPU3 ──→ GPU0
       [5..8]  [5..8]  [5..8]  [5..8]
```

- chunk
```text
Step 1:  GPU0 ──chunk0[1,2]──→ GPU1
         GPU1 ──chunk1[3,4]──→ GPU2
         GPU2 ──chunk0[1,2]──→ GPU3
         GPU3 ──chunk1[3,4]──→ GPU0

Step 2:  GPU0 ──(上一步收到的)──→ GPU1
         GPU1 ──(上一步收到的)──→ GPU2
         GPU2 ──(上一步收到的)──→ GPU3
         GPU3 ──(上一步收到的)──→ GPU0

Step 3:  GPU0 ──(上一步收到的)──→ GPU1
         GPU1 ──(上一步收到的)──→ GPU2
         GPU2 ──(上一步收到的)──→ GPU3
         GPU3 ──(上一步收到的)──→ GPU0
```
3 步后，每个 rank 完整拥有某个 chunk 的全局和。
之后走All-Gather也是同理

- slice
chunk 在 FIFO 里拆成 slice 逐个流动
一个 chunk发给下一个 rank 时，不是整体塞进 FIFO，而是拆成 slice 逐个塞：
```text
GPU0 的 FIFO:                         GPU1 的 FIFO:
┌────────┬────────┐                  ┌────────┬────────┐
│ slot 0 │ slot 1 │  ──slice0[1]──→  │ slot 0 │ slot 1 │
│  [1]   │  [2]   │  ──slice1[2]──→  │  [1]   │  [2]   │
└────────┴────────┘                  └────────┴────────┘

GPU0 先把 slice0=[1] 塞进 FIFO 槽位 0，GPU1 从槽位 0 取走；
再把 slice1=[2] 塞进槽位 1，GPU1 从槽位 1 取走。
两个 slice 流水线式流动，GPU1 边收边拼成 chunk0=[1,2]。
```

- step
传递单位：硬件上是 step——FIFO 一个槽位装一个 step 大小的数据，GPU 按槽位逐个消费。

## 算法与协议选择
`src/enqueue.cc` 中一个核心 helper 是 `topoGetAlgoInfo(...)`。它会读取已
经由 topology + tuning 子系统准备好的代价表，然后选出当前 collective 最
便宜的 `(algorithm, protocol)` 组合。


## CUDA Graph Capture
NCCL 通信本身已经成为 GPU 计算图的一部分，而不是游离在图外的独立操作。一旦通信被“录制”进图里，NCCL 就必须以对待一个 kernel 的严肃态度，来处理“捕获”这个语义。

helper `ncclPlannerSetCapturingGraph(...)` 会跟踪流是否正处在 CUDA graph
capture 中，并确保一组 NCCL group 中涉及的 streams 要么全都没 capture，
要么必须属于同一个 capture graph。

是否被 CUDA Graph 捕获”被视为该操作的一个 collective 属性，必须在所有参与该次通信的 rank 上保持一致

在普通 eager 模式下，group 内部可以灵活地混合“立即执行”的操作；但一旦进入 capture 模式，group 内部的所有 stream 都必须遵循同一套 capture 规则。