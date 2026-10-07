5. sync.Cond (条件变量)
核心作用
允许 goroutine 等待某个条件变为真，并在条件满足时被唤醒。它建立在一个 Locker（通常是 `*Mutex `或 `*RWMutex`）之上。


关键方法
Wait(): 原子地释放关联的锁并挂起当前 goroutine。当被唤醒时，会重新获取锁。
Signal(): 唤醒一个正在等待的 goroutine。
Broadcast(): 唤醒所有正在等待的 goroutine。


使用示例（生产者-消费者）
```go

var (
    mu   sync.Mutex
    cond = sync.NewCond(&mu)
    jobs []int
)

// 消费者
func consumer(id int) {
    for {
        mu.Lock()
        // 必须在循环中检查条件，防止虚假唤醒
        for len(jobs) == 0 {
            cond.Wait() // 释放 mu 并等待
        }
        job := jobs[0]
        jobs = jobs[1:]
        mu.Unlock()

        fmt.Printf("Consumer %d processed job %d\n", id, job)
    }
}

// 生产者
func producer() {
    for i := 0; ; i++ {
        mu.Lock()
        jobs = append(jobs, i)
        cond.Signal() // 通知一个消费者
        mu.Unlock()
        time.Sleep(time.Second)
    }
}
```


重要：Wait() 必须在循环中调用，以处理“虚假唤醒”（Spurious Wakeup）的情况。