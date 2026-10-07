## Device Plugin与DRA
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

## DRA的核心对象
- DeviceClass 
集群级别对象，定义设备类别与筛选条件
例如定义一个 h100 类别，筛选条件可能是：驱动是 NVIDIA、型号是 H100、显存 80GB。
用户申请的时候只需要声明需要的是h100，不关注底层细节

- ResourceClaim
命名空间级别对象，表示某个工作负载（Pod）对设备的申领
比如“我要 4 张 H100，并且在同一 NVSwitch 域”。最终调度器会把分配结果写入它的 status.allocation，记录用了哪个节点、哪个驱动、哪个设备池、哪些设备。类比“点菜单”。

- ResourceClaimTemplate
命名空间级模板。Pod 不直接写 ResourceClaim，而是引用模板；控制器会为每个 Pod 自动生成一份独立的 ResourceClaim。

- ResourceSlice
由DRA Driver上报的节点设备库存表，描述某个节点上面，某个驱动管理了什么设备以及这些设备的详细属性（容量、NVLink 拓扑等）

- DeviceTaintRule
集群级别对象，用来给设备打污点，让调度器避开某些设备

#### 工作流程
1. DRA Driver启动,发现节点上面的GPU,上报属性
DRA Driver通常以DeamonSet的形式跑在每个节点上面
2. 创建ResourceSlice(库存表)
Driver 把这些设备信息写成 ResourceSlice，相当于向集群登记：“这个节点有哪些 GPU，能力如何”
3. 根据库存表创建DeviceClass,定义集群中的设备与筛选条件
4. 用户创建 ResourceClaimTemplate（"我要 4 张 H100，同 NVSwitch 域"）
5. Pod创建的时候引用了ResourceClaimTemplate,此时Pod还没有真正分配到设备上面
6. resourceclaim-controller 为 Pod 生成 ResourceClaim
控制器看到 Pod 引用了模板，就为这个 Pod 生成一个实际的 ResourceClaim，相当于把模板变成具体订单，并绑定到这个 Pod
7. Scheduler 的 dynamicresources 插件介入调度
调度器发现 Pod 有未分配的 ResourceClaim，于是读取 DeviceClass 和 ResourceSlice，匹配设备属性、容量、拓扑、污点等条件，决定用哪些具体设备。
8. 匹配ResourceSlice -> 分配设备 -> 更新ResourceClaim.status
调度器将选中的设备写入ResourceClaim的status里面,相当于锁定库存
9. Pod 调度到对应节点
10. kubelet + DRA Driver將CDI注入设备
kubelet要启动某个pod，需要某个GPU了，就调用DRA Driver准备设备
DRA Driver返回CDI设备信息，表明需要暴露的设备，设置的环境变量，挂载的驱动库等等
容器运行时通过CDI将GPU注入容器：Kubelet 把这份 CDI 说明交给容器运行时（containerd / CRI-O）。容器运行时在创建容器时

## dynamicresources 插件在调度框架的位置
dynamicresources 是 kube-scheduler 内置插件，在四个扩展点介入

| 扩展点       | 做了什么                                                                  |
| --------- | --------------------------------------------------------------------- |
| PreFilter | 检查 Pod 引用的 ResourceClaim 是否已分配；未分配则触发 DRA 分配流程                        |
| Filter    | 验证节点是否能访问已分配的设备；有 `DRAResourceHealth` 时检查设备健康状态                       |
| Reserve   | 在调度周期内锁定已分配的设备，防止并发冲突                                                 |
| PreBind   | 等待设备绑定条件满足（`DRADeviceBindingConditions`），超时默认 10 分钟；不满足则清除分配、Pod 重入队列 |

DRA的设备分配在PreFilter阶段，调度器先决定这个Pod使用哪些具体设备，然后在决定Pod放到那个节点上面

## 可切分设备与GPU共享
DRA 原生支持把物理 GPU 切分为多个实例，在多个 Pod 间安全共享：
- MIG（Multi-Instance GPU）：硬件级切分，显存和计算单元物理隔离。
- MPS（Multi-Process Service）：进程级共享，适合推理场景。
- 时间切片：软件级轮转，适合低负载任务。
这些共享策略通过 DeviceClass 中的配置声明，调度器在分配时自动遵守。
