分布式的集合上面的API称之为算子

分为两种类型
- Transformation算子
返回结果是一个rdd的就是转换算子

- Action算子
返回值是空，python对象等等

转换算子是一个构建计划的过程
action才是让计划执行的核心，流水线的开关

