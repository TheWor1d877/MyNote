## 整体架构:按照时间线顺序
```text
用户应用
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ① 公共 API                                                   │
│   作用：接住用户请求，打包成内部作业后转给 planner。                │
│   对象：communicator（已存在），wrapper 填 ncclInfo              │
│   SUBSYS：COLL                                               │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ② bootstrap                                                 │
│   作用：在高速数据通道建立前，让各 rank 通过 CPU 侧 TCP 互相        │
│         认识并交换地址等元数据；完成后即拆除。                     │
│   对象：临时组件（communicator 初始化内部）                       │
│   SUBSYS：INIT                                                │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ③ 拓扑引擎                                                   │
│   作用：探测硬件建出拓扑图，在图上搜索 ring/tree 等结构，           │
│         并落成每个 rank 的 channel 布局。                      │
│   对象：topology graph、channel                               │
│   SUBSYS：GRAPH（channel 布局可看 INIT）                       │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ④ 调优模型                                                    │
│   作用：为每种 algorithm/protocol 组合估算延迟和带宽，             │
│         生成一张可供 planner 查用的代价表。                      │
│   对象：algorithm/protocol 评分表（存进 communicator）          │
│   SUBSYS：TUNING                                              │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ⑤ planner                                                   │
│   作用：每次调用时查代价表选出最便宜的 algo/protocol，             │
│         切好 chunk 并生成最终 launch plan。                    │
│   对象：消费 ③ 的 topology graph + ④ 的评分表                  │
│   SUBSYS：COLL/INIT（混合，无专用）                             │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ⑥ proxy + transport                                           │
│   作用：为每条连接选定并建立具体 transport，由 proxy 在需要时  │
│         协助 CPU 侧推进数据搬运。                              │
│   对象：transport（P2P/SHM/NET）、proxy                       │
│   SUBSYS：INIT + P2P/SHM/NET + PROXY                          │
└──────────────────────────────────────────────────────────────┘
   ↓
┌──────────────────────────────────────────────────────────────┐
│ ⑦ device kernel                                               │
│   作用：在 GPU 上执行最终通信 kernel，完成实际数据交换。       │
│   对象：device kernel                                          │
│   SUBSYS：无（CPU 日志覆盖不到，需 profiler）                  │
└──────────────────────────────────────────────────────────────┘

性质：①–④ 初始化期（一次性） → ⑤ 每次调用决策 → ⑥–⑦ 执行期
```

## 对象关系图
```
communicator（容器，持有全部状态）
 ├── topology graph（硬件抽象图，搜索路径的依据）
 ├── algorithm（高层形状）× channel（并行通道）
 ├── protocol（底层搬运风格）
 ├── transport（peer 间具体接通方式）
 └── proxy（需要 CPU 协助的宿主侧逻辑）
```

#### 1. communicator
**定义**：一个通信上下文对象（代码里 `ncclComm_t`），把一组 rank 绑定在一起，持有这组 rank 通信所需的全部运行时状态：拓扑、channel、连接、缓冲区、CUDA 流等。

**日志对应**：
```
ncclCommInitAll comm 0x5f895139c8a0 rank 0 nranks 1 ... Init START
ncclCommInitAll comm 0x5f895139c8a0 rank 0 nranks 1 ... Init COMPLETE
```
`0x5f895139c8a0` 就是这个 communicator 的地址。

**要点**：
- 调用 `ncclAllReduce` 时必须指定一个 communicator。
- 一个进程可以有多个 communicator（不同 rank 组），你这里只有 1 个，`nranks=1`。
- 上文所有其它术语（channel、algorithm、transport……）都是 communicator 的内部组成部分或属性。

#### 2. channel
**定义**：communicator 内部的一条并行数据通道。集合通信的数据被切块，分散到多个 channel 上并行搬运；每个 channel 有自己的 ring/tree 结构。

**日志对应**：
```
64 coll channels, 64 collnet channels, 0 nvls channels, 64 p2p channels
Ring 00 : 0 -> 0 -> 0
...
Ring 63 : 0 -> 0 -> 0
Channel 00/64 : 0
```

**要点**：
- channel 数 ≠ rank 数。单 rank 也有 64 个 channel，是并行的空壳（每个都是自环）。
- 可用 `NCCL_MAX_NCHANNELS` / `NCCL_MIN_NCHANNELS` 控制数量。
- channel 是“并行度”维度，rank 是“参与方”维度，两者正交。

#### 3. algorithm
**定义**：集合通信的高层数据流动形状，决定数据在 rank 之间怎么走。

**常见值**：

| 算法 | 形状 | 依赖 |
|------|------|------|
| `Ring` | 环，数据沿环逐跳传递 | 通用 |
| `Tree` | 树，根分发/汇聚 | 通用 |
| `CollNetDirect` / `CollNetChain` | 网络内归约 | 需 SHARP 等硬件 |
| `NVLS` / `NVLSTree` | NVLink Switch 内归约 | 需 NVLink Switch |
| `PAT` | Parallel Aggregated Tree，多树并行 | 大模型场景 |

**日志对应**：TUNING 表的列标题 `Algorithm | Tree | Ring | CollNetDirect ...`；运行时行 `Algo RING`。

**要点**：算法选择依赖消息大小、rank 数、拓扑。单卡下 Ring 退化为 `0 -> 0 -> 0`，Tree 退化为单根。

#### 4. protocol
**定义**：底层数据搬运的风格，决定同步频率、单次搬运粒度、是否 128 字节对齐等。

| 协议       | 特点                 | 适用消息    |
| -------- | ------------------ | ------- |
| `LL`     | 低延迟，每步同步           | 小（KB 级） |
| `LL128`  | LL 的 128B 对齐版，吞吐更高 | 中小      |
| `Simple` | 流式，少同步             | 大（MB 级） |

**日志对应**：TUNING 表的行标题 `Protocol | LL | LL128 | Simple`。
**要点**：
- algorithm 和 protocol 是**正交**的两个维度，组合起来才有意义：如 `Ring + Simple`、`Tree + LL128`。
- NCCL 按消息大小在协议间切换，小消息偏 LL，大消息偏 Simple。

#### 5. transport
**定义**：两个 peer（rank 之间）具体用什么机制接通并搬运数据。

| transport | 机制 | 场景 |
|-----------|------|------|
| `P2P` | GPU 直连（NVLink 或 PCIe P2P） | 同节点 GPU 间 |
| `SHM` | 共享内存，经 CPU 内存中转 | 同节点 GPU 间（无 P2P 时） |
| `NET` | 走网络（IB 或 Socket） | 跨节点 |

**日志对应**（多卡/多机才出现）：
```
Channel 00/32 : 0[1000] -> 1[1000] via P2P/CUMEM
Channel 00/32 : 0[1000] -> 1[1000] via SHM/direct
Channel 00/32 : 0[1000] -> 1[1000] via NET/Socket
```

**要点**：
- transport 是 per-connection 的，同一 communicator 的不同 channel 可能用不同 transport。
- 单卡下没有 peer，所以**没有任何 transport 日志**。


#### 6. proxy
**定义**：NCCL 中需要 CPU（host）协助推进的宿主侧逻辑/线程。GPU kernel 不能直接发起某些网络操作（如读写网卡），需要 proxy 线程代为推进。

**日志对应**：
```
[Proxy Service] Device 0 CPU core 14
[Proxy Service UDS] Device 0 CPU core 7
proxy listening socket at 10.148.118.181<42359>
threadThresholds 8/8/64 | 8/8/64 | 512 | 512
```

**要点**：
- proxy 主要在**网络传输**时活跃；P2P/SHM 大部分不需要 proxy。
- `threadThresholds` 决定何时启用多线程 proxy（消息越大越可能启用）。
- 单卡无传输，proxy 线程建立但不实际工作。

#### 7. topology graph
**定义**：NCCL 内部对系统硬件（GPU、NIC、PCIe switch、NVLink、CPU、NUMA）做的抽象图。节点是设备，边是链路，边上带带宽和链路类型（`LOC`/`PIX`/`PHB`/`SYS`/`NVL`）。

**日志对应**：
```
=== System : maxBw 5000.0 totalBw 5000.0 ===
GPU/0-1000 :GPU/0-1000 (0/5000.0/LOC) CPU/... (2/12.0/PHB) NET/0-0 (4/0.4/PHB) ...
ncclTopoPopulateNics : Filled wlp4s0 in topo with ...
```

**要点**：
- NCCL 的工作流是：**先建拓扑图 → 在图上搜索 ring/tree → 据此分配 channel 和 transport → 选 algorithm/protocol**。
- 单卡的图很简单：1 GPU + CPU + 7 NIC，链路类型 `LOC`（GPU 到自身）和 `PHB`（GPU 到 CPU/NIC）。
- 多卡多机时这张图会复杂很多，直接决定 NCCL 选哪条路径最快。

## SUBSYS
子系统	    作用
INIT	        通信器初始化、rank 发现、bootstrap、连接建立
GRAPH	拓扑探测、ring/tree 搜索、通道分配
TUNING	算法/协议选择、带宽延迟估计
NET	        网络插件、网卡探测、传输路径
ALLOC	内存分配
PROXY	代理线程/服务
[[Attachments/nccl.log]]

#### INIT
显示版本与环境（CUDA版本，NCCL版本，环境变量，外部网络插件）
然后对网络后端探测（IB，OOB）
RMA/GIN插件
初始化通信器（rank，nranks，cudaDev，nvmlDev，busId，commId）

#### GRAPH
###### 硬件探测与网络检测
7 个网卡：wlp4s0（无线）+ br-（Docker 网桥）+ 5 个 veth*（容器网卡）
无 NVLink、无 CollNet（SHARP）、无 NVLink Switch

###### 链路类型标记
标记	           含义	                                    快慢
LOC	           同一 PCI 设备内部	             最快
PIX	           同一 PCIe switch 下             快
PHB	           同一 PCIe host bridge 下     中
SYS	           跨 NUMA / 跨 socket     	     慢
NVL	           NVLink	                                 专用高速
GPU 到 CPU、GPU 到网卡都是PHB

###### 通信骨架
通信骨架

Channel是Communicator内部的一个组成部分，或者说，一个Communicator是由多个并行的Channel构成的。

```text
Ring 00 : 0 -> 0 -> 0        ← 单卡环 = 自己指向自己
Trees [0] -1/-1/-1->0->-1    ← 单卡树 = 只有根 0，无子节点
一共 64 个 ring / 64 个 tree，对应 NCCL 内部 64 个通信上下文。多卡时才会看到 0 -> 1 -> 2 -> ... -> 0 这种真实环。
```

一条逻辑上的 Ring 一次只在一对相邻 GPU 之间搬运数据，如果只用一条 Ring，可能吃不满 GPU 之间（比如多条 NVLink 或 PCIe 链路）的聚合带宽。所以 NCCL 会把一个集合通信操作拆成多个 Channel，每个 Channel 都是一条独立的 Ring/Tree，并行运行。
#### TUNING
核心是一张预计算表

| Algorithm | Tree LL | Tree LL128 | Tree Simple | Ring LL | Ring LL128 | Ring Simple | CollNet/NVLS (LL/LL128/Simple) |
|---|---:|---:|---:|---:|---:|---:|---:|
| AllReduce | 6.8/37.3 | 14/253.9 | 8.4/326.4 | 6.6/inf | 14/inf | 8.4/inf | 0.0/0.0 |

单元格 a/b：a = 延迟估计，b = 带宽估计。
-1.0/-1.0 = 该组合不可用；0.0/0.0 = 对当前拓扑无意义（需专用硬件）；inf = 单卡退化，非真实无穷带宽。
CollNet / NVLS / NVLSTree 全 0.0/0.0 → 没有 SHARP、NVLink Switch。

#### NET
网络路径
```text
NET/IB : No device found.                          → 无 IB
NET/Socket : Using [0..6] ...                       → Socket 枚举 7 个网卡
Using network Socket                                → 最终后端 = Socket
GPU Direct RDMA Disabled for HCA 0 'wlp4s0' ...     → 所有网卡不支持 GDR
```

多卡/多机才会看到真实路径
```text
Channel 00/32 : 0[1000] -> 1[1000] via P2P/CUMEM
Channel 00/32 : 0[1000] -> 1[1000] via SHM/direct
Channel 00/32 : 0[1000] -> 1[1000] via NET/Socket
```
