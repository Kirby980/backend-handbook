# PostgreSQL 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：PostgreSQL 18 + 17**（2026-05 时主流稳定版；PostgreSQL 无官方 LTS，每个大版本支持 5 年）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 PostgreSQL 之旅]) --> M1[模块 1: 架构基础]
    M1 --> P01[P01 整体架构]
    M1 --> P02[P02 堆表/TOAST]
    M1 --> P03[P03 MVCC]

    P03 --> M2[模块 2: 索引体系]
    M2 --> P04[P04 B-tree]
    M2 --> P05[P05 GIN/GiST/BRIN]
    M2 --> P06[P06 pgvector]

    P03 --> M3[模块 3: 事务/VACUUM]
    M3 --> P07[P07 隔离/锁/SSI]
    M3 --> P08[P08 VACUUM/膨胀]

    P04 --> M4[模块 4: 查询优化]
    M4 --> P09[P09 Planner]
    M4 --> P10[P10 EXPLAIN]
    M4 --> P11[P11 调优实战]

    P10 --> M5[模块 5: 高级 SQL]
    M5 --> P12[P12 JSONB/全文]
    M5 --> P13[P13 窗口/CTE/MERGE]
    M5 --> P14[P14 分区表]

    P08 --> M6[模块 6: 复制/HA]
    M6 --> P15[P15 WAL/物理流复制]
    M6 --> P16[P16 逻辑复制]
    M6 --> P17[P17 Patroni/PgBouncer]

    P14 --> M7[模块 7: 扩展生态]
    M7 --> P18[P18 PostGIS/Timescale/Citus]
    M7 --> P19[P19 FDW]

    P17 --> M8[模块 8: 生产化]
    M8 --> P20[P20 参数调优]
    M8 --> P21[P21 监控诊断]
    M8 --> P22[P22 PG 18/17 新特性]

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

## 🏛️ PostgreSQL 进程模型

```mermaid
flowchart TB
    Client1[客户端 1] --> Pg1
    Client2[客户端 2] --> Pg2
    Client3[客户端 N] --> PgN

    Postmaster[postmaster<br>监听 5432<br>fork backend] -.fork.-> Pg1[backend 1<br>独立进程]
    Postmaster -.fork.-> Pg2[backend 2<br>独立进程]
    Postmaster -.fork.-> PgN[backend N<br>独立进程]

    subgraph Shared[共享内存 shared_buffers]
        BufferPool[Buffer Pool<br>8KB 页缓存]
        WALBuf[WAL Buffer]
        CLog[CLOG<br>事务状态]
        LockTable[锁表]
    end

    Pg1 --> Shared
    Pg2 --> Shared
    PgN --> Shared

    subgraph BG[后台进程]
        BgWriter[bgwriter<br>脏页刷盘]
        Checkpointer[checkpointer<br>检查点]
        WalWriter[walwriter<br>WAL 落盘]
        AutoVac[autovacuum launcher<br>+ workers]
        Stats[stats collector<br>PG 15+ 改用共享内存]
        LogicalRep[logical rep launcher]
    end

    Shared --> BgWriter
    Shared --> Checkpointer
    Shared --> WalWriter
    Shared --> AutoVac
    Shared --> Stats

    BgWriter --> Disk[(磁盘<br>base/ + pg_wal/ + pg_xact/)]
    Checkpointer --> Disk
    WalWriter --> Disk

    style Postmaster fill:#fff3e0
    style Shared fill:#bbdefb
    style BG fill:#c8e6c9
```

**关键差异（vs MySQL）**：PG 是**多进程**模型（每连接 1 进程，约 10MB），MySQL 是多线程；这意味着 PG **必须配 PgBouncer 连接池**，否则 1000 并发 = 10GB+ 内存。

---

## 🧬 MVCC 行版本演化

```mermaid
sequenceDiagram
    participant T1 as 事务 100<br>(xid=100)
    participant Heap as 堆表行
    participant T2 as 事务 101
    participant T3 as 事务 102

    Note over Heap: 初始: xmin=50, xmax=0, data="A"
    T1->>Heap: UPDATE 把 A 改成 B
    Note over Heap: 旧版本: xmin=50, xmax=100<br>新版本: xmin=100, xmax=0, data="B"
    T1->>T1: COMMIT
    T2->>Heap: SELECT (snapshot 含 100)
    Heap-->>T2: 看到新版本 "B"
    T3->>Heap: SELECT (snapshot 不含 100)
    Heap-->>T3: 看到旧版本 "A"
    Note over Heap: 当所有快照都 >= 101 时<br>autovacuum 回收旧版本
```

**与 MySQL 的本质差异**：
- MySQL：旧版本走 undo log，由 purge 线程清理
- PostgreSQL：**旧版本就在堆里**，autovacuum 必须把它们标记为 dead 并回收——这就是 PG 独有的"膨胀"问题来源

---

## 🌳 索引家族决策树

```mermaid
graph TD
    Query{你的查询类型?}
    Query -->|等值/范围/排序| BTree[B-tree<br>默认选择]
    Query -->|JSONB 包含查询<br>数组成员<br>全文检索| GIN[GIN 倒排索引]
    Query -->|几何/范围类型<br>最近邻| GiST[GiST 通用搜索树]
    Query -->|空间分区数据<br>电话号码/IP 前缀| SPGiST[SP-GiST 空间分区]
    Query -->|超大时序表<br>顺序相关数据| BRIN[BRIN 块范围<br>体积极小]
    Query -->|精确等值<br>不需要范围| Hash[Hash<br>PG 10+ WAL]
    Query -->|向量相似性<br>RAG/推荐| Vector[pgvector<br>HNSW / IVFFlat]

    BTree --> Tip1[支持 INCLUDE 覆盖<br>+ deduplication]
    GIN --> Tip2[体积大、更新慢<br>但查询极快]
    GiST --> Tip3[PostGIS 默认索引]
    BRIN --> Tip4[10GB 表索引可能只有几 MB]
    Vector --> Tip5[2026 主推 HNSW]

    style BTree fill:#c8e6c9
    style GIN fill:#fff9c4
    style Vector fill:#e1bee7
```

---

## 🔒 锁等级图

```mermaid
graph TD
    LockType{PostgreSQL 锁}
    LockType --> Table[表锁<br>8 个等级]
    LockType --> Row[行锁<br>4 类]
    LockType --> Page[页锁<br>很少直接接触]
    LockType --> Advisory[Advisory Lock<br>应用级]

    Table --> AS[ACCESS SHARE<br>SELECT]
    Table --> RS[ROW SHARE<br>SELECT FOR UPDATE]
    Table --> RE[ROW EXCLUSIVE<br>INSERT/UPDATE/DELETE]
    Table --> SUE[SHARE UPDATE EXCLUSIVE<br>VACUUM/CREATE INDEX CONCURRENTLY]
    Table --> S[SHARE<br>CREATE INDEX]
    Table --> SRE[SHARE ROW EXCLUSIVE]
    Table --> E[EXCLUSIVE]
    Table --> AE[ACCESS EXCLUSIVE<br>DROP/TRUNCATE/ALTER]

    Row --> FU[FOR UPDATE<br>排他]
    Row --> FNKU[FOR NO KEY UPDATE]
    Row --> FS[FOR SHARE]
    Row --> FKS[FOR KEY SHARE]

    style Table fill:#ffccbc
    style Row fill:#bbdefb
```

---

## 🔄 WAL + 复制流

```mermaid
flowchart LR
    subgraph Primary[主库]
        Backend[Backend 进程] -->|生成 WAL 记录| WALBuf[WAL Buffer<br>共享内存]
        WALBuf -->|COMMIT 时 fsync| WALFile[(pg_wal/<br>16MB 段)]
        WALFile --> Archive[archive_command<br>归档到对象存储]
        WALFile --> WalSender[walsender<br>1 进程/备库]
    end

    WalSender -->|流复制 TCP| WalReceiver[walreceiver<br>备库]

    subgraph Standby[备库]
        WalReceiver --> StandbyWAL[(pg_wal/)]
        StandbyWAL --> Startup[startup recovery<br>redo apply]
        Startup --> StandbyData[(base/<br>堆表+索引)]
        Standby2Client[Hot Standby<br>只读查询] --> StandbyData
    end

    Slot[replication slot<br>防 WAL 清理] -.绑定.- WalSender

    style WALBuf fill:#fff9c4
    style WALFile fill:#bbdefb
    style WalSender fill:#c8e6c9
    style WalReceiver fill:#c8e6c9
```

---

## 🩹 VACUUM 生命周期

```mermaid
stateDiagram-v2
    [*] --> Live: INSERT
    Live --> Updated: UPDATE
    Updated --> Dead: 老版本变 dead
    Live --> Dead: DELETE
    Dead --> Vacuumed: autovacuum 标记为可重用
    Vacuumed --> Reused: 后续 INSERT/UPDATE 复用槽位
    Live --> Frozen: 经过 vacuum_freeze_min_age<br>(默认 5000 万事务)
    Frozen --> [*]: 永久可见<br>避免 wraparound
    Reused --> [*]
```

---

## 📊 查询规划器决策

```mermaid
flowchart TD
    SQL[SQL 文本] --> Parse[Parser<br>词法/语法 → 解析树]
    Parse --> Analyze[Analyzer<br>列/表/权限解析 → query tree]
    Analyze --> Rewrite[Rewriter<br>视图展开/规则]
    Rewrite --> Planner{Planner<br>表数 < geqo_threshold?}
    Planner -->|是 默认 12| DP[动态规划<br>枚举所有 join 顺序]
    Planner -->|否| GEQO[遗传算法 GEQO]
    DP --> Cost[Cost Model<br>seq/random_page_cost<br>+ pg_statistic]
    GEQO --> Cost
    Cost --> Plan[执行计划树]
    Plan --> Executor[Executor<br>Volcano 模型逐节点 pull]
    Executor --> Result[结果]

    style Planner fill:#fff3e0
    style Cost fill:#bbdefb
    style Executor fill:#c8e6c9
```

---

## 🚦 学习顺序建议

```mermaid
graph LR
    A[P01-P03<br>架构与 MVCC] --> B[P04-P06<br>索引]
    B --> C[P07-P08<br>事务与 VACUUM]
    C --> D[P09-P11<br>查询优化]
    D --> E[P12-P14<br>高级 SQL]
    E --> F[P15-P17<br>复制与 HA]
    F --> G[P18-P19<br>扩展]
    G --> H[P20-P22<br>生产化]

    style A fill:#c8e6c9
    style C fill:#ffccbc
    style F fill:#ffe0b2
```

按顺序读 = 完整心智模型；按需读 = 看 [INDEX.md](./INDEX.md) 中的"学习路径"小节。
