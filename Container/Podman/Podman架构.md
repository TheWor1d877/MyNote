```text
Podman CLI → libpod (状态管理 + 编排) → conmon → runc
                 ↑
   containers/image (镜像管理)
   containers/storage (存储管理)
```

## 关键组件
#### Podman CLI
直接在用户上下文中运行
使用 libpod 库处理容器逻辑
支持 root 和 rootless 模式


#### Common
轻量级 C 程序（~50KB）
每个容器对应一个 conmon 进程

职责：
监控容器主进程
收集 stdout/stderr 日志
处理信号转发
维护容器状态

与docker中的shim类似

#### Libpod
Podman 的核心库(库函数调用)
提供容器、Pod、卷、网络等抽象
直接调用底层运行时（runc/crun）.

负责容器的状态管理与容器编排

#### Storage Driver
使用 containers/storage 库
支持 overlay、btrfs、zfs、vfs 等
兼容 Docker 的镜像格式

## Rootless模式
原理：
- User Namespace 映射
```bash
cat /etc/subuid
# 输出： howardhe@ASUS-HE:~$ cat /etc/subuid
#		howardhe:100000:65536
```
从uid100000开始往后映射65536个
容器内 UID 0 → 宿主机 UID 100000
容器内 UID 1 → 宿主机 UID 100001

- Slirp4netns
用户态 TCP/IP 协议栈
将容器网络流量通过 TAP 设备转发到用户态
避免需要 root 权限创建 veth pair

- Rootlesskit（可选）
处理 rootless 模式的网络和挂载
创建 slirp4netns 网络栈
设置 FUSE overlayfs（如果内核不支持 user namespace overlay）