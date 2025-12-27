## 一次配置，终身受用
```bash
# 身份 & 安全
git config --global user.name "LastName FirstName"
git config --global user.email "work@company.com"
git config --global credential.helper store      # 仅在内网机器使用

# 默认分支名对齐平台
git config --global init.defaultBranch main

# 实用别名（官方推荐拼写）
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'restore --staged'
git config --global alias.last 'log -1 --oneline'

#修改默认编辑器为vim
git config --global core.editor vim
```



## 二、日常开发 ①：本地工作区
   
| 场景       | 命令                                         |
| -------- | ------------------------------------------ |
| 查看整体状态   | `git status -s`                            |
| 快速 Diff  | `git diff` / `git diff --staged`           |
| 原子提交     | `git add -p` → review 每块改动                 |
| 写消息      | `git ci -m "feat(auth): add SSO callback"` |
| 改最后一次    | `git ci --amend` （**未 push 前**）            |
| 丢弃工作区    | `git restore .`                            |
| 撤回暂存区    | `git restore --staged <file>`              |
| 查看HEAD文件 | `git show HEAD:<文件名称>`                     |
| 查看暂存区文件  | `git show :<文件名称>`                         |


## 三、日常开发 ②：分支与合并（Merge Request 流程）

| 场景             | 命令                                           |
| -------------- | -------------------------------------------- |
| 新建功能分支         | `git co -b feature/user-list`                |
| 推送并建立跟踪        | `git push -u origin feature/user-list`       |
| 保持线性历史（rebase） | `git pull --rebase origin main`              |
| MR 前整理提交       | `git rebase -i origin/main`                  |
| 合并 PR（平台点按钮）   | ***不做本地 merge***                             |
| 删除远端已合分支       | `git push origin --delete feature/user-list` |
| 清理本地           | `git branch -d feature/user-list`            |

## 四、紧急与修复
| 场景           | 命令                                          |
| ------------ | ------------------------------------------- |
| 暂存半成品        | `git stash save -m "WIP: search filter"`    |
| 恢复最新         | `git stash pop`                             |
| 安全撤销公共提交     | `git revert <commit>`                       |
| 本地回退（未 push） | `git reset --soft HEAD~1`                   |
| 找回“误删”       | `git reflog` → `git co -b recover HEAD@{2}` |

   
## 五、代码审查 & 差异速览
   ```bash
   # 当前分支与目标分支差异
   git diff origin/main...HEAD          # 三点比较，仅看分支新增
   
   # 图形化 log
   git log --oneline --graph --decorate --all
   
   # 按作者/日期过滤
   git log --author="Zhang" --since="1 week"
   ```
   
   ---
   
## 六、标签与版本发布
   ```bash
   # 语义化版本
   git tag -a v1.3.0 -m "release: v1.3.0"
   git push origin v1.3.0               # CI 自动打包
   ```
   
   ---
   
## 七、子模块（仅当项目强制依赖时）
   ```bash
   git submodule update --init --recursive
   ```
   
   ---
   
## 八、强制但安全推送（重写历史后）
   ```bash
   git push --force-with-lease           # 防止覆盖他人新提交
   ```
   
   ---


## 九、企业工作流口诀（背下来）
   1. **main** 保护，禁止直推  
   2. 需求必开 **feature** 分支，生命周期 ≤ 7 天  
   3. 提交前 `pull --rebase`，保持线性  
   4. MR 通过 **CI + CodeReview** 后平台合并  
   5. 合并即删分支，仓库常清  
   6. 公共历史一律用 **revert**，禁用 `reset --hard`
   
工作区与暂存区 status restore diff stash 
暂存区与本地仓库  diff(staged)
分支switch merge fetch
前后提交 rebase squash revert reset -hard --soft
查看工作区：  ls-files
