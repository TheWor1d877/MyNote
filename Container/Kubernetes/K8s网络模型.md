K8s 不实现网络，而是定义了 四个必须满足的网络规则（任何 CNI 插件都必须遵守）
- Rule 1: 所有 Pod 可以与所有其他 Pod 直接通信（无需 NAT）
-  Rule 2: 所有 Node 可以与所有 Pod 直接通信（无需 NAT）
- Rule 3: 每个 Pod 的 ip 地址在集群内是唯一的
- Rule 4: Pod 自己看到的 IP = 其他 Pod 看到的该 Pod 的 IP（无 SNAT/DNAT）

K8s 假设底层网络已打通，它只负责分配 IP 和策略
实际组网由 CNI 插件 实现（如 Flannel、Calico、Cilium）
这些规则使得 微服务可以直接用 IP 通信，无需关心底层拓扑K8s 要求“扁平网络”，简化应用开发

## CNI
CNI 是一个标准接口规范，定义了容器运行时（如 containerd）如何调用网络插件来配置 Pod 网络。

`/etc/cni/net.d/`	CNI 配置目录 —— kubelet 会读取这里的所有 .conf 或 .conflist 文件
```json
{
  "cniVersion": "1.0.0",
  "name": "crio",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "hairpinMode": true,
      "ipam": {
        "type": "host-local",
        "routes": [
            { "dst": "0.0.0.0/0" },
            { "dst": "::/0" }
        ],
        "ranges": [
            [{ "subnet": "10.85.0.0/16" }],
            [{ "subnet": "1100:200::/24" }]
        ]
      }
    }
  ]
}
```
在`/opt/cni/bin`文件下放置各个cni插件的二进制文件

## CNI规范
核心： 两个操作 + 一套数据格式

| 操作 | 触发时机 | 作用 |
|------|--------|------|
| `ADD` | 创建容器时 | 为容器配置网络（分配 IP、挂载网卡等） |
| `DEL` | 删除容器时 | 清理网络资源（释放 IP、删除设备等） |

输入输出使用JSON格式

## Flannel vs Calico 两种主流的CNI对比
#### Flannel 简单、基于 Overlay
- 数据路径
```text
Pod A (10.244.1.2) 
  → veth → linux bridge (cni0) 
  → VXLAN tunnel (UDP 8472) 
  → Node B 
  → cni0 → Pod B (10.244.2.3)
```
- 使用VXLAN封装跨界点流量
- 部署简单（一条 DaemonSet），适合小集群
- 性能损耗（封装/解封装）
- 无网络策略（NetworkPolicy）支持（除非配合其他插件）

#### Calico高性能、基于 BGP
- 数据路径
```text
Pod A (10.244.1.2) 
  → veth → linux routing table 
  → physical network (L3) 
  → Node B (via BGP route) 
  → Pod B (10.244.2.3)
```
- 让每个Node都成为BGP路由器，直接通告Pod CIDR
- 无封装，性能接近物理网络
- 原生支持 NetworkPolicy（基于 iptables/eBPF）
- 支持大规模集群（万级节点）
- 缺点：需要网络设备支持 BGP（或使用 IPIP 模式回退） 

- 对比

| 特性 | Flannel | Calico |
|------|--------|--------|
| 封装方式 | VXLAN / host-gw | BGP / IPIP / eBPF |
| 网络策略 | ❌（需额外插件） | ✅ 原生支持 |
| 性能 | 中（VXLAN 开销） | 高（无封装） |
| 部署复杂度 | 低 | 中（需规划 ASN、BGP peer） |
| 适用场景 | 开发测试、小集群 | 生产环境、AI 训练集群 |


在 GPU 多机训练中，NCCL 通信对网络延迟敏感，Calico BGP 模式是首选。

## 使用插件
一般更换CNI插件要彻底删除原来的集群，需要重建
```bash
minikube start --network-plugin=cni   # 不指定 --cni，让集群空跑

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/tigera-operator.yaml
   

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/custom-resources.yaml

kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=300s
   
kubectl get pods -n calico-system
```
查看minikube集群中关于cni的二进制文件与配置文件

在你创建 Operator 之前，Minikube 的 kubelet 已经启动了，它默认使用 bridge 插件（因为 --network-plugin=cni 会让 kubelet 使用默认的 bridge 插件，即 /etc/cni/net.d/1-k8s.conflist 这个文件，它是由 Minikube 自动生成的）。
所以需要禁用默认的bridge
```bash
sudo ip route del 10.244.0.0/16 dev bridge

sudo mv /etc/cni/net.d/1-k8s.conflist /etc/cni/net.d/1-k8s.conflist.disabled

sudo systemctl restart kubelet
```
- 验证
```bash
docker@minikube:/etc/cni/net.d$ ip route
default via 192.168.49.1 dev eth0 
blackhole 10.244.120.64/26 proto 80 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.49.0/24 dev eth0 proto kernel scope link src 192.168.49.2 

```
## 常见问题与调试技巧
- pod无法跨界点通信
防火墙阻塞 UDP 8472（Flannel VXLAN）或 IP 协议 4（IPIP）
Calico BGP 未建立邻居（检查 calicoctl node status

- Pod 获取不到 IP
CNI 插件是否 Running（kubectl get pods -n kube-system | grep -E 'flannel|calico'）
IP 池是否耗尽（calicoctl ipam show）
