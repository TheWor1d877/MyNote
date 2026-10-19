## 常用命令
- 初次使用
```bash
minikube start \
  --driver=docker \
  --image-mirror-country=cn \
  --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers \
  --kubernetes-version=v1.28.3 \
  --preload=false \
  --cpus=2 \
  --memory=2048 \
  --disk-size=20g
```
- 再次启动
```bash
minikube start
```
- 查看deployment
```bash
kubectl get deployments
```
- 删除deployment
```bash
kubectl delete deployments <name>
```
- 查看pod状态
```bash
kubectl get pods (-o wide)
kubectl get pods --watch # 实时监控变化
```
- 导入镜像
```bash
minikube image load nginx:latest
```
- 查看集群事件
```bash
kubectl get events
```
- 查看kubectl配置
```bash
kubectl config view
```
- 查看Pod容器中的应用程序日志
```bash
kubectl logs hello-node-xxxxx 
```
- 显示有关资源的详细描述
```bash
kubectl describe (pod) <pod name>
```
- 在Pod中的容器上执行命令
```bash
kubectl exec <command>
```
- 查看Service转发的Pod ip与 端口
```bash
kubectl get endpoints hello-node
```
- 查看replicaSet（维持Pod副本数量的资源）
```bash
kubectl get rs
```
ReplicaSet 是 Deployment 和 Pod 之间的桥梁，平时你不需要直接操作它，但排查问题时很有用。
#### kube使用本地镜像构建deployment
###### 方法一： 修改deployment使用本地镜像策略
```bash
# 修改 Deployment，添加 imagePullPolicy: IfNotPresent
kubectl patch deployment hello-node -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","imagePullPolicy":"IfNotPresent"}]}}}}'
# 查看变化（Pod 会重建）
kubectl get pods --watch
```
###### 方法二：指定镜像拉取策略
```bash
kubectl create deployment hello-node --image=nginx:latest --dry-run=client -o yaml > hello-node.yaml

# 编辑文件，添加 imagePullPolicy
sed -i '/image: nginx:latest/a\        imagePullPolicy: IfNotPresent' hello-node.yaml

# 应用配置
kubectl apply -f hello-node.yaml

# 查看状态
kubectl get pods --watch
```
#### Service
- 创建service
```bash
kubectl expose deployment hello-node --type=xxxx --port=xxxx --target-port=xxxx 
```
port是Service向外暴露的端口
targetport是内部pod服务真正使用的端口
type有四种类型

| 类型                | 说明        | 访问范围    | 适用场景     |
| ----------------- | --------- | ------- | -------- |
| **ClusterIP**（默认） | 集群内部 IP   | 仅集群内部   | 内部服务调用   |
| **NodePort**      | 节点上的端口    | 集群外部可访问 | 开发测试环境   |
| **LoadBalancer**  | 云服务商负载均衡器 | 公网可访问   | 生产环境（云上） |
| **ExternalName**  | 映射到外部域名   | 集群内部    | 访问外部服务   |
- 启动service
```bash
minikube service hello-node
```
#### 插件
- 查看所有可用插件
```bash
minikube addons list
```
- 启动一个插件
```bash
minikube addons enable metrics-server
```
- 查看插件安装的资源
```bash
kubectl get pod,svc -n kube-system | grep metrics-server
```
-n 指的是kube-system这个命名空间里面的
- 禁用插件
```bash
minikube addons disable metrics-server
```
- 使用插件（metrics-server）
```bash
minikube top pod
```
#### 清理
- 清理集群中创建的资源
```bash
kubectl delete service hello-node
kubectl delete deployment hello-node
```

#### 使用代理
```bash
kubectl proxy
```
在 8001 端口运行
一旦代理启动，就可以通过 API 来访问集群中的各种资源了。
proxy是用来访问集群信息，而不是服务资源
用于程序员调试写脚本等等用途

#### 在容器上面运行命令
```bash
kubectl exec -it <Pod名称> -- /bin/sh
```
-i 交互模式（STDIN打开）
-t 分配伪终端（能输入命令）
-- 分割符，后面是容器内要执行的命令

exec是pod层面的命令

#### 扩缩应用
通过在使用 kubectl create deployment 命令时设置 --replicas 参数， 你可以在启动 Deployment 时创建多个实例。
- 将deployment扩容到三个
```bash
kubectl scale deployments/kubernetes-bootcamp --replicas=3
```
##### 负载均衡
```bash
# 同时查看所有 3 个 Pod 的日志（需要开三个终端窗口，或用 tmux）
# 终端1：
kubectl logs -f hello-node-xxx-aaa
# 终端2：
kubectl logs -f hello-node-xxx-bbb
# 终端3：
kubectl logs -f hello-node-xxx-ccc
```
在另一个终端持续发送请求
```bash
# 获取 Service 访问地址
minikube service hello-node --url
# 假设输出：http://192.168.49.2:30393
# 循环发送 30 个请求
for i in {1..30}; do curl -s http://192.168.49.2:30393 | grep -i "hostname\|pod" ; sleep 0.5; done
```
观察日志

#### 滚动更新
要查看应用程序当前的镜像版本，可以运行 describe pods 子命令， 然后查找 Image 字段：
```bash
kubectl describe pods
```
要将应用程序的镜像版本更新为 v2，可以使用 set image 子命令， 后面跟着 Deployment 名称和新版本的镜像：
```bash
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=docker.io/jocatalin/kubernetes-bootcamp:v2
```
##### rollout
- 查看更新状态
```bash
kubectl rollout status deployment hello-node
```
- 查看历史版本
```bash
kubectl rollout history deployment hello-node --revision=2 
```
如果你想看某个版本的详细信息，可以加上 --revision 参数。
- 回滚版本
kubectl rollout undo deployment hello-node

##### kubectl set image 工作流程
- 修改deployment的pod模板，更改image字段
- 创建新的ReplicaSet（副本集），他的pod模板指向新版本镜像
- 逐步替换pod
- 保留历史版本
