```bash
# 创建环境激活脚本
mkdir -p /export/server/miniconda3/envs/pyspark/etc/conda/activate.d
cat > /export/server/miniconda3/envs/pyspark/etc/conda/activate.d/set_java.sh << 'EOF'
#!/bin/bash
export ORIGINAL_JAVA_HOME=$JAVA_HOME
export ORIGINAL_PATH=$PATH
export JAVA_HOME=/export/server/jdk17
export PATH=/export/server/jdk17/bin:$PATH
EOF

# 创建环境停用脚本
mkdir -p /export/server/miniconda3/envs/pyspark/etc/conda/deactivate.d
cat > /export/server/miniconda3/envs/pyspark/etc/conda/deactivate.d/unset_java.sh << 'EOF'
#!/bin/bash
export JAVA_HOME=$ORIGINAL_JAVA_HOME
export PATH=$ORIGINAL_PATH
unset ORIGINAL_JAVA_HOME
unset ORIGINAL_PATH
EOF

chmod +x /export/server/miniconda3/envs/pyspark/etc/conda/activate.d/set_java.sh
chmod +x /export/server/miniconda3/envs/pyspark/etc/conda/deactivate.d/unset_java.sh
```
