## 基础模型
Linux网络中的两种内容:设备与转发逻辑
#### 设备（device）
比如：
eth0：你的物理网卡
docker0：Docker 创建的虚拟网卡
vethXXX：容器用的虚拟网线

每个设备都有：
一个名字
一个 MAC 地址
一个 qdisc（队列规则，tc 就是操作它的）
一个 ndo_start_xmit 函数（真正把数据发出去的函数）

#### 转发逻辑
路由：根据 IP 地址决定包走哪个设备
bridge：根据 MAC 地址决定帧走哪个端口
转发逻辑不是设备，它是内核里的一段代码。

#### docker网络结构
```text
┌─────────────────────────────────────────────────────┐
│ host（宿主机）                                        │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │ docker0（一个 bridge 设备）                   │   │
│   │ IP: 172.17.0.1                              │   │
│   └──┬──────────────┬──────────────┬────────────┘   │
│      │              │              │                │
│   vethA_host    vethB_host    vethC_host            │
│      │              │              │                │
│   ┌──┴──┐        ┌──┴──┐        ┌──┴──┐            │
│   │容器A │        │容器B │        │容器C │            │
│   │eth0  │        │eth0  │        │eth0  │            │
│   │.0.2  │        │.0.3  │        │.0.4  │            │
│   └─────┘        └─────┘        └─────┘            │
└─────────────────────────────────────────────────────┘
```
docker0既是一个设备（有 qdisc，有 ndo_start_xmit）,又是一个 bridge（有 FDB，有转发逻辑）
这两副面孔，对应两条完全不同的内核路径。
数据走哪副面孔，取决于数据从哪来、到哪去。

## 三种场景
注意: ingress qdisc 在 Linux 里默认是不存在的。你不主动挂，它就不存在，自然不会触发。
   
#### host → 容器（host 主动发包给容器）
比如：host 上 ping 172.17.0.2
```text
第1跳：host 进程产生 ICMP 包
第2跳：host IP 栈查路由表，发现 172.17.0.2 走 docker0
第3跳：host IP 栈调用 dev_queue_xmit(docker0),先把数据包发给docker0
第4跳：docker0 的 egress qdisc 被触发  ← 这里！
第5跳：docker0 的 ndo_start_xmit（br_dev_xmit）被调用
第6跳：进入 bridge 转发逻辑，查 FDB
第7跳：找到 vethA_host 端口
第8跳：调用 dev_queue_xmit(vethA_host)
第9跳：vethA_host 的 egress qdisc 被触发
第10跳：vethA_host 的 ndo_start_xmit
第11跳：vethA_container 收到
第12跳：容器 A 内 eth0 收到
```
dev_queue_xmit：内核协议栈的通用发送入口，属于网络核心层
ndo_start_xmit：网卡驱动的实际发送的位置，属于设备驱动层
二者之间经过docker0的egress

#### 容器 → host
比如：容器内 nginx 发数据给 host 进程
```text
第1跳：容器内 nginx 产生 HTTP 响应
第2跳：容器内 IP 栈查路由，发现目标是 host（172.17.0.1 或外部）
第3跳：容器内 IP 栈调用 dev_queue_xmit(容器eth0)
第4跳：容器内 eth0 的 egress qdisc 被触发  ← 你挂这里 ✅
第5跳：容器内 eth0 的 ndo_start_xmit
第6跳：vethA_container 收到（veth pair 的另一端）
第7跳：vethA_host 收到
第8跳：vethA_host 发现自己是 bridge 的端口
第9跳：调用 bridge 的接收函数 br_handle_frame  ← 注意！不是 dev_queue_xmit(docker0)
第10跳：br_handle_frame 查 FDB
第11跳：发现目标是 host 自己（不是另一个 veth 端口）
第12跳：调用 br_pass_frame_up
第13跳：把包交给 host 网络栈
第14跳：host 进程收到
```
因为数据是从 vethA_host “进来”的，bridge 接收到它，然后决定交给 host。整个过程没有“从 docker0 设备发出去”这个动作。
所以 docker0 的 egress qdisc 没有被触发

#### 容器→ 容器
比如：容器 A 发数据给容器 B
```text
第1跳：容器 A 内进程产生数据
第2跳：容器 A 内 IP 栈调用 dev_queue_xmit(容器A eth0)
第3跳：容器 A eth0 的 egress qdisc 被触发
第4跳：容器 A eth0 的 ndo_start_xmit
第5跳：vethA_container 收到
第6跳：vethA_host 收到
第7跳：vethA_host 发现自己是 bridge 端口
第8跳：调用 br_handle_frame
第9跳：查 FDB，发现目标是容器 B，对应 vethB_host
第10跳：调用 br_forward
第11跳：调用 dev_queue_xmit(vethB_host)  ← 注意！是 vethB_host
第12跳：vethB_host 的 egress qdisc 被触发  ← 限速挂这里
第13跳：vethB_host 的 ndo_start_xmit
第14跳：vethB_container 收到
第15跳：容器 B eth0 收到
```

## 规律
容器都能通过bridge直接跟host连接
但是host不认识具体容器,只能先跟docker0打交道
