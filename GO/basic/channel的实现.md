#面试-Go语言-复习 解释 Go 的 channel 底层结构，发送、接收、关闭分别做了什么。

Go的channel本质就是一个带锁的环形缓冲区，加上两个等待队列(sendq,recvq)与一把mutex
两个等待队列一个存发数据的goroutine一个存接受数据的goroutine
发送接收关闭都是在操作这三个东西

## 结构
channel底层是hchan，核心字段是环形数组buf存数据，使用sendx与recvx指示读写位置
dataqsiz是缓冲区容量，一把互斥锁 lock，还有两个等待队列 sendq 和 recvq,存的是被阻塞的 goroutine（用 sudog 包起来）。
#### lock
lock保护的是buf，sendx，recvx，两个等待队列，与closed标志
做成无锁的要考虑多消费场景下两个队列 的一致性，还要考虑并发问题，复杂度高，收益不一定够

缓冲的channel：是使用buf数组存储数据，使用sendx与recvx两个索引模容量实现FIFO循环读写

无缓冲的channel：没有buffer，发送和接收必须严格配对才能推进，否则会阻塞自己

#### 发送
然后看 recvq 里有没有正在等的接收者——有的话说明缓冲区是空的，直接把数据拷贝给接收者，唤醒它，走人，根本不进缓冲区。
如果没有等待者就把数据放入缓冲区，移动sendx指针
如果缓冲区满，goroutine就将自己挂在sendq上面，阻塞自己，等待被唤醒

#### 接受
接受与发送是对称的，先看 sendq 里有没有等待的发送者——有的话，直接从发送者那里取数据，如果是满的还要去sendq中唤醒Goroutine
否则看缓冲区有没有数据，有就取；没有就把自己挂到 recvq 上阻塞。

#### 关闭
把 closed 标志位置 1,然后把 recvq 里所有等待的接收者全部唤醒——它们会拿到零值加一个 ok=false;接受方永远不painc，关闭后仍然能读取数据，读完数据再读读到的ok=false,v=0
`v, ok := <-ch`

再把 sendq 里等待的发送者全部唤醒，但它们会 panic,因为往已关闭的 channel 发数据是运行时错误。

## 补充：select实现
```go
select {
case v := <-ch1:
    // ch1 有数据
case ch2 <- 42:
    // ch2 能发出去
default:
    // 上面都不满足，立刻走这
}
```
哪个 case 的 channel 操作能立刻完成，就走哪个；如果多个都能走，随机选一个；如果一个都不能走，看有没有 default,有就走 default,没有就阻塞等。

1. 加锁是所有channel按照随机顺序查看是否有能走的
2. 如果全都不就绪，就将自己goroutine挂载到所有channel的等待队列上面，然后阻塞自己
3. 某个 channel 有动静了（比如别的 goroutine 往 ch1 发了数据），它会从 ch1 的 recvq 里唤醒我们这个 sudog。goroutine 被唤醒后，得把所有 channel 上挂的自己的 sudog 全部摘掉——因为现在只走了一个 case，其他 channel 上还挂着我呢，不摘就是悬空引用。

#### sudog：实现同时等多个channel的基础
一个 goroutine 如果阻塞在一个 channel 上，它就在那个 channel 的 sendq 或 recvq 里挂一个 sudog,里面记着："我是哪个 goroutine""我在等哪个 channel""数据要放/取到哪个地址"。

一个 goroutine 可以同时在多个 channel 的等待队列里各挂一个 sudog——这就是 select 能"同时等多个 channel"的实现基础。


## 总结
关键在于，channel 的设计核心不是高性能队列，而是与 goroutine 调度深度集成：当 buffer 满或空时，不是返回错误，而是让 G 进入等待队列并 park，等对方到来再由 runtime 唤醒。这使得 channel 天然支持阻塞语义和公平调度。

```go
package minichan

import "sync"

type MiniChan struct {
	buf      []interface{} // 缓冲区
	capacity int           // 容量
	sendIdx  int           // 下一个写入位置
	recvIdx  int           // 下一个读取位置
	count    int           // 当前元素个数
	mu       sync.Mutex
	cond     *sync.Cond // 用于等待send/recv
	closed   bool
}

func NewMiniChan(capacity int) *MiniChan {
	ch := &MiniChan{
		buf:      make([]interface{}, capacity),
		capacity: capacity,
	}
	ch.cond = sync.NewCond(&ch.mu)
	return ch
}

func (ch *MiniChan) SendBuffered(v interface{}) error {
	ch.mu.Lock()
	defer ch.mu.Unlock()

	if ch.closed {
		panic("send on closed channel")
	}

	for ch.count == ch.capacity {
		ch.cond.Wait()
		if ch.closed {
			panic("send on closed channel")
		}
	}

	ch.buf[ch.sendIdx] = v
	ch.sendIdx = (ch.sendIdx + 1) % ch.capacity
	ch.count++

	ch.cond.Broadcast()
	return nil
}

func (ch *MiniChan) RecvBuffered() (interface{}, bool) {
	ch.mu.Lock()
	defer ch.mu.Unlock()

	for ch.count == 0 && !ch.closed {
		ch.cond.Wait()
	}

	if ch.count == 0 && ch.closed {
		var zero interface{}
		return zero, false
	}

	v := ch.buf[ch.recvIdx]
	ch.buf[ch.recvIdx] = nil // 避免内存泄漏
	ch.recvIdx = (ch.recvIdx + 1) % ch.capacity
	ch.count--

	ch.cond.Broadcast()
	return v, true
}

func (ch *MiniChan) Close() {
	ch.mu.Lock()
	defer ch.mu.Unlock()

	ch.closed = true
	ch.cond.Broadcast()
}

```