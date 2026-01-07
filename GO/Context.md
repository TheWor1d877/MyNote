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

## 理解
#### Context的核心目标
在goroutine树中实现取消与超时的<span style="color:rgb(221, 85, 85)">跨层级</span>传播

#### 核心接口
```go
type Context interface {
	Done() <-chan struct{}
	Err() error
	Deadline() (deadline time.Time, ok bool)
	Value(key interface{}) interface{}
}
```

#### 四种实现
###### emptyCtx
```go
//An emptyCtx is never canceled, has no values, and has no deadline.
// It is the common base of backgroundCtx and todoCtx.
type emptyCtx struct{}
func (emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}
func (emptyCtx) Done() <-chan struct{} {
	return nil
}
func (emptyCtx) Err() error {
	return nil
}
func (emptyCtx) Value(key any) any {
	return nil
```

###### cancelCtx
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value        
	//Done() 返回的channel
	children map[canceler]struct{} // set to nil by the first cancel call
	err      atomic.Value          // 取消原因
}
```
调用cancel的时候，关闭done channel(通知所有的监听者)
遍历children并且调用他们的cancel()

###### timerCtx
```go
type timerCtx struct {
    cancelCtx        // 嵌入 cancelCtx（复用其取消逻辑）
    timer *time.Timer // 超时定时器
    deadline time.Time
}
```
在cancelCtx的基础上面增加一个time.Timer
当定时器触发的时候，自动调用cancel()
如果提前手动取消 -> 停止计时器

###### valueCtx
```go
type valueCtx struct {
	Context
	key, val interface{}
}
```
重写Value() 方法，现查找自己的key，没有就递归遍历父Context
形成一条单项链表
适合传入userID,traceID等元数据

#### 注意：
###### Done
Done() channel 是 <- chan struct{}
类型的原因：
struct{}是零字节类型，关闭它只传递信号，不传递数据
接收方使用select监听
```go
select {
case <- ctx.Done():
	return .....
case result := <- workCh:
	//正常处理
}
```

###### cancel
context 提供取消信号
goroutine必须能监听这个信号，如果当前正在执行一个不相应context的阻塞操作(纯计算或者未包装的syscal)，就不会中断
应用：
在实现 Raft 日志 apply 时，如果 apply 是磁盘密集型（如写 WAL），必须分段检查 ctx.Done()；
否则 leader 切换后，旧 leader 的 apply goroutine 会继续写入，造成状态不一致！

###### Context超时 vs 网络超时
应用层 请求总耗时 context.WithTimeOut(parent.5s)
RPC层 单次网络调用 RPC的CallOption
Raft层 选举超时 time.Ticker + select监听ctx.Done()

常见问题：
超时嵌套导致过早失败
解决方法： 让子context共享父亲deadline
```go
deadline, ok := ctx.Deadline()
if !ok {
	deadline := time.Now().Add(800 * time.Millisecond)
}
rpcCtx, _ := context.WithDeadline()
```
go的http.NewRequestWithContext
