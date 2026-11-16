batch指的是一次 GPU 前向传播里同时处理多个请求。

continuous batching 的意思是：这个 batch 的成员每一步都可以变，不需要等整批请求结束

## Prefill 实例的 continuous batching
#### 组批判断：一个新 prefill 请求进running队列的时机
- 第一层，max_num_seqs。 prefill 实例同时处理的请求数上限。

- 第二层，token 预算余量。 本步 max_num_batched_tokens 减去已分配给 running prefill 请求的量，剩下的够不够这个新请求至少算一部分 prompt。不够就继续排队。

- 第三层，block 可分配性。 allocate_slots 检查本步要写的 token 需要多少个 block，block pool 里有没有。prefill 算完后 KV 是否本地保留，取决于 connector 实现，但计算这一步需要 block 来写。

#### prefill 实例处理请求

Prefill 实例的 continuous batching 和普通实例共用同一个调度器主循环：请求从 waiting 入队，调度器按 token 预算和 block 可用性决定它哪一步进 running，进去之后每步推进 num_computed_tokens。出队条件是 num_computed_tokens 追平 num_tokens_with_spec。

在 PD 分离的 prefill 实例上，max_tokens=1 让 num_tokens_with_spec 只比 prompt 长度多一，所以请求的出队点实际上是“prefill 全部算完 + 采样出首 token”，之后 KV 交给 connector 导出。调度器不感知 PD 分离，它只认 token 计数。

## Decode 实例的 continuous batching
#### 组批判断：一个新 decode 请求进running队列的时机
- 第一层，max_num_seqs。 decode 实例同时维护的序列数上限。这通常是主要瓶颈，因为 decode 请求的 token 消耗极低，序列数容易堆高。

- 第二层，KV block 容量。 从 prefill 传过来的 KV 需要落到 decode 侧的 block 里。allocate_slots 检查空闲 block 够不够接纳这个请求的 prompt KV。不够就排队或触发抢占。

- 第三层，token budget。 理论上存在，但 decode 每步 1 token/请求，max_num_batched_tokens 在正常并发下很难成为限制。只有在极端高并发时才需要考虑。

#### decode 实例处理请求

Decode 实例的 continuous batching 和普通实例共用同一个调度器主循环：请求从 waiting 入队，调度器按 token 预算和 block 可用性决定它哪一步进 running，进去之后每步推进 num_compute
prefill 实例处理请求d_tokens。出队条件是 num_computed_tokens 追平 num_tokens_with_spec，且生成逻辑判定请求结束（EOS 或达到 max_tokens）。

在 PD 分离的 decode 实例上，请求入队时 num_computed_tokens 已经等于 prompt 长度（KV Cache 由 prefill 实例算好并传输过来），所以它没有“追 prompt”这个阶段，直接进入逐 token 的 decode 推进。
每步调度只给每个 running 请求分配 1 个 token（或投机解码下的几个 token），num_tokens_with_spec 随生成逐步增长。
出队点就是生成逻辑判定结束的那一刻，之后 KV 释放。调度器不感知 PD 分离，它只认 token 计数；但 decode 实例的 waiting 队列多了一层前置检查——KV 是否已就绪，这个信息通过 connector 的 scheduler 侧接口进入调度决策。
