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

