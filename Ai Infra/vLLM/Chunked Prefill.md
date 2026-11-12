## 背景
#### 单实例，prefill 和 decode 混在同一个 GPU 上
所有请求进同一个调度器，共享同一个 max_num_batched_tokens 预算

问题：调度器优先把 prefill 塞进 batch。一个长 prefill 会连续占用 GPU 多个调度步，正在 decode 的请求被挤到后面，token 间延迟出现尖峰。

chunked prefill：把长 prefill 切成不超过预算的片段，每步只算一小段。decode 请求每步只占 1 个 token 预算，所以不会被一个巨大的 prefill 彻底堵死

#### PD 分离，有独立的 prefill 池和 decode 池
请求先到 prefill 实例，算完 KV cache，再传给 decode 实例 

decode 实例上没有正在跑的长 prefill 来抢占 GPU 时间，decode 请求的 token 间延迟天然就稳。prefill 池自己怎么调度，不影响 decode 池的节奏。

此时chunked predill的作用是
控制单个prefill占用的显存峰值
让多个 prefill 请求更细粒度地共享 prefill 实例的算力

## 关键参数`max_num_batched_tokens`
每一批次的token预算
调小 → decode 被 prefill 打断的次数少，ITL 好，但 TTFT 变差（prefill 推进慢）；

调大 → TTFT 好，吞吐高，但 decode 等待被拉长

## Chunked Prefill的调度规则
nano-vllm的调度规则与vllm是相反的，先取 waiting 里的 prefill，再取 running 里的 decode”是 nano-vllm/V0 的语义
vllm在v1中是如果PD不分离，先给 running 里的 decode 每条 1 个 token，剩余预算再给 waiting 里的 prefill，不够就切 chunk
PD分离之后 这个问题不存在

#### 单卡，PD不分离
最朴素的vLLM部署，waiting 队列和 running 队列都在同一个调度器里，共享同一个 max_num_batched_tokens(默认2048)

“先 decode”不等于“只做 decode”。 如果 running 队列为空，或者 running 里的请求都已经因为显存不够被抢占，这一步的预算就全部给 prefill。

#### 单卡/多卡，PD分离
物理上拆成两个实例，一部分做 prefill，一部分做 decode，前面放一个 proxy/router。单卡上跑 PD 分离，就是一张卡上两个进程（或分时共享），一个 P 实例、一个 D 实例，通过 KV transfer 连起来。


## 与CUDA Graph的关系
Chunked prefill 是调度层的 token 预算切分机制。 它决定“这一轮 GPU 要处理哪些 token、各多少”，输出的是一个变长的 token 集合。

CUDA Graph 是执行层的 kernel 启动优化。 它把固定形状的计算图捕获下来，下次同样形状直接 replay，省掉 kernel 启动开销。

二者之间没有关系

prefill 阶段默认不走完整 CUDA Graph，默认的 piecewise 模式会跳过 attention，把其余算子放进图

完整 CUDA Graph 只对纯 decode（uniform decode）批次生效。
decode 批次形状规整且固定，而 prefill 的 attention 形状随 token 数变化，与 Graph 的静态捕获机制不兼容。