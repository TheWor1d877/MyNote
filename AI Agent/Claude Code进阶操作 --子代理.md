## 子代理
子代理是 Claude 派生出的独立工作单元，拥有独立的上下文窗口。用途：
- 研究委托：让子代理去读测试文件或探索某个模块，主上下文只保留结论-；子代理最后只将研究结果发送给主会话
- 并行执行：多个子代理可以同时处理独立子任务；
- 隔离上下文：子代理的中间过程不会污染主对话。
子代理是会话内部的执行者,不管是交互式还是后台都能使子代理

相比于传统的IDE会话是一条连续的大对话,所有内容都在一个上下文窗口中,子代理关键在于上下文隔离与工具权限控制

缺点:每个子代理都有独立的上下文窗口，启动时需要重新收集信息，从“白纸”开始工作，这会消耗额外的 token 和时间

## 使用子代理的时机
- 自包含,一次性的脏活,不想污染主会话上下文
- 只关心最终结论,不关心过程
- 需要严格控制权限,跟主会话权限分离
- 多个任务并行跑起来

## 内置子代理
Explore	快速搜索和理解代码库，只读，不修改	只读
Plan	在计划模式下收集代码库信息来制定方案	只读
general-purpose	通用型，处理不适合其他专业代理的任务	继承主对话权限

## 基本操作
/tasks 列出当前会话的所有后台任务，包括已完成的子代理

#### 创建子代理
- 直接让 Claude 帮你创建
Create a personal code-improver subagent in ~/.claude/agents/ that scans files and suggests improvements for readability, performance, and best practices. Make it read-only and have it use Sonnet.
Claude 会自动帮你生成好包含 name、description、tools、model 和系统提示词的 Markdown 文件-1-9。

- 手动创建
子代理本质是一个带 YAML frontmatter 的 Markdown 文件，放在以下位置之一

.claude/agents/	    范围: 当前项目（优先级更高）
~/.claude/agents/	范围: 你机器上的所有项目

#### 使用子代理
- 自动委派
你不需要记住子代理的名字。只要任务描述与 description 字段匹配，Claude 会自动把活派给它-1。比如你问“帮我检查这个 API 有没有 SQL 注入风险”，Claude 会根据 security-auditor 子代理的描述自动选择它。
- 手动点名使用
当你知道该用谁的时候，直接指定-2-9：
Use the code-reviewer subagent to check my recent changes

或者用 @ 提及：
@agent-code-reviewer look at the auth changes
#### 禁用某个子代理自动委派
如果你不希望 Claude 自动使用某个内置或自定义的子代理（比如 Explore），可以在 settings 文件里把它加入 deny 列表
```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

这样 Claude 就完全无法自动调用它们了。你也可以在启动时用命令行参数临时禁用：claude --disallowedTools "Agent(Explore)"

