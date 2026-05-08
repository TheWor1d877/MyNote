**Kubernetes 对象**是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。

Kubernetes 对象是一种“意向表达（Record of Intent）”。一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。通过创建对象，你本质上是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的期望状态（Desired State）。

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 spec（规约） 和对象 status（状态）。 对于具有 spec 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： 期望状态（Desired State）。

Kubernetes 对象就是你告诉 K8s "我想要什么" 的方式，K8s 负责帮你实现。
K8s 的日常工作：不断对比 spec 和 status，如果不一致，就去修复。

## 必须字段
在想要创建的 Kubernetes 对象所对应的清单（YAML 或 JSON 文件）中，需要配置的字段如下：
- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一标识对象的一些数据，包括一个 `name` 字符串、`UID` 和可选的 `namespace`
- `spec` - 你所期望的该对象的状态


## 服务器字段验证
当你 `kubectl apply` 时，K8s 会检查你的 YAML 文件有没有写错字段：

从 Kubernetes v1.25 开始，API 服务器提供了服务器端[字段验证](https://kubernetes.io/zh-cn/docs/reference/using-api/api-concepts/#field-validation)， 可以检测对象中未被识别或重复的字段。它在服务器端提供了 `kubectl --validate` 的所有功能。

`kubectl` 工具使用 `--validate` 标志来设置字段验证级别。它接受值 `ignore`、`warn` 和 `strict`，同时还接受值 `true`（等同于 `strict`）和 `false`（等同于 `ignore`）。`kubectl` 的默认验证设置为 `--validate=true`。

`Strict`
严格的字段验证，验证失败时会报错

`Warn`
执行字段验证，但错误会以警告形式提供而不是拒绝请求

`Ignore`
不执行服务器端字段验证

