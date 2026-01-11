## 无缓冲channel
```go
// 创建无缓冲通道
ch := make(chan int)  // 或 make(chan int, 0)
```

特点：发送和接收必须同时准备好（握手同步）
发送方阻塞直到接收方准备好
接收方阻塞直到发送方准备好

## 有缓冲channel
```go
// 创建有缓冲通道，容量为 N
ch := make(chan int, 3)  // 可以缓冲 3 个元素
```
特点：缓冲区未满时发送不阻塞，缓冲区不空时接收不阻塞
用作 任务队列、限流器、信号池 的理想工具。

## channel 底层
底层使用CAS，高效

天然支持阻塞等待

符合Go的CSP并发哲学：
"不要通过共享内存来通信，而要通过通信来共享内存。"
## CAS
Commpare and swap,硬件级别的原子指令，比较并且交换
用于在多个goroutine下面无锁的更新共享变量，整个过程不可分割
如下，sync/atomic包封装了CAS操作：
```go
//非原子操作
count++

// 使用CAS来原子更新count变量count++
old := atomic.LoadInt64(&count)
ok  := atomic.CompareAndSwapInt64(&count,old,old+1)
```
#### CAS高效的原因
无需加锁，避免线程阻塞
乐观并发控制，先假设没有人竞争，失败之后再重试
硬件支持，x86有CMPCHG指令
