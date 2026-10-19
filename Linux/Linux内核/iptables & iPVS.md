#面试-云原生-复习  解释 kube-proxy 的 iptables 模式与 IPVS 模式差异，以及高并发推理网关下如何选型

当你在 Linux 上访问一个 IP（比如 10.96.0.1:80），数据包会经过内核的 Netfilter 框架。
Netfilter 是 Linux 内核中处理网络包的钩子系统（hook system），它在关键路径上设置了检查点：
iptables IPVS都是在这些hook点上面工作的工具
## iptables 通用型网络包处理框架
一个基于规则链的包过滤/NAT框架，是Netfilter的用户空间配置工具
表： 提供不同的功能，每个表都能提供不同的动作，可以将动作放到表上面
链： 提供相关的钩子
动作： 匹配之后的行为


![[Attachments/Pasted image 20260617131101.png]]

缺点:
使用链表存储Service（规则链条的头节点），规则数量大，遍历需要的时间长
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

#### 时间复杂度
kube-proxy 为每个 Service 生成一堆 iptables 规则。数据包要从上到下逐条匹配规则，规则越多越慢，复杂度大致是 O(n)

## IPVS
Linux内核实现四层负载均衡器，工作在TCP/UDP上面，能够将发送往某个port:ip的请求按照某种算法转发到后端的一组真实服务器上面
它支持多种调度算法，iptables只能随机选择Pod
#### k8s上面的运行方式
在k8s中kube-proxy会在节点上面创建一个叫作kube-ipvs0的dummy网卡，把Service的Cluster IP绑定上去，然后在这个网卡中设置转发规则

数据包发送过程：
物理网卡->网络栈（发现目标IP在看kube-ipvs0上面）->通过网络发送给这个端口->进入 IPVS 模块做DNAT，修改目标地址->包重新进入网络栈，作为“转发包”再走一遍路由和转发

- 性能问题：
查询Service的相关信息走的是哈希表，O(1)级别的时间复杂度
但是O(1)的那个常数项非常大，IPVS需要中间涉及虚拟接口的介入，要多走一次网络栈，而iptables是直接一个hook完成，规则匹配与DNAT都是在一起完成
所以IPVS在Service数量少的时候并不比iptables快
#### 工作模式
| 模式                               | 原理                        | 优点   | 缺点      |
| -------------------------------- | ------------------------- | ---- | ------- |
| DR（Direct Routing）               | 修改 MAC 地址，RealServer 直接回包 | 性能最高 | 需要同网段   |
| TUN（IP Tunneling）                | 封装 IP-in-IP 包             | 跨网段  | 额外封装开销  |
| NAT（Network Address Translation） | 修改目的 IP，回包经 LB            | 简单通用 | LB 成为瓶颈 |
|                                  |                           |      |         |
## nftables
nftables 模式。Kubernetes 1.33 之后 nftables 已经稳定，性能比 IPVS 更好，而且 IPVS 在 1.35 已经被 deprecated。新建集群、内核够新的话，直接上 nftables 是更合理的选择。
nftables 是 Linux 内核里新一代的包过滤和转发框架，用来替代 iptables

iptables使用链表作为查找处理连的方式，而nftables使用哈希表作为查找方式，key是目标 IP + 协议 + 端口

## 高并发推理网关下面的选择
Service数量不是很多，但是Pod副本多，连接长

kube-proxy 模式不应该是你选型的核心依据，推理网关的性能瓶颈通常在用户态转发层（Envoy/Nginx）和连接管理上，不是 kube-proxy 这层

如果集群规模不大（Service 几百个以内），iptables 完全够用，性能甚至略优于 IPVS。如果集群本身已经几千个 Service，或者你打算未来扩到很大规模，IPVS 的 O(1) 查找优势才体现出来。

推理网关如果是短连接模式，需要注意端口复用的bug：
后来 Pod A 被销毁了（比如滚动更新，旧副本被删）。
但 IPVS 表里那条旧记录还没被清理掉
客户端又用同样的四元组发起新连接
导致 IPVS 把新连接转发到一个已经销毁的 Pod 上