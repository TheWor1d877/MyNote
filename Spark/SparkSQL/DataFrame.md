## 组成
在结构层面
- StructType对象描述这个给DataFrame的表结构
- StructField描述一个列的信息

在数据层面
- Row对象记录一行数据
- Column对象记录一列数据并包含列的信息
![[Attachments/Pasted image 20250829152919.png]]

## 构建
#### 从RDD转换而来
```python
spark.createDataFrame(rdd,schema=['name','age'],)
```
schema是列名的信息

- 打印表结构
```python
df.printSchema()
```

- 打印数据
```python
df.show(20,False)
```
1. 展示多少数据
2. 是否对列进行截断，列的数据长度超过20个字符串就截断

- 输出
```
root
 |-- name: string (nullable = true)
 |-- age: long (nullable = true)

+-------+---+
|name   |age|
+-------+---+
|Michael|29 |
|Andy   |30 |
|Justin |19 |
+-------+---+

```
#### 从RDD使用toDF构建
```python
rdd.toDF(schema=...)
```
对类型不敏感，靠Spark推断
需要提前做类型转换
#### 从StructType构建
```python
    struct = StructType()\
           .add("name",StringType(),nullable=False)\
           .add("age",IntegerType(),nullable=False)
      
           df = spark.createDataFrame(rdd,schema=struct)
```
定义StructType类型然后传参传这个StructType就行

#### 基于Pandas的DataFrame
```python
    pdf = pd.DataFrame({
        "id":[1,2,3],
        "name":["Alice",'Peter',"John"]
    })

    df = spark.createDataFrame(pdf)

    df.show()
```

#### 通过SparkSQL的统一API构建
```python
sparksession.read.format()
.option()
.schema()
.load()
```
- 从txt文件中读取
```python
    schema = StructType()\
    .add("name",StringType(),nullable=False)\
    .add("age",StringType(),nullable=False)

    df = spark.read.format("text")\
    .schema(schema=schema)\
    .load("hdfs://ubuntu1:8020/input/sql/people.txt")

    df.printSchema()
```
- 从json读取
json自带类名与类型，更加简单
自动识别
```python
    df = spark.read.format("json")\
    .load("hdfs://ubuntu1:8020/input/sql/people.json")

    df.printSchema()
```
- 从csv读取
```python
    df = spark.read.format('csv')\
    .option("sep",',')\
    .option('header',True)\
    .option('encoding','utf-8')
    .schema(字符串形式|StructType类型|等等)
    .load(...)
```
- 从parquet读取
parquet是Spark中常用的一种列式文件存储格式
与Hive中的ORC相似
序列化存储的，具有压缩属性
```python
    df = spark.read.format("parquet")\
    .load("hdfs://ubuntu1:8020/input/sql/people.json")

    df.printSchema()
```

## 编程
- DSL风格
```python
df["name"]
.select
.where
.groupby
...
```
- SQL风格
```python
df.createTempView() # 注册一个临时表
df.createOrReplaceTempView() # 注册一个临时表，如果存在就替换
df.createGlobaltempView() # 注册一个全局表，能跨越SparkSession
```
注册之后使用
```python
spark.sql()
```

## 计算函数SQL包
```python
from pyspark.sql import functions
```

里面有相应的工具包

## 写出
#### 基于统一API
```python
df.write.mode("overwrite").format("text"|"csv"|"json"|"parquet").option(...).save(路径)
```
默认是parquet文件
- text读入的时候只能单列读入，写出也是一样

#### 通过JDBC的读写，写入到MySQL中
Spark的JDBC驱动安装路径
/export/server/miniconda3/envs/pyspark/lib/python3.8/site-packages/pyspark/jars

```python
# 1. 写入df到mysql数据库中
df.write.mode("overwrite").\
    format("jdbc").\
    option("url", "jdbc:mysql://node1:3306/bigdata?useSSL=false&useUnicode=true").\
    option("dbtable", "movie_data").\
    option("user", "root").\
    option("password", "87780801").\
    save()
```