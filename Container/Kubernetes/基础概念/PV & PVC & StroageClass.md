## Introduction
PVC是用户的存储申请，需要什么样子的存储
PV是集群级别的存储资源对象，描述存储的大小、访问模式、回收策略等
StorageClass负责动态生成PV，StorageClass 描述"用哪个 provisioner、什么参数、什么回收策略去创建存储"，PVC 里指定 storageClassName，供给器就会按需动态创建 PV 再绑定。

K8s 的绑定控制器负责把 PVC 匹配到一个满足条件的 PV 上，绑定是一对一的，绑完这块 PV 就被这个 PVC 独占。
Pod中不需要引用PV，而是引用PVC，这样就实现了声明式解耦——用户只声明需求，系统负责满足。

用户创建PVC，检查PVC是否指定了StroageClass，如果没有，就用静态的PV资源，否则k8s调用StroageClass的Provisioner,自动创建PV


## PV 与ConfigMap的区别
PV持久化存储数据，在单节点上面类似于docker中的Volumn，主要用于写入 + 持久保存数据，重启Pod之后数据还在

ConfigMap 用于 只读注入（内容由用户预先定义，运行时通常不修改）

| 资源        | 底层存储                                                             | 是否可写                              | 生命周期                |
| --------- | ---------------------------------------------------------------- | --------------------------------- | ------------------- |
| PV        | 可以是：<br>• 本地磁盘（hostPath）<br>• 网络存储（NFS, Ceph）<br>• 云盘（EBS, Disk） | 可读写                               | 独立于 Pod，可长期存在       |
| ConfigMap | 存储在 etcd 中，挂载时通过 tmpfs（内存文件系统）暴露给容器                              | 默认只读<br>（可通过 subPath 实现部分可写，但不推荐） | 随命名空间管理，与 Pod 无直接绑定 |

当你把 ConfigMap 挂载到 Pod 时，K8s 实际是在容器内创建一个 tmpfs 文件系统，把 etcd 中的 key-value 映射为文件。这意味着：
修改挂载的 ConfigMap 文件 不会同步回 etcd
重启 Pod 后，ConfigMap 内容会重置为原始值


## 模板
#### PV
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  # 【核心】存储容量
  capacity:
    storage: 10Gi  # 必须指定，单位 Gi/Mi 等

  # 【核心】访问模式（决定能否跨节点）
  accessModes:
    - ReadWriteOnce   # RWO: 单节点读写（最常见）
    - ReadOnlyMany    # ROX: 多节点只读
    - ReadWriteMany   # RWX: 多节点读写（需特殊后端）

  # 【关键】回收策略（Pod 删除后 PV 如何处理）
  persistentVolumeReclaimPolicy: Retain | Delete | Recycle
    # Delete: 自动删除底层存储（如云盘）
    # Retain: 保留数据，需手动清理（用于备份）
    # Recycle: 已废弃（旧版 NFS 清空）

  # 【核心】存储类型（决定底层实现）
  hostPath:           # 仅单节点测试用！
    path: /mnt/data
  nfs:                # 多节点共享存储
    server: nfs-server
    path: /exports/data
  csi:                # 现代标准（Ceph, AWS EBS, 阿里云等）
    driver: ebs.csi.aws.com
    volumeHandle: vol-xxxx

  # 【重要】绑定到特定 StorageClass（静态 PV 时可选）
  storageClassName: fast-ssd
```
accessModes 不是“建议”，而是存储后端的能力声明。你不能在 AWS EBS 上声明 ReadWriteMany，因为 EBS 本身不支持。
hostPath 绝对不能用于生产多节点集群！它只在当前节点有效，Pod 调度到其他节点就挂了
<span style="color:rgb(221, 85, 85)">在现代Kubernetes环境中，YAML文件里几乎从来不写PV（持久卷）。PV完全由StorageClass自动创建。</span>


#### PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  # 【必须】访问模式（必须被某个 PV 支持）
  accessModes:
    - ReadWriteOnce

  # 【必须】资源请求
  resources:
    requests:
      storage: 5Gi  # 必须 ≤ 某个 PV 的 capacity

  # 【关键】指定 StorageClass（触发动态供给）
  storageClassName: fast-ssd  # ← 若省略，使用 default SC

  # 【高级】选择器（匹配带特定 label 的 PV）
  selector:
    matchLabels:
      env: prod
```

如果指定了 storageClassName → 尝试动态创建 PV
   否则 → 在所有未绑定的静态 PV 中找匹配项（容量 ≥、accessMode 兼容、label 匹配）
#### StroageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # 设为默认
provisioner: rook-ceph.rbd.csi.ceph.com  # CSI 驱动名

# 【核心】传递给后端存储的参数
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering

# 【重要】回收策略（覆盖 PV 的默认值）
reclaimPolicy: Delete

# 【高级】是否允许卷扩展
allowVolumeExpansion: true

# 【多节点关键】挂载选项
mountOptions:
  - discard
```
Provisioner是灵魂，决定了如何创建底层存储，常见的的provisioner：
kubernetes.io/aws-ebs → AWS 云盘
diskplugin.csi.alibabacloud.com → 阿里云云盘
rook-ceph.rbd.csi.ceph.com → Ceph RBD 块存储
nfs.csi.k8s.io → NFS

前提是本地存储了相关的CSI

## 多节点数据存储的核心原理
PV 必须基于“网络存储”或“分布式存储”，而非本地磁盘

| 存储类型   | 是否支持多节点     | 典型实现                     | 适用场景                               |
| ------ | ----------- | ------------------------ | ---------------------------------- |
| 本地存储   | 否           | `hostPath`, `local` PV   | 单节点开发、性能极致优化（如 Local PV + Pod 亲和性） |
| 网络块存储  | 是（但通常 RWO）  | AWS EBS, 阿里云云盘, Ceph RBD | 数据库（MySQL, PostgreSQL）             |
| 网络文件存储 | 是（支持 RWX）   | NFS, CephFS, GlusterFS   | Web 服务器共享静态文件、Jenkins workspace    |
| 对象存储   | 是（但非 POSIX） | S3, OSS                  | 日志归档、大文件存储（需应用改造）                  |

