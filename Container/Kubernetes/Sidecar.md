边车容器是与主应用程序容器在同一 Pod 内一起运行的辅助容器。这些容器通过提供额外的服务或功能（如日志记录、监控、安全或数据同步）来增强或扩展主应用容器的功能， 而无需直接修改主应用程序代码。


使用 Kubernete`SidecarContainers` 特性允许 **Init 容器**设置 `restartPolicy: Always`，使其像 Sidecar 容器一样：
- 在主应用容器启动**之前**就开始运行
- 主容器启动**之后继续在后台运行*
- 典型场景：日志收集、代理、流量劫持等 Sidecar 模式s 对边车容器的原生支持可以带来以下几个好处：

#### yaml文件写法
```yaml
spec:
  initContainers:
    - name: log-collector       # 这是 Sidecar 容器
      image: fluent/fluent-bit:3.0
      restartPolicy: Always     # 【最关键的一行】加上这行，它就变身了
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  # 这里放主业务容器
  containers:
    - name: main-app
      image: my-app:v1
	    volumeMounts:
            - name: shared-logs
            mountPath: /var/log/app
    volumes:
        - name: shared-logs
        emptyDir: {}   # 通过这个共享文件夹互通日志
      ......     
```
#### 优点
启动顺序有保证了：就像代码写的那样，Sidecar 必须在主容器之前就绪。比如日志收集器先启动好了，主程序才开始打印日志，保证一条都不会丢。
   
Job 能正常结束了：如果你跑一个一次性任务（比如数据迁移），以前主程序跑完了，Sidecar 还在那儿死等，任务永远结束不了。现在主程序一停，Sidecar 会被系统优雅地终止，Job 状态立刻变 Completed

自带健康检查：系统会自动帮你守护这个 Sidecar。万一它挂了（比如日志采集进程崩溃），Kubernetes 会自动帮你重启它，不用你操心。

#### Sidecar与DeamonSet的区别

| 维度 | DaemonSet | Sidecar |
|------|-----------|--------|
| 谁需要它？ | 集群管理员 / 运维：提供基础设施能力 | 应用开发者：增强单个应用的功能 |
| 生命周期绑定 | 绑定 节点（Node） | 绑定 应用 Pod |
| 部署控制权 | 集群统一部署，应用无感知 | 应用 YAML 中显式定义，开发者可控 |
| 资源归属 | 属于平台层 | 属于应用层 |
