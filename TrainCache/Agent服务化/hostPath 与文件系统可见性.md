hostPath 是 Kubernetes 的一种卷类型。Pod 的 YAML 里写一个宿主路径，kubelet 把这个宿主目录挂进容器。挂载动作是一次 bind mount：同一批 inode 在容器的挂载点上再出现一次。容器读写这个挂载点，读写的是宿主磁盘上的同一份数据，没有复制。

bind mount 是 VFS 挂载树上的操作。每个进程属于一个 mount namespace，每个 mount namespace 有一棵挂载树，树的节点是挂载点，挂载点下面接着某个文件系统的根。bind mount 在容器的挂载树里加一个节点，这个节点接的是宿主的一个目录。容器在这个节点下面能看到宿主那棵子树，看不到宿主的其他部分。

## 破坏的是文件系统可见性

hostPath 打开的是文件系统可见性这一层。被点名的宿主目录，容器内能直接看到并读写。容器和宿主各自有独立的 mount namespace，容器运行时启动容器时创建容器的 mount namespace，kubelet 在这个 namespace 里加挂载点。两个 namespace 的编号不同，这是两个独立的挂载树视图。

mount namespace 没有被共享。共享的是 inode。容器 `/mnt/blocks/foo` 和宿主 `/var/cache/traincache/foo` 指向同一个 VFS inode，因为 bind mount 把宿主那棵子树的 inode 接进了容器的挂载树。VFS 的 inode cache 按 super_block 组织，同一个文件系统里的同一个 inode 号在内核里只有一个 VFS inode。容器写这个 inode，宿主读这个 inode，读写落在同一份 page cache 上。

在容器里从挂载点向上走，回不到宿主。`/mnt/blocks` 是 bind mount 的挂载点根。VFS 解析 `..` 时，从挂载点根回到挂载点的父目录，这个父目录属于容器的挂载树，是容器的 `/mnt`。再走一次 `..`，到容器的 `/`。宿主上 `/var/cache` 这个父目录不在容器的挂载树里，所以 `..` 走不到。inode 共享，祖先路径不共享。

## 哪些 namespace 没动

PID、IPC、UTS、network 这几个 namespace 由各自的开关决定，hostPath 不改变它们。容器里的 `ps` 只列出容器自己的进程，`hostname` 是 Pod 名。要让容器和宿主共享进程、IPC 或网络，得打开 hostPID、hostIPC、hostNetwork，这三个开关和 hostPath 无关。user namespace 也不被 hostPath 改变。kind 环境里容器和宿主的 user namespace 编号相同，这是 Docker 默认没开 user namespace 映射的结果，这个环境事实不进入 hostPath 的结论。

## 只读挂载约束谁

`readOnly: true` 是 mount 层的一个标志，约束这个挂载点上的写入。容器往这个挂载点写，VFS 返回 `Read-only file system`。同一个宿主目录被另一个 Pod 以可写方式挂载时，那个 Pod 的写入不受这个只读标志约束。只读标志属于挂载点，不属于目录，也不属于 inode。

## 同节点跨 Pod 可见的逻辑

同一个节点上，Pod X 以可写方式挂 `/var/cache/traincache`，训练 Pod 以只读方式挂同一个目录。Pod X 改掉 blocks 里某个文件的字节，训练 Pod 读这个文件，读到的是改后的内容。训练 Pod 的只读标志约束训练 Pod 自己的挂载点，管不到 Pod X 的挂载点。两个 Pod 的挂载点接的是同一批 inode，宿主上 `cat` 那个文件，字节已经变了。这是 hostPath 在节点级的读写可见性：同节点任意 Pod 只要挂得上同一个目录，就能读写缓存块。

## TrainCache 接受这个代价

TrainCache 的训练读数据走 hostPath。读路径不经过 Agent，不经过网关，DataLoader 直接读 page cache。这是 preheat 模型的前提，也是 TrainCache 全部实验结论的前提。

代价是同节点 Pod 之间对缓存目录的读写可见性。兜底放在数据层：块按内容寻址，带 xxh3 校验，字节被改后校验不过，进 `quarantine/` 走修复。挂载层给出的是同节点互相可见，数据层给出的是被改的字节进不了训练。TrainCache 面向同集群内可信的自家训练任务。多租户共享集群里跨租户读缓存目录这个风险，TrainCache  明确不接受。

## CSI 换了什么、没换什么

CSI 换掉的是控制面。Pod 不再写宿主路径，改成申领 PVC。卷的创建、回收、生命周期由 StorageClass 和 CSI 驱动的 Controller 侧管，访问控制可以在 RBAC 和准入控制上做。

CSI 没换数据面。Node 侧的 CSI 驱动把存储挂到 kubelet 给的 target path，kubelet 再把 target path bind mount 进容器的 mount namespace。容器里看到的仍然是一个挂载点接一棵子树，仍然共享 inode，仍然不共享 mount namespace。CSI 驱动自己在节点上以特权运行，JuiceFS 这种还要常驻一个 FUSE 进程。

CSI 收窄了“谁能指哪条宿主路径”，没收窄“容器能看到宿主什么”。多租户场景还要配 StorageClass 准入、卷加密、按租户隔离的存储后端。TrainCache 如果换成 JuiceFS CSI，读路径多一层 FUSE，内核态和用户态之间来回切换，零额外延迟这个前提被打破，节点上还多一个特权进程。TrainCache 选 hostPath，是选了读路径直接读 page cache，把校验放在 xxh3 和 quarantine 上。

## 一句话结论

hostPath 没有共享 mount namespace。它拆掉的是文件系统可见性这一层：容器有自己的挂载树，宿主的一棵子树被 bind mount 进来，共享 inode，不共享祖先路径。