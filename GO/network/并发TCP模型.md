## goroutine-per-connection
最常用的方式
```go
listener, _ := net.Listen("tcp","8080")
for{
	conn, _ := listener.Accept()
	go handleConn(conn)
}
```
简单直接：无需设计复杂状态机或任务队列
天然支持全双工：读写可分别在两个 goroutine 中进行
4
#### 设置最大连接数
无限制 accept 会导致内存耗尽。
