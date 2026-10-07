初始栈为2kb，按需增长
创建成本低，用户态调度
数量上限是百万级别
GO运行时 M:N调度器调度，而不是OS内核
使用CSP模型通信
你不能强制终止一个 goroutine —— 必须让它自己退出。
## CSP
 Communicating Sequential Processes（通信顺序进程）
 不要通过共享内存来通信，而应该通过通信来共享内存
#### 核心组件
- Process
- Channel
- Communication

## 注意
1. 不要裸写无限循环goroutine(泄漏)
```go
func main() {
    ch := make(chan bool)
    go func() {
        fmt.Println("working...")
        time.Sleep(time.Second)
        ch <- true // 阻塞，直到主 goroutine 接收
    }()
    <-ch // 等待子 goroutine 完成
    fmt.Println("done")
}
```
2. Goroutine 中的变量捕获（闭包陷阱）
```go
go func(i int) {
    fmt.Println(i)
}(i)
```

