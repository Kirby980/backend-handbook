# 系统设计 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始系统设计之旅]) --> M1[模块 1: 方法论与估算]
    M1 --> S01[S01 面试方法论]
    M1 --> S02[S02 容量估算]

    S02 --> M2[模块 2: 可扩展性构件]
    M2 --> S03[S03 接入层与水平扩展]
    M2 --> S04[S04 数据层扩展]
    M2 --> S05[S05 缓存与异步化]

    M2 --> M3[模块 3: 经典案例]
    M3 --> S06[S06 短链]
    M3 --> S07[S07 秒杀]
    M3 --> S08[S08 Feed 流]
    M3 --> S09[S09 IM]
    M3 --> S10[S10 排行榜计数]
    M3 --> S11[S11 网盘]
    M3 --> S12[S12 搜索补全]
    M3 --> S13[S13 附近的人]
    M3 --> S14[S14 延迟队列]
    M3 --> S15[S15 支付订单]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style S07 fill:#ffccbc
    style S15 fill:#ffccbc
```

---

## 🧭 RESHADED 答题框架（每道题都走一遍）

```mermaid
flowchart LR
    R[R 需求<br>Requirements] --> E[E 估算<br>Estimation]
    E --> S[S 接口<br>API/Service]
    S --> H[H 高层设计<br>High-level]
    H --> A[A 数据模型<br>Data/Algo]
    A --> D[D 详细设计<br>Deep dive]
    D --> E2[E 演进扩展<br>Evolve/Scale]
    E2 --> D2[D 权衡讨论<br>Discuss/Tradeoff]

    style R fill:#c8e6c9
    style E fill:#fff9c4
    style H fill:#bbdefb
    style D2 fill:#ffccbc
```

> 顺序不死板，但**需求和估算永远在最前**——跳过这两步直接画框是面试最大扣分项。

---

## 🏛️ 通用可扩展架构骨架

```mermaid
flowchart TB
    Client[客户端 / App] --> DNS[DNS / GSLB]
    DNS --> CDN[CDN / 静态边缘]
    DNS --> LB[L4/L7 负载均衡]
    LB --> GW[API 网关<br>鉴权/限流/路由]

    GW --> SvcR[读服务<br>无状态横向扩展]
    GW --> SvcW[写服务<br>无状态横向扩展]

    SvcR --> LCache[本地缓存]
    SvcR --> RCache[(Redis 集群)]
    RCache -.未命中.-> DBR[(只读副本)]
    SvcW --> DBW[(主库<br>分库分表)]
    DBW -->|复制| DBR

    SvcW -->|削峰/解耦| MQ[(Kafka / RocketMQ)]
    MQ --> Worker[异步消费者<br>幂等]
    Worker --> DBW
    Worker --> Search[(Elasticsearch)]
    Worker --> OSS[(对象存储)]

    style GW fill:#fff3e0
    style RCache fill:#ffcdd2
    style MQ fill:#fff9c4
    style DBW fill:#bbdefb
```

---

## 📈 读密集 vs 写密集：选不同的武器

```mermaid
graph TD
    Q{这道题瓶颈在<br>读还是写?}
    Q -->|读密集<br>如 Feed/详情页| Read[多级缓存<br>读副本<br>CDN<br>读扩散预计算]
    Q -->|写密集<br>如 秒杀/IM/日志| Write[削峰异步化<br>批量合并<br>分库分表<br>LSM/追加写]
    Q -->|读写都高<br>如 排行榜| Both[Redis 内存结构<br>写聚合+近实时<br>冷热分离]

    style Read fill:#c8e6c9
    style Write fill:#ffccbc
    style Both fill:#fff9c4
```

---

## 🔥 秒杀削峰漏斗（S07）

```mermaid
flowchart TD
    U[百万用户点击] --> CDN2[CDN 拦静态/答案页]
    CDN2 --> Limit[网关限流<br>令牌桶/滑窗]
    Limit --> Verify[验证码/答题<br>打散请求]
    Verify --> RedisStock[Redis 原子扣减<br>Lua 防超卖]
    RedisStock -->|扣成功| MQ2[(消息队列<br>异步下单)]
    RedisStock -->|售罄| Fail[快速失败]
    MQ2 --> Order[订单服务<br>真实扣库存]
    Order --> DB[(DB 最终一致)]

    style Limit fill:#fff3e0
    style RedisStock fill:#ffcdd2
    style MQ2 fill:#fff9c4
```

---

## 📰 Feed 流推拉模式（S08）

```mermaid
graph TB
    subgraph Push[写扩散 / 推模式]
        P1[发帖] --> P2[写入每个粉丝的收件箱]
        P2 --> P3[读时直接拉收件箱<br>读快·写放大]
    end

    subgraph Pull[读扩散 / 拉模式]
        L1[发帖只写自己发件箱] --> L2[读时聚合所有关注者<br>写快·读放大]
    end

    subgraph Hybrid[推拉结合 推荐]
        H1[普通用户: 推]
        H2[大 V: 拉]
        H3[读时合并两路]
    end

    style Push fill:#bbdefb
    style Pull fill:#fff9c4
    style Hybrid fill:#c8e6c9
```

---

## 💬 IM 消息可靠投递（S09）

```mermaid
sequenceDiagram
    participant A as 发送方
    participant GW as 接入网关(长连接)
    participant S as 消息服务
    participant B as 接收方

    A->>GW: 发消息(client_msg_id)
    GW->>S: 持久化 + 分配 seq
    S-->>A: ACK(server_msg_id, seq)
    S->>B: 推送(在线)
    B-->>S: 投递 ACK
    Note over S,B: 未 ACK 则重推<br>接收端按 client_msg_id 去重<br>按 seq 保证有序
    S->>B: 离线则转推送通道
```

---

## 💰 支付订单状态机（S15）

```mermaid
stateDiagram-v2
    [*] --> 待支付: 创建订单(幂等)
    待支付 --> 已支付: 支付回调(幂等校验)
    待支付 --> 已关闭: 超时未付/用户取消
    已支付 --> 已完成: 履约成功
    已支付 --> 退款中: 申请退款
    退款中 --> 已退款: 退款成功
    已关闭 --> [*]
    已完成 --> [*]
    已退款 --> [*]

    note right of 已支付
        回调必须幂等
        + 对账兜底
        + 本地消息表保证最终一致
    end note
```

---

## 🎓 学习顺序建议

```mermaid
graph LR
    A[S01-S02<br>方法论+估算] --> B[S03-S05<br>三层构件]
    B --> C[S06-S09<br>四大高频案例]
    C --> D[S10-S15<br>进阶案例]

    style A fill:#c8e6c9
    style C fill:#ffccbc
    style D fill:#fff9c4
```

按顺序读 = 完整答题能力；面试突击 = 看 [INDEX.md](./INDEX.md) 的「路径 A」。
