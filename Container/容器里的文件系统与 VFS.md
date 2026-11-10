容器启动时，runc 在容器自己的 mount namespace 里搭一棵挂载树。这棵树的根是镜像的 rootfs，常见形态是 overlayfs。overlayfs 把镜像的只读层叠起来，上面加一个可写层。容器里的 `/etc/os-release` 读的是镜像里这一份文件，宿主上的 `/etc/os-release` 容器看不到。容器根来自镜像，这个事实不因 hostPath 改变。

VFS 是内核里的文件系统抽象层。它给上层系统调用一个统一的文件模型，给下层 ext4、overlayfs、proc、tmpfs 这些具体文件系统一套统一的接入接口。VFS 不负责数据在磁盘上怎么摆，那是 ext4 的事；ext4 不负责路径怎么解析、挂载点怎么跳转，那是 VFS 的事。

## VFS 管的四类对象

`super_block` 表示一个已挂载的文件系统实例，对应一个分区或一个文件系统。`inode` 表示一个文件或目录的元数据，内核按 `(super_block, inode 号)` 缓存。`dentry` 表示一个名字到 inode 的映射，组成内存里的目录树，加速路径解析。`file` 表示一个进程打开的文件，每次 `open` 产生一个，带偏移和打开标志。

具体文件系统把自己的操作注册到这几类对象上。`super_operations` 管分配和销毁 inode、同步、统计。`inode_operations` 管 lookup、create、mkdir、权限检查。`file_operations` 管 open、read、write、mmap、release。`address_space_operations` 管读写页和脏页回写。VFS 收到系统调用后查对象上的操作表，调用具体文件系统的函数。

## 路径怎么变成一个 inode

解析 `/a/b/c` 由 VFS 统一做。VFS 从根目录或当前工作目录开始，逐级查 dentry cache。未命中时调具体文件系统的 lookup。ext4 的 lookup 读目录数据块，目录项里存的是名字到 inode 号的映射，找到 `a` 的 inode 号后，VFS 用这个号构造或命中 VFS inode。路径继续走到 `b`、`c`，最后拿到目标 inode。目录大了 ext4 用 htree 索引加速查找，语义不变。

inode 号是一个文件系统内全局唯一的整数。ext4 按块组存放 inode，superblock 里记 `s_inodes_per_group` 和 `s_inode_size`，组描述符表里记每个块组 inode 表的起始块号 `bg_inode_table`。给一个 inode 号，先算它在哪个块组、组内第几个，再算磁盘字节偏移：`inode_table_block * block_size + index * inode_size`。内核读这个偏移处的 `ext4_inode` 结构体，拿到元数据。

数据块的位置在 inode 的 `i_block[15]` 里。ext4 默认启用 extent，这 60 字节放一个 extent 树头，小文件直接放最多 4 个 extent，每个 extent 给出逻辑块号、长度和物理块号。大文件的树头里 `eh_depth` 大于 0，inode 里放索引节点，指向数据块里的下一层，叶子节点才是逻辑块到物理块的映射。读文件偏移 `off` 时，逻辑块号是 `off / block_size`，在 extent 树里查到物理块号，再乘块大小加块内偏移，得到磁盘位置。

## 三层树

ext4 内部有目录树。目录也是文件，数据块里存目录项数组，每项是名字到 inode 号的映射，目录项层层嵌套。这棵树在单个文件系统内部。

VFS 有 dentry 树。内核把路径名到 inode 的映射缓存在内存里，路径解析沿 dentry 树走。这棵树还是单文件系统视角，负责加速查找。

VFS 还有 mount 树。一个 mount namespace 里有一棵挂载树，每个挂载点是一个节点，节点下面接着某个文件系统的根。路径解析走到挂载点时，VFS 跳到那个文件系统的根继续走。bind mount 是 mount 树这一层的操作：把一棵子树挂到另一个挂载点上。

hostPath 做的事在 mount 树这一层。kubelet 在容器的 mount namespace 里加一个挂载点，这个挂载点接的是宿主的一个目录。inode 没有变，还是宿主那批 inode；变的是容器的 mount 树里多了一个节点，这个节点下面接的是宿主子树。

## 容器视角下什么隔离、什么共享

mount 树按 mount namespace 隔离。容器有自己的 mount namespace，就有自己的挂载树视图。容器看到的 `/` 是它自己的挂载树根，宿主看到的 `/` 是宿主挂载树根。两个 namespace 的编号不同，这是两棵独立的挂载树。

inode cache、dentry cache、page cache 按 super_block 共享，不按 namespace 隔离。同一个 ext4 文件系统被容器和宿主同时访问时，同一个 `(super_block, inode 号)` 在内核里只有一个 VFS inode，同一份 page cache。hostPath 挂进来后，容器 `/mnt/blocks/foo` 和宿主 `/var/cache/traincache/foo` 指向同一个 VFS inode。容器写这个文件，写的是这个 inode 的页，宿主读同一个 inode，读到的就是改后的内容。

容器根通常是 overlayfs，overlayfs 本身也是注册到 VFS 的文件系统。容器里读一个文件可能穿过两层 VFS 分发：系统调用进 VFS，VFS 调 overlayfs，overlayfs 再调下层文件系统。不管几层，路径解析和挂载点跳转始终由 VFS 统一做。

## `..` 为什么回不到宿主

路径解析按当前 mount 树走。`/mnt/blocks` 是 bind mount 的挂载点根。VFS 解析 `..` 时，从挂载点根回到挂载点的父目录，这个父目录属于容器的挂载树，是容器的 `/mnt`。再走一次 `..`，到容器的 `/`。宿主上 `/var/cache` 这个父目录不在容器的挂载树里，所以 `..` 走不到。

共享的是 inode，不是祖先路径。同一个文件在容器里叫 `/mnt/blocks/foo`，在宿主上叫 `/var/cache/traincache/foo`，它的文件身份共享，路径祖先按各自挂载树走。

## 一句话收束

VFS 是内核的统一文件抽象层，管对象模型、路径解析、mount 树、缓存、权限和操作分发。ext4 是它下面的具体实现，通过操作表注册进来，负责 inode 定位、extent 映射和页读写。容器有自己的 mount namespace 和自己的挂载树，根是镜像 rootfs；inode cache 和 page cache 按 super_block 共享。hostPath 在容器的挂载树里 graft 一棵宿主子树，共享 inode，不共享 mount namespace，也不共享祖先路径。


## 面试
#### hostPath 把宿主目录直接塞进 Pod，首先破坏的是哪一层隔离？复现的 case 是什么？

hostPath 破坏的是**文件系统可见性**这一层，落在 VFS 的 mount 树上。kubelet 在容器的 mount namespace 里加一个挂载点，bind mount 宿主的一个目录。容器在这个挂载点下面读写的，就是宿主那批 inode。

容器有自己独立的 mount namespace，根是镜像 rootfs。hostPath 没让容器和宿主共享 mount namespace，它只是在容器挂载树里接了一棵宿主子树。共享的是 inode，不是 namespace。

容器里从 `/mnt/blocks` 往上 `..`，回的是容器的 `/mnt`，再到容器的 `/`。宿主上那个目录的父目录不在容器挂载树里，`..` 爬不出去。inode 共享，祖先路径不共享。

PID、IPC、UTS、user、network 这几个 namespace 不受 hostPath 影响。容器里 `ps` 只看到自己的进程，`hostname` 是 Pod 名。要共享进程、IPC、网络，得开 hostPID、hostIPC、hostNetwork，跟 hostPath 无关。

复现三档，按机制推出来的现象：

- **case A 可见性穿透**：挂 hostPath 的 Pod 在容器里 `cat /mnt/blocks/foo` 读出宿主文件；同节点不挂 hostPath 的 Pod 读同一路径报 `No such file or directory`。差别来自 volume，不是运行时给的。
- **case B 可写落盘**：容器里 `echo hello > /mnt/blocks/bar`，宿主 `ls` 能看到 `bar`。卷设 `readOnly: true` 再写，报 `Read-only file system`。
- **case C 跨 Pod 篡改**：Pod X 可写挂同一个目录，改掉 blocks 里的字节；训练 Pod 只读挂同一目录，读到改后的内容。训练 Pod 的 `readOnly` 只约束自己的挂载点，管不到 Pod X。

#### 这个 case 对 TrainCache 的挂载模型意味着什么？为什么仍然接受？CSI 变了什么、没变什么？为什么 CSI 不等于自动安全？

这个 case 意味着**节点级的读写可见性**：同节点任意 Pod 只要挂得上同一个缓存目录，就能读写缓存块。

TrainCache 接受这个代价，因为训练读数据走 hostPath，读路径不经过 Agent、不经过网关，DataLoader 直接吃 page cache。这是 preheat 模型的前提，也是 TrainCache 全部实验结论的前提。

兜底不在挂载层，在数据层：块按内容寻址，带 xxh3 校验，字节被改后校验不过，进 `quarantine/` 走修复。挂载层给的是「同节点互相可见」，数据层给的是「被改的字节进不了训练」。TrainCache 面向同集群内可信的自家训练任务；多租户跨租户读缓存目录这条，TrainCache 明确不接受。

CSI 换了**控制面**：Pod 不再写宿主路径，改成申领 PVC，卷的创建、回收、生命周期、访问控制归 StorageClass 和 CSI 驱动管。谁能指哪条宿主路径，从 Pod 作者收归到存储系统和集群管理员。

CSI 没换**数据面**：Node 侧驱动把存储挂到 kubelet 给的 target path，kubelet 再 bind mount 进容器 mount namespace。容器里看到的还是一个挂载点接一棵子树，共享 inode，不共享 mount namespace。驱动自己还在节点上特权运行，JuiceFS 这种还得常驻一个 FUSE 进程。

CSI 不等于自动安全：它收窄了「谁能指哪条宿主路径」，没收窄「容器能看到宿主什么」。卷挂进来后，容器对这个挂载点下就是可读可写。同一个 PV 被多个 Pod 挂，互相可见。多租户还得配 StorageClass 准入、卷加密、按租户隔离的存储后端。

对 TrainCache 来说，换 JuiceFS CSI 读路径多一层 FUSE，零额外延迟这个核心收益被削掉，节点上还多一个特权 FUSE 进程。TrainCache 选 hostPath，是选了读路径直接读 page cache，把校验放在 xxh3 和 quarantine 上。