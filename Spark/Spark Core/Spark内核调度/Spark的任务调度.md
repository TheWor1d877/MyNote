Spark的任务，由Driver进行调度
这个过程包含
- 逻辑DAG的产生
- 分区DAG的产生
- Task划分
- 将Task分配给Executor并且监控他的工作

## DAG调度图
基于DAG图，明白每个分区中的Task的任务与交互

## Task调度器
基于DAG Scheduler的产出，来规划这些逻辑的Task，应该那些屋里的executor上面执行,以及监控管理他们的运行

