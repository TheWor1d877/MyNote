#面试-云原生-复习  什么是 CRD？它和自定义控制器如何共同实现声明式 API？
## Controller
#面试-云原生-复习 解释 Informer、InformerFactory、Lister、WorkQueue 在 controller 中的协作机制

k8s中的自动化，本质上就是反复做三件事：观察实际状态 -> 对比期望状态 -> 不一致就手动对齐
这个对比，动手的过程就叫作：reconclie（调谐）
它只关心此刻期望和实际的差距，不关心中间发生了什么，这个特性叫作“水平触发”，相对的是边缘触发：只看变化事件

Controller就是做调谐的控制器组件，它只维护状态，并且每一次调谐都是幂等的，因为controller可能因为各种原因处理同一个对象
#### Controller的组件
Informer： 跟API Server建立Watch连接，第一次先拿全量快照，然后Watch持续接收变更事件，将收到的对象存进本地缓存，并且维护缓存与API Server之间的同步关系

InformerFactory： 负责Infromer的复用逻辑，同一个Factory下，监听所有Pod的同进程Controller复用一个Infromer
```text
Controller A ─┐
Controller B ─┼──► [Factory → Informer] ──► API Server ◄──► pod
Controller C ─┘
```

Lister： 是本地缓存读对象的调用接口，提供相关方法，直接从内存返回对象，零网络调用，零API Server压力

WorkQueue： 解耦Infromer事件速率与Controller处理速率
Informer 收到事件后，EventHandler 把对象的 key（namespace/name）扔进 WorkQueue，然后立刻返回。Controller 有独立的 worker goroutine 从队列里取 key，执行真正的 reconcile 逻辑。

## CRD(Custom Resource Definition)
在 Kubernetes 中，CRD 是一种扩展 API 的机制，允许你定义自己的资源类型，CRD描述一个自定义资源的相关字段，以及字段的类型CRD只做API拓展，但是不拓展行为，行为的拓展需要Controller实现

Controller需要认识资源才能去控制，资源分成内置资源与自定义资源(CRD)，这也就需要CRD先注册

CRD 负责“能声明”，Controller 负责“让声明变成现实”。、

插曲： CRD管 API 扩展，DRA管资源分配

## Operator
#面试-云原生-复习  解释 Kubernetes Operator 模式，以及为什么分布式训练框架常用 Operator 管理。

Operator = CRD + 自定义Controller
用CRD定义一种新资源，用Controller监听这种资源，将他解析为具体的动作，在reconclie循环中翻译成对内置资源的操作

自定义的Cntroller只管理CRD注册的新资源，不管别的内置资源

## 分布式训练框架常用Operator
原生k8s的Job/Deployment表达不了寻不是训练的协同需求，只有Operator采耳脑功能将需求封装到控制循环中
- 角色不对称： Pytorch有Master与Worker，普通Job来自同一个模板，无法表达这种差异
- 容错语义： 普通Pod挂了，只要重启即可，Master挂了可能任务需要重来，Worker挂了看情况，支持弹性的框架可以缩容继续
- 无法在正确的时候将地址塞给正确的Pod：Worker 先启动，Master 后启动，Headless Service 默认只把 Ready 的 Pod 发布到 DNS 里。Master 还没 Ready，所以 DNS 解析结果是空的。Worker 拿不到地址，NCCL 初始化卡死
要解决这个，你得知道去 Service 上加一个配置：publishNotReadyAddresses: true

## 创建CRD与Controller
#### 创建CRD
CRD 允许你向 Kubernetes API 注册全新的资源类型，就像 Deployment 一样原生。
告诉k8s,website是什么东西
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com  
spec:
  group: example.com
  scope: Namespaced  # 资源属于命名空间
  # scope： Cluster  # 资源是集群级别的，不属于任何命名空间
  names:
    plural: websites        # 复数形式，用于 URL: /apis/example.com/v1/websites
    singular: website       # 单数形式
    kind: Website           # Go 结构体风格，用于 YAML 中的 kind 字段
    shortNames:
      - web                 # 简写，支持 kubectl get web
  versions:  
    - name: v1  # 版本号
      served: true  # 是否对外提供此版本的服务
      storage: true # 是否作为持久化存储的版本（只能有一个 true）
      schema: # 数据校验规则
        openAPIV3Schema: # 使用openAPIV3Schema校验规则
          type: object
          required:
           - spec
          properties:
            spec:
              type: object
              required:
               - domain
              properties:
                domain:
                  type: string
                  pattern: "^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$"
                  description: "网站域名"
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
                image:
                  type: string
                  default: "nginx:latest"
                enabled:
                  type: boolean
                  default: true
```
如果用户提交的 YAML 不符合 Schema，Kubernetes API Server 会直接拒绝

CRD 创建完成后，你就可以像使用原生资源一样使用它了。
```yaml
# my-website.yaml
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-blog
  namespace: default
spec:
  domain: "blog.example.com"
  replicas: 3
  image: "nginx:1.25"
  enabled: true
```
上面的 Website 资源创建后，Kubernetes 只是把它存储到 etcd 中，就像存一个 ConfigMap 一样。它不会自动帮你部署 Nginx。

注意CRD只是注册一个新的API类型，不会自动创建Deployment，Service等资源
需要使用Controller来将资源真正生效

Controller的作用：
Watch 监听： 监听CRD资源的创建更新，删除等事件
Reconcile 对比  期望状态与实际状态的对比
Act 执行: 监听到的不同，需要作出更改，创建，更新，删除底层的Deployment，Service等资源
#### 创建Controller
使用kubebuilder来写一个完整的Go语言Controller
创建项目与api
```bash
# domain: API组名后缀 repo: 指定Go模块路径，影响代码导入路径
kubebuilder init --domain example.com --repo example.com/website-operator

# group: 设置API组名 version: API版本号 kind: 资源类型
kubebuilder create api --group example --version v1 --kind Website
```
编写Controller
Controller 就是一个"死循环的搬运工"：盯着你写的 CRD，发现变了就去调 Kubernetes，让实际的东西跟上你的期望。
核心思想：Reconcile（调谐）
Reconcile 的核心逻辑：
期望状态（spec） → 对比 → 实际状态 → 不一致就修正

## CRD高级特性
#### 子资源
CRD 支持启用 status 和 scale 子资源
status与scale子资源本质上是为了让自定义医院在行为上面更像k8s的原生资源


scale 子资源则让自定义资源能够接入标准的扩缩容接口，从而支持 kubectl scale 和水平 Pod 自动扩缩容（HPA）。

- status资源
status 子资源负责将spec(期望状态)与status(当前状态)隔离，防止相互覆盖
主端点负责只接受对spec的修改，忽略status字段，而status 只能通过 /status 端点更新，控制器负责更新，用户不能随意修改。
k8s的系统资源都按照这种方式管理
Deployment 的 status：
```yaml
apiVersion: apps/v1
kind: Deployment
status:
   type: object
   ￼properties:
	 ￼availableReplicas:
	   type: integer
	 ￼conditions:
	   type: array
	   ￼items:
		 type: object
# 启用子资源
  replicas: 3          # 当前副本数
  availableReplicas: 3 # 可用副本数
  conditions:          # 各种状态条件
    - type: Available
      status: "True"
```


- scale资源
scale子资源可以让自定义资源像Deployment一样被扩缩容
启用后，k8s 会为该 CRD 暴露一个 /scale 端点，返回标准的 autoscaling/v1 Scale 对象。这个对象只包含 replicas 字段，不涉及你自定义资源中其他复杂的 spec 字段。

只需要告诉API Server，自定义资源中那个字段对应“期望副本数”，哪个字段对应“实际副本数”。
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: website-hpa
spec:
  scaleTargetRef:
    apiVersion: example.com/v1
    kind: Website
    name: my-site          # ← HPA 控制这个 Website
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          averageUtilization: 80
```

## CRD资源创建的对象
[[Container/Kubernetes/基础概念/CRD实例的声明方式|CRD实例的声明方式]]