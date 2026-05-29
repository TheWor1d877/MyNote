gRPC 是 Google 开发的高性能、开源的远程过程调用（RPC）框架
让你像调用本地函数一样调用远程服务器上的函数

相对于HTTP，gRPC在性能上面有优势

| 特性 | REST/HTTP | gRPC |
|------|-----------|------|
| 协议 | HTTP/1.1 (文本) | HTTP/2 (二进制) |
| 性能 | 低（头部冗余） | 高（头部压缩、多路复用） |
| 数据格式 | JSON/XML (可读但臃肿) | Protocol Buffers (紧凑二进制) |
| 流式传输 | ❌ 不支持 | ✅ 支持（大文件上传必备） |
| 强类型 | ❌ 弱类型 | ✅ 编译时检查 |

## Hello World
```proto
// proto/storage.proto
syntax = "proto3";

// 包名（生成Go代码时会用到）
package storage;

// 生成Go代码的选项
option go_package = "./proto;storage";

// 请求消息
message PutRequest {
  string key = 1;    // 存储键（如 "/data/file1"）
  bytes value = 2;   // 存储值（二进制数据）
}

// 响应消息
message PutResponse {
  bool success = 1;  // 操作是否成功
  string message = 2; // 错误信息（如有）
}

// gRPC 服务定义
service StorageService {
  // 一元 RPC：小对象写入
  rpc Put(PutRequest) returns (PutResponse);
}
```
#### 生成go代码
```bash
# 在项目根目录执行
protoc --go_out=. --go-grpc_out=. proto/storage.proto
```

## server
```go
package main

import (
	"context"
	"log"
	"net"

	pb "gRPC-pra/proto"

	"google.golang.org/grpc"
)

type server struct {
	pb.UnimplementedStorageServiceServer
}

func (s *server) Put(cxt context.Context, req *pb.PutRequest) (*pb.PutResponse, error) {
	log.Printf("Received PUT request for key: %s", req.Key)

	return &pb.PutResponse{
		Success: true,
		Message: "Data stored successfully",
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer()
	pb.RegisterStorageServiceServer(s, &server{}) //将新的server注册到type server

	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

## client
```go
package main

import (
	"context"
	pb "gRPC-pra/proto"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

func main() {
	conn, err := grpc.Dial("localhost:8080",
		grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("failed to dial: %v", err)
	}
	defer conn.Close()

	client := pb.NewStorageServiceClient(conn)

	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	req := &pb.PutRequest{
		Key:   "test/file1",
		Value: []byte("hello world"),
	}

	resp, err := client.Put(ctx, req)

	log.Printf("PUT response: success=%v, message=%s", resp.Success, resp.Message)
}

```