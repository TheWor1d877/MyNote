当你在 Linux 上访问一个 IP（比如 10.96.0.1:80），数据包会经过内核的 Netfilter 框架。
Netfilter 是 Linux 内核中处理网络包的钩子系统（hook system），它在关键路径上设置了检查点：
iptables IPVS都是在这些hook点上面工作的工具
## iptables 通用型网络包处理框架
一个基于规则链的包过滤/NAT框架，是Netfilter的用户空间配置工具
提供表（Table）→ 链（Chain）→ 规则（Rule） 的层级结构
- 四张表
ptables 的“表”不是数据结构，而是人为的功能分组。

| 表（Table） | 作用                | 常见链（Chain）              |
| -------- | ----------------- | ----------------------- |
| `filter` | 包过滤（ACCEPT/DROP）  | INPUT, FORWARD, OUTPUT  |
| `nat`    | 网络地址转换（DNAT/SNAT） | PREROUTING, POSTROUTING |
| `mangle` | 修改包头（TTL, TOS）    | 所有 hook 点               |
| `raw`    | 跳过连接跟踪            | PREROUTING, OUTPUT      |
- 五条链

| 链 | 触发时机 | 典型用途 |
|----|----------|----------|
| `PREROUTING` | 包刚进入内核，路由决策前 | DNAT（修改目的地址） |
| `INPUT` | 包 destined to 本机进程 | 防火墙过滤 |
| `FORWARD` | 包需要转发到其他主机 | 路由器/网关过滤 |
| `OUTPUT` | 本机进程发出的包 | 出站过滤 |
| `POSTROUTING` | 包即将离开内核 | SNAT（修改源地址） |

![[Attachments/Pasted image 20260617131101.png]]
- 动作

| 作用                 |
| ------------------ |
| 允许包通过              |
| 静默丢弃               |
| 丢弃并发送错误包           |
| 修改目的地址（需在 `nat` 表） |
| 修改源地址（需在 `nat` 表）  |
| 动态 SNAT（用于动态 IP）   |
| 跳转到用户定义链           |
|                    |

缺点:
使用链表存储，规则数量大，遍历需要的时间长
1000 个 Service × 10+ 个 Pod = 10,000+ 条规则
每次新建连接都要遍历数千条规则 → CPU 飙升，延迟增加

#### k8s中默认使用iptables
```bash
howardhe@ASUS-HE:~$ kubectl get pods -n kube-system | grep kube-proxy
kube-proxy-5b7pm                   1/1     Running   4 (3m41s ago)    20h
howardhe@ASUS-HE:~$ kubectl logs -n kube-system kube-proxy-5b7pm | grep -i "using"
I0617 04:59:59.705926       1 server_linux.go:53] "Using iptables proxy"
E0617 04:59:59.910285       1 server.go:255] "Kube-proxy configuration may be incomplete or incorrect" err="nodePortAddresses is unset; NodePort connections will be accepted on all local IPs. Consider using `--nodeport-addresses primary`"
I0617 04:59:59.930147       1 server_linux.go:136] "Using iptables Proxier"
```

#### k8s如何使用iptables实现Service？
- 核心机制：DNAT + 负载均衡
ClusterIP 是虚拟 IP，无真实网卡绑定,内核默认舍弃,使用iptables才能给予网络包DNAT的机会
kube-proxy 在 nat 表生成规则，将 ClusterIP 流量 DNAT 到后端 Pod
- 规则生成逻辑（以 Service my-svc:80 → Pods 为例）
▶ 步骤 1：创建服务链
```bash
# 总入口链（所有访问 10.96.0.1:80 的流量进入）
# 所有发往 10.96.0.1:80 的 TCP 流量，交给 KUBE-SVC-XXXX 链处理
-A PREROUTING -d 10.96.0.1/32 -p tcp --dport 80 -j KUBE-SVC-XXXX
```
▶ 步骤 2：负载均衡（随机选择后端）
```bash
# 50% 概率选第一个 Pod
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.5 -j KUBE-SEP-YYYY

# 剩余流量选第二个 Pod
-A KUBE-SVC-XXXX -j KUBE-SEP-ZZZZ
```
▶ 步骤 3：DNAT 到具体 Pod
```bash
-A KUBE-SEP-YYYY -d 10.244.1.5/32 -p tcp --dport 8080 -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-ZZZZ -d 10.244.2.3/32 -p tcp --dport 8080 -j DNAT --to-destination 10.244.2.3:8080
```


## IPVS
Linux内核实现四层负载均衡
专为负载均衡设计的内核模块（属于 LVS 项目）
拥有高性能 + 丰富的的调度算法
使用虚拟服务表作为底层结构
- 虚拟服务表
哈希表结构
k8s可以使用虚拟服务表实现Service

#### 工作模式
| 模式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| DR（Direct Routing） | 修改 MAC 地址，RealServer 直接回包 | 性能最高 | 需要同网段 |
| TUN（IP Tunneling） | 封装 IP-in-IP 包 | 跨网段 | 额外封装开销 |
| NAT（Network Address Translation） | 修改目的 IP，回包经 LB | 简单通用 | LB 成为瓶颈 |

