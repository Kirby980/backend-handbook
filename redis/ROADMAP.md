# Redis 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：Redis 8.0 / Valkey 8.x**（2026-05 时主流稳定版）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Redis 之旅]) --> M1[模块 1: 底层内核]

    M1 --> R01[R01 数据结构内部]
    M1 --> R02[R02 内存与过期]
    M1 --> R03[R03 持久化]

    R01 --> M2[模块 2: 分布式]
    R02 --> M2
    R03 --> M2

    M2 --> R04[R04 复制与 Sentinel]
    M2 --> R05[R05 Cluster]

    R04 --> M3[模块 3: 编程模型]
    R05 --> M3

    M3 --> R06[R06 事务/Pipeline/脚本]
    M3 --> R07[R07 Streams/Pub-Sub]

    R06 --> M4[模块 4: 性能与新特性]
    R07 --> M4

    M4 --> R08[R08 性能模型]
    M4 --> R09[R09 Redis 8 新类型]

    R08 --> M5[模块 5: 生态与生产]
    R09 --> M5

    M5 --> R10[R10 生产实践]
    M5 --> R11[R11 Redis vs Valkey]
    M5 --> R12[R12 客户端]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#ffe0b2
```

---

## 🔑 内部数据结构选择决策

```mermaid
flowchart TD
    Type{数据类型?}

    Type -->|String| Str{长度 <= 44?}
    Str -->|是| EmbStr[embstr 紧凑分配]
    Str -->|否| RawStr[raw SDS]
    Type -->|String 纯整数| IntStr[int 编码]

    Type -->|List| ListCfg[quicklist<br>每节点 listpack]

    Type -->|Hash/Set/ZSet| Small{元素少且单值小?}
    Small -->|是| LP[listpack 紧凑]
    Small -->|否 Hash| HT[hashtable]
    Small -->|否 Set 全整数| IS[intset]
    Small -->|否 Set 字符串| HT2[hashtable]
    Small -->|否 ZSet| SkipDict[skiplist + dict<br>双结构]

    Type -->|Stream| RT[Radix Tree<br>+ 每节点 listpack]
    Type -->|Bitmap| BM[String 复用]
    Type -->|HyperLogLog| HLL[String 复用 12KB]

    style EmbStr fill:#fff3e0
    style LP fill:#fff3e0
    style SkipDict fill:#fce4ec
    style RT fill:#fce4ec
```

> 关键阈值（Redis 8 默认）：listpack ↔ 大结构切换由 `hash-max-listpack-entries`、`list-max-listpack-size`、`set-max-listpack-entries`、`zset-max-listpack-entries` 控制。

---

## 🔄 持久化策略选择

```mermaid
flowchart LR
    Need{数据丢失容忍度?}

    Need -->|可以丢几分钟| OnlyRDB[纯 RDB<br>save 配置]
    Need -->|秒级| Hybrid[混合 RDB+AOF<br>aof-use-rdb-preamble yes<br>everysec fsync]
    Need -->|不能丢| AOF_Always[AOF always fsync<br>性能下降 10x+]
    Need -->|纯缓存 重启 OK| Disable[全部关闭<br>save '' / appendonly no]

    style Hybrid fill:#c8e6c9
    style AOF_Always fill:#ffcdd2
```

---

## 📡 复制拓扑选型

```mermaid
graph LR
    Single[单实例] --> Master_Slave[主从复制<br>读写分离]
    Master_Slave --> Sentinel_Cluster{需要自动故障转移?}

    Sentinel_Cluster -->|不分片| Sentinel[Sentinel 模式<br>3+ Sentinel 节点]
    Sentinel_Cluster -->|要分片| Cluster[Redis Cluster<br>16384 slot<br>gossip 协议]

    Cluster --> Proxy{客户端不支持 Cluster?}
    Proxy -->|是| Proxy_Layer[Twemproxy/Predixy<br>已不推荐]
    Proxy -->|否| Cluster_Aware[Cluster-aware<br>客户端 MOVED/ASK]
```

---

## ⚡ 性能调优速决树

```mermaid
flowchart TD
    Slow[Redis 慢]

    Slow --> Q1{slowlog 有命令?}
    Q1 -->|有| Cmd[KEYS / SMEMBERS bigkey / 复杂 Lua<br>→ 改成 SCAN / 分片 / 拆 key]
    Q1 -->|无| Q2{INFO clients<br>连接数高?}

    Q2 -->|高| Conn[client OUTPUT_BUFFER 满 / TIME_WAIT<br>→ 连接池 + RESP3 single conn multiplexing]
    Q2 -->|不高| Q3{latency monitor<br>事件?}

    Q3 -->|expire-cycle 卡顿| Expire[改 hz / lazyfree<br>active expire CPU 占用]
    Q3 -->|fork / aof_fsync| Persist[改 fsync 策略 / appendfsync<br>关大 RDB 改 fork-less]
    Q3 -->|backlog full| Replication[扩 repl-backlog-size<br>主从带宽]
    Q3 -->|无| Q4{INFO memory<br>used_memory / used_memory_rss?}

    Q4 -->|碎片率 > 1.5| Defrag[开 activedefrag yes]
    Q4 -->|内存接近 maxmemory| Evict[调 maxmemory-policy<br>看 evicted_keys]
```

---

## 🆕 Redis 8 / Valkey 8 关键演进时间线

```mermaid
timeline
    title Redis 关键里程碑
    2009 : Redis 1.0
    2013 : Redis 2.6 Lua 脚本
    2015 : Redis 3.0 Cluster GA
    2017 : Redis 4.0 模块系统 + 混合持久化
    2018 : Redis 5.0 Streams
    2020 : Redis 6.0 RESP3 + I/O 多线程 + ACL
    2021 : Redis 7.0 Functions + sharded Pub/Sub + ACL v2
    2024 March : 改双许可证 RSALv2/SSPL → 引发 Valkey fork
    2024 April : Valkey fork (Linux Foundation, BSD-3)
    2025 May : Redis 8.0 重新开源 (AGPLv3) + 集成模块
    2026 : Redis 8.x / Valkey 8.x 并存格局
```

---

> 📁 本路线图位于 `/data/workspace/dp4/redis/ROADMAP.md`
