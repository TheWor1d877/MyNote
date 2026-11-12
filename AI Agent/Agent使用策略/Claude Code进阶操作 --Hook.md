Hook 的本质是在 Claude Code 生命周期的特定节点，自动触发你预设好的逻辑。它让你从“完全信任 Agent”变成“信任但验证”——无论 Claude 自己怎么决定，该执行的规则一定会执行

三个关键逻辑:
- 事件（Event）：什么时候触发？比如“工具执行前”、“会话开始时”、“Claude 回答完毕时”
- 匹配器（Matcher）：对谁触发？比如“只对 Bash 工具”、“只对 Edit 工具”
- 处理器（Handler）：触发后干什么？运行一个 Shell 脚本、发个 HTTP 请求、让模型判断一
## 配置位置
~/.claude/settings.json：全局生效，只对你本机有效
.claude/settings.json：项目级，可提交到 Git，团队共享
.claude/settings.local.json：项目级但 gitignore，只对自己生效

## 事件
Hook 的本质是**在 Claude Code 生命周期的特定节点，自动触发你预设好的逻辑**。它让你从“完全信任 Agent”变成“信任但验证”——无论 Claude 自己怎么决定，该执行的规则一定会执行。

| 事件 | 触发时机 | 能做什么 |
|---|---|---|
| **PreToolUse** | 工具执行**前** | **拦截危险操作**、修改参数 |
| **PostToolUse** | 工具执行**成功后** | 自动格式化代码、记录日志 |
| **UserPromptSubmit** | 你提交提示词后 | 预处理、注入额外上下文 |
| **SessionStart** | 会话开始/恢复时 | 设置环境变量、加载上下文 |
| **Stop** | Claude 完成响应时 | 阻止过早结束、触发总结 |
| **Notification** | Claude 需要你输入时 | 发桌面通知 |

其中 **PreToolUse 是最常用的**，绝大多数“护栏”需求都在这里实现。

## 匹配器
使用matcher字段过滤Hook在什么情况下执行
不同事件类型，匹配器过滤的对象也不同
## 处理器
处理器通过 type 字段指定，共有 5 种类型：
   
类型	             作用	                                   关键字段
command	运行 Shell 脚本/命令	            command
http	            向指定 URL 发送 POST 请求	url、headers、allowedEnvVars
prompt	   调用 LLM 进行单轮判断	        prompt、model
agent	       启动子代理进行多轮工具验证	prompt、model
mcp_tool	   调用已连接的 MCP 工具	        server、tool、input
## 实例
防止 Claude 执行 rm -rf 这类破坏性操作。你在 PreToolUse 事件里挂一个脚本:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block-dangerous.sh"
          }
        ]
      }
    ]
  }
}
```
脚本 block-dangerous.sh 从 stdin 拿到 JSON 输入，检查命令是否包含危险模式，用 exit 2 阻止执行
```bash
#!/bin/bash
INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty')

if echo "$COMMAND" | grep -qE 'rm\s+-rf\s+/|git\s+push\s+.*--force'; then
    echo "BLOCKED: 命令匹配危险模式" >&2
    exit 2
fi
exit 0
```

## 注意
Hook 脚本以你的完整用户权限运行，没有任何沙箱隔离。这意味着一个写错的脚本可能删除你的文件、泄露你的凭证。

