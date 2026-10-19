#面试-云原生-复习 解释 Service、Endpoints、EndpointSlice 的关系，以及大规模推理服务下 EndpointSlice 为什么重要

## Service
默认情况下，Pod 只能通过 Kubernetes 集群中的内部 IP 地址访问。 要使得容器可以从 Kubernetes 虚拟网络的外部访问，你必须将 Pod 通过Service 公开出来。

Service分到一个虚拟IP或者DNS地址，客户端访问这个IP,k8s保证将请求转发到任意一个Pod上面，这样就实现了负载均衡
这里的关键是解耦，客户端不需要知道有多少Pod，也不需要知道Pod的具体IP
#### Service工作原理
Service本身不处理流量，依赖组件进行转发
- kube-proxy
运行在每个Node上面
监听Service与Endoint变化
生成iptables/IPVS规则
有Iptables与IPVS两种模式，但是一般只能用于小集群

- CNI插件直通
Calico/Cilium可以直接绕过kube-proxy
直接在 eBPF/XDP 层实现 Service 转发
优势：更低延迟、更高吞吐、支持 DSR（Direct Server Return）

- LoadBalancer
云厂商集成：自动创建外部负载均衡器（AWS ELB, GCP CLB 等）
自动分配公网 IP 并关联 NodePort/ClusterIP
关键特性：
依赖 Cloud Controller Manager (CCM)
成本较高（每个 Service 一个 LB 实例）
支持高级功能：健康检查、SSL 终止等

-  ExternalName
返回 CNAME 记录，将 Service 映射到外部域名
不创建 ClusterIP，无 Endpoints
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db
spec:
  type: ExternalName
  externalName: my.database.example.com
```
访问ClusterIP直接解析为url网站

#### Service的四种类型
- ClusterIP(默认类型)
仅仅在集群内部可达的的虚拟IP
没有外部暴露能力
必须通过集群内部的Pod或者Node进行访问

- NodePort
在 每个 Node 的固定端口（30000-32767） 上暴露 Service
自动创建关联的 ClusterIP
NodePort不会自动高可用，需要手动配置负载均衡

## Endpoint
Endpoint是Service背后的地址映射关系的列表
kube-controller-manager 里有一个 endpoints controller，它使用watch持续观察Pod或者Service，一旦发生变化就更新对应的EndPoint对象

EndPoint本质上就是一堆IP:Port的映射关系，以及相关Pod是否准备好

用户可以手动创建Endpoint对象来绑定IP，一般不推荐

有了Endpoint 就将Servicee与Pod的声明周期解耦了:
Pod 随时扩缩容，崩溃重建（IP发生变化）
Service 必须保持稳定（ClusterIP 不变）

一个Endpoint示例：
```text
[ip: 10.244.120.72
nodeName: minikube
targetRef:
  kind: Pod
  name: nginx-5d57f8c97-7mpq2
  namespace: default
  uid: b03134a9-b951-44f9-8f1f-34f980504e6d, ip: 10.244.120.75
nodeName: minikube
targetRef:
  kind: Pod
  name: nginx-5d57f8c97-qxqx8
  namespace: default
  uid: 4f0ab79e-3efc-45aa-b16c-6be2bbc7e036, ip: 10.244.120.79
nodeName: minikube
targetRef:
  kind: Pod
  name: nginx-5d57f8c97-52p5j
  namespace: default
  uid: c2b0e63f-0068-41f0-a778-b1240566ce95]
```
## EndpointSlice
1. 如果Pod数量非常大，单个Endpoint对象变得十分巨大
2. 每个Pod的变化都要更新整个Endpoint对象，这让etcd压力变大
3. 更新是原子的，Endpoint是整体替换，但是往往需要修改的内容很少，只需修改一片即可
4. 推理场景下副本多，负载波动经常，扩缩容频繁，滚动更新也是很频繁的
   这些操作都会大致后端Pod地址发生变化，所以对Endpoint的管理更加重要了


解决方案： 分片处理
```yaml
# EndpointSlice 1
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-svc-abcde
  labels:
    kubernetes.io/service-name: my-svc  # ← 关键关联标签
endpoints:
- addresses: ["10.244.1.5"]
  conditions: { ready: true }
- addresses: ["10.244.2.3"]
ports:
- port: 8080
```
更新粒度小（只改一个 slice）
支持拓扑感知路由（如 topology.kubernetes.io/zone）
## kube-proxy
kube-proxy是k8s集群的网络规则维护员
不直接转发任何数据包，只负责在Node上面创建与维护规则，让内核根据这些规则完成转发
selector是负责转发数据包的

Kube-proxy负责：
- Service的ClisterIP转发
- NodePort转发