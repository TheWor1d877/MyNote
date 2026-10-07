## 两种模式介绍
- Cluster模式
Driver运行在YARN容器的内部，与Master处于同一个容器
运行效率高，但是日志查看难
- Client模式
Driver运行在客户端进程中，比如Driver运行在spark-submit程序进程中
缺点：通讯效率低，查看日志简单，直接在shell中查看日志

![[Attachments/Pasted image 20250826220555.png]]

## Client模式演示
```bash
# 启动的时候可以带上性能参数
bin/spark-submit --master yarn --deploy-mode client(默认) --driver-memory 512m --num-executor 3 --total-executor-core 3 py文件名
```
Driver是计算的boss
Master是资源调度的boss


## 两种模式的详细流程
#### Driver模式
![[Attachments/Pasted image 20250826222420.png]]
简图：客户端 => Driver => ResourceManager =>  Master => NodeManager => Executor => Driver
#### Cluster模式
![[Attachments/Pasted image 20250826222812.png]]
简图：客户端 => ResourceManager(Driver) =>  Master => NodeManager => Executor => Driver