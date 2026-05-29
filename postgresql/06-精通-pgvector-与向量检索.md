# 精通 pgvector 与向量检索：HNSW、IVFFlat 与 RAG 实战

> 关联章节：[精通多类型索引](./05-精通-多类型索引.md)、[精通 JSONB 与全文检索](./12-精通-JSONB-与全文检索.md)、[精通扩展生态](./18-精通-扩展生态.md)

---

## 引言：PostgreSQL 怎么就成了 RAG 默认向量库

2022 年底 ChatGPT 火起来之前，没人会把 PG 和"向量库"联想到一起。当时大家讨论 RAG（Retrieval-Augmented Generation）的存储层默认是 Pinecone、Milvus、Weaviate、Qdrant 这些专门的 ANN 数据库。

但 2024-2025 风向变了。**pgvector** 这个不到 5000 行 C 代码的 PG 扩展，加上 PG 16+ 内置的 HNSW 索引支持，让一台普通的 PG 服务器就能处理千万级 embedding 向量的相似度检索，性能足够撑住中小型 RAG 应用。Supabase、Neon 这些云 PG 服务把 pgvector 标配化，AWS RDS / Aurora / GCP CloudSQL 也都把它列入白名单。

为什么这件事重要？因为 **"少一个组件"** 在工程上是巨大的胜利：

- 不用再维护一个独立的向量库
- 不用解决"PG 主数据和向量库的一致性"
- 一条 SQL 就能完成 "结构化过滤 + 向量检索 + 业务字段返回"
- 备份、监控、HA、RBAC 全部复用 PG 生态

这一章把 pgvector 从基础语法讲到生产化：HNSW vs IVFFlat 怎么选、维度怎么定、混合检索怎么做、和专门向量库相比什么时候必须升级。读完之后你应该能：

- 在 PG 18 上建出 HNSW 索引并解释 m / ef_construction / ef_search 三个参数
- 给一个 1000 万 embedding 表做容量与内存估算
- 写出"BM25 全文 + 向量相似 + RRF 融合"的混合检索 SQL
- 区分 pgvector / pgvectorscale / 专门向量库的取舍

---

## 第 1 章：向量检索 101

### 1.1 什么是 embedding

Embedding 是把"任何东西"（文本、图片、音频...）映射到一个固定维度的浮点向量。同一个 embedding 模型下，**语义相似的东西在向量空间中距离更近**。

```
"我想买一双跑鞋"   → [0.12, -0.34, 0.56, ..., 0.78]  (1536 维)
"推荐运动鞋"       → [0.15, -0.31, 0.59, ..., 0.74]
"今天天气怎么样"   → [-0.45, 0.62, -0.18, ..., 0.03]
```

前两个向量的余弦相似度可能 0.92，和第三个相似度只有 0.05。

### 1.2 距离度量

| 度量 | 操作符 | 数学意义 | 适用 |
|---|---|---|---|
| **L2 距离**（欧氏距离） | `<->` | sqrt(Σ(a_i - b_i)²) | 几何意义，但向量未归一化时数值漂移大 |
| **内积**（dot product） | `<#>` | -Σ(a_i × b_i)（取负是为了 ASC 排序） | 推荐系统、未归一化向量 |
| **余弦相似度** | `<=>` | 1 - cos(θ) = 1 - (a·b)/(\|a\|\|b\|) | **最常用**，归一化向量首选 |
| **L1 距离**（曼哈顿） | `<+>` | Σ\|a_i - b_i\| | 较少用 |

> 💡 OpenAI / Anthropic / 大多数主流 embedding 模型输出的向量都是**归一化**的（模长=1）。此时余弦相似度和内积等价（差一个符号），但用余弦更直观。

### 1.3 ANN（Approximate Nearest Neighbor）的意义

精确最近邻搜索（brute force）= 计算查询向量和**每个向量**的距离，O(N) 时间。1000 万向量 × 1536 维 × 一次浮点运算 = 单查询几百毫秒。

ANN（近似最近邻）通过索引把搜索复杂度降到接近 O(log N)，代价是**召回率（recall）不是 100%**——可能漏掉真实的 top-K 中的几个。生产环境通常追求 95%+ 召回率，对应的延迟 < 50ms。

---

## 第 2 章：pgvector 入门

### 2.1 安装

```bash
# Docker（最简单）
docker run -d \
  --name pg18-vec \
  -e POSTGRES_PASSWORD=pwd \
  -p 5432:5432 \
  pgvector/pgvector:pg18

# 源码安装
cd /tmp && git clone --branch v0.7.4 https://github.com/pgvector/pgvector.git
cd pgvector && make && sudo make install
```

```sql
CREATE EXTENSION vector;
SELECT extversion FROM pg_extension WHERE extname = 'vector';
-- 期望：0.7.4 或更新
```

### 2.2 数据类型

pgvector 0.7+ 提供 4 种向量类型：

| 类型 | 元素 | 用途 | 单元素大小 |
|---|---|---|---|
| `vector(d)` | float4 | 标准 embedding | 4 byte |
| `halfvec(d)` | float2 | 半精度，节省一半空间 | 2 byte |
| `bit(d)` | bit | 二进制量化，节省 32 倍 | 1 bit (打包) |
| `sparsevec(d, n)` | (idx, val) 对 | 稀疏向量（大多数维度为 0） | 变长 |

`d` 是维度。`vector` 最大维度 16000，HNSW 索引最大 2000，IVFFlat 索引最大 2000。

### 2.3 基础操作

```sql
-- 建表
CREATE TABLE documents (
    id        bigserial PRIMARY KEY,
    title     text,
    content   text,
    embedding vector(1536),  -- OpenAI text-embedding-3-small
    metadata  jsonb,
    created_at timestamptz DEFAULT now()
);

-- 插入
INSERT INTO documents (title, content, embedding) VALUES
  ('Go 1.26 发布说明', '...', '[0.12, -0.34, ...]'::vector);

-- 精确查询（无索引，brute force）
SELECT id, title, embedding <=> '[0.1, -0.3, ...]'::vector AS distance
FROM documents
ORDER BY embedding <=> '[0.1, -0.3, ...]'::vector
LIMIT 10;
```

`<=>` 是余弦距离操作符。ORDER BY 后必须用**同一个操作符**才能走 ANN 索引——这是 pgvector 的核心约定。

### 2.4 距离操作符使用示例

```sql
-- L2 距离
SELECT * FROM documents ORDER BY embedding <-> q LIMIT 10;

-- 内积（注意 <#> 返回负内积，所以 ORDER BY ASC = 找最相似）
SELECT * FROM documents ORDER BY embedding <#> q LIMIT 10;

-- 余弦距离（推荐）
SELECT * FROM documents ORDER BY embedding <=> q LIMIT 10;

-- 取相似度（1 - 距离）
SELECT id, 1 - (embedding <=> q) AS similarity
FROM documents
ORDER BY embedding <=> q
LIMIT 10;
```

---

## 第 3 章：IVFFlat 索引（倒排聚类）

### 3.1 IVFFlat 工作原理

IVFFlat = Inverted File with Flat compression。**先对所有向量做 k-means 聚类成 `lists` 个簇，查询时只扫离查询点最近的 `probes` 个簇。**

```
所有向量 1000 万
   ↓ k-means 聚成 1000 个簇
[Cluster 1] [Cluster 2] ... [Cluster 1000]
   ↑               ↑
查询时挑最近 probes=10 个簇
扫这 10 个簇里的所有向量（约 10 万）
```

### 3.2 创建 IVFFlat 索引

```sql
-- 必须先有数据才能建（k-means 要训练）
INSERT INTO documents (embedding) SELECT ...  -- 至少 1000+ 行
ANALYZE documents;

CREATE INDEX docs_emb_ivfflat ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 1000);  -- 推荐值 sqrt(rows) 或 rows/1000
```

`vector_cosine_ops` 对应 `<=>`；`vector_l2_ops` 对应 `<->`；`vector_ip_ops` 对应 `<#>`。

### 3.3 IVFFlat 关键参数

| 参数 | 默认 | 调优 | 影响 |
|---|---|---|---|
| `lists` | — | sqrt(N) 或 N/1000，N 为行数 | 簇数。多 → 每簇行少但簇间距小；少 → 反之 |
| `probes` | 1 | 1-N，N 越大召回越高 | 查询时扫的簇数。决定召回率与延迟 |

```sql
-- 查询时调 probes
SET ivfflat.probes = 10;
SELECT ... ORDER BY embedding <=> q LIMIT 10;
```

经验：`probes = sqrt(lists)` 是常见起点；要求 95% 召回的话 `probes = lists/100 ~ lists/50`。

### 3.4 IVFFlat 的局限

- **必须先 INSERT 再 CREATE INDEX**——k-means 训练数据要够
- 新插入的向量**不会重新平衡**到合适的簇——簇分布会逐渐失衡，召回率下降
- 周期性 `REINDEX` 重建索引才能恢复
- 召回率随数据漂移降低

这些缺点让 IVFFlat 在 2023 年后逐渐让位给 HNSW。

---

## 第 4 章：HNSW 索引（分层图，PG 16+ 主推）

### 4.1 HNSW 的核心思想

HNSW = Hierarchical Navigable Small World。借用 Skip List 的多层结构 + Small World Graph 的"短路径"特性。

```
Layer 2 (稀疏顶层)     A ─────────────── D
                       │                 │
Layer 1 (中间层)       A ─── B ─────── D ─── E
                       │     │         │     │
Layer 0 (全部节点)     A ─ B ─ C ─ D ─ E ─ F ─ G ─ ...
                       (每个点和它的 M 个邻居相连)
```

查询时从顶层开始走"近邻链"，逐层下降，每层都贪心走向最近邻——这让搜索路径长度只有 O(log N)。

### 4.2 创建 HNSW 索引

```sql
CREATE INDEX docs_emb_hnsw ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (
    m = 16,                  -- 每节点连接数
    ef_construction = 64     -- 构建时候选数
);
```

可以**先建索引再插入数据**——这是 HNSW vs IVFFlat 的一大区别。

### 4.3 HNSW 三大参数

| 参数 | 默认 | 含义 | 调大 → |
|---|---|---|---|
| `m` | 16 | 每节点的邻居连接数 | 召回率高、内存大、构建慢 |
| `ef_construction` | 64 | 构建时维护的候选列表大小 | 召回率高、构建慢、查询时间不变 |
| `ef_search`（查询时） | 40 | 查询时维护的候选列表大小 | 召回率高、查询慢 |

```sql
-- 查询时调 ef_search
SET hnsw.ef_search = 100;
SELECT ... ORDER BY embedding <=> q LIMIT 10;
```

**典型生产配置**：
- 高精度场景：`m=32, ef_construction=200, ef_search=100`
- 平衡（默认）：`m=16, ef_construction=64, ef_search=40`
- 极致速度：`m=8, ef_construction=40, ef_search=20`

### 4.4 HNSW 内存估算

HNSW 索引几乎全部内存常驻（不依赖磁盘随机 IO）。粗略公式：

```
内存 ≈ rows × (dim × 4 byte + m × 2 × 8 byte) × layer_factor
```

其中 `layer_factor` 约 1.1-1.2。例：

- 100 万行 × 1536 维 × 4 = 6 GB（向量本身）
- 加上图结构 m=16: 额外 ~250 MB
- **共约 6.3 GB**

对于一台 64 GB 服务器，能支撑约 1000 万 1536 维向量的 HNSW 索引——这是单实例 pgvector 的甜蜜区。

### 4.5 HNSW vs IVFFlat 决策表

| 维度 | IVFFlat | HNSW |
|---|---|---|
| 索引构建速度 | 快（k-means 训练） | 慢（多次贪心搜索） |
| 查询延迟（同召回率） | 较慢 | 快 |
| 召回率上限 | 95% 左右 | 99%+ 可达 |
| 内存占用 | 小 | 大（几乎全内存） |
| 增量更新 | 不平衡，需定期重建 | 平衡，无需重建 |
| 建索引前数据要求 | 必须有训练数据 | 无 |
| 大数据集（>1 亿） | 内存压力小，但延迟差 | 内存吃紧 |
| **推荐场景** | 数据基本静态 / 内存受限 | **大多数 RAG 场景** |

> 2026 主流：**新项目默认 HNSW**。IVFFlat 留给极端内存受限或数据完全静态的场景。

---

## 第 5 章：量化与压缩

### 5.1 半精度（halfvec）

把 float4（32 位）换成 float2（16 位），空间减半，召回率损失极小（多数模型 < 1%）。

```sql
-- 用 halfvec 重建
ALTER TABLE documents ADD COLUMN embedding_half halfvec(1536);
UPDATE documents SET embedding_half = embedding::halfvec(1536);

CREATE INDEX docs_emb_half_hnsw ON documents
USING hnsw (embedding_half halfvec_cosine_ops);
```

1000 万 × 1536 维：vector 6 GB → halfvec 3 GB。生产价值显著。

### 5.2 二进制量化（bit）

更激进：把浮点向量量化成 0/1 bit，空间缩小 32 倍。适合做**第一阶段粗筛**，再用原始向量精排。

```sql
ALTER TABLE documents ADD COLUMN embedding_bit bit(1536);
UPDATE documents SET embedding_bit = ...;  -- 自定义量化函数

-- 用 Hamming 距离查询
CREATE INDEX ON documents
USING hnsw (embedding_bit bit_hamming_ops);

-- 两阶段查询：先 bit 粗筛 100 个，再原向量精排
SELECT * FROM (
    SELECT * FROM documents
    ORDER BY embedding_bit <~> query_bit  -- hamming
    LIMIT 100
) sub
ORDER BY embedding <=> query_vec
LIMIT 10;
```

### 5.3 稀疏向量（sparsevec）

如果 embedding 大部分维度为 0（如 BM25 sparse encoding 或 SPLADE），用 sparsevec 节省巨大：

```sql
-- sparsevec(d, n)：d 维度，n 实际非零数
CREATE TABLE docs_sparse (
    id bigserial PRIMARY KEY,
    sp sparsevec(30000, 50)  -- 3 万维但只 50 个非零
);
```

---

## 第 6 章：pgvectorscale（Timescale 出品）

### 6.1 pgvectorscale 是什么

Timescale 团队 2023 年推出的 pgvector 增强扩展。核心贡献是 **StreamingDiskANN** 索引——基于微软的 DiskANN 算法，**索引可以部分驻留磁盘**而不是全部内存。

```sql
CREATE EXTENSION vectorscale;

CREATE INDEX docs_emb_diskann ON documents
USING diskann (embedding vector_cosine_ops)
WITH (
    storage_layout = 'memory_optimized',  -- 或 plain
    num_neighbors = 50,
    search_list_size = 100
);
```

### 6.2 何时选 pgvectorscale

- **数据集巨大**（千万+ 到亿级），HNSW 内存放不下
- 接受**轻微的查询延迟提升**（10-30ms vs HNSW 的 3-10ms）
- 用 Timescale 生态

### 6.3 vs HNSW 简表

| 维度 | pgvector HNSW | pgvectorscale DiskANN |
|---|---|---|
| 内存需求 | 高（全内存） | 低（部分磁盘） |
| 查询延迟 | 极低 | 略高 |
| 适合规模 | < 1000 万 | 1 亿级 |
| 成熟度 | 高 | 较新但快速发展 |
| 云支持 | 普及 | 部分云（Timescale Cloud / 自建） |

---

## 第 7 章：维度选择指南

不同 embedding 模型的输出维度差异很大。选维度本质是在**召回质量 / 存储成本 / 推理成本**三者间权衡。

| 模型 | 维度 | 单条空间 (vector) | 1 千万条索引 | 备注 |
|---|---|---|---|---|
| `text-embedding-3-small` (OpenAI) | 1536（可裁剪到 512） | 6 KB | 60 GB | 性价比标配 |
| `text-embedding-3-large` (OpenAI) | 3072（可裁剪到 256） | 12 KB | 120 GB | 高质量场景 |
| `text-embedding-ada-002` (OpenAI 老模型) | 1536 | 6 KB | 60 GB | 已被 3-small 替代 |
| `bge-base-zh-v1.5` (BAAI) | 768 | 3 KB | 30 GB | 中文性价比 |
| `bge-large-zh-v1.5` | 1024 | 4 KB | 40 GB | 中文高质量 |
| `all-MiniLM-L6-v2` (sentence-transformers) | 384 | 1.5 KB | 15 GB | 轻量多语言 |
| `voyage-3-large` (Voyage) | 1024 | 4 KB | 40 GB | 评测领先 |
| `voyage-3.5` (Voyage) | 1024 | 4 KB | 40 GB | Voyage AI（Anthropic 推荐，与 Claude 配套） |

**Matryoshka Embedding（俄罗斯套娃）**：OpenAI v3 系列支持降维使用——把 1536 维直接截前 512 维仍然能保留 90%+ 的语义信息。生产中常见做法是：

- 主索引存 512 维（节省 67% 空间）
- 精排时用完整 1536 维（实时 API 取或本地缓存）

```sql
-- 截维度
SELECT
    embedding[1:512]::vector(512) AS emb_small,
    embedding AS emb_full
FROM documents;
```

---

## 第 8 章：RAG 实战 — 混合检索

### 8.1 为什么要混合检索

纯向量检索的盲点：

- **关键词专有名词**："iPhone 17 Pro" 这种型号词向量模型经常 hash 成相近向量
- **精确数字 / 代码**：向量召回不可靠
- **新词**：embedding 训练时间截止后的术语

纯关键词（BM25）的盲点：

- **语义近义**："跑鞋" vs "运动鞋" vs "训练鞋"
- **跨语言**

混合检索 = **关键词召回 + 向量召回，结果用 RRF 融合**。

### 8.2 RRF（Reciprocal Rank Fusion）

```
RRF_score(d) = Σ 1 / (k + rank_i(d))
```

`k=60` 是经验常数。每个检索器各自给一个 rank，最终分数是 rank 倒数和。简单、参数少、效果稳健。

### 8.3 完整 SQL

```sql
WITH
-- 关键词召回 top-50
keyword AS (
    SELECT id,
           ts_rank(to_tsvector('simple', title || ' ' || content),
                   websearch_to_tsquery('simple', 'iPhone 17 Pro 评测')) AS score,
           row_number() OVER (
               ORDER BY ts_rank(
                   to_tsvector('simple', title || ' ' || content),
                   websearch_to_tsquery('simple', 'iPhone 17 Pro 评测')
               ) DESC
           ) AS rank
    FROM documents
    WHERE to_tsvector('simple', title || ' ' || content)
          @@ websearch_to_tsquery('simple', 'iPhone 17 Pro 评测')
    ORDER BY score DESC LIMIT 50
),
-- 向量召回 top-50
vec AS (
    SELECT id,
           1 - (embedding <=> $1::vector) AS score,
           row_number() OVER (ORDER BY embedding <=> $1::vector) AS rank
    FROM documents
    ORDER BY embedding <=> $1::vector LIMIT 50
),
-- RRF 融合
fused AS (
    SELECT
        COALESCE(k.id, v.id) AS id,
        COALESCE(1.0/(60 + k.rank), 0) + COALESCE(1.0/(60 + v.rank), 0) AS rrf_score
    FROM keyword k
    FULL OUTER JOIN vec v ON k.id = v.id
)
SELECT d.id, d.title, d.content, f.rrf_score
FROM fused f
JOIN documents d ON d.id = f.id
ORDER BY f.rrf_score DESC
LIMIT 10;
```

参数 `$1` 是查询的 embedding（在应用层调 embedding API 算出来传入）。

### 8.4 Go pgx 客户端示例

```go
package main

import (
    "context"
    "fmt"

    "github.com/jackc/pgx/v5/pgxpool"
    "github.com/pgvector/pgvector-go"
)

type Doc struct {
    ID      int64
    Title   string
    Content string
    Score   float64
}

func hybridSearch(ctx context.Context, pool *pgxpool.Pool,
    query string, qVec []float32) ([]Doc, error) {

    rows, err := pool.Query(ctx, `
WITH keyword AS (
  SELECT id, row_number() OVER (
      ORDER BY ts_rank(
          to_tsvector('simple', title || ' ' || content),
          websearch_to_tsquery('simple', $1)
      ) DESC) AS rank
  FROM documents
  WHERE to_tsvector('simple', title || ' ' || content)
        @@ websearch_to_tsquery('simple', $1)
  ORDER BY ts_rank(
      to_tsvector('simple', title || ' ' || content),
      websearch_to_tsquery('simple', $1)
  ) DESC LIMIT 50
),
vec AS (
  SELECT id, row_number() OVER (ORDER BY embedding <=> $2) AS rank
  FROM documents
  ORDER BY embedding <=> $2 LIMIT 50
)
SELECT d.id, d.title, d.content,
       COALESCE(1.0/(60+k.rank), 0) + COALESCE(1.0/(60+v.rank), 0) AS score
FROM documents d
LEFT JOIN keyword k ON k.id = d.id
LEFT JOIN vec     v ON v.id = d.id
WHERE k.rank IS NOT NULL OR v.rank IS NOT NULL
ORDER BY score DESC
LIMIT 10
`, query, pgvector.NewVector(qVec))
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var docs []Doc
    for rows.Next() {
        var d Doc
        if err := rows.Scan(&d.ID, &d.Title, &d.Content, &d.Score); err != nil {
            return nil, err
        }
        docs = append(docs, d)
    }
    return docs, nil
}

func main() {
    pool, _ := pgxpool.New(context.Background(), "postgres://...")
    defer pool.Close()

    // qVec 由 embedding API 算出
    qVec := []float32{0.12, -0.34, /* ... 1536 维 ... */}
    docs, _ := hybridSearch(context.Background(), pool, "iPhone 17 评测", qVec)
    for _, d := range docs {
        fmt.Printf("[%.4f] %s\n", d.Score, d.Title)
    }
}
```

依赖：
```bash
go get github.com/jackc/pgx/v5
go get github.com/pgvector/pgvector-go
```

### 8.5 元数据过滤 + 向量召回

实际 RAG 经常要"先按结构化字段过滤，再向量检索"。pgvector 的索引可以和 WHERE 子句组合：

```sql
SELECT id, title FROM documents
WHERE category = 'tech'
  AND created_at > now() - interval '30 days'
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

不过有个**坑**：如果 `WHERE` 过滤掉的行很多，HNSW 走索引返回 10 行后再过滤可能不够 10 行。解决方案：

```sql
SET hnsw.iterative_scan = strict_order;  -- pgvector 0.8+
-- 或 ef_search 调大
SET hnsw.ef_search = 100;
```

或者用 **预过滤分区**：把热门类别物理分区，每个分区独立 HNSW 索引。

---

## 第 9 章：与专门向量库对比

### 9.1 对比矩阵

| 维度 | pgvector | Elasticsearch dense_vector | Milvus | Qdrant | Pinecone |
|---|---|---|---|---|---|
| 部署形态 | PG 内 | ES 内 | 独立 | 独立 | 托管 |
| 规模上限 | 单机千万 / pgvectorscale 亿级 | 亿级 | 十亿级 | 亿级 | 十亿级 |
| 索引算法 | HNSW / IVFFlat | HNSW | HNSW / IVF / DiskANN / GPU | HNSW | 闭源 |
| 元数据过滤 | SQL 全功能 | 强 | 较弱 | 较强 | 较强 |
| 事务 | ✅ ACID | ✅（部分） | ❌ | ❌ | ❌ |
| 混合检索（向量+全文） | ✅（同库） | ✅（同库） | ❌ | 较新支持 | ❌ |
| 学习成本 | 极低（会 SQL 就行） | 中 | 高 | 中 | 低 |
| 运维成本 | **极低**（复用 PG） | 中 | 高 | 中 | 极低（托管） |
| 成本 | 复用 PG 节点 | 独立集群 | 独立集群 | 独立集群 | 按 vector 数量计费 |

### 9.2 什么时候必须升级到专门向量库

- 向量数 > 5000 万且要 < 10ms 延迟
- 需要 GPU 加速（Milvus）
- 多租户向量库平台（Pinecone 等托管服务）
- 需要复杂的向量 + 标量混合索引

**绝大多数 RAG 应用永远不需要离开 pgvector**——这是 2026 年的事实。

---

## 第 10 章：pgvector 0.7+ 新特性

### 10.1 改进的 HNSW 构建并行

PG 16+ 配合 pgvector 0.6 起支持并行构建 HNSW，大幅缩短构建时间：

```sql
SET max_parallel_maintenance_workers = 4;
CREATE INDEX ... USING hnsw ...;  -- 自动并行
```

### 10.2 二进制量化与稀疏向量

如前所述，`halfvec` / `bit` / `sparsevec` 都是 0.7 系列加入的。

### 10.3 Iterative scan（0.8+）

解决 "过滤太多导致 HNSW 返回不够 10 行" 的问题：

```sql
SET hnsw.iterative_scan = relaxed_order;
-- 或 strict_order
```

### 10.4 SubVector（subvector index）

把向量切片后建多个小索引，加速极高维（>2000 维）场景。

---

## 第 11 章：性能调优 checklist

### 11.1 索引参数

- HNSW `m`：16 起步，要求高召回升 32-48
- HNSW `ef_construction`：64 起步，质量优先升 128-200
- HNSW `ef_search`（运行时）：召回率不够时升到 100-200
- IVFFlat `lists`：rows/1000 起步
- IVFFlat `probes`：sqrt(lists) 起步

### 11.2 内存

- `shared_buffers`：足够装下索引（向量数据 + 图结构）
- `maintenance_work_mem`：建索引时调到 8-16 GB
- `work_mem`：每会话 64-256 MB，避免向量排序溢出磁盘

### 11.3 并行

```sql
SET max_parallel_workers_per_gather = 4;
SET max_parallel_maintenance_workers = 4;
```

### 11.4 监控

```sql
-- 看 HNSW 索引大小
SELECT pg_size_pretty(pg_relation_size('docs_emb_hnsw'));

-- 看查询走没走索引
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM documents
ORDER BY embedding <=> '[...]'::vector LIMIT 10;
-- 期望看到 Index Scan using docs_emb_hnsw

-- 测召回率（vs 暴力扫的 ground truth）
WITH gt AS (  -- ground truth
    SELECT id FROM documents
    ORDER BY embedding <=> '[...]'::vector
    LIMIT 10
), hnsw AS (
    SELECT id FROM documents
    WHERE id IN (SELECT id FROM (
        SELECT id FROM documents
        ORDER BY embedding <=> '[...]'::vector
        LIMIT 10
    ) sub)
)
SELECT count(*)::float / 10 AS recall FROM hnsw;
```

---

## 生产实践

1. **新项目默认 HNSW**：除非数据极静态或内存极受限。
2. **维度优先 1536，预算紧张降到 512（matryoshka）**：90%+ 召回 + 67% 空间节省。
3. **embedding 列单独存表**：主表存业务字段，副表只存 id + embedding，避免主表行宽爆炸影响 OLTP。
4. **HNSW 索引提前建好再灌数据**：HNSW 支持增量构建，IVFFlat 不行。
5. **混合检索是 RAG 默认**：BM25 + 向量 + RRF，单纯向量召回的 RAG 质量普遍偏低。
6. **元数据过滤场景预热 ef_search**：HNSW + 高过滤率会"少返回"，要么调 ef_search 要么改用分区。
7. **maintenance_work_mem 调大再建索引**：默认 64MB 建 1000 万向量 HNSW 要小时级，调到 8GB 能压到 10-20 分钟。
8. **生产监控召回率**：定期抽样对比 brute force 和 HNSW 的 top-K，召回率下降到 90% 以下要警觉。
9. **使用 pgvector-go 等官方客户端**：手动序列化向量容易出错。
10. **pgvectorscale 用于亿级**：千万级 HNSW 已足够；亿级才考虑切换或升级硬件。

---

## 陷阱清单

1. **distance 操作符不匹配** → CREATE INDEX 用 `vector_cosine_ops`，查询用 `<->`，索引不会被使用。必须 cosine ↔ `<=>`、L2 ↔ `<->`、ip ↔ `<#>` 一一对应。
2. **ORDER BY 表达式与索引不一致** → `ORDER BY embedding <=> q + 0.001` 这种修改后的表达式走不了索引。
3. **WHERE 过滤后 LIMIT 不够 10 行** → 解决用 `hnsw.ef_search` 调大或 `iterative_scan`。
4. **embedding 未归一化但用 cosine** → 归一化后用 `<=>` 或 `<#>` 都行；未归一化时只有 `<->` 数值有几何意义。
5. **ANN 召回率假设 100%** → ANN 是近似的。监控真实召回率，不要盲信。
6. **maintenance_work_mem 太小** → 建索引慢得离谱。
7. **过早升级到专门向量库** → 大多数应用 < 1000 万向量，pgvector 完全够。先用 pgvector 跑半年再评估。
8. **HNSW 与高并发写** → HNSW 写性能比 B-tree 慢。每秒写入 > 1000 条的场景要测试，必要时批量写。
9. **重启后 shared_buffers 冷启动** → HNSW 查询第一次很慢（10+ 秒），预热脚本必备。
10. **向量列存 text 而不是 vector** → 字符串解析每次几十毫秒。务必用 `vector(d)` 类型。
11. **忘了 pg_dump/pg_restore 时 ext 兼容** → 备份恢复时确保目标库也安装了相同版本 pgvector。
12. **HNSW 建在 nullable 列** → NULL 不会建进索引，但行还在表里，部分查询路径退化为 Seq Scan。
13. **不做 ANALYZE** → planner 估算行数不准，可能不选 HNSW 索引。
14. **混淆 cosine 距离与相似度** → `<=>` 返回距离 [0, 2]，相似度 = 1 - 距离 [-1, 1]。
15. **极高维度（> 2000）用 HNSW** → 性能急剧下降。考虑降维或 SubVector。

---

## 2026 现状

| 主题 | 状态 |
|---|---|
| **pgvector HNSW** | 事实标准，几乎所有 PG 云服务原生支持 |
| **pgvector 0.7+** | halfvec / bit / sparsevec 量化与压缩成熟 |
| **pgvectorscale (DiskANN)** | 亿级数据集崛起；Timescale 重点推 |
| **Matryoshka embedding** | OpenAI v3 系列引领；混合维度索引常见 |
| **混合检索（BM25 + 向量）** | RAG 默认；RRF / 加权融合都常见 |
| **PG 18 异步 IO** | 对 HNSW 部分查询路径有性能改善 |
| **Anthropic / Voyage embeddings 集成** | Anthropic Files API + Voyage AI 嵌入（Anthropic 推荐第三方，无自研模型） |
| **多模态向量** | 图像 / 视频 embedding 入库与文本同库混存 |
| **CloudNativePG + pgvector** | K8s 上的标准化部署模式 |
| **AWS Aurora pgvector 优化** | Aurora 专门做了 HNSW 优化版本 |
| **Supabase Vector** | 把 pgvector 包装成产品；社区影响力大 |

---

## 练习题

1. 你的 1500 万 1536 维 embedding 表，需要 < 30ms p99 latency。HNSW 还是 IVFFlat？关键参数怎么选？

   **答案**：HNSW。`m=16, ef_construction=64` 起步，`ef_search=80` 调到 95%+ 召回率。内存预估 ~10GB，需要 32GB+ 服务器。

2. 同样的表，HNSW 索引建完用了 4 小时。怎么提速？

   **答案**：调大 `maintenance_work_mem`（8-16 GB），开 `max_parallel_maintenance_workers = 4-8`，pgvector 0.6+ 支持并行构建。

3. 业务要求"只在用户能看到的部门内做向量检索"，但 HNSW 总是返回不足 10 行。怎么办？

   **答案**：(1) 调大 `hnsw.ef_search` 到 100-200；(2) pgvector 0.8+ 开 `hnsw.iterative_scan = relaxed_order`；(3) 按部门预分区，每个分区独立 HNSW。

4. 用 OpenAI text-embedding-3-large（3072 维），1000 万条数据。预算只够 32GB 内存。怎么塞下？

   **答案**：(1) Matryoshka 降维到 768，召回率还能保 90%+；(2) 切到 halfvec，空间减半；(3) 切 pgvectorscale DiskANN，索引允许部分上磁盘。

5. 同库做 BM25 + 向量混合检索，给出完整 SQL 骨架，并解释 RRF 中 k=60 的作用。

   **答案**：参考第 8 章 RRF SQL。k=60 是论文经验值，平滑 rank 倒数曲线避免 top-1 权重过大；也避免 rank=0 导致除零。

6. HNSW 索引重启后第一次查询慢 10 秒，平时 5ms。原因？

   **答案**：HNSW 索引几乎全内存，重启后 shared_buffers 冷启动，第一次查询要把图节点页面从磁盘读入。解决：启动后跑预热脚本（SELECT 几个典型 query），或用 `pg_prewarm` 扩展预加载。

7. 你想测当前 HNSW 索引的实际召回率，怎么做？

   **答案**：抽样 100-1000 个测试 query；对每个 query 分别用 `SET LOCAL enable_indexscan = off` 跑 brute force 得 ground truth top-K，再用索引跑 top-K，计算交集比例。详见第 11.4 节。

8. 一个 RAG 应用上线半年，发现召回质量下降。怎么排查？

   **答案**：(1) 看是否新增数据未 ANALYZE；(2) 检查 IVFFlat 是否簇分布失衡（用 IVFFlat 要 REINDEX）；(3) 检查 embedding 模型是否升级，新老向量混存导致空间不一致——这是最常见的；(4) 监控 hint bits 与 dead tuple 比例，必要时 VACUUM。

---

## 延伸阅读

- pgvector GitHub：[github.com/pgvector/pgvector](https://github.com/pgvector/pgvector)
- pgvectorscale GitHub：[github.com/timescale/pgvectorscale](https://github.com/timescale/pgvectorscale)
- HNSW 论文：Malkov & Yashunin, *Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs*（2016）
- DiskANN 论文：Subramanya et al., *DiskANN: Fast Accurate Billion-point Nearest Neighbor Search on a Single Node*（NeurIPS 2019）
- Matryoshka Embeddings：[matryoshka.dev](https://matryoshka.dev/)
- OpenAI embedding 文档：[platform.openai.com/docs/guides/embeddings](https://platform.openai.com/docs/guides/embeddings)
- 关联章节 — [P05 多类型索引](./05-精通-多类型索引.md)、[P12 JSONB 与全文检索](./12-精通-JSONB-与全文检索.md)、[P18 扩展生态](./18-精通-扩展生态.md)、[ai-backend/A06 Embedding 与 RAG](../ai-backend/A06-精通-Embedding-与-RAG.md)
