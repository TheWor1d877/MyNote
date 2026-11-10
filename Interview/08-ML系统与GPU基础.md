# ML 系统与 GPU 基础（分级版）
## T1 · 必须脱口而出

1. ★ 解释数据并行、张量并行、流水线并行的切分维度与通信模式。
   【要点】答题顺序：先一句话说清各自切哪一维，再逐个给「切什么｜每步通信｜量随什么涨｜落在哪种互联」，最后补一句三者是正交轴、可以叠乘。切分维度：DP 切 batch，每卡模型完整；TP 切单层权重矩阵；PP 切层，一个 stage 负责一段连续的层。通信模式：DP 是 step 末的梯度 all-reduce（DDP）或 reduce-scatter 加参数 all-gather（FSDP，见 11 题），量随参数量线性、与卡数无关（见 13 题）；TP 是每个 microbatch 的前反向各 2 次 all-reduce（Megatron-LM, arXiv:1909.08053），量随层数与 TP 度线性涨，所以压在机内 NVLink 域；PP 只在 stage 边界传一次激活与梯度，点对点、量最小，代价是气泡 `(p−1)/m`（见 14 题），所以跨机段先想到它。收尾要给一个真实配置形态：4D 并行写成 `DP×TP×PP×EP`，例如 1024 卡 = TP 8 × PP 8 × DP 16。会被追问的两处：TP 内部为什么是「列并行接行并行」这一对（中间那段不必 all-reduce，只在行并行输出后同步一次）；单卡为什么放不下 7B（16 字节/参数的静态账见 8 题，7B 光优化器与权重就要 112 GB）。出处：11、13、14、16 题。

2. ★ 解释 AllReduce、AllGather、ReduceScatter 的数学语义与通信量。
   【要点】先给集合语义，用 4 个 rank 各持 4 个数的小例子走完三个算子：AllReduce 之后每个 rank 拿到 4 个数，位置 i 是四个 rank 在第 i 位上的归约结果；ReduceScatter 之后每个 rank 只拿到 1 个数（自己那一段的归约）；AllGather 没有归约，每个 rank 拿到 16 个数的拼接。再给关系式：AllReduce = ReduceScatter + AllGather，所以通信量是两段相加 `2·(N−1)/N·S`（13 题），单段各为 `(N−1)/N·S`。口径要先声明：`S` 是最终每卡持有的字节数，报的是每卡流量，全集群对穿流量还要乘 N/2 一类因子——面试官追的就是这个分母。再补两个：All-to-All 是转置语义，MoE 路由用它（16、28 题）；Broadcast 只有一份源，与 AllReduce 的归约方向不同。加分项一句：NCCL 会按消息尺寸与拓扑在 ring 与 tree 之间选算法，tree 用 `log N` 层换延迟、带宽利用次优，小消息先受延迟限制（13 题那句），交换机内归约（SHARP/NVLS）能把一部分归约搬进 NVSwitch（这串按当日 NCCL 文档核，记不准报量级）。出处：13 题负责账，这题负责定义，别在语义题里报式子。
3. ★ 什么是 KV Cache？它缓存的是什么，为什么 decode 阶段必须依赖它。
4. ★ 解释 KV Cache 的数学含义，为什么它只缓存 K 和 V。
5. ★ 什么是 GQA 与 MQA？它们如何减少 KV Cache 显存。

## T2 · 应当会答

6. ★ 什么是 ZeRO 的三个阶段？它分别切分优化器状态、梯度和参数。
   【要点】先立分母：bf16 混合精度下每个参数的静态占用是 16 字节 = 参数 2 + 梯度 2 + fp32 主权重 4 + Adam 一阶矩 4 + 二阶矩 4（推导与激活项在 8 题）。再用一张四列表把三个阶段摊开：Stage 1 切优化器状态，每卡 `4 + 12/N`；Stage 2 再切梯度，每卡 `2 + 14/N`；Stage 3 连参数也切，每卡 `16/N`（arXiv:1910.02054）。第三列写通信形态：Stage 1 与 DDP 相同（梯度 all-reduce），Stage 2 把梯度同步改成按桶 reduce-scatter，Stage 3 每层前反向各要 all-gather 参数、用完即释放，通信量升到 DDP 的约 1.5 倍（与 11 题 FSDP 那句是同一笔账）。第四列给一个数：N=8 时 Stage 1 单独把每参数静态占用从 16 降到 5.5 字节，省掉 65.6%，算式 `12/16·(1−1/N)`——这一句要在面试官问「为什么先做 Stage 1」时报出来，理由是 16 字节里 12 字节在优化器状态上。两句收尾：FSDP 是 Stage 3 的 PyTorch 原生实现，把 ZeRO 与 FSDP 说成两套方法是常见失分点；ZeRO-Offload 与 ZeRO-Infinity 挪到哪一级存储、代价进哪段时间归 12 题答，别在这题里混着讲。出处：8、11、12 题。
7. ★ 解释 draft model 与 target model 的接受率如何影响加速比。
8. 量级估算：参数量翻倍，训练显存需求为什么远不止翻倍？请把参数、梯度、优化器状态与激活值的账拆开算一遍。
   【要点】分三本账报，顺序固定。第一本静态：每参数 16 字节（2+2+4+4+4，构成见 6 题），随参数量线性，这一项本身只按 2 倍涨。第二本动态：激活 ≈ `L·b·s·h` 乘一个与 FFN 宽度、注意力实现、是否开重计算有关的系数，量级 `34·L·b·s·h` 字节这个式子在 Megatron 相关材料里读到，出处没逐字核到，报量级并声明（UNVERIFIED）；参数量翻倍时若 micro-batch 与序列长度不变，这项按层数与隐藏维的乘积涨，比参数项快。第三本不变量：CUDA 上下文与 cudnn 缓冲约 0.5 GB 量级、通信 buffer（`bucket_cap_mb` 默认 25 MB × 桶数）、显存碎片，这三项与参数量无关，所以「显存/参数」的比值随规模变化——把「远不止翻倍」落在这个比值上讲，比讲绝对倍数稳。收尾给可实证的口径：单卡上 `torch.cuda.max_memory_allocated()` 前后对照，加 `sum(p.numel() for g in opt.param_groups for p in g)` 乘字节数直接算优化器状态，这两条在 32G 笔记本 CPU 侧也能跑，不需要多卡。陷阱两处：把推理侧的 KV Cache 字节数当训练激活报（那是 20 题的式子）；报倍数时不说 batch 与序列是否固定。出处：6、20 题、15 章带宽与容量账。
9. ★ 解释 CUDA 的线程层次：thread、warp、block、grid 的关系。
   【要点】按「谁共享什么、谁被谁调度、谁能同步」三个问题串，背定义会在追问下散架。thread 是最小编程单位，寄存器私有。warp 是 32 个 thread 的硬件调度组，SM 一次发射一整条 warp（SIMT），组内走同一指令；出现分支时两条路径串行执行，代价约等于分支路径数——这一句是「warp divergence」的落点。block 是软件单位，共享同一块 shared memory，用 `__syncthreads()` 内部对齐，大小取 32 的倍数、上限 1024 线程。grid 是一次 kernel launch 的全部 block，block 之间没有同步原语（cooperative launch 是特例，别当常规），只能靠多次 launch 或原子操作分阶段。硬件对应要给出数：一个 SM 容纳多个 block，Ampere/Hopper 每 SM 最多 2048 线程即 64 warp、寄存器 65536 个 32 位 bank，shared 与 L1 合池在 A100 上是 192 KB、单 block 可申请到 164 KB（这串数字按 CUDA 架构手册核，记不准就报量级）。occupancy 的三个限制因子逐个点名：每线程寄存器数、每 block shared memory、每 SM 的 block 与 warp 上限，谁先到顶谁定。最后接回他的位置：decode 阶段每步几百次小 kernel，launch 开销与 occupancy 决定 GPU 空转比例，这条接到 26 题 CUDA Graph 与 23 题的编译层。出处：10、24、26 题。
10. ★ 解释 CUDA 的 shared memory、bank conflict，以及如何优化访问模式。
   【要点】三段：为什么搬、什么时候不值得搬、冲突长什么样与怎么修。为什么搬：shared memory 是 block 内显式管理的片上存储，带宽比 HBM 高一个数量级（A100 HBM2e 峰值约 1.55 TB/s，80 GB 版约 2 TB/s；片上那侧的聚合带宽按架构手册核，答题报「一个数量级」这个量级就行），重复读取的数据搬进去省的是 HBM 往返。什么时候不值得：数据只被读一次、或分块后同步等待与占用超过收益——判据是复用次数乘每次省下的字节，能不能盖过同步成本与 occupancy 损失（9 题那个每 block shared 上限就是分块天花板）。bank conflict 讲现象与机制：shared memory 分 32 个 bank、每 bank 每周期服务 4 字节；同一 warp 的多个线程命中同一 bank 的不同地址时串行化，冲突度等于落在同一 bank 上的线程数；同一 warp 读同一地址免费（硬件广播）。典型例子要能当场画：`__shared__ float s[32][32]` 按列访问时，线程 i 读 `s[i][j]`，相邻线程的地址差 32 个 float，正好同 bank，退化成 32 路串行；把数组改成 `s[32][33]` 补一列，或交换循环顺序改成按行读，或让线程与列对应。修法清单四件：padding、swizzle（异或重排索引）、访问维度与线程维度对齐、一次读 128 位（`float4`）。与 24 题分清边界：那题管 global memory 上 warp 的访存能否合并成少数事务，这题管片上 bank 是否错开，两条路径独立发生。接回系统侧：FlashAttention 的 tiling 就是把 Q/K/V 的分块先搬进 shared 再喂给 tensor core（17 题）。答题边界按 33 题三段给，写过 toy kernel 就说写过哪三个、在哪个 shape 上撞到过冲突；没写过就停在「能指出冲突现象与两种修法」，别说调过 occupancy。出处：9、17、24、33 题，S8 立项第六节那条接法。

11. DDP 与 FSDP 各自把什么切了、每一步跑哪些集合通信、通信量差多少？面试官问「你这个规模为什么用 FSDP 不用 DDP」时，答案要落在这三行上。
   【要点】DDP：参数、梯度、优化器状态每卡全量，梯度按桶做 all-reduce（`bucket_cap_mb` 默认 25 MB）。FSDP2（`fully_shard`，基于 DTensor 做按参数 dim-0 切分；`DeviceMesh` 1D 即纯 FSDP，2D 是 HSDP）：前向 all-gather、反向 all-gather、梯度 reduce-scatter，通信量约为 DDP 的 1.5 倍，换来三项状态的显存分摊。`reshard_after_forward` 默认 True，改成 False 或整数（ZeRO++ 风格部分重切）就是在显存与带宽之间挪位置；混合精度走 `MixedPrecisionPolicy(param_dtype, reduce_dtype, output_dtype)`，CPU offload 走 `OffloadPolicy`。选型的句子要具体：单卡放得下模型与优化器状态时用 DDP，放不下多少才付得起那 1.5 倍通信。出处：PyTorch `distributed.fsdp.fully_shard` 文档、DDP notes。

12. ZeRO 三个阶段各切什么？ZeRO-Offload 与 ZeRO-Infinity 分别把东西挪到哪一级存储，挪过去的代价进哪段时间？
   【要点】Stage 1 切优化器状态，Stage 2 加梯度，Stage 3 加参数（arXiv:1910.02054）。Offload 把优化器状态与梯度放 CPU（`offload_optimizer.device="cpu"`），Infinity 再把参数放 CPU 或 NVMe（`offload_param.device="cpu"|"nvme"`）。代价按 15 章的带宽表算：CPU 与显存之间只有 PCIe 4.0 x16 有效 ~25 GB/s，NVMe 顺序读 3–7 GB/s，这两跳都直接进 step 时间。DeepSpeed 在 PyPI 的最新版本 0.19.9（2026-10-29），项目已捐给 Linux Foundation（2025-01）；PyTorch 官方文档推荐 FSDP2，FSDP1 的弃用声明没找到（UNVERIFIED，别把博客说法当官方口径）。出处：DeepSpeed ZeRO tutorial。

13. ring AllReduce 的通信量为什么几乎与卡数无关？把式子写出来，再说清它省的是什么、赔的是什么。
   【要点】每节点总通信量 2·(N−1)/N·S（S 为数据量），N 变大时趋于 2S，与 N 无关，这就是 bandwidth-optimal 的由来；赔的是步数 2(N−1)，延迟项随卡数线性增长（Sergeev & Del Bello，Horovod，arXiv:1802.05799）。两条推论要主动给：小尺寸高频同步时延迟先成为瓶颈，所以要做梯度桶化；跨机段要减步数，机内走 NVLink、机间走 RDMA 的分层 all-reduce。与本章 AllReduce 语义题的分工：那题讲数学，这题讲账。

14. 流水线并行的气泡公式是什么？加 stage、加 microbatch、换 interleaved schedule，三件事各自改哪个变量？
   【要点】气泡占比 (p−1)/m（p 为 stage 数、m 为 microbatch 数；换算成总时间占比是 (p−1)/(m+p−1)），interleaved/VPP 变成 (p−1)/(v·m)（Narayanan et al., arXiv:2104.04473）。1F1B 省的是激活显存，它改的是显存曲线，气泡照旧。判据给数值：m 至少 4p 以上，否则加卡带来的并行度先被气泡抵消。与他项目的接口：气泡期间数据可以提前取，气泡期间的梯度同步没法省，所以缓存层的收益在高 PP 度下会被放大。出处：15 章机时账。

15. 一个 step 里数据加载、前反向、通信怎么重叠？重叠失败在指标上长什么样？
   【要点】三条重叠路径：DataLoader 多 worker 预取把解码与增广藏在 GPU 后面，pinned memory + `cudaMemcpyAsync` 让 H2D 与计算并行；梯度累积用 `model.no_sync()` 只在最后一个 micro-step 同步，省掉 N−1 次 all-reduce；反向逐层触发 reduce-scatter/all-gather，与本层计算流水。重叠失败的可观测形态：GPU 利用率均值不掉、方差变大（个别 rank 在等），或全体 step 时间同步抬高且数据队列见底（12 章 6 题、31 题）。答完补一句他的实测口径：他测的是数据等待占比，通信重叠度是从框架侧指标读出的，两者不能混报。出处：05 章、12 章。

16. TP 之外还有 SP/CP/EP，它们各解决什么，通信点分别落在哪？
   【要点】Megatron 的 TP 在每个 transformer 层做 2 次前向 all-reduce 与 2 次反向（arXiv:1909.08053），量随层数与 TP 度线性涨，所以 TP 一般压在机内 NVLink 域。sequence parallel（`--sequence-parallel`，要求 TP>1）把 all-reduce 拆成 reduce-scatter + all-gather，激活显存跟着分摊（arXiv:2205.05198）。context parallel（`--context-parallel-size`、`--cp-comm-type p2p|a2a|a2a+p2p`）切序列长度给长上下文，通信是 K/V 的环形交换。expert parallel 每个 MoE 层 2 次 all-to-all（`--expert-model-parallel-size`）。现状：Megatron-Core 0.20.0（2026-10-29）引入 Megatron-FSDP v2 与 MXFP8 通信；torchtitan v0.4.0（2026-10-28）是 PyTorch 官方参考栈，组合 FSDP2+TP+PP+CP+EP+DCP。

17. GEMM 自己就会做 tiling，为什么还需要 FlashAttention？把最原始的算法逻辑讲清，再说 FA2、FA3、FA4 各自改的是哪一层。
   【要点】问题在 IO 不在算力。标准实现要把 `S = QKᵀ`（N×N）写回 HBM，再读回来做 softmax 得到 `P`，再写回，再读出来乘 V。FlashAttention 论文（arXiv:2205.14135，2022-05-27）给的标准实现 HBM 传输量是 `4N²+4Nd` 个元素，Theorem 2 给的界是 `Θ(N²d²/M + Nd)`，M 是片上 SRAM 容量；显存占用从二次降到线性。第一句要答在链路上：GEMM 的 tiling 优化的是单个矩阵乘内部的数据复用，Attention 需要 tiling 的原因是整条 `QKᵀ → softmax → ×V` 不许把 N×N 的中间结果落到 HBM，这是跨算子的问题，单个 GEMM 做得再好也管不到。
   分块下的 softmax 才是难点，也是「会 tiling 不等于会 Attention」的落点。softmax 必须先减最大值防 exp 溢出，而分块时看不到整行。论文 Algorithm 1 的做法是 online softmax：每块推进时维护 running max `m` 与 running sum `l`，用 `exp(m_old − m_new)` 把已经累加的输出重新标定，再叠上新块的贡献。这套重标定是数学等价的（标题写的是 exact attention），代价是片上多几次乘法与一次 rescale。普通 GEMM tiling 里没有这一步。
   反向更贵：不保存 `S` 与 `P`，只存输出 `O` 和每行的 logsumexp，反向时重算。这是拿算力换带宽，在 compute-bound 的规模上划算，序列很短时不一定。
   三版各自改的层次（报数字先报硬件条件，全部是论文自报口径）：FA2（arXiv:2307.08691，2023-07-17）改并行划分与浪费，三件事是减少非 matmul FLOPs、单个 head 沿序列维跨 thread block 并行、block 内 warp 之间重新划分工作以减少 shared memory 往返；论文称比 FA1 快约 2 倍，A100 上到理论峰值的 50–73%（FA1 是 25–40%），端到端 GPT 训练最高 225 TFLOPs/s、约 72% MFU。FA3（arXiv:2407.08608，2024-07-11）利用 Hopper 的异步能力：TMA 与 warp specialization、matmul 与 softmax 交错重叠、FP8 下用分块量化加 incoherent processing 压误差；论文称 H100 上比 FA2 快 1.5–2.0 倍，FP16 到 740 TFLOPs/s（75% 利用率，FA2 只有 35%），FP8 接近 1.2 PFLOPs/s。FA4（arXiv:2603.05451，2026-03-05）针对非对称硬件扩展做流水线协同设计，CuTe-DSL 实现，flash-attention 仓库 README 给出 `pip install flash-attn-4`；论文称 B200 BF16 比 cuDNN 9.13 快至 1.3 倍、最高 1613 TFLOPs/s（71%）。FA3 正文里 WGMMA、ping-pong 调度这些具体名字没能逐字核到，答的时候按「异步与交错」讲，别把细节当定论。
   系统侧的接口（这题对你真正的用处）：Attention 是 O(N²) 计算量配 O(N) 显存，长上下文服务的成本曲线按这两项算，也解释 06 章 1 题、2 题里 prefill 是 compute-bound、decode 是 bandwidth-bound 的来由（decode 每步只新增一行查询，却要读全部 KV）。prefix cache 省下的正是这段 prefill 重算，能省多少按 06 章 7 题的命中率折。
   答题边界：没写过 kernel 就明说读过论文与接口层、没做过 kernel 实现，可验证的三段答法按本章 33 题；论文数字记不准就按 16 章第五节的规矩声明量级。

18. 训练侧要确定性，到底在要求什么？「和旧实现 bitwise 一致」与「多次运行稳定复现、误差满足训练要求」是同一件事吗？不确定性从哪来，开关的代价落在哪。
   【要点】两种口径先分开。PyTorch 的 `torch.use_deterministic_algorithms(True)` 文档原话是「给定相同输入、相同软硬件，总是产生相同输出」，同时明确写了「这一项单独设置不足以让整个应用可复现」，遇到只有非确定性算法可用的算子会抛 RuntimeError。和旧 baseline bitwise 对齐是回归对比的口径，多次运行自身可复现加误差达标是实现替换的口径。换 kernel 的人通常只需要后者，因为浮点加法顺序一变就不可能和旧的 bitwise 相同。把这个区分讲出来，就是在回答里体现了确定性有层次。
   不确定性从哪来：浮点加法不满足结合律，累加顺序由调度决定。FlashAttention 反向里 dQ 按 KV 块分别算出，再用 atomicAdd 累到同一个 `dQaccum`，顺序不定。flash-attn 的 `deterministic` 参数（`hopper/flash_attn_interface.py`，默认 False）README 原话是「反向的确定性实现，稍慢、多占显存；正向始终是确定性的」；`csrc/flash_attn/src/flash_bwd_kernel.h` 的注释给出做法，让每个 thread block 累到不同的 `dQ_accum` 缓冲，避开同一地址上的原子竞争。PyTorch 2.14.1 侧的封装在 `torch/nn/attention/_fa3.py`，由 `torch.are_deterministic_algorithms_enabled()` 决定是否走确定性反向，并把 `num_splits` 强制为 1，原因是 split-K 类切分同样引入原子累加。
   代价落在哪：固定累加顺序会带来等待与串行化，额外缓冲占显存，或者改用两级归约多一趟读写。这与本章 13 题、15 题是同一类账：省什么、赔什么、赔进哪段时间。
   怎么验，顺序不能反：先固定口径，再谈一致。三层依次是同输入跑两次比 bitwise（检验新实现自身可复现）、跑多次比统计稳定性、对参考实现（fp64 或 eager attention）比误差，并说清容差的来源是训练侧对梯度尺度的要求，loss 曲线只作参考。判据先写死再动手，这跟 14 章给验证定门禁是同一个纪律。
   没写过 kernel 时的答法：把口径区分讲完，接一句自己在评测协议里怎么处理同类问题（基准固定同一随机种子与同一份参数快照，比相对差值不比绝对值），可验证性的四层按 16 章第四节。没做过就别编。

19. 一个 token 从 embedding 进到 logits 出来，一层 decoder 里依次算什么？参数量落在哪几块，拿一份公开的 config 把 7B 的账手算一遍。
   【要点】一层的顺序是 RMSNorm → QKV 投影 → RoPE → attention → 输出投影 → 残差 → RMSNorm → FFN → 残差。参数集中在两处：attention 的四个投影、FFN 的 up/gate/down 三个投影。拿 LLaMA-2-7B 的 config 字段算（`hidden_size` 4096、`num_hidden_layers` 32、`num_attention_heads` 32、`num_key_value_heads` 5、`intermediate_size` 11008、`vocab_size` 32000）：`W_q` 与 `W_o` 各 4096×4096 = 16.8 M，GQA 的 `W_k`、`W_v` 各 4096×(5×128) = 2.6 M，attention 合计 38.8 M；SwiGLU 的 FFN 是 3×4096×11008 = 135.3 M。一层 174 M，32 层 5.6 B，加 embedding 0.13 B，落在 6.7 B 这个量级——这就是「7B」的来历。稠密模型的 FFN 通常是参数大头，attention 的占比随 GQA 与层数变化，所以 06 章 46 题里 MoE 把 FFN 换成专家之后，账要整段重算。答题落点：字段名要报得出，配置差异要能说出它改的是显存还是带宽。config 数字以当日 HuggingFace 仓库为准，记不准就报量级。

20. 参数量、每 token 的 KV 字节数、单步 FLOPs 三个式子自己推一遍，然后从「模型 + 并发目标」推到需要几张卡。
   【要点】三式。每 token 的 KV 字节 = 2（K 与 V）× 层数 × `num_key_value_heads` × `head_dim` × dtype 字节数：LLaMA-2-7B 按 fp16 是 2×32×5×128×2 = 80 KB/token，32k 上下文单条序列 2.5 GB；15 章 7 题那个 70B FP8 的例子（80 层、8 个 KV 头、head_dim 128）代入同式得 2×80×8×128×1 = 160 KB/token，32k 就是 5 GB。GQA 省的就是 `num_key_value_heads` 这一项，MLA 改的是整式的形状（见本章 22 题）。权重字节 = 参数量 × dtype 字节数，7B fp16 是 14 GB，再扣激活、临时缓冲与 CUDA 上下文，剩下的才是 KV 池，`--gpu-memory-utilization` 管的正是这条分配线。计算量：单 token 前向约 2N FLOPs（N 为参数量），训练前向加反向按 6ND 记（D 为 token 数），attention 项 2N·d·s 在长上下文时才追上投影项。这三式是 06 章 1 题、2 题「prefill compute-bound、decode bandwidth-bound」的定量入口，也是本章 29 题 MFU 与 goodput 的分母。面试里报 FLOPs 一律注明是理论峰值还是实测吞吐，两个口径差出来的名字就叫 MFU。

21. RoPE 把位置写在哪里？为什么超出训练长度就坏，位置插值与 YaRN 各改什么，对服务侧的成本曲线有什么影响。
   【要点】RoPE（arXiv:2104.16673）不给 embedding 加位置向量，而是按位置对角旋转 Q 与 K，让点积只依赖相对位置；每个频率维度一个旋转角，长程维度转得慢。模型对没见过的旋转角外推没有约束，超出训练长度后注意力打分失真。位置插值把位置整体缩放 `(t → t·L_train/L_target)` 让角回到见过的范围；YaRN（arXiv:2309.00071）按维度分频段处理，高频少插、低频多插，并用温度补偿注意力打分。系统侧的接法要算出来：`max_model_len` 决定单条序列 KV 上限（每 token 字节数乘长度），也决定 prefix cache 里一条长前缀占多少块、被一次驱逐带走多少；KV 与序列长度成正比，attention 的计算量与它自己的中间量与长度平方成正比，所以长上下文的成本曲线由这两条线一起抬。15 章 7 题算并发路数时用的就是这个乘积。社区常见的 NTK-aware 与「一根频率线就扩到百万 token」这类说法没有论文口径可引，标 UNVERIFIED，答题别当结论。

22. MLA 把 KV 缓存成什么？和 GQA、MQA 比省的是式子里的哪一项，对缓存层、路由层、传输层各改了哪一个前提。
   【要点】MLA 出自 DeepSeek-V2（arXiv:2405.04434，2024-05）：不逐头缓存 K 与 V，而是把它们联合下投影成一个低秩潜向量 `c_KV`（DeepSeek-V2-Lite 的 config 字段是 `kv_lora_rank: 512`），外加一份解耦的 RoPE 键（`qk_rope_head_dim: 64`）；推理时缓存这份潜表示，attention 内部再上投影还原各头的 K、V。拆掉的是本章 20 题那个式子里的 `num_key_value_heads × head_dim`，换成 `kv_lora_rank + qk_rope_head_dim`，于是每 token 每层从 64 KB 量级降到 (512+64)×2 B ≈ 1.1 KB；论文给的说法是 KV cache 较 MHA 基线削减 93.3%，另有博客写成 MHA 的 5%–10%（博客口径，数字按论文那句报）。GQA、MQA 减的是头的共享度、要动训练，MLA 同样在训练期定型，但它是把 K 与 V 联合压，换来接近 MHA 的质量；`--kv-cache-dtype fp8` 是运行期的另一条轴，两者叠乘。vLLM 侧的支持起点有博客写成 0.5.x 起、并配 FlashMLA 内核，这串版本断言没到仓库逐字核实，标 UNVERIFIED。三个前提要主动讲：缓存层的每字节命中率对应的 token 数变了；路由层的「实例还有多少 KV 余量」这个信号的分母变了，06 章 12 题的扩缩容判据要重算；传输层的 PD 分离搬运量随注意力结构走，06 章 39 题的账要跟着降。延伸：DeepSeek V3.2 的 DSA（Lightning Indexer 打 top-k、把 attention 从 O(L²) 降到 O(L·k)）只在单一中文博客里读到，k 值与训练细节未核实，答题时先说这是读到的说法。这一题是他被追问「换个模型你那套路由还成立吗」时唯一拿得出的答案，答法是结构变了哪一项、判据跟着怎么改。

23. Triton 是什么？torch.compile、算子融合、CUDA Graph 各解决哪一段开销，什么负载下失效？
   【要点】四个词分四层，按「谁在什么时候做什么」讲，不并列背。Triton 是写 GPU kernel 的 Python DSL（`@triton.jit`），由 OpenAI 维护、走 PyPI，产物经 LLVM 到 PTX；它替代手写 CUDA 的那部分是 tile 与访存布局，抽象单位是 program，一个 program 对应一个 CTA、内部含多个 thread。torch.compile（`torch.compile(model, mode="reduce-overhead")`）走 torchdynamo 抓图、Inductor 出代码，Inductor 在 GPU 上默认吐 Triton。算子融合是它的收益点：LayerNorm 与逐元素链原本每个算子写回一次 HBM、下一次再读回来，融合后一趟读写，省的是 HBM 带宽，算力不变，用算术强度（FLOPs/字节）对 roofline 判断有没有到拐点。CUDA Graph 把一串 kernel launch 录成一次提交，省的是 launch 开销与 CPU 侧调度；失效点是形状变化——动态 batch 与变长序列每变一次形状就要重新捕获，这正是 06 章 decode 阶段动态 batch 让引擎难走这条路的原因。vLLM 与 SGLang 都在持续跟进这三层，具体 kernel 的归属随版本变，标 UNVERIFIED，答题报「机制归哪层」；「谁家有」随版本变化，容易被推翻。档位说明：这条按 7:3 只算识字量，目标是能说出每一层省哪段开销、赔什么，不立项、不投简历。

## T3 · 扩展与速查

24. ★ 解释 CUDA 的 global memory 合并访问，什么样的访问模式最差。
   【要点】先讲机制：warp 的一次访存指令由硬件合成向 HBM 发的事务，事务粒度是 sector（32 字节，一组 cache line 拆成 4 个 sector），一次连续 128 字节的 warp 访问可以并成 4 个 sector 事务。再说最好模式：线程 i 读 `a[tid]`，32 个线程覆盖连续 128 字节。最差模式给三个具体形态：跨步为大的按列遍历（转置的朴素写法，与 10 题的 shared 修法成对）；指针追逐与链表式索引；embedding 与稀疏特征表按 hash 取随机行——每线程一个 sector，带宽利用率掉到 1/32 这个量级。修法清单：数据布局从 AoS 改 SoA、结构体内按访问维度重排、对齐到 128 字节、向量化读写（`float4` 一次 16 字节）、只读路径用 `__ldg`/`const __restrict__` 走只读缓存。可复算的一句账：把「利用率 1/32」写成实测带宽对峰值带宽的比值，峰值按卡的规格报（A100 40GB SXM 约 1.55 TB/s）。与 10 题分界写清楚：合并访问发生在 global 与寄存器之间，看 warp 的地址是否连续；bank conflict 发生在 shared 上，看地址是否错开 32 个 bank，同一个 kernel 里两处都可能出问题。出处：10、33 题，S8 立项第五节第 3 条（不声称优化过生产算子）。
25. ★ 什么是 CUDA Stream 与 Event？如何用它们实现拷贝与计算重叠。
26. ★ 什么是 CUDA Graph？它如何降低 kernel launch 开销，捕获时有哪些限制。
27. 解释通信与计算重叠的常见实现方式。
   【要点】按「谁藏谁」分四类，每类给开关名与失效条件，别笼统答「用多个 stream」。框架自动：DDP 在反向图里按拓扑逆序给梯度桶挂 hook，桶满即发 all-reduce，通信落在剩余层的计算后面（`bucket_cap_mb` 默认 25 MB，桶太小则通信次数涨、太大则暴露窗口长，见 11 题）。集合形态改造：FSDP/ZeRO-3 的参数 all-gather 提前发起（Megatron 侧 `--overlap-param-gather`、`--overlap-grad-reduce`，PyTorch 侧 `set_modules_to_forward_prefetch`），本层计算时预取下一层参数。流与拷贝：独立通信 stream 加 `cudaMemcpyAsync` 配 pinned memory，让 H2D 与计算并行（05 章 17 题、25 题那条链）；CUDA Graph 只省 launch，改不了依赖顺序（23、26 题）。算法重排：1F1B 让反向与下一个 microbatch 的前向交错，PP 的 stage 边界用 `--overlap-p2p-communication` 把 send/recv 藏进计算；梯度累积用 `no_sync()` 直接省掉 N−1 次同步（15 题）。失效条件三条，主动给：计算段短于通信段时重叠窗口不够，小模型小 batch 先撞上；NCCL 的通信 kernel 也占 SM 与寄存器，与计算抢发射资源；机内 NVLink 与机间网卡各自有自己的带宽上限，跨机段藏不住就要看分层 all-reduce（13 题）。观测口径：GPU 利用率均值不掉而方差变大，或全体 rank 的 step 时间同步抬高，两种形态分开报（12 章 6 题、31 题）。接回他的项目：P1 藏的是等数据的时间，这里藏的是等通信的时间，两笔都进同一个 step 预算，高 PP 度下的气泡期正好可以拿来做数据预取（14 题那句）。出处：11、13、14、15 题。
28. 什么是 MoE？它的路由机制对系统负载均衡提出什么要求。
   【要点】白话一句：把 FFN 换成 `E` 个专家，门控给每个 token 打分选 top-`k`（常见 k=2），一次前向只激活 `k/E` 的 FFN 参数。系统侧的意义先说对偶：参数量与每 token 计算量解耦，容量涨、FLOPs 按激活参数走——账要按 20 题那两个口径分开算（权重字节按总参数、每 token 计算按激活参数）。再答路由对系统的三条要求。一、通信点：专家分布在别的卡上就必经 all-to-all，每个 MoE 层前反向各 2 次（EP，见 16 题），token 数 × 隐藏维的收发落在所有 EP rank 上，这类流量跟不上 NVLink 的聚集带宽，所以 EP 度通常受机内域限制。二、负载不均：门控打分不稳定会让热点专家被过量 token 命中，一个 rank 的 FFN 排队，整个 step 等它——这就是「负载不均」的机制来源，跟 DP 里数据分片不均衡造成的慢节点是同类问题（05 章 13 题）。三、容量因子：每个专家每批只接固定槽位，超出就丢弃（GShard/Switch Transformer 一路的做法，容量因子 `C` 常取 1.0–1.25 量级，具体默认值按当日实现核）；丢弃带来训练不稳与梯度缺失，加大 `C` 则空转显存和带宽。均衡方法讲演进两条：辅助损失把各专家接收比例拉平（Switch Transformer, arXiv:2101.03961）；DeepSeek-V3 用无辅助损失的偏置调整（读到的说法，论文口径没逐字核，标 UNVERIFIED）。推理侧的接法要主动补：权重侧账变了，KV 侧账按 20 题式子随层数与 KV 头走，专家多并不改每 token 的 KV 字节；decode 阶段访存受限、算力等待，路由让有效参数远大于激活参数，显存下限由总权重决定，这一条接到 06 章 46 题。会被追的一问：「总参 200 B、激活 21 B，你要几张卡放权重」——按 20 题算权重字节再除以单卡容量，答完补一句「还要给 KV 池留位置」。出处：16、20 题，06 章 46 题。

29. 大规模训练的故障率怎么进你的容量账？goodput 与 MFU 分别是什么口径？
   【要点】goodput 是扣掉故障、恢复、重算浪费之后的有效训练进度每单位时间（Google Cloud 的定义文《Goodput metric as measure of ML productivity》）；MFU 是实测吞吐相对按峰值 FLOPS 算出的理论吞吐的比值（PaLM, arXiv:2204.02311）。可直接引用的实测条件：Llama 3.1 405B 用 16,384 张 H100 跑 54 天，非预期中断 419 次，约每 3 小时一次，有效训练时间占比 >90%，GPU 相关故障占 58.7%、HBM3 约 17.2%，MFU 38–43%（arXiv:2407.21783）。用法：把「等数据」造成的 1% 好吞吐损失折算成机时钱（13 章的账），这是他把系统问题译成机时钱的固定句式。

30. 训练任务挂了怎么自动拉回来？NCCL 与弹性训练的当前参数名报一遍，并说清默认值造成的误诊。
   【要点】当前变量族是 `TORCH_NCCL_*`：`TORCH_NCCL_ASYNC_ERROR_HANDLING` 默认 3，`TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC`、`TORCH_NCCL_ENABLE_MONITORING`、`TORCH_NCCL_MONITOR_INTERVAL_MSEC`、`TORCH_NCCL_DUMP_ON_TIMEOUT`、`TORCH_NCCL_TRACE_BUFFER_SIZE`（flight recorder 的开关与容量）；旧名 `NCCL_ASYNC_ERROR_HANDLING`、`NCCL_BLOCKING_WAIT` 已标弃用。torchrun 的重启次数是 `--max-restarts`，默认 0，参数表里没有 `--restarts`；另有 `--monitor-interval`、环境变量 `TORCHELASTIC_RESTART_COUNT`。误诊来源：默认配置下节点掉队后任务卡到超时退出，自动重启不会发生，表象是 NCCL 超时，实际原因是弹性没开。出处：PyTorch ProcessGroupNCCL 环境变量文档、elastic run 文档。

31. checkpoint 的保存与恢复怎么算账？异步保存解决哪一段，代价落在哪？
   【要点】`torch.distributed.checkpoint`（DCP）提供 `dcp.save`、`dcp.async_save`（返回 Future）、`dcp.load`，配合 `Stateful` 协议、`AsyncStager`/`BlockingAsyncStager`，状态字典走 `get_state_dict`/`set_state_dict` 与 `StateDictOptions(full_state_dict, cpu_offload, broadcast_from_rank0)`。异步保存把写盘移出 step 关键路径，代价是暂存副本占主机内存、且写盘未完成时新副本要覆盖旧副本，这里要么排队要么加缓冲。恢复侧的账：权重从存储进显存的分钟级开销（15 章），它决定 checkpoint 间隔的下限和最优值。可延伸的一句：间隔太短会挤压数据管道的带宽，与他缓存层争资源。出处：PyTorch DCP 文档。

32. bf16、fp16、FP8 三档精度各对系统提什么要求？
   【要点】fp16 指数位 5 bit，梯度易下溢，需要 loss scaling（`torch.amp.GradScaler('cuda')`，配 `torch.autocast('cuda', dtype=...)`）；bf16 指数位 8 bit、动态范围与 fp32 同级，免掉缩放，硬件要求 Ampere 之后。FP8 训练：Hopper（H100/H200）走 Transformer Engine 或 Megatron `--fp8-format`；MXFP8（1 个 32 元素块共享一个 scale）只在 Blackwell（B200/GB200）上有硬件路径，Megatron-Core 0.20 与 torchtitan（`float8.py`、`mxfp8/`、torchao `Float8CurrentScaling`）支持；DeepSeek-V3 用 FP8 block-scaled 与 UE8M0 scale。答题落回系统层：降精度省的是显存与带宽，谁受益取决于负载是 memory-bandwidth-bound 还是 compute-bound（06 章 2 题）。

33. 面试官问「你读过 Megatron 或 vLLM 的源码吗」，怎么答得可验证？
   【要点】2026 年的 JD 把框架源码阅读写成显性要求（懂车帝大模型 Infra 方向列 SGLang/vLLM/Megatron/DeepSpeed）。可验证的答法只有三段：入口在哪个文件、一条 step 依次经过哪几个函数、我在哪一行下过断言或改过行为。报仓库名和「看过」没有信息量。他没读过训练框架就明说读的是推理与数据侧，然后把本章前几题按机制答完整。声称读过又被追问细节，代价比承认没读过更大（16 章第四节的过程痕迹四层）。

34. ★ 上机做 NCCL 通信优化，前五步依次是什么？为什么第一步不是调参？
   【要点】第一步量露出来的通信时间，不是通信总时间。DDP 按 bucket（`bucket_cap_mb` 默认 25 MB）在反向里挂 hook，桶满即发 all-reduce，与剩余层的 backward 重叠；抓一步 torch profiler 导 chrome trace，看 nccl kernel 与计算 kernel 在时间轴上重叠多少，露出接近零则调 NCCL 无收益，瓶颈在数据加载或计算（27 题）。第二步分三种病因：小消息多、每次几 KB 是延迟主导，药是合并桶、梯度累积、换 Tree；大消息带宽跑不满是带宽主导，药是查拓扑与 channel 数；各 rank 到达 collective 的时刻差很大是慢节点，药不在 NCCL，去查数据加载与计算不均，这一条最常被误诊成网络故障（12 章 12 题）。分法是跑 `all_reduce_perf` 扫 message size 8 B 到 1 GB 看 busbw 曲线，小 size 段低属正常，大 size 段仍低于硬件上限才是通信有病；上限自己算，IB 200 Gb/s 即 25 GB/s 每网卡，ring 的 busbw = algbw × 2(N−1)/N。第三步读 `NCCL_DEBUG=INFO` 配 `NCCL_DEBUG_SUBSYS=INIT,GRAPH,ENV,TUNING`：INIT 段看卡间走 P2P/IPC 还是 SHM 还是 Socket，GRAPH 段看 ring 顺序与 channel 数，ENV 段看实际读到的变量。三个常见坑：容器没挂 `/dev/infiniband` 则静默退回 TCP、带宽掉一个数量级；多网卡没指定 `NCCL_IB_HCA` 则选错网卡；PCIe ACS 开着则 P2P 走不了退回 SHM。第四步调参一次只动一个、跑三遍取中位数：`NCCL_ALGO` 小消息 Tree、大消息 Ring、有 NVSwitch 用 NVLS；`NCCL_PROTO` 的 LL 带宽利用率约一半但延迟低、LL128 约 95%、Simple 跑满；`NCCL_MIN_NCHANNELS`/`MAX_NCHANNELS` 多了抢 SM、少了带宽不满；`NCCL_BUFFSIZE` 默认 4 MB。第五步是重叠层，不碰 NCCL 内部：`bucket_cap_mb` 两头取舍、`gradient_as_bucket_view=True` 省一次拷贝、FSDP 的 forward/backward prefetch 决定下一层参数何时提前 gather（27 题）。没有 IB 的机器上带宽数字不可采信，与 S2 的 Soft-RoCE 止损同一条判据；单卡双 rank 能练读输出与认路径，练不到集群性能。出处：NCCL 环境变量文档、nccl-tests、S2 与 N4 的止损条款。

35. ★ 千卡训练挂了一次，恢复时间拆成哪四段？各段的杠杆是什么，哪一段在千卡规模下最大？
   【要点】四段按时间顺序：检测（进程真死到系统判定死）、定位与隔离（找出坏节点踢出资源池）、重新集合（rendezvous 重建 process group）、加载 checkpoint 与预热（读回权重与优化器状态、重建 CUDA context、NCCL 重新建链、数据管道定位到上次 sample）。先量四段各占多少再决定优化哪段，打点位置是进程启动、rendezvous 完成、load 完成、第一个 step 完成，杀一个 rank 看日志时间差。检测段是第一大杠杆且最易被忽略：`init_process_group` 的 timeout 默认 30 分钟，改成 5 分钟并打开 `TORCH_NCCL_ASYNC_ERROR_HANDLING`（默认 3）让 watchdog 超时后主动结束进程，配 `--max-restarts`（默认 0，不开则自动重启不发生，30 题）；代价是慢节点可能被误杀，所以要配 per-rank 的每步计算时间与到达 collective 时刻分布比较。定位段靠心跳与 DCGM 的 XID 错号（63/64 是 ECC、79 是掉卡、74 是 NVLink）。加载段在千卡规模下最大，账要会算：70B 混合精度 Adam 每参数 16 字节，权重 2 + 梯度 2 + 一阶矩 4 + 二阶矩 4 + fp32 主权重 4，共 1.12 TB；分片到 1024 卡每卡读 1.1 GB，存储聚合带宽 100 GB/s 理论 11 秒，但一千客户端同读一份对象存储时元数据与连接数先撑不住，实际几分钟，所以 checkpoint 必须每 rank 写自己的分片（DCP 的做法）而不是 rank0 汇总写一份（31 题、15 章 2 题）。保存间隔按 √(2 × 保存成本 × 平均故障间隔) 定（Young/Daly），保存成本实测、故障间隔用集群历史（29 题的 Llama 3.1 每 3 小时一次可作参照）。再往上是热重启：checkpoint 在邻居节点主机内存冗余一份，故障时从邻居读，恢复从分钟级到秒级（Gemini, SOSP'23）；上机后先量「从存储读」与「从邻居内存读」的差，这个差决定要不要做冗余。慢节点比硬故障难治，硬故障有进程退出，慢节点只是慢、collective 等它、全集群一起慢（MegaScale, NSDI'24 的 per-rank 诊断与自动隔离）。最后是演练：定期杀 rank、断网、降频、写满盘，量四段时间验证告警与自动恢复真的会动。规模答法：笔记本用 gloo 四进程跑机制并量四段，标明口径；千卡用数字账外推，外推假设报出来，不声称做过千卡。出处：PyTorch DCP 与 elastic 文档、MegaScale、Gemini、N4。

## 增补 · GPU 深度、PyTorch 资源视角、CUDA 到接口层（36–53，2026-11-09）

> 三档优先级：GPU 深度按 T1 准备，是面试重点；PyTorch 只到训练与推理对资源的需求，不碰模型设计；CUDA 说到延迟降在哪段、动态 shape 为什么受限、vLLM 与 TensorRT 怎么绕，再深就不答。题号从 36 续，1–35 不动，「08 章 17 题」这类跨章引用仍然有效。硬件数字按当日手册核，报不准就说量级如此、会后核实。

### GPU 深度（36–43）

36. SM 靠什么隐藏访存延迟？occupancy 拉满和算力跑满之间差什么？
   【要点】三段。第一段发射模型：一个 SM 有多个 warp scheduler，每周期各向一个调度分区发一条指令；warp 切换不动内存，寄存器和 PC 都在片上，所以切换本身没有成本（`__syncthreads()` 卡住的是 block 内全部 warp，跟切换成本无关）。第二段讲谁提供可发射的 warp：访存要等几百个周期，这段空档由其他就绪 warp 填；就绪 warp 有两个来源，驻留 warp 数（occupancy）和单个 warp 内互相不依赖的指令链（ILP）。第三段讲上限：驻留 warp 数由三样取最小，每 SM 的 warp 上限（A100/H100 每 SM 64 warp、2048 线程）、每线程寄存器数 × 线程数对 65536 的整除结果、shared memory 对每 SM 配额（10 题）。追问「occupancy 100% 吞吐为什么没涨」，四条依次答：warp 都阻塞在同一条依赖链上没有 ILP、已经撞带宽上限（37 题）、Tensor Core 没被用上（39 题）、grid 除不尽 SM 数在尾部空转（40 题）。答题边界：报 occupancy 的定义式和三个限制因子，「提高 occupancy 就一定变快」这句不说。

37. 算术强度怎么算，roofline 拐点在哪，用它判断 prefill 与 decode 谁受限？
   【要点】定义：算术强度 = FLOPs ÷ 从 HBM 读的字节数。拐点：AI\* = 峰值算力 ÷ 峰值带宽。代 A100 的 bf16 稠密峰值 312 TFLOPS、HBM2e 1.55 TB/s（与 10 题同一组数字），拐点约 200 FLOPs/byte，低于它这个 kernel 就是带宽受限。再代真实形状：decode 每步每层把权重读一遍、每参数算 2 FLOPs，每字节摊到约 2 FLOPs（batch=1 的 GEMV），跟拐点差两个数量级，瓶颈在带宽；prefill 是 b×s 很大的 GEMM，一份权重被复用几千次，AI 上千，撞算力上限。这三行是 06 章 1、2 题那句「prefill compute-bound、decode bandwidth-bound」的分母，也是 20 题三个式子的落地处。两条追问：batch 从 1 推到 64，decode 的瓶颈会不会挪——GEMV 变 GEMM，AI 随 batch 线性涨，会在某个 batch 跨过拐点，这个数自己按式子算；int8/int4 量化省的是字节，AI 随之上移，所以小 batch 上量化收益最大。答题边界：报 FLOPs 先说理论峰值还是实测吞吐，两个口径差出来的名字叫 MFU（29 题）。

38. 从寄存器到主机内存五级存储，各自的带宽与延迟量级，哪一级归你控制？
   【要点】按延迟从短到长排：寄存器（每线程私有，无访存周期）、shared memory 与 L1（同一块 SRAM，A100 每 SM 合池 192 KB、单 block 最多 164 KB，按手册核）、L2（全 GPU 共享，A100 40 MB、H100 50 MB，按手册核）、HBM（1.55 TB/s 量级、几百 ns）、主机内存（过 PCIe 4.0 x16 有效约 25 GB/s，15 章那张表）。要说的三点：shared 与 HBM 的带宽差一个数量级，这是 10 题往 shared 搬数据和 17 题不许把中间量落到 HBM 的全部理由；L2 由硬件管理，程序只能间接影响，CUDA 给的口子是把一段 buffer 标成 persisting 让它留在 L2（access policy window，名字按 CUDA 文档核，讲机制就够）；PCIe 与 NVLink 的量级差决定 15 题权重预取和 PD 分离搬运的账。追问「小 tensor 反复读为什么不放 shared」：shared 的生命周期跟 block 走，跨 kernel 的复用管不到。

39. Tensor Core 什么时候走上它，走不上差多少？
   【要点】条件四条：数据类型（fp16/bf16/tf32/FP8/INT8 有路，fp32 走不到）、维度对齐（mma 的 tile 对 m/n/k 有倍数要求，16 与 8 的具体倍数按代际不同）、形状要能整除到 warp 级（否则尾部浪费）、数据在 shared 里的布局满足矩阵片要求（10 题的 bank conflict 在这里同样管）。走不上就退回 CUDA core 做同样的乘加，差值按当日硬件核，报不准就说一个数量级。指令名按代际说：`mma.sync` 一路通用，Hopper 上是 wgmma、Blackwell 上是 tcgen05 这一类（两个名字按 PTX 文档核，别当定论）。系统侧的接口两条要主动给：模型的 hidden size 取 128 的倍数、attention 的 head_dim 取 64 或 128，除了数学原因还有这一层对齐（21、22 题）；FP8 的 scale 粒度是同一层的事（tensor-wise 还是 block-wise，DeepSeek-V3 用 block-scaled 与 UE8M0，32 题）。追问「fp32 的模型能不能靠 TC 提速」：tf32 是中间档，`torch.backends.cuda.matmul.allow_tf32` 决定矩阵乘走不走它，默认值随版本变过，开口先报你查的版本。

40. grid 的 block 数除不尽 SM 数会亏多少？这个浪费叫什么，怎么修？
   【要点】机制：block 按 wave 轮转发射，最后一波没排满，空转的那段叫 wave quantization 或者 tail effect。算例要会当场推：A100 108 个 SM，每 SM 驻留 1 个 block 的配置跑 1000 个 block，ceil(1000/108)=10 波，实际工作量只有 9.26 波，最后一波只占 28 个 SM，浪费约 7%；block 数越接近 SM 数的整数倍越省。追问「为什么大 GEMM 不觉得疼、decode 的小 kernel 疼」：每 block 工作量大时尾部摊薄，每 block 只有几十微秒时取整空转就是净损失——这正是 41 题那三个利用率口径打架的现场。修法按成本排三条：调 block 尺寸让 grid 对齐；persistent kernel，发 SM 数 × 每 SM 驻留数那么多个 block，block 内部循环取任务，省掉反复发射和尾部；多 stream 让不同 kernel 的尾部互补。`__launch_bounds__` 和 maxrregcount 靠压寄存器换驻留 block 数，也在改这道取整。

41. `nvidia-smi` 的 GPU-Util 都 100% 了，吞吐为什么还能很差？
   【要点】先给定义：这一列是采样周期内至少有一个 kernel 在跑的时间占比，它不分跑的是一个 warp 还是六十四个，不分用的是 CUDA core 还是 Tensor Core，不分在搬数据还是在算。于是三种情况都能 100% 而吞吐低：kernel 在跑但 occupancy 极低（36 题）、在跑但全是带宽搬运（37 题）、TC 完全没用上（39 题）。第二层报真正要看的三个口径：SM 侧吞吐占比、DRAM 带宽实测值对峰值、TC 利用率，Nsight Compute 的 section 里都有对应 metric，指标名不背，开口说明是从 ncu 读出来的。第三层报采集侧的坑：`nvidia-smi --query-gpu=utilization.gpu` 的采样窗口和查询周期错配会把短 kernel 的空档平均掉；MIG 切的是 SM 与显存分区，报数前要说清整卡还是切片；MPS 下多进程共享同一张卡，这一列变成共享域的总体值；DCGM 的 `DCGM_FI_DEV_GPU_UTIL` 是同一个口径导出给 Prometheus 的指标，35 题的 XID 也从 DCGM 读。追问「利用率均值正常但 P99 变差」：方差和分位比均值有用，跟 12 章 6 题、31 题同一套读法。

42. 同一份配置两次跑出 20% 的差，先查什么？
   【要点】先查时钟再查软件。第一步读 throttle 状态：NVML 的 `nvmlDeviceGetCurrentClocksThrottleReasons` 给出功耗封顶、硬件慢速、热慢速、同步加速这一类位（名字按 NVML 文档核），`nvidia-smi -q -d PERFORMANCE,POWER,CLOCK` 给当前时钟与上限。第二步知道笔记本为什么特殊：功耗预算与桌面卡不同档，充电与电池两种供电下同一份配置能差出你看到的这个数——S8 坑清单里这条是唯一有明确物理原因、能当场复现的。第三步锁频再测：`nvidia-smi -lgc` / `-lmc`（要 root，需要还原时清掉），报出你锁到的值；锁不住说明被功耗或温度压着，这个事实本身就是结论。第四步测量纪律：预热到时钟稳定才开始采样，每个采样点跑固定次数取中位数（16 题的重复口径），把 power、temperature、clock 三列与吞吐写进同一份 CSV，出结论前先看这三列有没有台阶。答题边界：写进简历前确认自己真的复现过，没有就按 16 章第四节说成读到的机制。

43. 显存够不够看哪个数字？为什么只报一个口径会被追一句？
   【要点】两个口径先分开：allocated 是 tensor 实际占的字节，reserved 是 PyTorch 的 caching allocator 从 CUDA 拿住不还的字节（`torch.cuda.memory_allocated()` / `memory_reserved()`，峰值另有 `max_memory_*`），8 题的实测口径就是这两个。差值来自 block 复用与碎片：tensor 大小不一、释放顺序和申请顺序不同，会出现总字节够但没有一段连续空间。两个数一起报、再解释差值，才算懂这一层。控制手段：`PYTORCH_CUDA_ALLOC_CONF` 的 `expandable_segments:True` 用可扩展段降碎片（变量名当前有效，行为按当日文档核）；`empty_cache()` 把 reserved 还给驱动，代价是下一次分配慢，适合做实验边界，不适合放服务路径。vLLM 走另一条：启动时按比例一次性圈走显存建 KV block pool，池内自己分页，绕开 PyTorch 分配器，这就是 51 题那句 OOM 落点。实测 block 数比按显存算的纸面值小，差额归给权重、激活、临时缓冲、CUDA context、碎片五块（51 题与 S8 的 51 题），这笔账要和 8 题对上。

### PyTorch 资源视角（44–51）

44. PyTorch 这个库管到哪一层，边界在哪？
   【要点】五块按调用顺序：Python 前端（`torch.nn`、`torch.optim`、Tensor API）、ATen 算子库（每个算子一份 C++ 实现加多后端分派表）、autograd 引擎（45 题那条反向图）、`torch.distributed`（DDP、FSDP、DTensor、ProcessGroup 后端）、`torch.compile`（dynamo 抓图、inductor 出代码，23 题）。dispatcher 只讲到「同一个 `add` 按 dtype 与 device 查 kernel 表，决定走 CUDA 还是 CPU、走 ATen 默认实现还是自定义后端」，再往下的 autograd key、boxing 不碰。边界一句话讲完：PyTorch 排 kernel，不管请求。它不知道一批序列里哪个该先出、不管 KV block 的分页与驱逐、不做副本间路由，这三件事分别是 vLLM 的 scheduler、block manager 和 P2 网关（46、50 题、06 章）。追问「为什么推理服务不把 eager PyTorch 当运行时」：一次算子一回 Python 调用与分派的 CPU 开销、allocator 与反复变形状不友好（43 题）、形状变化让编译产物重建（53 题），三笔开销自己用 profiler 的 CPU 侧 aten:: 时间量，量不准就报口径。

45. `loss.backward()` 一次调用依次发生什么？`no_grad`、`inference_mode`、`eval()` 各管哪一段？
   【要点】路径：前向把 GradFn 节点串成图，`backward` 从 loss 起按拓扑序取就绪节点，每个节点由上游梯度算出对输入的梯度，累加进叶子 tensor 的 `.grad`，非叶子节点的中间梯度算完就释放。要第二阶梯度传 `create_graph=True`，`retain_graph=True` 只是别把图拆掉。累加语义解释两件事：`.grad` 要自己清零，以及 15 题 `no_sync()` 为什么只在最后一个 micro-step 同步。三个开关分开：`model.eval()` 只切 Dropout 与 BatchNorm 的行为，梯度照记；`torch.no_grad()` 关图构建，仍做版本检查；`torch.inference_mode()` 再多关掉版本计数，产出的 tensor 不能再喂进 autograd。验证循环包 `no_grad`，纯推理包 `inference_mode`。追问两条：第二次 backward 报错是因为图已释放、`.grad` 越攒越大是因为累加，同一条原因的两面。

46. 训练一步的显存由哪几块组成？打印哪四个数？
   【要点】五块清单：权重、梯度、优化器状态、激活、临时缓冲与保留差。静态按 6 题的 16 字节/参数（bf16 混合精度加 Adam：权重 2 + 梯度 2 + fp32 主权重 4 + 一阶矩 4 + 二阶矩 4），激活按 `b·s·h` 的线性项加 attention 的 `b·h·s²` 项，用 FA 就没有 materialize 的 s² 中间量（17 题）。四个打印点：`memory_allocated`、`memory_reserved`、`max_memory_allocated`、`max_memory_reserved`（43 题），再加一行 `sum(p.numel() for g in opt.param_groups for p in g)` 把参数量坐实，8 题的实测流程就是这两句。追问「batch 翻倍会不会 OOM」：先把激活那项翻倍算出新峰值，再对 reserved 报余量，报完再动手跑。这个顺序跟 14 章给验证定门禁是同一个纪律。

47. batch size、序列长度、层数三根轴各把哪块资源顶起来？
   【要点】一根轴一句话：层数顶权重与优化器状态，线性；层数也顶反向要保留的激活，线性；序列长度同时顶 attention 的二次项和 KV 的线性项；batch 只顶线性项。手算流程四步：静态（参数量 × 每参数字节，6 题）→ 激活（`L·b·s·h` × 每元素字节 × 系数，34 这个系数按 Megatron 的说法标 UNVERIFIED，自己按保留了几个中间量推）→ KV（推理才有，20 题的式子）→ 减 CUDA context 与碎片余量。同一套式子在训练里回答「能不能跑」，在推理里回答「能开多大并发」：每 token KV 字节 × 上下文长度 × 并发数 ÷ 池大小（06 章）。追问「显存不够，减 batch 还是减序列」：减 batch 省线性项、伤吞吐；减序列省二次项、伤任务本身；两项都动不了静态那三块，静态放不下要 TP 或 FSDP（11、12、16 题）。

48. 混合精度一次 step 的完整链路，`GradScaler` 的四步各挡什么？
   【要点】顺序：`scale` 把 loss 乘上系数放大梯度、避开 fp16 下溢；`backward` 走放大后的梯度；`unscale_(optimizer)` 把梯度除回来，这一步必须在 `step` 前，否则主权重更新被 scale 污染，梯度范数类操作也全错；`step` 内部先扫 inf/nan，发现就不更新权重；`update` 在跳过时把系数减半、连续若干步干净才乘回去。bf16 免掉整套的原因按 32 题那句：指数位 8 bit、动态范围与 fp32 同级，下溢不发生在梯度上。哪些算子留在 fp32 报类别（归约、softmax、loss 这类对精度敏感的），清单名按 `torch.autocast` 文档核，开口先报你查的版本。追问两条：`autocast` 不管权重的存储 dtype，只标计算 dtype，权重是 fp32 还是 bf16 由模型自己定；`GradScaler` 只对有可学习参数的分支有意义，冻结参数时 found_inf 检查照常跑、不更新。

49. 拿到一个训练脚本，五分钟内怎么判断它用的是哪种并行？
   【要点】四个读点依次：看 import 与包装，DDP 是 ` DistributedDataParallel(model)`，FSDP2 是 `fully_shard` 逐层调，老 FSDP1 是 `FullySharded` 包整模型（11 题）；看 `DeviceMesh` 的维度，一维是纯 FSDP，二维是 HSDP，再看 `init_process_group` 的 backend 是 nccl 还是 gloo，gloo 说明跑在 CPU 上，数字口径要跟着改（20 题）；看配置文件字段，Megatron 是 `--tensor-model-parallel-size`、`--pipeline-model-parallel-size`、`--context-parallel-size`、`--expert-model-parallel-size`，torchtitan 是并行配置块，DeepSpeed 是 `zero_optimization.stage` 加 offload 段（16 题）；跑一步 profiler 数集合通信 kernel，反向里出现 all-reduce 是 DDP，出现 all-gather 加 reduce-scatter 是 FSDP 或 ZeRO-3，MoE 层有 all-to-all，气泡看时间轴上 rank 之间的空档（15 题）。收尾一句判据：静态账按 6 题算，单卡放得下权重加优化器状态就用 DDP，放不下再谈通信量那 1.5 倍（11 题）。

50. KV Cache 和 continuous batching 为什么影响显存？跟训练的显存账差在哪？
   【要点】先把式子摆出来：每 token 每层 2 × `num_key_value_heads` × `head_dim` × dtype 字节（20 题），LLaMA-2-7B fp16 是 80 KB/token，GQA 省的就是头的这一项（5 题），MLA 换掉整个形状（22 题）。训练阶段没有 KV Cache：前向算出来的 K、V 留在激活里，反向要用就重算，序列长度由 batch 摊平；推理阶段每序列每步只新增一行查询，之前所有 token 的 K、V 必须留着，decode 的带宽账就是这么来的（37 题）。continuous batching 让不同请求在不同 token 位置进出，池的占用变成动态量：能塞多少并发由池大小除以每条序列的字节数决定，池被瓜分完就触发抢占与重计算（06 章 18 题）。三条追问各给一条：为什么 32k 上下文比 4k 贵得比线性多——KV 线性涨，attention 中间量和重算代价跟着序列长度平方走（21 题）；为什么显存还剩 20% 却报 no available blocks——池按 block 分页，碎片加预留加在途序列占位；为什么 `--gpu-memory-utilization` 调高会提高并发上限同时抬高抢占频率（S8 的取舍，43 题那两个口径是同一笔账的两端）。


51. 那句 out of memory 落在哪一块？报错里的三个数分别读法是什么？
   【要点】先看报错行的三个数：已经分配多少、试图再分配多少、设备总容量多少（PyTorch 的 CUDA OOM 文案带这三项）。判定流程：分配请求落在权重上还是 KV 池上还是激活上，训练侧权重与优化器状态是静态项（6 题），推理侧池子在启动时就按 `--gpu-memory-utilization` 圈定、运行期申请失败说明池子见底、报的是 no free blocks 这一类而不是 CUDA OOM（vLLM 的 scheduler 侧信号，06 章 18 题）。三种解法各对应一笔账：降并发或降上下文（改 KV 那项，20 题的式子）、降 batch 或开激活重计算（改激活那项，47 题）、上 TP 或 FSDP（改静态那项，11、12 题）。追问「为什么显存还剩 20% 就跑不动了」：那 20% 是 reserved 与 allocated 的差加碎片（43 题），池子按 block 分页、剩下的字节凑不出一个整块；这条和 43 题那句「实测 block 数比纸面值小」是同一笔账的两端。S8 的 51 题在 GPU 上测的就是这三块各占多少，那笔差额要和 8 题的静态账对上。

### CUDA 到接口层（52–53）

52. CUDA Graph 降的是哪一段延迟？为什么它对动态 shape 不友好？
   【要点】省的两段：CPU 侧每次 kernel launch 的提交成本，GPU 前端对串行的发射调度。一次 decode step 发几十个 kernel，launch 成本与 kernel 本身执行时间同量级，所以时间轴上有空隙——profiler 里看 GPU 区间不是满的、利用率不满但没在等数据（41 题三种口径一起看），这个空隙才是 Graph 的目标。判据先算再动手：空隙占比乘 step 时间，跟预计收益比，低于几个百分点不折腾。为什么动态 shape 受限：捕获时把每个节点的 kernel 参数和显存地址写进 graph 里固化，输入地址或形状一变就要重新捕获，而重新捕获的成本高于省下的那点 launch——vLLM 的 decode 是动态 batch，这是它把捕获做成按 batch size 分桶的原因（46 题）。捕获的三条限制报得出：不能在图里做同步等待与主机回调，地址不能随便换（写进固定 buffer 或复用 memory pool），变长循环要拆成静态节点序列（26 题）。torch.compile 的 `mode="reduce-overhead"` 走的就是 Inductor 生成 Triton kernel 再用 CUDA Graph 打包提交（23 题），graph break 一处就退化成一段 eager，反复换形状会反复重建产物。

53. 在 vLLM 与 TensorRT 里认出 CUDA 这一层，到哪一步就停？
   【要点】三件事要认得出：profiler 与 nsys 的 kernel 名（`void flash::...` 这类 attention kernel、`cutlass::` 前缀的 GEMM、`ncclDevKernel_AllReduce_...`、TensorRT 自己插件的命名），launch 之间的空隙归 CPU 提交、归通信还是归依赖（41、52 题），H2D 与 D2H 的 memcpy 在流上有没有跟计算重叠（25 题）。两条路的分工一句分清：TensorRT 在构建期把整张静态图交给 kernel 选择与融合，动态 shape 靠 optimization profile 声明最小、最优、最大三档，profile 数量涨则 engine 体积与构建时间一起涨（字段名按 TRT 文档核）；vLLM 在运行期决定哪些请求进这一批、KV 分页怎么分配，kernel 是随包发布的实现加可选的 flash-attn、cutlass 后端（归属随版本变，标 UNVERIFIED，答「机制归哪层」比答「谁家有」稳）。停在接口层：说得出这一层省的是哪段开销、赔的是什么（46、52 题），说不出 mma 的 tile 形状与 PTX 细节就明说没写过 kernel（33 题），TC 那层的深度按 39 题。
