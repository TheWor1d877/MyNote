## map
将rdd中的算子一条一条的处理，然后返回一个rdd对象
```python
rdd.map(T) -> U
```
T,U表示的是类型不限

## flatmap
将rdd先执行map然后再解除嵌套
```python
rdd.flatmap(T) -> U
```


## mapValues
针对二元元组RDD，对内部的二元元组的value执行map操作
```python
rdd.mapValues(V) -> V
```

## reduceByKey
针对KV型RDD，自动按照key分组，然后按照reduce逻辑，完成组内数据的聚合
```python
rdd.reduceByKey(V,V) -> V
```
比如单词统计，让所有的相同单词的次数相加
- 先在分组之前执行reduce聚合操作，然后再若干次（次数少于总共的组数）进行IO，最后汇总

## groupBy
将RDD参数进行分组
```python
rdd.groupBy(T) -> K
```
T 指的是按照什么进行分组
- 演示
```python
    data = [('a',1),('b',2),('c',1),('a',2),('b',2)]
    rdd = sc.parallelize(data,2)
    result = rdd.groupBy(lambda x:x[0])
    print(result.collect())
    print(result.map(lambda x:(x[0],list(x[1]))).collect())
```
- 输出
```
[('b', <pyspark.resultiterable.ResultIterable object at 0x74847bd5ec70>), ('c', <pyspark.resultiterable.ResultIterable object at 0x74847bd5eeb0>), ('a', <pyspark.resultiterable.ResultIterable object at 0x74847bd5eee0>)]
[('b', [('b', 2), ('b', 2)]), ('c', [('c', 1)]), ('a', [('a', 1), ('a', 2)])]
```
## groupByKey
针对key进行分组，就是上面groupBy的一种表示
```python
rdd.groupByKey() == rdd.groupBy(lambda x:x[1])
```
- 分组的过程中：针对每一个数据要去哪个地方都要走一次IO

## filter
与python一样
```python
rdd.filter(T) -> bool
```

## distinct
```python
rdd.distinct(参数)
```
指定对几个区间进行去重，一般不需要进行传参
什么类型的都能去重

## union
将两个rdd合并成一个rdd输出
```python
rdd.union(rdd1)
```
将rdd1合并到rdd上面
- union算子不去重
- rdd类型不同也是可以合并的

## join
与sql中的join是一样的
- 只能用于kv型算子
```python
rdd.join(rdd1) #内连接
rdd.leftOuterJoin(rdd1) #左外连接
```
按照键进行合并，空的数据使用None代替

## intersection
求交集
```python
rdd.intersection(rdd1)
```

## glom
将rdd加上嵌套
方便看出来rdd的分布情况

## sortBy
对RDD的算子进行排序
```python
rdd.sortBy(T,ascending=True,numPartition=1) -> U
```
numPartition用多少分区进行排序
默认使用升序排序
当你设置 numPartitions=1 时，Spark 会将所有数据通过 Shuffle 移动到一个分区里。这个分区最终会包含全部已排序的数据。
- 优点：数据完全有序。
- 缺点：失去了并行性。因为所有数据都在一个分区里，后续任何对此 RDD 的操作都只能由一个 CPU 核心来处理，无法利用集群的分布式计算能力。如果数据量非常大，单个节点可能无法容纳所有数据，会导致内存不足（OOM）错误。
有多个分区的时候，会将整个数据移动到n个分区中进行排序，临界点的选取采用“随机抽样”决定

## sortByKey
针对 kv型，按照k进行排序


