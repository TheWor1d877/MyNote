- 有向无环图制作的一个执行流程图

## Job与Action
Action是执行链条的开关
一个Action对应着一个Job对应着一个DAG

## DAG的宽窄依赖与阶段划分
- 窄依赖： 父RDD的一个分区，全部将数据发送给子RDD的一个分区
- 宽依赖(shuffle)：父RDD的一个分区将数据发送给子RDD的多个分区

shuffle会造成性能问题

- 阶段划分依据：按照宽依赖进行划分
