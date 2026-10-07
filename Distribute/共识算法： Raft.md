## Raft中的角色
在任何时刻，集群中的每个节点（Node）只能处于以下三种状态之一：
- Leader（领导者）： 只有 Leader 能接收写入请求。其他节点收到写入请求会转发给 Leader。
- Follower（跟随者）： 沉默的备份者。所有读写请求要么转发，要么回复“我不知道，去找 Leader”。
- Candidate（候选者）： 临时工。只在选举期间存在，选完要么变 Leader，要么变 Follower。

## Leader选主
集群刚开始时，所有节点都是 Follower。它们会互相发送心跳（Heartbeat）来确认对方活着。

物理过程（以 3 个节点为例）：

初始状态：Node1、Node2、Node3 都是 Follower。它们各自有一个 选举计时器（Election Timer），随机在 150ms~300ms 之间倒计时。

超时触发：Node2 的计时器先归零（比如 200ms）。它等不到 Leader 的心跳，于是切换成 Candidate，给自己投一票，然后向 Node1 和 Node3 发送 “投票请求（RequestVote RPC）”。

投票与统计：
Node1 收到请求，检查自己的日志和任期号，发现 Node2 的日志不比自己旧，于是投赞成票。
Node3 同样投赞成票。
Node2 收到 3 票（自己+Node1+Node3），超过半数（3/2+1=2），获胜成为新 Leader。

维持统治：Node2 成为 Leader 后，立即向所有 Follower 发送心跳（AppendEntries RPC，即使没有新数据），告诉它们“我还活着”。其他节点的选举计时器收到心跳后重置，不再发起选举。

任期号（Term）：每次选举，任期号 +1。如果某个 Candidate 的任期号小于当前任期，其他节点会拒绝投票。

随机超时：选举计时器是随机的，大大减少了多个节点同时发起选举导致选票分裂的概率。

## 日志复制
客户端发起请求：客户端把写请求 {"op": "SET", "key": "x", "value": "1"} 发送给 Leader（Node2）。

Leader 写入日志（Append）：
Node2 收到请求，将其包装成一条日志条目（Log Entry），格式为 [索引=1, 任期=1, 命令=SET x=1]。
将这条日志写入自己的磁盘（WAL 文件）。

并行复制：Node2 在下一次心跳或立即，将这条日志条目并行发送给 Node1 和 Node3（AppendEntries RPC）。

Follower 确认（Ack）：
Node1 收到后，将日志写入自己的磁盘，返回 成功。
Node3 同样操作，返回 成功。

Leader 提交（Commit）：
Node2 收到了 超过半数（2 个） 的确认回复（自己+Node1 或 Node3）。
Node2 将这条日志标记为 “已提交（Committed）”，并在内存中真正执行这条命令（比如把内存中的 Key-Value 从 0 改成 1）。
然后 Node2 回复客户端：“写入成功”。

Follower 应用（Apply）：
Node2 在后续的心跳消息中，会带上“我已经提交到索引 1”。
Node1 和 Node3 收到后，发现自己的本地日志索引 1 还没执行，于是执行这条命令，把 Key-Value 改成 1。


key：
半数确认：只要多数派节点写入了日志，数据就算持久化了，不要求所有节点都成功。
顺序一致：所有节点上的日志条目顺序都是按照 Leader 分配的索引来的，保证了全局顺序。

## Raft应对故障
#### Leader宕机
心跳超时：Node1、Node2、Node4、Node5 在 150ms~300ms 内没收到 Node3 的心跳。

发起选举：Node1（计时器先归零）变成 Candidate，任期 +1（变为 4），请求其他节点投票。

获得多数票：Node1 收到 Node2、Node4 的投票（共 3 票），成为新 Leader（任期 4）。

继续服务：Node1 作为新 Leader 接受客户端写入。此时 Node3 如果恢复，发现自己的任期（3）小于当前任期（4），会自动降级为 Follower，并接受新 Leader 的日志覆盖

Term小于currentTerm的node不会被投票

#### Follower宕机
写入时：Leader Node3 向 Node5 发复制请求，但超时。只要剩余节点（Node1、Node2、Node4）达到多数（3），写入仍然成功。

恢复时：Node5 重启后，发现自己的日志落后，会向 Leader 请求缺失的日志条目（从上一个索引开始），Leader 会将积压的日志批量发给它。

#### 网络分区
分区前：Node3 是 Leader。
分区后：Node1、Node2（多数派）与 Node3（少数派）隔离。

Node1、Node2 超时后选举 Node1 为新 Leader（任期 4）。
Node3 虽然还在运行，但收不到多数派的确认，无法提交任何新日志。
网络恢复后：Node3 收到来自 Node1 的更高任期（4）的心跳，自动降级为 Follower，并同步 Node1 的日志。


## Term
Term（任期）是全局唯一的逻辑时钟，不是每台机器单独拥有的。
同一时刻有且只有一个有效的Term编号

Term 统一全局的方式：
1. 选举时：
Candidate 发起投票请求，必须在RPC里带上自己的 currentTerm（比如 Term=5）。

Follower 收到后，会比较：
如果 请求中的 Term > 本地的 currentTerm：说明自己落后了，立即更新本地 Term=5，并给 Candidate 投票。
如果 请求中的 Term < 本地的 currentTerm：说明 Candidate 已经过时了，拒绝投票。

2. 心跳/日志复制时：
Leader 在每次 AppendEntries RPC 中，都会带上自己的 Term（比如 Term=4）。

Follower 收到后，同样比较：
如果 请求中的 Term > 本地 Term：说明自己错过了选举，立即更新本地 Term=4，并降级为 Follower。
如果 请求中的 Term < 本地 Term：说明 Leader 已经过时了，拒绝该RPC，并回复“我的 Term 更大，你应该降级”。