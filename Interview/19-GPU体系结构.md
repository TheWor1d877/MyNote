# GPU 体系结构

## 必会

1. 一个 SM 里面有哪些部件？warp scheduler 每个周期做什么？
2. warp 怎么被调度执行？显存访问需要等待几百个周期时，GPU 靠什么隐藏延迟？
3. occupancy 是什么？寄存器、shared memory 和 block 限制分别怎么约束驻留 warp？
4. occupancy 拉满但吞吐不涨，可能有哪些原因？指令级并行和 occupancy 有什么关系？
5. GPU 和 CPU 的核为什么采用不同设计？GPU 为什么能堆大量执行单元？
6. SIMT 和 SIMD 有什么区别？
7. 从寄存器到主机内存，GPU 存储分哪几级？各级由谁管理，带宽和延迟差多少？
8. shared memory 和 L1 为什么共用 SRAM？配置切分会怎样影响 kernel？
9. L2 的作用范围是什么？它和 shared memory 分别接住什么复用？
10. HBM 和 GDDR 有什么区别？为什么训练卡普遍采用 HBM？
11. 寄存器、shared memory、L2 和 HBM 的延迟各是什么量级？反复读取的数据应该放在哪一级？block 结束后 shared memory 的数据为什么不能跨 kernel 复用？
12. 算术强度怎么算？roofline 的两条边和拐点分别是什么？
13. 给定峰值算力和带宽，怎么判断 kernel 受算力还是带宽限制？怎么算理论上限？
14. batch=1 的 decode 为什么常被描述为 GEMV？它的算术强度怎么算，结果说明什么？
15. prefill 的大 GEMM 为什么通常受算力限制？batch 从 1 增加到 64，算术强度和拐点位置怎么变化？
16. 量化为什么在小 batch 时收益更大？它改变了算术强度算式中的哪一项？
17. Tensor Core 和 CUDA core 的分工是什么？GEMM 满足哪些条件才能走上 Tensor Core，怎么确认实际路径？
18. fp32 矩阵乘为什么走不了 Tensor Core？tf32 拿什么换什么？
19. hidden size 和 head_dim 为什么常取 128 或 64 的倍数？背后有哪些硬件约束？
20. FP8 的 per-tensor 和 block-wise scaling 有什么区别？FP4 更粗的 scale 粒度为什么仍能控制精度损失？
21. 给定每线程寄存器数和每 block 的 shared memory 用量，怎么算一个 SM 能驻留多少 warp？
22. wave 量化是什么？1000 个 block 跑在 108 个 SM 上，最后一波浪费多少？有哪些修法？
23. 寄存器 spill 把数据放到哪一级？限制每线程寄存器数和 launch bounds 分别改变什么？
24. MFU 偏低时，怎么从框架、kernel 到硬件逐段拆损耗？
25. nvidia-smi 的 GPU-Util 统计什么？它看不见哪些等待？
26. GPU-Util 显示 100% 但吞吐很差，可能有哪些情况？判断 GPU 是否真的在工作要看哪些指标？
27. SM 吞吐、DRAM 带宽利用率和 Tensor Core 利用率怎么联读，定位到具体资源瓶颈？
28. 采样窗口为什么会平均掉短 kernel 的空隙？decode 负载为什么不能只看均值，还要看方差和分位数？
29. NVLink、PCIe 和 NVSwitch 分别解决什么问题？哪些通信应该走哪条路径？
30. 张量并行为什么通常限制在机内？IB 200 Gb/s 怎么换算成可用带宽，跨机通信预算怎么算？
31. 怎么查 GPU 拓扑并判断 NVLink 域？PCIe P2P 需要什么条件，ACS 会怎样影响它？
32. MIG 和 MPS 分别切分或共享什么资源？各适合什么负载，隔离边界在哪里？
33. DCGM 导出到 Prometheus 时，开启 MIG 后利用率统计的是整卡还是切片？多进程共卡时性能怎么归因？
34. GPU 的 XID、ECC 错误和掉卡分别怎么发现？集群健康检查查什么，命中什么条件摘节点？
35. 时钟、功耗和温度怎么影响持续性能？同规格 GPU 在不同机箱里为什么可能跑出不同结果？
36. 测一个 kernel 的完整流程是什么？nsys 和 Nsight Compute 按什么顺序使用，分别看什么？

## 加分

1. 怎么实测一张卡的 HBM 带宽？实测值低于标称时，先查哪些因素？
2. 用 roofline 算出理论上限后，实测离上限很远，先查什么？
3. Hopper 的 warp group 级矩阵乘、TMA 和 warp specialization 分别改变什么？
4. Hopper 为什么能支持 FP8 训练？MXFP8 每 32 个元素共享一个 scale，对精度和带宽意味着什么？
5. L2 access policy window 怎么用？哪些访问模式下 L2 命中率也救不了性能？
6. persistent kernel 怎么减少 wave 尾部损失？多流互补和调 block 大小分别适合什么情况？
7. NCCL 为什么为大消息和小消息选择不同算法？ring、tree 和分层 all-reduce 各优化什么？
8. NVLS 和 SHARP 分别把哪段归约移到哪里？网内归约怎么改变通信量？
9. 容器没有挂载 InfiniBand 设备时，NCCL 会怎么降级？怎么发现这种静默退化？
10. MPS 和 MIG 下，怎么把利用率归因到具体进程或切片？
11. 一张卡要摸到标称 TFLOPS，需要满足哪些条件？
12. Nsight Compute 的哪些 section 分别回答访存、占用和指令问题？
13. 显存带宽受限时，系统层有哪些杠杆？按每 token 需要读取的字节数怎么排序？
14. 给新负载估算硬件预算时，哪些标称值可用，哪些必须实测？片上 SRAM 芯片适合哪类负载？
