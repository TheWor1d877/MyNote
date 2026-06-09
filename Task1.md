## 二
第4章：云原生开发与部署
目标：将服务容器化并部署到 K8s  
产出：Docker 镜像 + Helm Chart + ConfigMap
小节   内容
4.1   多阶段 Dockerfile 编写（减小镜像体积）

4.2   GoLand Docker 插件构建与调试

4.3   K8s Deployment 编写（探针、资源限制）

4.4   Helm Chart 封装（values.yaml 参数化）

4.5   ConfigMap 与 Secret 管理配置

4.6 HPA

3.5   gRPC 与 K8s 集成（Service、Ingress、TLS）




第9章：自动化运维与 CI/CD
目标：实现开发到部署自动化  
产出：GitHub Actions 流水线
小节   内容
9.1   Makefile 统一构建命令

9.2   单元测试 + 覆盖率报告

9.3   GitHub Actions CI（测试、构建、推送镜像）

9.4   ArgoCD/GitOps 自动部署到 K8s

9.5   日志收集（Loki + Promtail）

openTelemetry


## 三

第5章：分布式存储核心理论
目标：理解 CAP、一致性、复制协议  
产出：系统设计文档 + Raft 共识模拟
小节   内容
5.1   CAP 定理与权衡（AP vs CP 系统）

5.2   一致性模型（Linearizability, Sequential）

5.3   分片（Sharding）与副本（Replication）策略

5.4   WAL（Write-Ahead Log）原理

5.5   Raft 共识算法简化实现（选主、日志复制）

💼 面试价值：回答“如何设计一个分布式存储系统”


## 核心

| 类别 | 你需要掌握的技术 | 为什么 |
| :--- | :--- | :--- |
| 1. 存储系统 | POSIX 文件系统、S3 兼容接口、自研存储协议 | AI 框架（PyTorch/TensorFlow）通过这些读数据 |
| 2. 缓存/加速层 | 本地 SSD 缓存、内存缓存、分层存储（Tiered Storage） | 解决远程存储慢的问题 |
| 3. 云原生集成 | Kubernetes CSI Driver、Operator、Pod 挂载 | 让训练任务能像用本地盘一样用你的存储 |
| 4. 性能可观测性 | 暴露 Prometheus 指标（Counter/Gauge） | 证明你的方案有效：“GPU 等待时间下降了 80%” |
| 5. 数据加载优化 | 与 PyTorch DataLoader 协同、预取（prefetch）、批处理 | 减少 I/O 阻塞 |
