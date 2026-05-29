拦截器类似于 Web 框架中的中间件（middleware），可以在不修改业务逻辑的前提下，统一处理请求/响应前后的通用逻辑。

拦截器是被框架注入到调用链中的，它通过闭包持有真正的 handler，并在合适时机决定是否调用它。

拦截器不“传递”数据，而是“包裹”执行流
## 为什么需要拦截器
在分布式系统中，每个 RPC 调用都可能需要：
- 记录调用日志（谁调用了什么？耗时多少？）
- 验证调用者身份（Token 是否合法？）
- 限制 QPS（防止雪崩）
- 上报指标（Prometheus 监控）
- 捕获 panic（避免服务崩溃）
  
如果把这些逻辑写进每个 handler，代码会严重重复且难以维护。
拦截器就是 gRPC 提供的“中间件”机制，实现逻辑复用与解耦

## 服务端一元拦截器（UnaryServerInterceptor）详解
```go
type UnaryServerInterceptor func(
    ctx context.Context,
    req interface{},
    info *UnaryServerInfo,
    handler UnaryHandler,
) (resp interface{}, err error)
```

| 参数 | 作用 |
|------|------|
| `ctx` | 包含 metadata、deadline、cancel 信号 |
| `req` | 已反序列化的请求对象（如 `*PutRequest`） |
| `info` | 包含方法名（`/KVStore/Put`）、服务描述等 |
| `handler` | 真正的业务逻辑函数，调用它才会执行你的 `Put` 方法 |

## 服务端流式拦截器(StreamServerInterceptor)详解
```go
func StreamServerInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error
```
srv interface{} 指向gRPC服务实现结构体（&server{}）
ss,核心对象，指向当前的双向流
info用于判断流类型（纯服务端流？双向流？）
handler执行方法

## 多拦截器链
```go
import "github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors"

myServer := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        AuthInterceptor,      // 1. 认证
        LoggingInterceptor,   // 2. 日志
        PrometheusInterceptor,// 3. 监控
        RecoveryInterceptor,  // 4. Panic 恢复
    ),
)
```

## 参考示例
```go
func LoggingStreamInterceptor(srv interface{}, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
	method := info.FullMethod
	ser := info.IsServerStream
	if ser {
		log.Printf("yes! ServerStream")
	}
	cli := info.IsClientStream
	if cli {
		log.Printf("yes! ClientStream")
	}

	var ClientInfo string = "default"
	if md, ok := metadata.FromIncomingContext(ss.Context()); ok {
		if uas := md["user-agent"]; len(uas) > 0 {
			ClientInfo = uas[0]
		}
	}
	log.Printf("开始流式调用： %s | client %s", method, ClientInfo)

	start := time.Now()
	err := handler(srv, ss)
	duration := time.Since(start)

	if err != nil {
		log.Printf("流式调用结束: %s | 耗时: %v | 错误: %v", method, duration, err)
	} else {
		log.Printf("流式调用成功: %s | 耗时: %v", method, duration)
	}

	return err
}

```
- 在主函数中
```go
	s := grpc.NewServer(
		grpc.StreamInterceptor(LoggingStreamInterceptor),
	)
```

