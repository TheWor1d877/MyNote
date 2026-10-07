## countByValue
适用于kv型的RDD
返回一个dict,按照value统计次数
```python
rdd.countByValue()
```

## collect
将rdd中的数据搜集到一个Driver中形成一个list对象
注意：内存问题！结果数据集不要太大

## reduce
reduceByKey是transaction算子
而reduce是一个action算子
```python
rdd.reduce()
```

## fold
带有初始值的reduce
- 分区内聚合
- 分区间聚合
二者都要加上初始值
```python
[[1,2,3],[4,5,6]]
# 假设初始值是10
# 计算 (10+1+2+3)+（10+4+5+6）+ 10
```

## first
取出RDD中的第一个元素

## take
取前n个元素，以list的形式放里面

## top
按照降序排序的take

## count
计算有几个元素

## takeSamples
```python
rdd.takeSamples(True or False(是否重复),数量,随机数种子)
```

## forEach
没有返回值的map

## saveASTextFile
写出，支持本地与HDFS文件系统
Executor自行写入，性能好，会写出多份文件

