Role与ClusterRole是基于RBDC的权限模型
通过“角色 + 绑定”机制，实现最小权限原则（Principle of Least Privilege）
## RBAC四大核心对象
| 对象 | 作用 |
|------|------|
| Role / ClusterRole | 定义权限集合（能对哪些资源执行什么操作） |
| RoleBinding / ClusterRoleBinding | 将角色绑定到主体（User / Group / ServiceAccount） |

Role RolezBinding 是namespae级别的
ClusterRole ClusterRoleBinding是集群级别的

## Role：命名空间内的权限定义
Role 是一个 Namespace 内的权限规则集合，仅能授予对该 Namespace 内资源的访问权限。

```yaml
# pod-reader-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]          # 核心 API 组（Pod, Service 等）
  resources: ["pods"]      # 资源类型
  verbs: ["get", "list"]   # 允许的操作
```
Role 不能跨 Namespace，也不能访问集群级资源（如 Node, PersistentVolume, Namespace 本身）

## ClusterRole
ClusterRole 可授予对集群级资源或所有 Namespace 中资源的访问权限。
```yaml
# secret-reader-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

ClusterRole 本身不绑定主体，必须通过 ClusterRoleBinding 或 RoleBinding 引用。

##  RoleBinding 
作用域：单个 Namespace
可绑定对象：
Role（同 Namespace）
ClusterRole（但权限被限制在该 Namespace 内）
```yaml
# rolebinding-to-clusterrole.yaml
 apiVersion: rbac.authorization.k8s.io/v1
 kind: RoleBinding
 metadata:
   name: read-secrets-in-default
   namespace: default
 subjects:
 - kind: ServiceAccount
   name: my-app
   namespace: default
 roleRef:
   kind: ClusterRole    # 引用集群角色
   name: secret-reader  # 但实际只能读 default Namespace 的 Secret
   apiGroup: rbac.authorization.k8s.io
```


## ClusterRoleBinding
   作用域：整个集群
   可绑定对象：仅 ClusterRole
   效果：主体获得 ClusterRole 定义的完整集群权限
   示例：授予运维用户集群管理员权限
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: ops-admin
   subjects:
   - kind: User
     name: alice@example.com
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: cluster-admin  # 内置超级角色
     apiGroup: rbac.authorization.k8s.io
   ```

## k8s 中的身份体系
- User与ServiceAccount两种认证方式
User是外部用户认证方式，ServiceAccount是内部用户认证方式
#### User
User 是 Kubernetes 中用于标识集群外部实体（通常是人或自动化系统）的身份。
注意：Kubernetes 本身不存储 User 的密码或凭证，它只验证由可信第三方提供的身份证明
- 工作流程
用户通过外部系统（如 CA 或 OIDC）获得身份凭证（证书或 Token）
用户使用 kubectl 发起请求，附带凭证
API Server 验证凭证有效性 → 确认 User 身份
RBAC 系统检查该 User 是否有权限执行操作

#### ServiceAccount
ServiceAccount 是 Kubernetes 为Pod 内运行的进程提供的内置身份机制。每个 Pod 默认关联一个 ServiceAccount，用于向 API Server 证明自己的身份

创建的ServiceAccount带有一个Token，当容器创建的时候Token会自动挂载到容器内部
应用读取Token并且 在请求API Server的时候携带

ServiceAccount 的权限不是限制“Pod 自身的行为”，而是限制“Pod 内程序代表它向 API Server 发起的请求”。

- 示例代码
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: default

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader-rbac
rules:
  - apiGroups: [""]
    resources: ["pod"]
    verbs: ["get","list"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader-role

```