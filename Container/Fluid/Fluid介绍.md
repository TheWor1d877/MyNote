Fluid 是一个运行在 Kubernetes 上的数据编排与加速引擎
将底层的各种存储使用CRD抽象成标准的Dataset资源，让 Kubernetes 调度器能感知“哪个节点有缓存数据，从而把计算任务优先调度到有缓存的地方，实现“计算跟着数据走”

```text
业务应用 Pod 要读写数据
↓
Fluid（运行在应用和存储之间）负责：识别应用需要的数据、调度到有缓存的节点、注入 FUSE 客户端
↓
JuiceFS（运行在 Fluid 的框架内）负责：真正把数据从对象存储拉过来、缓存到本地、提供 POSIX 文件接口
↓
对象存储（如 MinIO）真正存放数据
```
## 基本流程
1. K8s 要创建 Pod，但不知道放哪个节点。

2. Fluid 的 Webhook 拦截这个 Pod，给它注入一条“数据亲和性”规则（preferredDuringSchedulingIgnoredDuringExecution）-5-11。

3. 规则内容：优先调度到“有该 Dataset 缓存 Worker Pod 的节点”。

4. K8s 调度器读到这条规则后，就把 Pod 放到有缓存的节点上。

所以 Fluid 告诉 K8s 的是 “哪些节点有缓存 Worker”，而不是“哪个数据块在哪个节点”。数据块的具体位置，是 JuiceFS 客户端在运行时自己通过一致性哈希去找的。

## 如果没有Fluid
如果每个应用 Pod 都直接挂载 JuiceFS（比如用 hostpath 或简单的 CSI），会面临：

缺乏 K8s 原生的调度与感知：Kubernetes 调度 Pod 时，根本不知道哪个节点上有 JuiceFS 的缓存数据。它可能把 Pod 调度到没有任何缓存的节点上，导致第一次读取数据非常慢，缓存的作用被完全浪费了

运维负担重、灵活性差：需要在每个计算节点上手动安装和维护 JuiceFS 客户端。挂载参数是写死的，很难根据不同应用的需求动态调整，也缺乏统一的监控和管理界面

资源隔离性差：运行在宿主机上的 JuiceFS 客户端进程，其 CPU、内存消耗无法被 Kubernetes 有效限制，可能会与业务应用争抢资源，影响稳定性

## Fluid管理JuiceFS(下层存储引擎)的缓存
Fluid 自己不实现数据缓存逻辑，它是一个编排层。在 JuiceFS 场景下，Fluid 做的其实是：
- 把 JuiceFS 的 FUSE 客户端容器化成 Pod（Worker Pod / FUSE Pod）
- 让 K8s 调度器感知到哪个节点有 JuiceFS 缓存数据，从而把计算任务调度过去

Fluid 拉起的缓存节点和 JuiceFS 的缓存节点不是一对一关系，而是 “Fluid Worker Pod 组” 对应 “JuiceFS 缓存组” 的关系-1-4。

具体来说：
Fluid 侧：JuiceFSRuntime 会创建一个 StatefulSet，拉起一组 Worker Pod（比如 3 个、5 个

JuiceFS 侧：这些 Worker Pod 通过相同的 缓存组名称（Fluid 默认设置为`<namespace>-<runtime>`）自动组成一个分布式缓存组，同一个缓存组内的节点，用一致性哈希决定某个数据块该放在哪个节点。所以数据是“散落”在这组 Worker 里的，而不是每个 Worker 对应固定的数据范围

## Fluid调度
Fluid 在调度训练 Pod 时，只看“这个节点有没有缓存服务（worker）”，不看“缓存里有没有数据”，更无法感知某个具体节点上是否已经缓存了“这次训练所需的那部分数据”

所以即使 JuiceFS 里是空的，只要缓存 worker 还在，节点还带着“我有缓存能力”的标签，训练 Pod 照样会被调度到这些节点上

Fluid 注入的 nodeAffinity 指向的是一个标签（如` fluid.io/node`），这个标签标记的是“这个节点上有 Fluid 的缓存 Worker 在运行”
节点 A（有 Worker + 刚好有你这次要的数据）：会被打上标签。
节点 B（有 Worker + 但你这次要的数据不在它的缓存里）：同样会被打上标签。

K8s 调度器看到两个节点都满足 `nodeAffinity`，它无法区分“谁的本地缓存里正好有 `/data/batch-001.tar`”。所以 Pod 照样可能被调度到节点 B。

