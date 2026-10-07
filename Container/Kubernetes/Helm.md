## YAML Hell问题
在 Helm 出现之前，部署一个复杂应用到 Kubernetes 需要：
- 编写 Deployment.yaml, Service.yaml, Ingress.yaml, ConfigMap.yaml, Secret.yaml 等多个文件。
- 不同环境（dev/staging/prod）需要维护多套几乎相同的 YAML，仅参数不同。
- 升级或回滚时，手动追踪变更并重新 kubectl apply，极易出错。
- 缺乏版本控制和依赖管理。
Helm 的本质，是为了解决上述问题而诞生的<span style="color:rgb(221, 85, 85)"> 声明式应用打包和生命周期管理工具</span>

## 核心概念
- Chart： 一个Helm应用包，包含预配置的k8s资源清单模板的目录或者压缩包
- Repository： Chart的存储仓库，从公共仓库或者私有仓库获得Chart
- Release： Chart 在 Kubernetes 集群中的一个运行实例，Release会被Helm追踪他的版本历史

Helm 会将每个 Release 的所有版本信息（包括完整的渲染后清单）以 Secret 的形式存储在集群中（默认在 kube-system 命名空间）。这使得 rollback 操作变得极其可靠和简单。
## 目录结构
```text
my-app-chart/
├── Chart.yaml          # Chart 的元数据 (名称, 版本, 描述, 依赖等)
├── values.yaml         # 默认配置值
├── charts/             # 依赖的子 Chart (通过 `helm dependency build` 下载)
├── crds/               # 自定义资源定义 (CRD)，Helm 会先安装它们
├── templates/          # 核心！所有 Kubernetes 清单模板都放在这里
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl    # 命名模板 (partials)，用于复用逻辑
└── README.md           # 使用说明
```

## 常用命令
| 命令 | 作用 |
| :--- | :--- |
| `helm search hub <keyword>` | 在 [Artifact Hub](https://artifacthub.io/) 上搜索公共 Chart |
| `helm repo add <name> <url>` | 添加一个 Chart 仓库 (如 `helm repo add bitnami https://charts.bitnami.com/bitnami`) |
| `helm install <release-name> <chart>` | 安装一个 Release |
| `helm upgrade <release-name> <chart>` | 升级一个已存在的 Release |
| `helm rollback <release-name> <revision>` | 回滚到指定的历史版本 |
| `helm list` | 列出当前命名空间下的所有 Release |
| `helm status <release-name>` | 查看 Release 的详细状态和历史 |
| `helm uninstall <release-name>` | 卸载一个 Release |
| `helm template <release-name> <chart>` | 本地渲染模板，不提交到集群。用于调试和 CI/CD 预览。 |

## 高级特性
1. 依赖管理
复杂的 Chart 往往依赖其他 Chart
执行 helm dependency update 后，依赖的 Chart 会被下载到 charts/ 目录下。Helm 会按依赖关系图自动处理安装顺序。
2. Hooks
Helm 提供了强大的 Hook 机制，允许你在 Release 生命周期的特定时间点执行 Job 或其他资源
3. Library Charts
这是一种特殊的 Chart，不包含任何可部署的 Kubernetes 资源，只包含可在其他 Chart 中复用的模板和函数
4. OCI Registry 支持
现代 Helm 支持将 Chart 推送到任何兼容 OCI (Open Container Initiative) 的镜像仓库（如 Harbor, AWS ECR, Docker Hub）。这统一了应用制品（Docker 镜像 + Helm Chart）的存储和分发

## 使用Helm
#### 安装
| 特性 | 模式一：直接远程安装 | 模式二：本地修改后安装 |
| :--- | :--- | :--- |
| 适用场景 | 90% 的情况：只需改配置（values） | 需要改模板逻辑（如增删资源、改标签） |
| 是否需要本地文件 | ❌ 不需要 | ✅ 需要 |
| 与上游同步 | ✅ 自动（升级时指定新版本） | ❌ 手动 merge |
| 命令复杂度 | 简单 (`helm install ... --set`) | 复杂（需管理目录） |
| 类比 Docker | `docker run -e ... image:tag` | `Dockerfile` + `docker build` |
推荐使用模式1：
```bash
## 创建一个自定义 values 文件
cat > my-redis-values.yaml <<EOF
auth:
 enabled: true
 password: "my-strong-password"
architecture: standalone
replica:
 replicaCount: 0 # standalone 模式下不需要副本
EOF

# 安装 Release
helm install my-redis oci://resipostry-1.docker.io/bitnamicharts/redis -f my-redis-values.yaml --create-namespace --namespace redis-test
```

验证状态：
```bash
# 查看 Release 状态
helm status my-redis -n redis-test

# 获取访问信息
kubectl get pods,svc -n redis-test

# 进入 Pod 测试连接
kubectl exec -it my-redis-master-0 -n redis-test -- redis-cli -a my-strong-password
```

## 创建自己的Helm Chart
```bash
helm create my-first-chart
cd my-first-chart
```

- 修改values.yaml
- 本地渲染与调试
永远不要直接部署未经验证的 Chart！
```bash
# 渲染模板，输出到 stdout
helm template my-app . --debug

# 或输出到文件检查
helm template my-app . > rendered-manifests.yaml
```
- 部署到集群
```bash
helm install my-app . --create-namespace --namespace my-app-test
```

## 两种安装模式
模式一: 直接从远程仓库安装（最常用）
这是 生产环境和快速实验的首选方式。你不需要手动管理 Chart 文件。
```bash
# 1. 添加仓库（只需一次）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 2. 直接安装，并通过 --set 或 -f 覆盖配置
helm install my-redis bitnami/redis \
  --namespace redis-demo \
  --create-namespace \
  --set auth.enabled=false \
  --set architecture=standalone
```
关键点
bitnami/redis 是一个 远程引用，Helm 会在安装时自动在后台下载、缓存并渲染该 Chart。
你不需要 helm pull 或 helm fetch。
所有自定义配置通过 --set（命令行）或 -f custom-values.yaml（文件）传入。
产生的 Release 完全独立，其配置快照会被 Helm 存储在集群中（作为 Secret），用于后续的 upgrade 和 rollback。


模式二：先拉取到本地，再修改（适合深度定制）
当你需要大规模修改模板逻辑（而不仅仅是改 values）时，才需要此模式。
```bash
# 1. 将远程 Chart 拉取到本地
helm pull bitnami/redis --untar
# 生成一个名为 `redis` 的目录

# 2. 进入目录，修改 templates/ 下的文件或 values.yaml
cd redis
vim values.yaml
# 或 vim templates/deployment.yaml

# 3. 从本地目录安装
helm install my-redis-local . \
  --namespace redis-demo \
  --create-namespace
```
关键点
此模式下，你拥有了 Chart 的完整源码，可以做任意修改。
缺点：你脱离了上游 Chart 的更新轨道。未来 Bitnami 发布安全修复，你需要手动同步代码。
