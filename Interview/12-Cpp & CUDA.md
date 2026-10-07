# C++ & CUDA 面试题库（修订版）

覆盖 C/C++ 内存模型与并发、编译链接、CUDA 执行模型与性能分析的面试题，共 53 题，标注 ★ 的为面试必会 P0 题。

> **分层使用说明（2026-10-06 起生效，按岗位定位重新标层）**
> - **P0 必会（★）**：C/C++ 内存布局、对齐、malloc/RAII/智能指针、atomic 内存序、SIMD、编译链接（1–26）——平台岗写 Go 之外，K8s 组件、存储引擎、CUDA 组件的 C/C++ 依赖都需要这层常识；
> - **P1 概念级**：CUDA 执行模型、Stream、统一内存（27–36）——P1 的 §9.4 知识边界题已覆盖概念，这里只要求"知道 Stream 解决什么、为什么统一内存会有页迁移抖动"；
> - **P2 跳过**：CUDA kernel 优化、CUDA Graph、occupancy、bank conflict、maxrregcount、CUDA Graph 捕获限制（37–40、42–53）——CUDA kernel 工程师主场，平台岗只需知道"FlashAttention/CUDA Graph 存在，降低 launch 开销"这一句话级结论；
> - **岗位分工一句话**：平台岗考 CUDA 的上限是"知道数据怎么进 GPU、Stream 怎么重叠"，下限是"别去和 CUDA 工程师拼 kernel 优化"。

## C/C++ 内存模型与数据布局

★1. 解释 C 的内存布局：栈、堆、BSS、data、rodata 各自存放什么。
2. 什么是未定义行为？举例说明它为什么可能让编译器做出反直觉优化。
3. 解释指针与数组的区别，以及数组退化为指针时丢失了什么信息。
4. 什么是指针对齐？为什么未对齐访问在某些架构上会崩溃或变慢。
★5. 解释结构体内存对齐与 padding，以及如何通过重排字段减小结构体大小。
★6. 什么是缓存友好的数据布局？AoS 与 SoA 分别适合什么访问模式。
7. 解释 C 的 volatile 关键字，它和原子操作、内存屏障的区别是什么。
8. 什么是 restrict 指针？它如何帮助编译器优化。

## 内存管理与 C++ 抽象

★9. 解释 malloc/free 的底层实现思路，以及常见分配器如 jemalloc、tcmalloc 的差异。
10. 什么是内存池？在高吞吐数据管道中为什么常用内存池替代频繁 malloc。
★11. 解释 C++ RAII，以及它如何保证异常安全与资源释放。
12. 什么是移动语义？std::move 真的移动了什么？
13. 解释左值、右值、完美转发，以及它们对零拷贝封装的意义。
★14. 什么是智能指针 unique_ptr、shared_ptr、weak_ptr？各自开销与适用场景。
15. 解释 shared_ptr 的引用计数线程安全性，以及它不能保证什么。
16. 什么是虚函数表？虚函数调用相比普通函数调用多出哪些开销。
17. 解释 C++ 模板的编译期展开，以及它如何实现零开销抽象。
18. 什么是 CRTP？它在高性能库中如何避免虚函数开销。

## 并发与原子操作

★19. 解释 std::atomic 的内存序：relaxed、acquire、release、seq_cst。
20. 什么是无锁队列？ABA 问题如何产生，如何用 tagged pointer 或 hazard pointer 解决。
21. 解释 false sharing 在 C/C++ 中的表现，以及如何用 alignas 缓解。

## 编译、链接与向量化

★22. 什么是 SIMD？编译器自动向量化需要满足哪些条件。
23. 解释编译链接过程：预处理、编译、汇编、链接各自做什么。
24. 什么是静态链接与动态链接？它们对容器镜像大小和启动速度的影响是什么。
25. 解释符号可见性与 -fvisibility=hidden 在大型 C++ 项目中的作用。
26. 什么是 ABI？为什么 C++ ABI 兼容性比 C 更难保证。

## CUDA 执行模型与显存访问

★27. 解释 CUDA 的线程层次：thread、warp、block、grid 的关系。
28. 什么是 warp divergence？它如何影响 kernel 性能。
★29. 解释 CUDA 的 shared memory、bank conflict，以及如何优化访问模式。
30. 什么是 CUDA occupancy？寄存器压力、共享内存如何限制并发 warp 数。
★31. 解释 CUDA 的 global memory 合并访问，什么样的访问模式最差。

## CUDA 内存管理

★32. 什么是 CUDA Stream 与 Event？如何用它们实现拷贝与计算重叠。
33. 解释 cudaMalloc 与 cudaMallocAsync 的差异，以及显存池的价值。
34. 什么是统一内存？它在训练与推理中的优缺点是什么。
35. 解释 GPU 页错误与 UVM 迁移，为什么它可能造成性能抖动。
36. 解释 cudaHostRegister 注册内存的原理与代价，以及 GPUDirect Storage 对内存注册（预先注册的固定缓冲区）提出了哪些要求？

## Kernel 优化与性能分析

★37. 什么是 CUDA Graph？它如何降低 kernel launch 开销，捕获时有哪些限制。
38. 解释 kernel fusion 在 CUDA 层面的收益来源。
★39. 什么是 Nsight Systems 与 Nsight Compute？分别用来定位哪类问题。
40. 解释如何用 Nsight 判断一个 kernel 是 compute-bound、memory-bound 还是 latency-bound。

## 思考与取舍题

42. shared memory 占满不一定比走 L2 快：构造一个"把数据全塞进 shared memory 反而更慢"的场景，并说明此时该看哪个计数器来裁决。
43. warp divergence 消除后 kernel 反而变慢：给出至少两种可能的解释，并说明哪种情况下"消除分支"其实是伪优化。
44. cudaMallocAsync 池化消除了频繁分配开销，却引入碎片的新形态：描述池化后碎片长什么样，以及它与 cudaMalloc 时代碎片在排查方式上的差异。
45. 统一内存的页迁移抖动在高带宽场景反而可能可接受：构造这样一个场景，并给出你判断"可接受"的量化标准。
46. occupancy 冲到 100% 性能反而下降：是资源竞争、缓存污染还是调度开销？设计一组实验把它定位出来。
47. CUDA Graph 捕获后遇到动态 shape 回退到逐 kernel launch：估算高频小 kernel 场景下这次回退的代价量级，并辩论"为 Graph 牺牲 shape 灵活性"值不值。
48. 现象诊断：Nsight 显示 kernel 是 memory-bound，把 occupancy 提高后耗时纹丝不动。候选原因：(a) DRAM 带宽已打满，并发再高也无济于事；(b) L2 命中率下降抵消了收益；(c) kernel 实际卡在延迟等待而非带宽不足。请排序并指出对应的证据。
49. 反直觉追问：循环展开 8 倍导致寄存器溢出到 local memory，性能有时不降反升——什么条件下这是合理的？
50. 取舍辩论：用 maxrregcount 强行压寄存器避免溢出，还是放任编译器自由分配？在"单 kernel 耗时占比高"与"多 kernel 排队"两种上下文里分别投票。
51. 量级估算：一个 kernel 在 108 个 SM 上每周期各读取 128 字节、主频 1.4 GHz：估算它需要的显存带宽并与 HBM 峰值比较；带宽利用率到多少时你才承认"已经打满"？
52. 假设破坏：下一代卡显存带宽翻倍但容量减半：你现有的 kernel 优化清单里，哪些优先级要上调，哪些可以直接删掉？
53. 现象诊断：两个理论访存量相同的 kernel（SoA 与 AoS 布局）实测相差 4 倍。候选原因：(a) 全局内存合并访问粒度差异；(b) L2 sector 利用率不同；(c) warp 发射与调度瓶颈。请排序并给出各自的关键指标。

## 综合设计题

41. 如果让你写一个 GPU 数据预取或 KV Cache 搬运的 CUDA 组件，你会如何设计线程层次、内存布局、Stream 重叠与错误处理？
