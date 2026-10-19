#面试-Go语言-复习 解释 Go 的 pprof：CPU、heap、block、mutex、goroutine profile 分别怎么看

pprof观察Go的runtime程序内部的执行情况的，分为两大类： 采样型与快照型
CPU、block、mutex、heap 是采样，goroutine 是快照。

## CPU profile
采样，看的是“CPU 时间花在哪，top 看哪些函数占的 CPU 最多
采样是有偏的，采样窗口太短会漏掉低频但耗时的路径，一般至少跑 30 秒。
他只统计这在CPU上面跑的时间，IO等待，锁等待都看不到

## heap profile
采样，看的是堆内存
两个视角： -inuse_space 和 -alloc_space
inuse 是当前还活着的对象占了多少，alloc 是从程序启动到现在总共分配了多少
看 inuse 才能发现“谁一直不释放

## block profile
看的是 goroutine 在阻塞等待上花了多少时间——等 channel、等锁、等 IO 都算
典型场景：channel 那头没人收，一堆 goroutine 卡在 send 上，block profile 一眼就能看出来。

## mutex profile
也是采样，专门看锁竞争
它跟 block 的区别是：block 看所有阻塞，mutex 只看锁等待
注意它只统计没立刻拿到锁的情况，能立刻拿到的锁不记录，所以它反映的是竞争程度，不是锁的总耗时

## goroutine profile
看的是当前所有 goroutine 的调用栈，相同调用栈的 goroutine 会被合并，数量看着不直观
通过goroutine的完整栈能看出来现在有多少Goroutine，都卡在哪里

## 整体
先看 goroutine 数量有没有异常涨，锁定了可疑方向；如果是 CPU 高，上 CPU profile；如果是内存涨，上 heap 的 inuse；如果是“CPU 不高但就是慢”，那大概率在等，上 block 和 mutex
## 使用
[[GO/performance/pprof 启用与 CPU 剖析|pprof 启用与 CPU 剖析]]