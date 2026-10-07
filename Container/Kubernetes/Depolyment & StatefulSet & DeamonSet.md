Kubernetes 的核心思想是 “声明式 API + 控制器模式”。Pod 本身是易失性的（节点故障即消失），因此需要上层控制器来：
- 保证期望副本数量
- 管理Pod身份
- 处理升级，回滚，扩缩容

但不同应用对“身份”“顺序”“存储”的要求截然不同：
- Web 服务：任意副本可互换 → 无状态
- 数据库/消息队列：每个实例有唯一身份和稳定存储 → 有状态
- 日志采集/监控代理：每台机器必须运行一个 → 节点绑定

## Deployment: 无状态应用的标准答案
Deployment 是 Pod 的"经理"或"管理员"。
你告诉 Deployment："我要运行 3 个 nginx，版本 1.20"，Deployment 负责执行并维持这个状态

| 职责       | 说明                       |
| -------- | ------------------------ |
| **数量保证** | 你要求 3 个 Pod，挂了 1 个就补 1 个 |
| **滚动更新** | 从 v1.0 升级到 v2.0，逐个替换，不停机 |
| **一键回滚** | 新版本有问题，秒回旧版本             |
| **版本控制** | 记住每次更新记录，随时可回退           |
#### 核心特征
| 特性     | 行为                                        |
| ------ | ----------------------------------------- |
| Pod 身份 | 无身份（Pod 名称随机，如 `nginx-7d5b8c9f4-xk2l9`）   |
| 网络标识   | 通过 Service 负载均衡访问，Pod IP 动态分配             |
| 存储     | 通常使用临时卷或共享存储（如 NFS），不绑定特定 Pod             |
| 扩缩容    | 并行创建/删除 Pod，无顺序要求                         |
| 更新策略   | RollingUpdate（默认）：逐步替换旧 Pod；Recreate：先删后建 |
#### 使用场景
HTTP 微服务（Nginx、Spring Boot）
无状态计算任务（图像处理、批处理 Worker）
可水平扩展的中间件（Redis Cluster 的 proxy 层）

#### 关键机制
replicaSet + revision History

- Deployment 不直接管理 Pod，而是创建 ReplicaSet（RS）
- 每次更新（如改镜像）会生成新 RS，旧 RS 保留（用于回滚）
- revisionHistoryLimit 控制保留多少历史版本（默认 10）

caSet 是确保指定数量的 Pod 副本始终运行的控制器。它的职责很单一：

数量控制：如果 Pod 挂了，它启动新的；如果太多了，它终止多
#### 命令示例
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
#### yaml文件示例
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  labels:
    app: nginx-server
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      name: nginx-server
      labels:
        app: nginx-server
    spec:
      containers:
        - name: nginx-server
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              protocol: TCP
      restartPolicy: Always
      
```
## StatefulSet：为有状态应用而生
StatefulSet 是专门用来运行“有状态应用”的控制器，比如数据库（MySQL、Redis）、消息队列（Kafka、ZooKeeper）等。
它保证每个 Pod 都有 稳定的、唯一的身份标识 和 持久化存储，即使 Pod 被删除重建，它的“身份”和“数据”也不会变。

#### 核心特征
| 特性 | StatefulSet 行为 |
|------|------------------|
| Pod 身份 | 稳定、唯一的序号标识（如 `mysql-0`, `mysql-1`） |
| 网络标识 | 每个 Pod 有 固定 hostname 和 DNS 记录（`mysql-0.mysql.default.svc.cluster.local`） |
| 存储 | 每个 Pod 绑定独立 PVC，即使 Pod 被删除重建，PVC 仍关联原 Pod 序号 |
| 启停顺序 | 扩容：`0 → 1 → 2`；缩容：`2 → 1 → 0`（有序） |
| 更新策略 | RollingUpdate（有序）、OnDelete（手动删 Pod 才更新） |

#### 实现细节
- Headless Service
Headless Service 是一种特殊的 Kubernetes Service，它的核心特点是 不分配 Cluster IP，也不通过 kube-proxy 做负载均衡，而是直接返回后端 Pod 的 IP 列表（通常是 DNS 域名解析直接解析到 Pod IP）。

定义一个headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None  # 关键：这里设为 None
  selector:
    app: myapp
  ports:
    - port: 8080
```
举例说明：
```bash
￼￼# 解析 Service 名称
  dig mysql-headless.default.svc.cluster.local
  # 返回：多个 A 记录 → 直接是 Pod 的真实 IP
  10.1.1.1
  10.1.1.2
  10.1.1.3
```

- volumeClaimTemplates
自动生成 PVC，命名规则：`<claimName>-<statefulSetName>-<ordinal>`

#### 使用场景
分布式数据库：MySQL MGR、PostgreSQL Patroni、MongoDB Replica Set
消息队列：Kafka、RabbitMQ（需固定 Broker ID）
存储系统：Etcd、MinIO 分布式模式、Ceph MDS（配合 Rook）
<span style="color:rgb(221, 85, 85)"> 注意：StatefulSet 不等于高可用！它只提供稳定身份和存储，集群协议（如 Raft、Paxos）仍需应用自身实现。</span> 

#### 案例： 使用StatefulSet部署nginx
```yaml
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


## DaemonSet：确保每（合格）节点运行一个 Pod
#### 核心特征
| 特性 | 行为 |
|------|------|
| 调度目标 | 自动在每个节点（或满足 nodeSelector/taint 的节点）部署一个 Pod |
| Pod 身份 | 无全局序号，但与节点强绑定 |
| 典型用途 | 节点级代理：日志收集（Fluentd）、监控（Node Exporter）、网络插件（Calico）、存储插件（CSI Node Plugin） |

#### 机制解读
1. 新节点加入集群 → 自动创建 Pod
2. 当node节点被标记为不可调度的时候，仍然会运行DeamonSet Pod,除非使用 nodeAffinity排除
#### 使用场景
- 日志收集
把这台机器上所有容器的日志文件读出来，发到中央日志系统（如 ELK）

- 监控节点
上报 CPU、内存、磁盘、网络等指标（如 Prometheus Node Exporter）

- 网络，存储插件
分布式存储的 CSI 驱动
如 Ceph CSI、JuiceFS CSI 的 Node Plugin 就是以 DaemonSet 形式部署

这些功能都依赖“本机资源”（文件、硬件、内核模块），不能跨机器远程操作。

#### 练习： 实现一个本地缓存预热DeamonSet
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cache-warmup-config
data:
  data-001.csv: "http://192.168.49.2:8080/datasets/data-001.csv"
  model-v2.bin: "http://192.168.49.2:8080/models/model-v2.bin"
  # 使用本机ip作为演示，避免网络波动问题
  
---
# cache-warmup-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cache-warmup
spec:
  selector:
    matchLabels:
      app: cache-warmup
  template:
    metadata:
      labels:
        app: cache-warmup
    spec:
      containers:
        - name: warmer
          image: busybox
          imagePullPolicy: IfNotPresent
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                echo "[$(date)] Checking cache config..."
                for file in /config/*; do
                  if [ -f "$file" ]; then
                    filename=$(basename "$file")
                    url=$(cat "$file")
                    target="/cache/$filename"
                    if [ ! -f "$target" ]; then
                      echo "[$(date)] Warming up $filename from $url"
                      wget -q -O "$target" "$url" && echo "  -> Success" || echo "  -> Failed"
                    else
                      echo "[$(date)] $filename already cached"
                    fi
                  fi
                done
                sleep 30
              done
          volumeMounts:
            - name: config-volume
              mountPath: /config
            - name: cache-volume
              mountPath: /cache
      volumes:
        - name: config-volume
          configMap:
            name: cache-warmup-config
        - name: cache-volume
          hostPath:
            path: /cache
            type: DirectoryOrCreate
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
          # 如果node设置了NoSchedule，这种pod在这样的node中具有容忍（豁免权），
          # 并且允许进入该node进行调度
```
在minikube集群中打开8080端口的http服务
实现网络资源的    本地缓存预热

## Deployment 与 St
- Deployment (无状态) 的行为
Deployment 认为所有 Pod 都是完全相同的、可互换的。
节点 NotReady 后，Deployment 会立即判定该 Pod 已死。它会在另一个健康的节点上，创建一个全新的、名字不同的 Pod
这个新 Pod 不附带任何旧数据。它使用一个全新的、空的存储卷。如果你的应用是无状态的（比如一个Web服务器），这完全没问题。

- StatefulSet (有状态) 的行为
StatefulSet 认为每个 Pod 都有唯一且持久的身份。nSet跟sidcar有点类似阿，一个是pod级别的，一个是容器级别的
   Deamon Set是不是将node上面所有的pod的日志收集
当节点A宕机，StatefulSet 控制器不会立即重建 。它会等待一个可配置的时间，通常是等到原节点被 kubectl delete node 或经过 --pod-eviction-timeout (默认5分钟) 后，且确认 确实已经不在任何地方运行。
在确认原Pod已死亡后，StatefulSet 会在一个新节点上，用旧名字重建 Pod。
Kubernetes 会自动将 原本属于这个容器的持久化卷（PersistentVolume）重新挂载到这个新 Pod 上。启动后，应用就能看到故障前的所有数据。

#### 核心区别
Pod 的名字和身份（最重要）
启动和删除顺序
存储 Deployment新 Pod 拿到的是新空盘（除非特殊配置），StatefulSet新 Pod 自动挂载到原来的盘


