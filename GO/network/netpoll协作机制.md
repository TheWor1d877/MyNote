Go 的网络模型 = 用户态 goroutine + 内核 epoll
GO使用少量线程就能高效处理成千上万的连接
## 传统IO模型的瓶颈
- `conn.Read()` 会**真正阻塞 OS 线程**
- 1 万连接 → 需要 1 万 OS 线程
- 每个线程栈 ～1～8MB → 内存爆炸
- 线程切换开销巨大 → CPU 打满
## GO的做法: 非阻塞IO + 事件驱动 + 用户态调度
底层使用非阻塞socket，所有的net.Conn的fd都设置O_NONBLOCK

Go在runtime内部实现了一个跨平台的IO多路复用器叫做 netpoll
监听IO事件

使用GMP进行调度，当发现G没数据，就暂时挂起，让netpoll监听这个fd，发现可读就再次唤醒

| 特性 | 说明 |
|------|------|
| 位置 | `src/runtime/netpoll.go`（是 runtime 的一部分） |
| 启动方式 | 程序启动时自动创建一个 network poller 线程（通常是 M0） |
| 工作模式 | 后台线程循环调用 `epoll_wait()`，等待 I/O 事件 |
| 用户无感 | 你只写 `conn.Read()`，runtime 自动走 netpoll 流程 |

