## Hive的两部分
- 元数据管理
- SQL解析然后提交到MR执行

现在我们将第二部分改成SQL解析成RDD的解释器，然后交给SparkSQL执行计算

## Spark连接Hive
1. 元数据管理启动
2. Hive的元数据管理仓库与端口让Hive知道

在Spark的conf中创建hive-site.xml文件
```xml
<configuration>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
        <description/>
    </property>

    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://node3:9083</value>
        <description/>
    </property>
</configuration>
```
- 给Spark配置mysqlJar包
