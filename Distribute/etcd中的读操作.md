它默认的线性一致读，在协议层面确实绕不开 Leader，但它把“经过 Leader”这件事优化到了极致，并且额外提供了一条完全不经过 Leader 的“捷径”供业务选择。

## 线性一致读
1. 获取安全读点
请求所在的节点（无论是 Leader 还是 Follower）会向 Leader 请求当前的 commit index
etcd中通过`MsgReadIndex`消息实现

2. Leader身份确认
Leader 收到请求后，会记录下当前的 `commit index` 作为 ReadIndex。为了确保自己还是合法的 Leader（防止网络分区导致旧 Leader 返回脏数据），它需要向集群其他节点发送一轮心跳。
- 在`ReadOnlySafe`等待多数派确认
- 在`ReadOnlyLeaseBased`依赖租约在有效期内直接确认

3.  等待本地应用
一旦 ReadIndex 被确认，发起请求的节点会检查自己的状态机（appliedIndex）是否已经应用到了这个索引位置。如果还没追上，它会等待；直到本地数据至少和 ReadIndex 一样新，才会真正读取并返回给客户端


在这里，Follower可以处理读请求，但是依然需要向Leader申请一个安全读点

## 串行读
如果你确实不想让读请求经过 Leader，etcd 提供了 串行读 选项。

当你将请求的 consistency 模式设置为 serializable 时，节点会直接读取本地状态机，不发送任何 MsgReadIndex 消息，不与 Leader 或集群其他节点发生任何网络交互。

代价：这种读法不保证线性一致性。如果该节点因为网络延迟或分区，其本地数据落后于 Leader，客户端就会读到旧数据（Stale Read）

## 取舍
Raft 解决的是“写”的共识问题，它本身并没有规定“读”必须怎么实现。
'etcd 作为工程系统，把“读”拆成了两个不同语义的产品选项，让调用方根据业务场景自己权衡。]
这不是架构上的妥协，而是一种安全默认 + 显式降级的设计选择。