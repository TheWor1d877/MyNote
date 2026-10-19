#面试-云原生-复习 什么是 Admission Controller？Validating 与 Mutating 的先后顺序是什么？

使用kubectl apply一个Pod，这个请求到了kube-apiserver之后，不是直接写进etcd的
先通过一下关卡： 认证，授权，Admission

Admission能看到完整的请求体，也能在对象落盘之前做最后的干预

Admission Controller 就是在这个阶段运行的Webhook，每个只关注自己那一小块逻辑，比如“这个 Pod 的 nodeSelector 合不合法”“这个 namespace 的资源配额还有没有余量”。

先后顺序：Mutating 先，Validating 后
- Mutating
修改对象，比如给相关字段打上默认值，往Pod中注入sidecar，以及给对象打标签，返回JSON Patch

- Validating
最后判断对象是否合规，返回allowed或者denied与拒绝原因

验证必须看最终要落盘的那个对象。如果 Validating 先跑，它看到的是一个还没被补全默认值、还没被注入 sidecar 的半成品，判断就没有意义了。