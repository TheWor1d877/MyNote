## 场景设定
```text
┌─────────── Node 0 ───────────┐   ┌─────────── Node 1 ───────────┐
│  GPU0 ══ GPU1 ══ GPU2 ══ GPU3 │   │  GPU4 ══ GPU5 ══ GPU6 ══ GPU7 │
│    ║      ║      ║      ║     │   │    ║      ║      ║      ║     │
│   NIC0   NIC1   NIC2   NIC3   │   │   NIC4   NIC5   NIC6   NIC7   │
└────┬──────┬──────┬──────┬─────┘   └────┬──────┬──────┬──────┬─────┘
     │      │      │      │              │      │      │      │
     └──────┴──────┴──────┴────── IB ────┴──────┴──────┴──────┘
                    （全互联 InfiniBand 网络）

┌─────────── Node 2 ───────────┐   ┌─────────── Node 3 ───────────┐
│  GPU8 ══ GPU9 ══ GPU10══ GPU11│   │ GPU12 ══ GPU13══ GPU14══ GPU15│
│    ║      ║      ║      ║     │   │    ║      ║      ║      ║     │
│   NIC8   NIC9  NIC10  NIC11   │   │  NIC12  NIC13  NIC14  NIC15   │
└────┬──────┬──────┬──────┬─────┘   └────┬──────┬──────┬──────┬─────┘
     └──────┴──────┴──────┴──────────────┴──────┴──────┴──────┘
```
节点内：4 GPU 通过 NVLink 全互联（NVL 链路，带宽极高）
节点间：每个 GPU 有自己的 NIC，走 IB 网络（NET 链路）
16 个 rank，16 个 NIC——这意味着有 16 条独立的网络路径可用

## 先建立硬件图
先把整台机器画成一张图。节点是硬件实体（GPU、CPU、NIC、PCIe switch、NVSwitch），边是它们之间的物理链路，边上标着带宽和链路类型（NVL/PIX/PHB/SYS/NET…）。

```text
节点 = { GPU0..GPU15, NIC0..NIC15, CPU0..CPU3, PCIe switch... }
边 = {
  GPU0—GPU1: NVL, 带宽 300 GB/s
  GPU0—NIC0: PIX, 带宽 50 GB/s
  NIC0—IB:   NET, 带宽 25 GB/s
  ...
}
```
这一步只是客观描述硬件，不涉及 ring。

## ncclTopoCompute搜索图中的结构
ncclTopoCompute（src/graph/search.cc）不是只搜一种结构。它会为每一种启用的算法分别调用搜索，生成该算法专属的通信骨架
#### 搜索ring
搜索是**递归尝试 + 多候选**的。以构建一条 ring 为例：

```text
尝试从 GPU0 出发：
  GPU0 → GPU1 → GPU2 → GPU3 → (跨节点) → GPU4 → GPU5 → ... → GPU15 → 回到 GPU0
         ↑ 节点内走 NVLink        ↑ 走 NIC0 → IB → NIC4
```

搜到一条完整环后，**不会停**。因为 channel 数还没满（NCCL 默认要建多个 channel，每个 channel 一条 ring），搜索会**换起始点/换 NIC**，再搜一条：

```text
第二条 ring：
  GPU0 → GPU3 → GPU2 → GPU1 → (跨节点) → GPU5 → GPU4 → ... → GPU15 → 回到 GPU0
         ↑ 节点内换条路径        ↑ 换 NIC1 → IB → NIC5
```

第三条、第四条……如此反复，直到**所有可用 NIC 对都被用上**。最终搜出**多条物理路径不同的 ring 实例**。

#### 搜索Tree
```text
━━━━━━━━━━━━━━━━ 第 1 层：节点内归约（NVLink，快）━━━━━━━━━━━━━━━━

   Node 0                         Node 1
  ┌──────────────────┐          ┌──────────────────┐
  │      GPU0        │          │      GPU4        │
  │    (head/根)     │          │     (head)       │
  │    ↑    ↑    ↑   │          │    ↑    ↑    ↑   │
  │  GPU1 GPU2 GPU3  │          │  GPU5 GPU6 GPU7  │
  └────────┬─────────┘          └────────┬─────────┘
           │ 4 份 → 归约成 1 份           │ 4 份 → 归约成 1 份
           ▼                             ▼
        ┌──────┐                      ┌──────┐
        │ NIC0 │                      │ NIC4 │
        └───┬──┘                      └───┬──┘

━━━━━━━━━━━━━━━━ 第 2 层：节点间树形互联（IB，慢）━━━━━━━━━━━━━━━━

                    GPU0 (根)
                   /
                GPU4
           （只传 1 份聚合数据，不是 4 份原始数据）
```

####  CollNet / NVLS 搜索
两种算法依赖特殊硬件。

CollNet：需要网卡支持 SHARP 等网络内归约能力。只有在 collnetEnable 开启且硬件支持时，搜索才会尝试建立 CollNet 图。
NVLS：需要 NVLink Switch 硬件。搜索器会为 NVLS 寻找合适的“head rank”等结构。
如果硬件不支持，这些搜索会优雅地失败，对应的图不会被创建，自然也不会进入后续的调优阶段。

## 落地成channel
搜索出来的多个ring/tree后，ncclTopoPreset/Postset（connect.cc）把搜索出的图实例落到 communicator 的 channel 布局上
一条完整的 ring 本身就是一条 channel 的骨架。所谓“多个 channel 并行”，就是多条独立的 ring 同时跑，每条 ring 搬自己那份数据

```text
search.cc 搜出多条完整 ring 候选
R0: 0→1→2→...→15→0
R1: 0→3→2→...→15→0
...
R(N-1): ...
        │
        ▼  connect.cc：Preset 标局部前驱/后继，Postset 汇总全局并绑定
┌───────────────────────────────────────────────────────────────────────┐
│ comm->channels[0]   = { ring:R0,    prev:15, next:1,  transport:... } │
│ comm->channels[1]   = { ring:R1,    prev:15, next:3,  transport:... } │
│ ...                                                                   │
│ comm->channels[N-1] = { ring:R(N-1), ... }                            │
└───────────────────────────────────────────────────────────────────────┘
        │
        ▼  一次 AllReduce：数据切成 N 份， channel 拿一份
┌──────────────┬──────────────┬─────┬──────────────┐
│ Channel 0    │ Channel 1    │ ... │ Channel N-1  │
│ ring R0      │ ring R1      │     │ ring R(N-1)  │
│ chunk 0      │ chunk 1      │     │ chunk N-1    │
└──────┬───────┴──────┬───────┴─────┴──────┬───────┘
       ▼              ▼                      ▼
    并行推进       并行推进               并行推进
```


“两个节点之间多条 ring”的本质：Node0 和 Node1 之间有 4 对 NIC 路径（NIC0↔NIC4、NIC1↔NIC5、NIC2↔NIC6、NIC3↔NIC7），每条路径上各跑一条 ring，对应不同的 channel。
channel 数（并行度）和算法选择是正交的——无论选 Ring 还是 Tree，channel 都会并行工作

## 调优模型
[[AI Infra/NCCL/NCCL数学与性能计算|NCCL数学与性能计算]]

代价表的实际结构：
```text
索引：[collective] × [algorithm] × [protocol] × [消息大小桶]

以 AllReduce 在某个大消息桶（如 1MB–4MB）为例：

           LL          LL128        Simple
Ring       t_R_LL      t_R_LL128    t_R_Simple    ← 16 条 ring 并行后的综合耗时
Tree       t_T_LL      t_T_LL128    t_T_Simple    ← 多 channel 并行后的综合耗时
CollNet    ...         ...          ...           ← 硬件支持时才有值
NVLS       ...         ...          ...           ← 硬件支持时才有值
```

## planner
按消息大小查表，选最便宜的组合

每次 ncclAllReduce 到来时，planner 在 enqueue.cc 里做的是
```text
消息大小 = 本次 count × datatype_size
  → 落入对应的消息大小桶
  → 在该桶的代价表中，比较所有有效 (algo, protocol) 的 estimated time
  → 选最小的那个
  → 按选定 algo 去查 pattern（Ring→RingTwice，Tree→TreeUpDown）
  → 按channel 切 chunk，生成 launch plan
  → 并行结构同时执行
```