## TCP/IP在Go中的实现基础
Go 使用 net 包封装了对 TCP/IP 协议的操作。
```go
ln, _ := net.Listen("tcp", ":8080")
conn, _ := ln.Accept()
```

Go 的 net.Conn 是对操作系统 socket 的封装，其读写操作最终会调用系统调用（如 read, write, recv, send）。

## Go的网络模型：Netpoller与Goroutine调度
Goroutine：轻量级线程，由 Go 运行时调度
Netpoller（网络轮询器）：基于 epoll/kqueue/IOCP 的事件驱动机制

当一个Goroutine读到conn.Read()的时候，如果数据没有就绪，就会将这个Goroutine挂起，将这个socket注册到epoll
当 socket 可读时，epoll 通知 Go 运行时
Go 运行时唤醒对应的 Goroutine，继续执行

## HTTP2的自动协商协议
客户端（如浏览器）在 TLS ClientHello 中声明支持 h2
服务端若支持，则返回 h2，并切换到 HTTP/2 处理逻辑
```go
	server := &http.Server{
		Addr: ":8080",
		TLSConfig: &tls.Config{
			NextProtos: []string{"h2", "http/1.1"},
		};
	}
```

## 多路复用
HTTP/2 不再把整个请求/响应当作一个整体传输，而是拆成 小数据块（帧，Frame），每个帧都带一个 流 ID（Stream ID）。

单个 TCP 连接上可并行处理多个 流（Stream）
每个流有唯一 ID，帧中包含 Stream ID 字段

每个流的数据被切成多个 DATA 帧
所有帧交错混合在一个 TCP 连接上传输

Go 通过 http2.serverConn 管理所有流：
```go
type serverConn struct {
    streams map[uint32]*stream // 流ID -> stream对象
}
```

