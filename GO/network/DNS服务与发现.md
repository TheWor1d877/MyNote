**解决“集群节点地址从哪来”的问题**，让分布式存储系统在 K8s 或动态环境中自动发现彼此，**告别硬编码 IP 列表**。
再真实场景中节点会扩容缩容，节点故障重启，无法知道所有的IP地址，所以需要服务发现

## 解决方案： Headless Service + DNS SRV
通过 DNS 查询 `_port._tcp.service.namespace.svc.cluster.local` 获取所有 Pod IP:Port

## 优点
通过DNS SRV自动发现集群成员，无需硬编码
完美适配 K8s StatefulSet
解耦部署拓扑	节点增删无需改代码