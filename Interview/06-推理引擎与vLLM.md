# 推理引擎与 vLLM（分级版）

按重要性分三档：T1 必须能脱稿讲满 3–5 分钟；T2 能讲清 90 秒；T3 为扩展与速查。★ 仅表示原题库标记，已不完全等同 T1。新增题带【要点】，其中的项目版本、组件与指标名按 2026-10-31 核实，厂商性能数一律标为卖方口径；开口先报版本。

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

28. 网关侧估前缀命中与订阅引擎的 KV 事件，差别在哪？llm-d 的 approximate 与 precise 两条实现各自怎么工作？
   【要点】approximate 由 `approx-prefix-cache-producer` 加 `prefix-cache-scorer` 完成，用字符到 token 的启发式在网关内存里维护 LRU 前缀索引，引擎不需要配合，成本低但会随引擎淘汰而偏离真实。precise 由 `precise-prefix-cache-producer` 加 KV-Cache Indexer 完成，Indexer 订阅引擎事件，vLLM 侧入口是 `--kv-events-config`（`KVEventsConfig`：`enable_kv_cache_events`、`publisher: "null"|"zmq"`、默认端点 `tcp://*:5557`、`replay_endpoint`、`buffer_steps`、`hwm`），另有 speculative indexing 补未落定的部分。取舍按三问：事件扇出量扛不扛得住（15 章算）、引擎重启与网关重启后索引怎么重建（replay 通道）、跨引擎是否要求同一套语义。他的冷启动预热角度接 19 章 3 题与 8 题。vLLM 侧还有 `vllm:kv_block_lifetime_seconds`、`vllm:kv_block_idle_before_evict_seconds`、`vllm:kv_block_reuse_gap_seconds` 可直接用来验证近似索引的陈旧度。

29. prefill 与 decode 分离之后，KV 从哪条路搬运？把 connector 与传输层的名字说全。
   【要点】vLLM 的配置入口是 `--kv-transfer-config`，字段 `kv_connector` 与 `kv_role: kv_producer|kv_consumer|kv_both`；v0.32.0 的 `vllm/distributed/kv_transfer/kv_connector/v1/` 目录下有 `NixlConnector`（含推送式 NIXL）、`MooncakeConnector`、`MooncakeStoreConnector`、`LMCacheConnectorV1`、`LMCacheMPConnector`、`FlexKVConnectorV1`、`MoRIIOConnector`（ROCm）、`SimpleCPUOffloadConnector`、`MultiConnector`，官方文档把这一整套标 experimental。SGLang 用 `--disaggregation-mode prefill|decode`、`--disaggregation-transfer-backend`（mooncake、nixl、mori、ascend）、`--disaggregation-ib-device`。传输层数字（自报口径，条件要复述）：Mooncake Transfer Engine 搬 40 GB（128k token 的 LLaMA3-70B KV）在 4×200Gbps RoCE 下 87 GB/s、8×400Gbps 下 190 GB/s，约为 TCP 的 2.4 倍与 4.6 倍。任何厂商性能数都要带「谁测的、什么负载」，这是 16 章第五节的规矩。

30. KV 卸载分几层？每层的方案、容量与延迟账怎么排？
   【要点】引擎内：vLLM 原生 `OffloadingConnector`（`CPUOffloadingSpec`、`TieringOffloadingSpec`，tier 可为 fs、obj、p2p）与 `SimpleCPUOffloadConnector`。外部：LMCache、Mooncake Store（`MooncakeStoreConnector`）、NVIDIA KVBM。SGLang 的 HiCache 是三层结构，L1 GPU、L2 宿主 DRAM、L3 外部存储，参数 `--enable-hierarchical-cache`、`--hicache-ratio`、`--hicache-size`、`--hicache-storage-backend mooncake|hf3fs|nixl|aibrix|file`、`--hicache-write-policy`、`--hicache-storage-prefetch-policy`。AIBrix v0.8.0 分 L1（Pod 内 DRAM，S3FIFO 策略）与 L2（分布式 KV 集群，后端 InfiniStore/HPKV/PrisKV，Redis 做元数据），由 KVCache CR 供给。排序依据回到 15 章：PCIe 4.0 x16 有效 ~25 GB/s、NVMe 顺序读 3–7 GB/s、对象存储首字节 10–100 ms，每降一层就多一跳进 TPOT。

31. 路由这件事的标准面到哪里为止？自己写网关与用 EPP 的边界怎么划？
   【要点】Gateway API Inference Extension v1.6.2（2026-09-17）之后：`InferencePool` 已在 `api/inference/v1`（stable），`spec.endpointPickerRef` 变成可选；完整 EPP、Body-Based Routing、Latency Predictor 三块从 GAIE 迁进 llm-d，GAIE 只保留规范、CRD、conformance 与一个 Lightweight EPP（LWEPP）；早期 Alpha API `InferenceObjective`、`InferenceModelRewrite`、`EndpointPickerConfig` 已从 GAIE 删除，还在引用它们的方案属于过期。llm-d 侧的插件 `prefix-cache-affinity-filter`、`prefix-cache-scorer`、`lora-affinity-scorer`、`kv-cache-utilization`、`load-aware`、`session-affinity`、`topology-affinity`、`mm-cache-affinity` 目前全为 Alpha 或 Beta，Alpha 需要 `--allow-experimental-plugins` 才加载。答题落点：他做的东西与标准面重叠哪一段、上游什么时候会内置、被内置之后他剩下什么（19 章 8 题就是这个追问）。

## T3 · 扩展与速查

32. SSE 流式转发有哪些工程坑？flush 时机、客户端取消传播、禁用压缩、超时重置分别怎么处理？
33. 多租户推理服务如何设计 token 预算与限流？请覆盖账户级、请求级、token 级三层控制与超卖策略。
34. ★ 解释 vLLM V1 引擎架构相比早期版本的关键变化。
35. ★ 解释 KV Cache 在 prefill 节点与 decode 节点之间的传输问题。
36. 把 chunked prefill 的块大小从 512 调到 8192，TTFT 与 TBT 各会滑向哪个极端？"块越大越接近完整 prefill"在什么负载下成立、什么负载下失效？
37. 什么是 sampling 参数对 decode 性能的影响？
38. 什么是推理服务的 GPU 时间片与优先级抢占？
39. 什么是 NVLink 域内通信与跨域通信的差异？

40. vLLM 当前哪些能力是默认开的、哪些要显式打开？报版本号再答。
   【要点】以 v0.32.0（2026-10-18）为准：`enable_prefix_caching` 默认 True（定义在 `vllm/config/cache.py`），哈希算法字段 `prefix_caching_hash_algo` 可设 sha256；`--async-scheduling`（`SchedulerConfig.async_scheduling`）不写时自动启用，遇到 pooling 模型、不被支持的 spec decode、V1 runner 配 PP 等情况会被关掉；`--enable-sleep-mode` 把权重换出，level 1 保留权重换出 KV、level 2 两者都丢，`--sleep-preserve-parameter-names` 控制保留项；0.31/0.32 这条线去掉了 cascade attention、停止支持 Python 3.10。同一条 0.31.0（2026-10-05）、0.30.0（2026-09-22）。答默认值题的规矩：先报版本，再报字段名，说不准就承认「这条我按 X 版本记的，会后核对」。

41. llm-d、NVIDIA Dynamo、AIBrix 三条分布式推理路线各自的重点差异在哪？
   【要点】llm-d 1.0.0（2026-10-20），CNCF Sandbox 项目（2026-03-12 入选），由 IBM Research、Red Hat、Google Cloud、NVIDIA、CoreWeave 在 2025-05 发起；模块拆在 `llm-d-router`、`llm-d-kv-cache`、`llm-d-latency-predictor`、`llm-d-async`、`llm-d-batch-gateway`、`llm-d-autoscaling`，1.0 的主题是多租户 flow control（概率准入、program-aware fairness、turn-priority）、路由高可用与零停机更新、`DisaggregatedSet`、session-aware routing 与 `previous_response_id`，基线镜像 pin vLLM v0.31.0。Dynamo v1.6.1（2026-10-27）的组件名：KV Block Manager（pip 包 `kvbm`，对 SGLang 的支持在 README 矩阵里仍标施工中，vLLM 与 TRT-LLM 已完成）、KV-aware Router（`--router-policy-config`，session prefix index 需 opt-in）、Planner（按 SLA 算最小副本数，接口 `/v1/min-endpoints`）、Grove（拓扑感知 gang 调度的 K8s operator）、GMS（GPU Memory Service，已知问题是分离式服务下 GMS failover 不支持）；`DynamoGraphDeploymentRequest` 标 beta。AIBrix v0.8.0（2026-10-13）走 Operator 加网关策略：`prefix-cache`、`prefix-cache-preble`、`least-kv-cache`、`load-balance`、`slo-pack-load`、`slo-least-load`、session affinity（Redis）、Least-Request-Top-K、External Routing 标 Experimental，自动伸缩分 metric/optimizer/predictive 三类。厂商性能数（llm-d 称 median TTFT 降 85%、吞吐 2.4 倍；Dynamo 称每 GPU 吞吐 7 倍、TTFT 快 2 倍、启动快 7 倍）全部是卖方口径，引用时必须带这句限定。

42. 宽专家并行与 MoE 服务化给网关和缓存层加了什么新要求？
   【要点】llm-d 1.0 把 wide expert parallelism 列为支柱能力之一。系统的变化点：请求落到哪块卡取决于专家分布，一个 EP 组才是调度的最小单位；专家路由让激活分布随 prompt 内容变化，DP rank 之间会出现热点；KV 传输量随并行度与层数放大。网关侧的追问：打分要不要带 EP 组维度、prefill 的专家热点会不会把某个 rank 打爆、`kv-cache-utilization` 这类指标在 EP 组内还代不代表得了实例压力。他没跑过 MoE 服务，答法是按机制推演加一句「这条我停在读 release notes 与源码的层面，没实测」，别把厂商数当自己的结论。
