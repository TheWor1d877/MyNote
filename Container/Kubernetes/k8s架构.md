由控制平面与Node节点组成
![[Attachments/Pasted image 20260507095234.png]]

## 控制平面组件
#### kube-apiserver
API 服务器是 Kubernetes控制平面， 该组件负责公开了 Kubernetes API，负责处理接受请求的工作。 API 服务器是 Kubernetes 控制平面的前端。
Kubernetes API 服务器的主要实现是kube-apiserver。 `kube-apiserver` 设计上考虑了水平扩缩，也就是说，它可通过部署多个实例来进行扩缩。 你可以运行 `kube-apiserver` 的多个实例，并在这些实例之间平衡流量。

#### etcd
一致且高可用的键值存储，用作k8s的后台数据库

#### kube-scheduler
kube-scheduler 是控制平面的组件， 负责监视新创建的、未指定运行节点的 Pod，并选择节点来让 Pod 在上面运行。
调度决策考虑的因素包括单个 Pod 及多个 Pod 集合的资源需求、 软硬件及策略约束、亲和性及反亲和性规范、数据位置、工作负载间的干扰及最后时限。
只负责分配Pod，只负责调度
#### kube-controller-manager
kube-controller-manager 是运行控制器进程的控制平面组件。
从逻辑上讲，每个控制器都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在同一个进程中运行。
控制器有许多不同类型。以下是一些例子：
- Node 控制器：负责在节点出现故障时进行通知和响应
- Job 控制器：监测代表一次性任务的 Job 对象，然后创建 Pod 来运行这些任务直至完成
- EndpointSlice 控制器：填充 EndpointSlice 对象（以提供 Service 和 Pod 之间的链接）。
- ServiceAccount 控制器：为新的命名空间创建默认的 ServiceAccount。
## 节点组件
#### kubelet
- 接收指令： 从控制平面接受相关的指令信息
- 启动/停止pod
- 健康检查： 定时检查Pod里面的容器是否还活着
- 向控制平面汇报
- 挂载存储/配置

#### kube-proxy
每个节点的网络代理，实现k8s服务概念的一部分

维护一些集群内部或者外部与pod进行网络通信的规则

## 插件
DNS
Web界面
容器资源监控
集群层面日志
