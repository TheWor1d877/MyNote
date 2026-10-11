## Goroutine泄漏
#面试-Go语言-复习 什么是 goroutine 泄漏？在高并发推理网关中如何排查和避免？

Goroutine 泄漏：指一个 Goroutine 因被永久阻塞或陷入无限循环，而无法正常退出，导致其占用的资源（栈内存、调度器槽位）无法被回收
Go的runtime不会回收一个还在运行的runtime，只要它没退出，它就一直在那儿。

原因：

| 场景                | 代码示例                                                                                            | 原因               |
| ----------------- | ----------------------------------------------------------------------------------------------- | ---------------- |
| Channel 阻塞        | ```ch := make(chan int); go func(){ <-ch }()```                                                 | 无发送者，接收者永久阻塞     |
| WaitGroup 忘记 Done | ```wg.Add(1); go func(){ /* 忘记 wg.Done() */ }()```                                              | `wg.Wait()` 永不返回 |
| Context 未取消       | ```ctx, cancel := context.WithCancel(context.Background()); go worker(ctx); // 忘记调用 cancel()``` | Worker 永不退出      |
| 网络/IO没有设置超时       |                                                                                                 | gRPC请求的调用对端一直不返回 |
- 产生的危害
内存压力，调度器压力
#### 排查goroutine
- 先查看监控指标
runtime.NumGoroutine() 是最好用的入口，把它打到监控里，看趋势
不能只看单一指标，查看比如goroutine 的栈聚合、按时间点做 diff、或者加打点统计某类 goroutine 的存活时长
- 采集goroutine
```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```
pprof抓的是栈快照，而且是抽样测试，在生产环境下开销比较低，但是也不能长时间高频使用，pprof 端口不能对公网暴露，得走内网或者加鉴权

- 分析问题
终极口诀（针对 goroutine 泄漏场景）

| 情况 | `flat` | `cum` | 含义 |
|------|--------|--------|------|
| 正常工作 | >0 | ≥flat | goroutine 在积极执行代码 |
| 永久阻塞（如 `<-ch` 没人发） | 0 | >0 | goroutine 卡住了，泄漏了！ |
| 已退出 | 不出现 | 不出现 | 安全 |

#### 避免goroutineleak
凡是会阻塞的操作都给一个退出路径
channel 操作用 select 配合 ctx.Done()；网络调用统一带 context 超时

#### 网关中的具体场景
网关一个请求进来，可能fan-out出多个goroutine，如果某个客户端中途断连，context 应该被 cancel，所有子 goroutine 得跟着退出。所以网关里context 传递必须贯穿整条调用链

## goroutineleak 实验特性（Go 1.26+）
这是 Go 官方团队提出的自动化泄漏检测方案，原理非常巧妙！
核心思想：利用 GC 的可达性分析
> 如果一个 Goroutine 阻塞在一个同步原语（如 Channel、Mutex）上，
> 而这个同步原语本身已经不可达（没有其他存活的 Goroutine 能访问它），
> 那么这个 Goroutine 就是泄漏的！

如何启用？
构建时开启实验特性：
```go
GOEXPERIMENT=goroutineleakprofile go build -o myapp .
```

程序中触发一次带泄漏检测的 GC：
```go
import "runtime"

// 在你想检测的时候调用
runtime.GC() // 这次 GC 会运行泄漏检测逻辑
```

通过新端点获取结果：
```go
# 获取自动检测出的泄漏 Goroutine 列表
go tool pprof http://localhost:6060/debug/pprof/goroutineleak
```


