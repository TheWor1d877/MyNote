两种实现算法：Ring AllReduce与Tree AllReduce
AllReduce 的通信量很大，在数据并行训练中每次反向传播都要执行。不同的通信模式在带宽利用率和延迟上有不同的取舍
没有一种算法在所有消息大小和拓扑下都最优，所以 NCCL 需要多种算法并根据场景切换。
## Ring AllReduce
Ring AllReduce 把 N 个 GPU 排成一个逻辑环，每个 GPU 只和左右邻居通信。

两个阶段：Reduce-Scatter与All-Gather

#### Reduce-Scatter
数据被切成 N 个 chunk。每个 GPU 将自己持有的 chunk 沿环传递，每经过一个 GPU 就累加一次。经过 N-1 步后，每个 GPU 持有一个已经完全归约的 chunk。
```text
以 4 个 GPU、4 个 chunk 为例：

初始：每个 GPU 持有全部 4 个 chunk 的本地数据
      GPU0: [a0, b0, c0, d0]
      GPU1: [a1, b1, c1, d1]
      GPU2: [a2, b2, c2, d2]
      GPU3: [a3, b3, c3, d3]

Reduce-Scatter 结束后（每 GPU 持有一个完整归约的 chunk）：
      GPU0: [A]   （A = a0+a1+a2+a3）
      GPU1: [B]   （B = b0+b1+b2+b3）
      GPU2: [C]
      GPU3: [D]
```

#### All-Gather
每个 GPU 把自己持有的那个已归约的 chunk 沿环转发。经过 N-1 步后，所有 GPU 都拿到全部 N 个 chunk 的归约结果
```text
All-Gather 结束后：
      GPU0: [A, B, C, D]
      GPU1: [A, B, C, D]
      GPU2: [A, B, C, D]
      GPU3: [A, B, C, D]
```

#### 特点
总步数：约 2(N-1) 步。
每 GPU 通信量：发送和接收各约 2(N-1)/N × S，S是初始GPU拥有的数据总量，N 很大时趋近于 2S。每 GPU 的通信量不随 GPU 数量线性增长。
链路负载：均匀，没有中心瓶颈。
适用场景：大消息、带宽主导的场景，比如训练中同步整个模型梯度。小消息下步数多，固定开销占比高，延迟会累积。
拓扑适应性：NCCL 会根据拓扑构建多个环或分层环，尽量让环上相邻的 rank 在物理上也相邻。


## Tree AllReduce
把 GPU 组织成树形结构。整个过程也分两个阶段，对应 Reduce 和 Broadcast。

#### Reduce（从叶子向根汇聚）
```text
以 4 个 GPU 的二叉树为例：

        GPU0（根）
       /        \
    GPU1        GPU2
    /
 GPU3

Reduce 阶段：
  GPU3 发给 GPU1 → GPU1 归约
  GPU1 发给 GPU0 → GPU0 归约
  GPU2 发给 GPU0 → GPU0 归约
  根节点 GPU0 持有完整归约结果
```

#### Broadcast（从根向叶子分发）
根节点把归约结果沿树向下发给子节点，子节点再继续向下分发。经过 O(log N) 轮后，所有 GPU 都拿到完整结果。
```text
Broadcast 阶段：
  GPU0 发给 GPU1 和 GPU2
  GPU1 发给 GPU3
  所有 GPU 持有完整归约结果
```

#### 特点
总轮次：约 2·log₂(N) 轮。8 个 GPU 约 6 轮（Reduce 3 轮 + Broadcast 3 轮），远少于 Ring 的 14 步。
每轮数据量：通常搬完整数据块或较大块，是“用带宽换延迟”的策略。
瓶颈：根节点附近的链路容易成为瓶颈，带宽利用率不如 Ring。NCCL 实际实现中会做树的重排、多棵树（forest）来缓解，不一定是单根二叉树。
适用场景：小消息、延迟敏感的场景。轮次少，固定开销累积少。


## 补充：如何拓扑感知
这里容易混淆：Ring、Tree 是逻辑算法，NVLink、PCIe、InfiniBand 是物理链路。
NCCL 的工作就是把逻辑算法映射到物理链路上。
根据不同路径的情况进行打分，接着在成本图上面构建ring，tree，channel等逻辑算法，尽量让数据走高带宽，低延迟路径
