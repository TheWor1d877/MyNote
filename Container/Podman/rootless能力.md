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
# 启动一个容器并进入 shell
podman run -it --name test-rootless alpine sh

# 在容器内查看 UID
whoami    # 输出：root
id        # 输出：uid=0(root) gid=0(root)

# 退出容器
exit
```

## 验证Docker的root环境
Docker Daemon（守护进程）总是以root权限运行的，所有容器都是由这个root权限的 deamon创建 与管理
普通用户执行docker run 也是root权限的操作
1. 检查 Docker Daemon 进程
```bash
# 查看 dockerd 进程
ps aux | grep dockerd

# 或者查看完整进程树
pstree -p | grep dockerd
```
2. 检查 Docker Socket 权限
```bash
ls -l /var/run/docker.sock
```
3. 验证容器进程的实际权限
```bash
# 启动一个容器
docker run -d --name test-container nginx

# 查看容器进程在宿主机上的实际 UID
ps aux | grep nginx
```

由于dackerd的root权限，在许多方面需要更多的安全边界检查
每次api调用都要验证用户是否有更多的权限
