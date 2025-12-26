三天前第一次打开 Wireshark，界面一滚就是几十个数据包，协议名、端口号、十六进制字段混在一起，说实话完全不知道从哪下手。
三天后，再次抓包，我已经能快速过滤掉无关流量，定位一次完整的请求，判断它在网络栈中的位置。

这篇文章不是 Wireshark 功能介绍，而是一次**从“看不懂”到“能分析”的实战记录**。
## 一、先解决最烦人的问题：为什么一抓包就是一堆 MDNS？

### 1.1 满屏 MDNS，其实不是你的问题

第一次抓包，最直观的感受是：
**为什么什么都没干，数据包却一直在刷？**

其中占比最大的一类，通常是 **MDNS（Multicast DNS）**，它有几个非常明显的特征：
![[Attachments/wireshark/1.png]]
* 目的地址几乎总是 `224.0.0.251:5353`
* 协议内容里大量出现 `.local`
* 不需要你进行任何操作，它也会周期性出现

这并不是异常流量，而是局域网设备在做**自动发现**。
电脑、手机、打印机、音箱都在用 MDNS：

* 告诉别人“我是谁”
* 询问“网络里有没有某类服务”
* 维持局域网状态同步

理解这一点很重要，因为这意味着：
**Wireshark 本身没有问题，网络也没有问题，只是你现在抓到的是“背景噪音”。**

### 1.2 过滤器：抓包的分水岭技能

Wireshark 的使用体验，很大程度上取决于你会不会写过滤器。
如图：
![[Attachments/wireshark/2.png]]

一些最常用、也是最实用的例子：
```bash
# 按协议
http
dns
tcp

# 按 IP
ip.addr == 192.168.1.1
ip.src == 192.168.1.100
ip.dst != 8.8.8.8

# 按端口
tcp.port == 80
udp.port == 53

# 组合条件
http and ip.addr == 192.168.1.1
tcp.port == 443 or tcp.port == 80
not arp
```

几个我实际用得最多的小技巧：
* <span style="color:rgb(221, 85, 85)">直接右键字段 → 作为过滤器应用，不用硬记语法</span>
* `tcp.stream eq N` 用来追一整条 TCP 会话，非常好用
* `tcp.analysis.flags` 可以快速看到重传、乱序等问题

一旦开始用过滤器，Wireshark 才真正“安静下来”。


## 二、访问一个网站，网络里到底发生了什么？
### 2.1 抓一次最简单的 HTTP 请求
为了避免一上来就被 HTTPS 干扰，我刻意选了一个 **明文 HTTP** 网站。
**操作过程很简单：**

1. 打开 Wireshark，选择正在使用的网卡
2. 浏览器访问：`http://httpbin.org/html`
3. 停止抓包，过滤：
   ```
   http and ip.addr == httpbin.org 的ip地址 and ip.addr == 你主机ip地址
   ```
![[Attachments/wireshark/3.png]]

4. 右键任意一个数据包,点击追踪流,追踪TCP或者HTTP流
Wireshark会：
- 自动把四元组 (src_ip,src_port,dst_ip,dst_port) 相同的所有包按 seq 排序；
- 弹出一个新窗口，以“文本”形式把客户端→服务器、服务器→客户端 的数据并排显示；
追踪TCP流：
```
GET /html HTTP/1.1
Host: httpbin.org
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8

HTTP/1.1 200 OK
Date: Wed, 17 Dec 2025 14:11:55 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 3741
Connection: keep-alive
Server: gunicorn/19.9.0
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

<!DOCTYPE html>
<html>
  <head>
  </head>
  <body>
      <h1>Herman Melville - Moby-Dick</h1>

      <div>
        <p>
          Availing himself of the mild, summer-cool weather that now reigned in these latitudes, and in preparation for the peculiarly active pursuits shortly to be anticipated, Perth, the begrimed, blistered old blacksmith, had not removed his portable forge to the hold again
...... // 篇幅限制，中间内容我删去一部分
young widow had a delicious grief, and her orphans a truly venerable, legendary sire to dream of in their after years; and all of them a care-killing competency.
        </p>
      </div>
  </body>
</html>
```
接下来你看到的，其实就是教科书里的流程，只不过这次是**真实发生的**。


### 2.2 关键数据包拆解
此时我们的面板变成了这样，自动过滤：`tcp.stream eq 1`
![[Attachments/wireshark/4.png]]

点进去一个仔细看看：
![[Attachments/wireshark/10.png]]
整体来说分为四部分：
- 总体信息
- 帧格式（OSI第二层）
- IP首部（OSI第三层
- TCP首部（OSI第四层）

总体上面看：
#### 1）DNS 查询（如果本地没有缓存）
```
你的 IP → DNS 服务器
查询：httpbin.org
```
我这里因为刚才访问过该网站，所以没有DNS查询步骤

#### 2）TCP 三次握手
![[Attachments/wireshark/9.png]]
以下是这张图的表格，怕你们看不清：

| 帧号 | 时间戳 (s)     | 源 IP          | 目的 IP         | 协议  | 长度 (B) | 报文摘要                                                                                                            |
| -- | ----------- | ------------- | ------------- | --- | ------ | --------------------------------------------------------------------------------------------------------------- |
| 14 | 3.126387494 | 100.87.84.203 | 34.193.26.89  | TCP | 74     | 39720 → 80 \[SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK\_PERM TSval=1772232773 TSecr=0 WS=128                     |
| 19 | 3.392793852 | 34.193.26.89  | 100.87.84.203 | TCP | 74     | 80 → 39720 \[SYN, ACK] Seq=0 Ack=1 Win=26847 Len=0 MSS=1460 SACK\_PERM TSval=1050374714 TSecr=1772232773 WS=256 |
| 20 | 3.392839560 | 100.87.84.203 | 34.193.26.89  | TCP | 66     | 39720 → 80 \[ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=1772233040 TSecr=1050374714                                 |

这一步非常值得反复看几次，因为后面所有 HTTP 都建立在它之上。
如果你是新手，对于下面的报文看不懂，你可以去补充一些TCP的基础知识
Seq: 我要发的包序号
Ack: 希望对方发的包序号
Win: 缓冲区大小
Len: 包的总大小
###### 第 14 帧：SYN —— “喂，能听到吗？”
- 谁→谁：我（100.87.84.203:39720）→ 服务器（34.193.26.89:80）
- Seq=0：这是“起始序号”，Wireshark 帮你折成 0，真实绝对值是一个 32 位随机数。
- Win=64240：告诉对方“我接收窗口现在就 62 540 字节，别发超”。
- MSS=1460：单段最大数据 1460 B，防止 IP 分片。
- 长度 74 B = 14 (Eth) + 20 (IP) + 40 (TCP 含 20 B 选项)。
###### 第 19 帧：SYN-ACK —— “收到，可以聊！”
- 往返时间：3.392 793 – 3.126 387 ≈ 266 ms（物理距离 + 网络排队）。
- Seq=0、Ack=1：服务器也选了自己的初始序号 0，并把 Ack 设为1，表示“我期待你下字节从 1 开始”。
- Win=26847：服务器窗口比客户端小，因为它只是小博客站，内存少。
- WS=256：服务器窗口扩大因子 256，比客户端的 128 大，但基础窗口小，最终吞吐还是看瓶颈段。
- 真实窗口 = Win × 2 ^WS
- TSecr=1772232773：把客户端的 TSval 原样打回，这样客户端就能算出 RTT(跑一次来回所需的时间)。(TSval 是“我这边现在几点”，TSecr 是“刚才你那边几点”——来回一减就是 RTT)
- 长度也是 74 B，结构同第一条
###### 第 20 帧：纯 ACK —— “OK，我开始发数据！”
- 耗时：46 μs（本地系统回包，几乎瞬间）。
- Seq=1、Ack=1：客户端确认服务器的 SYN，同时把自己的序号推进 1。
- Len=0：没有数据，只是“握手收尾”报文。
- Win=64256：窗口比第一次多了 16 字节，其实是 TCP 协议栈把“SYN 占 1 字节”解掉后重新算了一次，无实质含义。
- 长度 66 B = 14 + 20 + 32（TCP 头无选项，所以 20 基本 + 12 字节时间戳选项）。
#### 3）HTTP 请求
![[Attachments/wireshark/11.png]]

#### 4）HTTP 响应
![[Attachments/wireshark/12.png]]

看到这里其实会有一种“恍然大悟”的感觉：
**原来浏览器访问网页，本质就是这么一来一回。**

#### 5）四次挥手
![[Attachments/wireshark/13.png]]
发现这并不是我们学过的完整的四次挥手，这是为什么呢?
原因就是：
服务器 65 秒先发出 FIN，我只回了 ACK，进入 TIME_WAIT。  
后面全是 Keep-Alive：我每 45 秒探一次活，对方也正常回 ACK。  
158 秒服务器直接发 RST，连接被内核立即回收，**③④两步 FIN 根本没出现**，所以看不到完整四次挥手。  
原因很简单：空闲超时，远端不想继续维护这条 TCP。

## 三、写自己的程序，再用 Wireshark 看它“说话”

如果只抓浏览器流量，很容易停留在“观察者”阶段。
真正加深理解的，是**自己写程序 → 再抓自己的包**。

### 3.1 一个最小 TCP 服务器

```python
# server.py
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('127.0.0.1', 9999))
server.listen()
print("TCP服务器监听 127.0.0.1:9999")

while True:
    client, addr = server.accept()
    data = client.recv(1024)
    client.send(b"Echo: " + data)
    client.close()
```

客户端直接用：

```bash
nc 127.0.0.1 9999
```

---

### 3.2 抓自己写的 TCP 程序

Wireshark 设置很简单：
* 接口选 `lo`（环回）
* 过滤器：
  ```
  tcp.port == 9999
  ```
- 在终端输入：
![[Attachments/wireshark/14.png]]
这时你可以非常清楚地看到：
![[Attachments/wireshark/15.png]]
* 三次握手是怎么发生的
* 数据对应哪些序列号
* 连接是如何被正常关闭的（FIN / ACK）

当你意识到：
**这些包正是你几行代码“发出去的结果”**
抽象的 TCP 概念一下子就具体了。

### 3.3 UDP 的对比更直观
```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('127.0.0.1', 9998))

while True:
    data, addr = sock.recvfrom(1024)
    sock.sendto(b"UDP Echo: " + data, addr)
```
client: `echo "hello,world" | nc -u 127.0.0.1 9998`
![[Attachments/wireshark/16.png]]
抓包后很明显：
* 没有握手
* 每个包都是独立的
* 没有重传、没有连接状态
UDP 和 TCP 的差异，在这里几乎是“肉眼可见”。

## 四、结语

Wireshark 本身并不难，难的是**你不知道该看什么**。
一旦学会过滤背景流量，再从应用层往下看，很多网络概念都会自然对上。

真正理解网络的瞬间，不是在书上，而是在 Wireshark 里看到：
**“这就是我刚才那一行代码发出去的包。”**