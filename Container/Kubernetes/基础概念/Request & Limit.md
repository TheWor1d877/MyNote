#面试-云原生-复习 什么是 Request 与 Limit？CPU 和内存的 Request/Limit 在调度与运行时分别如何生效？
## Request & Limit
Request 是给调度器看的"我至少要这么多"，Limit 是给内核看的"我最多只能这么多"Request 影响调度。

调度器在给 Pod 选节点的时候，看的是节点上所有 Pod 的 Request 总和有没有超过节点容量。Request 是调度器做的"预留",不是"实际占用"。
   
Limit 影响运行时。 Pod 调度上去之后，kubelet 会把 Request 和 Limit 翻译成 cgroup 参数写到内核里。Limit 是内核真正执行的硬约束——CPU 超了会被 throttle，内存超了会 OOM kill。这部分跟调度器没关系了，是 cgroup 的活。


#### CPU与内存的不同
CPU是可压缩资源： request看的是权重，而不是实际的量，按照比例进行分配
limit如果超了就会等待而不是杀死

内存是不可压缩资源（incompressible）
Limit看的是容器实际占用的内存包括匿名页，pagecache，内核内存等，超过了只能杀死进程
内存的 Request 不构成运行时保护
内存 Request不能设得远小于 Limit
## k8s的QoS
K8s 还根据 Request/Limit 的组合定义了 QoS 等级
- Guaranteed：每个容器都设了 Request 和 Limit，且两者相等。最高优先级，节点内存紧张时最后被杀。
- Burstable：设了 Request 但 Request < Limit，或者只设了其中一个。中等优先级。
- BestEffort：什么都没设。最低优先级，节点压力大时最先被杀