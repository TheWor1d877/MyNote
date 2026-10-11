## inotify是什么
inotify 是 Linux 内核提供的一种文件系统事件监控机制。

让应用程序可以"订阅"某个文件或目录的变化，当文件被创建、修改、删除、移动时，内核主动通知应用程序，而不需要应用程序不断轮询（polling）去检查

k8s要实现“声明式资源管理”的自我修复功能，需要重度依赖inotify

#### 两个关键的内核参数
```text
fs.inotify.max_user_instances   # 单个用户可创建的 inotify 实例数上限
fs.inotify.max_user_watches     # 单个用户可注册的 watch 总数上限
```

Ubuntu 桌面版的默认值通常偏低：
- max_user_instances 默认 128
- max_user_watches 默认 8192 或 65536

而一个 Kind 多节点集群（1 control-plane + 2 worker），每个节点容器内部都要跑 kubelet + containerd + cAdvisor，三个节点叠加很容易就把这个额度耗尽。

## 问题还原
```text
1. 执行 kind create cluster --config kind-config.yaml
   ↓
2. Kind 创建 3 个 Docker 容器（1 cp + 2 worker）
   ↓
3. control-plane 先启动，kubelet 成功运行，节点 Ready
   ↓
4. 两个 worker 容器启动，kubelet 开始初始化
   ↓
5. kubelet 要监控大量文件，向内核申请 inotify 实例
   ↓
6. 宿主机 inotify 资源已被前面的容器 + 系统进程耗尽
   ↓
7. 内核返回 "too many open files"
   ↓
8. kubelet 初始化失败，进程退出（exit code 1）
   ↓
9. systemd 尝试重启 kubelet，但资源仍然不够，反复失败
   ↓
10. worker 节点永远无法注册到集群
```

通过日志查看出问题所在：
```bash
docker exec -it traincache-worker bash
root@traincache-worker:/#
journalctl -u kubelet -n 80 --no-pager
```
问题：
```text
E1008 12:38:44.925204 dynamic_cafile_content.go:170] "Failed to watch CA file, will retry later" 
    err="error creating fsnotify watcher: too many open files"

E1008 12:38:44.937033 file_linux.go:59] "Unable to read config path" 
    err="unable to create inotify: too many open files" path="/etc/kubernetes/manifests"

E1008 12:38:44.937982 plugins.go:585] "Error initializing dynamic plugin prober" 
    err="flex volume: fsnotify watcher init: too many open files"

E1008 12:38:44.939636 dynamic_serving_content.go:144] "Failed to watch cert and key file, will retry later" 
    err="error creating fsnotify watcher: too many open files"

E1008 12:38:44.940428 manager.go:340] Registration of the raw container factory failed: 
    inotify_init: too many open files

E1008 12:38:44.940437 kubelet.go:1833] "Failed to start cAdvisor" 
    err="inotify_init: too many open files"

systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
systemd[1]: kubelet.service: Failed with result 'exit-code'
```

## 出现的问题
```bash
$ kubectl get nodes
NAME                       STATUS   ROLES           AGE   VERSION
traincache-control-plane   Ready    control-plane   8m53s v1.37.0
```
只看到 1 个节点，而不是配置的 3 个。

## 解决方案
核心操作：提高宿主机 inotify 限制

临时生效（当前会话）：
```bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
```

永久生效：
编辑 /etc/sysctl.conf，末尾添加：
```text
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

## 关键教训
Kind 的 ✓ 不等于成功	它只保证容器启动，不保证 kubelet Ready
多节点比单节点更吃资源	3 个节点 = 3 份 kubelet + containerd + cAdvisor
"时好时坏"往往是资源竞争	共享资源池的典型特征
inotify 是 K8s 的隐形依赖	本地跑 K8s 集群前应先调高限制