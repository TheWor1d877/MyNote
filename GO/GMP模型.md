Go 语言中的 GMP 模型 是 Go 运行时（runtime）用来实现 高并发、高效调度协程（goroutine） 的核心机制。

G： Goroutine，用户级轻量线程
M： Machine/OS Thread,操作系统线程，真正执行代码的实体
P： Processor，逻辑处理器，G与M之间的的调度桥梁
P 是 M 执行 G 的“许可证”——M 必须持有 P 才能运行 G。
## 角色
#### G
是go并发的基本单位
初始栈为2kb，比线程轻量
使用go func() 实现

#### M
对应一个真实的OS线程
负责执行具体的goroutine
M必须绑定P才能执行G，否则就会陷入休眠或者阻塞状态

#### P
数量，默认等于CPU核心数量
每个P维护一个本地的runqueue(可运行的G队列)
本地队列为空的时候尝试从其他P中偷去任务

## 调度流程
- 用户启动一个goroutine
- G 放入某个P的runqueue
- M绑定一个P，从其本地队列中取出G执行
- 如果本地队列为空，
偷取一半G(work-stealing)
- 如果G发生系统调用：
1. 如果是阻塞式syscall，M被阻塞，P与M解绑，交给另一个空闲M继续执行其他G
2. 如果是非阻塞式，不会阻塞M，效率更高

## P的作用
提高调度效率
没有P全局锁竞争严重，有了之后每个P有自己的本地队列，减少锁竞争
能使用working-stealing使得负载均衡
更好的使用多核CPU

## M的创建
1. 有P需要执行G的时候，但是没有空闲的M可用
2. 当某个 M 因阻塞式系统调用（blocking syscall）而被“困住”

## 比喻
P 是一个队列,包含本地 runqueue
M 是机器, 是 OS 线程
G  runnable 时是队列中的元素,运行时或 syscall 时不在队列
 G 自己在执行中主动通知调度器 要进 syscall
 M 在 syscall 返回后尝试抢 P，抢不到休眠
