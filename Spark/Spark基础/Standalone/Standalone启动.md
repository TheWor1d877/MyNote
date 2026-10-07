## 启动
返回Ubuntu1:
- 启动历史服务器
```bash
sbin/start-history-server.sh
```
- 启动其他的所有工作
```bash
sbin/start-all.sh
```

## 打开WebUI
```bash
ubuntu1:8080
```

## 执行计算工作
master的地址(访问方法)：spark://ubuntu1:7077

如： `./pyspark --master spark://ubuntu1:7077`

