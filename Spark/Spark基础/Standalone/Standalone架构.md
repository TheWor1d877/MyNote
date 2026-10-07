Standalone架构是Spark自带的集群模式
真实的在多个机器之间搭建SPark集群的环境，完全可以使用该模式搭建多机器集群，用于实际的大数据处理
StandAlone 是完整的Spark运行环境,其中:
<span style="color:rgb(221, 85, 85)">Master角色以Master进程存在, Worker角色以Worker进程存在
Driver和Executor运行于Worker进程内, 由Worker提供资源供给它们运行</span> 

## Standalone架构的进程
StandAlone集群在进程上主要有3类进程:
1. 主节点Master进程：
Master角色, 管理整个集群资源，并托管运行各个任务的Driver
2. 从节点Workers：
Worker角色, 管理每个机器的资源，分配对应的资源来运行Executor(Task)；
每个从节点分配资源信息给Worker管理，资源信息包含内存Memory和CPU Cores核数
3. 历史服务器HistoryServer(可选)：
Spark Application运行完成以后，保存事件日志数据至HDFS，启动HistoryServer可以查看应用运行相关信息

Standalone模式下面可以有多个任务同时进行，
每有一个任务就代表了master中有一个Driver

## 分层架构
启动一个pyspark就是一个Driver
一个Driver里面由多个子任务
每个子任务由多个阶段（也可能是一个）完成
在每个阶段都有多个Task

应用程序 => 子任务 => 阶段 => Task任务

在WebUI中能看到相关的DAG图

