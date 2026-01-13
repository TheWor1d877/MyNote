在 GoLand 中高效、安全地管理模块依赖，避免版本冲突和构建漂移，为分布式存储项目打下可靠工程基础。

## go mod tidy
删除 go.mod 中未被代码实际使用的依赖（清理）
添加代码中 import 了但 go.mod 里还没记录的依赖（补全）
它的目标是：让 go.mod 精确反映当前项目真实需要的依赖。

## go.sum文件
go.sum 是依赖的“校验清单”，用于保证构建可重现、防篡改。
go.sum 并不直接验证 GitHub/GitLab 上的源码，而是验证 Go 模块代理（或源）返回的「标准化模块 zip 文件」是否被篡改
对每个依赖（包括间接依赖），记录：
- 模块路径 + 版本
- 对应的 go.mod 文件的哈希
- 模块 zip 包的哈希

## go run/build/test/vet/fmt
1. go run → Run（运行）
2. go build → Build（构建二进制）
3. go test → Test（单元测试）
4. go vet → 静态检查（Vet）
5. go fmt → 代码格式化（Format）
再GoLand中只有build需要相关配置
