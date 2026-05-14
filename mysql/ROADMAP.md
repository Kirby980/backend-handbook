# MySQL 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：MySQL 8.4 LTS + 9.x**（2026-05 时主流稳定版）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 MySQL 之旅]) --> M1[模块 1: 基础架构]
    M1 --> M01[M01 整体架构]
    M1 --> M02[M02 InnoDB 索引]

    M02 --> M2[模块 2: 事务与并发]
    M2 --> M03[M03 事务/MVCC]
    M2 --> M04[M04 锁机制]

    M04 --> M3[模块 3: 高可用]
    M3 --> M05[M05 Binlog/复制]
    M3 --> M09[M09 高可用架构]

    M02 --> M4[模块 4: 性能]
    M04 --> M4
    M4 --> M06[M06 EXPLAIN/优化器]
    M4 --> M07[M07 Performance Schema]
    M4 --> M08[M08 Buffer Pool 调优]

    M06 --> M5[模块 5: 现代特性]
    M5 --> M10[M10 JSON/窗口函数]
    M5 --> M11[M11 MySQL 9 新特性]
    M5 --> M12[M12 分库分表]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#ffe0b2
```

---

## 🔑 SQL 一句话经过的层

```mermaid
flowchart LR
    Client[客户端] --> Connector[连接器<br>auth/线程池]
    Connector --> QueryCache[查询缓存<br>8.0 已移除]
    QueryCache -.已废弃.-> Parser
    Connector --> Parser[解析器<br>词法/语法→AST]
    Parser --> Preproc[预处理<br>列名/表名解析]
    Preproc --> Optimizer[优化器<br>cost model + 直方图]
    Optimizer --> Executor[执行器<br>调用 handler API]
    Executor --> InnoDB[InnoDB 引擎<br>B+树/MVCC/锁/Buffer Pool]
    InnoDB --> Disk[(磁盘<br>ibd / redo / undo / binlog)]

    style QueryCache fill:#ffcdd2
    style Optimizer fill:#fff3e0
    style InnoDB fill:#bbdefb
```

---

## 🌲 InnoDB B+ 树结构（聚簇索引）

```mermaid
graph TD
    Root[根页<br>16KB] --> N1[内部节点]
    Root --> N2[内部节点]
    Root --> N3[内部节点]
    N1 --> L1[叶子页<br>主键+完整行]
    N1 --> L2[叶子页]
    N2 --> L3[叶子页]
    N2 --> L4[叶子页]
    N3 --> L5[叶子页]

    L1 -.双向链表.- L2
    L2 -.双向链表.- L3
    L3 -.双向链表.- L4
    L4 -.双向链表.- L5

    style Root fill:#fff3e0
    style L1 fill:#c8e6c9
    style L2 fill:#c8e6c9
    style L3 fill:#c8e6c9
    style L4 fill:#c8e6c9
    style L5 fill:#c8e6c9
```

二级索引叶子页存"索引列 + 主键"——查二级索引找到主键，**再用主键回表取完整行**（除非覆盖索引）。

---

## 🔒 锁等级

```mermaid
graph TD
    Lock{锁类型}
    Lock --> Table[表锁<br>MDL / 自增锁]
    Lock --> Row[行锁]
    Row --> Record[Record Lock<br>单行]
    Row --> Gap[Gap Lock<br>区间但不含端点]
    Row --> NextKey[Next-Key Lock<br>Record+左开右闭区间]
    Row --> Insert[Insert Intention<br>插入意图]

    NextKey --> RR[RR 隔离级别默认]
    Record --> RC[RC 隔离级别<br>仅 Record，无 Gap]

    style RR fill:#fff3e0
    style RC fill:#fff3e0
```

---

## 🔄 复制拓扑

```mermaid
flowchart LR
    M[Master<br>主库] -->|binlog dump| R1[Replica 1<br>异步]
    M -->|半同步 ACK| R2[Replica 2<br>半同步]
    M -->|MGR Paxos| R3[Group Member]
    M -->|MGR Paxos| R4[Group Member]

    Router[MySQL Router] --> M
    Router -.故障转移.-> R2

    style M fill:#fff3e0
    style R3 fill:#c8e6c9
    style R4 fill:#c8e6c9
```

---

## ⚡ EXPLAIN 速决树

```mermaid
flowchart TD
    Slow[SQL 慢]
    Slow --> Type{type 字段}
    Type -->|ALL| FullScan[全表扫描<br>→ 建索引]
    Type -->|index| FullIdx[全索引扫描<br>→ 看能否 range / 缩窄]
    Type -->|range| Range[范围扫描<br>→ 看 rows 估算是否过大]
    Type -->|ref / eq_ref| Ref[索引等值查找<br>多数情况 OK]
    Type -->|const / system| OK[常量级 ✓]

    Range --> Extra{Extra}
    Extra -->|Using filesort| Sort[需要排序<br>→ 排序列加索引]
    Extra -->|Using temporary| Tmp[需要临时表<br>→ GROUP BY/DISTINCT 列加索引]
    Extra -->|Using index| Cover[覆盖索引 ✓]
    Extra -->|Using where| Where[索引未完全过滤<br>→ ICP 检查]
```

---

## 🆕 MySQL 主要版本时间线

```mermaid
timeline
    title MySQL 关键里程碑
    2010 : MySQL 5.5 InnoDB 默认引擎
    2013 : MySQL 5.6 GTID / 在线 DDL
    2015 : MySQL 5.7 JSON 类型 / 性能大跃进
    2018 : MySQL 8.0 直方图 / CTE / 窗口函数
    2023 : MySQL 8.0 EOL announcement → 推 LTS 机制
    2024 April : MySQL 8.4 LTS（长期支持至 2032）
    2024 July : MySQL 9.0 创新版（vector type 引入）
    2025 : MySQL 9.x 持续迭代
    2026 : 主流生产基线 = 8.4 LTS；9.x 用于尝鲜
```

---

