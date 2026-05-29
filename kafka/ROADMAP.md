# Kafka 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：Apache Kafka 4.0**（2025 GA，默认 KRaft）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Kafka 之旅]) --> M1[模块 1: 核心模型]
    M1 --> K01[K01 Topic/Partition/Offset]
    M1 --> K02[K02 KRaft 元数据]

    M1 --> M2[模块 2: 编程接口]
    M2 --> K03[K03 Producer]
    M2 --> K04[K04 Consumer/KIP-848]

    M2 --> M3[模块 3: 可靠性]
    M3 --> K05[K05 ISR/HW/LEO]
    M3 --> K07[K07 精确一次/事务]

    M2 --> M4[模块 4: 性能]
    M4 --> K06[K06 存储/Segment]
    M4 --> K08[K08 性能调优]

    M3 --> M5[模块 5: 生态]
    M4 --> M5
    M5 --> K09[K09 Streams]
    M5 --> K10[K10 Connect/Schema Registry]
    M5 --> K11[K11 安全]
    M5 --> K12[K12 运维/替代品]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#ffe0b2
```

---

## 🔑 一条消息的端到端旅程

```mermaid
sequenceDiagram
    participant App as 应用
    participant Prod as Producer Client
    participant Bk1 as Broker (Leader)
    participant Bk2 as Broker (Follower 1)
    participant Bk3 as Broker (Follower 2)
    participant Cons as Consumer Client

    App->>Prod: send(record)
    Prod->>Prod: 序列化 → partitioner → accumulator (buffer)
    Note right of Prod: 批量 + 压缩

    Prod->>Bk1: Produce RPC (一批)
    Bk1->>Bk1: append 到 partition leader 日志
    par
        Bk2->>Bk1: Fetch (复制)
        Bk3->>Bk1: Fetch (复制)
    end
    Bk1->>Bk1: 等 ISR ACK
    Bk1->>Prod: ProduceResponse (offset)
    Prod->>App: callback(metadata)

    loop poll loop
        Cons->>Bk1: Fetch from leader
        Bk1->>Cons: records (up to HW)
        Cons->>Cons: 处理
        Cons->>Bk1: OffsetCommit
    end
```

---

## 📦 Partition 物理布局

```mermaid
flowchart TD
    Topic[Topic: orders<br>6 partitions]
    Topic --> P0[Partition 0<br>Leader: B1<br>ISR: B1,B2,B3]
    Topic --> P1[Partition 1<br>Leader: B2<br>ISR: B1,B2,B3]
    Topic --> P2[Partition 2<br>Leader: B3<br>ISR: B1,B2,B3]
    Topic --> P3[Partition 3<br>Leader: B1<br>ISR: B1,B2,B3]
    Topic --> P4[Partition 4<br>Leader: B2<br>ISR: B1,B2,B3]
    Topic --> P5[Partition 5<br>Leader: B3<br>ISR: B1,B2,B3]

    P0 --> Seg[多个 Segment 文件<br>00000000.log 00000000.index<br>00100000.log 00100000.index<br>...]

    Seg --> Index[.index 稀疏索引<br>offset → file position]
    Seg --> TimeIdx[.timeindex<br>timestamp → offset]
    Seg --> Log[.log 实际数据]
```

---

## 🗳️ KRaft Controller Quorum

```mermaid
graph LR
    subgraph KRaft Controller Quorum
    C1[Controller 1<br>Leader] -.Raft.- C2[Controller 2<br>Voter]
    C2 -.Raft.- C3[Controller 3<br>Voter]
    C3 -.Raft.- C1
    end

    subgraph Brokers
    B1[Broker 1<br>+ optional Controller]
    B2[Broker 2]
    B3[Broker 3]
    B4[Broker 4]
    end

    C1 -.MetadataLog 推送.-> B1
    C1 -.MetadataLog 推送.-> B2
    C1 -.MetadataLog 推送.-> B3
    C1 -.MetadataLog 推送.-> B4

    style C1 fill:#fff3e0
```

> Kafka 4.0 起 ZooKeeper 完全移除。Controller 数量推荐 3 或 5（quorum 大小 = (N+1)/2）。`process.roles=broker,controller` 可让一台机同时担两职（dev / 小集群）。

---

## 🔒 KIP-848 新 vs 老 Consumer 协议

```mermaid
sequenceDiagram
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    participant Bk as Broker (Group Coordinator)

    rect rgba(255,200,200,0.3)
        Note over C1,Bk: 老协议（stop-the-world rebalance）
        C2-->>Bk: JoinGroup（新成员加入）
        Bk-->>C1: SyncGroup (你 revoke 所有 partition)
        Bk-->>C2: SyncGroup (你 revoke 所有 partition)
        Note right of Bk: 此刻全 group 停止处理
        Bk-->>C1: 新分配 [p0,p1]
        Bk-->>C2: 新分配 [p2,p3]
        Note over C1,Bk: 恢复处理
    end

    rect rgba(200,255,200,0.3)
        Note over C1,Bk: KIP-848 新协议（增量、broker 主导）
        C2-->>Bk: ConsumerGroupHeartbeat（加入）
        Bk->>Bk: 计算增量分配方案
        Bk-->>C2: 你拿 [p2,p3]
        Note right of C1: C1 持续处理 [p0,p1] 不中断
        Bk-->>C1: 下次心跳告知最终态
    end
```

---

## 🛡️ 可靠性等级矩阵

```mermaid
flowchart TD
    Q[配置]
    Q --> A1[acks=0<br>fire and forget]
    Q --> A2[acks=1<br>leader 写入即返]
    Q --> A3[acks=all + min.insync.replicas=2<br>多数派持久]

    A3 --> Idem{idempotent=true?}
    Idem -->|是| ExactlyP[exactly-once 写入<br>单 partition]
    Idem --> Txn{transactional=true?}
    Txn -->|是| EOS[跨 partition exactly-once<br>+ consumer read-committed]

    style A3 fill:#c8e6c9
    style EOS fill:#fff3e0
```

---

## 🚀 性能极限决策

```mermaid
flowchart TD
    Need{需求}
    Need -->|纯吞吐, 可丢数据| Throughput[acks=0 + compression=lz4<br>+ 大 batch.size + linger.ms]
    Need -->|低延迟单条| Latency[acks=1 + batch.size 小 + linger.ms=0]
    Need -->|强可靠 + 高吞吐| Balanced[acks=all + ISR=2 + lz4 + 适中 batch + idempotent]
    Need -->|多 partition 事务| TXN[acks=all + transactional + isolation=read_committed<br>注意 LSO 延迟]

    style Balanced fill:#c8e6c9
```

---

## 🆕 Kafka 关键里程碑

```mermaid
timeline
    title Kafka 历史
    2011 : 0.7 LinkedIn 开源
    2014 : 0.8 副本机制
    2017 : 0.11 事务 / 精确一次
    2019 : 2.4 incremental cooperative rebalance
    2021 : 2.8 KRaft early access
    2022 : 3.3 KRaft production ready (KIP-833)
    2023 : 3.6 Tiered Storage (Early Access)
    2024 : 3.7-3.9 KRaft 与 ZK bridge 模式完善
         : 3.9 Tiered Storage GA（生产可用）+ 动态 KRaft quorum KIP-853
    2025 : 4.0 (3 月) ZK 完全移除 / KIP-848 GA
         : 4.1 (年末) 持续优化与新特性
    2026 : 主流生产基线 Kafka 4.x KRaft / 替代品 Redpanda Pulsar 成熟
```

---

## 🆚 现代消息系统对比

```mermaid
graph TD
    Choose{需求}

    Choose -->|生态 + 标准 + 工具完备| Kafka[Apache Kafka 4.0<br>KRaft / 主流]
    Choose -->|极低延迟 + 单进程 + 无 JVM| Redpanda[Redpanda<br>C++ Raft + Kafka API]
    Choose -->|多租户 + 地理复制 + 长存储| Pulsar[Apache Pulsar<br>broker stateless + BookKeeper]
    Choose -->|海量冷数据 + 成本敏感| WS[WarpStream<br>stateless broker + S3]

    style Kafka fill:#fff3e0
```

---

