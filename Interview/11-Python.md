# Python 面试题库（修订版）

覆盖 GIL 与并发模型、内存管理、C 扩展与 PyTorch 数据加载的面试题，共 45 题，标注 ★ 的为面试必会 P0 题。

## GIL 与并发模型

★1. 解释 CPython 的 GIL 是什么，它保护了什么，又不保护什么。
2. 为什么 GIL 对 CPU 密集型多线程程序影响巨大，但对 I/O 密集型程序影响较小？
★3. 解释 Python 多线程、多进程、asyncio 三种并发模型的适用场景与代价。
4. 什么是 asyncio 事件循环？它如何调度协程，以及和 Go netpoll 的异同。
5. 解释 async/await 的本质，协程在什么时刻会让出控制权。
6. 什么是阻塞事件循环？哪些常见调用会阻塞 asyncio，如何避免？
7. 解释 asyncio 中的 Task、Future、Coroutine 的区别。
8. 什么是 asyncio 背压？如何用 Semaphore、Queue 控制并发。
9. 解释 Python 多进程中的 fork、spawn、forkserver 启动方式差异。

## 内存管理与序列化

★10. 在 PyTorch DataLoader 中，num_workers > 0 时数据是如何跨进程传递的？
★11. 解释 Python 对象序列化 pickle 的开销，以及它如何影响 DataLoader 吞吐。
12. 什么是共享内存？Python 中如何用 shared_memory 或 mmap 降低数据拷贝？
13. 解释 Python 内存管理：引用计数、循环 GC、内存池 pymalloc。
14. 什么是内存泄漏在 Python 中的表现？如何用 tracemalloc、objgraph 排查？
15. 解释 Python 对象的 __slots__ 的作用，以及它对内存和访问速度的影响。
16. 什么是弱引用？它在缓存和避免循环引用中的价值是什么。

## C 扩展与 CUDA 交互

17. 解释 Python 的 GIL 在 C 扩展中的释放，为什么 numpy、PyTorch 能绕过 GIL 并行。
18. 什么是 C 扩展？pybind11、ctypes、cffi 各自的适用场景与开销。
19. 解释 Python 调用 CUDA 的路径，以及同步、异步调用对训练数据加载的影响。
20. 什么是 CUDA Stream 与 Python 异步的交互，如何避免不必要的同步。

## PyTorch DataLoader

21. 解释 PyTorch DataLoader 的 collate_fn 的作用，以及它可能成为瓶颈的原因。
★22. 什么是 pin_memory？在 DataLoader 中它如何与 CUDA 拷贝配合？
★23. 解释 DataLoader 的 prefetch_factor 与 num_workers 的协作关系。
24. 什么是 IterableDataset？它相比 Map-style Dataset 在流式数据场景中的优势是什么。
★25. 解释 PyTorch Sampler 与 DistributedSampler 的工作原理，以及 epoch 边界如何处理。
26. 什么是 DataLoader 的 worker 重启？它如何影响训练稳定性。

## 服务性能与工程实践

27. 解释 Python logging 在高并发推理服务中的性能陷阱，如何做异步日志。
28. 什么是 Python 的 GIL 争用？如何用多进程或 C 扩展规避。
29. 解释 Python 的垃圾回收对低延迟推理服务的影响，如何调优 gc 参数。
30. 什么是 Python 的 frozenset、tuple、dict 在哈希与内存上的差异。
31. 解释 Python 生成器与迭代器的惰性求值，它在流式数据管道中的价值。
32. 什么是 context manager？它在资源释放与 CUDA 上下文管理中的作用。
33. 解释 Python 的异常处理开销，以及热点路径中如何避免异常驱动控制流。
34. 什么是 Python 的 typing 与运行时开销，typing 是否影响推理性能。

## 思考与取舍题

36. DataLoader 的 num_workers 超过物理核数后反而变慢：描述这条性能曲线的机制，并说明如何区分它是"过度订阅 CPU"还是"内存带宽已打满"。
37. 一个 100 MB 的 tensor 走 pickle 跨进程传输：它实际会经历哪几次内存拷贝？哪一步最容易被忽略？
38. C 扩展释放 GIL 后线程真正并行了，CPU 反而可能成为新瓶颈：从内存带宽与缓存竞争的角度，辩论"线程数 = 核数"这条经验还剩多少效力。
39. fork 出的子进程里，共享内存对象与普通对象的引用计数分别会发生什么？这个差异如何在"看起来正常、跑一天就泄漏"的现象里埋雷？
40. gc.freeze 在 DataLoader worker 启动时到底冻结了什么？在什么负载特征下它值得调用，什么情况下只是安慰剂？
41. 现象诊断：多进程训练的常驻内存随 epoch 单调上涨。候选原因：(a) fork 后 COW 页被写触碰逐渐私有化；(b) shared_memory 段创建后未正确 close/unlink；(c) worker 内的缓存容器无界增长。请排序并说明你会先抓哪个证据。
42. 反直觉追问：把预处理逻辑从 Python 改写成 C 扩展后，wall time 降了、CPU 总占用却升了——这一定是坏事吗？
43. 取舍辩论：batch 跨进程传递用 pickle 走管道，还是用 shared_memory 加环形缓冲？请在 batch 很小与 batch 很大两种极端下分别辩论一次。
44. 量级估算：batch 为 256×3×224×224 的 float32、每秒 50 个 batch 走 pickle 进管道：估算管道需要承载的字节率，并判断瓶颈会先出现在序列化 CPU 还是管道带宽。
45. 假设破坏：如果 CPython 明天彻底移除 GIL，现有"多进程 DataLoader + 主进程聚合"的架构里哪些设计要推倒重审？哪些理由依然成立？

## 综合设计题

★35. 如果让你优化一个 PyTorch 训练数据加载 pipeline，你会从 GIL、多进程、共享内存、序列化、pin memory、CUDA Stream 哪些角度入手？
