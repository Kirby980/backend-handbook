# 精通 gRPC 生产实践

> 课程编号：B05
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — gRPC
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：超越 hello world 的 gRPC

Go 课程 G28 讲了 gRPC 的语法和基本概念。本章重点在**生产部署**：服务发现、负载均衡、熔断、超时、重试、流量控制、可观测性、健康检查、与 REST/网关的整合。这些是把 gRPC 从 demo 推向生产的关键。

---

## 第一章：gRPC 在架构中的位置

### 1.1 典型场景

- **微服务之间**：内部 RPC（替代 REST + JSON）
- **移动 client ↔ 后端**：减小流量
- **服务 ↔ 数据平面**（Envoy xDS、Kubernetes API）
- **流式数据**（实时 metric、事件）

### 1.2 gRPC vs REST

| 维度 | gRPC | REST |
|---|---|---|
| 协议 | HTTP/2 | HTTP/1.1 或 2 |
| 编码 | Protobuf 二进制 | JSON 文本 |
| Schema | 强制 .proto | 可选 OpenAPI |
| 流 | 双向流原生 | 需 SSE/WebSocket |
| 浏览器 | 需 gRPC-Web 网关 | 原生 |
| 工具链 | protoc 必需 | curl 即可 |
| 性能 | 高（10x+ JSON 解析） | 中等 |
| 可读性（调试） | 低 | 高 |

经验：**外部 API → REST**（浏览器、第三方）；**内部 → gRPC**（性能 + schema）。

### 1.3 gRPC-Web

浏览器不能直接打 HTTP/2 + trailers + binary—需要 Envoy 或 grpc-web proxy 转换。架构：

```
浏览器 → gRPC-Web → Envoy → gRPC (HTTP/2)
```

---

## 第二章：服务发现

### 2.1 三种主流

**DNS-based**
```
grpc.NewClient("dns:///user-service:50051", ...)
```
client 周期性查 DNS A 记录拿到多个 IP；适合 K8s headless service、Consul。

**xDS (Envoy)**
Envoy 控制平面把后端列表推给 client。Istio、Cloud-native 主流。

**手工 endpoint list**
```
// grpc-go 无内置 static scheme；固定地址列表用 manual.Resolver（自定义注册），
// 单地址可用 passthrough:///10.0.1.1:50051
r := manual.NewBuilderWithScheme("myfixed")
r.InitialState(resolver.State{Addresses: []resolver.Address{
    {Addr: "10.0.1.1:50051"}, {Addr: "10.0.1.2:50051"},
}})
grpc.NewClient(r.Scheme()+":///", grpc.WithResolvers(r))
```
适合本地开发、固定部署。

### 2.2 Kubernetes Service

```yaml
apiVersion: v1
kind: Service
metadata: { name: user-service }
spec:
  clusterIP: None    ← headless
  ports: [{ port: 50051 }]
  selector: { app: user }
```

`clusterIP: None`（headless）让 DNS 返回所有 pod IP；普通 ClusterIP 会经过 kube-proxy 做 L4 负载均衡，但**对 HTTP/2 长连接**不友好（粘到一个后端）。

---

## 第三章：负载均衡

### 3.1 L4 与 L7

- **L4**：TCP 层负载（kube-proxy、AWS NLB）—— gRPC 长连接下流量绑定一个后端，不均匀
- **L7**：gRPC 层（client side 或 envoy）—— 每 RPC 独立路由

### 3.2 client-side LB

```go
conn, _ := grpc.NewClient(
    "dns:///user-service:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingConfig":[{"round_robin":{}}]}`),
)
```

策略：
- `round_robin`：轮询
- `pick_first`（默认）：粘住第一个连得通的
- `weighted_round_robin`：按权重
- `least_request`：连接最少
- `priority`：故障转移组

### 3.3 proxy-based（Envoy / Linkerd）

每个 pod 一个 sidecar，client 把 gRPC 发给 sidecar，sidecar 做 L7 LB + 重试 + 熔断。

优点：策略不在每个 client 实现；语言无关。
缺点：多一层网络 + 复杂部署。

### 3.4 经验

- 小规模 / 同语言栈 → client-side LB
- 多语言 + 复杂治理 → service mesh

---

## 第四章：超时与 deadline 传播

### 4.1 client 必设 deadline

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

gRPC framework 通过 `grpc-timeout` HTTP/2 header 把 deadline 传给 server。

### 4.2 server 端尊重 ctx

```go
func (s *server) GetUser(ctx context.Context, req *pb.Req) (*pb.Resp, error) {
    return s.db.Get(ctx, req.Id)   // 透传
}
```

ctx 取消时 server 调用栈每一层都能感知；无需手动 cancel。

### 4.3 deadline cascading

整条调用链共享一个 deadline——client 5s → API gateway 5s → user-service 5s → DB 5s。下游不能延长上游 deadline。这避免"线程阻塞链式累积"。

### 4.4 推荐配置

| 层 | timeout |
|---|---|
| User-facing API | 30s |
| 内部服务-服务 | 5-10s |
| DB query | 1-3s |
| Cache read | 100-500ms |
| External webhook | 30s |

每层比下游略宽，给 retry / 重连留余量。

---

## 第五章：重试

### 5.1 gRPC built-in retry

```go
serviceConfig := `{
  "methodConfig": [{
    "name": [{ "service": "user.v1.UserService" }],
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2,
      "retryableStatusCodes": ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
    }
  }]
}`
```

### 5.2 何时该重试

仅幂等操作（GET、idempotent PUT/DELETE）。非幂等 POST 重试可能重复扣款。

可重试错误：
- `UNAVAILABLE`：临时不可达
- `DEADLINE_EXCEEDED`：超时（小心 retry 又超时）
- `RESOURCE_EXHAUSTED`：服务过载（带退避）

不该重试：
- `NOT_FOUND`、`INVALID_ARGUMENT`、`PERMISSION_DENIED`
- `INTERNAL`（除非确定无副作用）

### 5.3 指数退避 + 抖动

```
attempt 1: 100ms
attempt 2: 200ms + jitter
attempt 3: 400ms + jitter
attempt 4: 1000ms (max)
```

抖动（random ±10-50%）避免"thundering herd"——所有 client 同时重试压垮 server。

### 5.4 retry budget

避免无限重试雪上加霜：
- 每个 client 最多 10% 的请求是重试
- 服务过载时 retry 也减少

Envoy 等 mesh 有内置 retry budget。

---

## 第六章：熔断（Circuit Breaker）

### 6.1 原理

下游连续失败 → 切断流量一段时间，让它恢复；定期试探。三种状态：

```
Closed (正常)
  ↓ 失败率 > 阈值
Open (拒绝所有请求，立即失败)
  ↓ 经过冷却时间
Half-Open (允许少量探测)
  ↓ 成功 → Closed；失败 → Open
```

### 6.2 实现

Go：`sony/gobreaker`、`afex/hystrix-go`。
Envoy：内置 outlier detection。

### 6.3 与 retry 配合

熔断打开时 retry 也无意义——直接失败给上游，让上游也熔断。这是"故障扩散"的解药。

---

## 第七章：流量控制

### 7.1 HTTP/2 flow control

每个 stream 和连接级都有 window。Server 可以"暂停发送"让 client 消化。gRPC framework 自动处理。

### 7.2 client 端限流

```go
import "golang.org/x/time/rate"

lim := rate.NewLimiter(rate.Limit(100), 200)   // 100 RPS, burst 200
if err := lim.Wait(ctx); err != nil { return err }
client.Call(ctx, req)
```

### 7.3 server 端限流

```go
import "google.golang.org/grpc"

func RateLimit(maxConcurrent int) grpc.UnaryServerInterceptor {
    sem := make(chan struct{}, maxConcurrent)
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
            return handler(ctx, req)
        default:
            return nil, status.Error(codes.ResourceExhausted, "overloaded")
        }
    }
}
```

或基于 token bucket、leaky bucket、Adaptive Concurrency 等。

---

## 第八章：健康检查

### 8.1 gRPC Health Checking Protocol

标准 `.proto`：

```protobuf
service Health {
    rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
}
```

### 8.2 服务端实现

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"

healthSvc := health.NewServer()
healthpb.RegisterHealthServer(s, healthSvc)
healthSvc.SetServingStatus("user.v1.UserService", healthpb.HealthCheckResponse_SERVING)
```

### 8.3 K8s livenessProbe / readinessProbe

```yaml
livenessProbe:
  grpc:
    port: 50051
readinessProbe:
  grpc:
    port: 50051
    service: "user.v1.UserService"
```

K8s 1.24+ 原生支持 gRPC probe。否则用 grpc-health-probe 二进制。

---

## 第九章：keep-alive 与连接管理

### 9.1 server 端

```go
import "google.golang.org/grpc/keepalive"

s := grpc.NewServer(
    grpc.KeepaliveParams(keepalive.ServerParameters{
        MaxConnectionIdle:     5 * time.Minute,
        MaxConnectionAge:      30 * time.Minute,
        MaxConnectionAgeGrace: 5 * time.Second,
        Time:                  30 * time.Second,
        Timeout:               10 * time.Second,
    }),
)
```

- `MaxConnectionAge`：强制定期断开，让 LB 重新分配
- `Time` / `Timeout`：keep-alive ping

### 9.2 client 端

```go
conn, _ := grpc.NewClient(addr,
    grpc.WithKeepaliveParams(keepalive.ClientParameters{
        Time:                30 * time.Second,
        Timeout:             10 * time.Second,
        PermitWithoutStream: true,
    }),
)
```

`PermitWithoutStream: true` 让 ping 不被打断（即使无 RPC 进行）。

### 9.3 连接老化

NAT、Cloud LB 可能在长空闲后丢弃连接。keep-alive ping 让两端确认存活。

---

## 第十章：可观测性

### 10.1 OpenTelemetry interceptor

```go
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

s := grpc.NewServer(grpc.StatsHandler(otelgrpc.NewServerHandler()))
conn, _ := grpc.NewClient(addr, grpc.WithStatsHandler(otelgrpc.NewClientHandler()))
```

自动注入 trace、metric 到 OTel。

### 10.2 metrics 必备

- 每方法 QPS
- 延迟分位（p50/p95/p99）
- 错误率（按 status code 分类）
- 连接数 / 活跃 stream

Prometheus + grpc_prometheus 是经典组合。

### 10.3 日志

```go
func logging(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    slog.InfoContext(ctx, "rpc",
        "method", info.FullMethod,
        "duration", time.Since(start),
        "code", status.Code(err),
    )
    return resp, err
}
```

---

## 第十一章：网关与 REST 桥接

### 11.1 grpc-gateway

由 .proto 自动生成 REST endpoint：

```protobuf
service UserService {
    rpc GetUser(GetUserRequest) returns (User) {
        option (google.api.http) = {
            get: "/v1/users/{id}"
        };
    }
}
```

生成的 gateway 接受 REST `GET /v1/users/42`，转 gRPC 调后端。

### 11.2 envoy gRPC-JSON 转码

类似 gateway，但在 Envoy 中配置——无需额外二进制。

### 11.3 浏览器：gRPC-Web

```
浏览器 → grpc-web → Envoy（grpc-web filter）→ gRPC backend
```

---

## 第十二章：生产级最佳实践

1. **proto 文件单独仓库**：多服务共享 schema，buf 控制 break 变更。
2. **client 必 deadline**：永远不要无限等。
3. **interceptor 链**：recover、metric、log、trace、auth。
4. **K8s headless service + client-side round_robin**：均匀分发。
5. **重试限幂等 + 退避 + 抖动**。
6. **熔断保护下游**：sony/gobreaker 或 envoy outlier。
7. **health check**：让 K8s 自动剔除故障 pod。
8. **MaxConnectionAge**：避免连接长期粘一个后端。
9. **keep-alive ping**：穿越 NAT / Cloud LB。
10. **生产关 reflection**：减小攻击面。

---

## 第十三章：常见陷阱清单

### ❌ 陷阱 1：K8s Service + gRPC 不均匀
ClusterIP 类型 L4 负载均衡 + 长连接 = 流量全到一个 pod。用 headless + client-side LB。

### ❌ 陷阱 2：retry 非幂等
重复扣款。仅 GET 类操作重试，或加 Idempotency-Key。

### ❌ 陷阱 3：deadline 太宽
"5 分钟超时" → 慢 query 拖垮整个系统。每层都该有合理上限。

### ❌ 陷阱 4：忘了 conn.Close()
goroutine + 连接泄漏。

### ❌ 陷阱 5：单 channel 撑超高 QPS
所有 RPC 复用一个 TCP 连接 → 单连接 HOL 与 cwnd 限制。多 channel 或 envoy 解决。

### ❌ 陷阱 6：proto 字段编号重用
旧 client 解析新数据出错。永远 reserved。

### ❌ 陷阱 7：客户端没设置 keep-alive
NAT 60s 超时 → 连接静默断开 → 下次调用慢（重连）。

---

## 第十四章：练习题

**练习 1**：K8s 中 gRPC client 拿到 service IP 后流量不均匀，可能原因和解法？

**练习 2**：为何非幂等 POST 重试危险？给个真实 bug 例子。

**练习 3**：implement 一个 server interceptor，对每方法做"每秒最多 100 调用"限流。

**练习 4**：解释 `MaxConnectionAge` 在生产中的作用。

**练习 5**：浏览器要直连 gRPC backend 需要什么？

---

## 参考答案

**练习 1**：Kubernetes 普通 `Service` 走 kube-proxy L4 LB，HTTP/2 长连接被绑定第一个后端。解法：
- `clusterIP: None`（headless）+ DNS 返回所有 pod IP
- client-side LB `round_robin`
- 或上 Envoy / Istio sidecar

**练习 2**：转账 API：client POST，server 处理完成但响应丢失 → client 超时 retry → 二次扣款。修：用 Idempotency-Key。

**练习 3**：
```go
import "golang.org/x/time/rate"

func RateLimit() grpc.UnaryServerInterceptor {
    limits := sync.Map{}   // method → *rate.Limiter
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        v, _ := limits.LoadOrStore(info.FullMethod, rate.NewLimiter(rate.Limit(100), 100))
        lim := v.(*rate.Limiter)
        if !lim.Allow() {
            return nil, status.Error(codes.ResourceExhausted, "rate limited")
        }
        return handler(ctx, req)
    }
}
```

**练习 4**：强制连接周期性重建。避免：
- 老连接绑定一个 server，新 server 扩容后流量不流向新 pod
- 长连接累积内存
- 不能触发新版本 server 的功能

**练习 5**：gRPC-Web + 代理（Envoy 内置 filter 或 grpcwebproxy）。浏览器不能直接处理 HTTP/2 trailers + binary。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 服务发现 | DNS / xDS / manual |
| 负载均衡 | client-side round_robin + headless service |
| 超时 | client 必设；deadline 跨链传播 |
| 重试 | 仅幂等 + 退避 + 抖动 + budget |
| 熔断 | 防故障扩散 |
| 健康检查 | gRPC Health Protocol + K8s probe |
| keep-alive | NAT 穿越；MaxConnectionAge |
| 观测 | OpenTelemetry interceptor |

下一篇 **B06 — 精通 GraphQL** 会讲清 schema 设计、query/mutation/subscription、resolver、N+1、federation。

---

