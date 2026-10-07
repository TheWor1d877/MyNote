核心作用
不是锁，而是一个计数信号量。用于等待一组 goroutine 完成它们的工作。
关键方法
Add(delta int): 增加（或减少）内部计数器。通常在启动 goroutine 前调用。
Done(): 将计数器减 1。通常在 goroutine 的末尾调用。
Wait(): 阻塞，直到计数器变为 0。

使用示例

```go
func main() {
    var wg sync.WaitGroup
    urls := []string{"a.com", "b.com", "c.com"}

    for _, url := range urls {
        wg.Add(1) // 在启动 goroutine 前增加计数
        go func(u string) {
            defer wg.Done() // 确保 goroutine 结束时计数减一
            fetch(u)
        }(url)
    }

    wg.Wait() // 主 goroutine 在此等待所有任务完成
    fmt.Println("All fetches done.")
}
```

关键点：
Add() 必须在对应的 goroutine 启动之前调用，否则可能导致竞态条件。
WaitGroup 是可重用的，但必须等 Wait() 返回后才能再次使用。