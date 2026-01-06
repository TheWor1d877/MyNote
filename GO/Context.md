context 是 Go 云原生、微服务、HTTP/gRPC 开发中必备的核心包，用于在 goroutine 树中传递取消信号、超时、截止时间、请求元数据（如 traceID）。

## Context 的四大核心能力

| 能力                    | 函数                       | 用途                           |
| --------------------- | ------------------------ | ---------------------------- |
| **1. 取消（Cancel）**     | `context.WithCancel()`   | 主动通知子任务停止                    |
| **2. 超时（Timeout）**    | `context.WithTimeout()`  | 固定时间内自动取消                    |
| **3. 截止时间（Deadline）** | `context.WithDeadline()` | 到指定时间点自动取消                   |
| **4. 携带值（Value）**     | `context.WithValue()`    | 传递请求上下文（如 user ID, trace ID） |

## 最佳实践
1. 所有入口接收ctx作为第一个参数
```go
func (s *Service) GetUser(ctx context.Context,id string)(*User,error)
```
2. 不要把Context存到Struct中
```go
//错误
type Service struct {
    ctx context.Context // 危险！生命周期不匹配
}
```
3. HTTP/gRPC中自动携带context
```go
http.Request.Context() 获取请求级 context
gRPC 的 ctx 自动包含 metadata、超时等
```

