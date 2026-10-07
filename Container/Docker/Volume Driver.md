如果想扩展 Docker 的存储能力，而不是完全替换镜像层管理方式，Docker 提供了另一种扩展机制——Volume Driver 插件系统-8。
Volume Driver 允许你实现自定义的数据卷存储后端，比如：

- 数据卷存储在 NFS 服务器上
- 将数据卷存储在 AWS S3、阿里云 OSS 等对象存储中-7
- 将数据卷存储在 SSH 可访问的远程主机上

当你使用 volume 挂载时，挂载点目录就完全脱离了 OverlayFS 的管理，由底层的原生文件系统（ext4、xfs 等）直接管理。这就是为什么能避免大文件修改的 Copy-on-Write 代价。

```bash
容器视角: /data/large-file.txt
    ↓
OverlayFS merged 目录
    ├── upper/ (可写层 - OverlayFS 管理)
    └── lower/ (只读层 - OverlayFS 管理)
```

