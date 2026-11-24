## SSE长连接
SSE（Server-Sent Events）
它是一种基于 HTTP 的服务端向客户端持续推送数据的技术
客户端请求一次，服务端可以在同一条 HTTP 响应中，持续发送多条事件，直到连接关闭。

SSE 建立在http之上，底层通常使用 TCP。
#### 工作流程
1. 客户端发送 HTTP 请求。
2. 服务端返回 HTTP 200。
3. 响应头声明 Content-Type: text/event-stream。
4. 服务端持续向响应体写入事件数据。
5. 客户端持续读取和解析这些数据。

#### 数据格式
SSE 有自己的事件格式，但本质上是文本协议。
每个事件最后的空行表示一个事件结束
`event`事件类型
`data`事件携带的数据
`id`事件 ID，可用于断线重连
`retry`建议客户端重连等待时间

#### SSE 和 WebSocket 
| 对比   | SSE                | WebSocket    |
| ---- | ------------------ | ------------ |
| 通信方向 | 服务端向客户端推送          | 双向通信         |
| 底层   | HTTP 响应流           | WebSocket 协议 |
| 数据格式 | UTF-8 文本           | 文本和二进制       |
| 自动重连 | 浏览器 EventSource 支持 | 通常需要自己实现     |
| 典型场景 | AI 流式输出、实时通知       | 聊天、实时协作、双向交互 |
ChatGPT 输出回答：适合 SSE。
在线多人游戏：通常更适合 WebSocket。

## Go 服务端实现 SSE
普通 Go HTTP Handler 写响应,hander返回之后，相应结束
```go
func handler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello"))
}
```

SSE需要Handler一直运行，持续向 ResponseWriter 写数据，直到连接断开或者业务结束。
```go
func handler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")

    for i := 0; i < 3; i++ {
        fmt.Fprintf(w, "data: %d\n\n", i)
        w.(http.Flusher).Flush()

        time.Sleep(time.Second)
    }
}
```
Flush() 的作用是：把当前缓冲的数据尽可能向下游发送，让客户端不必等后续数据。

## 客户端接收SSE
浏览器原生提供了 EventSource API，用来接收 SSE
```js
const es = new EventSource("/stream");

es.onmessage = (event) => {
    console.log("收到数据：", event.data);
};

es.onerror = (event) => {
    console.log("连接异常");
};
```
浏览器的 EventSource 默认支持自动重连。假如网络断开，浏览器会等待一段时间，再重新发送请求，服务端重新建立响应流。

服务端可以通过 SSE 的 retry 字段建议重连等待时间，retry: 3000表示客户端建议等待 3000 毫秒后重连

#### 断线之后避免数据丢失
服务端可以给每条事件分配 ID假设客户端收到了 101、102，但 103 还没收到就断线。
浏览器重新连接时，可以携带：
```http
Last-Event-ID: 102
```

SSE 协议本身只提供事件 ID 和重连机制，并不负责保存历史事件。

是否能够补发，取决于服务端有没有持久化或者缓存事件，以及有没有实现基于服务端ID 的恢复逻辑。

## SSE连接的资源占用
Go 的 HTTP Server 通常会为每个连接安排 goroutine 处理请求。对于 SSE，这个 goroutine 会在整个流期间持续存在
但是goroutine在长时间内都是阻塞，它不会占用一个 CPU 核心一直运行

SSE会产生以下资源占用：
goroutine：Handler 长期存活，等待数据
TCP 连接：内核维护连接状态、缓冲区等
内存：goroutine 栈、业务状态、缓冲数据
文件描述符：每个 Socket 都需要 FD
消息队列：慢客户端可能积压尚未发送的数据

## SSE下Goroutine泄露场景
- 用了 context.Background() 而不是 request context
所有子 goroutine 的 context 必须从 stream 的 context 派生，用context.WithCancel(streamCtx)，stream context 一 cancel，全部级联退出

- channel没有关闭或者没人收取
如果客户端断开后，上游还在往一个无缓冲或者已满的 channel 里写，那个写 goroutine 就永远阻塞在那儿了

- 心跳定时器没停
SSE 要定期发心跳保活，通常会起一个 ticker goroutine。如果这个 goroutine 的 select 只监听 ticker 和 event channel，没监听 ctx.Done()，客户端断了它还在那儿 tick。