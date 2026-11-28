`std::atomic` 用于实现原子操作，保证多线程访问共享变量时，操作不会被其他线程观察到中间状态。
但原子性不等于内存可见性，C++ 还提供了不同的内存序（Memory Order），用于约束线程之间的内存操作顺序。

- **Release（释放语义）**：保证当前线程在 release 操作之前的内存写入，不会被重排到 release 之后。
- **Acquire（获取语义）**：保证当前线程在 acquire 操作之后的内存读取，不会被重排到 acquire 之前。


- 优点：相比 `memory_order_seq_cst`，acquire/release 提供更弱的顺序约束，通常有利于性能优化。
- 优点：可以建立线程之间的数据发布与获取关系。
- 缺点：内存序使用错误可能导致数据竞争、读取到未初始化数据等问题。
- 缺点：需要理解 happens-before 和 synchronizes-with 关系，调试难度较高。