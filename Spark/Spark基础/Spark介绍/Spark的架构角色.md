## 回顾:YARN的角色
- 资源管理层面
集群资源管理者： ResourceManager
单机资源管理者: NodeManager
- 计算任务层面
单任务管理者：ApplicationMaster
单任务执行者： Task

## Spark
![[Attachments/Pasted image 20250825163841.png]]
- Master是集群的资源管家
- Worker是单机资源的管家
- Driver是单任务的管理者
- Executor：负责执行任务
特例：在Local模式下Driver也可以执行任务

Local模式下只能运行一个Spark程序，如果运行多个Spark程序，就是由多个相互的独立的Local进程在执行
