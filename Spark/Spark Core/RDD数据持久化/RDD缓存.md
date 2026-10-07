## RDD的数据是过程数据
新的RDD生成，代表老RDD数据的消失
 一个RDD只能迭代一次，多了就会自动销毁
<span style="color:rgb(221, 85, 85)">要想再使用这个RDD，只能从RDD根部开始</span>

## RDD缓存
```python
# RDD3 被2次使用，可以加入缓存进行优化
rdd3.cache()                           # 缓存到内存中.
rdd3.persist(StorageLevel.MEMORY_ONLY) # 仅内存缓存
rdd3.persist(StorageLevel.MEMORY_ONLY_2) # 仅内存缓存，2个副本
rdd3.persist(StorageLevel.DISK_ONLY) # 仅缓存硬盘上
rdd3.persist(StorageLevel.DISK_ONLY_2) # 仅缓存硬盘上，2个副本
rdd3.persist(StorageLevel.DISK_ONLY_3) # 仅缓存硬盘上，3个副本
rdd3.persist(StorageLevel.MEMORY_AND_DISK) # 先放内存，不够放硬盘
rdd3.persist(StorageLevel.MEMORY_AND_DISK_2) # 先放内存，不够放硬盘，2个副本
rdd3.persist(StorageLevel.OFF_HEAP) # 堆外内存(系统内存)

# 如上API，自行选择使用即可
# 一般建议使用rdd3.persist(StorageLevel.MEMORY_AND_DISK)
# 如果内存比较小的集群，建议使用rdd3.persist(StorageLevel.DISK_ONLY) 或者就别用缓存了 用Checkpoint

# 主动清理缓存的API
rdd.unpersist()
```
- 存储的时候是分散存储的

缓存是不安全的(设计上认为是不安全的)
- 缓存丢失，就要重新计算
- 缓存分区越多，风险越高
- 写入内存的话，并发写入，写入性能更好
- 保留血缘关系
- 轻量化的技术，保存不重要的，临时的
- 保留的内存与硬盘位置我们不能手动指定
## CheckPoint技术
仅支持是硬盘存储
在设计上是安全的，不保留血缘关系
不会丢失
采用的策略是: 集中收集存储
我们可以控制它存储在HDFS上面，这样保证了数据的安全
#### 使用
```python
# 保存
sc.setCheckpointDir("hdfs://ubuntu1:8020/cache/......")

# 使用直接调用算子就可以保存
rdd.checkpoint()
```