单点故障问题的解决办法：
基于文件的单点恢复(只能用于开发与测试环境)：
基于zooKeeper的Standby Master(可以用于生产环境)：
![[Attachments/Pasted image 20250826192656.png]]
哪个master先启动哪个就是活跃的，剩下的就是standby的

## 基于Zookeeper实现HA
#### 构建Zookeeper
- 解压并且创建软连接
- 在ubuntu1上面修改配置文件
```bash
cd /export/server/zookeeper/conf/
cp zoo_sample.cfg zoo.cfg
mkdir -p /export/server/zookeeper/zkdatas/
```
编辑zoo.cfg文件
```bash
#Zookeeper的数据存放目录
dataDir=/export/server/zookeeper/zkdatas
# 保留多少个快照
autopurge.snapRetainCount=1
# 日志多少小时清理一次
autopurge.purgeInterval=1
# 集群中服务器地址
server.1=ubuntu1:2888:3888
server.2=ubuntu2:2888:3888
server.3=ubuntu3:2888:3888
```
- 添加myid属性
在ubuntu1主机的/export/server/zookeeper/zkdatas/这个路径下创建一个文件，文件名为myid ,文件内容为1
```bash
echo 1 > /export/server/zookeeper/zkdatas/myid 
```
- 分发安装包
在node1主机上，将安装包分发到其他机器
第一台机器上面执行以下两个命令
```bash
cd /export/server/
scp -r /export/server/zookeeper-3.4.6/ ubuntu2:$PWD 
scp -r /export/server/zookeeper-3.4.6/ ubuntu3:$PWD
```

- 修改其他机器上面的myid，并且创建软连接
```bash
cd /export/server/
ln -s zookeeper-3.4.6/ zookeeper 
echo 2 > /export/server/zookeeper/zkdatas/myid
```

```bash
cd /export/server/
ln -s zookeeper-3.4.6/ zookeeper 
echo 3 > /export/server/zookeeper/zkdatas/myid
```

- 启动zookeeper
```bash
/export/server/zookeeper/bin/zkServer.sh start
```

- 查看zookeeper的运行状态
```bash
/export/server/zookeeper/bin/zkServer.sh  status
```

#### 设置HA模式
先在 spark-env.sh 中，删除：`SPARK_MASTER_HOST=node1`

增加
```bash
SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=ubuntu1:2181,ubuntu2:2181,node3:2181 -Dspark.deploy.zookeeper.dir=/spark-ha"
```


将spark-env.sh 分发到每一台服务器上
```bash
scp /export/server/spark/conf/spark-env.sh ubuntu2:/export/server/spark/conf/
scp /export/server/spark/conf/spark-env.sh ubuntu3:/export/server/spark/conf/
```
最后重启standalone

- 注意：
zookeeper会自动切换master，整个过程等待20s左右

- ubuntu2的master不一定在8080端口，可能被zookeeper占用，需要提前查看
```bash
# 在zoo.cfg里面更改
admin.serverPort=8080
```
- 确认进程的端口号
```bash
netstat -anp|grep 端口号
```

