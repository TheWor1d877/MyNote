Dev Container 就是一个“装了你整个开发环境的 Docker 容器”。
你写代码、编译、调试、运行……所有操作都在这个容器里进行，而不是在你自己的电脑上。

## Dev Container 的核心组成
1. 一个标准的 Docker 容器
基础镜像：比如 golang:1.23
额外安装：delve（Go 调试器）、git、kubectl、helm 等工具
用户权限：通常创建非 root 用户（安全）
2. 一个配置文件：.devcontainer/devcontainer.json
这个文件告诉 GoLand：
用哪个镜像？
要装哪些额外工具？
要映射哪些端口？（比如 8080 给 Web 服务，2345 给调试器）
启动后要运行什么命令？
3. GoLand 的魔法
GoLand 会在容器里启动一个“后台服务”（backend）
你的本地 GoLand 界面（UI）和这个 backend 通信
结果：你感觉像在本地开发，其实所有操作都在容器里