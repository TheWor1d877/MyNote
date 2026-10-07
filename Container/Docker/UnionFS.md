Union File System（联合文件系统） 是一种特殊的文件系统技术，它的核心能力是：
将多个目录合并成一个统一的文件系统构图
关键特性：
- 分层合并： 多个只读层 + 一个可写层 = 完整的文件系统
- 写时复制： 修改文件的时候才复制到可写层
- 层优先级： 上层文件会覆盖下层文件

## Docker 镜像的分层结构
镜像构建过程：
```go
Layer 4: [nginx config files]     ← 最上层（最新）
Layer 3: [nginx binary] 
Layer 2: [ubuntu packages]
Layer 1: [ubuntu base system]     ← 最底层（最老）
```
每个layer都是一个只读的文件系统快照

- 使用history来查看分层结构
```bash
howardhe@ASUS-HE:~$ docker history nginx
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
7aaca76c508f   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY 30-tune-worker-processes.sh /docker-ent…   4.62kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY 20-envsubst-on-templates.sh /docker-ent…   3.02kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY 15-local-resolvers.envsh /docker-entryp…   389B      buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY 10-listen-on-ipv6-by-default.sh /docker…   2.12kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY docker-entrypoint.sh / # buildkit          1.62kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c set -x     && groupadd --syst…   82.7MB    buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV DYNPKG_RELEASE=1~trixie                     0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV PKG_RELEASE=1~trixie                        0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV ACME_VERSION=0.4.1                          0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV NJS_RELEASE=1~trixie                        0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV NJS_VERSION=0.9.9                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENV NGINX_VERSION=1.31.1                        0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   LABEL maintainer=NGINX Docker Maintainers <d…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   # debian.sh --arch 'amd64' out/ 'trixie' '@1…   78.6MB    debuerreotype 0.17

```
## OverlayFS 工作原理
OverlayFS (文件系统)：这是Linux内核自带的一种先进的文件系统技术。它的神奇之处在于，能让你把“甲目录”和“乙目录”叠加在一起，形成一个“丙目录”。如果甲、乙有相同的文件，在丙里只会看到乙（上层）的文件，它“覆盖”了甲（下层）的文件。这就是容器“镜像层只读，容器层可写”的原理。
overlayfs 是一个“傀儡大师”，它自己没有任何能力直接读写硬盘数据。
它所有的操作都必须依赖于一个底层的、真正的文件系统（如 ext4、xfs）。
它就像一个中间层，所有的“lowerdir”（只读下层）和“upperdir”（可写上层）目录，都必须存放在 ext4 这样的文件系统上。
```text
+-------------------------------------+
|           你的应用程序                |
+-------------------------------------+
|          Docker / Podman            |  ← 用户空间软件
+-------------------------------------+
|       glibc (系统调用接口)            |  ← 标准C库
+-------------------------------------+
|          Linux 内核                 |
|  +-----------------------------+   |
|  | 进程管理 | 内存管理 | 网络栈  |   |
|  +-----------------------------+   |
|  |        VFS (虚拟文件系统层)      |   ← 内核中的抽象层
|  +-----------------------------+   |
|  |  ext4  |  xfs  |  btrfs  |     |
|  |        OverlayFS ← 就是这里！    |   ← 内核模块
|  +-----------------------------+   |
|           硬盘驱动程序               |
+-------------------------------------+
|              物理硬盘                |
+-------------------------------------+
```

现代Docker使用Overlay2存储驱动
目录结构
```text
/var/lib/docker/overlay2/
├── <layer-id>/
│   ├── diff/          ← 这一层的实际文件内容
│   └── link           ← 短链接标识
├── <container-id>/
│   ├── diff/          ← 容器的可写层
│   ├── merged/        ← 联合挂载点（容器看到的完整文件系统）
│   ├── lower/         ← 指向所有只读层的链接
│   └── upper/         ← 实际指向 diff/（可写层）
└── l/                 ← 短链接目录
```
#### overlay2受欢迎原因
旧版的 overlay 驱动有一个致命的缺陷，就是在长时间运行后容易耗尽文件系统的 Inode（可以理解为文件索引数量），导致无法创建新文件。

## UnionFS的写时复制机制
场景演示：
假设镜像中有文件 /app/config.txt，现在容器要修改它：

步骤 1：读取文件
- 容器进程请求读取 /app/config.txt
- OverlayFS 在 upperdir 中查找 → 不存在
- 继续在 lowerdir 中查找 → 在 layer2 中找到
- 直接返回 layer2 中的文件（零拷贝！）

步骤 2：修改文件
- 容器进程要修改 /app/config.txt
- OverlayFS 发现文件在只读层
- 自动将文件从 layer2 复制到 upperdir
- 然后修改 upperdir 中的副本

步骤3：删除文件
OverlayFS 不会真正删除底层文件
   而是在 upperdir 中创建一个特殊的 whiteout 文件
   文件名格式：`.wh.<original-filename>`
   
结果：
- 原始镜像层完全不受影响
- 只有被修改的文件才会占用额外空间
- 其他容器仍然使用原始的只读文件

## 优缺点
- 快速启动
容器启动不需要复制整个文件系统
只需要创建轻量级的可写层
启动时间从秒级降到毫秒级

- 每个镜像层都有唯一 ID
可以精确追踪文件系统变化
支持高效的镜像 diff 和 rollback



- 层数过多问题
Dockerfile 指令越多，层数越多
层数过多会影响性能（路径查找变慢）
解决方案：合理合并 RUN 指令.

- 大文件修改代价高
修改大文件会触发完整的 Copy-on-Write
解决方案：避免在容器中修改大文件，使用 volume 挂载