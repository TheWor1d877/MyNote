一个 kernel 慢，到底该怪带宽还是怪算力？
Roofline是一个判断资源约束的模型。它告诉你：在给定硬件上，性能上限由哪块物理资源先封顶。

## 算术强度
算术强度 Arithmetic Intensity，AI
```text
AI = FLOPs / Bytes
```
每搬运1字节数据,能执行多少次浮点运算


AI 低：搬了很多数据，但没算几下 → 带宽压力大。
AI 高：同样一份数据被反复计算 → 算力压力更大。

## Roofline图
水平线：峰值算力。再高的 AI 也超不过它。
斜线：AI × 带宽。AI 越低，性能越被带宽限制。
脊点 Ridge Point：两条线交点，AI* = 峰值算力 / 峰值带宽。
以 H100 为例，FP16 稠密 Tensor Core 算力约 990 TFLOPS，HBM 带宽约 3.35 TB/s，脊点大约在 295 FLOP/Byte 附近。

判断规则：
AI < 脊点：Memory-bound，性能被带宽限制。
AI > 脊点：Compute-bound，性能被算力限制。
离屋顶线越远：说明实现效率越低，还有优化空间。

