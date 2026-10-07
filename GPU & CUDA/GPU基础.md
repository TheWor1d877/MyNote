CPU的一个核心是"全能选手"：能取指令、解码、分支预测、乱序执行、大容量缓存……它追求的是单个线程的极致低延迟。
GPU的一个CUDA Core本质上只是一个ALU（算术逻辑单元）。它不能独立取指令、不能解码、不能做分支决策。它就是一个纯粹的"计算工人"，等着上面的"工头"告诉它做什么。

## GPU层级架构
```text
GPU芯片
 └── GPC（图形处理集群）—— 最大的物理分区
      └── TPC（纹理处理集群）
           └── SM（流多处理器）—— 真正的"计算单元"
                ├── CUDA Core（通用计算）
                ├── Tensor Core（矩阵加速）
                ├── RT Core（光追加速）
                ├── 寄存器文件（Register File）
                ├── 共享内存（Shared Memory）
                └── Warp调度器
```

SM才是GPU的"CPU核心"等价物。 一个SM内部包含几十个CUDA Core + 调度器 + 片上内存。以A100为例，它有108个SM，每个SM含64个CUDA Core，所以108 × 64 = 6912。


#### SM内部存储层级
```text
SM 内部存储层级（从快到慢、从私有到共享）：

寄存器（Register）        ← 线程私有，编译器分配
    ↓
共享内存（Shared Memory） ← Block 内共享，程序员控制
    ↓
L1 缓存                   ← SM 内自动缓存，对程序透明
    ↓
L2 缓存                   ← 全 GPU 共享
    ↓
全局内存（Global Memory） ← 显存，容量大但慢
```
Shared Memory（共享内存） 是 GPU 中位于 SM 内部的一块高速、可编程的片上存储，专门用于同一个线程块（Thread Block）内的线程之间共享数据。
每个SM之间L1缓存是独立的,L2缓存,显存都是共享的


| 层级 | 延迟 | 容量 | 作用 |
|------|------|------|------|
| 寄存器 | ~0周期 | 每线程64KB | 线程私有变量 |
| 共享内存 | 1-2周期 | 每SM 164KB | 同SM内线程共享 |
| L1/L2缓存 | 30-50周期 | 几十MB | 自动缓存 |
| 全局显存（HBM） | 400-600周期 | 40-80GB | 主存储 |
全局显存的延迟是寄存器的几百倍。 如果CUDA Core每次都要从显存取数据，99%的时间都在等数据，算力利用率不到1%。
#### SM的关键能力
<span style="color:rgb(221, 85, 85)">SM 的关键能力是：同时驻留很多 warp。当某个 warp 因为访存、同步或分支停顿，scheduler 立刻切换到另一个就绪 warp。</span>

GPU的高吞吐是靠总有线程跑得快,而不是单一线程

<span style="color:rgb(221, 85, 85)">这也解释了为什么 occupancy 很重要：如果每个线程占太多寄存器，或者每个 block 占太多 shared memory，SM 上能同时放的 warp 就少，延迟隐藏能力下降</span>

#### register file,shared memoty, L1的资源竞争
SM的资源是固定的,这些资源被同时驻留的Wrap/Block瓜分,每个Bolck拿得多,总wrap数就会小

资源	                     说明
寄存器文件	         比如 64K 个 32-bit 寄存器
Shared Memory	 比如 100KB 左右
最大 Block 数	     比如 32 个
最大 Warp 数	     比如 64 个

如果kernel中每个Bolck声明的shared memory多:
```text
kernel 里 __shared__ 声明得大
        ↓
每个 Block 需要占用更多 Shared Memory
        ↓
SM 的 Shared Memory 总量固定
        ↓
能同时容纳的 Block 数量 = SM_SharedMem_总量 / 每个Block_用的SharedMem
        ↓
每个 Block 用的越多，能放的 Block 越少
        ↓
Block 少 → 里面的 Warp 总数少 → Occupancy 下降
```

#### Coalesced Access: wrap访问global memory的基本规则
一个 warp 有 32 个线程。每个线程读一个 float，总共 128 字节。如果这 32 个线程访问的地址是连续、对齐的，硬件会把它们合并成很少的内存事务
好模式：
```text
thread 0  → data[0]
thread 1  → data[1]
...
thread 31 → data[31]
```
32 个 float 正好 128 字节，通常一次事务就能满足。

坏模式：
```text
thread 0  → data[0]
thread 1  → data[32]
thread 2  → data[64]
...
thread 31 → data[31*32]
```
每个线程访问的地址落在不同内存段，硬件可能需要很多次事务，带宽利用率大幅下降。
在 AI 场景里，batch 维度通常放在最外层、连续维度上，就是这个原因：让同一个 warp 处理连续样本或连续特征，访存更规整。

#### shared memory banking
Shared memory 虽然快，但不是“随便访问都快”。它被划分成多个 bank，常见是 32 个 bank，每个 bank 宽度 4 字节。
理想情况是：warp 内 32 个线程各访问不同 bank，一次完成。
如果多个线程访问同一个 bank 的不同地址，就会发生 bank conflict，访问被串行化。

## SIMT
一条指令，多个线程同时进行
每32个线程成为一个Wrap（线程束）
同一个wrap中的所有线程在同一时刻执行同一条指令
```text
Grid
 └─ Block（线程块，调度到单个 SM）
      └─ Warp（32 个线程，硬件最小调度单位）
           └─ Thread（最小执行单元）
```
Grid: 描述这个kernel需要处理多少数据
Block: 决定哪些线程可以共享shared memory
Wrap: 决定硬件如何批量执行和隐藏延迟
Thread


Block是程序员视角下面的资源单位,同一个block下面的线程共享shared memory
Warp 是硬件视角的执行单位：SM 不直接调度 thread，而是调度 warp。

举例: 
一个 block 有 256 个线程，它会被拆成 8 个 warp。SM 上的 warp scheduler 每次挑一个“可以跑的 warp”发射指令。32 个线程一起执行 add，就同时加 32 个数据；一起执行 load，就同时发起 32 个访存请求。

#### wrap divergence
Warp divergence 发生在同一个 warp 内线程走了不同分支。
```c
if (data[tid] > 0.5f) {
    data[tid] *= 4.0f;
} else {
    data[tid] += 2.0f;
}
```

如果 warp 里一部分线程进 if，一部分进 else，硬件不会让它们并行跑两条路。它会先让走 if 的线程执行，屏蔽掉走 else 的线程；然后再让走 else 的线程执行，屏蔽掉走 if 的线程。
也就是说，这个 warp 在这一段代码上有效吞吐下降。最坏情况是 32 个线程各走各的分支，相当于串行执行。