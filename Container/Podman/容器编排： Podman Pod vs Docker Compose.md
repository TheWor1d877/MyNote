## Docker Compose： 声明式编排
```txt
Docker Compose 架构：
┌─────────────────┐
│ docker-compose.yml │ ← 声明式配置文件
└────────┬────────┘
         │
┌────────▼────────┐
│ Docker Compose  │ ← 外部编排工具
│ CLI/Engine      │
└────────┬────────┘
         │
┌────────▼────────┐
│   容器1         │ ← 独立容器
│   web-app       │
├─────────────────┤
│   容器2         │ ← 独立容器  
│   database      │
├─────────────────┤
│   容器3         │ ← 独立容器
│   cache         │
└─────────────────┘
```
声明式配置： 通过Yaml文件定义整个应用栈

## Podman Pod 原生容器组
```bash
Podman Pod 架构：
┌─────────────────┐
│     Pod         │ ← 容器组（共享命名空间）
│ ┌─────────────┐ │
│ │   容器1     │ │ ← 共享网络、IPC、UTS 命名空间
│ │   web-app   │ │
│ ├─────────────┤ │
│ │   容器2     │ │ ← 共享相同的网络栈
│ │   sidecar   │ │
│ └─────────────┘ │
└─────────────────┘
```
原生支持：Pod 是 Podman 内置的一等公民
共享命名空间：容器间共享网络、IPC 等资源
Kubernetes 兼容：直接映射到 Kubernetes Pod 概念

可以创建pod，然后在pod中运行容器
```bash
# 创建 Pod
podman pod create \
  --name myapp \
  --publish 8080:80 \
  --share net,ipc,uts

# 在 Pod 中运行容器
podman run -d \
  --pod myapp \
  --name web \
  -v ./html:/usr/share/nginx/html \
  nginx:latest

podman run -d \
  --pod myapp \
  --name db \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres:13

podman run -d \
  --pod myapp \
  --name cache \
  redis:latest redis-server --appendonly yes
```
核心特点：
原生支持：Pod 是 Podman 内置的一等公民
共享命名空间：容器间共享网络、IPC 等资源
Kubernetes 兼容：直接映射到 Kubernetes Pod 概念
轻量级：无需额外的编排工具

