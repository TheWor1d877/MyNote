插件机制：不重编 NCCL 也能扩展能力

`src/plugin/plugin_open.cc` 负责动态打开外部共享库，支持：

- network plugin
- tuner plugin
- profiler plugin
- env plugin

这是一个极其关键的架构决策：NCCL 核心保持稳定，而网络、调优策略和观测能力则可以按站点需求扩展。