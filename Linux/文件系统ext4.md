文件系统就是磁盘上面的结构化布局加上管理算法

## ext4的组成
执行：
```bash
sudo mkfs.ext4 /dev/sdb1
```

ext4会向分区中写入以下结构
```bash
EXT4 文件系统布局：
┌─────────────────────────────────────────────────────────────┐
│ 超级块 (Superblock)                                         │
├─────────────────────────────────────────────────────────────┤
│ 块组描述符表 (Block Group Descriptor Table)                 │
├─────────────────────────────────────────────────────────────┤
│ 数据块位图 (Data Block Bitmap)    ┐                          │
│ inode位图 (inode Bitmap)         ├─ 块组0                   │
│ inode表 (inode Table)            │                          │
│ 数据块 (Data Blocks)             ┘                          │
├─────────────────────────────────────────────────────────────┤
│ 数据块位图                    ┐                              │
│ inode位图                     ├─ 块组1                      │
│ inode表                       │                              │
│ 数据块                        ┘                              │
└─────────────────────────────────────────────────────────────┘
```
#### Superblock
```bash
sudo dumpe2fs /dev/sdb1 | head -20
```
如果 superblock 损坏了，整个文件系统就“失忆”了！所以 ext4 会存**多个备份**（在 block group 0, 1, 3, 5...）
#### inode table
每个 inode 在磁盘上有固定大小（通常是 256 字节），连续存放。
inode 0：保留（不用）
inode 1：坏块列表（legacy）
inode 2：根目录 /
inode 3~10：日志、ACL 等系统用
inode 11+：普通文件/目录
```bash
ls -id /
# 输出：2 /
```
#### Data Block —— 文件内容的家
- 普通文件的内容存在这里。
- 目录文件的内容（即 (filename, inode#) 列表）也存在这里！
看目录文件的内容（原始字节）:
```bash
mkdir /tmp/test && echo "hello" >> /tmp/test/f1
debugfs -R "dump /tmp/test /dev/stdout" /dev/nvme0n1p4 2 > /dev/null | hexdump -C | head
```
#### bitmap
标记空闲

## FFS
磁盘很慢，但如果你让“相关的数据放得近一点”，就能大幅减少磁头移动（或 SSD 的寻址开销），从而提速！

把磁盘分成 Cylinder Groups（柱面组） → Linux 叫 Block Groups
统 Unix FS 把 inode 和 data block 散落在整个磁盘。

FFS 把磁盘切成多个“自治小区”（Block Group），每个组包含：
自己的 superblock 副本
自己的 inode table
自己的 data blocks
自己的空闲 bitmap

最大限度使用局部性

## 崩溃一致性
“如果系统在写文件写到一半时突然断电，磁盘会变成什么样？还能恢复吗？”
问题根源：写操作不是原子的！
#### 把hello写入一个文件f.txt
1. 分配一个datablock
2. 把hello写进block
3. 分配一个inode
4. 在inode中记录块信息
5. 在父目录中添加一条文件名与inode号的映射
#### 一致性目标：
磁盘状态必须“要么全做，要么全不做”
原理：先写日志，再写真实数据
修改分为三步骤：
1. 讲要做的修改写入日志区
2. 把修改应用到真实位置
3. 标记日志为已提交
如果断电发生在第1步之后、第3步之前 → 开机时我看到 journal 有未完成的操作，就自动重做。
结果：要么 f.txt 完全存在，要么完全不存在——不会出现“文件名有了但内容是垃圾”。
#### 日志默认
日志默认只记录元数据inode与目录，不记录大块内容
linux默认行为打开了data = order,模式保证数据块先写，在更新inode，避免指针指向垃圾
#### 补充
###### write
write 之后数据没有落盘
“write() 只是把数据交给内核缓存。要确保持久化，必须调用 fsync() 或 fdatasync()
###### fsync
只有fsync之后才从缓冲区持久化，代价是性能问题
对关键事务日志（如 WAL）调用 fsync()，但对普通数据采用批量提交 + 延迟 fsync 来平衡性能与安全。”
###### 原子更新
使用rename完成原子更新
```bash
echo "new config" > config.json.tmp
fsync(config.json.tmp)
mv config.json.tmp config.json   # rename 是原子的！
```
###### WAL
先写数据，然后再改数据
采用 WAL 模式：所有修改先 append 到日志文件，fsync 日志后，再更新内存或数据文件。重启时重放日志
###### 避免部分写
如果一次跨越多个磁盘block，断电可能只写了一半
解决办法：写小于512B的数据，保证在硬件上面的原子性

## 数据完整性/数据校验
方案1️⃣：附加校验和（Checksum） → 推荐！简单有效
原理：
写文件时，计算内容的 hash（如 SHA256,CRC32）
把 hash 一起存到文件末尾 或 单独存 .sha256 文件
读文件时，重新计算 hash 并比对

 方案2️⃣：**使用带校验的存储格式**
别自己造轮子！用已内置校验的格式：
如SQLite：
```python
import sqlite3
conn = sqlite3.connect('data.db')
# SQLite 默认对每个 page 做 checksum（需编译时开启）
# 读取时自动验证，损坏会报 "database disk image is malformed"
```
