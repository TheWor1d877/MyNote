传统数据库追求的是高并发，顺序IO，强一致型
而AI Infra追求的是高吞吐，随机IO，一般情况下追求的是最终一致性

## B+ Tree vs LSM-Tree
B+树为了支持高效的随机读（O(log N)），必须保证数据在磁盘上相对有序。但随机写入会导致严重的页分裂（Page Split）和随机 I/O。在机械硬盘时代，这是致命的，所以 B+树高度依赖 Buffer Pool（Page Cache）（现在内存中写好，最后在后台慢慢刷新到硬盘中）来吸收随机写。

结果就是：
写放大：插一行，可能要移动半个页的数据。
随机 I/O：新页、旧页、父页，散落在磁盘不同位置，磁头来回跑。
页分裂频繁：随机插入时，页的填充率可能只有 50%，空间浪费，树也更容易变高。

这就是为什么现代 AI 存储的元数据层和 KV Cache 存储（如基于 Redis 的 PagedAttention 存储）更倾向于 LSM-Tree（Log-Structured Merge-Tree）。LSM 将随机写转化为顺序追加（Append-only），完美契合 NVMe SSD 的物理特性，且天然支持零拷贝（Zero-Copy）和 RDMA 直接读取。

## 事务隔离与MVCC
PostgreSQL 的 UPDATE 不是原地修改，而是“插入新版本 + 标记旧版本无效（xmax）”，这导致严重的表膨胀（Bloat），需要后台 VACUUM 清理。

InnoDB 则通过 Undo Log 维护版本链。在 RR（可重复读）级别下，为了防止幻读，InnoDB 会动用 Next-Key Lock（行锁+间隙锁），这在热点数据更新时会导致严重的锁等待。

在 AI 训练数据流中，数据通常是只读的（Read-only），或者只有后台异步的 Append 操作。我们根本不需要 MVCC 这种为了“读写并发”而设计的沉重机制。在构建训练数据网关时，我们可以采用无锁的内存映射（mmap）或基于 Epoch 的无锁队列。

在推理服务的 KV Cache 管理中，KV Cache 的生命周期由 Attention 机制严格绑定，不需要复杂的事务隔离，只需要基于引用计数的内存分配器（如 vLLM 的 PagedAttention），这比 RDBMS 的 MVCC 轻量了几个数量级。

## 主从复制与高可用
传统RDBMS的主从复制（PG为例）追求不丢数据，通过 walsender 和 walreceiver 传输 WAL，存在主从延迟（Seconds_Behind_Master），半同步复制（Semi-Sync），增加了网络延迟
AI Infra中的追求的是RTO最小化，所以比起传统RDBMS的增量Checkpoint，更愿意做全量异步Checkpoint或者利用类似 RDMA 的技术直接将 GPU 显存中的权重 DMA 到远端存储，绕过 CPU 和 Page Cache，把 Checkpoint 耗时从分钟级压缩到秒级。

## 索引优化
在推理服务的 KV Cache 管理中，vLLM 的 PagedAttention 本质上就是一个针对 GPU 显存优化的“索引结构”。传统的 KV Cache 需要分配连续内存，导致严重的内存碎片（类似 B+树页分裂）。

PagedAttention 将 KV Cache 分成固定大小的 Block（类似 B+树的 Page），通过一个轻量级的 Block Table（类似二级索引）进行非连续映射。这不仅消除了内存碎片，还让 KV Cache 的换入换出（Swap in/out）变成了类似操作系统的 Page Fault 处理，极大地提升了高并发下的显存利用率。