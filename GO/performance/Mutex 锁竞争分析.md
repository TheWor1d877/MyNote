锁竞争 (Lock Contention): 当多个 goroutine 同时尝试获取同一个锁时，除了第一个成功的，其他都必须阻塞等待

| 场景 | C++ | Go |
|------|-----|----|
| 无竞争 | 几乎零开销（原子操作） | 几乎零开销（fast-path） |
| 有竞争 | 线程被 OS 挂起，上下文切换开销巨大 | goroutine 被 runtime 挂起，开销较小，但累积效应依然严重 |

## 使用pprof分析Mutex锁竞争
