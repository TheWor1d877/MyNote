GPU有两套独立的层级
硬件层级：GPU → GPC → TPC → SM → Partition
执行层级：Grid → Block → Warp → Thread

```text
执行层级                    硬件层级
────────                    ────────
Grid ──────────────────→  整个 GPU（Block 分发到各 SM）
  │
  Block ───────────────→  一个 SM（Block 不跨 SM）
    │
    Warp ──────────────→  SM 内的一个 Partition（由该分区的 Scheduler 管理）
      │
      Thread ──────────→  执行单元上的一个 lane
```

## 一次kernel启动
#### 一次kernel启动的层级
一次 kernel 启动对应一个 Grid，Grid 由多个 Block 组成，GPU 的全局工作分发单元把 Grid 中的 Block 逐个分配给 SM。
一个 SM 能同时放多少个 Block，不取决于 Block 总数，而取决于每个 Block 占用的资源数量，比如寄存器、共享内存、线程数等，资源够就多放几个，不够就少放。
这里有两个关键约束：一个 Block 只能落在一个 SM 上，不能跨 SM 分布；但一个 SM 可以同时承载多个 Block，只要资源允许。

#### 一次kernel启动的调度
Block 落到 SM 后会被拆成若干 Warp，每个 Warp 固定 32 个线程，Warp 是 SM 调度的最小单位
Warp 被分配到 SM 的某一个 Partition，由该 Partition 的 Warp Scheduler 管理，一个 Warp 一旦分配到某个 Partition，就不会在两个 Scheduler 之间迁移
Scheduler 每个时钟周期从自己管理的就绪 Warp 中选一个，发射一条指令

Occupancy 本质上就是这个 Warp 池里有多少 Warp 可供 Scheduler 挑选：池子越大，某个 Warp 等内存而停住时，Scheduler 越有可能从池子里挑别的就绪 Warp 来填上空隙
Warp 池越大，Scheduler 可挑选的就绪 Warp 越多，单个 Warp 的等待越容易被其他 Warp 掩盖，指令发射空隙越少，延迟隐藏能力越强。

## Warp Divergence
正常情况下一个warp下的所有thread都执行同一条指令
但是如果指令流中有一个条件分支，但是warp中的thread对条件的判断结果不同，就会出现Divergence
```c
int a = data[tid];   // tid 是线程编号，每个线程不同
if (a > 0) {
    out[tid] = a * 2;
} else {
    out[tid] = 0;
}
```

#### 举例
假设Warp内16个线程满足条件，16个不满足。硬件不会让两条路径并行执行。它先让走路径A的16个线程执行，另外16个被屏蔽；路径A执行完后，再让走路径B的16个线程执行，前16个被屏蔽。有效吞吐在这个分支段降为一半。
```text
Warp 内 32 个线程：
Thread 0-15: 条件为真  ──→ 执行路径 A（Thread 16-31 屏蔽）
Thread 16-31: 条件为假 ──→ 执行路径 B（Thread 0-15 屏蔽）

时间轴：
[==== 路径 A 执行 ====][==== 路径 B 执行 ====]
← 只有 16 个线程活跃 →← 只有 16 个线程活跃 →
```

- 最坏情况
32个线程各走各的分支，Warp被串行执行32次。但实际中这种极端情况很少见，大多数分支发散的粒度小于32