## Device Plugin
K8s原生只认识CPU和内存。GPU是"异构设备"，K8s调度器不知道节点上有几块GPU、型号是什么、拓扑关系如何。Device Plugin就是GPU与K8s之间的翻译官。
#### 核心工作流程
1. 注册
Device Plugin启动，通过gRPC向kubelet注册，声明自己管理nvidia.com/gpu资源
2. 上报
Device Plugin调用nvidia-smi扫描设备，向kubelet上报GPU列表，kubelet将GPU作为Extended Resource写入Node Status
3. 调度
用户Pod声明 resources.limits.nvidia.com/gpu: 1，调度器根据Node Status中的GPU数量做匹配，选定节点
4. 分配
kubelet调用Device Plugin的Allocate()接口，返回具体GPU设备ID（如GPU-0），nvidia-container-toolkit在容器启动时注入驱动、挂载设备节点
#### 关键点
GPU是不可压缩资源，requests必须等于limits，不能像CPU那样超分
GPU只能在limits中指定，不能只写requests
禁止通过NVIDIA_VISIBLE_DEVICES环境变量或--gpus参数绕过调度器申请GPU，否则会导致资源争抢，这些做法让容器直接看到 GPU，但 Kubernetes 调度器完全不知道这个 Pod 用了 GPU。
- 不启动MIG，如果两个应用都写request  = 1会发生什么：
如果节点上有 ≥ 2 张 GPU：两个 Pod 会被调度到同一节点，但 Device Plugin 会给它们分配不同的 GPU（比如一个用 GPU 0，一个用 GPU 1），彼此隔离。
如果节点上只有 1 张 GPU：第二个 Pod 将无法调度，一直处于 Pending 状态，直到第一张 GPU 被释放。

###### 配置
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda
    image: nvidia/cuda:12.6.0-base-ubuntu24.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1    # 申请 1 张整卡
      # requests 可以不写，K8s 默认用 limits 的值
      # 如果写了，必须和 limits 相等
```
以下操作会绕过调度器，导致资源争抢：
```bash
# ❌ 禁止直接 docker run
docker run --gpus all my-image

# ❌ 禁止在 Pod 中硬编码环境变量
env:
- name: NVIDIA_VISIBLE_DEVICES
  value: "all"

# ❌ 禁止 privileged: true
securityContext:
  privileged: true
```
## MIG
MIG（Multi-Instance GPU）是NVIDIA Ampere架构（A100）及以上支持的硬件级分区技术。它不是软件虚拟化，而是在芯片层面把一块物理GPU切成多个完全隔离的实例。

每个MIG实例拥有：
- 独立的SM（流多处理器）——计算核心物理隔离
- 独立的显存——显存带宽和容量硬隔离
- 独立的L2 Cache和内存总线——I/O通道隔离

MIG缺点：
MIG 的隔离是深入到芯片内部数据通路的。如果允许 MIG 实例之间进行 P2P 直连，就需要在隔离的硬件单元之间重新建立高带宽的通信路径，这可能会削弱甚至破坏隔离的完整性。因此，在 MIG 模式下，NVLink 和 NVSwitch 都被禁用，实例间的通信只能通过系统内存（CPU 内存）进行，效率自然更低。

适用条件：
多租户共享
混合负载运行
小任务上面使用GPU

## 软件级共享：Time-Slicing 与 MPS
如果硬件不支持MIG，或者采用更加灵活的共享粒度，可以采用软件方案
#### Time-Slicing
NVIDIA GPU Operator 原生支持 Time-Slicing。它把一张物理 GPU 虚拟成多个逻辑副本，多个 Pod 轮流使用同一张卡。
优点：
配置简单，改一下 ConfigMap 就能生效；
没有显存虚拟化开销；
适合开发测试、Notebook、离线批处理。
缺点也很直接：没有显存隔离。多个 Pod 共享同一张卡的显存空间，任何一个 Pod 申请过多显存，都可能把其他 Pod 挤掉。
#### MPS
MPS 允许多个 CUDA 进程共享同一个 GPU 上下文，进程之间可以真正并行执行，吞吐损耗通常小于 5%。
它比 Time-Slicing 更适合“同一模型多副本”或“可信环境内的高并发推理”。但 MPS 同样缺乏强显存隔离，一个进程异常仍可能影响其他进程。
这两类方案都属于“软共享”：能提升利用率，但隔离性弱，生产环境里要配合监控和配额一起用

## 未来方向： k8s的 DRA动态资源分配

Device Plugin 的问题在于表达能力太弱，只能告诉调度器“有几张卡”，没法表达显存、算力、拓扑、NVLink 关系这些维度。

RDA从 K8s 1.26 引入，到 1.31+ 逐步稳定，再到 1.36 中扩展可分区设备、设备污点等能力，DRA 正在把 GPU 从“一个整数资源”变成“可描述、可切分、可拓扑约束的一等公民”。
它的核心变化是：
资源申请不再只写在 Pod 的 resources 里；
而是通过 ResourceClaim、DeviceClass、ResourceSlice 等对象描述；
调度器可以基于显存、算力、拓扑、NVLink 域做更精细的匹配；
支持单卡动态切分为多个逻辑单元，并在多个 Pod 间共享。
不过 DRA 的 YAML 写法比 Device Plugin 复杂不少。HAMi-DRA 这类项目正在做一层自动转换：用户仍然写类似 nvidia.com/gpu 的声明，Webhook 自动把它转成 DRA 的 ResourceClaim