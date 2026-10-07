## 项目结构规范
```bash
your-distributed-storage/
├── proto/                  # 所有协议文件
│   ├── v1/                 # 【关键】版本化目录
│   │   ├── storage.proto   # 存储核心接口
│   │   └── common.proto    # 公共类型（错误码、元数据等）
│   └── buf.yaml            # （可选）现代 Protobuf 管理工具配置
└── go.mod
```

避免命名冲突：v1.StorageService vs v2.StorageService
清晰版本边界：当需要破坏性变更时，新建 v2/ 目录
符合行业标准：Google Cloud APIs、Kubernetes API 都这样做

## 公共类型定义
这是所有服务的基础库
```protobuf
syntax = "proto3";

// 包名规范：反向域名 + 项目名 + 版本
package yourcompany.storage.v1;

//统一错误码，并预留未来扩展位
enum StatusCode {
  STATUS_UNKNOWN = 0;        // 必须保留 0（Proto3 默认值）
  STATUS_OK = 1;
  STATUS_NOT_FOUND = 2;      // 键不存在
  STATUS_ALREADY_EXISTS = 3; // 写入冲突（如 CAS 失败）
  STATUS_INVALID_ARGUMENT = 4;
  STATUS_INTERNAL_ERROR = 5;

  reserved 6 to 10; 
}

//统一请求头，响应头，相应体
message RequestHeader {
  string client_id = 1;      // 客户端标识
  string trace_id = 2;       // 分布式追踪 ID（如 Jaeger）
  int64 timeout_ms = 3;      // 客户端期望超时时间
}

message ResponseHeader {
  string node_id = 1;        // 处理请求的节点 ID
  int64 timestamp_ns = 2;    // 服务端处理完成时间（纳秒）
  int32 queue_time_ms = 3;   // 请求排队时间（用于监控延迟）
}

message GenericResponse {
  ResponseHeader header = 1; // 响应头（必须放首位）
  StatusCode status = 2;     // 状态码
  string error_message = 3;  // 人类可读错误信息
}
```
trace_id：为后续集成 OpenTelemetry 埋点
统一相应体：避免每个方法重复定义`status/error`字段

#### reserved字段：
防止未来误用已经废弃的编号
假如说定义了枚举字段email=2,想要废弃
正确做法：
```protobuf
message User {
  string name = 1;
  string phone = 3;        // 使用新编号
  
  // 永久封锁编号 2 和字段名 "email"
  reserved 2;
  reserved "email";
}
```
编译时检查：如果有人不小心复用编号 2，protoc 会直接报错！
```bash
$ protoc user.proto
user.proto: Field numbers 2 are reserved.
```

## 核心接口
```protobuf
// proto/v1/storage.proto
syntax = "proto3";

package yourcompany.storage.v1;

// 导入公共定义
import "v1/common.proto";
option go_package = "github.com/yourname/storage/proto/v1;v1";

// 数据模型（MVCC 关键！）
message StorageKey {
  string key = 1;            // 业务键（如 "/users/123/profile"）
  int64 lease_id = 2;        // 租约ID（用于分布式锁，参考 etcd）这个文件属于哪个租户
  reserved 3 to 5;           // 预留字段（未来可能加 namespace）
}

message StorageValue {
  bytes data = 1;            // 原始数据（Protobuf 不关心内容）
  int64 create_time_ns = 2;  // 创建时间（用于 TTL）
  int64 version = 3;         // 【核心】MVCC 版本号（TiKV/etcd 核心机制）
  reserved 4 to 5;
}


// Put 操作（带条件写入）
message PutRequest {
  RequestHeader header = 1;
  StorageKey key = 2;
  StorageValue value = 3;
  
  // 条件写入（类似 CAS）
  oneof condition {
    bool if_not_exists = 4; // “只有当文件不存在时才存”，否则容易出现互相覆盖的问题
    //常用于创建唯一用户名或者初始化配置文件
    int64 expected_version = 5; // “只有当当前版本是X时才覆盖”，防止并发冲突，防止修改旧版本
  }
}

message PutResponse {
  GenericResponse response = 1;
  int64 new_version = 2;     // 写入后的实际版本号（客户端需记录）
}


// Get 操作（支持版本查询）
message GetRequest {
  RequestHeader header = 1;
  StorageKey key = 2;
  int64 version = 3;         // 指定版本查询（实现 MVCC 读）
}

message GetResponse {
  GenericResponse response = 1;
  StorageValue value = 2;
}


// 流式 RPC（大文件分块上传）
message ChunkedPutRequest {
  oneof payload {
    // 首块：包含元数据
    ChunkedPutInit init = 1;
    // 数据块
    ChunkedPutData data = 2;
    // 结束块
    ChunkedPutFinish finish = 3;
  }
}

message ChunkedPutInit {
  StorageKey key = 1;
  int64 total_size = 2;      // 总大小（用于预分配）
  int32 chunk_size = 3;      // 建议 64KB-1MB
}

message ChunkedPutData {
  bytes chunk_data = 1;
  int32 chunk_index = 2;     // 块序号（从0开始）
}

message ChunkedPutFinish {
  uint32 checksum = 1;       // CRC32 校验和（保证数据完整性）
}

message ChunkedPutResponse {
  GenericResponse response = 1;
  int32 received_chunk_index = 2; // 当前接收的块序号（用于重传）
}

// gRPC 服务定义
service StorageService {
  // 一元 RPC：小对象读写
  rpc Put(PutRequest) returns (PutResponse);
  rpc Get(GetRequest) returns (GetResponse);
  
  // 流式 RPC：大文件分块上传
  rpc ChunkedPut(stream ChunkedPutRequest) returns (ChunkedPutResponse);
  
  // 服务端流：订阅键变更（类似 etcd Watch）
  rpc Watch(WatchRequest) returns (stream WatchResponse);
}
```

#### 详解
- 整体结构速览（先看森林，再看树木）：
`service StorageService`
- 数据存储方式
`StorageKey` 和 `StorageValue`
为什么需要version?
假设你和同事同时编辑同一个文件：
你拿到的是版本 3
同事先保存了版本 4
你保存时系统会拒绝（因为你说“我要覆盖版本3”，但实际已是版本4）
这就是 MVCC（多版本并发控制），etcd/TiKV 的核心机制！

#### 关键点


| 特性 | 实现方式 | 参考系统 |
|------|----------|----------|
| MVCC | `version` 字段 | TiKV, etcd |
| 租约 | `lease_id` | etcd Lease |
| 条件写入 | `oneof condition` | Cassandra LWT |
| 分块传输 | `stream` + `oneof` | S3 Multipart Upload |
| 数据校验 | `checksum` | HDFS, Ceph |

## 版本兼容性黄金法则
1. 只增不减字段
2. 使用reversed标记字段
