**Rank 讲解**

**1. 为什么需要 rank**

数据并行训练要把同一个模型复制到多块 GPU 上，每块 GPU 处理不同的数据批次，再通过梯度通信保持模型一致。为了让这些进程能互相识别、通信、分工，需要给每个进程一个唯一编号，这个编号就是 rank。

**2. rank 与 world_size**

- rank：数据并行训练中每个 GPU 进程的编号，从 0 开始。
- world_size：参与训练的进程总数，即 rank 的取值范围是 0 到 world_size-1。
- 例：8 卡训练时 world_size=8，rank 分别为 0 到 7。

rank 0 通常承担协调角色，比如打印日志、保存汇总信息；其他 rank 负责各自的训练计算。

**3. rank 从哪来**

使用 torchrun 启动时，torchrun 负责拉起所有进程，并把 RANK、WORLD_SIZE 等环境变量注入每个进程。训练代码通过读取这些环境变量知道自己是谁、总共有多少进程，再据此初始化通信组。

**4. rank 在 checkpoint 链路上的落点**

保存 checkpoint 时，每个 rank 各写一份 manifest，存放路径为目录名 `rank-<rank>/`。恢复时，每个进程读取自己编号对应的目录。这样每个进程的保存与恢复互不干扰，路径由 rank 唯一确定。

**5. 单 Pod 多 rank 的问题**

一个 Pod 内可以运行多个 rank。8 卡一台机器用 torchrun 默认用法，就是单 Pod 起 8 个进程，rank 0 到 7 同在一个 Pod 内。

问题在于 checkpoint 的完整性判定。若目录与 Pod 一一对应，判定「world_size 份 manifest 都在」才成立。多 rank 时每个训练进程要往自己的子目录写，训练框架不一定配合，判定就不再可靠。因此一期关闭了多 rank，代价是 torchrun 默认用法跑不了。扩展方案是把 rank 目录下再分一层进程序号、manifest 增加 local_rank 字段，归入 version 2。

**6. 现状行为**

一个 Pod 两个 rank 时，准入将其标记为不支持，并拒绝注入 sidecar。