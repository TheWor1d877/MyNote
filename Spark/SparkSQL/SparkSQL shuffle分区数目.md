当在 SparkSQL 中执行任务（Job）时，如果发生了 Shuffle（洗牌）操作（例如 join, group by, order by 等），Spark 需要将数据重新分布到不同的分区。

Spark 为这种 Shuffle 操作后的数据设置的默认分区数量是 200。这个默认值由配置参数 `spark.sql.shuffle.partitions `控制。

但是分区数不合理的危害:
- 分区数太少：会导致每个分区的数据量过大，可能引起内存溢出（OOM）、数据倾斜以及无法充分利用集群资源等问题。
- 分区数太多：会产生大量的小任务，导致不必要的任务调度开销和管理开销，同样会降低性能

## 与spark.default.parallelism比较
1. 数据输入 (起点)：
- 从文件创建 DataFrame：分区数由文件格式和大小决定（如 HDFS 块数），不受这两个参数影响。
- spark.range(n).toDF() 或 RDD转换而来：分区数可能由 spark.default.parallelism 影响。

2. 数据处理 (中途)：
- 进行 select, filter, withColumn 等窄变换：分区数保持不变，继承自上游。
- 进行 groupBy("key").count(), df1.join(df2) 等宽变换：
	- 如果这是 DataFrame/SQL 操作，分区数由 spark.sql.shuffle.partitions 决定。
	- 如果这是 RDD 操作（如 rdd.reduceByKey），分区数由 spark.default.parallelism 决定（如果用户未指定）。

3. 最终输出 (终点)：数据的分区数决定了写入文件时的文件数