| 维度     | Docker                             | Podman                                                         |
| ------ | ---------------------------------- | -------------------------------------------------------------- |
| 架构模式   | 客户端-服务器（C/S）架构，依赖 `dockerd` 守护进程   | 无守护进程（daemonless），直接调用 OCI 运行时（如 runc）                         |
| 权限模型   | 默认需要 root 权限运行容器（除非配置 rootless 模式） | 原生支持 rootless 容器（普通用户即可运行）                                     |
| 安全性    | 守护进程是特权进程，存在单点故障和攻击面               | 更安全：无守护进程 + 用户命名空间隔离                                           |
| 命令兼容性  | 自成体系                               | 高度兼容 Docker CLI（`alias docker=podman` 可直接切换）                   |
| Pod 概念 | 不原生支持 Kubernetes 的 Pod 模型          | 支持 Pod 管理（多个容器共享网络/存储命名空间）                                     |
| 生态系统   | 成熟、广泛支持（Compose、Swarm、Desktop 等）   | 生态较新，需 `podman-compose` 替代 Docker Compose；对 Windows/macOS 支持有限 |
| 镜像存储   | 使用自己的镜像存储驱动（如 overlay2）            | 兼容 OCI 镜像格式，可与 Docker 镜像互换                                     |

最关键的是无守护进程与原生支持rootless


为什么 Red Hat 推 Podman 而弃用 Docker？” → 回答：安全、轻量、符合 OCI 标准、避免厂商锁定。

“Podman 能完全替代 Docker 吗？” → 视场景而定：Linux 服务器上可以；开发机若用 macOS/Windows 则 Docker Desktop 仍占优。