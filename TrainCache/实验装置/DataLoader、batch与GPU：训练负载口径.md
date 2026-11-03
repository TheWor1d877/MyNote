## 概念关系

DataLoader 的 worker 进程读盘、解码、collate，产出的 batch 留在 CPU 内存。训练循环调用 `next(loader)` 取走 batch，再用 `.to(device, non_blocking=True)` 搬到 GPU。DataLoader 全程不碰 GPU。

实验指标 `wait` 是 `next(loader)` 的阻塞时长，在主机侧计时，与计算设备无关。它统计数据没到、算力空转的那段时间。

## 四种负载口径

| 口径 | 数据 | 计算 | 能支持的结论 |
|---|---|---|---|
| A CPU 真模型 | 真实读盘 | 模型前后向，CPU | 真实训练循环下的等待比例；慢，受 CPU 型号影响 |
| B CPU 模拟算力 | 真实读盘 | matmul 跑满 200ms | 受控 A/B 对比，step 时间可复现 |
| C GPU 真模型 | 真实读盘 | 模型前后向，GPU | GPU 是否等数据；需要真 GPU 节点 |
| D 标准库 reader 加模拟算力 | 真实读盘 | Python matmul | 现有五张图与对照层，去掉训练框架变量 |

## 选择

主对照用 B 或 D，不上 GPU。缓存系统比的是 I/O，GPU 只增加变量；CIFAR 级小模型在 RTX 4060 上一步几毫秒，JPEG 解码可能占大头，判别力下降。

需要真实训练循环的证据，加 A 作为独立实验臂：同一份 trace、同一个模型、CPU 跑，报 step 时间与 wait/step。

需要 GPU 等数据的证据，另起 C，单机真 GPU，不混进 kind 对照。指标用 GPU 利用率或 step 分段计时；wait/step 只覆盖数据等待，不能代替 GPU 空闲时间。

kind 集群不接 GPU。四个 worker 共用一块卡，还要配设备插件、运行时和镜像，没有实验意义。

GPU 训练只需一行设备搬运，不必写 CUDA 代码。
