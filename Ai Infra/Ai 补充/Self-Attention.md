## Q、K、V 的生成与计算逻辑
Query（查询向量 Q）：代表"当前 Token 想找什么信息"，是主动的查询方。
Key（键向量 K）：代表"其他 Token 能提供什么信息"，是被动的被检索方。
Value（值向量 V）：代表"其他 Token 实际携带的内容"，是最终被加权聚合的信息


输入序列X分别乘三个可以学习的权重矩阵得到QKV:
$Q = XW_Q,    K = XW_K,    V = XW_V$
这三个矩阵的作用是将同一个输入投影到三个不同的语义子空间，使每个 Token 能以不同的"身份"参与交互——同一个词在作为查询方和作为被检索方时，呈现的特征可以完全不同。

$$
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V
$$
![[Attachments/Pasted image 20260929102903.png]]


每一层 Attention 都有自己的W所以每一层算出来的 Q、K、V 都是独立的
下一个Token的生成使用的KV可以使用以前的KV做基础，从KV Cache中得到

每一个Q都用不到历史Q

## Self-Attention的优化方向
Self-Attention 的计算复杂度为 $O(n^2d)$
与序列长度呈平方关系。在长上下文场景（如 128K Token）下，Attention 层的 FLOPs 占整个前向传播的 60% 以上。
- FlashAttention：通过分块计算和重计算技术，将 HBM 访问次数从 O(n^2) 降至 O(n) 
- GQA（分组查询注意力）：将多个 Query 头共享同一组 Key/Value，在保持模型效果的同时将 KV Cache 显存占用降低 4~8 倍；
多头注意力中每个头使用相同的KV
- 稀疏 Attention：仅计算局部窗口或特定模式的注意力，将复杂度从 O(n^2) 降至 O(nlogn) 或 O(n) 。


