官方使用CAS + read/write 双map实现的无锁读,细粒度写的并发安全map
核心设计是
```go
type Map struct {
	read automic.Value //只读结构，存放map[interface{}]*entry
	mu sync.Mutex
	dirty map[interface{}]*entry
	misses int //读未命中计数器，用于触发dirty提升
}
```

read 指向一个只读结构，不用加锁就能读写
dirty 写操作这里进行，需要加锁
misses 当 read 里找不到 key 而必须去 dirty 找时累加，超过阈值会把 dirty 提升为新的 read，摊销锁开销。
删除不是真正删除，而是先标记延迟清理，减少rehash的问题