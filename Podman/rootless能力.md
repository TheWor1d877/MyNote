Rootless 容器 是指 普通用户（非 root）无需 sudo 权限即可创建、运行和管理容器 的能力。

==无需sudo==就能拉取镜像：
```bash
podman pull alpine

podman run --rm -p 8080  -v $(pwd):/data nginx

curl http://localhost:8080
```

## 如何实现: User Namespace
将宿主机器上面的普通用户uid映射成容器内部的root（UID0）
容器内认为自己才是root

## 查看uid/gid的映射范围
```bash
grep $(whoami) /etc/subuid

grep $(whoami) /etc/subgid
```
 输出示例：howardhe:100000:65536
 从uid10000开始连续分配65536个uid进行映射
 容器内 UID 0 → 宿主机 UID 100000
 容器内 UID 1 → 宿主机 UID 100001

## 验证容器进程的uid映射
```bash

```
## 验证Docker的root环境
Docker Daemon（守护进程）总是以root权限运行的，所有容器都是由这个root权限的 deamon创建 与管理
普通用户执行docker run 也是root权限的操作

