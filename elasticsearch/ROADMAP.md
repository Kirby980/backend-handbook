# Elasticsearch 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：Elasticsearch 9.x + OpenSearch 3.x**（2026-05 时主流）

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 ES 之旅]) --> M1[模块 1: 内核]
    M1 --> E01[E01 倒排索引/Lucene]
    M1 --> E02[E02 Segment/合并]

    M1 --> M2[模块 2: 分布式]
    M2 --> E03[E03 集群拓扑]
    M2 --> E04[E04 分片/路由]

    M2 --> M3[模块 3: 建模与查询]
    M3 --> E05[E05 Mapping/Analyzer]
    M3 --> E06[E06 Query DSL]
    M3 --> E07[E07 BM25/Reranking]

    M3 --> M4[模块 4: 现代化]
    M4 --> E08[E08 向量检索]
    M4 --> E09[E09 写入/近实时]

    M4 --> M5[模块 5: 生产]
    M5 --> E10[E10 性能调优]
    M5 --> E11[E11 ES vs OpenSearch]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#ffe0b2
```

---

## 🔑 一次搜索的物理路径

```mermaid
sequenceDiagram
    participant Client
    participant Coord as Coordinating Node
    participant Shard1 as Primary Shard 0
    participant Shard2 as Replica Shard 1
    participant Shard3 as Replica Shard 2

    Client->>Coord: POST idx/_search {query}
    Coord->>Coord: 路由：选 N 个分片<br>(primary or replica)
    par 并行 query phase
        Coord->>Shard1: 本地查询 → 返回 top K doc_id+score
        Coord->>Shard2: 本地查询 → 返回 top K doc_id+score
        Coord->>Shard3: 本地查询 → 返回 top K doc_id+score
    end
    Coord->>Coord: 全局归并取 top K
    par 并行 fetch phase
        Coord->>Shard1: 取 doc_id 对应的 _source
        Coord->>Shard2: 取 doc_id 对应的 _source
    end
    Coord->>Client: 结果
```

**关键观察**：默认走 **query then fetch** 两阶段——query 阶段返回 doc_id + score 列表（轻量），fetch 阶段拿完整 source（重）。这是为什么"用 size:1000 翻深页"会慢到不行。

---

## 📦 Segment 与写入流水

```mermaid
flowchart TD
    Doc[新文档] --> InMem[In-memory buffer<br>+ translog]
    InMem -->|refresh 默认 1s| SegInMem[新 segment in-memory<br>可见但未持久化]
    SegInMem -->|refresh| FsCache[OS Page Cache 的 segment 文件]
    FsCache -->|flush 默认 translog 10GB（上限磁盘 1%）| Disk[(磁盘 segment + fsync translog)]

    SegInMem --> Merge[Tiered Merge Policy<br>定期合并小段]
    Merge --> BigSeg[Bigger segment]

    style InMem fill:#fff3e0
    style SegInMem fill:#fff9c4
    style FsCache fill:#bbdefb
    style Disk fill:#c8e6c9
```

- **refresh**：让新数据可被搜索（默认 1s，可调大降写入压力）
- **flush**：把 translog fsync + segment 落盘（保证 crash 安全）
- **merge**：把小 segment 合并成大 segment（节省文件句柄、降低查询时段扫数量）

---

## 🏗️ 集群拓扑（hot-warm-cold-frozen）

```mermaid
graph LR
    subgraph 控制面
    Master[Master eligible<br>3 节点 quorum]
    end

    subgraph 数据面
    Hot[Data hot<br>SSD<br>新数据/高写入]
    Warm[Data warm<br>SSD/HDD<br>近一周]
    Cold[Data cold<br>HDD<br>读少写无]
    Frozen[Data frozen<br>对象存储<br>可搜索快照]
    end

    subgraph 入口
    Coord[Coordinating only<br>路由+聚合]
    Ingest[Ingest pipeline<br>解析/enrich]
    end

    Coord --> Hot
    Coord --> Warm
    Coord --> Cold
    Coord --> Frozen

    Hot -.ILM rollover.-> Warm
    Warm -.ILM age.-> Cold
    Cold -.searchable snapshot.-> Frozen

    Master -.集群状态.-> Coord
    Master -.集群状态.-> Hot
    Master -.集群状态.-> Warm
```

---

## 🎯 分片大小决策

```mermaid
flowchart TD
    Q[新建索引]
    Q --> Vol{预期单分片数据量?}
    Vol -->|< 10GB| Small[1 分片 + N 副本即可]
    Vol -->|10-50GB| Medium[2-5 分片<br>留余地]
    Vol -->|> 50GB| Plan[要 rollover：<br>按天/周 切新索引<br>每分片 30-50GB]

    Plan --> ILM[ILM 策略<br>hot→warm→cold→delete]

    Vol2{查询并发?}
    Vol -->|每秒>100 QPS 单索引| Vol2
    Vol2 -->|高| MoreShard[副本 ≥ 2<br>分散读]
```

> 经验值：**单分片 30-50 GB 是甜区**。再大 → segment 多、merge 重、节点恢复慢；再小 → 元数据开销占比高，集群状态膨胀。

---

## 🆚 Elasticsearch vs OpenSearch 决策

```mermaid
flowchart TD
    Choose{需求}
    Choose -->|要新 ML/向量/Kibana 企业特性| ES[Elasticsearch 9<br>AGPLv3/ELv2/SSPL]
    Choose -->|要 Apache 2.0 / 完全 OSS| OS[OpenSearch 3<br>Apache 2.0 LF 治理]
    Choose -->|已在 AWS 跑| OS2[OpenSearch<br>原生托管]
    Choose -->|完全云中立 + 兼容旧栈| OS3[OpenSearch<br>避免许可证锁定]
    Choose -->|向量检索性能极限| ES2[ES 9<br>更新 Lucene 9.x/10.x]

    style ES fill:#fff3e0
    style OS fill:#c8e6c9
```

---

## 📈 ES 关键版本时间线

```mermaid
timeline
    title Elasticsearch 关键里程碑
    2010 : Elasticsearch 首个版本(0.x)
    2016 : 5.0 Lucene 6 (BKD)
    2019 : 7.0 默认单 type / Cluster Coordination
    2021 January : 7.11 改 SSPL/ELv2 → 引发 OpenSearch fork
    2021 April : OpenSearch fork by AWS (Apache 2.0)
    2022 : ES 8.0 安全默认 / kNN(ANN) 技术预览
    2022 August : ES 8.4 kNN(ANN) search GA
    2024 : ES 8.16 重新引入 AGPLv3 选项 / ES 8.x 持续完善 vector
    2024 June : ES 8.14 ES|QL GA
    2025 : ES 9.0 Lucene 10 / ES|QL Lookup Join 预览
    2025 May : Redis 8 重新开源 (前车之鉴)
    2026 : ES 9.x 与 OpenSearch 3.x 并存
```

---

