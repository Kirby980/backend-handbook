# MongoDB 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：MongoDB 8.0 LTS + 9.x**（2026-05 时主流稳定版）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 MongoDB 之旅]) --> M1[模块 1: 数据模型]
    M1 --> M01[M01 文档模型/BSON]
    M1 --> M02[M02 索引]

    M02 --> M2[模块 2: 存储与查询]
    M2 --> M03[M03 WiredTiger]
    M2 --> M04[M04 查询/聚合]

    M04 --> M3[模块 3: 分布式]
    M3 --> M05[M05 副本集]
    M3 --> M06[M06 分片集群]

    M05 --> M4[模块 4: 一致性与性能]
    M06 --> M4
    M4 --> M07[M07 事务/一致性]
    M4 --> M08[M08 性能调优]

    M07 --> M5[模块 5: 现代特性]
    M08 --> M5
    M5 --> M09[M09 Change Streams]
    M5 --> M10[M10 Schema 设计]
    M5 --> M11[M11 安全]
    M5 --> M12[M12 运维/替代品]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#ffe0b2
```

---

## 🔑 一次 find 请求的物理路径

```mermaid
sequenceDiagram
    participant App as 应用
    participant Drv as MongoDB Driver
    participant Mgs as mongos<br>（分片场景）
    participant Pri as Primary mongod
    participant Sec as Secondary mongod
    participant WT as WiredTiger Cache

    App->>Drv: find({user_id: 42})
    Drv->>Mgs: OP_MSG（含 read preference）
    Note over Mgs: 路由：按 shard key 找目标 shard
    alt 单 shard 路由
        Mgs->>Pri: query
    else 跨 shard scatter-gather
        Mgs->>Pri: query
        Mgs->>Sec: query
    end
    Pri->>WT: 查 cache（命中则返回）
    WT-->>Pri: 文档或缺页
    Pri->>Pri: 索引/集合扫描
    Pri-->>Mgs: 文档批
    Mgs-->>Drv: 合并结果
    Drv-->>App: cursor
```

**关键观察**：默认走 **primary**。read preference 选 `secondary` / `nearest` 可分流，但要权衡新鲜度。

---

## 🌲 索引结构

```mermaid
flowchart TD
    Doc["Document<br>#123;user_id, name, age, addr#125;"]
    Doc --> ID[("_id 索引<br>主键 B-tree")]
    Doc --> Sec1[("单字段索引<br>#123;name: 1#125;")]
    Doc --> Sec2[("复合索引<br>#123;user_id: 1, age: -1#125;")]
    Doc --> Geo[("2dsphere 索引<br>地理")]
    Doc --> TTL[("TTL 索引<br>#123;createdAt: 1, expireAfter: 3600#125;")]
    Doc --> Text[("text 索引<br>全文")]
    Doc --> WC[("wildcard 索引<br>#123;'$**': 1#125;")]
    Doc --> Vec[("向量索引<br>vectorSearch 8.0+")]

    style ID fill:#fff3e0
    style Sec2 fill:#c8e6c9
    style Vec fill:#fce4ec
```

> ESR 规则：复合索引字段顺序 **Equality → Sort → Range** 是最优。

---

## 📦 WiredTiger 存储栈

```mermaid
flowchart LR
    Q[Query] --> Cache[WiredTiger Cache<br>~50% RAM]
    Cache -->|命中| Ret[返回]
    Cache -->|未命中| Disk[(磁盘 .wt 文件)]
    Disk --> Cache

    Write[Write] --> JournalBuf[Journal Buffer]
    JournalBuf -->|每 100ms 或 100MB fsync| JFile[(Journal 文件)]
    Write --> Dirty[Cache 中 dirty page]
    Dirty -.checkpoint 每 60s.-> Disk

    style Cache fill:#fff3e0
    style Disk fill:#bbdefb
    style JFile fill:#c8e6c9
```

- **Journal**：WAL，崩溃恢复
- **Checkpoint**：默认 60 秒一次，把 dirty page 刷盘
- **Cache**：默认 (RAM - 1GB) × 50%，是性能关键

---

## 🏗️ 副本集拓扑

```mermaid
graph LR
    subgraph PSS架构
    P[Primary<br>RW]
    S1[Secondary 1<br>RO]
    S2[Secondary 2<br>RO]
    end

    Client[Client] --> P
    Client -.read preference: secondary.-> S1
    P -.oplog 复制.-> S1
    P -.oplog 复制.-> S2
    S1 -.投票.- S2
    S2 -.投票.- P

    style P fill:#fff3e0
    style S1 fill:#c8e6c9
    style S2 fill:#c8e6c9
```

**最小生产**：3 节点 PSS（Primary + 2 Secondary）。

不要用 PSA（2 数据 + 1 Arbiter）—— Arbiter 不存数据，failover 时数据节点只剩 1 个 → 没法形成 quorum 保证安全写入。

---

## 🔪 分片集群拓扑

```mermaid
graph TD
    Cli[Client / App] --> R[mongos Router<br>多副本无状态]

    subgraph "Config Server (CSRS)"
        C1[Config 1]
        C2[Config 2]
        C3[Config 3]
    end

    R -.读元数据.-> C1

    subgraph "Shard 1 (副本集)"
        S1P[Primary]
        S1S1[Secondary]
        S1S2[Secondary]
    end

    subgraph "Shard 2 (副本集)"
        S2P[Primary]
        S2S1[Secondary]
    end

    subgraph "Shard N"
        SNP[Primary]
    end

    R --> S1P
    R --> S2P
    R --> SNP

    Bal[Balancer<br>在 Config 上运行] -.迁移 chunk.-> S1P
    Bal -.迁移 chunk.-> S2P

    style R fill:#fff3e0
    style C1 fill:#bbdefb
    style Bal fill:#fce4ec
```

---

## 🎯 Shard Key 决策

```mermaid
flowchart TD
    Q1[选 shard key]
    Q1 --> Card{基数高?}
    Card -->|低,如status| Bad[❌ 数据集中少数 chunk<br>必然不均衡]
    Card -->|高,如user_id| Q2{写入频率分布?}
    Q2 -->|均匀| Q3{是否单调递增?}
    Q2 -->|集中,如某大客户| Bad2[❌ 热点 shard]
    Q3 -->|是,timestamp| Hot[⚠️ 热点 chunk<br>所有写入到最大 chunk]
    Q3 -->|否,hash 后| Good[✅ Hashed Shard Key<br>或 ranged 但加 hash 前缀]

    style Good fill:#c8e6c9
    style Bad fill:#fce4ec
    style Bad2 fill:#fce4ec
    style Hot fill:#fff3e0
```

> 三原则：**基数 + 频率 + 单调性**。最容易踩坑的是单调递增 key（timestamp、ObjectId 前 4 字节是时间）。

---

## 🛡️ 一致性矩阵

```mermaid
flowchart TD
    W[Write Concern]
    W --> W1["w:1<br>只确认 primary"]
    W --> WM["w:majority<br>多数节点写入"]

    R[Read Concern]
    R --> RL["local<br>看本节点 latest"]
    R --> RM["majority<br>看多数已 commit 的"]
    R --> RS["snapshot<br>事务用快照"]
    R --> RLin["linearizable<br>强一致+确认是 Primary"]

    Sess[Causal Consistency]
    Sess -.client session.-> RM

    EOS{真正强一致?}
    WM --> EOS
    RM --> EOS
    EOS -->|是| WMM["w:majority + r:majority"]

    style WMM fill:#c8e6c9
```

---

## 🆚 MongoDB vs 替代品

```mermaid
flowchart TD
    Choose{需求}
    Choose -->|主流文档库 + 大生态| Mongo[MongoDB 8.0]
    Choose -->|要 SQL 兼容 + AWS| DocDB[AWS DocumentDB<br>API 兼容但底层是 AWS 自研]
    Choose -->|完全 OSS + 兼容 wire 协议| Ferret[FerretDB<br>建在 PostgreSQL 之上]
    Choose -->|多模型 KV/Doc/Graph 一体| Couch[Couchbase]
    Choose -->|无 SSPL 限制 + 自托管| Ferret2[FerretDB]
    Choose -->|要 vector + AI 一站式| Mongo2[MongoDB 8 内置 vectorSearch]

    style Mongo fill:#c8e6c9
    style DocDB fill:#fff3e0
    style Ferret fill:#bbdefb
```

---

## 📈 MongoDB 关键版本时间线

```mermaid
timeline
    title MongoDB 关键里程碑
    2009 : 1.0 首次发布
    2015 : 3.0 引入 WiredTiger 引擎（可选）
    2015 December : 3.2 WiredTiger 成为默认存储引擎
    2018 : 4.0 多文档事务 副本集
    2019 January : SSPL 协议（不是 OSI 开源）
    2020 : 4.4 hedged reads / 复合 hashed key
    2021 : 5.0 Time Series Collections / live resharding
    2022 : 6.0 Queryable Encryption (Preview)
    2023 : 7.0 LTS / Queryable Encryption GA / Time Series 优化
    2024 October : 8.0 LTS / vectorSearch GA / 查询性能提升
    2025 : 9.x 持续演进
    2026 : 主流生产基线 MongoDB 8.0 LTS
```

---

## 🚀 性能分层决策

```mermaid
flowchart TD
    Slow[查询慢]
    Slow --> Idx{用索引了?}
    Idx -->|否| BuildIdx[创建索引<br>ESR 规则]
    Idx -->|是| Cov{覆盖索引?}
    Cov -->|否| AddProj[加合适 projection<br>+ 复合索引含返回字段]
    Cov -->|是| WS{"Working Set < RAM?"}
    WS -->|否| AddRAM[加 RAM 或分片]
    WS -->|是| Profile{看 profiler}
    Profile -->|locks 多| Lock[查热点 collection]
    Profile -->|fetch 多| Fetch[doc 太大?]
    Profile -->|sort 大| Sort[加 sort index]

    style BuildIdx fill:#c8e6c9
    style AddRAM fill:#fff3e0
```

---

