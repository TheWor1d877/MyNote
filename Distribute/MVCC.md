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
只有当当前只等于预期值的时候，才更新为新值，否则失败，CAS操作本身是原子操作
失败就重试（乐观锁思想）
###### 显示锁
是一种悲观锁
使用场景：
- 冲突概率高
- 重试成本大（可能涉及到外部的系统调用）

#### 写偏斜
两个事物都依据自身的快照判断合法，但是两个事物相加是不合法的,需要升级到Serializable隔离级别或者增加应用层校验

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

####
Qwen3-Max
MVCC 面试题精讲：从原理到分布式场景（附答案模板）
作为你的老师，我整理了 云原生/分布式存储方向最常考的 MVCC 面试题，并给出技术深度 + 回答技巧。这些问题在阿里、腾讯、字节、PingCAP 等公司的后端/存储岗面试中高频出现。
🔥 一、基础概念类
Q1: 什么是 MVCC？它解决了什么问题？
考察点：核心思想理解
回答模板：
“MVCC（Multi-Version Concurrency Control）是一种无锁并发控制机制，通过为数据维护多个历史版本，实现读写不互斥。
它主要解决传统锁机制下的两大痛点：
1️⃣ 读阻塞写：读操作需加共享锁，阻止写入
2️⃣ 写阻塞读：写操作加排他锁，阻止读取
在 MVCC 下，读操作可直接访问历史版本，无需等待写事务提交，大幅提升并发性能。”
加分项：
“在分布式系统中，MVCC 还是实现快照隔离（Snapshot Isolation） 和时间点恢复（PITR） 的基础。”
Q2: MVCC 能解决所有并发问题吗？
考察点：对 MVCC 局限性的认知
回答模板：
“不能。MVCC 主要优化读-写冲突，但对写-写冲突仍需配合其他机制：
丢失更新（Lost Update）：两个事务同时修改同一行，后提交者覆盖前者的修改
→ 需用 CAS（Compare-And-Swap） 或 显式锁 解决
写偏斜（Write Skew）：事务基于过期快照做业务判断
→ 需升级到 Serializable 隔离级别 或使用 应用层校验”
举例：
“银行转账场景：A、B 账户总和 ≥ 0。两个事务同时从 A 转出，可能都基于初始快照判断合法，导致总和 < 0。”
⚙️ 二、实现原理类（MySQL InnoDB 为例）
Q3: MySQL 中 MVCC 是如何实现的？
考察点：底层实现细节
回答模板（分三部分）：
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
图示辅助：



##  RR（可重复读）和 RC（读已提交）隔离级别下 MVCC 行为有何不同？
考察点：隔离级别与 MVCC 的协同
回答模板：
“关键区别在于 Read View 的创建时机：
RC 级别：每次 SELECT 都创建新 Read View
→ 能看到其他事务最新已提交的数据（不可重复读）
RR 级别：事务首次 SELECT 时创建 Read View，后续复用
→ 整个事务看到一致快照（可重复读）
注意：
当前读（如 SELECT ... FOR UPDATE）不走 MVCC，直接加锁读最新数据
幻读在 RR 级别通过 间隙锁（Gap Lock） 解决，非 MVCC”