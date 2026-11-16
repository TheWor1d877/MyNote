推理引擎通常会重新写一套自己的算子
推理引擎在加载模型时，会把模型代码里原本调用的普通算子，替换成自己的算子
比如，推理引擎重写的 `Linear` 会包含张量并行（`Tensor Parallelism`）的切分逻辑；重写的 `Attention` 会包含读取 `block_table` 和 `slot_mapping` 的逻辑
## attention.py(注意力机制)
#### store_kvcache_kernel
这个 `kernel` 遍历当前批次的所有 `Token`，根据 `slot_mapping` 找到它在物理显存池里的"槽位（`slot`）"，然后把刚算出来的 `Key` 和 `Value` 填进去。

#### forward函数
```python
    def forward(self, q: torch.Tensor, k: torch.Tensor, v: torch.Tensor):
        context = get_context()
        k_cache, v_cache = self.k_cache, self.v_cache 
        if k_cache.numel() and v_cache.numel():
            store_kvcache(k, v, k_cache, v_cache,  context.slot_mapping) # 将kv写入显存池子
        if context.is_prefill:
            if context.block_tables is not None:    # prefix cache
                k, v = k_cache, v_cache
            o = flash_attn_varlen_func(q, k, v,max_seqlen_q=context.max_seqlen_q,cu_seqlens_q=context.cu_seqlens_q,max_seqlen_k=context.max_seqlen_k,cu_seqlens_k=context.cu_seqlens_k,softmax_scale=self.scale, causal=True,block_table=context.block_tables)
        else:    # decode
            o = flash_attn_with_kvcache(q.unsqueeze(1), k_cache, v_cache,cache_seqlens=context.context_lens,block_table=context.block_tables,softmax_scale=self.scale, causal=True)
        return o

```
###### prefill阶段
正常的 `Prefill` 阶段，`q`, `k`, `v` 都在手里，直接用 `flash_attn_varlen_func` 算就行

特殊点（`Prefix Cache`）：如果命中了前缀缓存（`block_tables` 不为空），那当前 `prompt` 的 `k`, `v` 是不完整的（只有新算的那部分）。所以代码直接把 `k`, `v` 替换成了整个物理显存池（`k_cache`, `v_cache`），配合 `block_table` 传给底层去读历史缓存

###### decode阶段
`decode` 时，当前序列只有一个 `Q`（所以 `q.unsqueeze(1)`）。
传入整个显存池`k_cache`, `v_cache`，外加 `block_table`，而不是传入完整的 `K`、`V` 

`flash_attn_with_kvcache` 是 `FlashAttention` 官方专门为推理优化的函数。它内部会根据 `block_table`（逻辑块到物理块的映射）和 `context_lens`（当前序列总长），自动去物理显存池里把历史的 `K`、`V` 找出来，和当前的 `Q` 进行计算



## linear.py(线性层/矩阵乘法)
`TP`: 张量并行
一个基础类 + 四种模式：
#### 基础类`LinearBase`
`input_size` 与 `output_size` 记录的是`TP`后属于这张卡的部分
`tp_dim` 记录这个权重矩阵在哪一个维度被切开
然后加载空的权重占位符号
`torch.empty`：这里只是占个位置，分配了对应形状的显存，里面的数值是未初始化的垃圾值。

#### 完全复制 `ReplicatedLinear`
不切分，直接加载
`weight_loader` 直接 `param.data.copy_(loaded_weight)`，整块搬运
前向方式：`F.linear(x, self.weight, self.bias)`
```text
权重矩阵 WW 的形状是(4, 4)
我们用 w1w1​ 到 w16w16​ 表示矩阵里的16个数字：
      x1   x2   x3   x4
    +----+----+----+----+
y1  | w1 | w2 | w3 | w4 |
y2  | w5 | w6 | w7 | w8 |
y3  | w9 | w10| w11| w12|
y4  | w13| w14| w15| w16|
    +----+----+----+----+
```

#### 列并行 `ColumnParallelLinear`（按输出维度切）
按行（`tp_dim=0`）切分输出维度。例如输出是 1024，两张卡各切 512。因为每张卡都有完整的输入X，所以各自算出来的局部输出拼起来就是完整输出
从全量权重里，用 `narrow` 选择出 `[tp_rank * shard_size : (tp_rank+1) * shard_size]` 这一块。
前向方式: `F.linear(x, self.weight, self.bias)`
```text
卡0 拿到的 W：
[ w1, w2, w3, w4 ]
[ w5, w6, w7, w8 ]
卡1 拿到的 W：
[ w9, w10, w11, w12 ]
[ w13,w14, w15, w16 ]
```
两张卡都拿完整的`[x1,x2,x3,x4]`

###### 合并列并行
背景： 在 `Llama/Qwen` 的 `MLP` 层里，通常有两个并行的线性层（门控 `gate_proj` 和 上投影 `up_proj`），它们读取同一个输入X。与其分开算两次，不如把它们拼成一个大矩阵一次算完

把两个独立的权重矩阵（比如 `A` 和 `B`）横向拼接在一起，然后当成一个大的列并行处理。
假设 `A` 和 `B` 都是 (4,4) 的矩阵：
```text
A 矩阵               B 矩阵
[ a1, a2, a3, a4 ]   [ b1, b2, b3, b4 ]
[ a5, a6, a7, a8 ]   [ b5, b6, b7, b8 ]
[ a9, a10,a11,a12]   [ b9, b10,b11,b12]
[ a13,a14,a15,a16]   [ b13,b14,b15,b16]
```
拼接成一个(4,8)的大矩阵

#### 行并行 `RowParallelLinear`（按输入维度切）
按列（`tp_dim=1`）切分输入维度。例如输入是 1024，两张卡各拿 512 的输入特征去算。
`narrow` 抠出输入维度的分片。遇到偏置（`bias`），每张卡都完整保留（因为偏置不涉及切分）。
```text
卡0 拿到的 W     卡1 拿到的 W
[ w1, w2 ]      [ w3, w4 ]
[ w5, w6 ]      [ w7, w8 ]
[ w9, w10]      [ w11,w12]
[ w13,w14]      [ w15,w16]
```
输入 X：卡0 只拿 `[x1,x2]`；卡1 只拿`[x3,x4]`
将两张卡的部分结果按元素相加：y1=y0_1+y1_1y1​=y0_1​+y1_1​。这样才拼出完整的 y1,y2,y3,y4y1​,y2​,y3​,y4​。

行并行需要通信开销： 等待所有的`y`都算完了进行`all_reduce`才行

####  特殊版本：`QKVParallelLinear`（GQA 专用）
策略：本质上和上面的 `Merged` 一模一样。  
假设有 `W_q`、`W_k`、`W_v`三个矩阵，物理上把它们横向拼成一个大矩阵 `[W_Q | W_K | W_V]`，然后按行切给两张卡。

（注：实际代码中，因为 `GQA` 的存在，`Q`、`K`、`V` 的维度大小可能不同，但切分原理完全一致：各自切各自的行头，再把切下来的部分拼到每张卡上）。

- 卡0：拿 `Q` 的前半部分头，`K` 的前半部分头，`V` 的前半部分头。
- 卡1：拿 `Q` 的后半部分头，`K` 的后半部分头，`V` 的后半部分头。
- 输入 X：两张卡都拿完整的 `[x1,x2,x3,x4]`。
###### GQA
`MHA`的三权重矩阵
```text
W_q (4行 × 4列)
         x1   x2   x3   x4
       +----+----+----+----+
 q头0  | a1 | a2 | a3 | a4 |
 q头1  | a5 | a6 | a7 | a8 |
 q头2  | a9 |a10 |a11 |a12 |
 q头3  |a13 |a14 |a15 |a16 |
       +----+----+----+----+

W_k (4行 × 4列)
         x1   x2   x3   x4
       +----+----+----+----+
 k头0  | b1 | b2 | b3 | b4 |
 k头1  | b5 | b6 | b7 | b8 |
 k头2  | b9 |b10 |b11 |b12 |
 k头3  |b13 |b14 |b15 |b16 |
       +----+----+----+----+

W_v (4行 × 4列)
         x1   x2   x3   x4
       +----+----+----+----+
 v头0  | c1 | c2 | c3 | c4 |
 v头1  | c5 | c6 | c7 | c8 |
 v头2  | c9 |c10 |c11 |c12 |
 v头3  |c13 |c14 |c15 |c16 |
       +----+----+----+----+
```

`GQA`的三权重矩阵
```text
W_q (4行 × 4列，不变)
         x1   x2   x3   x4
       +----+----+----+----+
 q头0  | a1 | a2 | a3 | a4 |
 q头1  | a5 | a6 | a7 | a8 |
 q头2  | a9 |a10 |a11 |a12 |
 q头3  |a13 |a14 |a15 |a16 |
       +----+----+----+----+

W_k (2行 × 4列，变少)
         x1   x2   x3   x4
       +----+----+----+----+
 k头0  | b1 | b2 | b3 | b4 |
 k头1  | b5 | b6 | b7 | b8 |
       +----+----+----+----+

W_v (2行 × 4列，变少)
         x1   x2   x3   x4
       +----+----+----+----+
 v头0  | c1 | c2 | c3 | c4 |
 v头1  | c5 | c6 | c7 | c8 |
       +----+----+----+----+
```

`Q` 头与 `K/V` 头的对应关系（多对一）
```text
 q头0 ──┐
        ├──> 都用 k头0 / v头0
 q头1 ──┘

 q头2 ──┐
        ├──> 都用 k头1 / v头1
 q头3 ──┘
```


## activation.py
激活函数，提供非线性变换

## layernorm.py
层归一化，通常实现的是 `RMSNorm`

## rotary_embedding.py
旋转位置编码 `RoPE`
`RoPE` 就是给每个词的 `Q`、`K` 向量按"它是第几个词"转一个对应的角度；两个词转过的角度差，天然就是它们的相对距离

## embed_head.py
一块是 `Embedding`，把输入的 `Token ID`（比如数字 1024）查表变成巨大的向量；另一块是 `LM Head`，把模型最后输出的隐藏向量，映射回词表大小（比如 15 万维），算出每个词的概率

## sampler.py
它接收 `embed_head` 吐出来的 `logits`，结合用户传进来的 `temperature`（温度）、`top_p` 等参数，用概率分布抽样出下一个 `Token ID`