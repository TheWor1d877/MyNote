缓冲区为了减少系统调用
page Cache 为了减少IO次数

缓存文件IO vs 直接文件IO
区别就在于是否经过Page Cache

用户缓冲区对齐，绕过内核缓存，延迟高
吞吐量不稳定，并发性能低下

什么时候使用directIO?
page cache是负担的时候

1. 自己有一套自己的buffer pool
2. 大文件顺序扫描，直接使用DMA进入用户态,省一次memcpy的过程