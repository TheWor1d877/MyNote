Pod是K8s调度与管理的最小原子单位，他的生命周期由kubelet与Control Plain共同驱动，并不单纯由容器进程决定

- 一个 Pod 可包含多个容器（主容器 + Sidecar + Init 容器），但它们共享网络、存储、IPC 命名空间
- Pod 一旦被调度到节点，其生命周期状态由 Phase（阶段） 和 Conditions（条件） 共同描述。
- Pod 不会自我修复：如果 Pod 被删除或所在节点宕机，它不会重启——这是上层控制器（如 Deployment）的职责。

## Pod的五个Phase
| Phase | 含义 | 触发条件 | 是否可逆 |
|-------|------|--------|--------|
| `Pending` | 已创建但未调度成功或镜像拉取中 | API Server 接收 Pod 对象，但尚未绑定到节点 | ✅ 可能转为 Running 或 Failed |
| `Running` | 已绑定到节点，至少一个容器在运行 | kubelet 成功启动容器 | ❌（正常流程不可逆） |
| `Succeeded` | 所有容器正常退出（exit code 0） | Job/CronJob 场景常见 | 终态 |
| `Failed` | 至少一个容器非正常退出（非 0 退出码） | 应用崩溃、OOMKilled 等 | 终态 |
| `Unknown` | 无法获取 Pod 状态（如节点失联） | kubelet 与 API Server 失去通信 | 可能恢复或变为 Failed |

Running不一定就是应用可用，可能在初始化（加载大文件）

## Init容器
- 解耦初始化逻辑：将环境准备（如下载配置、等待依赖服务、生成证书）与主应用分离。
- 可以比主容器有更高权限：Init 容器可拥有更高权限（如挂载 Secret、修改 sysctl），主容器保持最小权限。
- 顺序执行且必须成功： 所有 Init 容器按定义顺序执行，任一失败则 Pod 重启（取决于重启策略）。

Init 容器不与其他容器并行运行。
所有 Init 容器完成后，主容器才启动。

如果Init容器启动失败，那么就会根据`.spec.restartPolicy`决定是否重启


## Pod重启策略
仅三种值，且只适用于 Pod 内容器退出，不适用于节点故障：
Always（默认）：容器退出即重启（Deployment 使用）
OnFailure：仅非 0 退出码时重启（Job 使用）
Never：永不重启（调试场景）

## Liveness与Readiness 探针 
[[GO/gin/服务器配置/健康检查|健康检查]]

## 使用init容器
```yaml
apiVersion: v1      # 失败 2 次就重启（更快看到效果）
kind: Pod
metadata:
  name: full-demo
spec:
  initContainers:
  - name: setup
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Init 完成于 $(date)" > /workdir/init-stamp; sleep 2']
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  containers:
  - name: main
    image: nginx:alpine
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html   # 让 nginx 能提供 Init 写的文件
  volumes:
  - name: workdir
    emptyDir: {}
```
## 配置错误的 Liveness Probe，观察 CrashLoopBackOff
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-liveness
spec:
  containers:
  - name: app
    image: nginx:alpine
    livenessProbe:
      httpGet:
        path: /healthz      # 这个路径不存在！
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
      failureThreshold: 3   # 连续失败 3 次后重启
    ports:
    - containerPort: 80
```

CrashLoopBackOff 是 Kubernetes 中最常见的 Pod 故障状态之一，表示容器反复启动、崩溃、重启，进入了一个死循环。