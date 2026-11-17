## NVIDIA Container Toolkit
`docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi`容器内能输出 GPU 信息即成功。
nvidia-ctk runtime configure 会自动修改 /etc/docker/daemon.json，注入：
```json
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

## 架构拆解
| 组件                       | 角色         | 职责                                                                                                      |
| ------------------------ | ---------- | ------------------------------------------------------------------------------------------------------- |
| libnvidia-container      | 底层 C 库     | 与 NVIDIA 内核驱动（nvidia.ko）和用户态库（libcuda.so、libnvidia-ml.so）交互，动态探测驱动版本、GPU 拓扑、显存容量，生成设备节点集合（/dev/nvidia*） |
| nvidia-container-toolkit | 协议翻译器      | 解析 `--gpus` 参数，生成容器启动参数，自动注入 CUDA 库路径到 `LD_LIBRARY_PATH`                                                |
| nvidia-container-runtime | OCI 运行时封装器 | 符合 OCI 规范的 runc 封装器，在容器创建阶段介入，修改 config.json，注入 GPU 设备节点、驱动库挂载点、安全策略，然后交给真正的 runc 执行                    |

## GPU映射到容器内的方法
Docker 引擎本身不原生支持 GPU。它只把 /dev/nvidia* 当作普通字符设备，无法自动识别驱动依赖、CUDA 上下文、显存管理。ubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.15.0/deployments/static/nvidia-device-plugin.yml

NVIDIA Container Toolkit解决这个翻译的问题

流程：
1. 用户执行`docker run --gpus all nvidia/cuda:12.6.0-base nvidia-smi`
2. 识别到runtime = nvidia，然后调用nvidia-container-runtime
3. nvidia-container-runtime替换了docker默认的runtime，nvidia-container-runtime在 runc 真正启动容器之前，插入自己的一段逻辑，去修改这份 config.json。
4. 使用libnvidia-container来探测宿主机，驱动版本，GPU数量，设备节点等等
5. 动态修改容器的 linux.devices、linux.mounts、process.env：
     - 挂载设备节点：/dev/nvidia0 → 容器内 /dev/nvidia0
     - 挂载驱动库：/usr/lib/x86_64-linux-gnu/libcuda.so → 容器内同路径
     - 设置环境变量：CUDA_VISIBLE_DEVICES=0, LD_LIBRARY_PATH=...
     - 添加可能需要的二进制文件
6. 交给 runc 执行 namespace 隔离与进程启动
7. 容器内进程通过 ioctl 调用 nvidia-uvm 驱动接口，完成显存分配与 CUDA 初始化

注意： GPU驱动永远只在宿主机上面，容器启动的时候通过bind，mount挂载到容器内部

核心思想： 驱动与镜像解耦

优点：
- 同一套 vLLM 镜像，可以在驱动版本不同的节点上运行（只要宿主机驱动 ≥ 容器 CUDA 要求的最低版本）
- 驱动升级不需要重新构建镜像，只需重启容器
- 镜像体积从几个 GB（含驱动）降到几百 MB（只含 CUDA 运行时）

## Device Plugin
#面试-云原生-复习  什么是 Device Plugin？GPU 厂商如何通过它把 GPU 资源暴露给 Kubernetes

K8s原生只认识CPU和内存。GPU是"异构设备"，K8s调度器不知道节点上有几块GPU、型号是什么、拓扑关系如何。Device Plugin就是将GPU暴露成可调度资源的机制

#### 核心工作流程
1. 注册
Device Plugin启动，通过gRPC向kubelet注册，声明自己管理nvidia.com/gpu资源
2. 上报
Device Plugin调用nvidia-smi扫描设备，向kubelet上报GPU列表，kubelet将GPU作为拓展资源写入Node Status

3. 调度
用户Pod声明` resources.limits.nvidia.com/gpu: 1`，调度器根据Node Status中的GPU数量做匹配，选定节点

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

## NVIDIA Container Toolkit 与Device Plugin
Device Plugin 是“调度员”，Container Toolkit 是“搬运工”。

二者协作：
1. Device Plugin 上报 GPU 资源 → kubelet → API Server
2. 调度器根据 nvidia.com/gpu: 1 选定节点
3. kubelet 调用 Device Plugin 的 Allocate() 接口
4. Device Plugin 返回：
   - 设备路径：/dev/nvidia0
   - 环境变量：NVIDIA_VISIBLE_DEVICES=0
5. kubelet 通过 CRI 调用 containerd 启动容器
6. NVIDIA Container Toolkit 检测环境变量 NVIDIA_VISIBLE_DEVICES
7. 自动把对应的 GPU 设备文件和 CUDA 库挂载进容器

## MIG
#面试-云原生-复习 什么是 MIG？它和整卡分配、时间片共享相比，在推理服务隔离性上有什么取舍？

MIG（Multi-Instance GPU）是NVIDIA Ampere架构（A100）及以上支持的硬件级分区技术。而是在芯片层面把一块物理GPU切成多个完全隔离的实例。

每个MIG实例拥有：
- 独立的SM（流多处理器）——计算核心物理隔离
- 独立的L1/L2 Cache和内存总线——I/O通道隔离
- 独立的显存——显存带宽和容量硬隔离

MIG缺点：
MIG 的隔离是深入到芯片内部数据通路的。在 MIG 模式下，NVLink 和 NVSwitch 都被禁用，实例间的通信只能通过系统内存（CPU 内存）进行.
MIG 的切分配置是预先定义的，改配置通常需要清空 GPU 上的所有负载。整卡分配没有这个问题，想给谁就给谁。

适用条件：
- 多租户共享
- 混合负载运行
- 小任务上面使用GPU

MIG 的隔离性是硬件给的，时间片的“隔离性”是调度器口头承诺的，整卡分配则根本不需要讨论隔离。

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

## k8s DRA 动态资源分配

Device Plugin 的问题在于表达能力太弱，只能告诉调度器“有几张卡”，没法表达显存、算力、拓扑、NVLink 关系这些维度。

RDA从 K8s 1.26 引入，到 1.31+ 逐步稳定，再到 1.36 中扩展可分区设备、设备污点等能力，DRA 正在把 GPU 从“一个整数资源”变成“可描述、可切分、可拓扑约束的一等公民”。
它的核心变化是：
资源申请不再只写在 Pod 的 resources 里；
而是通过 ResourceClaim、DeviceClass、ResourceSlice 等对象描述；
调度器可以基于显存、算力、拓扑、NVLink 域做更精细的匹配；
支持单卡动态切分为多个逻辑单元，并在多个 Pod 间共享。
不过 DRA 的 YAML 写法比 Device Plugin 复杂不少。HAMi-DRA 这类项目正在做一层自动转换：用户仍然写类似 nvidia.com/gpu 的声明，Webhook 自动把它转成 DRA 的 ResourceClaim
