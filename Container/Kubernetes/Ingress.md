## Service的问题
Kubernetes 原生 Service 是 L4 抽象，而现代 Web 应用需要 L7（HTTP/HTTPS）路由能力。

HTTP/HTTPS 协议基于域名和路径路由，而 NodePort 和 LoadBalancer 只能按 IP + 端口 区分服务。
如下：
```text
https://shop.example.com/        → 前端页面
https://shop.example.com/api/    → 后端 API
https://admin.example.com/       → 管理后台
```
这三个 URL 共享同一个公网 IP 和 443 端口，靠 Host 头 和 URL 路径 区分后端服务。

但 Kubernetes 的 Service（即使是 LoadBalancer）无法实现这种七层（L7）路由——它只工作在四层（L4，TCP/UDP）。

## Ingress 的定义：L7 路由规则的声明
Ingress 是 Kubernetes 的一种 API 对象，用于管理对集群内服务的 HTTP 和 HTTPS 访问。它提供基于 Host 和 Path 的路由规则，并支持 TLS 终止、负载均衡等高级功能

Ingress 是一组规则，类似于Service一样
要让 Ingress 生效，必须部署一个 Ingress Controller —— 这才是真正的反向代理（如 Nginx、Traefik）

#### Ingress 的组成部分
- Ingress resource(资源对象)： 使用yaml描述路由规则
- Ingress Controller(控制器)： 一个运行在集群中的 Pod（通常以 Deployment + Service 形式部署）。这是真正的反向代理
- 外部访问入口
完整数据流：
用户浏览器 → Ingress Controller Pod（L7代理）→ 直接查询 Endpoint → 直接转发到后端 Pod IP

## 创建Ingress资源
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: shop.local               # 自定义域名
    http:
      paths:
      - path: /                    # 根路径
        pathType: Prefix           # 前缀匹配
        backend:
          service:
            name: frontend         # 转发到 frontend Service
            port:
              number: 80
      - path: /api/                # /api/ 路径
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 8080
```


| 能力 | Service (L4) | Ingress (L7) |
|------|--------------|--------------|
| 基于 IP:Port 路由 | ✅ | ❌ |
| 基于 Host/Path 路由 | ❌ | ✅ |
| TLS 终止 | ❌ | ✅ |
| 路径重写 | ❌ | ✅（通过注解） |
| 依赖外部组件 | kube-proxy | Ingress Controller |

