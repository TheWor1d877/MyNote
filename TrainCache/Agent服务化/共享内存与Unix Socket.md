##  共享内存在Linux上面的实现
本质: 文件+映射
```text
shm_open("/my_shm")  ≈  open("/dev/shm/my_shm")   // 一个 tmpfs 上的文件
+ mmap(... MAP_SHARED ...)                          // 映射进地址空间
```

`shm_open`将共享内存伪装成一个文件来管理，直接复用文件系统现成的权限
可以像文件系统一样进行打开、关闭、删除、命名等操作
/dev/shm 就是一个 tmpfs 挂载点（内存文件系统，重启即失）

共享内存需要使用相对偏移量，因为在不同你的进程中可能虚拟地址是不同的
共享内存只管共享，不管多进程之间的协调，必须配合信号量、互斥锁使用
shm_unlink 只是删除名字，已映射的进程仍能用；直到最后一个进程 munmap，内存才真正释放。这一点很像 unlink 一个已被打开的文件

完整流程：
```c
// 1. 创建/打开共享内存对象，返回 fd
int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);

// 2. 设置大小（新建对象大小为 0，必须显式设）
ftruncate(fd, size);

// 3. 映射到本进程地址空间
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE,
               MAP_SHARED, fd, 0);

// 4. 映射后 fd 可关闭，不影响映射
close(fd);

// 5. 像操作普通指针一样读写 p（但多进程要加锁！）

// 6. 清理
munmap(p, size);          // 解除映射
shm_unlink("/my_shm");    // 删除对象名
```

Go 标准库没有直接封装 shm_open，但可以通过 `golang.org/x/sys/unix` 手动调用系统调用来实现 POSIX 共享内存。

## TrainCache中DataLoader钩子的决策权衡
TrainCache 的 Agent 需要跟Python Sidecar 和训练程序中的 DataLoader 钩子这两个组件通讯
Sidecar与Agent之间选择是共享内存,数据类型是消费数据队列，为了高效共享持续更新的队列
而DataLoader与Agent之间是Unix Socket，数据类型数进度与状态事件，他们之间发送简单的事件通知

- 为什么钩子使用 Unix Socket？
数据量不同。 Sidecar 需要持续发布消费顺序，适合使用共享内存减少数据复制；钩子主要发送少量事件，没有明显必要引入共享内存。
实现复杂度不同。 Unix Datagram Socket 可以直接发送事件。共享内存则需要额外设计队列结构、读写同步、溢出处理和进程重启后的恢复机制。
职责不同。 共享内存 ring 用于共享持续更新的消费顺序；Socket 用于传递进度和状态事件。两者针对不同需求选择不同的 IPC 机制，更有利于控制系统复杂度。

如果钩子也使用共享内存，就需要为事件通知额外维护一套共享内存队列，包括：
生产者与消费者之间的同步。
队列满时的处理策略。
进程重启后的状态恢复。
跨容器共享内存的部署与权限配置。
对于少量事件通知，这些额外机制可能得不偿失。