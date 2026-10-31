# ML 系统与 GPU 基础（分级版）

按重要性分三档：T1 必须能脱稿讲满 3–5 分钟；T2 能讲清 90 秒；T3 为扩展与速查。★ 仅表示原题库标记，已不完全等同 T1。新增题带【要点】，其中的外部项目版本、API 名与论文编号按 2026-10-31 核实；开口答题先报版本，报不准就说「量级如此，会后核实」。

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

11. DDP 与 FSDP 各自把什么切了、每一步跑哪些集合通信、通信量差多少？面试官问「你这个规模为什么用 FSDP 不用 DDP」时，答案要落在这三行上。
   【要点】DDP：参数、梯度、优化器状态每卡全量，梯度按桶做 all-reduce（`bucket_cap_mb` 默认 25 MB）。FSDP2（`fully_shard`，基于 DTensor 做按参数 dim-0 切分；`DeviceMesh` 1D 即纯 FSDP，2D 是 HSDP）：前向 all-gather、反向 all-gather、梯度 reduce-scatter，通信量约为 DDP 的 1.5 倍，换来三项状态的显存分摊。`reshard_after_forward` 默认 True，改成 False 或整数（ZeRO++ 风格部分重切）就是在显存与带宽之间挪位置；混合精度走 `MixedPrecisionPolicy(param_dtype, reduce_dtype, output_dtype)`，CPU offload 走 `OffloadPolicy`。选型的句子要具体：单卡放得下模型与优化器状态时用 DDP，放不下多少才付得起那 1.5 倍通信。出处：PyTorch `distributed.fsdp.fully_shard` 文档、DDP notes。

12. ZeRO 三个阶段各切什么？ZeRO-Offload 与 ZeRO-Infinity 分别把东西挪到哪一级存储，挪过去的代价进哪段时间？
   【要点】Stage 1 切优化器状态，Stage 2 加梯度，Stage 3 加参数（arXiv:1910.02054）。Offload 把优化器状态与梯度放 CPU（`offload_optimizer.device="cpu"`），Infinity 再把参数放 CPU 或 NVMe（`offload_param.device="cpu"|"nvme"`）。代价按 15 章的带宽表算：CPU 与显存之间只有 PCIe 4.0 x16 有效 ~25 GB/s，NVMe 顺序读 3–7 GB/s，这两跳都直接进 step 时间。DeepSpeed 在 PyPI 的最新版本 0.19.9（2026-10-29），项目已捐给 Linux Foundation（2025-01）；PyTorch 官方文档推荐 FSDP2，FSDP1 的弃用声明没找到（UNVERIFIED，别把博客说法当官方口径）。出处：DeepSpeed ZeRO tutorial。

13. ring AllReduce 的通信量为什么几乎与卡数无关？把式子写出来，再说清它省的是什么、赔的是什么。
   【要点】每节点总通信量 2·(N−1)/N·S（S 为数据量），N 变大时趋于 2S，与 N 无关，这就是 bandwidth-optimal 的由来；赔的是步数 2(N−1)，延迟项随卡数线性增长（Sergeev & Del Bello，Horovod，arXiv:1802.05799）。两条推论要主动给：小尺寸高频同步时先炸的是延迟，所以要做梯度桶化；跨机段要减步数，机内走 NVLink、机间走 RDMA 的分层 all-reduce。与本章 AllReduce 语义题的分工：那题讲数学，这题讲账。

14. 流水线并行的气泡公式是什么？加 stage、加 microbatch、换 interleaved schedule，三件事各自改哪个变量？
   【要点】气泡占比 (p−1)/m（p 为 stage 数、m 为 microbatch 数；换算成总时间占比是 (p−1)/(m+p−1)），interleaved/VPP 变成 (p−1)/(v·m)（Narayanan et al., arXiv:2104.04473）。1F1B 省的是激活显存，它改的是显存曲线，气泡照旧。判据给数值：m 至少 4p 以上，否则加卡带来的并行度先被气泡吃掉。与他项目的接口：气泡期间数据可以提前取，气泡期间的梯度同步没法省，所以缓存层的收益在高 PP 度下会被放大。出处：15 章机时账。

15. 一个 step 里数据加载、前反向、通信怎么重叠？重叠失败在指标上长什么样？
   【要点】三条重叠路径：DataLoader 多 worker 预取把解码与增广藏在 GPU 后面，pinned memory + `cudaMemcpyAsync` 让 H2D 与计算并行；梯度累积用 `model.no_sync()` 只在最后一个 micro-step 同步，省掉 N−1 次 all-reduce；反向逐层触发 reduce-scatter/all-gather，与本层计算流水。重叠失败的可观测形态：GPU 利用率均值不掉、方差变大（个别 rank 在等），或全体 step 时间同步抬高且数据队列见底（12 章 6 题、31 题）。答完补一句他的实测口径：他测的是数据等待占比，通信重叠度是从框架侧指标读出的，两者不能混报。出处：05 章、12 章。

16. TP 之外还有 SP/CP/EP，它们各解决什么，通信点分别落在哪？
   【要点】Megatron 的 TP 在每个 transformer 层做 2 次前向 all-reduce 与 2 次反向（arXiv:1909.08053），量随层数与 TP 度线性涨，所以 TP 一般压在机内 NVLink 域。sequence parallel（`--sequence-parallel`，要求 TP>1）把 all-reduce 拆成 reduce-scatter + all-gather，激活显存跟着分摊（arXiv:2205.05198）。context parallel（`--context-parallel-size`、`--cp-comm-type p2p|a2a|a2a+p2p`）切序列长度给长上下文，通信是 K/V 的环形交换。expert parallel 每个 MoE 层 2 次 all-to-all（`--expert-model-parallel-size`）。现状：Megatron-Core 0.20.0（2026-10-29）引入 Megatron-FSDP v2 与 MXFP8 通信；torchtitan v0.4.0（2026-10-28）是 PyTorch 官方参考栈，组合 FSDP2+TP+PP+CP+EP+DCP。

## T3 · 扩展与速查

17. ★ 解释 CUDA 的 global memory 合并访问，什么样的访问模式最差。
18. ★ 什么是 CUDA Stream 与 Event？如何用它们实现拷贝与计算重叠。
19. ★ 什么是 CUDA Graph？它如何降低 kernel launch 开销，捕获时有哪些限制。
20. 解释通信与计算重叠的常见实现方式。
21. 什么是 MoE？它的路由机制对系统负载均衡提出什么要求。

22. 大规模训练的故障率怎么进你的容量账？goodput 与 MFU 分别是什么口径？
   【要点】goodput 是扣掉故障、恢复、重算浪费之后的有效训练进度每单位时间（Google Cloud 的定义文《Goodput metric as measure of ML productivity》）；MFU 是实测吞吐相对按峰值 FLOPS 算出的理论吞吐的比值（PaLM, arXiv:2204.02311）。可直接引用的实测条件：Llama 3.1 405B 用 16,384 张 H100 跑 54 天，非预期中断 419 次，约每 3 小时一次，有效训练时间占比 >90%，GPU 相关故障占 58.7%、HBM3 约 17.2%，MFU 38–43%（arXiv:2407.21783）。用法：把「等数据」造成的 1% 好吞吐损失折算成机时钱（13 章的账），这是他把系统问题译成机时钱的固定句式。

23. 训练任务挂了怎么自动拉回来？NCCL 与弹性训练的当前参数名报一遍，并说清默认值造成的误诊。
   【要点】当前变量族是 `TORCH_NCCL_*`：`TORCH_NCCL_ASYNC_ERROR_HANDLING` 默认 3，`TORCH_NCCL_HEARTBEAT_TIMEOUT_SEC`、`TORCH_NCCL_ENABLE_MONITORING`、`TORCH_NCCL_MONITOR_INTERVAL_MSEC`、`TORCH_NCCL_DUMP_ON_TIMEOUT`、`TORCH_NCCL_TRACE_BUFFER_SIZE`（flight recorder 的开关与容量）；旧名 `NCCL_ASYNC_ERROR_HANDLING`、`NCCL_BLOCKING_WAIT` 已标弃用。torchrun 的重启次数是 `--max-restarts`，默认 0，参数表里没有 `--restarts`；另有 `--monitor-interval`、环境变量 `TORCHELASTIC_RESTART_COUNT`。误诊来源：默认配置下节点掉队后任务卡到超时退出，自动重启不会发生，表象是 NCCL 超时，实际原因是弹性没开。出处：PyTorch ProcessGroupNCCL 环境变量文档、elastic run 文档。

24. checkpoint 的保存与恢复怎么算账？异步保存解决哪一段，代价落在哪？
   【要点】`torch.distributed.checkpoint`（DCP）提供 `dcp.save`、`dcp.async_save`（返回 Future）、`dcp.load`，配合 `Stateful` 协议、`AsyncStager`/`BlockingAsyncStager`，状态字典走 `get_state_dict`/`set_state_dict` 与 `StateDictOptions(full_state_dict, cpu_offload, broadcast_from_rank0)`。异步保存把写盘移出 step 关键路径，代价是暂存副本占主机内存、且写盘未完成时新副本要覆盖旧副本，这里要么排队要么加缓冲。恢复侧的账：权重从存储进显存的分钟级开销（15 章），它决定 checkpoint 间隔的下限和最优值。可延伸的一句：间隔太短会挤压数据管道的带宽，与他缓存层争资源。出处：PyTorch DCP 文档。

25. bf16、fp16、FP8 三档精度各对系统提什么要求？
   【要点】fp16 指数位 5 bit，梯度易下溢，需要 loss scaling（`torch.amp.GradScaler('cuda')`，配 `torch.autocast('cuda', dtype=...)`）；bf16 指数位 8 bit、动态范围与 fp32 同级，免掉缩放，硬件要求 Ampere 之后。FP8 训练：Hopper（H100/H200）走 Transformer Engine 或 Megatron `--fp8-format`；MXFP8（1 个 32 元素块共享一个 scale）只在 Blackwell（B200/GB200）上有硬件路径，Megatron-Core 0.20 与 torchtitan（`float8.py`、`mxfp8/`、torchao `Float8CurrentScaling`）支持；DeepSeek-V3 用 FP8 block-scaled 与 UE8M0 scale。答题落回系统层：降精度省的是显存与带宽，谁受益取决于负载是 memory-bandwidth-bound 还是 compute-bound（06 章 2 题）。

26. 面试官问「你读过 Megatron 或 vLLM 的源码吗」，怎么答得可验证？
   【要点】2026 年的 JD 把框架源码阅读写成显性要求（懂车帝大模型 Infra 方向列 SGLang/vLLM/Megatron/DeepSpeed）。可验证的答法只有三段：入口在哪个文件、一条 step 依次经过哪几个函数、我在哪一行下过断言或改过行为。报仓库名和「看过」没有信息量。他没读过训练框架就明说读的是推理与数据侧，然后把本章前几题按机制答完整。声称读过又被追问细节，代价比承认没读过更大（16 章第四节的过程痕迹四层）。
