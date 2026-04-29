第2章：高性能日志与监控系统（生产级可观测性）
目标：实现结构化日志 + 指标采集 + 可视化  
产出：带 zap + Prometheus + Grafana 的可观测存储服务
小节   内容
2.1   zap 结构化日志入门（JSON、字段、性能优势）

2.2   多文件日志路由（Info/ERROR 分离、按模块拆分）

2.3   日志轮转与容器化适配（lumberjack + stdout）

2.4   Prometheus 指标埋点（Counter, Gauge, Histogram）

2.5   Grafana 看板搭建（QPS、延迟、错误率监控）

💼 面试价值：体现 SRE 思维，回答“如何排查线上问题”

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

第6章：自研存储引擎（核心项目）
目标：实现本地块存储 + WAL + 内存池  
产出：可读写的 chunking 存储引擎（简历项目）
小节   内容
6.1   块存储接口设计（WriteChunk, ReadChunk）

6.2   WAL 日志实现（持久化、回放）

6.3   内存池优化（sync.Pool 减少 GC）

6.4   并发安全（RWMutex / 分片锁）

6.5   性能压测（wrk2 + pprof 分析）

💼 面试价值：核心简历项目，展示底层编码能力

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

第10章：项目整合与简历包装
目标：将所有模块整合为完整项目  
产出：GitHub 仓库 + README + 架构图 + 面试话术
小节   内容
10.1   项目架构设计（gRPC + WAL + K8s）

10.2   README 编写（亮点、架构图、启动指南）

10.3   性能对比（vs MinIO/Ceph）

10.4   面试高频问题准备（设计、优化、故障）

10.5   扩展方向（S3 兼容、纠删码）

💼 面试价值：直接用于简历和项目介绍

go语言学习清单，咱们按照这个来学习
