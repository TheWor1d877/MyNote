NCCL 的集合操作（allreduce、broadcast 等）必须挂在一个通信器上才能执行

通信器内部给每个参与者分配一个 rank，取值 0 到 nranks - 1，同一个通信器内不重复
nranks 就是通信器内 rank 的总数，也就是参与者的数量。

## ncclCommInitRank
Comm 是 communicator 的缩写
```c
ncclResult_t ncclCommInitRank(
    ncclComm_t* comm,      // 输出：建好的通信器句柄
    int nranks,             // 这份通信契约一共几个参与者
    ncclUniqueId commId,    // 所有参与者共享的“同一份合同”标识
    int rank                // 当前调用者在这份合同里的编号
);
```
 ncclUniqueId：一个固定大小的字节数组，大小由 NCCL_UNIQUE_ID_BYTES 定义，当前为 128 字节。它由 ncclGetUniqueId 生成，内容包含 NCCL 建立 bootstrap 连接所需的寻址信息。

 bootstrap本质上是 NCCL 在初始化阶段临时搭建的一套 CPU 侧控制面通信网络，用于在 NVLink、InfiniBand 等高速数据通道建立之前，各个 rank 必须先通过一个最基础的通道互相认识、交换地址和硬件信息。

 同一个通信器内所有 rank 必须持有逐字节相同的 ncclUniqueId。NCCL 靠这个值让互不认识的参与者找到对方，值不同就无法进同一个通信器

通信器内部对象 ncclComm 创建后，直接把这两个值存进字段，后面所有操作都读这里:
comm->rank  = rank;
comm->nRanks = nranks;

## ncclUniqueId/Rank的生成与分配
`ncclCommInitAll` 与 `ncclCommInitRank`

ncclCommInitAll 是 ncclCommInitRank 的封装。它内部做的就是：生成一个 ncclUniqueId，然后对每个设备循环调用 ncclCommInitRank。

单进程管理多卡用前者，pthread 和 MPI 都用后者，但拿唯一 ID 的途径不同

三种方式的差别不是初始化逻辑不同，而是参与者之间的可见性不同，导致 id 传递途径不同
#### 单进程多卡 `ncclCommInitAll`
这个函数同时完成“生成 id”和“初始化多个 rank”两件事。调用者只给设备列表和数量，不管 id。

这种方式只适用于同一进程能看到全部设备的情况。它省掉了手动管理 id 的步骤，代价是失去了对不同 rank 初始化时序的控制。

#### pthread：主线程生成，参数传递
主线程调一次 ncclGetUniqueId，把得到的 ncclUniqueId 和每个线程对应的 rank 作为线程参数传进去。每个线程在自己的上下文里调 ncclCommInitRank

线程共享地址空间，所以 id 可以直接当参数传，每个工作线程管自己的卡

每个工作线程做两件事：cudaSetDevice 绑自己的卡，ncclCommInitRank(&comm, nranks, id, my_rank) 建自己的通信器。

#### MPI：多进程，id 靠广播传
**Message Passing Interface**，消息传递接口

进程之间不共享地址空间，参数传递那套用不了。rank 0 调 ncclGetUniqueId 生成 id，用 MPI_Bcast 广播到所有进程，然后每个进程各自调 ncclCommInitRank

id 的传递路径是：rank 0 的内存 → MPI 网络 → 每个进程的内存。这是它和 pthread 的根本差别。

## 查询
NCCL提供三个基础查询函数：

函数	                              查询内容                                        对应创建时的输入
ncclCommCount	     通信器内 rank 总数	                     nranks 参数
ncclCommUserRank	当前调用者在该通信器内的 rank	 rank 参数
ncclCommCuDevice	该通信器绑定的 CUDA 设备编号	 cudaSetDevice 设定的设备

单进程 ncclCommInitAll 场景下，拿到的是通信器数组comm，遍历每个元素分别查询。ncclCommCuDevice 在这里最有用：你可以确认 comms_i 确实绑到了 devlist_i 对应的 GPU，而不是因为 cudaSetDevice 遗漏导致绑错。

pthread 和 MPI 场景下，每个线程或进程持有自己的通信器，查询放在 ncclCommInitRank 返回之后、进入集合操作之前。典型用途是打印当前 rank 和绑定的设备，确认 rank 映射与预期一致。调试 rank 错位时，这是成本最低的验证手段。

上面三个函数查的是“身份信息”，不反映通信器运行状态。真正需要周期性检查的是 ncclCommGetAsyncError,网络错误等通信故障不会从 ncclCommCount 这类函数反映出来，而是通过异步错误通道报告。如果某个 NCCL 操作返回 ncclInProgress，必须持续查询通信器状态直到它变为 ncclSuccess


## 销毁
销毁操作对应 ncclCommDestroy，它负责释放通信器占用的资源。三种创建方式在销毁上的差别，只在于你手里有几个通信器句柄，以及这些句柄分布在哪些执行单元里。

调用 ncclCommDestroy(comm) 会释放分配给该通信器的资源，并且等待该通信器上未完成的操作结束，然后才执行销毁。它不会与其他 rank 做同步，所以不需要像集合操作那样使用 group 语义

#### 三种模式下的销毁方案
ncclCommInitAll（单进程多卡）：拿到的是 comms 数组，循环对每个元素调用 ncclCommDestroy。NCCL 官方示例的做法就是在程序末尾遍历数组销毁。-2-4

pthread：每个工作线程持有自己的 ncclComm_t，在线程退出前由该线程自己调用 ncclCommDestroy。主线程不持有这些句柄，不负责销毁。

MPI：每个进程持有自己的通信器，在进程结束前各自调用 ncclCommDestroy。进程之间不需要协调销毁顺序，因为该函数不做跨 rank 同步。