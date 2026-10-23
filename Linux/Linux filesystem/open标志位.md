#面试-存储内核-复习 什么是 O_SYNC、O_DSYNC、fsync、fdatasync 的差异。
## O_SYNC / O_DSYNC 
打开文件时带 O_SYNC，之后每一次 write() 返回时，数据和所有元数据都已经到达持久介质。你不需要再调 fsync。代价是每次 write 都等一次完整的落盘，吞吐会掉得比较厉害。

和 O_SYNC 几乎一样，唯一区别是：write 返回时保证的是数据完整性，而不是文件完整性。那些不必须的元数据（比如 mtime）可以不刷。所以理论上 O_DSYNC 比 O_SYNC 略轻一点，因为它可能少一次 inode 更新。



|一次性调用|每次 write 自动生效|刷什么|
|---|---|---|---|
|文件完整性|`fsync`|`O_SYNC`|数据 + 全部元数据|
|数据完整性|`fdatasync`|`O_DSYNC`|数据 + 必须的元数据|

**一个容易踩的坑**：O_SYNC/O_DSYNC 是 open 时设的“持续生效”策略，fsync/fdatasync 是“按需触发”的一次性动作。如果你已经用了 O_SYNC，再调 fsync 基本是多余的。反过来，如果你用普通方式 open，靠 fsync/fdatasync 来控制落盘时机，灵活性更高，也是更常见的工程做法。