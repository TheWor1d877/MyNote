## BlockTable(v0)
每个Sequence自己有一张 block table，这张表记录逻辑块号与物理块的对应关系
```text
BlockTable（每个 SequenceGroup 一个）
┌─────────────────────────────────────┐
│ req_A: [ 7,  3, 12,  ... ]          │  逻辑块 0 → 物理 7
│ req_B: [ 2,  9,  1,  ... ]          │  逻辑块 1 → 物理 3
│ req_C: [ 7,  3, 15,  ... ]          │  逻辑块 2 → 物理 12
└─────────────────────────────────────┘
```
同一个 SequenceGroup 内，可以在未满块内做写时复制（COW）。
这意味着 block table 的条目在请求生命周期内可能被改写，不是 append-only 的

当请求结束，这个block_table跟着sequence一起消失，但是物理块可以活下来(除非没有开prefix cache)
表本身不管理块的生命周期，每个块自己维护引用计数，到0就可以删除了

## BlockPool + Index(v1)

将 BlockTable 的结构拆成两层：

```text
第一层（CPU 侧，控制面，权威）：
KVCacheManager
├── req_to_blocks: {req_A: [7,3,12], req_B: [2,9,1]}
└── BlockPool（全局块池，ref_cnt / hash，不感知请求）
└── OffloadingManager + CPU pinned 张量池 (只有开启offloading才存在)

第二层（GPU 侧，数据面 + 索引副本）：
block_table[max_reqs, max_blocks]   ← 每 step 把当前 batch 的映射同步给 GPU
gpu_kv_cache                        ← KV 数值本体（见层2末尾）
```

这里的"两层"是控制面 / 数据面的划分，不是 KV 数据的两级缓存。 KV 字节只有一份，在显存里；开了 offload 才会有 CPU 侧副本。
CPU与GPU之间是生产者与消费者的关系，Scheduler 的 BlockPool 负责"生成block_id 并管它的生命周期"，Worker 的 BlockTable 负责"把 block_id维护成kernel 能查询的二维表"

#### 层1: CPU侧（控制面，权威）

BlockPool 是全局扁平的物理块池，不感知请求。 它管理的是 GPU 显存块的分配与复用，自身在 CPU 进程内。 每个块都带有一些元数据：引用计数、哈希值等，KVCacheBlock 只有这些字段，不含 KV 字节。

```text
BlockPool
┌──────────────────────────────────────────────┐
│ blocks: [KVCacheBlock(0), KVCacheBlock(1),   │
│          KVCacheBlock(2), ...]               │
│          ↑ 只有 block_id / ref_cnt / hash    │
│ free_block_queue: 空闲块的双向链表              │
│ cached_block_hash_to_block: hash → block     │
└──────────────────────────────────────────────┘
```

调度器不直接碰 BlockPool，而是通过 KVCacheManager 来分配和查询。 KVCacheManager 是引擎级别的单例组件，内部持有 req_to_blocks 字典来记录每个请求分到了哪些块。 req_to_blocks 本质上还是"请求 → 物理块列表"的映射，但它不再是 V0 那种内嵌在调度器里的 BlockTable，而是作为 Manager 的内部状态存在。

请求侧的索引变成了 req_to_blocks，由 KVCacheManager 持有。

###### 只缓存满块：命中率与重算成本的取舍

一个块写满之后，才根据它里面的完整 token 内容算哈希，存进 cached_block_hash_to_block，供后来的请求命中复用。

没写满的块，内容还会变，哈希不稳定，所以不往里放。每次都要重新算，牺牲少量命中长度，换取哈希表、块管理和缓存一致性的大幅简化

这就直接引出了 append-only：既然未满块不进哈希表、不参与跨请求共享，那就不存在"两个请求共享一个未满块、某个要写时得复制"的场景。

总结：默认路径下，不完整的块不会获得哈希值，也不会被注册到全局哈希表

- 代价: 每个请求的末尾最多有 block_size 个 token 无法被前缀缓存覆盖。而且命中还有一个额外的截断——即使满块全部命中，命中上限是 num_tokens - 1，最后一个 token 必须重新跑一遍 forward 来产生 logits，所以实际可命中的 token 数还要再减 1。

#### 层2: GPU侧（数据面 + 索引副本）

有一个 BlockTable 类
```text
BlockTable
┌──────────────────────────────────────────────────────┐
│ block_table: [max_num_reqs, max_num_blocks_per_req]  │
│              int32，CPU 权威 + GPU 副本                │
│                                                      │
│ num_blocks_per_row: [max_num_reqs]                   │
│              每行实际用了几个块                         │
│                                                      │
│ slot_mapping: [max_num_batched_tokens]               │
│              int64，compute_slot_mapping 产出         │
└──────────────────────────────────────────────────────┘
```

这个类在 vllm/v1/worker/block_table.py 它维护一张 `[max_num_reqs, max_num_blocks_per_req]` 的 int32 张量，加一个 `num_blocks_per_row` 记录每行实际用了几个块，再加一个 slot_mapping 的 int64 buffer

调度器在 CPU 侧决定好当前 batch 的块分配，通过 append_row / add_row 写进这张表的 numpy 视图，然后同步到显存。attention kernel 下一步直接从显存里读这张表

###### KV 数值本体位置
上面两个结构都是元数据，真正装 K/V 数值的是显存里的大张量：

```text
gpu_kv_cache（通常每层一个）
shape ≈ [num_gpu_blocks, block_size, num_kv_heads, head_dim]
```

KVCacheBlock 的 block_id 就是这张表的下标，靠它和 BlockPool 对齐。 所以 block_table 的"CPU 权威 + GPU 副本"是索引的副本，不是 KV 数据的副本。

## solt计算链路
slot = physical_block_id × block_size + offset_in_block
offset_in_block 来自 token 在逻辑块内的位置

具体查询链路(类似操作系统的分页管理)： 
```text
token 的绝对位置 pos
    ↓ pos // block_size
逻辑块号 logical_block_idx
    ↓ 查 block_table[req_id][logical_block_idx]
物理块号 physical_block_id
    ↓ pos % block_size
块内偏移 offset_in_block
    ↓ 拼接
全局 slot = physical_block_id * block_size + offset_in_block
```
block_table 存的是“逻辑块 → 物理块”的映射，但 slot 本身不是 block_table 直接存出来的
block_table 给你物理块号，你还需要拿 token 位置去算块内偏移，两者拼起来才是 slot

## 全池显存占满的反应
get_new_blocks(num_blocks) 在 free queue 不够时，不返回空列表，直接抛 ValueError

调度器在尝试分配前会用 get_num_free_blocks() 做前置判断，不够就直接走抢占逻辑，而不是让 get_new_blocks 抛异常 。异常是防御性的，不是正常控制流。
