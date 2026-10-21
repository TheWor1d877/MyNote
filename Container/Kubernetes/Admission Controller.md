#面试-云原生-复习 什么是 Admission Controller？Validating 与 Mutating 的先后顺序是什么？
注： 这里的Controller与调谐的那个Controller没有关系

使用kubectl apply一个Pod，这个请求到了kube-apiserver之后，不是直接写进etcd的
先通过一下关卡： 认证，授权，Admission

Admission能看到完整的请求体，也能在对象落盘之前做最后的干预

Admission Controller 就是在这个准入阶段运行的控制器集合，每个控制器只关注自己那一小块逻辑，比如“这个 Pod 的 nodeSelector 合不合法”“这个 namespace 的资源配额还有没有余量”。

控制器分为两类：
1. 内置控制器: 编译在kube-apiserver 里的代码
2. 动态准入控制器: API Server 按配置调用集群内外的外部 HTTP 服务，通过Webhook的方式通知自己的服务代码
3. 
动态准入控制器分为两个：
Mutating   → 变更阶段调用
ValidatingWebhook → 验证阶段调用

先后顺序：MutatingWebhook先，ValidatingWebhook 后

- MutatingWebhook
修改对象，比如给相关字段打上默认值，往Pod中注入sidecar，以及给对象打标签，返回JSON Patch
- ValidatingWebhook
最后判断对象是否合规，返回allowed或者denied与拒绝原因

验证必须看最终要落盘的那个对象。如果 Validating 先跑，它看到的是一个还没被补全默认值、还没被注入 sidecar 的半成品，判断就没有意义了。

