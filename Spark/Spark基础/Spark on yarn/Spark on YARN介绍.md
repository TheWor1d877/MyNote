## 本质
Master角色由YARN的ResourceManager担任
Worker角色由YARN的NodeManager担任
Driver运行在YARN容器内部 或者提交任务的客户端中
Executor运行在YARN提供的容器中

这种情况下
Spark的角色与MapReduce就是一样的了
![[Attachments/Pasted image 20250826210019.png]]

YARN管理资源调度了，Spark只管在容器内部计算


