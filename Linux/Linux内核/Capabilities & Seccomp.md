## Capabilities
 传统的 Linux 进程只有两种身份：要么是拥有至高无上权力的根用户（root，权限 0），要么是啥也干不了的普通用户。就像一个要么是“大楼总管理员”，要么是“普通访客”，没有中间状态。
 
 为了运行一个需要绑定 80 端口（1024 以下端口只有 root 能开）的 Web 服务，过去你可能需要让整个容器进程都拥有 root 权限。这太危险了，就像一个租客要“开一扇窗户”，你却给了他“整栋楼的总钥匙”。
 
Capabilities 的解决方案：
Capabilities 把超级管理员（root）的超能力，拆分成了大约 40 个互不重叠的小权限。每个小权限就像一把只能开特定门的专用钥匙。
#### 使用
我们来查看当前 shell 进程拥有的权限。
```bash
# 查看当前shell进程的Capabilities（以十六进制显示）
grep Cap /proc/self/status

# 解码十六进制权限，查看人类可读的格式
capsh --decode=0000000000000000
```
查看可执行文件的 Capabilities
```bash
# 查看 /bin/ping 文件被赋予了哪些 Capabilities
getcap /bin/ping
```

- 输出内容解读
```text
/ # grep Cap /proc/self/status
CapInh:	0000000000000000
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
CapBnd:	00000000a80425fb
CapAmb:	0000000000000000
```
- Inh: 子进程能继承哪些权限
- Prm: 当前进程能使用的权限上限
- Eff: 当前已经激活的权限（权限用完要及时关闭激活状态）
- Bnd:系统允许的权限上限
- Amb:非特权进程能保留的权限: 子进程能获得的权限，Prm除了继承父进程还能拥有的
## Seccomp
即使有了 Capabilities，一个进程（即使是普通用户）仍然可以向 Linux 内核发起系统调用。系统调用数量庞大（Linux 有 300-400 个），其中很多是容器完全不需要的，比如：用于休眠硬件的 swapon、用于修改系统启动参数的 reboot、用于直接读写内存的 bpf 等。

允许所有系统调用，就像允许任何访客（哪怕只有大堂权限）使用大楼里的任何内部开关和操作面板，万一有人按下“紧急制动”（reboot 系统调用），整个宿主机就重启了。

Seccomp 的解决方案：
Seccomp (Secure Computing Mode) 是内核的一个系统调用过滤器。它就像一个在内核门口的严格安检员，检查每一个进程发出的系统调用。

#### 工作模式
白名单模式 (默认，最常用)： 只允许调用一个预先定义好的“安全名单”里的系统调用，其他的一律拒绝。
黑名单模式： 只拒绝调用一个“危险名单”里的系统调用，其他都允许。
   
Docker 自带了一个默认的 Seccomp 配置文件（default.json），里面大约列出了 300 多个允许的系统调用，而屏蔽了 40 多个危险的系统调用。
#### 使用seccomp
Seccomp 是一个内核级别的“防火墙”，它会检查并限制进程发起的每一个系统调用。我们可以非常直观地观察它的效果。
- 观察运行中的容器的Seccomp的状态
```bash
# 1. 启动一个普通的容器
docker run -d --name seccomp_test alpine tail -f /dev/null

# 2. 获取容器主进程的 PID
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' seccomp_test)
echo $CONTAINER_PID

# 3. 查看该进程的 Seccomp 状态
cat /proc/$CONTAINER_PID/status | grep Seccomp
```
- 使用自定义的Seccomp
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["mkdir", "mkdirat"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
```

```bash
# 使用这个自定义的 Seccomp 配置启动一个容器
docker run --rm -it --security-opt seccomp=./deny-mkdir.json alpine sh
```
## 为什么拥有了namespce还是需要Capabilities与Seccomp
Namespace对进程来说是100%的物理隔离，但是对于内核资源来说，他是一种幻觉
如：poweroff
poweroff是一个系统调用，  
系统调用是直接给内核下命令，就像你给国家电网打电话说“拉闸”。
Namespace只是让你看到一张虚拟的电表，它让你以为自己是这个小区的电闸管理员
namespace无法虚拟化poweroff，因为“电源”是不可分割的物理资源。

Namespace无法涵盖所有的内核资源，在namespace之外的资源，容器与物理机是共用的关系
