每个进程拥有自己独立的文件描述符表（fd table），彼此完全隔离。

你的进程中的 fd = 5 和我的进程中的 fd = 5 毫无关系，可能指向完全不同的资源（甚至一个有效、一个无效）。

查看进程的文件描述符,假设进程pid是1234
```bash
ls /proc/1234/fd
```
```bash
howardhe@ASUS-HE:~$ ll /proc/16167/fd
total 0
dr-x------ 2 howardhe howardhe 13  1月  2 15:28 ./
dr-xr-xr-x 9 howardhe howardhe  0  1月  2 15:27 ../
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 0 -> /dev/pts/2
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 1 -> /dev/pts/2
lr-x------ 1 howardhe howardhe 64  1月  2 15:28 103 -> /snap/code/217/usr/share/code/v8_context_snapshot.bin
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 2 -> /dev/pts/2
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 3 -> 'socket:[1812184]'=
lr-x------ 1 howardhe howardhe 64  1月  2 15:28 37 -> anon_inode:inotify
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 38 -> 'socket:[7007]'=
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 4 -> 'anon_inode:[eventpoll]'
l-wx------ 1 howardhe howardhe 64  1月  2 15:28 45 -> /home/howardhe/.config/Code/logs/20260102T123959/ptyhost.log
lrwx------ 1 howardhe howardhe 64  1月  2 15:28 46 -> /dev/ptmx
lrwx------ 1 howardhe howardhe 64  1月  2 16:34 5 -> 'socket:[1862374]'=
lrwx------ 1 howardhe howardhe 64  1月  2 16:34 6 -> 'socket:[1862376]'=
lrwx------ 1 howardhe howardhe 64  1月  2 16:34 7 -> 'socket:[1723252]'=
howardhe@ASUS-HE:~$ 
```

3是server_fd,监听文件描述符
4是epoll_fd,epoll的文件描述符
567是client_fd已连接文件描述符

