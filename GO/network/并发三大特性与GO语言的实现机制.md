原子性：操作不可分割，中间状态对外不可见
可见性：一个 goroutine 对内存的修改，其他 goroutine 能立即看到
顺序性：程序执行顺序符合预期（避免重排序）

## 原子性
sync/atomic 包（无锁原子操作）
```go
var x int64

// 安全自增
atomic.AddInt64(&x, 1)

// 安全读取
val := atomic.LoadInt64(&x)
```
支持 int32/64, uint32/64, uintptr, unsafe.Pointer
性能极高（CPU 原子指令，如 CAS）

普通读可能读到缓存里的旧值、可能被编译器优化掉；atomic 读带 acquire 语义，建立 happens-before,且禁止重排。

## 可见性
#面试-Go语言-复习 解释 Go 的内存模型，happens-before 关系如何保证并发可见性。
- CPU 有缓存，编译器会优化。
- Goroutine A 修改了变量，Goroutine B 可能**永远看不到新值**（读的是寄存器或旧缓存）。

happens-before：如果事件 A happens-before 事件 B,那么 A 的写对 B 可见,而且 A 和 B 的执行顺序在观察者看来是 A 先于 B。注意它不是"时间上谁先发生",而是一种由同步操作建立的偏序关系。

#### 常见的happens-before规则
- 同一 goroutine 内,按程序顺序，前面的语句 happens-before 后面的。
- channel：对 channel 的发送 happens-before 对应的接收完成；channel 关闭 happens-before 接收方收到零值。
- sync.Mutex：Unlock happens-before 之后任何一次 Lock。
- sync.WaitGroup：Wait 返回 happens-before 之前所有 Done。
- goroutine 启动：go f() 这句话 happens-before f 里的第一句执行。

#### 实现原理
本质是同步操作会插入内存屏障（memory barrier）,屏障会阻止编译器重排、也会让 CPU 把缓存刷到一致状态。

happens-before 不是靠"加锁就万事大吉",而是两个 goroutine 必须在同一个同步原语上相遇（比如同一个 channel、同一把锁），才会建立这条边。

## 顺序性
Go 内存模型规定：**所有同步点（sync、channel 等）都是内存屏障（Memory Barrier）**，禁止跨越这些点的重排序。
```go
var a, b int
var done chan bool = make(chan bool)

// Goroutine 1
a = 1
b = 2
done <- true

// Goroutine 2
<-done
println(a, b) // 一定输出 "1 2"，不会是 "0 2" 或 "1 0"
```

