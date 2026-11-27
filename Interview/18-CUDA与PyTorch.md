# CUDA 与 PyTorch

## 必会

1. CUDA 的 thread、warp、block、grid 各是什么？谁共享存储，谁之间可以同步？
2. 一个 warp 的线程进入不同 if 分支时，硬件怎么执行？代价怎么算？
3. warp 的一次 global memory 访问怎么合并成显存事务？最好和最差的访问模式各是什么，分别怎么修？
4. shared memory 适合缓存什么数据？什么条件下值得从显存搬进去，什么条件下不值得？
5. bank conflict 怎么发生？冲突度怎么算，padding 和其他修法分别改变什么？
6. block 之间为什么没有同步原语？确实需要跨 block 协调时怎么办？
7. 一次 kernel launch 从 CPU 调用到 GPU 执行经过哪些阶段？哪些是纯开销？
8. CUDA stream 的执行语义是什么？同一流和不同流上的操作分别怎么排序？
9. 怎么让 H2D 拷贝和计算重叠？stream 和 event 各承担什么角色，哪些写法会产生假依赖？
10. 异步拷贝为什么需要 pinned memory？pageable 内存走异步 API 会发生什么？
11. 用 event 给 kernel 计时和用 CPU 侧计时有什么区别？CPU 计时会多算或少算什么？
12. stream 同步和 device 同步分别阻塞谁？各自适合什么场景？
13. PyTorch 从上到下管哪几层？它和推理引擎的边界在哪，哪些工作不由 PyTorch 承担？
14. dispatcher 怎么把 add 分派到 CPU 或 CUDA 实现？自定义算子应该挂在哪一层？
15. loss.backward() 依次发生什么？梯度怎么累到叶子 tensor 上，为什么每次都要清零？
16. no_grad、inference_mode 和 eval() 分别管哪一段？验证循环和纯推理各包哪个？
17. retain_graph 和 create_graph 分别改什么？第二次 backward 报错是哪条原因？
18. PyTorch profiler 怎么区分 CPU 提交和 GPU 执行的瓶颈？torch profiler 和 nsys 各看什么？
19. torch.compile 从模型到可执行产物经过哪些步骤？graph break 由哪些写法触发，代价落在哪？
20. 算子融合省了什么？为什么计算量不变也能提速，哪些负载收益很小？
21. Triton 替代手写 CUDA 的哪一部分？它的抽象单位是什么？
22. allocated 和 reserved 分别是什么？差值从哪里来？
23. caching allocator 怎么复用显存？为什么释放的 tensor 没有立刻把显存还给驱动？
24. 显存碎片怎么产生？总字节够却分不出连续空间时，有哪些缓解办法？
25. CUDA OOM 报错里的三个数怎么读？怎么判断分配发生在权重、激活还是临时缓冲上？
26. nvidia-smi 显示还剩两成显存，为什么仍可能 OOM？
27. empty_cache 什么时候适合调用，为什么不放进服务路径？推理引擎为什么启动时要自建显存池？
28. 激活重计算用什么换什么？什么规模下划算？
29. fp16 混合精度里，GradScaler 的四步分别防什么？bf16 为什么通常不需要 loss scaling？
30. autocast 管什么、不管什么？权重存储 dtype 由谁定，哪些算子需要保留 fp32？

## 加分

1. CUDA Graph 省的是哪段开销？什么样的负载收益最大，捕获成本怎么和 launch 开销比较？
2. 图捕获固化了哪些内容？哪些操作不能入图，tensor 固定地址和捕获内存池怎么配合？
3. 模型形状变化时，compile 和 CUDA Graph 分别要付出什么重建成本？为什么常按 batch size 分桶？
4. 除了合并访问，向量化、对齐和只读缓存分别怎么改善访存？
5. bank conflict 的 padding 和 swizzle 有什么区别？swizzle 怎么错开 bank？
6. FlashAttention 为什么在 GEMM 已经做 tiling 的情况下仍有用？分块后 softmax 怎么处理？
7. GradScaler 遇到 inf/nan 后怎么处理这一步？缩放系数按什么规则恢复？
8. 时间轴上怎么判断拷贝和计算真的重叠？event 和 kernel 区间应该呈现什么位置关系？
9. 实测模型显存峰值时，打印哪几个数、按什么顺序测？参数量和峰值怎么对应？
10. batch size、序列长度和层数分别顶起哪块显存？显存不足时减 batch 和减序列各伤什么？
11. 在 vLLM profiler 时间轴上怎么区分 attention、GEMM 和通信 kernel？
12. TensorRT 怎么处理动态形状？形状档位变多后，成本涨在哪？
13. TensorRT 的构建期路线和 vLLM 的运行期路线分别适合什么负载？
14. 面试官问「你读过 vLLM 或 Megatron 源码吗」，怎么答到可验证的程度？没写过 kernel 时，CUDA 题答到哪一层？
