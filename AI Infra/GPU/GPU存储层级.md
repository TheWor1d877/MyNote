## device视角下访存路径
```text
SM 内 LD/ST 单元
      ↓
L1（与 Shared Memory 共享物理池，SM 私有）
      ↓
L2（全 GPU 共享，所有 SM 的流量都经过这里）
      ↓
显存控制器
      ↓
HBM
```
#### 层级表与跳变

| 层级 | 作用域 | A100 容量 | 延迟量级 | 带宽量级 |
|---|---|---|---|---|
| 寄存器 | 线程私有 | 256 KB / SM（65536 × 32-bit） | ~1 周期 | 最高，不经过片上网络 |
| Shared Memory | Block 内共享 | 最多 164 KB / SM | ~20 周期 | ~19 TB/s |
| L1 | SM 私有缓存 | 与 Shared 共享 192 KB 池 | ~30 周期 | ~19 TB/s |
| L2 | 全 GPU 共享 | 40 MB | ~150–200 周期 | ~4–5 TB/s |
| HBM | 全 GPU 共享 | 40–80 GB | ~400–600 周期 | ~1.5–2 TB/s |

HBM延迟是寄存器延迟的几百倍。如果CUDA Core每次访存都直接等HBM，算力利用率会低到个位数。
GPU在SM上驻留足够多的warp，让某个warp等HBM时调度器能切换到其他就绪warp。这就把存储层级和资源约束连在了一起。

## 资源约束如何限制驻留warp数
SM能同时驻留多少warp，不是由warp数量本身决定的，而是由每个warp和每个Block消耗的资源决定的。
A100的SM有三个硬上限：64个warp、32个Block、65536个寄存器。实际驻留数要同时满足这三个硬上限

- 寄存器约束：每个线程使用的寄存器数按warp以256个为单位向上取整分配。
- Shared Memory约束：每个Block声明的Shared Memory大小直接限制了SM上能放几个Block。
- Block数约束：即使寄存器和Shared都很宽裕，SM最多也只能同时驻留32个Block。