## 通过ConfigMap更新配置
#### 通过作为卷挂载的 ConfigMap 更新配置
核心原理: 将 ConfigMap 作为卷挂载到 Pod 中，更新 ConfigMap 后，Pod 内的文件也会自动更新

- 创建初始ConfigMap
```bash
# 创建一个包含 nginx 配置的 ConfigMap
kubectl create configmap nginx-config \
  --from-literal=nginx.conf='server {
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

#### 通过 ConfigMap 更新 Pod 环境变量
环境变量的更新机制与文件挂载完全不同：修改 ConfigMap 后，已运行的 Pod 不会自动获得新环境变量，必须重建 Pod。

- 创建 ConfigMap 存放环境变量
```bash
# 方式1：从字面量创建
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_MODE=production \
  --from-literal=LOG_LEVEL=info

# 方式2：从文件创建
cat > app.properties <<EOF
APP_COLOR=blue
APP_MODE=production
LOG_LEVEL=info
EOF
kubectl create configmap app-config --from-env-file=app.properties
```
- 创建 Deployment，通过 envFrom 注入环境变量
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-env
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: alpine:latest
        imagePullPolicy: IfNotPresent
        command: ["sleep", "infinity"]
        envFrom:
        - configMapRef:
            name: app-config   # 注入整个 ConfigMap
        # 也可以单独注入某个键
        # env:
        # - name: SINGLE_VAR
        #   valueFrom:
        #     configMapKeyRef:
        #       name: app-config
        #       key: APP_COLOR
EOF
```
#### 创建多Pod共享的ConfigMap卷
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-reloader
spec:
  containers:
  # 主容器：nginx
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: shared-config        # 共享配置卷
      mountPath: /etc/nginx/conf.d
      readOnly: true
  
  # Sidecar 容器：监控配置变化并重载
  - name: reloader
    image: alpine:latest
    command: ["/bin/sh", "-c"]
    args:
    - |
      apk add --no-cache inotify-tools curl
      while true; do
        # 监控配置文件变化
        inotifywait -e modify,create,delete /etc/nginx/conf.d/
        echo "Config changed, reloading nginx..."
        # 让 nginx 重载配置
        curl -X POST http://localhost:80/nginx-reload
        # 或者通过共享进程命名空间发送信号
        # kill -HUP $(pidof nginx)
      done
    volumeMounts:
    - name: shared-config
      mountPath: /etc/nginx/conf.d
      readOnly: true
  
  volumes:
  - name: shared-config
    configMap:
      name: nginx-shared-config
      items:
      - key: site.conf
        path: site.conf
EOF
```
#### 设置ConfigMap为不可变
```bash
# 1. 先导出现有 ConfigMap
kubectl get configmap app-config -o yaml > app-config.yaml

# 2. 编辑文件，添加 immutable: true
# 3. 创建新的不可变 ConfigMap
kubectl apply -f app-config.yaml

# 4. 删除原 ConfigMap（可选）
kubectl delete configmap app-config
```
