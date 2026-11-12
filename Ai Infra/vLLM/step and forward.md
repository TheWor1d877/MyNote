vLLM 的引擎会不断循环调用 `step()`，每一次 `step()` 都包含三个明确的阶段，可以看作是心跳的一次搏动：

1. **Schedule（调度）**：调度器决定**这一次 `step()` 要处理哪些请求**。它会从“等待中”和“运行中”的队列里挑选出一批请求，组合成一个本次要执行的 batch。这一步完全在 CPU 上完成，不涉及 GPU 计算。
    
2. **Forward（前向）**：**这就是你关心的“一次 forward”**。Model Runner 会把上一步调度选出的那批请求，拼成一个大的 batch，在 GPU 上跑一次模型，计算出所有 token 的 logits。**一次 `step()` 里，通常只会有一次 forward**，其处理的 token 总数就是我们在之前讨论中提到的 `num_tokens`。
    
3. **Postprocess（后处理）**：根据 forward 算出的 logits，进行采样（选择下一个 token），然后把新生成的 token 追加到各个请求上，并检查是否有请求已经完成（比如遇到 EOS）。完成的请求会被清理并返回结果。