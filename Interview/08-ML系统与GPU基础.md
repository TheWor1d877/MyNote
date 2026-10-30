# ML 系统与 GPU 基础（分级版）

按重要性分三档：T1 必须能脱稿讲满 3–5 分钟；T2 能讲清 90 秒；T3 为扩展与速查。★ 仅表示原题库标记，已不完全等同 T1。

## T1 · 必须脱口而出

1. ★ 解释数据并行、张量并行、流水线并行的切分维度与通信模式。
2. ★ 解释 AllReduce、AllGather、ReduceScatter 的数学语义与通信量。
3. ★ 什么是 KV Cache？它缓存的是什么，为什么 decode 阶段必须依赖它。
4. ★ 解释 KV Cache 的数学含义，为什么它只缓存 K 和 V。
5. ★ 什么是 GQA 与 MQA？它们如何减少 KV Cache 显存。

## T2 · 应当会答

6. ★ 什么是 ZeRO 的三个阶段？它分别切分优化器状态、梯度和参数。
7. ★ 解释 draft model 与 target model 的接受率如何影响加速比。
8. 量级估算：参数量翻倍，训练显存需求为什么远不止翻倍？请把参数、梯度、优化器状态与激活值的账拆开算一遍。
9. ★ 解释 CUDA 的线程层次：thread、warp、block、grid 的关系。
10. ★ 解释 CUDA 的 shared memory、bank conflict，以及如何优化访问模式。

## T3 · 扩展与速查

11. ★ 解释 CUDA 的 global memory 合并访问，什么样的访问模式最差。
12. ★ 什么是 CUDA Stream 与 Event？如何用它们实现拷贝与计算重叠。
13. ★ 什么是 CUDA Graph？它如何降低 kernel launch 开销，捕获时有哪些限制。
14. 解释通信与计算重叠的常见实现方式。
15. 什么是 MoE？它的路由机制对系统负载均衡提出什么要求。
