## 抢占方式
vLLM V1 默认抢占模式是 RECOMPUTE 而非 SWAP。被抢占的请求释放全部 KV 块，回到 waiting 队列，之后从头重算 prefill。
重算拉长的是 TTFT，不直接影响 TPOT。

V1 里 swap 基本被移除了，只在一种情况下还用：多序列请求（比如 beam search），因为重算在这种场景下不适用。

swap 把 KV 写到 CPU 再读回，数据搬运的延迟比重算一段 prefill 高得多。 vLLM 的设计讨论和基准显示，重算的开销通常不到 swap 延迟的 20%；块大小较小时，数据搬运成本占比更高，重算优势更明显。

## allocate失败时候的全流程
#### 分配失败信号 
vLLM V1 的抢占由 sentinel return value 驱动，不是靠抛异常。KVCacheManager.allocate_slots() 在池子装不下时返回 None，Scheduler.schedule() 用 if new_blocks is not None 来判断是否进入抢占分支。
```text
Scheduler.schedule()  RUNNING 阶段
        │
        ▼
allocate_slots(req, num_new_tokens)
        │
        ├── 返回 [block_ids]  → 正常继续
        │
        └── 返回 None          → 进入模块 2
```

#### 调用方接收信号，执行抢占
Scheduler.schedule() 在遍历 RUNNING 队列时，对每个请求包一层 while True
```text
while True:
    new_blocks = allocate_slots(...)
    if new_blocks is not None:
        break
    victim = running.pop()        # 选牺牲品
    _preempt_request(victim)      # 回收其 KV
```
###### 淘汰策略：
- FCFS（默认）
`victim = self.running.pop()`
running 列表按准入顺序排列，pop() 取尾部——最晚进入 running 的请求先被踢。O(1)，不遍历、不读任何指标

代价：对“已算过多少”完全盲，可能踢到一条已 decode 很长的请求，把它的 KV 全打成重算。

- Priority
`victim = max(self.running, key=lambda r: (r.priority, r.arrival_time))`
priority 数值越小优先级越高，arrival_time 是到达时间戳。字典序比较：先取 priority 数值最大者（优先级最低），同级再取 arrival_time 最晚者。
max 取“最差”，与 waiting 队列用 min 取“最好”调度方向相反。

保的是优先级语义——低优先级请求不能挤占高优先级请求的 KV，哪怕它已经算了很多。同样不看 num_computed_tokens

后续修补 PR #51574 把 key 改成 (priority, -min(num_preemptions, 3), arrival_time)，让被抢多次的请求在同级里更晚被再选中，但第一序仍是 priority。

#### 回滚操作
把 victim 的 num_computed_tokens 归零、释放全部 KV block、塞回 waiting 队列。恢复时调度器看到的是一条“从没算过”的请求。

代价：与丢失 token 数成正比的额外 prefill 计算，加上这段计算期间 victim 无法产出 token 的排队时间

插回队首是防饥饿：被抢请求已经在跑、被迫中断。若排到队尾，新流量持续插队会使其长期得不到恢复，形成饥饿。prepend_request 保证其优先于全新请求重新调度。

## 抢占与cancel/timeout的区别
抢占：释放 → 请求还活着 → 回 waiting → 重新调度 → 重算 → 再占 block
cancel/timeout：释放 → 请求进终态（FINISHED_ABORTED / 超时）→ 不再调度