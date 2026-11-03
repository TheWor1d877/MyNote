## A. P1 TrainCache：训练全程数据流加速（1–58）

### A0 七问（1–7）

1. ★【数字】【证据题】✅ 探针 B 的同一组数据，median 口径是 71.7% → 7.5% → 7.7%，mean 口径是 70.4% → 27.3% → 17.6%。为什么两个口径讲出两个故事？你在 P1 报告里用哪个当主口径、哪个只能当辅助？如果面试官只记住 7.5% 而质疑“w4 已经把 stall 消掉了”，你怎么用冷态 p90 wait 992.4ms、热态 p50 wait 12.3ms 把长尾讲回来？
2. ★【失败】【证据题】✅ tc 限速连挂三次才生效：挂 docker0 主设备无效；挂 veth host 侧 egress 无效；最后挂到容器 netns 内 eth0 egress，才把 132MB 压到 11.04s（≈12MB/s）。每一层为什么无效？你分别用什么证据排除的（565MB/s 的假数、tc 计数只涨 12MB、tcpdump、overloads 计数）？如果没踩到这个坑，你的 817× 冷热比会错成什么样？
3. ★【对标】✅ 现勘用 Fluid 1.1.0-alpha.9（镜像 1.1.0-36f0467），跑分用 Fluid v1.0.8-6533239；JuiceFS CE 现勘是 1.3.0，跑分镜像是 1.3.1-oe2403sp3 + mount ce-v1.3.0。为什么会有两组版本？哪条结论必须用稳定版重跑，哪条只影响实验装置？如果你不主动交代这件事，面试官自己发现后，会怎么判断你的基线？
4. ★【复现】✅ 面试官拿到仓库，半小时内能跑到哪一步？从 kind 集群创建、镜像 `kind load`、MinIO 32 个 shard/7.9GiB、tc 100mbit+20ms、五组 up/down 到 `summary.csv`，哪些步骤是脚本化的、哪些必须手工？机器重启后 kind 证书 NotBefore 失效、kube-proxy/scheduler token 过期、kindest/node tag 只剩 dangling，这三个坑你在 README 里写了吗？
5. ★【上难度】⏳ 混沌五件套里哪一件不是设计要求的、是你自己加的？OrderSource 被杀 30 秒、Agent `kill -9`、netem 分区、位翻转、源站宕机，各自第一秒会发生什么、恢复判据是什么？哪一件你到现在还不敢说“已经恢复干净”？
6. ★【天花板】【证据题】✅ D 轨每个 epoch 2GB、tc 100mbit，理论上限是 2GB ÷ 12.5MB/s ≈ 160s；你实测 179–180s，达到理论上限的约 89%。剩下 11% 花在哪（TCP 慢启动、20ms RTT、协议开销、页缓存抖动）？这个上限能直接套到 A 轨小文件吗？不能的话，小文件的理论上限该怎么算（IOPS × 块大小、元数据路径、队列深度）？
7. ★【复盘】✅ 如果只写一篇复盘，你写《我为什么把 v3 的启动预热器改成 v4 的全过程加速器》。这篇的冲突点、证据链、你承认的判断错误分别是什么？为什么这个题目比《我实现了三阶段预取》更值得写？如果面试官读完只问一个问题，你希望他问什么？

### A1 现勘与对手边界（8–14）

8. ★【对标】【证据题】✅ 开源 Fluid 1.1.0-alpha.9 在 JuiceFSRuntime 上 webhook 注入 diff 为零；`status.cacheAffinity` 只有 `preferred` weight 100、`required` 为空，而且没有注入任何 Pod。这和 P1 笔记里“webhook 注入 nodeAffinity”的预期相反——你的缺口叙事塌了吗？为什么反而更成立？
9. 【证据题】✅ Pod 的 creationTimestamp、PodScheduled、startTime 都在同一秒 08:33:39；DataLoad 是 10:00 和 16:26 两次手动创建的独立资源。这个时序证明了什么、不能证明什么？如果有人说“Fluid 也支持定时 DataLoad，只是你没配”，你怎么回应？
10. 【失败】【证据题】✅ 两次 warmup：10:00 那次 0 files（0 B），DataLoad 仍然以 Complete 结束；16:26 那次 548 files（3.9GiB）。跑了 0 个文件也能 Complete，这对“预热完成”的语义意味着什么？你自己的 WarmupTask 用什么状态字段区分“跑完了”和“真的预热到了”？
11. 【失败】【证据题】✅ 3.9G 污染事件：源 URI 拼成 `train-data//train-data/`，`juicefs sync` 把 JuiceFS 自己的 chunk 对象也镜像进文件系统，数据从 1.97GiB 变 3.9GiB；warmup 超过 3Gi 配额，热读被 LRU 淘汰打折，系统零提示。这是装置错误还是产品缺陷？如果 P1 的预热成本记账做对了，它应该在哪一步报警？
12. ★【证据题】✅ 三 epoch 换序读：E0 0.90s（528 块、2.1197GB miss），E1 0.19s、E2 0.18s，E1/E2 页缓存全命中、UFS 零流量；节点 `/dev/shm` 只有 64MB，1GiB 声明配额被 freeratio 压到 60,397,976B（≈57.6MB）。所以“Fluid 无阶段感知”的强证据是日志级零预热动作，UFS 流量那条被页缓存削弱。你如何在报告里分级标注证据强度，避免把弱证据讲成强结论？
13. ★【对标】✅ 四方边界表：开源 Fluid、ACK ack-fluid、JuiceFS CE、JuiceFS EE。你最怕面试官挑哪一格？比如 ACK 已经有缓存亲和感知调度——如果商业版做得更接近，你的“真空地带”还剩多大？请用“块级就绪度”和“阶段感知”两个维度把真空画出来。
14. 【对标】✅ JuiceFS CE 1.3.0 只有本地缓存；分布式缓存组、IORPC、RDMA 都是 EE 5.2/5.3+ 的能力。这个边界对你的 P2P 臂和 S5 RDMA 臂意味着什么？为什么你的 P2P 叙事是“在 CE 上复现 EE 缓存组语义”，而不是“我也做了一个 JuiceFS”？

### A2 真实数字、装置与实验纪律（15–21）

15. ★【数字】【证据题】✅ 探针 A：冷读 177.308s / 11.4MB/s，热读 0.217s / 9301.9MB/s，817×，CV 0.0%/0.4%，7 轮 bootstrap CI 不重叠，热读窗口内 9000 端口 established 连接峰值 0。为什么这个装置能稳到 CV 接近 0？如果换一台机器，哪几个数会变、哪几个结论不会变？
16. ★【证据题】✅ 探针 B 的 12–14ms DataLoader 基座是装置 artifact；compute 从 3ms 校准到单线程 25ms，再到单线程 ×8 ≈200ms。为什么 3ms 时装置失去判别力（热态 wait 0.6ms 与 13ms 的差别就能把 stall 从 17% 推到 81%）？200ms 的 compute 是测量对象还是装置参数？换数据后你要重校准哪个旋钮？
17. ★【证据题】✅ 探针 C 的阈值全部取 0.30×冷态基准：cold_w0 71.7% → 阈值 21.5%，cold_w4 21.7% → 阈值 6.5%；实测 cold_w4 的 A/B 窗口是 24.1%/30.0%（C 窗口 40.0%），hot_w4 四个窗口全 0.0%。哨兵全 0 证明了什么？如果哨兵也出现 5%，你会先怀疑装置还是先怀疑缓存机制？
18. ★【数字】【证据题】✅ D 轨 8 个相邻 epoch 对里 7 对 r≥0.5，中位 r≈0.61，epoch2–9 中位 stall CV 0.8%，每 epoch 179–180s；但报告项③（epoch0 vs 后续，旧阈值 >1.2）实测 0.99，被你降级为只报告。为什么这个“不好看”的数反而是方法论的加分项？你有没有在报告里解释 0.99 的机制——恒定限速下 median stall 对冷态本来就不敏感？
19. ★【证据题】✅ 消费顺序探针 7/7 通过，12800 个样本端到端重建逐位一致。但三个 seed 语义陷阱：`.shuffle(100, seed=S)` 不带 epoch；RandomSampler 的 generator 有状态；`shardshuffle=True` 缺省随机。哪一个会让 OrderSource 在真实训练里“看起来对、其实每 epoch 都错”？你怎么在 Agent 里加断言防它？
20. ★【实验纪律】✅ 预注册冻结了五组、7 次重复、bootstrap CI、种子 20261015；但探针 B 的三个判定口径（w0 无管线冷态、w4/w8 有管线热态、mean 单调）是看了 wait 结构之后才定的。这算不算事后改口径？你怎么区分“根据机制选口径”和“p-hacking”？哪些阈值是预注册前就写死的？
21. ★【数字】【证据题】✅ M7 的 direct 组单次：startup_makespan 0.9s、total_makespan 1224s、stall_ratio 0.9885、hit_rate 0.0。为什么这个数不能直接进旗舰图？单次、冷缓存、32 个 shard/7.9GiB 与探针的 1.97GiB 不是同一口径；五组 × 7 次约 12 小时，中途某组失败你怎么处理缺失数据，才不会把“跑失败的组”从图里悄悄删掉？

### A3 失败路径与自我纠错（22–26）

22. ★【失败】【复盘】✅ v3 的启动预热器已经写了 CAS、路径索引、字节并发抓取，M1–M7 共 54 个单测全绿，但只在启动用了一次。探针 C（cold_w4 窗口 B 30.0%、C 40.0%，hot_w4 全 0）和现勘（Fluid 零阶段感知）把你推到 v4。讲清这个转向：哪些代码留了、哪些叙事扔了、54 个测试是资产还是沉没成本？
23. ★【失败】✅ checkpoint 的旧错误：权重到齐就让训练 step。缺 optimizer state（momentum/Adam 一阶二阶矩）在数学上改变了什么？为什么推理/eval 可以先到先用、训练不行？这个错误你是怎么发现的——loss 曲线、梯度统计，还是和完整 checkpoint 对拍？
24. 【失败】【证据题】✅ MinIO 镜像在 docker hub、玄源、DaoCloud、1ms.run、hub.rat.dev、1panel、quay.io、阿里云八处都没拉下来，最后降级 nginx。降级后哪些结论仍然成立（HTTP 字节流读路径）、哪些必须重测（S3 语义、元数据、分片列举）？你在报告里怎么标注这个边界？
25. 【失败】【复现】✅ kind 集群重启后证书 NotBefore 失效、kube-proxy/scheduler 的 service account token 过期、kindest/node 镜像 tag 只剩 dangling。这三件事对“一键复现”是致命伤：面试官照 README 跑，第一步就可能挂。你现在 README 里写了几条恢复命令？有没有把它们做成 make 目标或脚本？
26. ★【失败】【证据题】✅ 探针 B 第一次 compute 3ms 时，热态 stall 卡在 ~35%，压不到 15%；限单线程 25ms 仍然不行；最后单线程 ×8 ≈200ms 才让判据有区分度。你怎么证明后来的 71.7%/8.1% 是真实 I/O 饥饿，而不是你把装置调到“必然出信号”？如果面试官说“这不就是自己造了个想要的结论”，你怎么用 p90 wait 的尾部结构反驳？

### A4 核心机制（27–45）

27. ★【白板】三阶段闭环全回路：Job 声明子集 + K + consumeOrderRef → controller 向候选节点下发前 K 块清单 → Agent 预热并上报 `prefixLen` → gate 摘除/打分 → Pod 落地 → PersistentPrefetcher 按消费顺序持续预取 → 阶段切换事件到达 → 重预取。哪一步是测量点、哪一步是执行器？三阶段的优先级为什么是“启动预取 > 阶段切换重预取 > 训练中持续预取”？
28. ★【形式化】写出 `prefix_len(node) = max{k : ∀i<k, b_i ∈ R_node}`、`window_ready_ratio = |R∩W|/|W|`、`score = α·prefix_ready_ratio + (1−α)·window_ready_ratio`（α 默认 0.7），以及 required 模式的放行条件 `prefix_len ≥ K`。为什么 v3 的覆盖率口径会让“第 90 块到了、第 50 块没到”的节点拿到虚高分数？α 为什么默认 0.7，不是 0.5 或 1.0？α 交给 Job annotation 覆盖，用户凭什么会调？
29. ★【工程】SchedulingGate 的摘除如何幂等？controller 重启、重复 reconcile、gate 已被人工摘除三种情况分别怎样？不支持 scheduling gate 的集群上你的降级方案是什么？kind 里怎么 e2e 验证 gate 行为？
30. ★【性能】Score 插件为什么严禁网络调用？它只读 informer 本地缓存，缓存 TTL 内的陈旧信息会造成什么错误调度？读 `readyRatio` 会复发 v3 那种“覆盖率虚高但前缀没就绪”的错位吗？一次 RTT 的代价和一次错调的代价怎么权衡？
31. ★【降级】required 与 preferred 的语义差别？总超时（默认 5 分钟）后降级到“去 gate/去亲和正常调度”，为什么“Job 永不无限 Pending”是第一正确性原则？训练中持续预取长期跟不上消费（连续 30s cursor 追不上消费位点）进入 `PrefetchStarved` 时，你怎么讲清“不停”和“不卡”的边界？
32. ★【取舍】多副本 fanout 2–3 怎么定？预热字节随 fanout 线性涨、命中收益边际递减，你的 Pareto 最优点在哪？持续预取 fanout 1 和启动 fanout 3 的字节成本怎么对比？为什么“全程预算”必须独立上报，不能和启动预算混算？
33. ★【数字】什么条件下门控反而比直接调度慢（源站极快、缓存欠配、fanout=1）？三张旗舰图分别回答“什么时候不该用”的哪个子问题？启动盈亏曲面、全程 makespan 曲面、阶段切换 stall 曲线里，阴性区间各占多大比例？单次顺序 epoch 场景下，v4 为什么和 v3 表现相近？
34. ★【设计】OrderSource 三种实现：顺序生成器 sidecar（主路径）、DataLoader 钩子（精确校准）、Job annotation 声明（最简启动）。为什么主路径不用 DataLoader 钩子？annotation 为什么只能给静态顺序？sidecar 失联 30 秒后降级为 inotify 兜底的 LRU 预取，损失在哪？
35. ★【设计】PersistentPrefetcher 如何独立于 gRPC 预热接口消费顺序？`OnConsumeSideAdvance(consumedIndex)` 如何推进 cursor、把 `[cursor+1, cursor+lookahead]` 送进抓取管线（默认 lookahead=8）？命中率低于 30% 调到 4、高于 70% 调到 16，这个自适应和 TCP 拥塞控制的相似点在哪？重启后为什么从 dump 的 cursor 恢复，而不是重新启动预取？
36. 【设计】阶段切换事件源有哪三类（epoch 边界、shuffle 重排、curriculum stage 切换）？切换时旧阶段在途请求为什么不取消、旧阶段未消费块怎么处理、新阶段开头 K 块走什么优先级？`traincache_stage_switch_stall_seconds` 怎么定义？切换瞬间 stall 短暂回升是预期还是故障？切换后没回落说明什么？
37. 【实验设计】D 轨为什么是 v4 全程收益的专用场景（单 Job 连续 10 epoch、每 epoch 换 seed、缓存欠配 50%）？B 轨单 epoch 为什么测不出 v4 优势？D 轨的 stall 时间序列应该是什么形态？如果切换时 stall 没回升，说明预取策略没起作用还是顺序预测不准？真机校准时为什么必须单独跑一遍？
38. ★【存储】画一遍“索引层 `(path, chunkIndex)` 路径寻址 + 存储层 xxh3-128 CAS”的双层模型。CRD status 为什么加 `prefixLen` 和 `startupWindowReady`，`readyRatio` 为什么只留作粗略进度？百万块清单为什么不能进 CRD status（etcd 对象约 1.5MB 上限）？index shard 放对象存储、status 只存节点级聚合，在分片更新竞争和 controller 重启下怎么保持正确？xxh3-128 的碰撞概率在你数据规模下是多少，什么信任边界必须换 SHA-256？
39. ★【正确性】tmp + rename 的原子可见性解决什么具体故障（训练侧读到半块）？rename 崩溃、目录已存在、跨文件系统 rename 各自什么行为？缓存块为什么不逐块 fsync，这个假设依赖什么（崩溃后回源可取 + CAS 校验）？双水位 85%→75% 为什么要滞回，refcount 怎么钉住在读块？
40. ★【并发】workqueue + errgroup + 按字节加权信号量（bytes-in-flight 1GiB）各解决什么问题？为什么信号量按字节而不是按请求数？Agent 重启后如何扫描数据盘、校验 manifest、重建索引？扫描期间请求怎么处理？坏块回源路径怎么复用 singleflight？
41. ★【测量】缓存命中率的统计口径是什么？inotify/日志旁路采样有哪些误差来源？怎么和 eBPF、`/proc/diskstats` 交叉验证？inotify 观察到的实际打开顺序偏离 OrderSource 预期 20% 时，PersistentPrefetcher 怎么重新对齐 cursor，而不是继续按错误顺序预取？
42. 【控制面】manifest 与子集订阅如何用增量 List + ETag/版本号避免全量拉取？Job 声明了不存在的 shard、K 大于子集大小、consumeOrderRef 指向过期版本，这三种语义错配怎么暴露成事件而不是静默卡住？
43. ★【取舍】W-TinyLFU 在顺序 epoch 全量读取上预期与 LRU/LFU 无差异——这个阴性假设怎么预注册、怎么设计实验确认？如果准入策略最终全线无差异，你在简历和面试里怎么讲才不显得项目失败？lookahead 越大，预取块集合越大，准入策略才越重要；小 lookahead 时准入几乎无关，这个协同怎么报告？
44. 【支撑臂】checkpoint 上传的 multipart + manifest 原子发布点在哪？Range GET 并发恢复怎么校验分片完整性？加载与初始化具体重叠了哪两段时间？为什么权重和 optimizer state 的可见性必须一起发布？
45. 【P2P】P2P 臂为什么定位为“在 JuiceFS CE 上复现 EE 分布式缓存组语义”？一致性哈希环、peer 回源、失败穿透、origin 保底 20% 各解决什么？持续预取产生的抓取请求如何走 peer 询问，不需要新接口？D 轨 10 epoch 跑下来，持续预取请求 70% 以上走 peer 不回源，你怎么验证这个数？

### A5 故障注入与正确性（46–50）

46. 【上难度】⏳ 混沌五件套：Agent `kill -9`、netem 分区、位翻转、源站宕机、杀 OrderSource 30 秒。每一件的恢复判据是什么？RTO 怎么测、从哪个时间点算到哪个时间点？位翻转怎么被 CAS 变成可检测错误，而不是静默脏数据？
47. 【失败】⏳ Agent 在预取中途被 `kill -9`：in-flight map 里已发出的请求怎么办？落盘的半块怎么被识别（大小、crc、临时文件后缀）？重启后是重扫全盘还是信任 manifest？两种路径分别会漏掉什么？
48. 【故障】⏳ netem 分区和源站宕机分别先打崩哪个组件：Agent 的重试队列、controller 的 reconcile、还是 gate 的超时？多数派/少数派/不对称分区在 P1 里怎么映射到“Agent 失联、controller 失联、源站单向不可达”？5 分钟 gate 超时在分区期间会不会误放行？
49. ★【降级】⏳ 杀 OrderSource 30 秒后降级为 LRU 通用预取：降级判据是“30 秒没有 cursor 推进”，为什么不是 10 秒或 60 秒？降级后命中率、预取字节、训练 stall 各会怎么变？恢复后怎么从 LRU 模式切回顺序模式，cursor 怎么重新对齐？
50. ★【正确性】这个系统最大的正确性风险（不是性能风险）是什么？是半块可见、checkpoint 权重与 optimizer state 错位、CAS 碰撞，还是 gate 误摘除？你用什么机制兜底？如果只能保留一个正确性断言，你保留哪个？

### A6 实验臂：S2 / S5 / S6（51–58）

> 2026-11-02 口径：S2（51–52）不受影响。S5（53–55）不写代码、不跑传输臂，53 题那个 ⏳ 撤下——「P2P 块传输原型验了哪些功能」这一问不再存在，答法停在接口层与「性能不可采信」这条自设边界；S6（56–58）整项不执行，三题改成白板题，回答时逐条分清「读到」与「踩到」。

51. 【S2】【上难度】O_DIRECT 的三重对齐（缓冲区地址、长度、偏移）为什么必须满足？overlayfs 上 O_DIRECT 报 EINVAL 的根因是什么？你最后落在什么文件系统上验证，用 `posix_memalign` 对齐到 4K 的哪一段代码？
52. ★【S2】【天花板】4MiB 大块经 HTTP 抓取为什么是网络 bound，io_uring 省 syscall 对吞吐的预期是中性的？你的实验是否验证了这个阴性预期？小文件高 IOPS 臂的数字才是它的战场——两边都要报。v4 的持续预取让小文件请求更密集，你怎么设计“启动路径 vs 持续预取路径”的对照，证明收益到底来自 io_uring 还是来自请求分布变化？
53. ★【S5】【证据题】Soft-RoCE 只能验证功能、性能不可信——为什么？CQ 事件通知在 Soft-RoCE 上经常丢事件、请求挂起超时，所以轮询是被迫选择。传输臂 2026-11-02 起不写代码，这一题只答机制与接口层：rxe 里谁是那层「网卡」、`ibv_poll_cq` 与事件通知在软件栈上差别在哪一步、为什么任何 Soft-RoCE 性能数字都不能进报告。「原型验了哪些功能」这一问不再存在，被问到就答「没投那 4 周，验收口径是我自己写死的『功能可验证、性能不可采信』」。
54. ★【S5】【取舍】为什么是“独立 C++ 传输组件 + 薄 C ABI”，而不是 cgo 把 libibverbs 直接链进 Go 热进程？三个原因：cgo 切换开销（goroutine→C 线程约 2–5μs）、CQ 轮询放 C 进程更干净、Go GC 可能移动内存让 `ibv_reg_mr` 失效。这三个原因分别怎么用最小实验证明，而不是背结论？（2026-11-02 起这一题只答「怎么设计才能证明」，不声称做过。）
55. ★【S5】【天花板】RDMA Write 和 Send/Recv 在 P2P 拉取式场景怎么选？Write 是单边操作，B 预注册槽位、发布 rkey，A 直接写；Send/Recv 是双边，B 要提前 post receive buffer。`ibv_reg_mr` 为什么要锁页、为什么不能按需注册 4KB 小块？RoCE 的 PFC/ECN/MTU=4096 三前提怎么配合？4 周只做到“跑通”时，为什么降级成《为什么我没在主路径上 RDMA》而不是硬撑？
56. ★【S6】【工程】S6 整项 2026-11-02 起不执行，这题改成白板题：链路的每一跳谁执行、CRD 从哪来到哪去——device plugin TopologyInfo → kubelet Topology Manager → NFD topology-updater → NodeResourceTopology CRD → 调度插件 Filter/Score → bind。`KubeSchedulerConfiguration` v1 那几处易错（apiVersion、多 profile、schedulerName、pluginConfig 位置）只有读文档的份，被问到就说是读到、不是踩到。自定义 scheduler 的 RBAC 最少要哪些权限？
57. ★【S6】【取舍】为什么砍掉阶段 B（自研最小插件，2026-10-15），又在 11 月 2 日把剩下的「配置与链路」那一档一起收掉？P1 步骤 7 的插件比它完整在哪（数据亲和 + SchedulingGate + 门控 + 降级）？主动砍掉重复劳动，面试时怎么说才不像“没做完”？单 NUMA 游戏本上用 annotation/fake NRT 做实验为什么只能算装置、不能算结论？
58. ★【S6】【对标】数据亲和 × 拓扑亲和的权重-决策曲线该怎么设计——这条按设计讲，不跑：`score = w·数据亲和 + (1−w)·拓扑亲和` 里两个分量各自怎么归一化到 [0,1]，不归一化 w 的语义怎么塌？多插件分数归一化为什么没有银弹？S6 收掉之后留在 P1 里的资产只有步骤 7 的打分插件，`experimental/` 不再挂拓扑实验；DRA 相对 device plugin 改变了什么，v1.35 GA、v1.36 接入 Workload/PodGroup 这两条事实你怎么讲？
