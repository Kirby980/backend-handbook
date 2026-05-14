# Backend 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 5 月**——HTTP/3 主流、TLS 1.3 + post-quantum (X25519MLKEM768)、PostgreSQL 18、Redis 8 / Valkey、Kafka 4 (KRaft)、Istio Ambient GA、K8s Gateway API GA、Passkeys、OAuth 2.1 + DPoP、OTel、Prometheus 3。

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Backend 之旅]) --> M1[模块 1: 网络基础]
    
    M1 --> B01[B01 互联网原理]
    M1 --> B02[B02 HTTP 语义]
    M1 --> B03[B03 TLS 与 HTTPS]
    
    M1 --> M2[模块 2: API 设计]
    M2 --> B04[B04 REST]
    M2 --> B05[B05 gRPC 进阶]
    M2 --> B06[B06 GraphQL]
    M2 --> B07[B07 OpenAPI]
    
    M2 --> M3[模块 3: 数据库]
    M3 --> B08[B08 索引]
    M3 --> B09[B09 事务/ACID]
    M3 --> B10[B10 规范化]
    M3 --> B11[B11 分片]
    M3 --> B12[B12 复制/CAP]
    M3 --> B13[B13 N+1]
    M3 --> B14[B14 NoSQL]
    
    M3 --> M4[模块 4: 数据加速]
    M4 --> B15[B15 缓存策略]
    M4 --> B16[B16 Redis]
    M4 --> B17[B17 消息队列]
    
    M4 --> M5[模块 5: 架构与韧性]
    M5 --> B18[B18 微服务]
    M5 --> B19[B19 12-Factor]
    M5 --> B20[B20 韧性模式]
    M5 --> B21[B21 限流]
    
    M5 --> M6[模块 6: 安全与运维]
    M6 --> B22[B22 认证]
    M6 --> B23[B23 OWASP]
    M6 --> B24[B24 可观测性]
    M6 --> B25[B25 Web 服务器]
    
    M6 --> End([Backend 高级工程师])
    
    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef net fill:#48bb78,stroke:#2f855a,color:#fff
    classDef api fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef db fill:#ecc94b,stroke:#b7791f,color:#000
    classDef accel fill:#f56565,stroke:#c53030,color:#fff
    classDef arch fill:#9f7aea,stroke:#6b46c1,color:#fff
    classDef ops fill:#ed8936,stroke:#c05621,color:#fff
    
    class M1,M2,M3,M4,M5,M6 module
    class B01,B02,B03 net
    class B04,B05,B06,B07 api
    class B08,B09,B10,B11,B12,B13,B14 db
    class B15,B16,B17 accel
    class B18,B19,B20,B21 arch
    class B22,B23,B24,B25 ops
```

---

## 🟢 模块 1：网络基础（B01-B03）

```mermaid
graph LR
    B01[B01 互联网原理<br>DNS/TCP/HTTP/QUIC] --> B02[B02 HTTP 语义<br>methods/status/cache]
    B02 --> B03[B03 TLS+HTTPS<br>证书/握手/mTLS]
    
    classDef net fill:#48bb78,stroke:#2f855a,color:#fff
    class B01,B02,B03 net
```

**核心心智模型**：

```mermaid
graph TB
    Browser[浏览器] -->|DNS| DNS[DNS Server]
    DNS -->|IP| Browser
    Browser -->|TCP 3-way| Server[Web Server]
    Server -->|TLS handshake| Browser
    Browser -->|HTTP request| Server
    Server -->|HTTP response| Browser
    
    style DNS fill:#ecc94b
    style Server fill:#48bb78
```

---

## 🔵 模块 2：API 设计（B04-B07）

```mermaid
graph TB
    B04[B04 REST<br>资源/版本/分页]
    B05[B05 gRPC<br>HTTP/2 + protobuf]
    B06[B06 GraphQL<br>schema + resolver]
    B07[B07 OpenAPI<br>契约 + 代码生成]
    
    B07 -.约束.-> B04
    B05 -.对比.-> B04
    B06 -.对比.-> B04
    
    classDef api fill:#4299e1,stroke:#2b6cb0,color:#fff
    class B04,B05,B06,B07 api
```

**API 选型决策树**：

```mermaid
flowchart TD
    Start[新 API] --> Internal{内部还是外部?}
    Internal -->|内部| Perf{需要高性能?}
    Internal -->|外部| Public{公开 web?}
    
    Perf -->|是| gRPC[gRPC]
    Perf -->|否| REST1[REST]
    
    Public -->|是 + 多视图| GraphQL[GraphQL]
    Public -->|是 + 标准| REST2[REST + OpenAPI]
    Public -->|是 + 简单| REST2
    
    style gRPC fill:#4299e1,color:#fff
    style REST1 fill:#48bb78,color:#fff
    style REST2 fill:#48bb78,color:#fff
    style GraphQL fill:#9f7aea,color:#fff
```

---

## 🟡 模块 3：数据库（B08-B14）

```mermaid
graph TD
    B08[B08 索引] --> B09[B09 事务/ACID]
    B08 --> B13[B13 N+1 问题]
    B09 --> B10[B10 规范化]
    B10 --> B11[B11 分片]
    B11 --> B12[B12 复制/CAP]
    B10 -.-> B14[B14 NoSQL 选型]
    B12 -.-> B14
    
    classDef db fill:#ecc94b,stroke:#b7791f,color:#000
    class B08,B09,B10,B11,B12,B13,B14 db
```

**数据库选型决策**：

```mermaid
flowchart TD
    Start[选 DB] --> Rel{强关系 + 事务?}
    Rel -->|是| Scale{量级?}
    Rel -->|否| Pattern{访问模式?}
    
    Scale -->|< 数 TB| PG[PostgreSQL]
    Scale -->|> PB + 跨地域| NewSQL[CockroachDB/TiDB]
    
    Pattern -->|按 key 取| KV{需复杂结构?}
    Pattern -->|嵌套文档| Mongo[MongoDB]
    Pattern -->|高吞吐写入 + 时间序列| Cassandra[Cassandra]
    Pattern -->|关系跳跃查询| Neo4j[Neo4j]
    Pattern -->|时间序列度量| TS[InfluxDB/Timescale]
    Pattern -->|全文搜索| ES[Elasticsearch]
    
    KV -->|否| Redis[Redis]
    KV -->|是| Mongo
    
    style PG fill:#ecc94b
    style Redis fill:#f56565,color:#fff
    style Mongo fill:#48bb78,color:#fff
    style Cassandra fill:#9f7aea,color:#fff
```

---

## 🔴 模块 4：数据加速（B15-B17）

```mermaid
graph LR
    B15[B15 缓存策略<br>cache-aside/雪崩/穿透] --> B16[B16 Redis<br>数据结构/集群]
    B16 -.-> B17[B17 消息队列<br>Kafka/RabbitMQ/NATS]
    
    classDef accel fill:#f56565,stroke:#c53030,color:#fff
    class B15,B16,B17 accel
```

**缓存陷阱对照**：

```mermaid
graph TB
    Issue{缓存问题}
    Issue --> Avalanche[雪崩<br>大量同时过期]
    Issue --> Penetration[穿透<br>查不存在的 key]
    Issue --> Breakdown[击穿<br>单热 key 过期]
    
    Avalanche --> SolveA[随机 TTL]
    Penetration --> SolveB[空值缓存<br>+ Bloom filter]
    Breakdown --> SolveC[互斥锁 / 永不过期 + 后台刷]
    
    style Avalanche fill:#f56565,color:#fff
    style Penetration fill:#f56565,color:#fff
    style Breakdown fill:#f56565,color:#fff
    style SolveA fill:#48bb78,color:#fff
    style SolveB fill:#48bb78,color:#fff
    style SolveC fill:#48bb78,color:#fff
```

---

## 🟣 模块 5：架构与韧性（B18-B21）

```mermaid
graph TB
    B18[B18 微服务架构]
    B19[B19 12-Factor]
    B20[B20 韧性模式]
    B21[B21 限流/背压]
    
    B19 -.支撑.-> B18
    B18 -.依赖.-> B20
    B20 -.结合.-> B21
    
    classDef arch fill:#9f7aea,stroke:#6b46c1,color:#fff
    class B18,B19,B20,B21 arch
```

**韧性模式工具箱**：

```mermaid
graph TB
    Request[请求] --> Timeout{Timeout}
    Timeout -->|超时| Fail[失败]
    Timeout -->|未超时| Breaker{Circuit Breaker}
    Breaker -->|Open| FastFail[立即失败/降级]
    Breaker -->|Closed| Limiter{Rate Limiter}
    Limiter -->|超限| Reject[429]
    Limiter -->|放行| Call[调下游]
    Call -->|失败| Retry{Retry}
    Retry -->|可重试 + 幂等| Backoff[退避后重试]
    Retry -->|否| Fail2[向上抛]
    Call -->|成功| OK[OK]
    
    style FastFail fill:#f56565,color:#fff
    style Reject fill:#ecc94b
    style OK fill:#48bb78,color:#fff
```

---

## 🟠 模块 6：安全与运维（B22-B25）

```mermaid
graph TB
    B22[B22 认证<br>session/JWT/OAuth]
    B23[B23 OWASP Top 10<br>注入/IDOR/SSRF]
    B24[B24 可观测性<br>logs/metrics/traces]
    B25[B25 Web 服务器<br>Nginx/Caddy/Envoy]
    
    B22 -.-> B23
    B25 -.前置.-> B22
    B23 -.监控.-> B24
    
    classDef ops fill:#ed8936,stroke:#c05621,color:#fff
    class B22,B23,B24,B25 ops
```

**生产架构图**：

```mermaid
graph TB
    Client[Client] --> CDN[CDN]
    CDN --> WAF[WAF + DDoS]
    WAF --> LB[Cloud LB]
    LB --> Gateway[API Gateway<br>Auth + Rate Limit]
    Gateway --> Ingress[K8s Ingress<br>Nginx / Traefik]
    Ingress --> Mesh[Service Mesh<br>Envoy sidecars]
    Mesh --> Apps[Application Pods]
    
    Apps --> DB[(PostgreSQL)]
    Apps --> Cache[(Redis)]
    Apps --> MQ[(Kafka)]
    Apps --> Search[(Elasticsearch)]
    
    Apps -.metrics.-> Prom[Prometheus]
    Apps -.logs.-> Loki[Loki]
    Apps -.traces.-> Tempo[Tempo]
    Prom & Loki & Tempo --> Grafana[Grafana]
    
    classDef edge fill:#48bb78,color:#fff
    classDef app fill:#4299e1,color:#fff
    classDef data fill:#ecc94b
    classDef obs fill:#ed8936,color:#fff
    
    class Client,CDN,WAF,LB edge
    class Gateway,Ingress,Mesh,Apps app
    class DB,Cache,MQ,Search data
    class Prom,Loki,Tempo,Grafana obs
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（半年）

```mermaid
gantt
    title 6 个月 Backend 通学
    dateFormat YYYY-MM-DD
    section 月 1
    B01-B03 网络基础       :a1, 2026-05-12, 14d
    B04-B05 REST + gRPC    :a2, after a1, 14d
    section 月 2
    B06-B07 GraphQL + OpenAPI   :a3, after a2, 14d
    B08-B09 索引 + 事务         :a4, after a3, 14d
    section 月 3
    B10-B14 规范化到 NoSQL :a5, after a4, 30d
    section 月 4
    B15-B17 缓存与消息     :a6, after a5, 30d
    section 月 5
    B18-B21 架构与韧性     :a7, after a6, 30d
    section 月 6
    B22-B25 安全与运维     :a8, after a7, 30d
```

### 路径 B：API 工程师特化

```mermaid
graph LR
    A1[B01-B03<br>网络底盘] --> A2[B04 REST]
    A2 --> A3[B07 OpenAPI]
    A3 --> A4[B05/B06<br>gRPC/GraphQL]
    A4 --> A5[B22 认证]
    A5 --> A6[B20-B21<br>韧性 + 限流]
    
    style A1 fill:#48bb78
    style A2 fill:#4299e1
    style A3 fill:#4299e1
    style A4 fill:#4299e1
    style A5 fill:#ed8936
    style A6 fill:#9f7aea
```

### 路径 C：数据库专精

```mermaid
graph LR
    D1[B08 索引] --> D2[B09 事务]
    D2 --> D3[B12 复制]
    D3 --> D4[B10-B11<br>规范化 + 分片]
    D4 --> D5[B13 N+1]
    D5 --> D6[B15 缓存]
    D6 --> D7[B14 NoSQL]
    
    style D1 fill:#ecc94b
    style D2 fill:#ecc94b
    style D3 fill:#ecc94b
    style D4 fill:#ecc94b
    style D5 fill:#ecc94b
    style D6 fill:#f56565,color:#fff
    style D7 fill:#ecc94b
```

---

## 🧠 后端核心知识思维导图

```mermaid
mindmap
  root((Backend 路线图))
    网络
      DNS B01
      HTTP/1-2-3 B01
      HTTPS+TLS B03
      HTTP 语义 B02
    API
      REST B04
      gRPC B05
      GraphQL B06
      OpenAPI B07
    数据库
      索引 B08
      事务 B09
      规范化 B10
      分片 B11
      复制 CAP B12
      N+1 B13
      NoSQL 选型 B14
    数据加速
      缓存策略 B15
      Redis B16
      消息队列 B17
    架构
      微服务 B18
      12-Factor B19
      韧性 B20
      限流 B21
    安全运维
      认证 B22
      OWASP B23
      可观测 B24
      Web 服务器 B25
```

---

## 📊 难度与重要性矩阵

> Mermaid 的 `quadrantChart` 在多数渲染器（GitHub、VS Code、Hugo 等）兼容性差，这里改用表格。
> 难度：⭐ 简单 ⭐⭐ 中等 ⭐⭐⭐ 进阶 ⭐⭐⭐⭐ 难 ⭐⭐⭐⭐⭐ 极难
> 重要性：🔥🔥🔥🔥🔥 必学 → 🔥 选学

### 必学 + 难（高优先级，要花时间啃）

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| B08 索引 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 性能问题第一根因 |
| B09 事务 ACID | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 隔离级别 / MVCC 底子 |
| B11 分片 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 上规模必撞 |
| B12 复制/CAP | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 分布式系统认知基线 |
| B17 消息队列 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 解耦 / 异步主力 |
| B22 认证 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | OAuth 2.1 / Passkey 必懂 |
| B24 可观测性 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 没有就是黑盒 |

### 必学 + 简单（性价比最高）

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| B01 互联网原理 | ⭐⭐ | 🔥🔥🔥🔥🔥 | 一切上层的物理基础 |
| B02 HTTP 语义 | ⭐⭐ | 🔥🔥🔥🔥🔥 | 每天用，常被误用 |
| B03 TLS / HTTPS | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | TLS 1.3 + PQ 已主流 |
| B04 REST | ⭐⭐ | 🔥🔥🔥🔥🔥 | 接口规范基本功 |
| B07 OpenAPI | ⭐⭐⭐ | 🔥🔥🔥🔥 | 文档驱动开发 |
| B13 N+1 | ⭐⭐ | 🔥🔥🔥🔥🔥 | 95% 应用都中招 |
| B19 12-Factor | ⭐⭐ | 🔥🔥🔥🔥🔥 | 云原生入门 |

### 必学 + 中等

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| B05 gRPC | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 内部服务首选 |
| B10 规范化 | ⭐⭐⭐ | 🔥🔥🔥🔥 | 反范式何时用 |
| B15 缓存策略 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | Cache-aside / Write-through |
| B16 Redis | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | KV 之外的丰富用法 |
| B18 微服务 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 何时不该拆 |
| B20 韧性 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 重试 / 熔断 / 隔离 |
| B21 限流 | ⭐⭐⭐ | 🔥🔥🔥🔥 | token bucket / sliding window |
| B23 OWASP | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | Top 10 必知 |
| B25 Web 服务器 | ⭐⭐⭐ | 🔥🔥🔥🔥 | Nginx / Envoy 怎么放 |

### 选学（按需深入）

| 课程 | 难度 | 重要性 | 何时学 |
|---|---|---|---|
| B06 GraphQL | ⭐⭐⭐⭐ | 🔥🔥🔥 | BFF / 移动端聚合接口时 |
| B14 NoSQL | ⭐⭐⭐ | 🔥🔥🔥 | 文档库 / 时序 / 图场景 |

---

## 🔗 跨模块知识连接

```mermaid
graph TD
    Network[网络基础 B01-B03]
    API[API 设计 B04-B07]
    DB[数据库 B08-B14]
    Cache[缓存 + MQ B15-B17]
    Arch[架构韧性 B18-B21]
    Sec[安全运维 B22-B25]
    
    Network --> API
    API --> Arch
    DB --> Cache
    Cache --> Arch
    Arch --> Sec
    
    %% 横向关联
    DB -. 索引调优 .-> N1[B13 N+1]
    Cache -. 失效策略 .-> N2[B12 一致性]
    Arch -. 下游故障 .-> N3[B20 韧性]
    Sec -. 限流前置 .-> N4[B25 Nginx]
    Sec -. 认证前置 .-> N5[B22 in middleware]
    
    style Network fill:#48bb78,color:#fff
    style API fill:#4299e1,color:#fff
    style DB fill:#ecc94b
    style Cache fill:#f56565,color:#fff
    style Arch fill:#9f7aea,color:#fff
    style Sec fill:#ed8936,color:#fff
```

---

## ✅ 阶段性自检

### 模块 1（网络）后

```mermaid
graph LR
    Q1[404 vs 410<br>差别?] -.-> A1[404 不知道, 410 永久消失]
    Q2[HSTS 撤回难<br>原因?] -.-> A2[preload list 难撤]
    Q3[HTTP/3 不在 TCP<br>因为什么?] -.-> A3[避 HOL blocking]
    Q4[Vary 头<br>用来干什么?] -.-> A4[多版本响应 cache 分层]
```

### 模块 3（数据库）后

```mermaid
graph LR
    Q1[复合索引<br>列顺序?] -.-> A1[等值-范围-排序]
    Q2[READ COMMITTED<br>vs REPEATABLE READ?] -.-> A2[同事务内重读差别]
    Q3[CAP 中 CA<br>存在吗?] -.-> A3[分布式不存在]
    Q4[N+1 修复<br>三种?] -.-> A4[eager / JOIN / DataLoader]
```

### 模块 5（韧性）后

```mermaid
graph LR
    Q1[retry 不能用<br>什么场景?] -.-> A1[非幂等 + 无 Idempotency-Key]
    Q2[circuit breaker<br>三态?] -.-> A2[Closed / Open / Half-Open]
    Q3[token bucket<br>vs leaky?] -.-> A3[burst 容忍差别]
    Q4[Bulkhead<br>是什么?] -.-> A4[资源隔离防故障扩散]
```

---

## 📌 一键速查表

| 想做什么 | 看哪章 |
|---|---|
| HTTPS 证书 | B03 |
| 设计 REST 错误响应 | B04 |
| 用 protobuf | B05、G28 |
| 加索引但还是慢 | B08 |
| 转账并发 lost update | B09 |
| 高 QPS 数据库扛不住 | B11、B15 |
| 选 PostgreSQL 还是 MongoDB | B14 |
| Redis 缓存设计 | B15、B16 |
| 异步事件 / Kafka | B17 |
| 微服务通信故障 | B20 |
| 限流策略 | B21 |
| 登录系统 | B22 |
| OWASP 自查 | B23 |
| 看不到 prod 在干啥 | B24 |
| Nginx 反代配置 | B25 |

---

## 🆕 2026 关键技术演进图

```mermaid
graph LR
    subgraph 网络与安全
    N1[HTTP/3<br>RFC 9114] --> N2[QUIC v2 / Multipath]
    N3[TLS 1.3] --> N4[X25519MLKEM768<br>后量子默认]
    N4 --> N5[ECH<br>Encrypted ClientHello]
    end

    subgraph 数据存储
    D1[PostgreSQL 17] --> D2[PostgreSQL 18<br>异步 I/O · UUIDv7]
    D3[Redis BSD] --> D4[Redis 7.4 RSALv2/SSPL]
    D4 --> D5[Redis 8 AGPLv3]
    D4 --> D6[Valkey BSD-3]
    D7[Kafka ZK] --> D8[Kafka 3.3 KRaft GA]
    D8 --> D9[Kafka 4.0 移除 ZK<br>KIP-848 新协议]
    end

    subgraph 平台基础设施
    P1[K8s Ingress] --> P2[Gateway API v1.0 GA]
    P3[Istio Sidecar] --> P4[Istio Ambient GA<br>2024-11]
    P5[Cilium CNI] --> P6[Cilium Service Mesh<br>eBPF]
    end

    subgraph 认证授权
    A1[Password] --> A2[Passkeys 主流<br>5亿+ 在用]
    A3[OAuth 2.0] --> A4[OAuth 2.1<br>PKCE 强制]
    A4 --> A5[DPoP RFC 9449<br>sender-constrained]
    end

    subgraph 可观测
    O1[Jaeger / Zipkin] --> O2[OpenTelemetry<br>CNCF Graduated]
    O3[Prometheus 2.x] --> O4[Prometheus 3.0<br>native histograms / OTLP]
    end

    style N4 fill:#fff3e0
    style D2 fill:#fff3e0
    style D5 fill:#fff3e0
    style D9 fill:#fff3e0
    style P4 fill:#fff3e0
    style A2 fill:#fff3e0
    style A5 fill:#fff3e0
    style O4 fill:#fff3e0
```

---

> 📁 本路线图位于 `/data/workspace/dp4/backend/ROADMAP.md`
> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [QUIZ.md](./QUIZ.md) 测验
