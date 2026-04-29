- UDF
一对一的关系，输入一个值，输出一个值
- UDAF
多对一的关系，通常与groupBy一起使用
- UDTF
一对多的关系，类似于flatMap

目前Python仅支持UDF函数

## 定义UDF函数
```python
spark.udf.register(name,function,returnType)
```
注册的udf支持DSL与SQL类型
返回值用于DSL风格，传参给的名字用于SQL风格
- UDF函数针对于一列的每一条记录都执行一遍这个
name只能用于SQL风格
returnType用于DSL风格
```python
    def myUDF(number):
        return number*10
    udf1 = spark.udf.register("udf1",myUDF,IntegerType())
```
#### SQL风格
```python
df4 = df.selectExpr("id", "name", "age", "upper(name) as upper_name")
df4.show()
```
或者先注册成临时表
#### DSL风格（推荐）
```python
df.select(udf1('number')).show()
```

#### 其他返回值
返回值也可以是字典，列表等等类型的数据
- 列表返回值
```python
ArrayType(IntegerType())
```
- 字典返回值
```python
    StructType().add("name",StringType(),nullable=False).add("age",IntegerType(),nullable=False)
```

## 窗口函数
既显示聚集前的函数，又显示聚集后的函数
[[数据 && 信息/DB/SQL/10.  窗口函数|10.  窗口函数]]

