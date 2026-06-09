
## 在集群/命名空间中应用Pod安全标准
#### 三种模式
Pod 安全准入 允许你使用以下模式应用内置的 Pod 安全标准： enforce、audit 和 warn。
#### 三个安全标准

| 等级                  | 大白话                  | 适用场景             |
| ------------------- | -------------------- | ---------------- |
| **Privileged**（特权级） | 无限制，容器想干嘛干嘛          | 完全信任的、需要高权限的系统容器 |
| **Baseline**（基线级）   | 限制了一些明显危险的操作         | 大部分普通应用          |
| **Restricted**（严格级） | 最严格的限制，几乎不能做任何"出格"的事 | 高安全要求的生产环境       |

使用 `--dry-run=server` 来了解应用不同的 Pod 安全标准时会发生什么：
1. Privileged
```bash
kubectl label --dry-run=server --overwrite ns --all \
pod-security.kubernetes.io/enforce=privileged
```
2. baseline
```bash
kubectl label --dry-run=server --overwrite ns --all \
pod-security.kubernetes.io/enforce=baseline
```
3. restricted
```bash
kubectl label --dry-run=server --overwrite ns --all \
pod-security.kubernetes.io/enforce=restricted
```

#### 修改安全标准
```bash
kubectl label --overwrite ns my-app \
  pod-security.kubernetes.io/enforce=baseline \    # 门槛1: 必须过baseline
  pod-
  security.kubernetes.io/audit=restricted      # 门槛2: 不过restricted就记日志
  pod-
  security.kubernetes.io/warn=restricted \     # 门槛3: 不过restricted就给警告
```
不过baseline就直接enforce
不过restricted就audit与warn


#### 查看一个命名空间当前生效的PSA策略
`kubectl get ns my-app -o yaml`
- 输出：
```txt
howardhe@ASUS-HE:~$ kubectl get ns default -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2026-05-08T01:58:53Z"
  labels:
    kubernetes.io/metadata.name: default
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
  name: default
  resourceVersion: "35211"
  uid: 2a292940-16e9-4a0b-b3bc-3f733b04ea64
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

## 使用AppArmor限制容器对资源的访问
AppArmor 配置文件可以在 Pod 级别或容器级别指定。容器 AppArmor 配置文件优先于 Pod 配置文件。