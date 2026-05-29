- 传统机制锁：
读操作会阻塞写操作（需要加共享锁）
写操作会阻塞读操作（需要加排他锁）
高并发下性能急剧下降 → “读写互斥”瓶颈
- MVCC核心：版本快照
版本标识： 为每个数据版本添加“时间戳”
版本链： 用一条链表串起来同一个Key数据的所有版本
读视图： 定义事务“能看到哪些版本

## 版本存储结构
行存数据库（如 MySQL InnoDB）：
隐藏列：DB_TRX_ID（创建事务ID）、DB_ROLL_PTR（指向 undo log）
历史版本存于 undo log 链
LSM-Tree 存储（如 RocksDB, TiKV）：
Key 编码为 (user_key, timestamp) → (user:1001, 105)
多版本天然存在于 SSTable 中

## 性能
| 场景 | 锁机制 | MVCC |
|------|--------|------|
| 高并发读 | 所有读排队等锁 | 无锁并行读 |
| 长事务读 | 阻塞所有写入 | 写入可继续（生成新版本） |
| 写密集 | 写锁竞争激烈 | 写写冲突仍需处理（但读不受影响） |
| 存储开销 | 低 | 高（需保留历史版本） |

| 系统 | 存储引擎 | 时间戳方案 | 特点 |
|------|----------|------------|------|
| MySQL InnoDB | B+Tree | 事务ID | undo log 存历史版本 |
| PostgreSQL | Heap Table | 事务ID | 行内直接存 xmin/xmax |
| TiDB/TiKV | RocksDB (LSM) | PD 分配TS | 分布式快照隔离 |
| CockroachDB | RocksDB | HLC | 支持外部一致性 |
| etcd | BoltDB | Raft Index | 简化版 MVCC（仅用于 watch） |

## 注意
MVCC无法解决写写之间的冲突

#### 丢失更新
丢失更新：两个事务同时修改同一行，后提交者覆盖前者的修改，需要使用CAS或者显示锁解决问题
###### CAS
只有当当前只等于预期值的时候，才


## MySQL中的MVCC实现方法
1️⃣ 隐藏字段
“InnoDB 为每行记录添加两个隐藏列：
DB_TRX_ID：最近修改该行的事务 ID
DB_ROLL_PTR：指向 undo log 中的旧版本”
2️⃣ Undo Log 版本链
“每次 UPDATE/DELETE 时：
将旧行复制到 undo log
新行 DB_ROLL_PTR 指向该 undo 记录
形成以当前行为头的版本链”
3️⃣ Read View 可见性判断
“事务启动时创建 Read View，包含：
m_ids：当前活跃事务 ID 列表
min_trx_id：m_ids 中最小值
max_trx_id：下一个将分配的事务 ID
可见性规则：
若版本 DB_TRX_ID < min_trx_id → 可见（已提交）
若版本 DB_TRX_ID ∈ m_ids → 不可见（未提交）
若版本 DB_TRX_ID ≥ max_trx_id → 不可见（未来事务）”