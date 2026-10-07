保证HADOOP_CONF_DIR与YARN_CONF_DIR正确

## 连接到YARN
- 注意先要切断standalone`bin/stop-all.sh`
```bash
bin/pyspark --master yarn
```

