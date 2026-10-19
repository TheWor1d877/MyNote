#面试-Go语言-复习  在 SSE 长连接推理网关中，goroutine 泄漏的典型场景有哪些？
## SSE长连接推理网关
SSE（Server-Sent Events）是一种基于 HTTP 的服务器单向推送协议
相比于需要双向通信的 WebSocket，SSE 更轻量，天然适配“用户发一个提问，模型逐字吐出回答”这种单向、流式的场景
推理阶段显著特点：相应延迟高，输出内容长

SSE允许模型边推理边输出，每生成一个 token 就立刻推送给前端

因为一个请求可能要持续几十秒甚至几分钟，网关与模型服务器、网关与客户端之间必须维持 HTTP 长连接。

“推理网关”是近年来随着 LLM 普及而衍生出的概念，它建立在传统 Ingress/Gateway 之上，但增加了模型感知的能力

## Goroutine泄露
- 用了 context.Background() 而不是 request context
所有子 goroutine 的 context 必须从 stream 的 context 派生，用context.WithCancel(streamCtx)，stream context 一 cancel，全部级联退出

- channel没有关闭或者没人收取
如果客户端断开后，上游还在往一个无缓冲或者已满的 channel 里写，那个写 goroutine 就永远阻塞在那儿了

- 心跳定时器没停
SSE 要定期发心跳保活，通常会起一个 ticker goroutine。如果这个 goroutine 的 select 只监听 ticker 和 event channel，没监听 ctx.Done()，客户端断了它还在那儿 tick。