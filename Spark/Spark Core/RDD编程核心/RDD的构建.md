- 程序入口对象是SparkContext
## 创建方式
#### 通过并行化集合
本地转成分布式的方式
```python
new_rdd = sparkcontext.parallelize(集合对象，分区数)
```

#### 读取文件创建
读取文件创建出来一个RDD对象
```python
    spark = SparkSession.builder.getOrCreate()
    sc = spark.sparkContext
    print("begin")
    file_rdd = sc.textFile('hdfs://ubuntu1:8020/input/word.txt',3)
    part = file_rdd.getNumPartitions()
    print(part)
```

注意：输入的分区数是“参考数据”，具体是多少最终由spark决定

#### 小文件的读取
比textfile更加适合读取小文件
```python
    file_rdd = sc.wholeTextFiles('hdfs://ubuntu1:8020/input/tiny_files',3)
```