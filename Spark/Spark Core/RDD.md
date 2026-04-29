## 介绍
叫做<span style="color:rgb(221, 85, 85)">弹性分布式数据集</span>
数据抽象对象
- 是一个数据集，用于存放数据
- 数据是分布式存储的，用于分布式计算
- RDD中的数据可以存放在内存中或者磁盘中
- 可以进行动态的扩容与缩容

## 特性
 - RDD是有分区的
 - 计算方法会作用到每一个分区上面
 - RDD之间由相互依赖的关系的
 - KV形RDD可以有自己的分区
 - RDD分区的读取尽量靠近你数据所在地


####  RDD是有分区的
一份数据被分到了多个分区上面
```python
>>sc.parallelize([1,2,3,4,5,6,7,8,9],3).glom().collect()
[[1,2,3],[4,5,6],[7,8,9]]
```
#### 计算方法会作用到每一个分区上面
执行一个map操作对于所有的数据乘以10，所有的分区都会受到影响
```python
map(lambda x: x*10)
```
#### RDD之间由相互依赖的关系的
'血缘关系'
```python
    word_rdd = sc.textFile('hdfs://ubuntu1:8020/input/word.txt')
       words_rdd = word_rdd.flatMap(lambda line: line.split(' '))
       word_count_rdd = words_rdd.map(lambda word: (word, 1))
       result = word_count_rdd.reduceByKey(lambda a, b: a + b)
```
上述代码中，所有的RDD都是由上一个RDD迭代过来的，链条形状的关系
#### KV形RDD可以有自己的分区
可选的特性： 因为不是所有的元素都是kv型的
二元元组存储的数据就是kv型的
默认分区器：hash分区规则(Hive中的分桶表类似)，可以手动设置
#### RDD分区的读取尽量靠近你数据所在地
尽量使用本地路径减少网络使用
Spark在确保并行计算能力的前提下，尽量确保本地读取数据
不能为了读取数据，减少Executor的开设

