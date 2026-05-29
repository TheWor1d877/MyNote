Mutex 的扩展，适用于读多写少的场景。它允许多个 reader 同时持有读锁，但 writer 必须独占写锁。
关键方法
读操作: RLock(), RUnlock()
写操作: Lock(), Unlock()

任意数量的 goroutine 可以同时持有读锁。
只能有一个 goroutine 持有写锁，并且此时不能有任何读锁。
如果存在写锁，新的读锁请求会被阻塞。