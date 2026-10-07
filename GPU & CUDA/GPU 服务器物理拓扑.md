## NUMA: 非统一内存访问
在传统的 SMP（对称多处理）架构中，所有 CPU 核心共享同一根内存总线，就像所有人挤在一个厨房里用一个水龙头。当 CPU 核心数增加到几十上百个时，这根总线就成了瓶颈——这就是"内存墙"问题。

NUMA将服务器划分为多个节点，每个节点有自己的CPU核心，本地内存，内存控制器，GPU实例，PCLe交换机等
```text
NUMA Node 0                    NUMA Node 1
├── CPU 核心 0-31              ├── CPU 核心 32-63
├── 本地内存 512GB             ├── 本地内存 512GB
├── GPU-0, GPU-1, GPU-2, GPU-3 ├── GPU-4, GPU-5, GPU-6, GPU-7
└── PCIe 交换机 A              └── PCIe 交换机 B
```
如果 GPU-0（Node 0）需要读取 Node 1 的内存数据，数据必须跨越 NUMA 互联链路，延迟翻倍、带宽减半。这就是为什么拓扑感知调度要尽量让 GPU 和它需要的内存/数据在同一个 NUMA 节点内。
- 查看numa拓扑
```bash
numactl --hardware
```

## PCle
PCIe（Peripheral Component Interconnect Express）是一种高速串行点对点总线标准，用于连接 CPU 和各种外设（GPU、NVMe SSD、网卡等）。

GPU 通过 PCIe 插槽插在服务器上，数据从 GPU 到 CPU/内存必须走 PCIe 通道。

训练数据从内存加再到GPU显存走的是PCle
GPU计算结果写回内存走PCle

如果两张 GPU 不在同一个 PCIe 交换机下，它们之间的通信必须经过 CPU 和系统内存中转，带宽掉到 NVLink 的 1/30 甚至更低。这就是为什么"拓扑感知调度"要优先选择同一 PCIe 交换机下的 GPU 组合。

## NVLink
NVLink 是 NVIDIA 开发的GPU 专用高速互联技术，采用点对点结构，让多张 GPU 之间直接通信，绕过 CPU 和 PCIe。

NVlink主要用于节点内GPU通信

NVLink 拓扑类型
类型1：点对点直连（小规模）
```text
GPU-0 ↔ GPU-1 ↔ GPU-2 ↔ GPU-3
```
每张卡只和相邻的卡直连，适合 2-4 卡小规模训练。

类型2：NVSwitch 全互联（All-to-All，大规模）
```text
         NVSwitch（高速交换矩阵）
        /    |    |    |    \
    GPU-0 GPU-1 GPU-2 GPU-3 GPU-4 ...
```
通过 NVSwitch 芯片，实现任意两张 GPU 之间的高速直连，无阻塞通信。8 卡 A100/H100 服务器标配这种拓扑。