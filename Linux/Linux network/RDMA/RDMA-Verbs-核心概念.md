## 调用链分层
应用代码调用的是 libibverbs 用户态库。库里的操作分两类。

1. 资源类操作（创建销毁 PD、MR、CQ、QP）通过 `/dev/uverbs` 字符设备进入内核，由内核 verbs 层分配资源、交驱动落地，属于重量级操作。
2. 数据面操作（`ibv_post_send`、`ibv_post_recv`、`ibv_poll_cq`）在用户态完成：队列页和控制页被 mmap 进进程地址空间，应用直接往 SQ 写条目、直接从 CQ 读完成，全程不发系统调用。
数据面的这个性质叫内核旁路（kernel bypass），RDMA 低延迟的主要来源之一就是它。

rxe（Soft-RoCE 的内核模块）在这个架构里占驱动位。它是一个软件实现的 IB 驱动：复用普通以太网卡收发包，在内核里实现完整的 IB 传输语义。同一份 verbs 代码跑在 rxe 和真实 RoCE 网卡上行为一致，性能不同。

## 资源对象总览

| 对象 | 创建函数 | 返回物 | 所属 |
|---|---|---|---|
| context | `ibv_open_device` | `struct ibv_context *` | 一台 RDMA 设备 |
| PD | `ibv_alloc_pd` | `struct ibv_pd *` | context |
| MR | `ibv_reg_mr` | `struct ibv_mr` | PD |
| CQ | `ibv_create_cq` | `struct ibv_cq *` | context |
| QP | `ibv_create_qp` | `struct ibv_qp *` | PD，绑定一个 CQ |

![[Attachments/Pasted image 20261117203128.png]]

引用关系是一棵树：QP 创建时要指定所属 PD 和绑定哪个 CQ，所以两者必须先存在；然后MR 挂在 PD 下。

创建顺序为 PD → MR → CQ → QP，销毁严格倒序，因为销毁函数会检查引用。

## PD：Protection Domain，保护域

PD 是一圈权限边界，把 MR 和 QP 圈在一起，明确了能访问的内存的范围。

QP 执行操作时，硬件校验 WR 引用的 MR 与该 QP 是否同域；不同域的访问直接失败。
这个校验发生在硬件层，进程内多个 PD 之间互相隔离，一个 PD 里的 QP 碰不到另一个 PD 里的内存。

对端拿 rkey 访问时，硬件校验 rkey 与目标 MR 是否同域，跨域访问产生的 WC status 为 `IBV_WC_REM_PROT_ERR`（远程保护错误）。

销毁用 `ibv_dealloc_pd`，它是四个资源对象里第一个创建、最后一个销毁的。


## MR：Memory Region，注册内存
MR 注册的是一块内存区域。发送端要发数据，数据放在某块本地内存里，这块内存必须注册成 MR，硬件才能对它做 DMA 读。

`ibv_reg_mr(pd, addr, length, access_flags)` 把一段进程内存交给驱动处理
执行两件事情：
1. pin
网卡做 DMA 时按物理地址搬数据，但进程的虚拟地址对应的物理页可能被内核换出到 swap，也可能被搬到别的物理页。注册时驱动会把这些页钉住，之后就不会再被换出或迁移。
2. 建立映射表
设备侧会记下这块内存的虚拟地址到物理地址的映射。硬件执行 RDMA 操作时直接查这张表做地址翻译，CPU 不参与。

注册要进内核，锁页和建表都有开销，而且内存越大越明显。所以临时注册一大块内存是慢操作；高频传输通常的做法是启动时一次注册好，后面一直复用
回的 `struct ibv_mr` 里带两个 key：lkey 给本端 WR 的 sge 用，rkey 交给对端；对端用 remote_addr 加 rkey 就能定位到这块内存。


`access_flags` 决定能力：

| 标志 | 含义 |
|---|---|
| `IBV_ACCESS_LOCAL_WRITE` | 本端可写 |
| `IBV_ACCESS_REMOTE_WRITE` | 对端可写，单边 Write 的接收端必开 |
| `IBV_ACCESS_REMOTE_READ` | 对端可读 |
| `IBV_ACCESS_REMOTE_ATOMIC` | 对端可做原子操作 |

规范要求 `REMOTE_WRITE` 与 `LOCAL_WRITE` 同时开启，只开前者注册会失败。
对端写入的地址范围 `[remote_addr, remote_addr + len)` 超出注册范围时，发起端收到 `IBV_WC_REM_PROT_ERR`。

## QP：Queue Pair，队列对
`ibv_create_qp(pd, attr)` 返回 `struct ibv_qp` 和一个 24 位编号 QPN，对端寻址靠 QPN。
QP 内部是两条队列：Send Queue（SQ）和 Receive Queue（RQ）。应用提交的 WR 落进队列后叫 WQE（Work Queue Element，工作队列元素），容量 `max_send_wr`、`max_recv_wr` 在创建时指定，硬件按序取走执行。

传输类型常用 RC（Reliable Connection）：硬件对每个包做端到端确认，丢失的包自动重传，接收端按 PSN（Packet Sequence Number，24 位包序号）排序去重。

一条消息超过 MTU 时硬件按 MTU 分片发送，`path_mtu` 取 512、1024、2048、4096 字节之一。

一个 QP 刚创建出来不能直接用，要按顺序把它“推”到能正常收发数据的状态
QP 创建后处于 RESET 状态，按 RESET → INIT → RTR → RTS 手动迁移
每推一步都要通过 ibv_modify_qp 告诉它一些必要信息

| 状态 | 迁移后允许的动作 | 必填属性 |
|---|---|---|
| RESET | 初始状态，不收不发 | 无 |
| INIT | 只能提交 RQ（收） | `qp_access_flags`、`pkey_index`、`port_num` |
| RTR | SQ 也可用（发收都行） | `dest_qp_num`、`rq_psn`、`path_mtu`、`max_dest_rd_atomic`、`ah_attr`、`min_rnr_timer` |
| RTS | 正常收发 | `sq_psn`、`timeout`、`retry_cnt`、`rnr_retry`、`max_rd_atomic` |

各字段含义：

| 字段 | 填在哪段 | 含义 |
|---|---|---|
| `qp_access_flags` | INIT | 声明本端允许对端做 Write/Read/Atomic 中的哪些 |
| `pkey_index` | INIT | 分区键索引，单机环境用默认值 0 |
| `port_num` | INIT | 用设备的哪个物理端口 |
| `dest_qp_num` | RTR | 对端 QPN |
| `rq_psn` | RTR | 对端的起始 PSN，与对端的 `sq_psn` 一致 |
| `path_mtu` | RTR | 路径 MTU，两端取小 |
| `max_dest_rd_atomic` | RTR | 对端能同时响应的本端发起的 Read/Atomic 数 |
| `ah_attr` | RTR | 对端地址信息：GID、GID index 等 |
| `min_rnr_timer` | RTR | 对端 RQ 为空时回 RNR NAK 的定时器 |
| `sq_psn` | RTS | 本端的起始 PSN，与对端的 `rq_psn` 一致 |
| `timeout` | RTS | 等对端 ACK 的超时，单位 4.096 微秒 × 2^timeout |
| `retry_cnt` | RTS | 对端未回 ACK 时本端的重传次数 |
| `rnr_retry` | RTS | 对端 RQ 为空（RNR NAK）时的重试次数，7 表示无限 |
| `max_rd_atomic` | RTS | 本端能同时挂起的 Read/Atomic 操作数 |

rkey 与 remote_addr 填在 WR 里，随每次操作下发，属于 MR 层的参数，不出现在任何一段 QP 属性里。

出错的 QP 进入 ERR 状态，ERR 状态下不再执行任何 WR，只能查询属性或销毁。

## CQ、CQE 与 WC
#### CQ（Completion Queue，完成队列）
CQ完成队列，一个用来收“操作做完了”通知的队列。
CQ中存放的是完成标志与应用元数据
CQ是单向队列，硬件生产应用消费，应用不往CQ里放任何数据
它由 ibv_create_cq 创建，容量在创建那一刻就定死，后面不能扩

一个 QP 的发送队列 SQ 和接收队列 RQ 各自绑一个 CQ，发出去的 WR 完成了往哪个 CQ 报、收进来的 WR 完成了往哪个 CQ 报，由绑定关系决定。


CQ满了就CQ overrun，CQ 进入错误状态，之后不能正常用。所以 cqe_cnt 要给够，至少能容纳在途的 signaled WR 数量。

#### CQE（Completion Queue Entry，完成队列条目）
CQE是CQ中的条目，硬件每完成一个WR，就往对应的CQ中写一条CQE
优化： SQ 侧不是每条 WR 都产生 CQE，只有被标记为 signaled 的 WR 才产生。
批量发很多 WR 时，可以只给关键几条打 signaled，其余不打，避免 CQ 被大量完成条目淹没。

一条 CQE 对应一个 WR 的完成，里面记着这个 WR 完成得怎么样、是哪个 WR。

#### WC（Work Completion，工作完成）
 是应用侧的结构体 struct ibv_wc。应用调 ibv_poll_cq 从 CQ 里取 CQE 时，CQE 的内容会被拷进应用传进来的 ibv_wc 里。所以 CQE 是队列里的条目，WC 是取出来之后应用看到的那份数据，两者字段基本对应。


## WR：Work Request，工作请求

WR 是提交给硬件的任务描述。`ibv_post_send` 把 WR 提交进 SQ，`ibv_post_recv` 提交进 RQ。一条 WR 的核心字段：

| 字段 | 含义 |
|---|---|
| `wr_id` | 应用自填的 64 位标识，完成时在 WC 里原样带回 |
| `opcode` | 操作类型 |
| `sg_list` | sge 列表，每个 sge 填本地地址 addr、长度 length、lkey，数量上限 `max_send_sge` |
| `send_flags` | `IBV_SEND_SIGNALED`（要完成事件）、`IBV_SEND_INLINE`（数据直接内联进 WQE，省一次 DMA，长度不超过创建 QP 时的 `max_inline_data`） |
| `remote_addr` / `rkey` | 单边操作的目标内存，填对端 MR 的地址与凭证 |

常用 opcode 三种。
- SEND 是双边操作：对端 RQ 里必须有配对的 recv WR 消费它，消息长于对端 recv buffer 时对端收到 `IBV_WC_LOC_LEN_ERR`。SEND 语义下数据以消息为单位，对端拿到长度信息。
- RDMA WRITE 是单边写：发起端填好对端的 remote_addr 和 rkey，直接写对端内存，对端 CPU 不参与、不产生 CQE、不消费 RQ。接收端感知数据到达有两条路：一是用 `IBV_WR_RDMA_WRITE_WITH_IMM`，写在最后一个包附带一个 32 位立即数，对端 RQ 上会弹出一条 CQE，前提是接收端先 post 了 recv WR；二是应用层自己校验，比如收完约定长度的数据后比对 checksum。
- RDMA READ 是单边读：本端发起，把对端内存拉进本端 buffer，完成事件在本端 CQ 上。并发挂起的 READ 数受 `max_rd_atomic` 限制，超限时硬件排队等待。

#### 接收端取数据方式
事先 ibv_post_recv 挂一个 WR 到 RQ，WR 的 sge 里给出一个缓冲区地址和长度。意思是“我准备好往这儿收了”。

对端数据过来，硬件把数据 DMA 进这个缓冲区，然后往 CQ 写一条 CQE。

应用 ibv_poll_cq 拿到 CQE，看 wc.status 是否 IBV_WC_SUCCESS、wc.byte_len 是多少、wc.wr_id 对应哪个 WR，然后回到当初 post_recv 时给的那个缓冲区地址去读数据。

## 关键标识符一览

| 标识符 | 生成方 | 填在哪里 | 怎么到对端 |
|---|---|---|---|
| lkey | `ibv_reg_mr` | 本端 WR 的 sge | 不出本机 |
| rkey | `ibv_reg_mr` | 对端 WR 的 `remote_addr` 配套 | 带外（TCP / gRPC / 同进程变量） |
| QPN | `ibv_create_qp` | 对端 QP 迁 RTR 时的 `dest_qp_num` | 带外 |
| GID | `ibv_query_gid` 查询 | 对端 QP 的 `ah_attr` | 带外 |
| PSN | 应用自定 | 本端 `sq_psn`、对端 `rq_psn` | 带外 |

GID（Global Identifier）是 RDMA 层的 128 位网络地址，RoCE v2 下由 IP 地址映射生成，建链时两端互换。PSN 用随机起点，避免新连接复用旧连接的包序号导致残留包被误收。

## 整体流程：以一次 RDMA WRITE / SEND 为例
####  阶段0: 双方各自建立资源
1. ibv_open_device 拿到设备上下文
2. ibv_alloc_pd 建 PD
3. 准备数据缓冲区，ibv_reg_mr 注册成 MR。这一步进内核，锁页、建 VA→PA 表，这一步返回的结果是lkey（本端用）和 rkey（给对端用）
4. ibv_create_cq 建 CQ，CQE 环形缓冲被 mmap 进用户态
5. ibv_create_qp 建 QP，指定 send_cq 和 recv_cq 绑定哪个 CQ，SQ/RQ 环形缓冲被 mmap 进用户态。此时 QP 处于 RESET

#### 阶段1:  带外交换信息，推动QP状态机
两端通过 TCP 之类带外通道交换：QPN、GID、初始 PSN、B 的 rkey、B 的 remote_addr等字段
并且各自手动调 ibv_modify_qp 三次，将QP状态机推动到RTS状态

#### 阶段2(1): RDMA WRITE执行
阶段1中A已经拿到了B 的 rkey 和 remote_addr，可以直接写B的内存，B的CPU全程不参与

1. A将新数据写入A的MR缓冲区
2. A 应用 ibv_post_send，挂一个 RDMA_WRITE WR。sge 指向 A 的 MR（本地源），wr.rdma.remote_addr 填 B 的地址，wr.rdma.rkey 填 B 的 rkey，标记 signaled、
3. A 网卡 DMA 读 A 的 MR 里的 payload，封装成 WRITE 请求，带上 remote_addr 和 rkey放入SQ中
4. 包到 B 网卡。B 网卡校验 rkey 有效（对端没 dereg 过），按 remote_addr 查 B 的 MR 翻译表，DMA 写 payload 直接进 B 的 MR 缓冲区。B 的 CPU 和 B 的应用完全不知道这件事发生
5. B 网卡回 ACK
6. A 网卡收到 ACK，往 A 的 send CQ 写一条CQE
7. A 应用 poll send CQ，看到成功

B端没有任何CQE产生，B端根本没有CQ与WC，第 4 步之后，B 网卡把数据 DMA 写进 B 的 MR 就结束了

#### 阶段2(2): SEND执行
B侧前置： B 应用 ibv_post_recv，往 RQ 挂一个 WR(往mmap的RQ内存中写入条目)
这个WR的指明了：数据往哪放、最多放多少、完成后用什么 id 标记
- sge里面存储了地址与长度
- wr_id是标识，硬件完成时把它抄进 CQE，B 应用 poll 出 WC 后根据wr_id知道是哪次 post_recv 完成了
- opcode和标志位：表明这是个 recv WR，以及是否 signaled

1. A将新数据写入A的MR缓冲区
2. A 应用 ibv_post_send，挂一个 RDMA_WRITE WR。sge 指向 A 的 MR（本地源），wr.rdma.remote_addr 填 B 的地址，wr.rdma.rkey 填 B 的 rkey，标记 signaled、
3. A 网卡 DMA 读 A 的 MR 里的 payload，封装成 WRITE 请求，带上 remote_addr 和 rkey放入SQ中
4. 包到达B网卡，B网卡读取B的RQ里面的WR，按 sge 地址查 B 的 MR 翻译表，DMA 写 payload 进 B 的 MR 缓冲区。B 的 CPU 不参与
5. B 网卡往 B 的 recv CQ 写一条 CQE，回复A端ACK
6. A 网卡收到 ACK，往 A 的 send CQ 写一条 CQE

AB端收尾：
B端应用ibv_poll_cq 轮询 recv CQ，拿到 CQE 拷进 WC，看 wr_id 知道是哪个 recv WR 完成了，看 byte_len 知道收了多少，然后回到当初 post_recv 时 sge 指的那块 MR 缓冲区去读数据

A 应用 ibv_poll_cq 轮询 send CQ，拿到 CQE，status == IBV_WC_SUCCESS 就确认发送完成，可以复用或释放那块发送缓冲区

#### 阶段3: 销毁
顺序大致是：确认没有在途 WR → ibv_destroy_qp → ibv_destroy_cq → ibv_dereg_mr（撤翻译表、解钉页、rkey 作废）→ ibv_dealloc_pd → 关设备。

ibv_dereg_mr 之后，如果对端还拿旧 rkey 发 WRITE/READ，B 网卡查表失败，回保护错误，A 侧会收到对应的 WC 错误码