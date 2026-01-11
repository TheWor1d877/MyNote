带有循环索引的固定大小数组
配合两个等待队列(sendq,recvq)与一把mutex

有缓冲的channel：是使用buf数组存储数据，使用sendx与recvx两个索引模容量实现FIFO循环读写
无缓冲的channel：没有buffer，发送方与接收方直接handoff，一个goroutine唤醒另一个

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