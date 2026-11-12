Skills 是可复用的指令包。你创建一个 SKILL.md 文件，包含说明，Claude 会将其加入工具包。Claude 在相关时自动使用，你也可以用 /skill-name 直接调用。

CLAUDE.md 是事实性记忆（项目结构、构建命令、编码规范），每次会话都会加载。Skill 是过程性知识，只在被使用时才加载其正文，因此长参考资料在你需要之前几乎不花费任何成本。

## 使用方式
手动触发：在对话中直接输入 /skill-name 来调用。这种方式最适合有副作用的操作（比如部署、发消息），确保只在你明确想要的时候才执行-2-5。

自动触发：Claude 会根据你写在 SKILL.md 里的 description 字段，判断当前任务是否匹配，然后自动加载并使用它。这适合那些“相关时 Claude 应该知道”的知识或参考内容

#### 设置只能手动触发
在头部的yaml中加上:
```yaml
disable-model-invocation: true
```
设置后，Claude 就不会自动加载它，只有你输入 /skill-name 时才会生效

## 创建Skills
```bash
mkdir -p ~/.claude/skills/summarize-changes
nano ~/.claude/skills/summarize-changes/SKILL.md
```

SKILL.md 内容：
```markdown
---
name: summarize-changes
description: 总结当前 git 未提交的改动
---

请执行 `git diff`，然后用中文总结这些改动的主要内容。
保存后，在 Claude Code 会话中输入 /summarize-changes 即可调用。
```
