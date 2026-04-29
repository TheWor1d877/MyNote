- 在SparkSQL中提供的自适应查询
- 动态优化查询,增加查询性能

由于缺乏或者不准确的数据统计信息(元数据)和对成本的错误估算(执行计划调度)导致生成的初始执行计划不理想
在Spark3.x版本提供Adaptive Query Execution自适应查询技术

通过在”运行时”对查询执行计划进行优化，允许Planner在运行时执行可选计划，这些可选计划将会基于运行时数据
统计进行动态优化，从而提高性能。

Adaptive Query Execution AQE主要提供了三个自适应优化：
• 动态合并 Shuffle Partitions
• 动态调整Join策略
• 动态优化倾斜Join(Skew Joins)

开启AQE方式
set spark.sql.adaptive.enabled = true;

#### Shuffle Partitions
将一些小分区合并，提高性能
![[Attachments/Screenshot_20250831_103617.jpg]]

#### 动态调整Join策略
可以按照情况将sort merge join 动态变成  broadcast hash join
如图：
![[Attachments/Screenshot_20250831_104205.jpg]]
2直接将所有数据发送给1，不用排序1对1的合并了

#### 动态优化倾斜join
![[Attachments/Screenshot_20250831_104807.jpg]]

join操作的时候再shuffle之后如果发现其中的一块特别大，就会分割成两块进行操作

- 触发条件
1. ==分区大小== > spark.sql.adaptive skewJoin skewedPartitionFactor (default=10) × "median partition size(中位数分区大小)"
2. 分区大小 > spark.sql.adaptive skewJoin skewedPartitionThresholdInBytes (default=256MB)