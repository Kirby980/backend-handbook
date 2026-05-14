# 精通 gRPC 与 Protobuf

> 课程编号：G28
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — gRPC 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：为什么 gRPC

REST + JSON 在 web 边界无敌；但服务之间通信场景下：

- JSON 解析慢（反射）
- 字段无类型保证（前后端口约容易错）
- 文档 = 注释 + Swagger，常常过期
- HTTP/1.1 单连接单请求

gRPC 解决：

- 二进制 Protobuf，紧凑且解析快
- IDL（.proto）严格类型 + 代码生成
- HTTP/2 多路复用，单连接处理千请求
- 支持双向流、超时、元数据传播

本章拆开 Go gRPC 的开发流程、interceptor、流模式、生产配置。

---

## 第一章：从 .proto 到 Go 代码

### 1.1 hello.proto

```protobuf
syntax = "proto3";

package hello.v1;
option go_package = "github.com/me/hello/v1;hellov1";

message HelloRequest {
    string name = 1;
}
message HelloResponse {
    string message = 1;
}

service Greeter {
    rpc Hello(HelloRequest) returns (HelloResponse);
}
```

### 1.2 代码生成

```bash
protoc \
  --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  hello.proto
```

或用 buf：

```bash
buf generate
```

生成两个文件：
- `hello.pb.go`：message struct + 序列化方法
- `hello_grpc.pb.go`：server interface + client stub

### 1.3 实现 server

```go
type greeter struct {
    hellov1.UnimplementedGreeterServer
}

func (g *greeter) Hello(ctx context.Context, req *hellov1.HelloRequest) (*hellov1.HelloResponse, error) {
    return &hellov1.HelloResponse{Message: "Hello, " + req.Name}, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":50051")
    s := grpc.NewServer()
    hellov1.RegisterGreeterServer(s, &greeter{})
    s.Serve(lis)
}
```

### 1.4 client

```go
conn, _ := grpc.NewClient("localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()))
defer conn.Close()

client := hellov1.NewGreeterClient(conn)
resp, _ := client.Hello(ctx, &hellov1.HelloRequest{Name: "Alice"})
fmt.Println(resp.Message)
```

---

## 第二章：Protobuf 类型基础

### 2.1 标量类型

| proto | Go |
|---|---|
| `int32`/`int64` | int32/int64 |
| `uint32`/`uint64` | uint32/uint64 |
| `sint32`/`sint64` | int32/int64（zigzag 编码，负数省空间） |
| `fixed32`/`fixed64` | 4/8 字节定长 |
| `float`/`double` | float32/float64 |
| `bool` | bool |
| `string` | string |
| `bytes` | []byte |

### 2.2 message 嵌套与 oneof

```protobuf
message User {
    string name = 1;
    repeated string emails = 2;
    map<string, string> metadata = 3;
    oneof contact {
        string phone = 4;
        string slack = 5;
    }
}
```

### 2.3 字段编号

- 1-15 占 1 字节，**高频字段优先**
- 16-2047 占 2 字节
- 19000-19999 保留
- 字段一旦发布**永远不要改编号**（兼容性）

### 2.4 deprecated 与 reserved

```protobuf
message User {
    reserved 3, 4;       // 永久占位
    reserved "old_name"; // 字段名也保留
    string name = 1 [deprecated = true];
}
```

删除字段时 `reserve` 编号避免后续重用导致旧客户端解析错。

---

## 第三章：四种 RPC 模式

### 3.1 Unary（普通 RPC）

```protobuf
rpc GetUser(GetUserRequest) returns (GetUserResponse);
```

请求 → 响应，最常见。

### 3.2 Server Streaming

```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```go
func (s *server) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    for _, u := range users {
        if err := stream.Send(u); err != nil { return err }
    }
    return nil
}

// client
stream, _ := client.ListUsers(ctx, req)
for {
    u, err := stream.Recv()
    if err == io.EOF { break }
    if err != nil { return err }
    handle(u)
}
```

适合：分页 / 实时推送 / 大结果集。

### 3.3 Client Streaming

```protobuf
rpc UploadChunks(stream Chunk) returns (UploadResponse);
```

client 发送多个，server 一次性回复。适合：大文件上传、批量插入。

### 3.4 Bidirectional Streaming

```protobuf
rpc Chat(stream Message) returns (stream Message);
```

双向同时发送。适合：聊天、协同编辑、实时游戏。

---

## 第四章：Interceptor

### 4.1 Unary Server Interceptor

```go
func logging(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("%s %v err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}

s := grpc.NewServer(grpc.UnaryInterceptor(logging))
```

### 4.2 多 interceptor 链

```go
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(recover, requestID, logging, auth),
)
```

执行顺序：从外到内（recover 最外，auth 最内）。

### 4.3 Stream Interceptor

```go
func streamLogging(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    // ...
    return handler(srv, ss)
}
```

### 4.4 Client Interceptor

```go
conn, _ := grpc.NewClient(addr,
    grpc.WithUnaryInterceptor(clientLog),
    grpc.WithStreamInterceptor(clientStreamLog),
)
```

---

## 第五章：错误处理

### 5.1 gRPC status codes

| code | 含义 |
|---|---|
| OK | 成功 |
| Canceled | 客户端取消 |
| DeadlineExceeded | 超时 |
| NotFound | 资源不存在 |
| AlreadyExists | 已存在 |
| PermissionDenied | 无权限 |
| ResourceExhausted | 资源不够 |
| Unauthenticated | 未认证 |
| Internal | 内部错误 |
| Unavailable | 暂时不可用 |

### 5.2 返回 status

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    u, err := s.db.GetUser(req.Id)
    if errors.Is(err, ErrNotFound) {
        return nil, status.Error(codes.NotFound, "user not found")
    }
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }
    return u, nil
}
```

### 5.3 携带详细信息

```go
import "google.golang.org/genproto/googleapis/rpc/errdetails"

st := status.New(codes.InvalidArgument, "validation failed")
st, _ = st.WithDetails(&errdetails.BadRequest{
    FieldViolations: []*errdetails.BadRequest_FieldViolation{
        {Field: "email", Description: "invalid format"},
    },
})
return nil, st.Err()
```

### 5.4 Client 解析

```go
resp, err := client.GetUser(ctx, req)
if err != nil {
    st, _ := status.FromError(err)
    fmt.Println(st.Code(), st.Message())
    for _, d := range st.Details() {
        // 处理详细信息
    }
}
```

---

## 第六章：超时、deadline 传播

### 6.1 client 设 deadline

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

deadline 通过 gRPC metadata 传到 server。server 端 `ctx.Deadline()` 看到。

### 6.2 server 端

```go
func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    select {
    case <-ctx.Done(): return nil, status.FromContextError(ctx.Err()).Err()
    default:
    }
    // 调下游也带 ctx → deadline 自动传播
    return s.repo.Get(ctx, req.Id)
}
```

整条调用链共享同一 deadline——客户端 5s → DB query 也最多 5s。

### 6.3 cascade cancellation

client cancel → server 的 ctx 也 cancel → server 调下游服务的 ctx 也 cancel。这是 gRPC 的强项之一。

---

## 第七章：元数据（metadata）

### 7.1 类似 HTTP header

```go
import "google.golang.org/grpc/metadata"

// client 发
md := metadata.Pairs("authorization", "Bearer "+token)
ctx = metadata.NewOutgoingContext(ctx, md)

// server 读
md, _ := metadata.FromIncomingContext(ctx)
auths := md.Get("authorization")
```

### 7.2 常见用途

- 认证 token
- traceID（OpenTelemetry 自动注入）
- request 来源标识
- 客户端版本号

### 7.3 大写敏感

metadata 字段名**自动转小写**。所以 `Authorization` 和 `authorization` 是同一字段。

---

## 第八章：性能与配置

### 8.1 连接池

gRPC client 默认**一个 channel** = 一个 HTTP/2 连接。所有 RPC 在该连接上多路复用。

高 QPS 场景考虑多 channel：

```go
balancer := roundrobin.Name
conn, _ := grpc.NewClient("dns:///service:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"`+balancer+`":{}}]}`),
)
```

### 8.2 keepalive

```go
import "google.golang.org/grpc/keepalive"

s := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        Time:    30 * time.Second,
        Timeout: 10 * time.Second,
    }),
)
```

防止 NAT/防火墙超时静默断开。

### 8.3 max message size

```go
grpc.MaxRecvMsgSize(64*1024*1024)
grpc.MaxSendMsgSize(64*1024*1024)
```

默认 4MB。大消息用 streaming 而非单条扩大。

### 8.4 压缩

```go
import "google.golang.org/grpc/encoding/gzip"
_ = gzip.Name   // 注册压缩器
client.Call(ctx, req, grpc.UseCompressor(gzip.Name))
```

CPU 换带宽。10KB+ 的 message 通常值。

---

## 第九章：测试与调试

### 9.1 in-memory listener

```go
import "google.golang.org/grpc/test/bufconn"

lis := bufconn.Listen(1024 * 1024)
s := grpc.NewServer()
pb.RegisterGreeterServer(s, &greeter{})
go s.Serve(lis)

conn, _ := grpc.NewClient("passthrough://bufnet",
    grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
        return lis.Dial()
    }),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

不开端口、不依赖网络。

### 9.2 grpcurl

```bash
grpcurl -plaintext -d '{"name":"Alice"}' localhost:50051 hello.v1.Greeter/Hello
```

CLI 调 gRPC，类似 curl。

### 9.3 reflection

```go
import "google.golang.org/grpc/reflection"

s := grpc.NewServer()
pb.RegisterGreeterServer(s, &greeter{})
reflection.Register(s)
```

启用后 grpcurl / postman 可以"列出所有方法"。开发期超有用，生产可能想关。

---

## 第十章：生产级最佳实践

1. **proto 文件单独仓库**：多服务共享 schema。
2. **buf + buf.yaml + breaking 检查**：防止破坏性变更。
3. **永远 client.Timeout**：调下游必带 context。
4. **interceptor 链：recover + log + metrics + tracing + auth**。
5. **错误用 status.Error**：让 client 看 code。
6. **field 编号永不重用 + 删字段用 reserved**。
7. **大数据用 streaming**：单消息 < 4MB。
8. **keepalive 配置**：穿越 NAT/L4。
9. **生产关 reflection**：减少元数据暴露。
10. **OpenTelemetry interceptor**：trace 全链路。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：proto3 没有"零值"区分
默认 `int32 x` 是 0 时无法区分"未设置"vs"显式 0"。要可空用 `optional`（proto3 重新加回来）或 wrapper types（`google.protobuf.Int32Value`）。

### ❌ 陷阱 2：字段编号重用
删了一个字段后新字段用相同编号 → 老客户端解析时把新数据当老字段。

### ❌ 陷阱 3：忘了 `paths=source_relative`
默认 protoc-gen-go 生成的路径包含 `option go_package` 的全路径，结果生成在意外位置。

### ❌ 陷阱 4：消息太大
4MB 限制。大文件用 streaming chunks。

### ❌ 陷阱 5：client 不关 conn
goroutine 泄漏 + 连接累积。`defer conn.Close()`。

### ❌ 陷阱 6：用 raw errors 返回
client 拿到 codes.Unknown 失去语义。永远 status.Error。

### ❌ 陷阱 7：interceptor 顺序错
recover 必须最外；其他按逻辑层级。

---

## 第十二章：练习题

**练习 1**：写一个 .proto 定义 echo 服务，含 unary 和 server streaming 两种方法。

**练习 2**：实现一个 unary interceptor，把每个 RPC 的 duration 和 status code 推到 prometheus。

**练习 3**：以下 client 调用有何问题？
```go
client.GetUser(context.Background(), &pb.GetUserRequest{Id: 1})
```

**练习 4**：如何在 gRPC 中实现"client 取消正在进行的 server streaming"？

**练习 5**：解释为什么不应该在 .proto 中改字段编号。

---

## 参考答案

**练习 1**：
```protobuf
syntax = "proto3";
package echo.v1;
option go_package = "github.com/me/echo/v1;echov1";

message EchoRequest { string msg = 1; int32 count = 2; }
message EchoResponse { string msg = 1; }

service Echo {
    rpc EchoOnce(EchoRequest) returns (EchoResponse);
    rpc EchoStream(EchoRequest) returns (stream EchoResponse);
}
```

**练习 2**：
```go
var rpcDuration = prometheus.NewHistogramVec(...)
var rpcStatus = prometheus.NewCounterVec(...)

func metrics(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    code := status.Code(err).String()
    rpcDuration.WithLabelValues(info.FullMethod).Observe(time.Since(start).Seconds())
    rpcStatus.WithLabelValues(info.FullMethod, code).Inc()
    return resp, err
}
```

**练习 3**：用 `context.Background()` 没 deadline。修：`context.WithTimeout`。

**练习 4**：client 调 `cancel()`（context 关联的）→ gRPC framework 发送 RST_STREAM → server 端 ctx.Done()。server 端循环里 `select <-ctx.Done()` 终止。

**练习 5**：编号是 wire format 的 key——同一编号对应同一字段。改编号后：
- 老客户端发的数据，新 server 用错误字段解码
- 新客户端发的数据，老 server 不认识或解码到错字段

字段名可以改（不在 wire 中），编号不能。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 工具链 | protoc + protoc-gen-go + protoc-gen-go-grpc / buf |
| 四种 RPC | unary / server stream / client stream / bidi |
| Status | code + message + details |
| Interceptor | server/client + unary/stream |
| Deadline | 通过 context 自动跨服务传播 |
| Metadata | HTTP/2 header 类比 |
| 性能 | HTTP/2 多路复用；keepalive；大消息用 stream |

下一篇 **G29 — 精通 Go 数据库访问** 会讲清 database/sql、连接池、pgx、GORM、prepared statement、事务隔离级别。

---

