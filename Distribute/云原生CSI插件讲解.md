CSI插件的的吧呢是就是一套基于gRPC的标准化契约
CSI插件将存储硬件的工作从k8s的核心代码中剥离出来

## 三大核心gRPC服务
一个合格的CSI插件 必须实现下面三个gRPC服务：
- Identity Service： 握手协议，告诉k8s，我是谁我支持那些操作（如：创建卷，快照）
- Controller Service: 控制面，负责卷的声明周期管理,通常以Deployment的形式部署，通过调用个大存储厂商的API来创建一块云盘
- Node Service： 数据面，运行在每个计算结点上面，负责最底层的活： 格式化，挂载等等

## 动态供给（Dynamic Provisioning）
- Stroageclass
SC 不是存储本身，而是“创建存储的模板”。在 SC 中，你必须指定 provisioner（例如 diskplugin.csi.alibabacloud.com），并传入底层参数（如云盘类型 cloud_essd、回收策略 Delete）。

- WaitForFirstConsumer：解决可用区错配的“惨烈妥协”
这是一个极其关键的绑定模式。如果采用默认的 Immediate 模式，K8s 会先创建云盘，再调度 Pod。如果云盘创建在可用区 A，而 Pod 被调度到了可用区 B，挂载就会直接失败。
使用 WaitForFirstConsumer 模式，K8s 会先调度 Pod 到具体的节点，再根据该节点所在的可用区去调用 CSI 创建云盘。这在多可用区 AI 集群中是保命配置。


## 数据流加速
1. 系统层面的调度加速
 解决任务跑不起来，启动慢的问题。
 它把原本可能因为拓扑错配导致的“分钟级甚至小时级的 Pod Pending 等待”，变成了“毫秒级的顺畅调度”。在动辄几千个 Pod 的大模型训练集群中，这种“消除调度死锁、提升冷启动成功率”的系统级优化，就是最硬核的 AI Infra 能力。
2. 数据流的物理加速
比如阿里云用 ESSD、用 RDMA 网络、用 GPUDirect Storage 去提升 IOPS