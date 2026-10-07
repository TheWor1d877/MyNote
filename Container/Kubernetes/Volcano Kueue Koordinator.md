## 介绍
Volcano是批量调度器 
关键能力:PodGroup、Gang Scheduling、优先级队列、Binpack/Spread 策略、DRF 公平共享

Kueue	是队列 + 配额管理
关键能力: 把 Job 放进 Queue，再放进 ClusterQueue，做准入控制；不直接调度 Pod，而是决定"这个 Job 能不能被调度器处理"

Koordinator 是混部 + 细粒度资源
关键能力: CPU/GPU 隔离、QoS、NUMA 感知、延迟敏感型任务保护

## 关键不同：
Volcano是替换/增强调度器本身，管理的是怎么选择节点的问题

Kueue是在调度器前面，管理哪些Pod可以进入调度器的

Koordinator是在节点层，管理节点如何切分

组合： Kueue 做准入 → Volcano 做 Gang 调度 → Koordinator 做节点资源隔离。

## 队列管理的两个层次
#### 作业队列管理
训练作业提交后先进入队列,按照优先级,创建时间,DRF等排序
Volcano 的 Queue 支持：
   优先级队列：高优先级 Job 先调度。
   权重队列：多租户按权重分配集群份额。
   容量队列：每个队列有 guaranteed / capacity 资源上限。

Pod 队列（Pod Queue）
一个 Job 被队列放行后，它的 Pod 才进入调度器的活动队列。这里就是 QueueSort 插件在工作——决定同一队列里哪个 Pod 先被调度。
