首先按照local的方法在所有机子上配置python与相关患环境变量
## 编辑workers
编辑ubuntu1的workers文件
```bash
ubuntu1
ubuntu2
ubuntu3
```

## 编辑spark的env
进入spark-env.sh：
```bash   
JAVA_HOME=/export/server/jdk
   ##HADOOP软件配置文件目录，读取HDFS上文件和运行YARN集群
   HADOOP_CONF_DIR=/export/server/hadoop/etc/hadoop
   YARN_CONF_DIR=/export/server/hadoop/etc/hadoop
   
   ##指定spark老大Master的IP和提交任务的通信端口
   #告知Spark的master运行在哪个机器上
   export SPARK_MASTER_HOST=ubuntu1
   #告知sparkmaster的通讯端口
   export SPARK_MASTER_PORT=7077
   #告知sparkmaster的webui端口
   export SPARK_MASTER_WEBUI_PORT=8080
   
   #workercpu可用核数
   export SPARK_WORKER_CORES=1
   #worker可用内存
   export SPARK_WORKER_MEMORY=2g
   #worker的工作通讯地址
   export SPARK_WORKER_PORT=7078
   #worker的 webui地址
   export SPARK_WORKER_WEBUI_PORT=8081
   
   ##设置历史服务器
   #配置的意思是将spark程序运行的历史日志存到hdfs的/sparklog文件夹中
   export SPARK_HISTORY_OPTS="-Dspark.history.fs.logDirectory=hdfs://ubuntu1:8020/sparklog/ -Dspark.history.fs.cleaner.enabled=true"
```

## 修改spark的默认配置
spark-default.conf文件
```bash
#开启spark的日期记暴功能
spark.eventLog.enabled true
#设置spark日志记录的路径
spark.eventLog.dir hdfs://node1:8020/sparklog/
#设置spark日志是否启动压缩
spark.eventLog.compress true
```

## 配置日志为WARN级别


