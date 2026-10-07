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