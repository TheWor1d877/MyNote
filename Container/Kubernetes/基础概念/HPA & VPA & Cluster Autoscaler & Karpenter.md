#面试-云原生-复习 解释 HPA、VPA、Cluster Autoscaler、Karpenter 的差异，以及推理服务弹性扩缩容的难点
HPA管理的是副本数量，看的是CPU，内存，或者其他指标，然后加减Pod的Replica数
VPA管理的是单个资源的规格，观察Pod使用多少资源，然后调整对应的request/limit
Cluster Autoscaler是node级别的扩缩容，依赖节点组与云厂商的伸缩组之间通信来做扩缩容
Kerpenter是不设置节点组，先算需要什么机器，而且扩缩容策略比CA更高级

## HPA
根据应用负载（如 CPU、内存、自定义指标等），自动调整工作负载（Deployment/StatefulSet）的 Pod 副本数量。
增加或者减少Pod的数量
#### 关键组件
- Metrics Server (指标服务器)
从每个节点的 Kubelet 收集所有 Pod 的 CPU 和内存使用率，并聚合后通过 Metrics API (/apis/metrics.k8s.io/) 暴露给集群。 
- HPA Controller (控制器)
Kubernetes 控制平面中的一个内置控制器。它周期性地（默认每 15 秒）查询 Metrics API，计算出所需的目标副本数，并更新对应的工作负载对象（如 Deployment 的 replicas 字段）
- Workload Controller (工作负载控制器)

#### 注意事项
1. 多指标组合
HPA会独立计算每个指标所需的额副本数量，取得最大数量作为最终结果
2. 自定义指标
Custom Metrics
部署一个 Metrics Adapter (如 Prometheus Adapter, Keda)。
该 Adapter 从你的监控系统（如 Prometheus）拉取指标，并通过 Custom Metrics API (/apis/custom.metrics.k8s.io/) 暴露给 HPA。
3. 缩容冷却
HPA 默认有 5 分钟的缩容冷却期（可通过 behavior.scaleDown.stabilizationWindowSeconds 配置）。在此期间，即使指标下降，HPA 也不会缩容，以防止因短暂的负载下降而过早缩容。
4. 初始延迟
新创建的 Pod 在 --horizontal-pod-autoscaler-initial-readiness-delay（默认 30s）内不会被纳入指标计算，确保应用有足够时间启动和预热。

## VPA
VPA 全称 Vertical Pod Autoscaler（竖直 Pod 自动扩缩器）。
核心目标： 根据Pod的历史资源使用情况，自动设置resources.requests（特别是 CPU 和内存），达到避免资源浪费，避免OOMkilled/CPU节流的情况

#### VPA的缺点
如果要更改约束，需要重建Pod
所以VPA不适合用在： 不能容忍重启i的有状态服务上面，除非配合优雅终止与高可用架构
HPA + VPA 通常不同时用于同一个工作负载（因为 VPA 的重启会干扰 HPA 的副本决策）

#### 核心组件
- VPA Recommender（推荐器）
- VPA Updater（更新器）
- VPA Admission Controller（准入控制器）

#### VPA关键使用场景
单线程单进程任务，就算HPA启动多个Pod，总吞吐量也不会提升，因为瓶颈再单线程内部，但是:
VPA: 发现单个Pod的CPU时间不足，就会提升他的CPU request从而解决问题

CPU request不会提高进程优先级（nice），而是控制调度的时候的准入控制
当创建Pod的时候,k8s调度器会只考虑哪些剩余可分配CPU 大于当前Pod的request.cpu的节点

## Cluster Autoscaler
集群自动伸缩器
你事先在云上建好若干个节点组，比如"4 核 16G 的组"、"8 核 32G 的组"、"带 A100 的 GPU 组"，每个组背后对应云厂商的一个伸缩组。当它发现有 Pod 因为资源不够无法调度的时候：
先根据Pod所需要的资源需求，判断这个Pod应该属于那个节点组
然后调整节点组的期望节点数
云厂商的伸缩组收到消息之后，真正取创建一台机器，将pod放上去
等节点利用率低到一定阈值才缩容

## Karpenter
它同样盯着 Pending 的 Pod，但不查节点组，而是直接读这个 Pod 的调度要求：要多少 CPU 内存、要不要 GPU、要什么型号、亲和性是什么。然后它自己算出一个最合适的实例规格，直接调云厂商的接口把机器拉起来，起来之后还主动把 Pod 绑上去，不用等调度器再走一遍。

等负载降低之后可以将分散在多个节点上面的Pod挪到一起，把空出来的机器退掉，或者把贵的机型换成便宜的，Cluster Autoscaler 在这方面偏被动，主要靠节点利用率低到阈值才缩。

## 推理服务的弹性扩缩容难度
1. 冷启动慢，将模型拉入GPU，模型权重几百GB拉进GPU的时间是分钟级别的，加了预取缓存之后才压到几分钟
2. 扩缩容信号不好选： CPU 利用率对推理 Pod 没意义，GPU 利用率的含义也模糊。比较靠谱的是队列深度和 TTFT/TBT 这类 SLO 指标，但阈值设多少、缩容多快才不抖，得靠调。
3. 缩容要保守。GPU 节点贵，缩太快，流量一回来又得重新拉模型。所以推理服务的缩容窗口通常设得比普通服务长得多(扩容的成本太高了)