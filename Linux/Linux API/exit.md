调用 exit(status) 会：
- 调用所有通过 atexit() 注册的清理函数（按注册的逆序）；
- 刷新并关闭所有标准 I/O 流（如 stdout, stderr）；
- 关闭所有打开的流（C 库层面）；
- 将退出状态 status 返回给操作系统；
- 终止进程。