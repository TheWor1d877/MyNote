# 推理引擎与 vLLM（分级版）

## T1 · 必须脱口而出

1. ★ 解释大模型推理的两个阶段：prefill 与 decode，它们的计算特征和瓶颈有何不同？
2. ★ 为什么 prefill 阶段通常 compute-bound，而 decode 阶段通常 memory-bandwidth-bound？
3. ★ 解释 TTFT 与 TPOT 的定义，它们分别受哪些系统层因素影响？从网关、调度、KV Cache、批处理角度如何优化首 token 延迟？
4. ★ 什么是 continuous batching？它相比静态 batching 的优势是什么？
5. ★ 什么是 PagedAttention？它如何借鉴虚拟内存思想管理 KV Cache？
6. ★ 解释 KV Cache 的 block 管理与 paged 布局，block size 的选择如何影响显存碎片、显存带宽利用与吞吐？
7. ★ 解释 Prefix Caching 的命中条件与 hash 机制（block hash 如何识别可复用前缀），LRU 淘汰如何与之协作？它在多轮对话和 RAG 场景里省下什么？
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

32. 一个新请求进来，引擎怎么找到能复用的那段 KV？把切块、block hash 生成、最长公共前缀查找、后一块 hash 要不要依赖前一块四问一次答完，再说清 vLLM 的 hash 表与 SGLang 的 radix tree 在结构上差在哪。
   【要点】按 vLLM v0.32.0（2026-10-18）逐问：切块用 `--block-size`，默认 `DEFAULT_BLOCK_SIZE = 16`（`vllm/config/cache.py:72`），只缓存满块，末尾不满一块的 token 必须重算，所以查找时传的长度是 `num_tokens - 1`。hash 由 `hash_block_tokens(hash_function, parent_block_hash, curr_block_token_ids, extra_keys)` 算（`vllm/v1/core/kv_cache_utils.py:695`），真正喂进哈希函数的是 `(parent_block_hash, token 元组, extra_keys)` 三元组（同文件 724 行），链式依赖是明确设计：首块的 parent 取 `NONE_HASH`（`BlockHash(hash_fn(_NONE_HASH_SEED))`，172 行），密码学算法下种子固定为 `"vllm-none-hash"` 以便跨进程与跨节点复用，非密码学算法改用 `os.urandom` 随机种子。所以第五问的答案是要依赖，否则两棵不同前缀下的相同 token 块会撞同一个 key，命中就会把别人的 KV 接过来。`extra_keys` 装的是 LoRA（名字与路径）、多模态（identifier 与 offset）、`cache_salt`（只注入首块）、prompt_embeds（同文件 656 行附近），含义是 token 序列相同但 LoRA 或图不同就不算同一段前缀。算法选项 `prefix_caching_hash_algo` 有 `sha256`（默认）、`sha256_cbor`、`xxhash`、`xxhash_cbor`（`vllm/config/cache.py:143`），官方对非密码学算法的警告是可能泄露信息或行为未定义；v0.32.0 仓库里搜不到 `PrefixCollisionChecker`，这条按未落实处理。查找链是 Scheduler 到 `KVCacheManager.get_computed_blocks(request)`（`kv_cache_manager.py:264`），再到 `coordinator.find_longest_cache_hit(block_hashes, max_cache_hit_length)`（`kv_cache_coordinator.py:460/574`），最后到 `BlockPool.get_cached_block(hash, group_ids)`（`block_pool.py:198`），命中表是 `BlockPool.cached_block_hash_to_block`。返回值是 `(blocks, num_computed_tokens, shared_prefix_boundary)`，第二项直接赋给 `request.num_computed_tokens`，也就是这个请求跳过 prefill 的长度。混合架构下多 KV cache group 的取法是 `HybridKVCacheCoordinator` 源码注释写死的「A hit is the shortest across all groups」（同文件 70 行），取所有组的最小命中。淘汰侧 `FreeKVCacheBlockQueue` 是双向链表（`kv_cache_utils.py:246`），分配时 pop 队头即最旧，命中过的块由 `BlockPool.touch()` 增 `ref_cnt` 并从 free 链摘出。
   SGLang v0.5.22（2026-10-19）的对照做法：key 就是 token 序列本身（`RadixKey`，字段 `token_ids`、`extra_key`、`cache_salt`、`is_bigram`、`limit`，`python/sglang/srt/mem_cache/radix_cache.py:59`），一个节点可以存多 token，段中拆分靠 `_split_node`；匹配入口 `match_prefix(params) -> MatchResult`（同文件 364 行），返回 `device_prefix_len`、`last_device_node`、`last_host_node`，后两个字段正好是分层缓存的落点。使用中靠 `TreeNode.lock_ref` 与 `inc_lock_ref`/`dec_lock_ref` 保护，淘汰只从 `evictable_leaves` 堆里取，被锁住的部分计入 `protected_size`。`--radix-eviction-policy` 支持 `lru|lfu|slru|priority|tlru`，默认 `lru`（T-LRU 引 arXiv:2510.15152）；前缀缓存默认开，`--disable-radix-cache` 默认 False。
   结构差别的答题落点：hash 表把前缀编码成链式哈希，查询是逐块 O(块数) 的哈希查表，天然带 `cache_salt` 与多模态命名空间，代价是命中长度只能落在块边界；radix tree 把前缀存成树的边标签，最长前匹配是从根往下走，粒度可以到 token，节点上直接挂引用计数与所在层级，代价是并发下的树锁与节点分裂管理。vLLM 设计文档只自述「选了 hash-based 方案」，两个仓库都没有对二者的官方对比段落，所以这段差别是按源码结构读的，答题时按源码实现讲，别引成官方结论。出处：本章 7 题（命中条件与 LRU 的定义层）、本章 28 题（网关侧近似索引与 KV 事件订阅）、08 章 17 题（attention 层的 IO 账，说明省下的 prefill 值多少钱）。

33. 一批商品数据的离线 embedding 推理，和在线对话服务，考核的东西差在哪？这个分叉会改哪几个开关？
   【要点】指标定义按 vLLM v0.32.0 官方 `docs/benchmarking/cli.md`：TTFT 是发出请求到收到第一个流式输出，ITL 是相邻流式输出之间的间隔，TPOT 是 `(端到端延迟 − TTFT) / (输出 token 数 − 1)`。在线 SLO 挂这三个加错误率。离线批任务里单请求延迟进不了目标函数，看的是整体吞吐、GPU 利用率、容错和整批完成时间；llm-d-batch-gateway v1.0.0（2026-10-18）README 的写法是给批任务按队列深度与利用率动态限流派发，减少对交互式负载的干扰，容错只写到「batch processor 崩溃后自动恢复」这一层。
   往下推的四件事（这部分是按机制推的，不是官方口径，讲的时候要说清）：
   1. batching 策略变了。continuous batching 的价值是把不同到达时刻的请求拼起来、不让短请求等长请求；离线提交时刻本来就集中、批次可控，静态大批次也能吃满，chunked prefill 那套抢跑权衡随之变弱。vLLM `docs/configuration/optimization.md` 的建议只覆盖生成式（V1 默认开 chunked prefill，追求吞吐时 `max_num_batched_tokens > 8192`，小值利于 ITL、大值利于 TTFT），面向 embedding 批量的官方建议没找到，标 UNVERIFIED。
   2. 结构差异是硬证据。embedding 这类 pooling 模型 prefill 完就结束，没有逐 token decode。v0.32.0 的离线入口是 `vllm.LLM` 类（`generate`、`chat`、`classify`、`embed`、`score`、`encode`），模型侧用 `--convert embed` 与 `--pooler-config`，旧教程里常见的 `--task` 已经不在参数列表里。`vllm/config/vllm.py:1585-1596` 的原话是「当前异步调度的实现会拖累 pooling 模型的性能，所以默认关掉」，`runner_type == "pooling"` 时 `async_scheduling` 自动置 False。引擎在配置文件里就把两类负载分开处理，这种默认值差异比概念对比更值得报。
   3. 容错手段不同。在线不接受抢占与重算，判据在 03 章 39 题；离线可以整段重跑，于是闲时卡与 spot 给离线，配批次级重放与断点。公开资料里没有这方面的论述，这条属推断。
   4. 成本口径换分母。离线算的是「整批完成时间 × 卡数」，等价于本章 27 题那个每百万 token 成本模型的批量版；13 章 14 题的推理闲时变现，就是给这类任务开的窗口。
   反问阶段直接用：对方给出业务口径之后，把「离线批次多大、用什么指标验收、重跑代价谁承担」问回去，这是 16 章第一节的用法。

34. 强化学习的 rollout 跑在推理引擎上，它的负载特征跟在线对话差在哪？训练侧的权重更新怎么进引擎？
   【要点】先报位置：RLHF/RLVR 的训练循环里，采样这一段（给定 prompt 生成回答、算 reward、再更新）由推理引擎执行，PPO（arXiv:1707.06347）与 GRPO（DeepSeekMath，arXiv:2402.03300）是两种目标函数，采样阶段的系统形态与它们无关，跟引擎有关的是下面四条。
   1. 同一 prompt 采多条。GRPO 按组采样，组大小记为 G，组内 G 条回答共享同一个 prompt 前缀，于是前缀命中是结构性的，不是流量凑出来的。本章 7 题与 32 题那套命中判据在这里直接生效：prefill 只做一次，其余 G−1 条走同一批 block，靠引用计数保住不被淘汰（`BlockPool.touch()` 增 `ref_cnt`，见 32 题要点）。网关侧的路由打分如果按「每条请求」算 token 数，会把这一段重复计 G 遍。
   2. 权重频繁同步。训练步产出的新参数要搬进引擎，卡就那么多，训练侧要显存、引擎侧也要显存。vLLM 的 `--enable-sleep-mode`（本章 44 题已列）正是这个场景的开关：level 1 保留权重、把 KV 换出，唤醒后重建 KV 池；权重更新一次，缓存全废，本章 20 题那套命中率统计在训练任务里要按步重置。
   3. 输出长度分布随训练进度漂。RL 里策略变好通常表现为回答变长，同一份 SLO 下并发容量跟着变小。15 章 8 题的 Little 定律估算里，那个「平均 200 token」是静态假设，这类负载要按训练阶段的长度曲线重算。
   4. 批次到达时刻集中。一个 step 一次性灌进一批，到达呈脉冲分布；本章 23 题的排队策略与本章 33 题的离线批任务各有一半性质——批任务的吞吐导向加上在线服务的步内同步等待。
   开源框架名单：把训练与推理放在同一组卡上调度、由框架负责权重与显存交接的做法，公开可查的是 verl 与 OpenRLHF 这一族，它们的 rollout 走 vLLM 或 SGLang；具体的进程划分、通信方式与 sleep 时机没到源码逐行核实，答题时按机制推演并明说这一层。JD 标题写「推理\RL 方向」时（蔚来 2027 届那条），问的也是这四条负载特征，不要求写过训练循环。

35. vLLM、SGLang、TensorRT-LLM、LMDeploy 差在哪一层？什么条件下值得换引擎？
   【要点】四条路线优化的是不同的东西，别把它们摆成同一根轴上的四个分数。
   vLLM：调度优先的通用引擎，PagedAttention 加 V1 全异步调度（本章 38、44 题），模型覆盖与工具链最宽，默认起点。版本按本地核实过的 v0.32.0（2026-10-18）报。
   SGLang：同一层的另一个前缀缓存结构（radix tree，本章 32 题），外加约束解码这一族能力；多轮对话、Agent 工具链、共享长前缀的 RAG 这类前缀重叠率高的负载，收益来源是命中率，kernel 侧没有变化。版本按 v0.5.22（2026-10-19）报。
   TensorRT-LLM：走编译路线，模型编译成图再跑，kernel 融合与 FP8 在 NVIDIA 卡上提高上限；代价是只支持 NVIDIA、构建链复杂、模型或形状一变就要重新构建。值得换的条件是模型固定、流量长期稳定、编译与调优的人力摊得薄。融合的收益机制是减少访存往返，与 08 章 23 题同一条账。
   LMDeploy：TurboMind 是自研 C++ 运行时，差异点在量化（AWQ、GPTQ、bitsandbytes 这类权重格式）与国产硬件适配；社区与英文文档比前三个窄。当前版本号两份公开来源给的日期与号段不一致，标 UNVERIFIED，答题只报机制层。
   TGI 已进入维护模式这个说法只有单一博客来源，标 UNVERIFIED；引用前要自己去看仓库的 release 页。
   判据两句：先量前缀重叠率和模型更新频率，再决定要不要离开 vLLM；跨引擎的性能差必须在自己的负载上实测过才写进结论，第三方天梯榜的 tok/s 是别人家的 batch size。网关侧的连带影响要主动补一句：换引擎等于换打分字段（本章 28 题的 approximate 与 precise、12 题的余量信号在不同引擎上口径不同），他的 P2 靠一层 adapter 把指标语义对齐，这句话是他这题区别于「背选型文章」的地方。

## T3 · 扩展与速查

36. SSE 流式转发有哪些工程坑？flush 时机、客户端取消传播、禁用压缩、超时重置分别怎么处理？
37. 多租户推理服务如何设计 token 预算与限流？请覆盖账户级、请求级、token 级三层控制与超卖策略。
38. ★ 解释 vLLM V1 引擎架构相比早期版本的关键变化。
39. ★ 解释 KV Cache 在 prefill 节点与 decode 节点之间的传输问题。
40. 把 chunked prefill 的块大小从 512 调到 8192，TTFT 与 TBT 各会滑向哪个极端？"块越大越接近完整 prefill"在什么负载下成立、什么负载下失效？
41. 什么是 sampling 参数对 decode 性能的影响？
   【要点】按四个方向算，不要背参数名。
   分支数 `n`：同一个 prompt 采 G 条时 prefill 只做一次，G 条分支共用同一批 KV block，decode 每步那趟 KV 读同时服务 G 路，等于用一次带宽换 G 条输出；显存侧多出的是每分支的采样状态与各自的输出段。本章 34 题的 GRPO 就是这个形态。
   `temperature`、`top_p`、`top_k`：逐 token 对词表 logits 做变换与截断，词表 3.2 万、batch 256 时这一步是千万级的 elementwise 工作量，还夹在 kernel 串里，属于 decode 步长的固定尾巴。repetition penalty 与自定义 logit processor 要先在 CPU 上遍历，比在 GPU 上更贵。
   `beam search`：维护多条假设让每步的活跃序列数翻倍，与 continuous batching 的语义冲突（请求提前结束、KV 立刻该释放）。vLLM 对 beam search 的支持状态在近期版本里反复，没核实到当前口径，标 UNVERIFIED，答题时停在「它与按 token 调度的引擎配合最差」这一句。
   约束解码（`structured_outputs`、JSON schema、grammar）：状态机逐 token 给出允许的 token 集合，掩码在 GPU 上打，每请求的语法状态要跨步保留；语法编译做一次可以缓存复用。约束解码对 spec decoding 的接受率有额外影响，机制清楚（draft 也要在合法集合里出），量化到几个百分点没有自己的数，报「这条我只推到机制」。vLLM 侧的 xgrammar 与 guidance 后端名称来自第三方对官方文档的转述，标 UNVERIFIED。
   收尾两句：温度与 top-p 不改每 token 的算术量，改的是输出长度分布——均值不动、尾部变长，那么 15 章 8 题按平均长度估的卡数会低估 P99 时的占用；采样设置改变容量口径，压测必须带采样参数一起交，否则复现不了。

42. 什么是推理服务的 GPU 时间片与优先级抢占？
43. 什么是 NVLink 域内通信与跨域通信的差异？

44. vLLM 当前哪些能力是默认开的、哪些要显式打开？报版本号再答。
   【要点】以 v0.32.0（2026-10-18）为准：`enable_prefix_caching` 默认 True（定义在 `vllm/config/cache.py`），哈希算法字段 `prefix_caching_hash_algo` 可设 sha256；`--async-scheduling`（`SchedulerConfig.async_scheduling`）不写时自动启用，遇到 pooling 模型、不被支持的 spec decode、V1 runner 配 PP 等情况会被关掉；`--enable-sleep-mode` 把权重换出，level 1 保留权重换出 KV、level 2 两者都丢，`--sleep-preserve-parameter-names` 控制保留项；0.31/0.32 这条线去掉了 cascade attention、停止支持 Python 3.10。同一条 0.31.0（2026-10-05）、0.30.0（2026-09-22）。答默认值题的规矩：先报版本，再报字段名，说不准就承认「这条我按 X 版本记的，会后核对」。

45. llm-d、NVIDIA Dynamo、AIBrix 三条分布式推理路线各自的重点差异在哪？
   【要点】llm-d 1.0.0（2026-10-20），CNCF Sandbox 项目（2026-03-12 入选），由 IBM Research、Red Hat、Google Cloud、NVIDIA、CoreWeave 在 2025-05 发起；模块拆在 `llm-d-router`、`llm-d-kv-cache`、`llm-d-latency-predictor`、`llm-d-async`、`llm-d-batch-gateway`、`llm-d-autoscaling`，1.0 的主题是多租户 flow control（概率准入、program-aware fairness、turn-priority）、路由高可用与零停机更新、`DisaggregatedSet`、session-aware routing 与 `previous_response_id`，基线镜像 pin vLLM v0.31.0。Dynamo v1.6.1（2026-10-27）的组件名：KV Block Manager（pip 包 `kvbm`，对 SGLang 的支持在 README 矩阵里仍标施工中，vLLM 与 TRT-LLM 已完成）、KV-aware Router（`--router-policy-config`，session prefix index 需 opt-in）、Planner（按 SLA 算最小副本数，接口 `/v1/min-endpoints`）、Grove（拓扑感知 gang 调度的 K8s operator）、GMS（GPU Memory Service，已知问题是分离式服务下 GMS failover 不支持）；`DynamoGraphDeploymentRequest` 标 beta。AIBrix v0.8.0（2026-10-13）走 Operator 加网关策略：`prefix-cache`、`prefix-cache-preble`、`least-kv-cache`、`load-balance`、`slo-pack-load`、`slo-least-load`、session affinity（Redis）、Least-Request-Top-K、External Routing 标 Experimental，自动伸缩分 metric/optimizer/predictive 三类。厂商性能数（llm-d 称 median TTFT 降 85%、吞吐 2.4 倍；Dynamo 称每 GPU 吞吐 7 倍、TTFT 快 2 倍、启动快 7 倍）全部是卖方口径，引用时必须带这句限定。

46. 宽专家并行与 MoE 服务化给网关和缓存层加了什么新要求？
   【要点】llm-d 1.0 把 wide expert parallelism 列为支柱能力之一。系统的变化点：请求落到哪块卡取决于专家分布，一个 EP 组才是调度的最小单位；专家路由让激活分布随 prompt 内容变化，DP rank 之间会出现热点；KV 传输量随并行度与层数放大。网关侧的追问：打分要不要带 EP 组维度、prefill 的专家热点会不会把某个 rank 打爆、`kv-cache-utilization` 这类指标在 EP 组内还代不代表得了实例压力。他没跑过 MoE 服务，答法是按机制推演加一句「这条我停在读 release notes 与源码的层面，没实测」，别把厂商数当自己的结论。

47. ★ 显存不足时，被 in-flight 请求引用、却排在空闲链队头的 block 怎么处理？全池 block 都被引用时，分配函数返回什么、调度器怎么反应？驱逐与抢占在这一点上是不是同一件事？V1 默认用重算代替换出，这次重算对 TTFT 与 KV 计量的影响落在哪一段？
