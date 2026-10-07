MCP 是一个开放协议，让 Claude Code 能够使用超出内置工具集的外部工具。这些工具来自 MCP 服务器，可以是本地进程，也可以是托管服务。典型用途：搜索问题跟踪器、查询数据库、控制浏览器、操作 GitHub。
MCP是Agent的外部工具箱

MCP 服务器可以跑在你本机（比如通过 npx 启动一个本地进程），也可以是托管在云端的远程服务

## 配置方式
```bash
claude mcp add --transport <类型> <你给服务器起的名字> <服务器地址或启动命令>
```
配置完成之后,不会立刻下载二进制文件,只是验证配置能否写入,当agent第一次使用的时候才会下载


## 验证与管理
```bash
claude mcp list        # 查看所有服务器及连接状态
claude mcp get <name>  # 查看某个服务器的详细状态
claude mcp remove <name>  # 移除服务器
```

项目级：.mcp.json（项目根目录），只对当前项目生效；
用户级：~/.claude.json，对所有项目生效。
实际用途举例：配置 GitHub MCP 后，Claude Code 可以直接创建 Issue、提交 PR、查看 CI 状态。配置数据库 MCP 后，它可以直接查询表结构来辅助代码生成。

## 使用方式
不需要用 @ 来手动调用。

当你在 Claude Code 里向它描述一个任务时，Claude 会自动评估：这个任务是不是需要用到某个已经连接好的 MCP 服务器的工具。如果需要，它就会自己调用。

比如你添加了 Playwright 服务器后，直接对 Claude 说-3-12：
用 Playwright 打开 https://example.com 并告诉我页面标题。

或者你添加了 Sentry 的 MCP 服务器，直接问：
我有哪些 Sentry 项目？
Claude 就会自动通过 Sentry 的 MCP 服务器去查询。