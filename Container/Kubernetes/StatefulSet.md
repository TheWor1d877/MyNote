StatefulSet 是专门用来运行“有状态应用”的控制器，比如数据库（MySQL、Redis）、消息队列（Kafka、ZooKeeper）等。
它保证每个 Pod 都有 稳定的、唯一的身份标识 和 持久化存储，即使 Pod 被删除重建，它的“身份”和“数据”也不会变。

| 特性 | Deployment（无状态） | StatefulSet（有状态） |
|------|---------------------|----------------------|
| Pod 名称 | 随机生成，如 `web-7d5b8c9f4-xk2l9` | 固定有序，如 `web-0`, `web-1`, `web-2` |
| 网络标识（Hostname） | 每次重建会变 | 永远不变（`web-0` 的 hostname 就是 `web-0`） |
| 存储（Volume） | 通常不保留，或共享 | 每个 Pod 有自己专属的持久化磁盘（PVC），删了 Pod 磁盘还在 |
| 启动/停止顺序 | 并行，无顺序 | 有序：`web-0` → `web-1` → `web-2`（启动时）；反向停止（停止时） |
| 适用场景 | Web 服务器（Nginx）、API 服务等 | 数据库、分布式系统（需要记住“我是谁”） |

## 案例：使用statefulset部署nginx

```bash
apiVersion: v1
   kind: Service
   metadata:
     name: nginx
     labels:
       app: nginx
   spec:
     clusterIP: None
     ports:
     - port: 8080
       targetPort: 8080
       name: web
     selector:
       app: nginx
   ---
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: web
   spec:
     selector:
       matchLabels:
         app: nginx
     serviceName: "nginx"
     replicas: 1
     template:
       metadata:
         labels:
           app: nginx
       spec:
         containers:
         - name: nginx
           image: nginx:latest
           imagePullPolicy: Never
           ports:
           - containerPort: 8080
             name: web
           securityContext:
             runAsNonRoot: true
             allowPrivilegeEscalation: false
             capabilities:
               drop:
               - ALL
             runAsUser: 101
             seccompProfile:
               type: RuntimeDefault
           volumeMounts:
           - name: nginx-config
             mountPath: /etc/nginx/nginx.conf
             subPath: nginx.conf
         volumes:
         - name: nginx-config
           configMap:
             name: nginx-config
   ---
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: nginx-config
   data:
     nginx.conf: |
       user nginx;
       worker_processes auto;
       error_log /dev/stderr info;
       pid /tmp/nginx.pid;
   
       events {
           worker_connections 1024;
       }
   
       http {
           include       /etc/nginx/mime.types;
           default_type  application/octet-stream;
           access_log /dev/stdout;
           sendfile        on;
           keepalive_timeout  65;
   
           client_body_temp_path /tmp/client_temp;
           proxy_temp_path       /tmp/proxy_temp;
           fastcgi_temp_path     /tmp/fastcgi_temp;
           uwsgi_temp_path       /tmp/uwsgi_temp;
           scgi_temp_path        /tmp/scgi_temp;
   
           server {
               listen       8080;
               server_name  localhost;
               location / {
                   root   /usr/share/nginx/html;
                   index  index.html index.htm;
               }
               error_page   500 502 503 504  /50x.html;
               location = /50x.html {
                   root   /usr/share/nginx/html;
               }
           }
       }
```
