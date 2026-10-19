#面试-云原生-复习 解释 etcd 在 Kubernetes 中的角色，以及 etcd 性能恶化时集群会出现哪些典型症状。
## etcd
etcd是k8s的唯一状态存储源头，集群里面的所有对象都只存储在这里

API Server 自己不存数据，它只是 etcd 的翻译层和守门人

K8s 里除了 etcd，没有任何组件持有这个集群的“真实状态”。Controller 的本地缓存、Scheduler 的视图、kubelet 看到的 Pod spec，全都来自 API Server 从 etcd 读出来的数据。

etcd 用的是 Raft 共识算法，写操作需要多数节点确认才算提交。所以它对磁盘 fsync 延迟和网络稳定性极其敏感。

## 性能恶化产生的典型症状
1. kubectl卡    etcdserver: request timed out
2. 控制平面迟钝    Scheduler 调度变慢，Controller reconcile 卡住
3. Leader Election抖动  Lease 续约是写 etcd，etcd 写延迟一高，续约超时，Leader 主动放弃身份，反复重新选举。
4. 误判： kubelet 心跳写不进 etcd，Node Controller 误判 NotReady，触发 Pod 驱逐。