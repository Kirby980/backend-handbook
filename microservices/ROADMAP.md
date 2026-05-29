# 微服务架构 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 5 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始微服务之旅]) --> M1[模块 1: 基础与拆分]
    M1 --> M01[M01 架构总览]
    M1 --> M02[M02 服务拆分 DDD]
    M1 --> M03[M03 演进策略]

    M03 --> M2[模块 2: 通信]
    M2 --> M04[M04 同步通信]
    M2 --> M05[M05 服务发现]
    M2 --> M06[M06 API 网关 BFF]

    M2 --> M3[模块 3: 异步与一致性]
    M3 --> M07[M07 异步消息]
    M3 --> M08[M08 分布式事务]
    M3 --> M09[M09 幂等]
    M3 --> M10[M10 ES/CQRS]

    M3 --> M4[模块 4: 韧性]
    M4 --> M11[M11 限流熔断]
    M4 --> M12[M12 韧性模式]
    M4 --> M13[M13 分布式锁]

    M4 --> M5[模块 5: 治理]
    M5 --> M14[M14 配置中心]
    M5 --> M15[M15 服务网格]
    M5 --> M16[M16 分布式 ID]

    M5 --> M6[模块 6: 可观测]
    M6 --> M17[M17 链路追踪]
    M6 --> M18[M18 可观测三支柱]

    M6 --> M7[模块 7: 发布与测试]
    M7 --> M19[M19 灰度发布]
    M7 --> M20[M20 契约测试 / 混沌]

    M7 --> M8[模块 8: 安全与反模式]
    M8 --> M21[M21 微服务安全]
    M8 --> M22[M22 反模式与重构]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#e1bee7
    style M6 fill:#ffe0b2
    style M7 fill:#d7ccc8
    style M8 fill:#cfd8dc
```

---

## 🏛️ 微服务核心组件全景

```mermaid
flowchart TB
    Client[客户端] --> CDN[CDN / WAF]
    CDN --> Gateway[API 网关<br>Envoy / Kong]
    Gateway --> BFF[BFF<br>聚合层]

    BFF --> SvcA[Service A]
    BFF --> SvcB[Service B]
    BFF --> SvcC[Service C]

    SvcA <-->|gRPC + mTLS| SvcB
    SvcB <-->|事件| MQ[(Kafka / NATS)]
    MQ --> SvcD[Service D]
    SvcC --> SvcD

    SvcA --> CacheA[(Redis)]
    SvcA --> DBA[(PostgreSQL)]
    SvcB --> DBB[(MySQL)]
    SvcC --> DBC[(MongoDB)]

    subgraph Platform[平台基础设施]
        SD[服务发现<br>Consul / etcd]
        Config[配置中心<br>Apollo / Nacos]
        Vault[密钥<br>Vault]
        Mesh[服务网格<br>Istio Ambient]
    end

    SvcA -.注册.- SD
    SvcA -.读配置.- Config
    SvcA -.密钥.- Vault
    SvcA -.sidecar/ambient.- Mesh

    subgraph Observe[可观测]
        Otel[OpenTelemetry Collector]
        Prom[Prometheus]
        Loki[Loki]
        Tempo[Tempo]
        Grafana[Grafana]
    end

    SvcA -.metrics/traces/logs.- Otel
    Otel --> Prom
    Otel --> Loki
    Otel --> Tempo
    Prom --> Grafana
    Loki --> Grafana
    Tempo --> Grafana

    style Gateway fill:#fff3e0
    style Mesh fill:#bbdefb
    style MQ fill:#fff9c4
    style Observe fill:#e1bee7
```

---

## 🧩 DDD 战略设计

```mermaid
graph LR
    subgraph Domain[领域]
        Sub1[核心子域<br>Core]
        Sub2[支撑子域<br>Supporting]
        Sub3[通用子域<br>Generic]
    end

    Sub1 --> BC1[限界上下文 1<br>订单]
    Sub1 --> BC2[限界上下文 2<br>支付]
    Sub2 --> BC3[限界上下文 3<br>库存]
    Sub3 --> BC4[限界上下文 4<br>通知]

    BC1 -.防腐层 ACL.- BC2
    BC1 -.发布订阅.- BC3
    BC2 -.客户-供应商.- BC4

    style Sub1 fill:#ffccbc
    style Sub2 fill:#fff9c4
    style Sub3 fill:#c8e6c9
```

---

## 🔁 Saga（编排模式）

```mermaid
sequenceDiagram
    participant C as Client
    participant O as Order Service
    participant P as Payment Service
    participant I as Inventory Service
    participant N as Notification Service

    C->>O: 创建订单
    O->>O: 状态=PENDING
    O->>P: 扣款命令
    P-->>O: 扣款成功事件
    O->>I: 扣库存命令
    I-->>O: 库存不足！补偿事件
    O->>P: 退款补偿命令
    P-->>O: 退款完成
    O->>O: 状态=FAILED
    O->>N: 通知失败
    N-->>C: 失败提醒
```

---

## ⚡ Circuit Breaker 三态

```mermaid
stateDiagram-v2
    [*] --> Closed: 初始
    Closed --> Open: 错误率 > 阈值<br>(如 50% in 10s)
    Open --> HalfOpen: 等待 sleep_window<br>(如 30s)
    HalfOpen --> Closed: trial 成功
    HalfOpen --> Open: trial 失败
    Closed --> Closed: 请求成功
    Open --> Open: 拒绝请求<br>(快速失败 + fallback)
```

---

## 🌐 服务网格 Sidecar vs Ambient

```mermaid
graph TB
    subgraph SidecarMode[Sidecar 模式（Istio 1.x 经典）]
        AppA[App Pod] --- SidecarA[Envoy sidecar]
        AppB[App Pod] --- SidecarB[Envoy sidecar]
        SidecarA <-->|mTLS| SidecarB
    end

    subgraph AmbientMode[Ambient 模式（Istio 1.22+ GA）]
        AppC[App Pod]
        AppD[App Pod]
        Ztunnel1[ztunnel<br>node-level L4]
        Ztunnel2[ztunnel]
        Waypoint[Waypoint Proxy<br>L7 按 namespace]
        AppC --> Ztunnel1
        Ztunnel1 <-->|HBONE mTLS| Ztunnel2
        Ztunnel2 --> AppD
        Ztunnel1 -.可选 L7.-> Waypoint
    end

    style SidecarMode fill:#ffccbc
    style AmbientMode fill:#c8e6c9
```

---

## 🎯 OpenTelemetry 数据流

```mermaid
flowchart LR
    App[应用 + Otel SDK] --> Otel[Otel Collector<br>DaemonSet]
    Otel --> Prom[Prometheus<br>Metrics]
    Otel --> Loki[Loki<br>Logs]
    Otel --> Tempo[Tempo<br>Traces]
    Otel --> Storage[(对象存储<br>S3 / GCS)]

    Prom --> Grafana
    Loki --> Grafana
    Tempo --> Grafana

    Grafana --> Alert[Alertmanager]
    Grafana --> SLO[SLO Dashboard]

    style Otel fill:#fff3e0
    style Grafana fill:#bbdefb
```

---

## 🚦 灰度发布流程

```mermaid
flowchart LR
    V1[v1: 100% 生产流量] --> Deploy[部署 v2 0%]
    Deploy --> Cy1[Canary 1%]
    Cy1 --> Check1{SLO 通过?}
    Check1 -->|否| Rollback[回滚到 v1]
    Check1 -->|是| Cy5[Canary 5%]
    Cy5 --> Check2{SLO 通过?}
    Check2 -->|否| Rollback
    Check2 -->|是| Cy25[Canary 25%]
    Cy25 --> Cy50[Canary 50%]
    Cy50 --> Cy100[100% v2]
    Cy100 --> Cleanup[清理 v1]
    Rollback --> Investigate[复盘]

    style Rollback fill:#ffcdd2
    style Cy100 fill:#c8e6c9
```

---

## 🛡️ 零信任（Zero Trust）服务身份链

```mermaid
flowchart LR
    Workload[Workload] -->|启动| SVID[SPIFFE SVID<br>spiffe://acme.com/sa/order]
    SVID --> Citadel[Citadel/SPIRE<br>签发]
    Citadel --> Cert[X.509 cert<br>5min lifetime]
    Cert --> mTLS[mTLS handshake]
    mTLS --> Verify[对端验证身份]
    Verify --> Authz[基于 SVID 做授权]

    style SVID fill:#fff3e0
    style Cert fill:#bbdefb
```

---

## 🚧 反模式速查

```mermaid
mindmap
  root((微服务反模式))
    分布式单体
      共享数据库
      链式同步调用
      事务跨服务
      统一发布
    过细拆分
      服务多过工程师
      简单 CRUD 服务
      Chatty Services
    错误的边界
      按 CRUD 拆
      按技术层拆
      忽视上下文
    缺乏治理
      没有契约测试
      没有版本策略
      没有可观测
    Dual Write
      DB+消息双写不一致
    Big Ball of Mud
      微服务名义/单体本质
```

---

## 🎓 学习顺序建议

```mermaid
graph LR
    A[M01-M03<br>基础与拆分] --> B[M04-M06<br>通信]
    B --> C[M07-M10<br>异步与一致性]
    C --> D[M11-M13<br>韧性]
    D --> E[M14-M16<br>治理]
    E --> F[M17-M18<br>可观测]
    F --> G[M19-M20<br>发布与测试]
    G --> H[M21-M22<br>安全与反模式]

    style A fill:#c8e6c9
    style C fill:#fff9c4
    style D fill:#ffccbc
    style H fill:#cfd8dc
```

按顺序读 = 完整心智模型；按需读 = 看 [INDEX.md](./INDEX.md) 中的 6 条学习路径。
