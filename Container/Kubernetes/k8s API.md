Kubernetes [控制面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane)的核心是 [API 服务器](https://kubernetes.io/zh-cn/docs/concepts/architecture/#kube-apiserver)。 API 服务器负责提供 HTTP API，以供用户、集群中的不同部分和集群外部组件相互通信。

## Discovery API
提供有关 Kubernetes API 的信息：API 名称、资源、版本和支持的操作。 此 API 是特定于 Kubernetes 的一个术语，因为它是一个独立于 Kubernetes OpenAPI 的 API。 
通常是kubectl api-resources使用这个命令

## Open API
Kubernetes 同时提供 OpenAPI v2.0 和 OpenAPI v3.0。OpenAPI v3 是访问 OpenAPI 的首选方法， 因为它提供了对 Kubernetes 资源更全面（无损）的表示。由于 OpenAPI v2 的限制， 所公布的 OpenAPI 中会丢弃掉一些字段，包括但不限于 `default`、`nullable`、`oneOf`。
通常是kubectl explain在使用

## 使用场景

|场景|你需要知道的|
|---|---|
|日常用 kubectl|完全不用管这些，`kubectl` 帮你调好了|
|写 YAML|只用查 `kubectl explain` 或官方文档|
|写 Operator/Controller|需要理解 Discovery API 和 OpenAPI|
|直接 curl API|极少需要，除非做自动化工具的底层|
