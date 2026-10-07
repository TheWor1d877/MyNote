内置了完全的SparkAPI并且提交到Spark中执行

Pyspark与Spark的对比：
![[Attachments/Pasted image 20250826223232.png]]

## 解析WorldCount代码
```python
from pyspark.sql import SparkSession

if __name__ == "__main__":
    spark = SparkSession.builder \
        .appName("WorldCount") \
        .config("spark.driver.extraLibraryPath", "/export/server/hadoop/lib/native") \
        .config("spark.executor.extraLibraryPath", "/export/server/hadoop/lib/native") \
        .config("spark.driver.extraJavaOptions", "-Djava.library.path=/export/server/hadoop/lib/native") \
        .config("spark.executor.extraJavaOptions", "-Djava.library.path=/export/server/hadoop/lib/native") \
        .master("local[*]") \
        .getOrCreate()

    sc = spark.sparkContext

    word_rdd = sc.textFile('hdfs://ubuntu1:8020/input/word.txt')
    print("从HDFS读取文件")

    words_rdd = word_rdd.flatMap(lambda line: line.split(' '))
    word_count_rdd = words_rdd.map(lambda word: (word, 1))

    result = word_count_rdd.reduceByKey(lambda a, b: a + b)

    print("单词计数结果:")
    print(result.collect())

    spark.stop()
```
#### 原理
![[Attachments/Pasted image 20250828113639.png]]

- 讲文件分为多个部分，每个部分交给一个节点
- 在第一阶段中每个节点做自己该做的任务
- 在第二阶段中，所有节点交给Driver整合并输出
