# 推理引擎与 vLLM（分级版）

按重要性分三档：T1 必须能脱稿讲满 3–5 分钟；T2 能讲清 90 秒；T3 为扩展与速查。★ 仅表示原题库标记，已不完全等同 T1。

## T1 · 必须脱口而出

1. ★ 解释大模型推理的两个阶段：prefill 与 decode，它们的计算特征和瓶颈有何不同？
2. ★ 为什么 prefill 阶段通常 compute-bound，而 decode 阶段通常 memory-bandwidth-bound？
3. ★ 解释 TTFT 与 TPOT 的定义，它们分别受哪些系统层因素影响？从网关、调度、KV Cache、批处理角度如何优化首 token 延迟？
4. ★ 什么是 continuous batching？它相比静态 batching 的优势是什么？
5. ★ 什么是 PagedAttention？它如何借鉴虚拟内存思想管理 KV Cache？
6. ★ 解释 KV Cache 的 block 管理与 paged 布局，block size 的选择如何影响显存碎片、显存带宽利用与吞吐？
7. ★ 解释 Prefix Caching 的命中条件与 hash 机制（block hash 如何识别可复用前缀），LRU 淘汰如何与之协作？它在多轮对话和 RAG 场景中的价值是什么？
8. ★ 解释 KV Cache offload 到 CPU 内存或 NVMe 的取舍、换入换出机制与 vLLM swap 的触发条件，如何避免换入换出成为 PCIe 瓶颈？
9. ★ KV 感知路由：推理网关为什么需要知道 KV Cache 的位置？缓存位置注册与租约机制应如何设计？
10. ★ 解释推理网关的限流算法：令牌桶、漏桶、滑动窗口。
11. ★ 什么是 GPU 利用率？为什么它可能高但吞吐仍然低？
12. ★ 为什么 KV cache 剩余量比 GPU 利用率更适合作为推理服务弹性伸缩的信号？基于它做 HPA/KEDA 扩缩容要解决哪些问题？
13. ★ 解释 prefill-decode 分离（disaggregated prefill/decode）架构，它如何让两类资源独立扩缩容？
14. ★ 如果让你从零设计一个“vLLM + Kubernetes + 高并发网关”的推理基础设施，你会如何划分控制面、数据面、缓存层、调度层与可观测性层？

## T2 · 应当会答

15. 解释 vLLM 的调度器如何决定每一步哪些请求进入 batch。
16. 解释 prefill 与 decode 混合批处理，以及它为什么能提升 GPU 利用率。
17. 什么是 chunked prefill？vLLM 调度器中它如何与 decode 优先级协同？它如何改善长 prompt 下的延迟与吞吐？
18. 解释抢占与重计算，在显存不足时 vLLM 如何处理请求？
19. ★ 什么是 KV Cache 量化？它如何降低显存占用，又可能损失什么？
20. ★ prefix cache 命中率应如何测量？网关侧估计与引擎自报指标有何差异，各有什么坑？
21. ★ 什么是 speculative decoding？它如何用 draft 模型加速 decode？Medusa、EAGLE 等方法在系统层有何差异？draft 与 target 模型如何协同调度（含 vLLM 的集成方式）？
22. 什么是加权路由？在多个推理后端之间如何做负载均衡？
23. ★ 解释推理网关中的排队策略：FIFO、优先级队列、公平调度。
24. 什么是并发限制与队列深度控制？它们如何保护后端 GPU？
25. ★ 什么是 SSE 与 WebSocket？流式推理输出分别适合哪种协议？
26. 什么是背压传播？网关如何把后端压力传递到客户端？
27. 如何搭建每百万 token 的成本模型？需要计入哪些变量（缓存命中折扣、批处理效率、闲时摊销等）？

## T3 · 扩展与速查

28. SSE 流式转发有哪些工程坑？flush 时机、客户端取消传播、禁用压缩、超时重置分别怎么处理？
29. 多租户推理服务如何设计 token 预算与限流？请覆盖账户级、请求级、token 级三层控制与超卖策略。
30. ★ 解释 vLLM V1 引擎架构相比早期版本的关键变化。
31. ★ 解释 KV Cache 在 prefill 节点与 decode 节点之间的传输问题。
32. 把 chunked prefill 的块大小从 512 调到 8192，TTFT 与 TBT 各会滑向哪个极端？"块越大越接近完整 prefill"在什么负载下成立、什么负载下失效？
33. 什么是 sampling 参数对 decode 性能的影响？
34. 什么是推理服务的 GPU 时间片与优先级抢占？
35. 什么是 NVLink 域内通信与跨域通信的差异？
