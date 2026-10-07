nodeAffinity 是 Kubernetes 中将 Pod 调度到特定节点的一种机制，相当于给 Pod 设置"择偶标准"。
## 使用场景
- 你的应用需要 SSD 硬盘，但集群里只有部分节点有 SSD
- GPU 任务必须跑到有显卡的节点上
- 高优先级服务要跑到性能更好的节点上
- 某些服务需要跑到特定机架/机房的机器上

## 使用方法
- requiredDuringScheduling	硬性要求，必须满足	如果找不到，Pod 一直 Pending，永不调度
- preferredDuringScheduling	软性偏好，尽量满足	找不到也能调度到其他节点

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: nginx
      image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬性规则
        nodeSelectorTerms:  # 选择器条件（可以多个，OR 关系）
          - matchExpressions:  # 具体匹配规则（必须全部满足，AND 关系）
              - key: disktype   # 节点标签的 key
                operator: In    # 操作符
                values:         # 值列表
                  - ssd
                  - nvme
      preferredDuringSchedulingIgnoredDuringExecution:  # 软性规则
        - weight: 100  # 权重（1-100）
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values:
                  - us-east-1a
```

#### 操作符
操作符	             含义	
In	                     标签值在列表中	
NotIn	             标签值不在列表中	
Exists	             存在该标签（无论值）	
DoesNotExist	 不存在该标签	
Gt	                 值大于（数值比较）
Lt	                     值小于（数值比较）

