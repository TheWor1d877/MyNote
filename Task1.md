4.6 HPA

3.5   gRPC 与 K8s 集成（Service、Ingress、TLS）


9.1   Makefile 统一构建命令

9.2   单元测试 + 覆盖率报告

9.3   GitHub Actions CI（测试、构建、推送镜像）



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


### 第一层：核心基石（必深，T的那一竖）
这是你的“护城河”，是你未来面试和工作的核心竞争力。

| 领域 | 关键技术 | 掌握程度 | 时间预估 |
| :--- | :--- | :--- | :--- |
| **AI Infra** | vLLM/TGI/SGLang 原理与二次开发<br>PyTorch/TensorRT 推理优化<br>PD分离/Continuous Batching<br>KV Cache 管理与量化 | **深入源码级**<br>能贡献开源或自研小工具 | 3-4个月 |
| **K8s 云原生** | K8s 核心源码分析<br>Operator/CRD 开发<br>调度器扩展（Volcano/Kueue）<br>GPU Operator 底层原理 | **生产级开发能力**<br>能写Operator解决实际问题 | 2-3个月 |

### 第二层：支撑支柱（必广，T的那一横）
这些不需要源码级深入，但必须**理解原理并能熟练应用**。

| 领域 | 关键技术 | 掌握程度 | 时间预估 |
| :--- | :--- | :--- | :--- |
| **分布式存储** | CSI 机制与开发<br>JuiceFS/Ceph/Longhorn 架构对比与选型<br>POSIX/FUSE 接口理解<br>Raft 共识协议原理 | **原理级理解 + 能开发 CSI 插件** | 1.5-2个月 |
| **DevOps/平台** | GitOps（ArgoCD）源码与扩展<br>可观测性（OpenTelemetry）<br>CI/CD 流水线设计模式 | **平台级整合能力**<br>能搭建统一的开发者平台 | 1-1.5个月 |
| **向量数据库** | Milvus/Qdrant 架构<br>HNSW/IVF 索引原理<br>与推理服务集成模式 | **选型与集成能力**<br>能根据场景选择合适的方案 | 1个月 |

### 第三层：视野扩展（可选，但加分）
这些是让你从“做出来”到“做得更好”的进阶知识。

| 领域 | 关键技术 | 学习方式 |
| :--- | :--- | :--- |
| **AI 芯片架构** | GPU/NPU/TPU 架构差异<br>显存层次与访存模型 | 泛读论文 + 技术博客 |
| **高性能网络** | RDMA/RoCE/NVMe-over-Fabric<br>集合通信（NCCL） | 按需深挖 |
| **MLOps/LLMOps** | 模型版本管理（DVC）<br>实验追踪（MLflow）<br>提示词工程框架 | 了解概念即可 |
