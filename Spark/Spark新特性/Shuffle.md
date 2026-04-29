两种Shuffle管理器
#### HashShuffleManager
按照哈希分组，分组的过程在分区的内存buffer中完成，打包成文件发送
![[Attachments/Screenshot_20250831_095014.jpg]]

#### SortShuffleManager
- 普通运行机制
![[Attachments/Screenshot_20250831_100323.jpg]]

当数据数目到达一定后，就会分批排序(默认1w条)然后写入内存
最后写成一个文件
新task按照索引文件明确应该使用磁盘文件的哪部分，减少网络IO
接收端可以接受这些数据，这为Reduce操作准备了数据


- Bypass运行机制
![[Attachments/Screenshot_20250831_100339.jpg]]

Bypass机制是一种优化机制，当满足特定条件时，可以跳过数据排序的步骤，直接将数据写入对应的分区文件以提高性能。根据搜索结果，启用 SortShuffleManager 的 bypass 机制需要满足以下条件：
-  Shuffle Read Task 的数量必须小于等于配置参数  spark.shuffle.sort.bypassMergeThreshold  的值，默认情况下为 200。
task要从所有以前task的文件中拉取数据的时候，这个值就等于分区数
-  触发 Shuffle 的算子不能是聚合类算子，例如  reduceByKey ，而只能是非聚合类的Shuffle算子，比如 join 操作。

## 设置参数
`spark.shuffle.manager`	设置 ShuffleManager 类型	默认为 `sort`，可选 `hash` 或 `tungsten-sort`

`spark.shuffle.sort.bypassMergeThreshold`	触发 bypass 机制的阈值	默认 200，调大可跳过排序，提升写性能

`spark.shuffle.compress`	是否压缩 Shuffle 输出	默认为 true，减少 IO	

`spark.sql.shuffle.partitions`	默认 Reduce Task 数	根据数据量合理设置，避免过多任务	