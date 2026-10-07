## ReplicaSet
ReplicaSet 是确保指定数量的 Pod 副本始终运行的控制器。它的职责很单一：

- 数量控制：如果 Pod 挂了，它启动新的；如果太多了，它终止多余的。
- 身份标识：它只管理符合其标签选择器且由它创建的 Pod。

#### Deployment 与 RS 的关系：
Deployment 不直接操作 Pod。Deployment 的每次变更，实际是创建或删除一个 ReplicaSet。
```text
Deployment (v1.0) → 创建 → ReplicaSet (版本A) → 管理 → Pods (镜像:v1)
Deployment (v2.0) → 创建 → ReplicaSet (版本B) → 管理 → Pods (镜像:v2)
```

#### 更新Deployment对ReplicaSet的影响
Deployment 每次更新（包括修复配置）都会创建一个新的 ReplicaSet，但不会删除旧的 ReplicaSet，而是把它缩容到 0。

#### 手动管理ReplicaSet是错误的
ReplicaSet 应该由 Deployment 管理，在任何情况iag下apply的永远都是deployment而不是ReplicaSet。 ReplicaSet应该仅仅用于查看
- 删除最新的ReplicaSet会导致容器一起被删除
- 修改旧版ReplicaSet再提交不会发生任何变化
API Server 接受请求：Kubernetes 会接收你的修改指令，旧 ReplicaSet 的配置确实会被更新。
但是修改会被Deployment控制器立即覆盖
## Revision History
每次 Deployment 更新（如改镜像、环境变量），系统会创建一个新的 ReplicaSet，旧的那个不会立即删除，而是被保留下来。这个旧 RS 就是一个“历史版本”

你可以随时通过 kubectl rollout undo 切换回某个旧 RS，从而实现回滚。
回滚本质是：让新的 RS（当前版本）缩容到 0，让目标历史版本的 RS 扩容到期望数量

存储内容：实际应用过的运行时快照
存储位置： etcd（集群内部）
扩缩容记录： 保留（回滚时不动这些字段）
回滚操作：`kubectl rollout undo`（精准、安全）
审计能力： `kubectl rollout history`直接看

#### 为什么要保留历史版本
如果手动管理，需要自己的一套命名规范

ReplicaSet 历史版本：它记录的是应用该版本时的运行时状态，包括副本数、环境变量等所有运行时配置。当你 kubectl rollout undo 时，它只改变镜像和关联配置，不会动 HPA 或手动扩缩容的设置。

## rollout
- 回滚的策略
回滚操作修改的就是 Deployment 中的 spec.template 字段，也就是 Pod 模板
不会替换其他内容，如ReplicaSet的数量
- 回滚操作
回滚操作需要通过 kubectl 命令行来完成。

#### 使用rollout存在的问题
- 两种类型的命令
声明式 (kubectl apply -f your-file.yaml)：
这是 Kubernetes 推荐的标准方式。
它的原理是，把 your-file.yaml 这个完整的配置文件，原封不动地保存到资源的` kubectl.kubernetes.io/last-applied-configuration` 注解里。
下次你再 apply 时，kubectl 会对比这个注解、集群当前状态和你的新文件，算出最小差异并应用。

命令式 (kubectl rollout undo, kubectl edit, kubectl scale)：
这些是“一次性”命令，直接告诉 Kubernetes “把副本数改成3”或“回滚到上一个版本”。
<span style="color:rgb(221, 85, 85)">它们只管执行动作，完全不会去更新那个 last-applied-configuration 注解。</span>

- 解决办法
接受并更新注解，回滚后，立即用当前的集群状态去更新那个注解。
```bash
# 1. 执行回滚
kubectl rollout undo deployment/nginx-server

# 2. 用当前集群状态"覆盖"回注解，使其同步
kubectl apply --force-conflicts -f - <<EOF
$(kubectl get deployment nginx-server -o yaml --export)
EOF
```

注解是附带的元数据，可能要给工具、控制器、自动系统看的
给外部工具使用的

回滚一般就用于紧急恢复服务使用
