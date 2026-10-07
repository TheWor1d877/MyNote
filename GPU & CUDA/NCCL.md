NCCL 就是 NVIDIA 为多 GPU、多节点集合通信设计的库
## 通信集合原语
主要依靠集合通信原语来指定通信规则：

| 原语             | 含义                        | 典型场景            |
| -------------- | ------------------------- | --------------- |
| Broadcast      | 一个 GPU 把数据发给所有 GPU        | 参数初始化、下发配置      |
| All-Gather     | 每个 GPU 贡献一块，所有 GPU 拿到完整结果 | 模型并行中收集参数       |
| Reduce-Scatter | 所有 GPU 的数据先归约，再分散到各 GPU   | AllReduce 的中间阶段 |
| All-Reduce     | 所有 GPU 归约后，每个 GPU 都拿到完整结果 | 数据并行梯度同步        |
| All-To-All     | 每个 GPU 向所有 GPU 发送不同数据     | 专家并行、序列并行       |

其中 AllReduce 最关键。数据并行训练里，每张 GPU 独立算一个 batch 的梯度，然后通过 AllReduce 把全局梯度汇总、取平均，再同步回所有 GPU。

## Ring AllReduce
Ring AllReduce 把 N 个 GPU 排成一个逻辑环，每个 GPU 只和左右邻居通信。

它分为两个阶段：
Reduce-Scatter：数据被切成若干 chunk，沿环传递并逐步归约。每步每个 GPU 收到一个 chunk，累加后再传给下一个 GPU。
All-Gather：每个 GPU 把自己持有的最终 chunk 沿环转发，经过 N-1 步后，所有 GPU 都拿到完整结果。

它的好处是：
每条链路负载均匀；
没有中心瓶颈；
带宽利用率高；
通信量不随 GPU 数量线性爆炸。

缺点是通信步数约 2(N-1)，GPU 数量增加后，小消息的延迟会累积。

所以 Ring 适合大消息、带宽主导的场景，比如训练时同步整个模型梯度

## Tree AllReduce
Tree AllReduce 把 GPU 组织成树形结构。Reduce 阶段从叶子向根汇聚，Broadcast 阶段再从根向叶子分发。
它的通信轮次是 O(log N)，比 Ring 的 O(N) 少很多。

例如 8 个 GPU：
Ring 需要约 14 步；
二叉树只需要约 3 层。

所以 Tree 在小消息、延迟敏感时更有优势。但它的缺点是根节点附近容易成为瓶颈，带宽利用率不如 Ring。

## 拓扑感知
这里容易混淆：Ring、Tree 是逻辑算法，NVLink、PCIe、InfiniBand 是物理链路。NCCL 的工作就是把逻辑算法映射到物理链路上。
根据不同路径的情况进行打分，接着在成本图上面构建ring，tree，channel等逻辑算法，尽量让数据走高带宽，低延迟路径

##  NVLink 与 InfiniBand
NVLink 是节点内 GPU 互连。
以 H100 为例，NVLink 4.0 单 GPU 双向带宽可达约 900 GB/s，延迟通常在亚微秒到 1 微秒级别。 它专为 GPU-GPU 通信设计，支持多链路并行、低协议开销和统一虚拟地址。
InfiniBand 是节点间网络。
例如 400G IB 网卡，单向理论带宽约 50 GB/s 左右，实测健康集群通常可达 45 GB/s 以上，延迟在微秒级别。 它通过 RDMA 实现低 CPU 开销传输，但带宽远低于 NVLink。

节点内看 NVLink，节点间看 IB/RoCE。