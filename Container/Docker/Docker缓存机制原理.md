缓存的本质： 指纹比对
- Dockerfile 指令的内容
- 该指令涉及的文件内容
只有这两个都完全相同的时候才会认为i这一层之前构建过，可以直接复用


```dockerfile
FROM golang:1.23
   WORKDIR /app
   
   # 工具依赖（很少变化）
   COPY tools.go ./
   RUN go install golang.org/x/tools/cmd/goimports@latest
   
   # 项目依赖（偶尔变化）
   COPY go.mod go.sum ./
   RUN go mod download
   
   # 源代码（经常变化）
   COPY . .
   RUN go build -o main .
```

在多阶段构建中，构建阶段也可以利用缓存：

```go
# 构建阶段
FROM golang:1.23 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download          # ← 这层会被缓存
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o main .

# 运行阶段  
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

## Dockerfile 指令与镜像层的关系
- "千层蛋糕模型"
基本原则：每个指令 = 一层
但有重要例外！

指令分类：
- 创建新层的指令（Layer-creating instructions）：
RUN
COPY
ADD

- 不创建新层的指令（Metadata-only instructions）：
FROM
ENV
WORKDIR
USER
EXPOSE
VOLUME
CMD
ENTRYPOINT
LABEL
ARG