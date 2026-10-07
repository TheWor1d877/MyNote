iptables 规则分两条链：
- 链 1：处理 NodePort（外部流量）——不经过 ClusterIP
```text
外部请求 → NodeIP:NodePort → iptables DNAT → PodIP:targetPort
                                    ↑
                            这里直接修改目标地址为 Pod IP
                            ClusterIP 完全不参与！
```
访问NodePort，直接查Service对应的Endpoint通过DNAT直接访问Pod，不经过ClusterIP

外部流量不经过 ClusterIP 的二次转发，减少了一层 NAT 转换，性能更好。
- 链 2：处理 ClusterIP（内部流量）——必须经过虚拟 IP
```text
Pod 内部请求 → ClusterIP:port → iptables DNAT → PodIP:targetPort
                    ↑
              这里匹配的是虚拟 IP
              只有内部 Pod 才能访问这个 IP
```
只有内部IP才能访问ClusterIP！

集群内的 Pod 通过 ClusterIP 访问其他服务，这是 K8s 服务发现的核心机制。没有 ClusterIP，Pod 之间就没办法稳定通信了。