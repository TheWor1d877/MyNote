#面试-Go语言-复习 7. 解释 Go netpoll 与 epoll 的关系，以及网络 I/O 就绪后如何唤醒 G。
Go 的网络模型 = 用户态 goroutine + 内核 epoll
GO使用少量线程就能高效处理成千上万的连接
## 传统IO模型的瓶颈
- `conn.Read()` 会**真正阻塞 OS 线程**
- 1 万连接 → 需要 1 万 OS 线程
- 每个线程栈 ～1～8MB → 内存爆炸
- 线程切换开销巨大 → CPU 打满
## GO的做法: 非阻塞IO + 事件驱动 + 用户态调度
epoll是Linux的IO多路复用接口，epoll只告诉那些fd就绪了，不管读数据跟线程调度方式，这是Go的netpoll实现的

netpoll封装的 epoll，而且用的是 ET
在启动的时候创建一个epoll实例，然后将所有被Go runtime接管的fd都以非阻塞注册进去，也就是说所有的net.Conn的fd都设置O_NONBLOCK（使用阻塞，M就会被卡在一个fd上面，M的数量是有限的）

当read的时候：
如果数据包不到，不会阻塞线程，而是直接返回EAGAIN
这时候runtime把当前这个 G 挂起，从 M 上摘下来，让 M 去跑别的 G。
此时G的状态是挂起并等待

数据到了，内核让 epoll_wait 返回，某个 M 会拿到这批就绪的 fd。M根据一张 fd 到 G 的映射表反查出来G，然后把它重新放回调度队列，标记成 runnable，等着被某个 P 捡起来继续跑。

这张表由 netpoll 管理，访问要加锁或者用原子操作；多个 M 可能同时从 epoll_wait 拿到就绪事件，都会去查这张表、改 G 的状态，所以同步是必须的。

| 特性 | 说明 |
|------|------|
| 位置 | `src/runtime/netpoll.go`（是 runtime 的一部分） |
| 启动方式 | 程序启动时自动创建一个 network poller 线程（通常是 M0） |
| 工作模式 | 后台线程循环调用 `epoll_wait()`，等待 I/O 事件 |
| 用户无感 | 你只写 `conn.Read()`，runtime 自动走 netpoll 流程 |

