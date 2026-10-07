## ConfigMap
ConfigMap 是 Kubernetes 的一种 API 对象，用于存储非敏感的配置数据（如端口号、日志级别、功能开关），以键值对形式组织。


存储 UTF-8 文本（最大 1MB）
可通过 环境变量、Volume 挂载、命令行参数 注入 Pod
支持 热更新（挂载为 Volume 时）

#### 通过作为卷挂载的 ConfigMap 更新配置
核心原理: 将 ConfigMap 作为卷挂载到 Pod 中，更新 ConfigMap 后，Pod 内的文件也会自动更新

- 创建初始ConfigMap
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  log_level: "info"
  nginx.conf='server {
         listen 80;
         server_name localhost;
         location / {
             root /usr/share/nginx/html;
             index index.html;
         }
     }'
```

- 创建一个deployment将configmap挂载为卷
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-configmap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config           # 卷的名称
          mountPath: /etc/nginx/conf.d  # 挂载到容器的路径
          readOnly: true
      volumes:
      - name: config
        configMap:
          name: nginx-config     # ConfigMap 名称
          items:                 # 可选：指定要挂载的文件
          - key: nginx.conf      # ConfigMap 中的 key
            path: custom.conf    # 容器中的文件名
EOF
```
- 验证当前配置
```bash
# 进入容器查看配置
kubectl exec -it deployment/nginx-with-configmap -- cat /etc/nginx/conf.d/custom.conf
```
- 更新ConfigMap
```bash
kubectl edit configmap nginx-config
```
- 让配置生效
```bash
kubectl edit configmap nginx-config
```



## Secret
Secret 是 Kubernetes 的一种 API 对象，用于存储敏感信息（如密码、Token、TLS 证书），默认以 Base64 编码存储（注意：不是加密！）。

存放类型：
- 通用类型 `Opaque`
- TLS证书 `kubernetes.io/tls`
- 镜像仓库凭据 `kubernetes.io/dockerconfigjson`

Secret 以 Base64 字符串形式明文存储在 etcd
任何能访问 etcd 的人（如集群管理员）可直接读取
#### yaml模板示例
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=       # echo -n "admin" | base64
  password: czNjcjN0IQ==   # echo -n "s3cr3t!" | base64
```

YAML 中 data 字段必须是 Base64 编码的字符串。若不想手动编码，可用 stringData（仅 Secret 支持）：

## 使用ConfigMap & Secret的两种方式
- 通过环境变量注入： 容器启动时，Kubernetes 将值注入环境变量。后续 ConfigMap/Secret 更新不会影响已运行的 Pod。
```yaml
spec:
  containers:
  - name: app
    image: my-app
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```
- 通过Volumn挂载: 支持热更新：当 ConfigMap/Secret 更新时，挂载的文件内容会自动同步（约 1 分钟延迟）,应用需自行监听文件变化
```yaml
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secret-volume
      mountPath: /etc/secret
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-secret
```
Kubelet 定期（默认 1 分钟）向 API Server 查询 ConfigMap/Secret 的最新版本
如果发现变更，Kubelet 会原子性地替换挂载目录下的文件
应用进程需具备重载配置的能力（如 SIGHUP 信号、inotify 监听）

## ConfigMap 与 Secret对比

| 属性 | ConfigMap | Secret |
|------|-----------|--------|
| API 字段 | `data`（直接存字符串） | `data`（存 Base64 字符串）或 `stringData`（自动转 Base64） |
| 示例 YAML | ```data: { key: "value" }``` | ```data: { key: "dmFsdWU=" }``` |
| 底层 etcd 存储 | 明文 `"value"` | 明文 `"dmFsdWU="`（Base64 字符串） |
| 是否可读 | 任何人可读明文 | 任何人可 Base64 解码得明文 |

| 行为 | ConfigMap | Secret |
|------|-----------|--------|
| Pod 默认 ServiceAccount 权限 | 可读同 Namespace 所有 ConfigMap | ❌ 不可读任何 Secret（除非显式授权） |
| kubectl describe 输出 | 显示完整 `data` 内容 | 隐藏 `data` 内容（显示 `<REDACTED>`） |
| 审计日志敏感度 | 普通事件 | 标记为敏感操作（若启用审计） |

#### 使用场景对比
- Secret
当数据属于密码，token，密钥，证书等等敏感信息的时候使用
当需要k8s默认的RBAC(Role-Based Access Control ,基于角色的访问控制)的默认限制的时候
未来需要启动etcd静态加密

- ConfigMap
非敏感数据，对于开发人员可见

