## Https的好处
- 安全性
https下层加了SSL/TLS加密
加密防偷看，校验放篡改，证书防冒充

- 性能
慢一点，但是在TLS1.3,Http2，QUIC加持下可以忽略不记

- 端口
80 vs 443

- 搜索引擎
http不安全，浏览器标红

## Https的流程
1. client hello + 随机数
2. server hello + 随机数 + 证书
3. client key exchange 
4. server key exchange
以后走加密通道
在LTS1.3中握手压缩到1个RTT
