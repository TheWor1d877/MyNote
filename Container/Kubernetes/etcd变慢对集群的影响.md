当 etcd 变慢时，最先受到影响的通常是 API Server 的读写请求，随后影响 Controller、Scheduler 等控制面组件，最终影响集群的调度和资源管理能力。

## 三个层次
第一层：API Server 响应变慢。
- 创建、更新、删除 Pod 等操作耗时增加。
- kubectl 请求出现超时。
- API Server 的请求延迟升高，部分请求返回 5xx。

第二层：Controller 和 Scheduler 出现异常。
- Controller 无法及时更新资源状态。
- Scheduler 获取 Pod、Node 信息变慢。
- 新创建的 Pod 长时间处于 Pending。
- Deployment 扩缩容、故障恢复变慢。

第三层：集群控制面逐渐失去响应能力。
- Node Lease 续约延迟，可能导致 Node 被误判为 NotReady。
- Pod 状态长时间无法更新。
- 故障恢复和自动调度能力下降。
- API Server 可能出现大量请求堆积。

```
etcd 延迟升高
     |
     v
API Server 读写请求变慢
     |
     +----> Controller 调谐变慢
     |
     +----> Scheduler 调度变慢
     |
     +----> kubelet 状态更新变慢
     |
     v
集群控制面逐渐失去响应能力
```
