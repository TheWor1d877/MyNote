## Goroutine泄漏
Goroutine 泄漏：指一个 Goroutine 因被永久阻塞或陷入无限循环，而无法正常退出，导致其占用的资源（栈内存、调度器槽位）无法被回收

| 场景                | 代码示例                                                                                            | 原因                 |
| ----------------- | ----------------------------------------------------------------------------------------------- | ------------------ |
| Channel 阻塞        | ```ch := make(chan int); go func(){ <-ch }()```                                                 | 无发送者，接收者永久阻塞       |
| WaitGroup 忘记 Done | ```wg.Add(1); go func(){ /* 忘记 wg.Done() */ }()```                                              | `wg.Wait()` 永不返回   |
| Context 未取消       | ```ctx, cancel := context.WithCancel(context.Background()); go worker(ctx); // 忘记调用 cancel()``` | Worker 永不退出        |
| Mutex 死锁          | ```mu.Lock(); mu.Lock() // 同一线程二次加锁```                                                          | 自己锁死自己             |
| 无限循环无退出           | ```go func(){ for { time.Sleep(time.Second) } }()```                                            | 没有 break/return 条件 |
- 产生的危害
内存压力，调度器压力
- 采集goroutine
```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```
- 分析问题
终极口诀（针对 goroutine 泄漏场景）

| 情况 | `flat` | `cum` | 含义 |
|------|--------|--------|------|
| 正常工作 | >0 | ≥flat | goroutine 在积极执行代码 |
| 永久阻塞（如 `<-ch` 没人发） | 0 | >0 | goroutine 卡住了，泄漏了！ |
| 已退出 | 不出现 | 不出现 | 安全 |

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


