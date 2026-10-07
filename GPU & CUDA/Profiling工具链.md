优化顺序：
看整体时间线（PyTorch Profile）-> 找热点算子（Nsight System）-> 进入Kernel内部看微架构（Nsight Compute）

| 工具 | 看什么 | 回答什么问题 |
|---|---|---|
| PyTorch Profiler | 算子级 CPU/GPU 时间 | 哪个 op 最慢？DataLoader 是否拖后腿？ |
| Nsight Systems | 系统级时间线 | GPU 是否在等 CPU、通信、拷贝？ |
| Nsight Compute | 单个 kernel 内部 | 这个 kernel 为什么慢？bound 在哪？ |

PyTorch Profiler 最贴近框架层，适合快速定位算子。
Nsight Systems 看 CPU、GPU、CUDA API、memcpy、NCCL 的完整交互。
Nsight Compute 只看单个 kernel，但能提供 occupancy、warp stall、L1/L2 命中、DRAM 吞吐等深层指标。