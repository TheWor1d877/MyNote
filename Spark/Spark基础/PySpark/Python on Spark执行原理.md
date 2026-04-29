![[Attachments/Pasted image 20250828175709.png]]

Python中构建SparkContent，使用socket与JVMDirver进行沟通
使用Py4j'翻译成jvm程序'，然后交给Spark执行
在Executor内部Task将指令交给pyspark.daemon(一个守护进程，用于当作中转站)执行
所以本质上还是python的进程在执行
- 不能直接将Python给Executor执行


