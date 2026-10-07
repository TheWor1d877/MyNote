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

## 可见性
- CPU 有缓存，编译器会优化。
- Goroutine A 修改了变量，Goroutine B 可能**永远看不到新值**（读的是寄存器或旧缓存）。
Go 遵循 **Happens-Before（先行发生）** 内存模型。只要满足以下任一关系，写操作对读操作**一定可见**：

| Happens-Before 规则       | 示例                                              |
| ----------------------- | ----------------------------------------------- |
| **Goroutine 启动**        | 主 goroutine 中的操作 happens-before 新 goroutine 的启动 |
| **Channel 发送/接收**       | 发送 happens-before 接收完成                          |
| **Mutex 解锁/加锁**         | unlock happens-before 后续 lock                   |
| **WaitGroup Done/Wait** | Done() happens-before Wait() 返回                 |

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

