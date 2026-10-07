## 优化编译参数
```bash
# 减小编译体积（去除调试信息）
go build -ldflags="-s -w"
```

## 指定编译模式
```bash
# 指定编译模式
go build -buildmode=pie        # 位置无关可执行文件
go build -buildmode=c-shared   # C共享库
go build -buildmode=plugin     # 插件
```

## 交叉编译参数
```bash
# Linux 平台
GOOS=linux GOARCH=amd64 go build

# Windows 平台
GOOS=windows GOARCH=amd64 go build

# macOS 平台
GOOS=darwin GOARCH=arm64 go build

# 指定 CGO 状态
CGO_ENABLED=0 go build   # 禁用 CGO，生成静态二进制
```

## 调试与分析参数
```bash
# 逃逸分析（之前提到的）
go build -gcflags="-m"

# 禁用优化和内联（便于调试）
go build -gcflags="all=-N -l"

# 生成汇编代码
go build -gcflags="-S" main.go

# 竞态检测（编译时）
go build -race
```

## 其他
```bash
# 显示编译命令但不执行
go build -n

# 显示编译详细信息
go build -x

# 编译时包含版本信息
go build -ldflags="-X main.version=1.0.0 -X main.buildTime=$(date +%Y%m%d)"

# 编译测试二进制
go test -c -o mytest.test

# 修剪依赖（Go 1.21+）
go build -trimpath
```