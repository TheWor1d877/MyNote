scratch 是 Docker 提供的一个“虚拟空镜像”——它不包含任何文件、任何目录、任何程序，甚至没有 /bin/sh。scratch 的存在是为了让你能构建 “只有你的程序，别无他物” 的极致精简镜像。

Alpine Linux一个基于 musl libc 和 BusyBox 的轻量级 Linux 发行版。
特点：体积非常小（基础镜像仅 5 MB 左右），默认安装精简，注重安全（内核有补丁），且包管理工具 apk 使用方便。

如果可执行文件（exe）依赖外部 .so 动态库（shared libraries），那么它绝对不能在 scratch 镜像中运行。

