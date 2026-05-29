# 精通倒排索引与 Lucene：Elasticsearch 一切性能的物理根因

> 关联章节：[E02 Segment 与合并](./02-精通-Segment-与合并.md)、[E06 Query DSL](./06-精通-Query-DSL.md)、[E07 BM25](./07-精通-BM25-与-Reranking.md)、[E08 向量检索](./08-精通-向量检索.md)

---

## 引言：Elasticsearch 不是数据库

很多人把 Elasticsearch 当成"会全文检索的 NoSQL"。这个比喻误导了 90% 的性能问题。

**ES 是 Lucene 的分布式包装，而 Lucene 是为搜索而生的倒排索引引擎**。它和 MySQL / MongoDB 的底层逻辑根本不同：

- 数据库优化"按主键找一行"——B+ 树 / LSM 树
- 倒排索引优化"按词找所有相关行"——term → posting list

你往 ES 写一个文档，它**不仅仅是把这个文档存起来**——它要：
1. 把文档拆词（analyzer）
2. 对每个词更新一个全局的"出现在哪些文档"列表（posting list）
3. 还要存"这个文档总共有多少词、最长字段多长"用于评分
4. 还要存原始 `_source` 供返回

写入比数据库重很多。换来的是：**给我一个词，亚毫秒级返回所有相关文档**——这是 B+ 树做不到的。

这一章把 Lucene 的核心数据结构拆开看：FST 词典、posting list、doc values、stored fields。读完之后你应该能解释：

- 为什么 `keyword` 字段比 `text` 字段聚合快得多
- 为什么 `_source` 是甜蜜的负担
- 为什么 segment 不变性是 ES 一切设计的根因
- 为什么"我索引了 1 GB 文本，磁盘上 segment 是 3 GB"

---

## 第一章：倒排索引的基本模型

### 1.1 正排 vs 倒排

```
正排（传统数据库）：
  doc_id → [字段 1, 字段 2, 字段 3, ...]

倒排（搜索引擎）：
  term → [doc_id1, doc_id2, doc_id3, ...]
```

要回答"哪些文档包含 'elasticsearch'"：

- 正排：扫所有文档，每个去看是否含该词 → O(N) 全表扫描
- 倒排：直接查 `'elasticsearch' → [3, 17, 42, ...]` → O(log N) 词典查找 + O(K) 命中读取

ES 同时存正排（`_source` / doc_values）和倒排（postings）—— 倒排用于"找文档"，正排用于"看文档详情 + 排序 + 聚合"。

### 1.2 一份文档进入 Lucene 后被拆成什么

```
原始文档：
{
  "title": "Mastering Elasticsearch 9",
  "tags": ["search", "lucene", "es9"],
  "views": 1234,
  "ts": "2026-05-13T10:00:00Z",
  "vec": [0.12, 0.43, ...]
}

存储后（Lucene 内部）：
┌─────────────────────────────────────────────────────────┐
│ Inverted Index（倒排）                                   │
│   title:                                                │
│     "mastering" → [doc_42 @pos=0]                       │
│     "elasticsearch" → [doc_42 @pos=1, doc_17 @pos=...]  │
│     "9"          → [doc_42 @pos=2]                      │
│   tags:                                                 │
│     "search"  → [doc_42, doc_99, ...]                   │
│     "lucene"  → [doc_42, ...]                           │
│     "es9"     → [doc_42, ...]                           │
├─────────────────────────────────────────────────────────┤
│ Doc Values（列存正排）                                   │
│   views:    [doc_42=1234, doc_43=567, ...]              │
│   ts:       [doc_42=epoch_ms, ...]                      │
├─────────────────────────────────────────────────────────┤
│ Stored Fields（原始 _source）                            │
│   doc_42: { title:"...", tags:[...], views:1234, ... }  │
├─────────────────────────────────────────────────────────┤
│ Term Vectors（可选，per-doc 词频，跑 MLT 时用）           │
├─────────────────────────────────────────────────────────┤
│ Norms（每文档每字段的长度归一化值）                       │
│   title norm[doc_42] = 0.625                            │
├─────────────────────────────────────────────────────────┤
│ Vector Index（dense_vector 字段，HNSW 图）               │
│   doc_42.vec → 在 HNSW 图上一个节点                      │
└─────────────────────────────────────────────────────────┘
```

一个文档进 Lucene → 拆出 **5-7 套数据结构** 同时维护。**这是 ES 写比读重的根本原因**。

---

## 第二章：词典 FST —— 把字符串压缩到极致

### 2.1 问题

每个分片可能有几千万到几亿不同的 term。要支持"给一个 term 找到它的 posting list"，必须有一个高效的**词典**结构。

候选方案：
- 哈希表：O(1) 查找但**不支持范围 + 前缀查询**（`prefix:"elastic*"`）；且内存巨大
- B+ 树：支持范围但每节点固定开销，内存仍大
- Trie / Patricia：支持前缀；多 term 共享前缀省一些
- **FST (Finite State Transducer)**：Lucene 的选择

### 2.2 FST 的思想

FST = "前缀共享 + 后缀共享"的状态机。把 term 编码为状态机的路径：

```
词典：cat, can, dog
朴素：c-a-t / c-a-n / d-o-g       (8 个节点)
Trie：c-a-{t,n} / d-o-g            (6 个节点，前缀共享)
FST： c-a-{t,n} → 同一个 END        (5 个节点，后缀也共享)
       ^                ^
       d - o - g - END
```

**实测**：千万级英文 term 词典，FST 在内存只占几十 MB。**比朴素方案省 10-50x**。

### 2.3 FST 的另一个能力：term → "term 在文件中的 offset"

FST 不仅是个集合（in/out），还是个"映射"（state machine 走到终点输出一个值）。Lucene 用它把 term 映射到 posting list 在磁盘上的 offset。这意味着：

> **查一个 term 只需一次内存遍历 FST + 一次随机磁盘读 posting list**。

这是 Lucene 亚毫秒级查询的基础。

### 2.4 实际看一下

ES 提供了 `_analyze` API 看分词结果，但 FST 本身不暴露。你可以观察 segment 文件的 `.tip`（FST term index）+ `.tim`（terms dictionary 主文件）。一个典型 segment：

```
$ ls -la _0.*
_0.fdt   _0.fdx   _0.fnm   _0.nvd   _0.nvm
_0.tim   _0.tip   _0.doc   _0.pos   _0.pay
.cfs .cfe .si .liv ...
```

- `.tip` = term index（FST）
- `.tim` = term dictionary（term + 元数据）
- `.doc` = posting list（doc_id 列表）
- `.pos` = positions（用于短语查询）
- `.pay` = payloads / offsets

---

## 第三章：Posting List —— 文档号的高效压缩列表

### 3.1 单 term 的命中列表

`elasticsearch → [3, 17, 42, 199, 1024, 1025, 1026, 2000]`

朴素存：8 个 int32 = 32 字节。

### 3.2 Lucene 的优化

实际存的不是绝对 doc_id，而是 **delta**（差分）：

```
原始: [3, 17, 42, 199, 1024, 1025, 1026, 2000]
delta:[3, 14, 25, 157, 825,    1,    1,  974]
```

delta 序列里小数字明显增多。再配合**变长编码 / PFor / bitpacking** —— 一字节能编 7 bit 的小数。

Lucene 把 posting list 切成块（每块 128 doc）做这些编码。**典型压缩比 3-10x**。

### 3.3 SkipList 加速 AND 查询

`title:elasticsearch AND tag:search`：

- `elasticsearch` 的 posting list 长 1 万
- `search` 的 posting list 长 10 万

朴素：两个列表合并 → 11 万次比对。

实际：每个 posting list 维护 **skip pointers**——每隔 N 条记录一个跳跃指针，能在长 list 上快进。两个 list 合并时短的驱动长的，长的用 skip pointer 跳过大块不相干的 doc_id。**复杂度从 O(N+M) 降到 O(min(N,M) × log(max(N,M)))**。

这是为什么"两个高频词的 AND 查询" 也能亚秒级。

---

## 第四章：Doc Values —— 排序与聚合的列存正排

### 4.1 为什么倒排不够

倒排回答"包含 term X 的文档"。但搜索结果常常要：

- 按 `timestamp DESC` 排序
- 按 `category` 聚合统计
- `WHERE views > 1000` 过滤

这些都需要"给定 doc_id，取它的字段值"——**正排查询**。

老 Lucene 用 `fielddata`——把字段从倒排重新装载到 heap 上的 column 数组。问题：**heap 占用极大，是 OOM 的主要来源**。

### 4.2 Doc Values：磁盘列存

Lucene 4 引入 doc values：把"每个文档的某字段值"按列存到独立文件，**默认开启所有字段**（除了 `text` 字段——会被 analyzer 拆词，逐文档存不实用）。

```
doc_values 文件（每字段一组）：
  views.dvd: [doc_0 → 100, doc_1 → 500, doc_2 → 1234, ...]
  ts.dvd:    [doc_0 → ..., doc_1 → ..., ...]
```

读取时通过 **OS Page Cache**（mmap）按需加载，**完全不占 Java heap**。

### 4.3 keyword 比 text 聚合快的根因

- `keyword` 字段：原值整体作为一个 term；doc values 直接存原值 → 聚合扫一遍 doc values 即可
- `text` 字段：被分词成多个 term；要聚合"原始值"必须再回到 `_source`（昂贵），或者要在 mapping 加 `fielddata: true`（heap 浪费）

经验法则：**要聚合 / 排序 / 精确过滤的字段定义为 `keyword`**；要全文搜索的定义为 `text`；常用的字段两者都要（`text` + sub-field `.keyword`）。

```json
"category": {
  "type": "text",
  "fields": {
    "raw": { "type": "keyword" }
  }
}
```

之后 `category` 用于全文搜，`category.raw` 用于聚合 / 精确匹配。

### 4.4 关掉 doc values 省空间？

```json
"large_text_field": {
  "type": "keyword",
  "doc_values": false   // 不能再用于排序、聚合、script
}
```

只有你**确定永远不会聚合 / 排序 / script 访问** 这个字段时才关闭。**ES 索引体积常常一半来自 doc_values**——关掉能省不少磁盘，但失去后续灵活性。**日志场景的 raw_log 字段常关闭**。

---

## 第五章：Stored Fields 与 _source

### 5.1 _source 是什么

ES 写入文档时**默认把整个 JSON 原文存进 stored fields**（即 `.fdt` / `.fdx` 文件）。查询返回时从这里取 `_source` 返还。

```
PUT idx/_doc/42
{ "title": "abc", "tag": "x", "deep": {"nest": [1,2,3]} }

ES 实际存的 stored fields：
  doc_42: { "title": "abc", "tag": "x", "deep": {...} }   ← 原文压缩 (LZ4)
```

### 5.2 为什么默认存

简化使用——`GET idx/_doc/42` 直接拿到原文。但代价大：

- **磁盘 = 索引大小的 30-100%**：原文压缩了，但还是大头
- **fetch 阶段成本**：query 返回 doc_id 后要去 stored fields 读 _source 给你

### 5.3 何时该关 _source

```json
"_source": { "enabled": false }
```

**只有**这些条件全满足才考虑：

1. 你**永远不需要返回原文** —— 业务只关心搜索命中（如 audit log，只看是否出现）
2. 你**不做 reindex** —— reindex 必须读 _source
3. 你**不做 update by query** —— 同上
4. 你**接受失去未来灵活性**

99% 场景**不要关 _source**。如果只想省空间，改用 `_source.excludes`：

```json
"_source": { "excludes": ["binary_blob", "large_internal_field"] }
```

排除不需要返回的字段，但仍保留 reindex 能力。

### 5.4 LZ4 vs Deflate

```json
"index": { "codec": "best_compression" }   // 用 Deflate / Zstandard
```

默认 LZ4（快，压缩率 ~50%）。`best_compression` 用 Deflate（ES 8.16/8.17 起底层改为 Zstandard），压缩率 ~70% 但 indexing 慢 10-20%。

**适用**：cold / frozen tier 的存储。**不适用**：hot / warm（实时写入压力大）。

---

## 第六章：Segment 不变性 —— 一切设计的根

### 6.1 什么叫"不变"

Lucene 的核心约定：**写入完成的 segment 文件永不修改**。

后果：

- 删除？不真删，只在 `.liv` 文件里打"墓碑"位
- 更新？等价于"插入新版本 + 标记老版本墓碑"
- 高并发查询？读 segment 完全无锁
- 索引压缩？trivially 实现（不变的数据可做最优压缩）

### 6.2 不变性带来的代价

- **删除需要 merge 才能真清理空间**——只打墓碑的话，磁盘占用不降
- **更新即重写**——HBASE 风格"小修改大代价"
- **段越来越多**——必须定期合并

### 6.3 为什么 ES 选了不变性

不变性 = **能精确控制查询一致性**。Lucene 用 `IndexSearcher` 拿一个 segment 快照，**搜索期间永不变**——即使另一线程在写。这种"无锁、可重复"的特性是高并发查询的关键。

数据库就完全不同——B+ 树要支持原地改，要么牺牲并发（加锁），要么用 MVCC（额外 undo 链）。

---

## 第七章：Norms —— 评分必需的元数据

### 7.1 每个 (doc, field) 一个 norm 值

`title norms[doc_42] = 0.625` 这种。表示"这个文档的 title 字段有多长（短文档评分更高）"。Lucene 用一字节存 norm（量化到 256 档）。

### 7.2 norms 占用

每文档每字段 1 字节。对 10 亿文档 + 20 个字段的索引，norms 占 **20 GB**——不算小。

### 7.3 关闭 norms

```json
"my_keyword": {
  "type": "keyword",
  "norms": false   // 默认 keyword 已经 false
}
```

`keyword` 字段默认 `norms: false`——因为精确匹配不需要长度归一化。`text` 字段默认 `norms: true`——评分需要。

**优化**：如果某个 text 字段你只会用 filter / 不参与 score，可以显式关 norms。

---

## 第八章：Term Vectors —— 通常不要开

每文档每字段记录"该字段里出现了哪些 term + 各自频率 + 位置"。仅用于：

- `more_like_this` 查询（找相似文档）
- 高亮（少量场景）
- 部分情况下的快速重新分词

**默认关闭**。开启会让索引体积膨胀 30-50%。

```json
"content": {
  "type": "text",
  "term_vector": "with_positions_offsets"   // 5 档选项，开越大体积越大
}
```

99% 用不到——更现代的高亮（unified highlighter）不需要 term vectors。

---

## 第九章：dense_vector —— Lucene 9.x / 10.x 的新主角

### 9.1 字段类型

```json
"vec": {
  "type": "dense_vector",
  "dims": 768,
  "index": true,
  "similarity": "cosine",
  "index_options": {
    "type": "int8_hnsw",   // ES 8.14 起默认 int8_hnsw，省 4x 内存
    "m": 16,
    "ef_construction": 100
  }
}
```

### 9.2 底层结构：HNSW 图

HNSW = Hierarchical Navigable Small World 图。多层"邻居链表"，每层节点数指数级递减：

```
Layer 2:  少数 hub 节点
Layer 1:  中等量节点
Layer 0:  全部节点
```

搜索：从顶层任一节点开始 → 在当前层走"贪心选最近邻居"直到收敛 → 下到下一层重复。最终在 layer 0 给出 top K 近邻。

- 索引时间：O(log N × M × ef_construction)
- 查询时间：O(log N × M × ef_search)
- 内存：每节点 M × dims × 4 字节（or 1 byte 量化后）

### 9.3 量化：int8 / int4 / bbq

ES 8.12 起陆续提供 int8/int4/bbq 多档量化（bbq 于 8.16 预览、9.0 GA）：

| 量化 | 内存 | 召回率 | 适用 |
|---|---|---|---|
| `hnsw` (float32) | 4 × dims B | 100% | 小规模 |
| `int8_hnsw` (默认) | 1 × dims B | ~95% | 主流 |
| `int4_hnsw` | 0.5 × dims B | ~85% | 海量、可接受召回下降 |
| `bbq_hnsw` (binary quantized) | dims/8 B | ~70-80% | 极致内存 |

对 768 维向量：原始 3 KB → int8 768 B → int4 384 B → bbq 96 B。**1 亿向量从 300 GB 降到 9.6 GB**。

### 9.4 与倒排同时存在

`dense_vector` 字段独立存在 `.vec` / `.vex` / `.vem` 文件，**不和倒排互相干扰**。查询时可以"BM25 + kNN 同时跑 + RRF 融合"——这就是 hybrid search，详见 E08。

---

## 第十章：观察一个真实 segment

```
$ POST /myidx/_forcemerge?max_num_segments=1   # 先压实
$ GET /_cat/indices/myidx?v
$ GET /myidx/_stats/segments?human

"segments": {
  "count": 1,
  "memory_in_bytes": 12345678,         ← FST + 元数据占用（heap）
  "terms_memory_in_bytes": ...,
  "stored_fields_memory_in_bytes": ...,
  "norms_memory_in_bytes": ...,
  "points_memory_in_bytes": ...,
  "doc_values_memory_in_bytes": ...,
  ...
}
```

或者直接看 segment 文件系统：

```
$ ls -lah /var/lib/elasticsearch/nodes/0/indices/.../0/index/
total 5.2G
4.0K _0.cfe
512M _0.cfs        ← compound segment（一个 segment 一组文件打包）
4.0K _0.si
4.0K segments_3
4.0K write.lock
```

`.cfs` 是 compound file segment——把上述 `.tim` `.tip` `.doc` 等十几个文件打包成一个，减少文件句柄。default true。

---

## 第十一章：生产级最佳实践

1. **`keyword` vs `text` 用对**——聚合 / 排序 / 精确过滤 = `keyword`；全文检索 = `text`；常常两者都要（`text` + `text.keyword`）。
2. **`_source` 不要关**——除非真的不需要返回原文且不做 reindex。要省空间用 `_source.excludes`。
3. **不要存 term vectors**——99% 场景不需要，开了膨胀 30-50%。
4. **dense_vector 用 int8_hnsw 量化**——默认就是；除非召回要求极高才用 float32。
5. **关心 segment 数量**——单分片 segment 数维持在 10-30 之间；过多 → force_merge；过少（如 1）则失去新写入的灵活性。
6. **codec 分层**：hot 用 LZ4（`default`）；cold/frozen 用 `best_compression`（Zstandard）。
7. **`doc_values: false` 谨慎使用**——能省空间但失去未来灵活性。
8. **norms 在 filter-only text 字段上关掉**——少占 1 字节/(doc, field)。
9. **不要用 nested 类型存大数组**——每个 nested 对象是一个独立 hidden doc；10 个 nested 等于 10 倍文档数。
10. **预估磁盘占用** = `_source(0.5x) + 倒排(0.5x) + doc_values(0.3x) + norms(0.05x) + 向量(按量化) ≈ 1.3-2x 原始 JSON`。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：用 text 字段聚合
```json
"GET /idx/_search": {"aggs":{"by_cat":{"terms":{"field":"category"}}}}   // category 是 text → 报错"fielddata not enabled"
```
解决：定义 `category.keyword` 子字段；或更彻底地把 category 改成 `keyword` 主字段。

### ❌ 陷阱 2：dynamic mapping 把日期识别成 keyword
首条文档某字段是 `"2026-05-13"` 字符串 → 第二条变成 `"abcdef"` 字符串 → ES 把它当 keyword。**生产严禁 dynamic mapping**——显式 mapping，或用 dynamic templates。

### ❌ 陷阱 3：大文档 _source 把内存打爆
深嵌套 + 大数组的 _source 可能几 MB / doc。`SELECT _source FROM idx LIMIT 1000` 直接吃几 GB 网络 + heap。改用 `_source.includes` 只取需要字段。

### ❌ 陷阱 4：以为关掉 _source 没事
关 `_source` 后**不能 reindex、不能 update by query、不能 update by script、不能 cross-cluster replicate**。生产代价极大。

### ❌ 陷阱 5：document 用 ID = UUID 字符串
高基数 + 字符串 ID = doc_values / FST 都大。优先用顺序整数 ID（不指定 ES 自分配）；必须用 UUID 时用 UUIDv7（时间序，索引友好）。

### ❌ 陷阱 6：在同一索引存 1KB 文档和 10MB 文档
ES 写入流水会被大文档拖慢。**按文档大小分索引**。

### ❌ 陷阱 7：term vector 全开
误开 `with_positions_offsets_payloads` → 索引体积膨胀 2x。改用 unified highlighter，不需要 term vectors。

### ❌ 陷阱 8：用 nested 表示标签 / 多 tag
```
{"tags": [{"name":"a"},{"name":"b"}]}   // nested 类型
```
不必要的 nested 会让 doc 数 × 标签数。改成 keyword 数组：
```
{"tags": ["a", "b"]}                    // keyword array
```

### ❌ 陷阱 9：在频繁写的索引上 force_merge
force_merge 会**长时间锁定写入**且占大量 I/O。**只在 read-only 索引上做**（如 ILM 把 hot 切到 warm 后，再 force_merge）。

### ❌ 陷阱 10：以为 segment 数高没事
segment 多 = 查询要扫多份 = CPU / I/O 增加；FST heap 占用线性增长。生产 segment 数控制在 < 50 / 分片。

---

## 第十三章：练习题

**练习 1**：解释 FST 与 Trie 的核心差别，以及为什么 Lucene 选 FST 而不是 HashMap。

**练习 2**：以下两种 mapping，对存储 1 亿条用户行为日志（每条 ~200 字节 JSON）哪种磁盘占用更大？

```
// A
{ "user_id": "keyword", "event": "text", "_source": { "enabled": true } }

// B
{ "user_id": "keyword", "event": "text",
  "_source": { "enabled": false },
  "store": true   // 字段级单独 stored
}
```

**练习 3**：业务场景：每个商品有 ID、标题（中文，需要搜索 + 高亮）、tags（用于聚合）、价格（用于排序）、向量 embedding（用于相似检索）。写出 mapping。

**练习 4**：解释为什么大量 nested 类型会让索引体积放大几倍，以及替代方案。

**练习 5**：你怎么估算 1 亿条 768 维 float 向量在 ES 9 上用 `int8_hnsw` 索引会占多少磁盘？

---

## 参考答案

**练习 1**：Trie 只共享前缀；FST 同时共享前缀和后缀（DAG 而非树），且把每个 term 关联一个 output（如 posting list offset）。对千万级英文 term，FST 比 Trie 省 3-5x、比朴素 HashMap 省 50x。HashMap 不支持前缀 / 范围查询（`prefix:"elastic*"`），无法满足搜索需求。

**练习 2**：B 通常**更大**——`store: true` 是字段级单独存储（重复了 doc_values），而 `_source` 是统一压缩存。Lucene 对 _source 做 LZ4 块压缩，比每字段单独存高效。除非你的字段非常稀疏（一个文档只有少数字段），否则关 _source 改 store 反而占更多空间。

**练习 3**：

```json
PUT products
{
  "mappings": {
    "properties": {
      "id":     { "type": "keyword" },
      "title":  {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "raw": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "tags":   { "type": "keyword" },
      "price":  { "type": "scaled_float", "scaling_factor": 100 },
      "embedding": {
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine",
        "index_options": { "type": "int8_hnsw", "m": 16, "ef_construction": 100 }
      }
    }
  }
}
```

要点：title 用 IK 中文分词；title.raw 用于精确匹配 / 聚合；price 用 scaled_float（比 double 省一半空间且整数运算更快）；embedding 用 int8 量化的 HNSW。

**练习 4**：每个 nested 对象在 Lucene 是一个隐藏文档，被父文档"关联"。一个父文档有 100 个 nested 元素 = Lucene 实际有 101 个 doc。磁盘 / 内存 / 查询代价都按"隐藏文档数"算。替代方案：

1. **keyword 数组**：标签等无内部结构的，直接 array
2. **flattened 类型**：所有字段当 keyword 平铺（失去类型推断但很省）
3. **拆分索引**：父子放两个索引，按需 join（业务层处理）

**练习 5**：
- 单向量内存：768 dims × 1 byte (int8) = 768 B
- 1 亿向量原始：76.8 GB
- HNSW 图 overhead：M=16 → 每节点 16 个邻居 × 4 字节指针 = 64 B
- 总向量索引：~76.8 + 6.4 = **~85 GB**
- 加上 _source（如果向量也存在 _source 里，每条 ~3 KB JSON）：300 GB 多——所以**生产实践常 `_source.excludes: ["embedding"]`** 把向量从 _source 排除，节省一大半磁盘。

---

## 小结

| 数据结构 | 文件 | 用途 | 关键优化 |
|---|---|---|---|
| FST | .tip / .tim | term → posting offset 词典 | 前缀+后缀共享 |
| Posting List | .doc / .pos / .pay | term 命中的 doc_id 列表 | delta + PFor + skip pointer |
| Doc Values | .dvd / .dvm | 列存正排，排序聚合用 | mmap 不占 heap |
| Stored Fields | .fdt / .fdx | 原始 _source 压缩存 | LZ4 / Zstandard |
| Norms | .nvd / .nvm | per-(doc,field) 长度归一化 | 1 字节量化 |
| Term Vectors | .tvx / .tvd | per-doc 词频 + 位置 | 默认关 |
| Vector Index | .vec / .vex / .vem | HNSW 图 | int8 / int4 / bbq 量化 |

四条铁律：

1. **倒排负责"找文档"，正排负责"看字段"**——doc_values 是分水岭
2. **segment 不变性是一切设计的根**——决定了写入流程、合并策略、版本一致性
3. **磁盘占用 ≈ 1.3-2x JSON 原文**——按这个估算容量
4. **字段映射决定一切**——`text` vs `keyword`、`norms`、`doc_values` 在第一次写入时就固化，事后改要 reindex

下一篇 **E02 — 精通 Segment 与合并策略** 将拆开 refresh / flush / merge 三个过程，讲清"为什么 ES 是近实时" 与 "为什么我的索引体积比数据大很多"。
