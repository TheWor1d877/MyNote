dockerd： Docker平台的守护进程，用户与整个容器交互的统一入口
Containerd： 专门负责管理容器在宿主机器上面的声明周期
runc： 一套命令行工具，根据OCI标准，规范的创建与运行容器
shim： containerd-shim 是一个垫片进程，位于 containerd 和 runc 之间。每个运行的容器都对应一个独立的 shim 进程。
```bash
┌─────────────────┐
│   Docker CLI    │ ← 用户直接交互
├─────────────────┤
│  Docker Daemon  │ ← (dockerd) 单体应用管理层  
├─────────────────┤
│   Containerd    │ ← 工业级容器运行时（CNCF 毕业项目）
├─────────────────┤
│      runc       │ ← OCI 标准参考实现（真正的容器创建者）
└─────────────────┘
        ↓
┌─────────────────┐
│  Linux Kernel   │ ← Namespaces + Cgroups + Capabilities
└─────────────────┘
```
- containerd负责拉取镜像层，解压镜像层，叠加层合并，写入 config.json(打开镜像，构建文件系统)
- runc： 读取 config.json，执行系统调用，把准备好的目录真正挂载到容器的 mount namespace。（将文件系统加载到容器内部）

## Containerd 
containerd 负责“做什么”：操心容器运行的方方面面，从拉取镜像到最终删除，它都管。
Containerd承担了运行一个容器所需的所有高级操作与状态管理
#### Contianerd的具体功能
-  镜像管理：负责从 Docker Hub 等仓库 拉取 (pull) 容器镜像，并管理镜像在本地磁盘上的存储。
- 容器声明周期管理：接收创建、启动、停止、删除容器的请求，并协调底层组件去执行。
- 存储管理：为容器准备文件系统。它会将拉取的镜像解压并叠加，为每个容器创建隔离的读写层
- 网路附加： 为容器配置网络，将其加入到指定的网络命名空间中，使其能与其他容器或外部通信。
## runc
runc 是一个命令行工具，用于根据 OCI（Open Container Initiative，开放容器标准）规范创建和运行容器。它是 Docker 最初捐赠给 OCI 的项目，现在作为 OCI 标准的参考实现
#### runc的具体做法
1. 准备容器环境（namespace，cgroup，Capabilities，Seccomp）
2. 使用`fork()`创建容器进程，执行`pivot_root`切换到容器的rootfs,执行用户所需命令
3. 管理容器的声明周期
runc 本身是短暂存在的，一旦 runc run 启动容器后，runc 进程就会退出，容器的 init 进程由它直接启动，成为独立的进程树。
## shim
containerd-shim 是一个垫片进程，位于 containerd 和 runc 之间。每个运行的容器都对应一个独立的 shim 进程。
containerd进程与shim之间保持通信连接 ，shim是containerd创建的管家进程，shim是容器进程的父进程，保持容器进程的stdio，持续继续管理容器进程
#### shim的具体作用
- 解耦containerd与容器进程
```go
// 伪代码示例
// 没有 shim 的情况（糟糕的设计）
containerd (PID=1000)
  └── container-1 (PID=1001)
  └── container-2 (PID=1002)
// 问题：containerd 挂了 → 所有容器都挂

// 有 shim 的情况（实际设计）
containerd (PID=1000)
  
container-1-shim (PID=1003)  // 独立进程
  └── container-1 (PID=1004)
  
container-2-shim (PID=1005)
  └── container-2 (PID=1006)
// 好处：containerd 挂了 → shim 继续存在，容器继续运行
```
这样就能保证了contianerd的热升级
当 containerd 需要升级时，可以安全停止
所有 shim 进程不受影响，继续管理容器
升级完成后，新版本的 containerd 重新连接 shim

- 容器IO的持久化
```bash
# shim 负责保持容器的标准输入输出
docker run -it ubuntu bash
```
即使 containerd 重启，你还能看到输出,因为 shim 一直持有这些文件描述符
## dockerd
dockerd（通常称为 Docker Daemon）是 Docker 平台的核心守护进程，是用户与整个容器系统交互的统一入口
它面向用户，提供友好的 API 和丰富的功能，然后把复杂的任务交给 containerd 去执行。
containerd 是底层组件，像发动机，高效但不够易用
dockerd 是产品，像整车，提供方向盘、空调、导航等完整体验

## 执行`docker run`的完整流程
- Docker CLI 到 Docker Deamon
```bash
# 用户输入
docker run -d --name my-nginx -p 8080:80 nginx

# CLI 将命令转换为 HTTP API 调用
POST /containers/create
{
  "Image": "nginx",
  "HostConfig": {
    "PortBindings": {"80/tcp": [{"HostPort": "8080"}]},
    "PublishAllPorts": false
  }
}
```
- Docker Deamon 到Containerd
Docker daemon 通过 gRPC API 与 containerd 通信

gRPC 使用 Protobuf 进行序列化，数据体积更小、解析速度更快。在频繁的容器创建、停止、状态查询等操作下，这种性能优势对系统的整体吞吐量至关重要。

虽然 gRPC 通常基于 HTTP/2，但在 Docker 场景中，通信并非走 TCP 网络，而是通过Unix Domain Socket文件（如 /run/containerd/containerd.sock）
不走网络协议栈，并且文件系统权限天然提供了访问控制，增强了安全性
- Containerd 到 runc
containerd 通过 shim进程调用runc

shim关键作用：
解耦 containerd 与容器进程
即使 containerd 重启，容器继续运行
负责收集容器 stdout/stderr
处理容器退出信号

- runc 发送消息到内核进程，执行系统调用