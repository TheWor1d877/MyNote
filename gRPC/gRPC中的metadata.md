## Metadata
Metadata（元数据）是 gRPC 在 HTTP/2 头部（Headers）中传递的 key-value 对，用于携带 与业务消息体无关但对通信至关重要的控制信息。

不是请求/响应的一部分（不经过 protobuf 序列化）
跨进程传递（从客户端 → 服务端，甚至透传到下游服务）
轻量高效（基于 HTTP/2 header 压缩）

Go的Context只能在单个进程之内传递信息

metadata的数据结构:`type MD map[string][]string`


## 访问Metadata
推荐在拦截器中访问Metadata
```go
md, ok := metadata.FromIncomingContext(ctx)
或者
md, _ := metadata.FromIncomingContext(ss.Context())
```

## 发送Metadata
- client向server发送
```go
	// 示例：下载文件
	fmt.Println("\n📥 下载文件...")
	rand.Seed(time.Now().UnixNano()) // 添加随机种子
	myKeyValue = fmt.Sprintf("key_%d", rand.Int63())
	md = metadata.Pairs(
		"my_key", myKeyValue,
	)
	ctx = metadata.NewOutgoingContext(context.Background(), md)
	if err := downloadFile(ctx, client, "test.txt", "test_txt/downloaded_test.txt"); err != nil {
		log.Fatalf("下载失败: %v", err)
	}
```
- server向client发送
放到header里面，在业务逻辑之前发送
```go
    // 1. 构造要发送的 metadata
    header := metadata.Pairs(
        "x-storage-node", "node-07",
        "x-file-checksum", "sha256:abc123...",
        "x-file-owner", "alice",
    )

    // 2. 发送 header（必须在第一个 Send() 前调用！）
    if err := stream.SendHeader(header); err != nil {
        return status.Errorf(codes.Internal, "failed to send header: %v", err)
    }

```

## Metadata设计
- 分布式中的多租户隔离

| Metadata Key | 值示例 | 用途 |
|-------------|--------|------|
| `x-tenant-id` | `acme-corp` | 标识租户，用于：<br>- 存储桶命名空间隔离<br>- 配额统计<br>- 审计日志 |

| 维度 | 租户（Tenant） | 用户（User） |
|------|---------------|------------|
| 本质 | 组织/客户实体 | 个人身份 |
| 作用 | 定义数据隔离边界 | 定义操作权限 |
| 数量关系 | 1 个租户 → N 个用户 | 1 个用户 → 通常 1 个租户 |
| 生命周期 | 长期存在（如企业签约） | 可能频繁变动（员工入职/离职） |
| 技术载体 | `x-tenant-id` metadata | `Authorization: Bearer <token>` |
| 存储设计 | 数据表必含 `tenant_id` | 数据表可能含 `user_id`（非必需） |


- 分布式链路追踪

| Metadata Key | 值示例 | 用途 |
|-------------|--------|------|
| `x-trace-id` | `5bd66ef5095369c7b0d1f8f2bd86c5a4` | 全局唯一请求 ID |
| `x-span-id` | `e457b5a2e4d86bd1` | 当前操作 span ID |
- 认证

| Metadata Key | 值示例 | 用途 |
|-------------|--------|------|
| `authorization` | `Bearer <JWT>` | 用户身份凭证 |
| `x-role` | `admin` | （可选）角色信息（通常由 token 解析得出） |
不要在 metadata 中传递密码、密钥等敏感信息！

- 灰度与调试

| Metadata Key | 值示例 | 用途 |
|-------------|--------|------|
| `x-debug` | `true` | 开启详细日志 |
| `x-version` | `v2-canary` | 指定路由到新版本服务 |

