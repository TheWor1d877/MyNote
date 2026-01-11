## 行为
如果有多个case可以执行 -> 随机选择一个
如果没有case，有default -> 执行default
如果没有case，没有default -> 当前goroutine阻塞，直到某个channel可以执行

## 原理
1. 收集所有的case，将select语句转换为一个scase数组
2. 为了公平性，打乱case
3. 遍历每个channel,尝试不加锁的快速检查是否可读可写
4. 如果都不可用，进入阻塞状态，如果有可用的就立即执行
关键点： 一个G同时在多个channel中的等待队列中，必须保证只被唤醒一次次，其他队列能安全的移除它
go使用sudog和原子操作来解决

## 注意
nil channel在select中永远不会就绪
nil channel 被设置为一种禁止的状态

## select 与 epoll的关系
select与epoll都是多路复用，但是他们工作在不同的层及上面
二者都是采用了事件驱动 + 阻塞等待的方式,而不是轮询

epoll是操作系统内核提供的IO多路复用机制，用于高效监听文件描述符的可读可写事件，调用线程的时候会阻塞在内核态，直到有fd就绪

select 是用户态的并发原语，用于在多个channel之间做选择,不操作文件描述符

Go的网络库，底层使用了epoll(Linux)与kqueue(macos)，来实现高效的网络IO，select用于goroutine之间的用户态协调

epoll 让go能高效处理百万连接，select让go能优雅的的编排调度这些连接诶上面的业务逻辑
