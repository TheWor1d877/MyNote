```text
1. 先在对方仓库开 issue 讨论 —— 未经讨论的 PR 大概率白做，这是开源贡献最大的时间 sink
2. 网页 Fork 对方仓库 → 你的账号下出现一份可写的同名仓库
3. clone 你那份 fork，建一个分支
4. 改代码、加测试，commit 信息遵守对方规范（很多 CNCF 项目要求 DCO：每个 commit 带 Signed-off-by，
   用 git commit -s；有些还要求签 CLA）
5. push 到你的 fork
6. 在对方仓库开 Pull Request，Base 选对方主干，Compare 选你的分支
7. Review 迭代：改完 push 同一分支，PR 自动更新；不要 rebase/squash 到打乱 reviewer 的行号，
   等合并前再整理历史
8. 对方以 squash merge 合并 → 你本地 git pull --rebase 同步，删掉已合并的分支
```