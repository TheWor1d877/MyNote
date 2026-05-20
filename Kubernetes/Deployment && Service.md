Service 和 Deployment 都是"计划清单"，只是它们解决的是不同的问题。
二者管理的都是pod级别的

## Deployment
Deployment 是 Pod 的"经理"或"管理员"。
你告诉 Deployment："我要运行 3 个 nginx，版本 1.20"，Deployment 负责执行并维持这个状态

| 职责       | 说明                       |
| -------- | ------------------------ |
| **数量保证** | 你要求 3 个 Pod，挂了 1 个就补 1 个 |
| **滚动更新** | 从 v1.0 升级到 v2.0，逐个替换，不停机 |
| **一键回滚** | 新版本有问题，秒回旧版本             |
| **版本控制** | 记住每次更新记录，随时可回退           |

```bash
# 创建一个 Deployment（开店计划）
kubectl create deployment nginx --image=nginx

# 查看 Deployment
kubectl get deployments

# 从 1 个 Pod 扩展到 3 个
kubectl scale deployment nginx --replicas=3

# 更新镜像版本（滚动更新）
kubectl set image deployment/nginx nginx=nginx:1.20

# 回滚到上一个版本
kubectl rollout undo deployment/nginx
```

## Service
**Service 是 Pod 的"前台总机"或"电话黄页"。**
你告诉 Service："我要让别人能访问到那些 `app=nginx` 的 Pod"，Service 负责创建一个固定的入口。

| 问题                 | Service 的解决方案   |
| ------------------ | --------------- |
| Pod 的 IP 不稳定（重启就变） | Service 提供固定 IP |
| 多个 Pod 负载均衡        | Service 自动分发请求  |
| 外部无法访问内部 Pod       | Service 提供外部端口  |
|                    |                 |
默认情况下，Pod 只能通过 Kubernetes 集群中的内部 IP 地址访问。 要使得 hello-node 容器可以从 Kubernetes 虚拟网络的外部访问，你必须将 Pod 通过 Kubernetes Service 公开出来。

使用kubectl expose 命令将pod暴露出去（集群外，公网等等）

#### Service数据流
启动nginx：
```bash
kubectl expose depolyment --type=NodePort --port=8080 --target-port=80
```
数据流：
```txt
你的电脑
    │
    │ 访问 http://192.168.49.2:30982
    │         ↑               ↑
    │      Minikube IP     NodePort
    │
    ▼
Minikube 节点（192.168.49.2）
    │ 节点上监听 30982 端口
    │
    ▼
Service（10.101.228.231）
    │ Service 监听 8080 端口
    │
    ▼
Pod（10.244.0.x）
    │ Pod 监听 80 端口（target-port）
    │
    ▼
nginx服务
```
真正干活的nginx服务在80端口
这个Service在8080端口（内部访问）
浏览器需要访问30982端口
