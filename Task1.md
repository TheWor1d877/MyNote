第3章：gRPC 通信框架（分布式系统基石）
目标：构建高可靠节点间通信层  
产出：支持 Unary/Stream RPC 的存储服务接口
小节   内容
3.1   Protobuf 定义规范（消息、服务、版本兼容）

3.2   GoLand 中 protoc 自动生成代码（集成技巧）

3.3   gRPC 服务端实现（拦截器、错误码、超时控制）

3.4   流式 RPC（大文件分块上传/下载）

3.5   gRPC 与 K8s 集成（Service、Ingress、TLS）

💼 面试价值：对标 etcd/TiKV，展示分布式通信能力

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

💼 面试价值：证明能交付云原生应用，不只写代码

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


第7章：性能剖析与调优（pprof 实战）
目标：定位并解决性能瓶颈  
产出：火焰图 + 优化报告
小节   内容
7.1   pprof 启用与 CPU 剖析（GoLand 可视化）

7.2   内存逃逸分析（go build -gcflags="-m"）

7.3   Goroutine 泄漏检测

7.4   Mutex 锁竞争分析

7.5   GC 调优（GOGC、堆大小控制）

💼 面试价值：展示“不只是写功能，还能调性能”

第8章：网络协议栈与自定义协议
目标：理解 TCP/HTTP 底层，设计高效协议  
产出：自定义二进制协议解析器
小节   内容
8.1   TCP 连接管理（连接池、心跳）

8.2   HTTP/2 与 gRPC 关系

8.3   自定义二进制协议设计（Header + Body）

8.4   编解码优化（避免内存拷贝）

8.5   网络超时与重试策略


💼 面试价值：深入网络层，区别于 CRUD 工程师

第9章：自动化运维与 CI/CD
目标：实现开发到部署自动化  
产出：GitHub Actions 流水线
小节   内容
9.1   Makefile 统一构建命令

9.2   单元测试 + 覆盖率报告

9.3   GitHub Actions CI（测试、构建、推送镜像）

9.4   ArgoCD/GitOps 自动部署到 K8s

9.5   日志收集（Loki + Promtail）

💼 面试价值：DevOps 能力，提升团队效率

10.gin框架

## 核心

| 类别 | 你需要掌握的技术 | 为什么 |
| :--- | :--- | :--- |
| 1. 存储系统 | POSIX 文件系统、S3 兼容接口、自研存储协议 | AI 框架（PyTorch/TensorFlow）通过这些读数据 |
| 2. 缓存/加速层 | 本地 SSD 缓存、内存缓存、分层存储（Tiered Storage） | 解决远程存储慢的问题 |
| 3. 云原生集成 | Kubernetes CSI Driver、Operator、Pod 挂载 | 让训练任务能像用本地盘一样用你的存储 |
| 4. 性能可观测性 | 暴露 Prometheus 指标（Counter/Gauge） | 证明你的方案有效：“GPU 等待时间下降了 80%” |
| 5. 数据加载优化 | 与 PyTorch DataLoader 协同、预取（prefetch）、批处理 | 减少 I/O 阻塞 |

