限制场景：同时执行多个 Podman 命令

```bash
# 终端 1：执行长时间运行的命令
podman run --rm alpine sleep 3600

# 终端 2：尝试同时执行另一个命令
podman ps  # 这个命令会被阻塞，直到终端1的命令完成或放到后台
```

为什么会有这个限制？
- Podman 使用文件锁保护状态数据库
- 防止多个 podman 进程同时修改容器状态导致数据损坏
- 这只是命令执行的限制，不是容器运行的限制

解决方案：使用后台运行:
```bash
# 正确的做法：让容器在后台运行
podman run -d alpine sleep 3600  # -d 参数让容器后台运行

# 现在可以立即执行其他命令
podman ps        # 立即返回
podman images    # 立即返回
podman run -d nginx  # 立即启动另一个容器
```

Podman 命令执行是串行的，但容器运行是并行的
在实际使用中，只要使用 -d 参数让容器在后台运行，你就可以像使用 Docker 一样自由地管理多个并发容器