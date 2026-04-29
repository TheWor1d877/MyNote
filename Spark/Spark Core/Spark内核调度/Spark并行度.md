现有并行度再决定需要多少分区

默认并行度就是1
- 全局配置参数
```bash
spark.default.parallelism
```
一般设置成CPU总核心的2到10倍
让大多数的Task在等待，能让CPU在100%时间内出力
- 在Pycharm中设置
![[Attachments/Pasted image 20250829144324.png]]
- 一个RDD的分区只能别一个Task分区

一个并行度 = 一个分区 = 一个Task
一个Executor核心 = 一个线程