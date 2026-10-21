#面试-云原生-复习  解释 DRA（Dynamic Resource Allocation）与 NRI（Node Resource Interface）分别解决什么问题？相比 Device Plugin，它们在资源建模、分配灵活性与节点设备可观测性上补齐了哪些能力？

DRA替换的是设备怎么被描述，申领与分配的模型
NRI解决的是容器运行时在创建的时候如何被插件干预
## DRA
#### Device Plugin与DRA
Device Plugin只能请求标量数量的GPU资源，无法实现GPU分区以及拓扑感知调度等内容

有了DRA，可以更加灵活的请求资源：

| 维度     | Device Plugin            | DRA                         |
| ------ | ------------------------ | --------------------------- |
| 资源请求   | 标量数量 `nvidia.com/gpu: 2` | 结构化参数（型号、显存、拓扑、共享策略）        |
| 调度参与   | 被动：kubelet 上报数量，调度器只按数量筛 | 主动：DRA Controller 参与调度决策    |
| GPU 分区 | 需要额外 MIG Manager         | 原生支持 MIG / MPS / 时间切片       |
| 拓扑感知   | 不支持                      | 支持 NVLink/NVSwitch 域抽象      |
| 运行时重配置 | 不支持                      | 支持                          |
| 驱动要求   | NVIDIA Driver ≥ 525      | NVIDIA Driver ≥ 580，需启用 CDI |

CDI是容器设备接口，在DRA模式下面GPU走的是CDI路径，而不是Device Plugin的Unix Socket注册机制

资源建模上面，将设备从数字变成了带属性的资源，DRA驱动通过ResourceSlice把每台设备的结构化属性发布给调度器，用户可以创建ResourceClaim，使用表达式做细粒度过滤

分配灵活性上面，DRA把独占做成了一个可以配置的选项，多个Pod可以引用同一个ResourceClaim来共享设备，这是DRA的原生能力

可观测性上面，DRA 把设备状态纳入了 Kubernetes API，可以让驱动上报每台已分配设备的状态数据，比如网卡的 IP、MAC，或者设备的自定义健康条件

补充: DRA 里会“崩”的不是 CRD 对象本身，而是 DRA 驱动（Driver）的 kubelet plugin 进程。 这个进程通常以 DaemonSet 形式跑在每个节点上，它挂了 K8s 会按 DaemonSet 的逻辑把它拉起来

#### DRA的核心对象
- DeviceClass （集群级别）
集群级别对象，定义设备类别与筛选条件
例如定义一个 h100 类别，筛选条件可能是：驱动是 NVIDIA、型号是 H100、显存 80GB。
用户申请的时候只需要声明需要的是h100，不关注底层细节

- DeviceTaintRule（集群级别）
集群级别对象，用来给设备打污点，让调度器避开某些设备或者把某些设备保留给特定工作负载。

- ResourceSlice（节点级别）
是节点设备库存表，描述某个节点上面，某个驱动管理了什么设备（gpu-0、gpu-1）以及这些设备的详细属性（型号、显存、NVLink 拓扑等）

- ResourceClaim（命名空间级别）
用户或者工作负载发出的具体申领，引用一个DeviceClass，从这个类里拿满足某条件的设备，调度器匹配成功之后将结果写入`status.allocation`

- ResourceClaimTemplate（命名空间级别）
命名空间级模板。Pod 不直接写 ResourceClaim，而是引用模板；控制器会为每个 Pod 自动生成一份独立的 ResourceClaim。每个 Pod 拿到自己的设备，互不共享。这适合训练任务——每个 worker 需要自己的 GPU，不能和别人共用

你手动创建一个 ResourceClaim，然后让多个 Pod 直接引用它（通过 resourceClaimName）。这种情况下，所有引用同一个 Claim 的 Pod 共享同一份设备分配结果，并被调度到同一个节点。
Resource可以独占也可以共享

#### 工作流程
1. DRA Driver以DeamonSet的形式在每个节点上面启动,发现节点上面的GPU,上报属性

2. 创建ResourceSlice(库存表)
DRA Driver 把这些设备信息写成 ResourceSlice，相当于向集群登记：“这个节点有哪些 GPU，能力如何”

3. 根据库存表创建DeviceClass,定义集群中的设备与筛选条件

4. 用户创建 ResourceClaimTemplate详细说明需求

5. Pod创建的时候引用了ResourceClaimTemplate,此时Pod还没有真正分配到设备上面

6. resourceclaim-controller 为 Pod 生成 ResourceClaim

7. Scheduler 的 dynamicresources 插件介入调度
调度器发现 Pod 有未分配的 ResourceClaim，于是读取 DeviceClass 和 ResourceSlice，匹配设备属性、容量、拓扑、污点等条件，决定用哪些具体设备。

8. 匹配ResourceSlice -> 分配设备 -> 更新ResourceClaim.status -> 调度节点

9. kubelet启动Pod，并调用CDI创建容器
kubelet要启动pod，就调用DRA Driver准备设备，DRA Driver返回CDI设备信息，表明需要暴露的设备，设置的环境变量，挂载的驱动库等等（CDI路径）
Kubelet 把这份 CDI 说明交给容器运行时，容器运行时通过CDI将GPU注入容器，然后创建容器

## NRI
NRI允许用户通过插件，注册到容器运行时上，在容器的相关生命周期（创建、启动、删除）被回调，然后对容器的 OCI spec 做有限修改——加挂载、注入设备节点、设环境变量。

NRI路径：
管理容器启动，它没有通过 Device Plugin 或 DRA 申请 GPU。
containerd 准备创建容器，NRI 插件被回调。
插件检查这个容器：如果它是靠 NVIDIA_VISIBLE_DEVICES 环境变量要 GPU 的管理容器，就把 GPU 设备塞进去。
容器启动，能看到 GPU

NIR路径下，调度器根本不认识GPU这个设备，只认识变量名

#### 用途：
NRI 路径上的容器是管理容器，比如监控 agent、Device Plugin 自身。它们的需求是“看到节点上所有 GPU”，而不是“被分配一个或几个 GPU”