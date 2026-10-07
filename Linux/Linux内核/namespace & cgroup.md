在 Linux 系统中，Namespace 和 Cgroup 是实现容器化（如 Docker、Podman）的两大核心底层技术。它们解决的是不同维度的资源隔离问题。

## Namespace
为进程提供系统资源的隔离视图，让进程认为自己独享操作系统，看不到其他Namespace里面的进程与资源

Namespace 不是“给子进程用的特殊功能”，而是“每个进程都必然属于某个 namespace”。
内核为每个 namespace 类型都维护了一个初始的、默认的 namespace（称为 "root namespace" 或 "initial namespace"）。

**作用**：为进程提供**系统资源的隔离视图**。它让进程觉得自己是独享操作系统的，看不到其他 Namespace 里的进程和资源。

Linux 目前提供了 8 种主要的 Namespace：

| Namespace         | 隔离的资源              | 作用示例                                |
| :---------------- | :----------------- | :---------------------------------- |
| **Mount** (mnt)   | 文件系统挂载点            | 容器内修改 `/etc` 或挂载磁盘不影响宿主机            |
| **PID**           | 进程 ID 编号           | 容器内进程 PID 从 1 开始，看不到宿主机其他进程         |
| **Network** (net) | 网络设备、IP、端口、路由表     | 容器拥有虚拟网卡和独立 IP，端口冲突不会影响宿主机          |
| **UTS**           | 主机名和域名             | 容器可以有自己的 hostname（如 `web-server-1`） |
| **IPC**           | 进程间通信资源（信号量、共享内存等） | 隔离不同容器间的 IPC 通信                     |
| **User**          | 用户和用户组 ID          | 容器内的 root 可映射为宿主机上的普通用户，降低权限风险      |
| **Cgroup**        | Cgroup 控制组的视图      | 让进程只能看到自己的 Cgroup 路径（较新内核支持）        |
| **Time**          | 系统时间               | 不同容器可看到不同的系统时间（较新内核支持）              |
关键原理：对一个进程（如 clone() 创建的子进程）设置某个 Namespace 的 flag（如 CLONE_NEWPID），该进程及其子进程将会看到被隔离后的新资源视图。

#### 检查内核编译配置
```bash
cat /boot/config-$(uname -r) | grep -E 'CONFIG_NAMESPACES|CONFIG_CGROUPS'
```
- 使用lsns来查案当前系统中活跃的Namespace
```bash
lsns
```
#### 使用unshare创建命名空间
```bash
# 1. 创建各种 namespace
sudo unshare --pid --fork --mount --net --uts --ipc bash


```
## Cgroup(Control Group 控制组)
限制记录隔离进程组中使用的系统资源

如果没有 Cgroup，一个容器里的恶意或出错的进程可能会耗尽宿主机的所有内存或 CPU，导致整个系统崩溃。Cgroup 正是为了防止这种情况

主要功能：
- 资源控制：设置硬上限
- 优先级控制： 分配资源权重，比如AB容器的CPU时间权重
- 统计记录：监控资源使用情况
- 进程控制： 暂停，恢复，杀死一组进程

Cgroup版本
- v1：每种资源控制器独立挂载（如 cpu、memory、blkio）。ip
- v2：统一层级结构，所有控制器挂载在同一棵树上，功能更一致且安全。

每个进程都“属于”某个 Cgroup，但这不代表每个进程都“受限于”资源配额。
```bash
root@ASUS-HE:/sys/fs/cgroup# cat /proc/$$/cgroup
0::/user.slice/user-1000.slice/user@1000.service/app.slice/app-org.gnome.Terminal.slice/vte-spawn-a5d09db5-b9ae-41c9-9c35-7f48c75624d4.scope
```
#### 查看挂载点
```bash
mount | grep cgroup
```
#### 创建自己的Cgroup并设置限额
 ```bash
 # 1. 创建 cgroup 目录
 sudo mkdir /sys/fs/cgroup/my-container
 
 # 2. 设置内存限制 (100MB)
 echo 104857600 | sudo tee /sys/fs/cgroup/my-container/memory.max
 
 # 3. 设置 CPU 限制 (0.2 核心)
 echo "20000 100000" | sudo tee /sys/fs/cgroup/my-container/cpu.max
 
 # 4. 将当前 shell 加入 cgroup
 echo $$ | sudo tee /sys/fs/cgroup/my-container/cgroup.procs
 
 # 5. 测试内存限制
 stress --vm 1 --vm-bytes 200m  # 应该被 OOM kill
 
 # 6. 清理
 sudo rmdir /sys/fs/cgroup/my-container
 ```
 cgroup 虚拟文件系统（也叫 cgroupfs），它不存在于硬盘上，而是由内核动态生成在内存中。
 重启之后会消失，想要持久化的方法：
 1. 使用systemd单元文件
```bash
# /etc/systemd/system/mycontainer.service
[Service]
ExecStart=/your/command
MemoryMax=10M
CPUQuota=20%

[Install]
WantedBy=multi-user.target
```
2. 使用docker/k8s
## Namespace与Cgroup配合工作
1. 创建新进程：
调用`clone()`并且开启一些列namespace flag
此时，进程已经有了"被隔离的视野"
2. 加入Cgroup: 将这个新进程的PID加入到某个Cgroup控制组中，并设置相应的限制值
3. 切换根目录： 执行 pivot_root 或 chroot 将进程根目录改为容器镜像提供的文件系统（配合 Mount namespace）
最终效果：该进程及其子进程 看到的是独立的资源视图（Namespace 效果），但 最多只能使用被允许的资源量（Cgroup 效果）。


