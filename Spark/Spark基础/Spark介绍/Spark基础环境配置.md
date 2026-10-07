## 安装Spark
- 解压压缩包
- 创建软连接
- 配置annaconda
- 在annaconda中创建虚拟环境
```bash
conda create -n pyspark python=3.12
conda activate pyspark
```
- 配置环境变量
在/etc/profile下面配置环境变量
```bash
export JAVA_HOME=/export/server/jdk
export HADOOP_HOME=/export/server/hadoop
export SPARK_HOME=/export/server/spark
export PYSPARK_PYTHON=/export/server/anaconda3/envs/pyspark/bin/python3
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```
然后配置到/root/.bashrc下面
```bash
export JAVA_HOME=/export/server/jdk
export PYSPARK_PYTHON=/export/server/anaconda3/envs/pyspark/bin/python3
```

## 查看Spark执行任务的WebUI
`ubuntu1:4040`
