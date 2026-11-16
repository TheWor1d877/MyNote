NCCL 就是 NVIDIA 为多 GPU、多节点集合通信设计的库
只负责集合通信原语的加速，不管理训练循环，任务调度，编程模型等

```text
训练脚本 / 上层框架（PyTorch DDP、TensorFlow、JAX）
        ↓
NCCL
        ↓
GPU 驱动 + 互联硬件（NVLink / NVSwitch / PCIe / InfiniBand / RoCE）
```
## 通信形式
- 两种通信方式
点对点通信：点对点是两个 GPU 之间的 send/recv
集合通信：集合通信是一组 GPU 在同一个通信域内协同完成一次数据交换或聚合

- rank与communicator
每个参与集合通信的 CUDA device 被分配一个 rank。
一组 rank 构成一个 communicator
集合操作必须由 communicator 中的每个 rank 用相同的 count 和 datatype 调用，才能构成一次完整的集合操作
