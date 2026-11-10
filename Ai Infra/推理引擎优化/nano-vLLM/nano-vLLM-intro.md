## 整体路线图
核心就一个循环，三步：调度 → 执行 → 后处理
generate → step → schedule → model_runner → postprocess

Scheduler 决定这次跑哪个请求

ModelRunner执行前向，把选中的序列送入GPU，prefill的时候一次算完整的prompt，decode时候每个序列只生成一个新token

Scheduler.postprocess收尾： 把新 token 拼回序列，检查是否结束（命中 EOS 或达到 max_tokens），释放完成的序列占用的 KV 块。

## 目录
engine/  推理引擎核心代码
layers/   神经网络基础组件，各种算子
models/  具体的模型结构
utils/ 和 根目录 基础设施与入口

## engine/


#### scheduler.py
