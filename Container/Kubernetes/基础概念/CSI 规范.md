CSI是一套标准的 gRPC 接口，让驱动变成一个容器化的独立进程，将存储驱动从 K8s 核心中剥离出来。存储厂商只需实现这套接口，就能让自己的存储系统接入任何支持 CSI 的编排系统 （如 K8s, Mesos）
kubelet 和 external-provisioner、external-attacher 这些 sidecar 容器去跟驱动通信。驱动本身完全不知道 K8s 的存在

在 CSI 出现之前，Kubernetes 的存储驱动（如 AWS EBS, GCE PD）是 “in-tree” 的，即代码直接写在 Kubernetes 核心仓库里


## CSI的核心组件
- Controller plugin
运行在控制平面，它通常以 Deployment 的形式运行，不绑定到特定节点。
负责 CreateVolume、DeleteVolume、ControllerPublish 这类集群级操作，它通过 K8s API 和外部 sidecar 协作。

StorageClass 里指定 provisioner 为某个 CSI 驱动的名字，PVC 触发动态供给时，external-provisioner 就会去调这个驱动的 CreateVolume。


- Node Plugin
运行在 每个工作节点（Worker Node） 上。在 Kubernetes 中，它通常以 DaemonSet 的形式部署，确保每个节点都有一个实例。
核心职责：将块挂载到宿主上面，再通过bind mount进容器


## 工作流程
1. 用户创建PVC并指定StorageClass
2. External Provisioner (Sidecar)监听PVC事件，发现新的 PVC 后，会根据其 StorageClass 找到对应的 CSI Driver，然后调用该 Driver 的 Controller Plugin 的 CreateVolume 方法。
3. External Provisioner 根据返回信息创建一个 PersistentVolume (PV) 对象，并将其与 PVC 绑定。
4. Pod 调度：当一个使用该 PVC 的 Pod 被创建后，调度器会将其调度到一个合适的节点上。
5. kubelet 调用 Node Plugin：目标节点上的 kubelet 发现 Pod 需要挂载卷，它会调用本机 CSI Driver 的 Node Plugin。

注意： Sidecar Containers 是 Kubernetes CSI 实现的关键，它们将 CSI 规范的操作与 K8s 的 API 对象（PVC, PV, VolumeAttachment）桥接起来。