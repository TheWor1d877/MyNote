## CPU与GPU
CPU要的是单线程快：大缓存、分支预测、乱序执行，都是为了让一条指令流尽量少停顿
GPU要的是整体吞吐高：单线程慢一点没关系，只要SM上同时有足够多的warp在跑，某个warp等数据时切到另一个，算力就不会闲着

GPU的单个CUDA Core远远没有CPU复杂，但是数量上多得多
GPU的延迟隐藏主要靠线程级并行，不靠缓存和预测，GPU的性能不能只看单线程

CPU的带宽一般在几十到几百GB/s，GPU用的HBM可以到TB/s级别
GPU访存延迟也高，HBM延迟是寄存器延迟的几百倍。如果CUDA Core每次都直接等HBM，算力利用率会非常低。GPU靠的是让很多warp轮流跑，把等待时间盖住。

## GPU是独立计算实体
GPU访存不经过CPU。GPU有独立的显存控制器、L2缓存、GMMU和SM计算单元

SM执行load/store的时候，路径是：
`SM → L1 → L2 → 显存控制器 → HBM`

CPU访问显存的路径：
`CPU → PCIe → BAR/MMIO 或 DMA/cudaMemcpy → GPU显存`
- **BAR/MMIO** ：CPU 通过 PCIe，直接读写 GPU 暴露出来的地址窗口（BAR 映射的 MMIO 区域）。
- **DMA/cudaMemcpy** ：CPU 发起传输请求，由 DMA 引擎或 GPU 的复制引擎把数据从主机内存搬到 GPU 显存，CPU 不逐字节参与搬运。
这条路径经过PCIe，比GPU内部访存慢得多

## SIMT
SIMT是Single Instruction, Multiple Threads，单指令多线程。
多个线程同时执行，硬件将32个线程组成一个Warp,同一个Warp内的线程在同一时刻执行同一条指令,每个线程有自己的寄存器和数据地址。

组织层次：
```text
Grid
 └─ Block（线程块，调度到单个SM）
      └─ Warp（32个线程，硬件最小调度单位）
           └─ Thread（编程模型最小执行单元）
```

## 其他术语

| 术语            | 所属层级        | 含义                                   | 易混点                         |
| ------------- | ----------- | ------------------------------------ | --------------------------- |
| GPU           | 硬件层级        | 独立计算芯片，含SM、L2、显存控制器、GMMU等            | 不是CPU普通外设                   |
| GPC           | 硬件层级        | 图形处理集群，GPU内最大物理分区之一                  | 不是执行层级                      |
| TPC           | 硬件层级        | 纹理处理集群，GPC内分区                        | 不是线程块                       |
| SM            | 硬件层级        | 流多处理器，真正的计算单元                        | 不是Block；Block会被调度到SM        |
| CUDA Core     | 硬件层级        | SM内通用计算执行单元                          | 不是线程                        |
| Tensor Core   | 硬件层级        | 矩阵加速执行单元                             | 不是框架概念                      |
| RT Core       | 硬件层级        | 光追加速单元                               | 与通用计算区分                     |
| Grid          | 执行层级        | 一次kernel启动的线程集合                      | 不是硬件                        |
| Block         | 执行层级        | 线程块，调度到单个SM，Block内线程可共享Shared Memory | 不是SM                        |
| Warp          | 执行层级/硬件调度单位 | 32个线程组成，SM调度的基本单位                    | 不是Wrap，不是Block              |
| Thread        | 执行层级        | 编程模型最小执行单元                           | 不是CUDA Core                 |
| SIMT          | 执行模型        | 单指令多线程                               | 不是硬件层级                      |
| SIMD          | 执行模型/指令级并行  | 单指令多数据                               | 不等同于SIMT                    |
| Register      | 存储层级        | 线程私有寄存器                              | 容量按SM寄存器文件和每线程上限讲           |
| Shared Memory | 存储层级        | Block内共享的片上存储                        | 不是L1                        |
| L1            | 存储层级        | SM内缓存                                | 每个SM的L1独立                   |
| L2            | 存储层级        | GPU内共享缓存                             | 全GPU共享                      |
| HBM           | 存储层级        | 显存主存储                                | 容量大、延迟高                     |
| Occupancy     | 资源约束指标      | 实际驻留warp数与最大warp数之比                  | 受寄存器、Shared Memory、Block数限制 |
| PCIe          | 系统通路层级      | CPU与GPU/外设的通用总线                      | 共享总线，常是主机到GPU的瓶颈            |
| NVLink        | 系统通路层级      | GPU间高速直连链路                           | 不经过CPU                      |
| NVSwitch      | 系统通路层级      | GPU全互联交换芯片                           | 多卡全互联                       |
| NUMA          | 系统通路层级      | 非统一内存访问，跨节点延迟带宽不同                    | 影响GPU与内存的拓扑距离               |
