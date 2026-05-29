# 精通 API 网关与 BFF：Envoy、Kong、KrakenD 与 Apollo Router

> 关联章节：[精通同步通信](./04-精通-同步通信.md)、[精通服务发现与注册](./05-精通-服务发现与注册.md)、[精通服务网格](./15-精通-服务网格.md)

---

## 引言：网关是微服务的"前门"，也是责任最重的一段

把所有微服务直接暴露给客户端是不现实的：

- **多服务发现负担**：客户端要知道每个服务的地址
- **横切关注点分散**：每个服务都要实现鉴权 / 限流 / 日志
- **协议不统一**：内部 gRPC vs 外部 REST 难直接桥接
- **客户端体验差**：聚合 5 个服务的数据要发 5 个请求

**API 网关**就是统一入口——所有外部流量先到网关，网关分发、聚合、保护、监控。

但网关有它自己的陷阱：很容易变成"业务层"，最终变成新的单体。本章把这两件事讲清楚：

- 网关的"做什么 / 不做什么"边界
- BFF（Backend for Frontend）作为聚合层的角色
- Envoy / Kong / KrakenD / Apollo Router 选型

读完应该能：

- 在不同场景下选 Envoy / Kong / KrakenD / Apisix
- 区分 API 网关、BFF、Service Mesh 三者的边界
- 设计一个"网关 + BFF + Service Mesh"分层架构
- 配置 Envoy 实现 JWT 鉴权 + 限流 + tracing
- 解释 GraphQL Federation 作为 BFF 替代方案的优劣

---

## 第 1 章：API 网关的 7 大职责

| 职责 | 说明 |
|---|---|
| **路由** | URL / Host / Header → 后端服务 |
| **认证** | JWT 验证、OAuth Token 交换、API Key |
| **授权** | RBAC / ABAC、IP 白名单 |
| **限流** | 全局 / 用户 / API 维度限流 |
| **协议转换** | HTTP → gRPC、gRPC-Web → gRPC、SOAP → REST |
| **可观测** | 统一日志 / 链路 / 指标 |
| **缓存** | 静态资源 + 短时间 API 响应缓存 |

**不应该做的事**：
- 业务逻辑（数据组合、计算、状态管理）
- 长事务（网关应是无状态）
- 复杂的数据聚合（→ BFF 层）

---

## 第 2 章：网关产品对比

### 2.1 主流网关

| 网关 | 数据面 | 控制面 | 性能 | 易用 | 生态 |
|---|---|---|---|---|---|
| **Envoy + Istio** | Envoy（C++） | Istiod | 极高 | 学习曲线陡 | 与 K8s / Service Mesh 深度集成 |
| **Envoy Gateway** | Envoy | 简化版（Gateway API） | 极高 | 中 | K8s 原生 |
| **Apisix** | Nginx + Lua | Etcd | 极高 | 中 | 中国社区强大 |
| **Kong** | Nginx + Lua | DB / DB-less | 高 | 中 | 插件丰富 |
| **KrakenD** | Go | 配置驱动 | 高 | 简单 | 聚合能力强 |
| **Traefik** | Go | 自带 | 高 | 极简 | K8s 友好 |
| **Tyk** | Go | 自带 | 高 | 中 | 商业支持 |
| **Spring Cloud Gateway** | Java | Spring | 中 | Java 生态熟 | Spring 系标配 |
| **AWS API Gateway** | 托管 | 托管 | 中 | 简单 | AWS 集成 |

### 2.2 选型决策

```
你的环境是什么？
├── 纯 K8s + Service Mesh
│   └── Envoy Gateway / Istio Gateway
├── K8s 但不想引 Mesh
│   └── Traefik / Envoy Gateway / Apisix
├── 混合（K8s + VM）
│   └── Kong / Apisix
├── 公网 SaaS
│   └── Apisix / Kong / AWS API Gateway
├── 纯聚合场景（聚合多服务）
│   └── KrakenD / Apollo Router
└── Java 生态封闭
    └── Spring Cloud Gateway
```

### 2.3 Envoy 的特殊地位

Envoy 是 2026 年事实上的网关数据面标准——Istio / Gloo / Envoy Gateway / Apisix（部分）都用 Envoy。原因：

- 高性能（C++ + 异步 IO）
- xDS 协议（动态配置）
- 完善的可观测（metrics / tracing / access log）
- 强大的扩展（WASM Filter）
- HTTP/2、HTTP/3、gRPC、WebSocket 全支持

---

## 第 3 章：Envoy 实战配置

### 3.1 最小路由配置

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress
          route_config:
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/api/orders" }
                route: { cluster: order_service }
              - match: { prefix: "/api/users" }
                route: { cluster: user_service }
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: order_service
    connect_timeout: 1s
    type: STRICT_DNS
    load_assignment:
      cluster_name: order_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: order, port_value: 50051 }
```

### 3.2 JWT 验证

```yaml
http_filters:
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
    providers:
      auth0:
        issuer: https://example.auth0.com/
        audiences: ["api.example.com"]
        remote_jwks:
          http_uri:
            uri: https://example.auth0.com/.well-known/jwks.json
            cluster: auth0_jwks
            timeout: 1s
          cache_duration: 300s
        forward: true   # 把 JWT 转给后端
        payload_in_metadata: jwt_payload  # 把 claims 放进 metadata
    rules:
    - match: { prefix: /api }
      requires: { provider_name: auth0 }
- name: envoy.filters.http.router
```

### 3.3 限流（本地 + 全局）

```yaml
# 本地限流（每实例独立）
http_filters:
- name: envoy.filters.http.local_ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
    stat_prefix: http_local_rate_limiter
    token_bucket:
      max_tokens: 1000
      tokens_per_fill: 1000
      fill_interval: 1s
    filter_enabled:
      runtime_key: local_rate_limit_enabled
      default_value: { numerator: 100, denominator: HUNDRED }
    filter_enforced:
      runtime_key: local_rate_limit_enforced
      default_value: { numerator: 100, denominator: HUNDRED }
```

全局限流要外挂 Rate Limit Service（RLS），用 Redis 后端做跨实例计数。

### 3.4 Tracing

```yaml
tracing:
  http:
    name: envoy.tracers.opentelemetry
    typed_config:
      "@type": type.googleapis.com/envoy.config.trace.v3.OpenTelemetryConfig
      grpc_service:
        envoy_grpc:
          cluster_name: otel_collector
        timeout: 0.250s
      service_name: api-gateway
```

详见 [M17 链路追踪](./17-精通-分布式链路追踪.md)。

---

## 第 4 章：BFF（Backend for Frontend）模式

### 4.1 BFF 是什么

不同客户端（Web / iOS / Android / 第三方 SDK）的数据需求差异大。**为每种客户端写一个"专属后端"**——把后端服务聚合、裁剪、格式化成客户端友好的形式。

```
              ┌── Web BFF ──→ User / Order / Product Services
Client ───────┼── iOS BFF ──→ User / Order / Product Services
              └── Android BFF → User / Order / Product Services
```

### 4.2 BFF 解决什么

- **数据聚合**：客户端 1 个请求 → BFF 调 5 个服务 → 返回组合数据（解决 REST 多次往返）
- **数据裁剪**：客户端只要 5 个字段，BFF 过滤掉其他 50 个
- **协议转换**：BFF 用 REST 暴露给客户端，对内调 gRPC
- **客户端定制**：不同客户端不同字段、不同格式
- **解耦发布**：BFF 跟随客户端发布，不影响内部服务

### 4.3 BFF 不解决什么

- **业务逻辑**：BFF 只编排，不实现业务规则
- **持久化**：BFF 无状态，不直接访问数据库
- **跨客户端共享逻辑**：相同逻辑应该下沉到内部服务

### 4.4 BFF 所有权

经典争论："BFF 属于前端团队还是后端团队？"

答：**理想情况下属于前端团队**。理由：
- BFF 是为客户端"定制"的，前端最清楚需求
- 客户端变更频率高，BFF 跟随快
- 前端开发可以全栈，避免跨团队协作摩擦

实践中常见组织：前端写 BFF（Node.js / Go），后端写微服务（Java / Go）。

### 4.5 BFF 数量爆炸的反模式

不要为每个 UI 屏幕写一个 BFF，那是另一种"分布式单体"。经验：

- 1 个 BFF / 客户端平台（Web / iOS / Android / 第三方）
- 超过这个数量要审视是否拆错了

---

## 第 5 章：GraphQL Federation 作为 BFF 替代

### 5.1 BFF 的痛点

- BFF 数量随客户端增加而增加（维护负担）
- BFF 内的"数据聚合代码"重复（每个 BFF 都要做 User+Order 联查）
- 数据 schema 变更要同步多个 BFF

### 5.2 GraphQL Federation 的解法

让每个微服务声明自己的 GraphQL Subgraph，Apollo Router 自动合并成 Supergraph：

```
                    Apollo Router (Supergraph)
                   /         |         \
               UserSvc    OrderSvc   ProductSvc
              (subgraph)  (subgraph)  (subgraph)
```

客户端发一个查询：

```graphql
query {
  me {
    name
    orders(last: 5) {
      id
      total
      items { name price }
    }
  }
}
```

Router 自动拆分到 UserSvc / OrderSvc / ProductSvc 执行，合并结果。

### 5.3 BFF vs Federation 取舍

| 维度 | BFF | GraphQL Federation |
|---|---|---|
| 客户端工具链 | 简单（REST / JSON） | 需 Apollo Client / 类似工具 |
| 数据按需取 | 需手动设计 | 原生支持 |
| 跨服务数据组合 | BFF 手写 | Router 自动 |
| 服务侵入 | 0（服务无感知） | 服务需暴露 GraphQL schema |
| 学习曲线 | 低 | 中 |
| 灵活度 | 任意客户端定制 | Schema 约束强 |
| 适合规模 | < 5 个客户端 | 多客户端 + 大量服务 |

**2026 趋势**：大型组织（多客户端 + > 50 服务）逐渐迁到 Federation；中小型仍 BFF 主流。

---

## 第 6 章：网关 vs Service Mesh vs BFF 三者边界

```
                            外部流量 (公网)
                                  │
                            ┌─────▼──────┐
                            │  API 网关   │  ← 南北流量入口
                            │  鉴权/限流  │
                            └─────┬──────┘
                                  │
                            ┌─────▼──────┐
                            │   BFF      │  ← 客户端聚合层
                            │  (可选)     │
                            └─────┬──────┘
                                  │
                        ┌─────────┼─────────┐
                        │         │         │
                  ┌─────▼───┐ ┌──▼────┐ ┌──▼────┐
                  │ Service │ │Service│ │Service│  ← 东西流量
                  │   A     │ │   B   │ │   C   │
                  └─────────┘ └───────┘ └───────┘
                          ↑    ↑   ↑       ↑
                          └────┴───┴───────┘
                          Service Mesh（mTLS / 流量治理 / 可观测）
```

| 组件 | 流量方向 | 主要职责 | 边界 |
|---|---|---|---|
| **API 网关** | 南北（外 → 内） | 入口流量统一处理 | 不做业务、不聚合 |
| **BFF** | 客户端 ↔ 服务 | 数据聚合、协议转换 | 只编排，不业务 |
| **Service Mesh** | 东西（服务间） | mTLS / 韧性 / 可观测 | 应用透明 |

经验：**网关 + Mesh 是基础设施**，**BFF 是应用代码**。

---

## 第 7 章：完整网关案例

### 7.1 场景

一个电商 SaaS，外部 API + 移动端 App + Web 端。

### 7.2 拓扑

```
公网 → CDN → WAF → API 网关 (Envoy)
                       │
       ┌───────────────┼───────────────┐
       │               │               │
   公开 API         移动 BFF         Web BFF
   /v1/...        /mobile/...      /web/...
       │               │               │
       │       ┌───────┴───────┐       │
       │       │               │       │
       └──→ Internal Services (mTLS, gRPC)
            User / Order / Product / Payment / Inventory
```

### 7.3 网关配置职责

- TLS 终止
- 限流（按 API key / IP）
- JWT 验证
- 路由分发（公开 API / BFF）
- 全链路 trace 入口
- 静态资源 cache

### 7.4 BFF 职责

- 调用多个内部 gRPC 服务
- 数据聚合 / 裁剪
- 客户端定制格式（移动端简化字段）
- 错误统一处理（内部 gRPC error → REST Problem Details）

### 7.5 Mesh 职责

- 服务间 mTLS（SPIFFE 身份）
- 流量切分（金丝雀）
- 熔断 / 重试
- 服务级 metric / trace

---

## 第 8 章：网关的高可用

### 8.1 网关本身要 HA

```
        DNS / VIP
            │
   ┌────────┼────────┐
   │        │        │
GW Pod 1  GW Pod 2  GW Pod 3
   │        │        │
   └────────┼────────┘
            ▼
       后端服务
```

K8s 上：Deployment + Service + HPA。

### 8.2 灰度与回滚

网关的配置变更比应用变更更危险——一行错配可能瘫痪所有流量。

实践：
- 配置走 GitOps（Git → CI → Apply）
- 蓝绿网关（双套网关，DNS 切换）
- 渐进式配置推送（10% → 50% → 100%）

### 8.3 健康检查与熔断

网关到后端服务的健康检查（Envoy outlier detection）：

```yaml
outlier_detection:
  consecutive_5xx: 5
  interval: 10s
  base_ejection_time: 30s
  max_ejection_percent: 50
```

5 个连续 5xx 后摘除该实例 30 秒。

---

## 第 9 章：WebSocket / SSE 透传

### 9.1 WebSocket

```yaml
- name: envoy.filters.network.http_connection_manager
  typed_config:
    upgrade_configs:
    - upgrade_type: websocket
```

注意：WebSocket 是长连接，网关需要支持大量并发连接 + idle timeout 配置。

### 9.2 SSE（Server-Sent Events）

SSE 是单向 HTTP 长连接，网关需要：

- 不启用响应缓冲
- 超时不要触发（无限或很长）
- HTTP/2 复用（建议）

---

## 第 10 章：反模式

### 10.1 网关变业务层

症状：网关 Lua 脚本 / WASM Filter 实现业务规则（折扣计算、订单状态变更）。

后果：业务发布 = 网关发布 = 全局风险。

修复：业务逻辑下沉到服务，网关只做横切。

### 10.2 BFF 直接访问数据库

症状：BFF 不调内部服务，直接连 DB。

后果：DB schema 变更影响多个 BFF；业务逻辑分散在 BFF。

修复：BFF 只调内部服务，禁止直接 DB。

### 10.3 网关大泥球

症状：单个网关同时承担：外部 API、内部网关、BFF、协议转换、业务编排。

后果：单点风险极高、调试困难、变更危险。

修复：分层（南北网关 + BFF + Mesh），各司其职。

### 10.4 BFF 数量爆炸

症状：每个 UI 页面一个 BFF。

后果：30 个 BFF + 30 个微服务 = 60 个服务的运维负担。

修复：限制 BFF 粒度（1 个 / 客户端平台）。

---

## 生产实践

1. **Envoy + Istio / Envoy Gateway**：K8s 环境的首选数据面
2. **网关只做横切，不做业务**：业务逻辑下沉到服务
3. **BFF 由前端团队拥有**：减少跨团队协作摩擦
4. **BFF 不直接访问 DB**：始终通过内部服务
5. **网关变更走 GitOps + 渐进发布**：配置错误的杀伤力巨大
6. **网关 + Mesh 各司其职**：网关做南北，Mesh 做东西
7. **TLS 终止在网关层**：内部用 mTLS（Mesh 自动注入）
8. **限流分层**：CDN（连接级） + 网关（API 级） + 服务（业务级）
9. **JWT 在网关验证**：转发解析后的 claims 到后端（Header / metadata）
10. **公开 API 用 REST + OpenAPI**：内部用 gRPC + Mesh
11. **大型组织迁 GraphQL Federation**：减少 BFF 维护负担
12. **网关高可用 ≥ 3 实例 + 自动扩缩**：HPA 按 CPU / 连接数

---

## 陷阱清单

1. **网关把业务逻辑写进 Lua / WASM** → 网关发布 = 业务发布 = 高风险
2. **BFF 直接访问数据库** → 绕过服务层，破坏微服务边界
3. **BFF 数量 > 客户端平台数量** → 维护爆炸
4. **网关单实例 / 没做 HA** → 全公司流量入口故障
5. **网关配置变更不走灰度** → 单次错配瘫痪生产
6. **TLS 终止后内部裸 HTTP** → 东西流量未加密
7. **限流只在网关做** → 服务直接被调用绕过限流
8. **JWT 在每个服务重复验证** → 性能浪费，应在网关验证一次
9. **WebSocket 网关 idle timeout 太短** → 连接频繁断开
10. **网关 access log 没结构化** → 日志分析困难
11. **网关与 Mesh 重复做 mTLS** → 性能浪费 + 配置复杂
12. **Apollo Router 暴露给公网无复杂度限制** → 攻击者写深查询打垮服务

---

## 2026 现状

| 主题 | 状态 |
|---|---|
| **Envoy Gateway** | K8s 原生网关事实标准（基于 Gateway API） |
| **Istio Gateway** | 复杂场景仍主流 |
| **Apisix** | 国内 + 多云场景流行 |
| **Kong** | 老牌网关，企业版商业支持 |
| **Apollo Router** | GraphQL Federation 主流网关（Rust 重写性能强） |
| **Gateway API** | K8s GA，逐步替代 Ingress |
| **AWS API Gateway / GCP Cloud Endpoints** | 托管网关，云原生集成 |
| **GraphQL Federation** | 大公司 BFF 层主流方案 |
| **Connect-RPC** | 浏览器友好 gRPC，BFF 简化 |
| **WASM Filter** | Envoy 自定义扩展主流；Proxy-WASM 标准成熟 |

---

## 练习题

1. 一个公司有 30 微服务、3 客户端（Web/iOS/Android）。是否需要 BFF？

   **答案**：建议 3 个 BFF（每客户端一个），减少客户端到服务的网络往返、做客户端定制聚合。但如果服务数 > 50 且数据需求复杂，可以考虑 GraphQL Federation 替代多个 BFF。

2. 网关层 JWT 验证 vs 服务层 JWT 验证。优劣？

   **答案**：网关验证：减少重复逻辑、统一管理。代价：服务无法独立部署（需依赖网关）。**最佳实践**：网关验证签名 + 把 claims 转发到服务 metadata，服务做业务级授权（不重复验证签名）。

3. Envoy 限流 1000 QPS，但实际生产能跑 5000 QPS。为什么？

   **答案**：可能是 (1) 本地限流多副本（5 个网关 × 1000 = 5000）；(2) 没启用全局限流（external rate limit service）；(3) 限流规则匹配不全。生产高峰前要压测验证。

4. BFF 设计：移动端 BFF 内调用 5 个 gRPC 服务，P99 latency 300ms。怎么优化？

   **答案**：(1) 并发调用（goroutine + WaitGroup / Future）而不是串行；(2) 部分服务用 SSE / 流式推送；(3) 关键服务结果缓存；(4) 非关键服务异步调用。

5. 公开 API 接口 `POST /v1/orders`，要求"客户端重试不重复下单"。怎么设计？

   **答案**：要求客户端发 `Idempotency-Key` Header。网关层做 key 验证（必填）。服务层用 Redis 缓存 key + 响应 24h（→ M09）。

6. 网关到 5 个后端实例。其中 1 个偶发 5xx（< 1%）。Envoy 怎么处理？

   **答案**：outlier detection 配 `consecutive_5xx: 5`，5 个连续 5xx 摘除该实例 30s。同时配 retry 让单次失败不影响客户端。

7. GraphQL Federation 在 BFF 层，攻击者发出嵌套 50 层的查询。怎么防？

   **答案**：(1) 查询复杂度分析（每字段权重打分）；(2) 查询深度限制（如最大 10 层）；(3) 持久化查询（Persisted Queries，只允许预注册的查询）；(4) 网关层限流 + 鉴权。

8. 一个公司"网关 / BFF / Mesh"三层都有，但实际工程师把业务逻辑塞进了网关。怎么纠正？

   **答案**：(1) 制定明确边界文档；(2) 把现有业务逻辑迁回服务层（PR by PR）；(3) 网关配置评审强制（任何 Lua / WASM 修改要 review）；(4) CI 检测网关配置中的业务关键字（如 SQL / 业务规则）。

---

## 延伸阅读

- Envoy 官方文档：[envoyproxy.io/docs](https://www.envoyproxy.io/docs)
- Envoy Gateway：[gateway.envoyproxy.io](https://gateway.envoyproxy.io/)
- Apisix：[apisix.apache.org](https://apisix.apache.org/)
- Kong：[konghq.com](https://konghq.com/)
- KrakenD：[krakend.io](https://www.krakend.io/)
- Apollo Router：[apollographql.com/docs/router](https://www.apollographql.com/docs/router/)
- BFF Pattern：[samnewman.io/patterns/architectural/bff](https://samnewman.io/patterns/architectural/bff/)
- Gateway API（K8s）：[gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/)
- 关联章节：[M04 同步通信](./04-精通-同步通信.md)、[M05 服务发现](./05-精通-服务发现与注册.md)、[M11 限流熔断降级](./11-精通-限流熔断降级.md)、[M15 服务网格](./15-精通-服务网格.md)、[M21 微服务安全](./21-精通-微服务安全.md)
