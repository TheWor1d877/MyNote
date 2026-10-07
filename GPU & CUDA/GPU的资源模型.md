GPU本质上就是一种系统资源：计算资源池
讨论问题：
- GPU如何被容器化暴露
- GPU如何拿到数据
- GPU显存如何管理

在k8s中，GPU不是普通的CPU内存那些“可压缩资源”，而是不可压缩的独占资源
我们需要 做MIG（Multi-Instance GPU）——把一块物理GPU切成多个隔离的实例，每个实例有独立的SM、显存、缓存，K8s可以按"切片"粒度调度

## GPU的数据通路
数据从存储到GPU的数据通路
```text
分布式存储（Ceph/Alluxio）
    ↓ NFS/RDMA
宿主机内存（Page Cache）
    ↓ PCIe（~16-32 GB/s）
GPU显存（HBM，~1.5-3 TB/s）
    ↓ NVLink（如果多卡）
SM计算单元
```
这条路径上面PCIe是瓶颈

优化路径：
- GDS：让GPU直接从NVMe/分布式存储读数据，绕过CPU和宿主机内存，把PCIe拷贝次数从2次降到1次
- 数据预取与缓存：在训练循环里，当前step用数据A时，异步预取下一个step的数据B到显存或host pinned memory
- 存储格式优化：用RecordIO、TFRecord、Parquet等列式/打包格式，减少小文件I/O，提升顺序读吞吐

## GPU显存管理
推理的时候，请求的KV Cache随着sequence lenth线性增长，
传统做法：预分配连续内存，但是会导致严重的内存碎片，多并发的请求下，显存不够用，吞吐量低下
vLLM的做法（pagedAttention）：
把KV Cache切成固定大小的"页"（page），类似操作系统的虚拟内存分页
每个请求维护一个逻辑块表（block table），映射逻辑token位置到物理显存页
物理页可以非连续、可以共享（比如多个请求的prompt部分相同，可以CoW共享