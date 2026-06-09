```txt
1. Podman CLI 启动
   ↓
2. 创建 User Namespace
   - 将你的 UID 映射到容器内的 root
   ↓
3. 启动 slirp4netns (网络)
   - 设置虚拟网络接口
   - 配置端口转发
   ↓
4. 启动 fuse-overlayfs (存储)
   - 挂载镜像层
   - 创建可写层
   ↓
5. 启动 conmon (容器监控器)
   - 监控容器生命周期
   - 捕获日志输出
   ↓
6. 启动 runc/crun
   - 在 namespace 中执行容器进程
   ↓
7. Nginx 进程运行
   - 容器内以为自己是 root
   - 宿主机上实际是你的用户权限
```