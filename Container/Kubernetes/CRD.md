在 Kubernetes 中，CRD 是一种扩展 API 的机制，允许你定义自己的资源类型。

比如: 
Ingress 是k8s内置资源
但是IngressRoute不是，他是CRD
常见的 CRD 使用者：Prometheus Operator（Prometheus、Alertmanager 资源）、Istio（VirtualService、Gateway）、ArgoCD（Application）等。

## Helm对于CRD的处理方式

| 特性   | 普通资源（Deployment 等） | CRD               |
| :--- | :----------------- | :---------------- |
| 升级时  | 会被更新/替换            | 默认不更新（防止数据丢失）     |
| 卸载时  | 会被删除               | 默认不删除（防止数据丢失）     |
| 安装顺序 | Helm 自动排序          | 必须先于使用该 CRD 的资源创建 |

将CRD的yaml文件放到crds/文件夹下面
```text
my-chart/
├── Chart.yaml
├── values.yaml
├── crds/
│   └── my-crd.yaml
└── templates/
    └── ...
```
Helm会在安装Release之前创建所有CRD

crds目录下面是纯yaml，不会被模板语法渲染

## 创建CRD并使用
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
## 创建Controller
使用kubebuilder来写一个完整的Go语言Controller
创建项目与api
```bash
kubebuilder init --domain example.com --repo example.com/website-operator

kubebuilder create api --group example --version v1 --kind Website
```
编写Controller
Controller 就是一个"死循环的搬运工"：盯着你写的 CRD，发现变了就去调 Kubernetes，让实际的东西跟上你的期望。
核心思想：Reconcile（调谐）
Reconcile 的核心逻辑：
期望状态（spec） → 对比 → 实际状态 → 不一致就修正
```text
┌─────────────────────────────────────────────────────────────┐
│                      Reconcile 循环                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 获取 Website 实例                                        │
│     └─ 如果不存在（已被删除）→ 返回                          │
│                                                              │
│  2. 检查 Enabled 字段                                        │
│     └─ 如果 false → 删除 Deployment + Service → 返回         │
│                                                              │
│  3. 构造期望的 Deployment → 对比实际 → 创建/更新            │
│                                                              │
│  4. 构造期望的 Service → 对比实际 → 创建/更新               │
│                                                              │
│  5. 读取 Deployment 状态 → 更新 Website.Status              │
│                                                              │
│  6. 返回，等待下一次事件触发                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```
## CRD高级特性
#### 子资源
CRD 支持启用 status 和 scale 子资源
```yaml
versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas:
                type: integer
          status:
            type: object
            properties:
              availableReplicas:
                type: integer
              conditions:
                type: array
                items:
                  type: object
    # 启用子资源
    subresources:
      status: {}  # 启用 status 子资源
      scale:      # 启用 scale 子资源（支持 kubectl scale）
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.availableReplicas
```

- status资源
Deployment 的 status：
```yaml
apiVersion: apps/v1
kind: Deployment
status:
  replicas: 3          # 当前副本数
  availableReplicas: 3 # 可用副本数
  conditions:          # 各种状态条件
    - type: Available
      status: "True"
```
status 只能通过 /status 端点更新，控制器负责更新，用户不能随意修改。
写入之后，控制器可以只更新status，不影响spec

- scale资源
用于scale扩缩容
HPA（HorizontalPodAutoscaler） 就是通过 /scale 子资源自动调整副本数的。
HPA通过scale资源传来的数据进行扩缩容
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
如果没有scale 子资源,HPA 想改副本数，但它不知道：
你的资源有没有副本数？
副本数在哪个字段？（叫 replicas 还是 size 还是 count？）
怎么获取当前实际副本数？

HPA 是一个通用组件，它不能为每种自定义资源写特殊逻辑。
