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
```

## 日常开发 ①：本地工作区