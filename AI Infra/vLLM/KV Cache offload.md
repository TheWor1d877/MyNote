GPU侧BlockPool容量受显存限制。块被重新分配并覆写后，其哈希条目被删除，对应前缀的KV数据在GPU侧永久丢失，后续同前缀请求需要重算prefill。

开启offload后，块在GPU侧被驱逐前后，把内容复制一份到CPU pinned内存，用同一个块哈希记账。后续同前缀命中时，执行路径从"重算prefill"变为"从CPU拷贝回显存"。

## 控制面与数据面
```text
Scheduler 进程（控制面）              Worker 进程（数据面）
─────────────────────                ─────────────────────
KVCacheManager                        gpu_kv_cache（KVCache数据本体）
 ├── BlockPool                        block_table / slot_mapping
 │    cached_block_hash_to_block:
 │        BlockHash → gpu_block_id
 └── OffloadingManager（新增）
      _key_to_cpu_block_id:
          OffloadKey(hash + group) → cpu_block_id
      _cpu_block_id_to_key:
          cpu_block_id → OffloadKey
      _cpu_block_states:
          cpu_block_id → {FREE | STORED | LOAD_IN_FLIGHT | ...}
          
      lookup / prepare_load / prepare_store
          │ LoadSpec / StoreSpec
          └─────────► OffloadingHandler
                      pinned CPU 池 + 独立 CUDA stream
````

## 三个并发原语

控制面代码在 vllm/v1/kv_offload/。Manager 运行在调度器进程内，提供:

- lookup()：从首块开始统计连续命中长度
- prepare_load()：登记读请求，期间禁止淘汰该块
- prepare_store()：登记写请求，返回被驱逐的块

它与 BlockPool 是两套独立账本，仅通过块哈希对齐，不共享 ref_cnt 语义。 属于 BlockPool 的 cached_block_hash_to_block 不与 OffloadingManager 共享。

store = GPU → CPU：块要被驱逐、内容即将丢失时，拷一份到 CPU 侧保留 load = CPU → GPU：后续同前缀请求命中 CPU 副本，把内容拷回 GPU 新分配的块

#### lookup

输入：当前请求按顺序排列的前缀块哈希列表。 输出：从第一个块开始，连续命中 CPU 侧的块数。 作用：告诉调度器"这段前缀不用重算，但需要从 CPU 搬回来"。

#### prepare_store

触发时机：某个 GPU 块要被重新分配、其哈希条目即将被` _maybe_evict_cached_block `删除时。

动作：

- 在 CPU 侧分配一个空闲块号
- 生成一个 StoreSpec（含源 GPU 块号、目标 CPU 块号、字节范围）
- 把这个 spec 放进 kv_event_queue，交给 Worker 侧 handler 执行异步 D2H 拷贝
- 拷贝完成后，在 `_key_to_cpu_block_id `里插入条目，并在` _cpu_block_states` 里把该块状态置为 STORED

#### prepare_load

触发时机：lookup 命中了 CPU 侧的块，且调度器已为新请求分配了新的 GPU 块。

动作：

- 在 `_cpu_block_states`里把该块状态置为 LOAD_IN_FLIGHT，淘汰策略暂时跳过它
- 生成一个 LoadSpec（含源 CPU 块号、目标 GPU 块号、字节范围）
- 放进 kv_event_queue，交给 Worker 侧执行异步 H2D 拷贝
- 拷贝完成后，block_table 指向新的 GPU 块，请求可以 forward

