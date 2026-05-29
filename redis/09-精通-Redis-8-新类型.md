# 精通 Redis 8 新数据类型：JSON、TimeSeries、Bloom、Vector Set

> 关联章节：[01 数据结构](./01-精通-Redis-数据结构内部.md)、[11 Redis vs Valkey](./11-精通-Redis-vs-Valkey.md)

---

## 引言：从 KV 到"实时数据平台"

Redis 7.x 时代，**JSON / TimeSeries / Bloom / Cuckoo / Top-K / Count-Min Sketch / T-Digest** 都是独立的 Redis 模块（属于 RedisStack 子项目），需要单独 LOAD 加载。许多公司只用了"原版核心 Redis"，没装模块，结果在需要这些能力时被迫上 MongoDB / InfluxDB / Bloom 算法库。

**Redis 8.0 (2025-05) 把这些模块全部集成进核心**，作为内置数据类型——`MODULE LOAD` 都不需要。还新增了 **Vector Set**（beta，向量集合），让 Redis 也能做 AI 相似度检索。

这一章拆开看 Redis 8 内建的 8 类新数据类型：

- **JSON**：嵌套文档 + JSONPath 子文档操作
- **TimeSeries**：时间序列点 + 聚合规则
- **Bloom / Cuckoo Filter**：概率成员判定（"可能在 / 一定不在"）
- **Top-K**：top N 频次估算
- **Count-Min Sketch**：频次估算（误差有界）
- **T-Digest**：分位数估算（流式 P50/P99）
- **Vector Set**：向量相似度检索（beta）

> ⚠️ 这些功能 **Redis 8 + 内置**；**Valkey 8.x** 还在按子模块各自演化。如果你用 Valkey，需要单独装对应 module（valkey-search / valkey-json / valkey-bloom 等）。详 [11 Redis vs Valkey](./11-精通-Redis-vs-Valkey.md)。

---

## 第一章：JSON 类型

### 1.1 为什么需要

老办法用 String 存 JSON：

```
SET user:42 '{"name":"alice","age":30,"hobbies":["go","redis"]}'
GET user:42         # 拿整个文档
```

每次改一个字段都要 GET 整个 → 改 → SET 整个。**业务上没法局部更新**。

Redis 8 JSON 类型支持 **JSONPath** 表达式精细操作子文档：

```
JSON.SET user:42 $ '{"name":"alice","age":30,"hobbies":["go","redis"]}'
JSON.GET user:42 $.name                  # "alice"
JSON.SET user:42 $.age 31                # 局部更新
JSON.ARRAPPEND user:42 $.hobbies '"python"'
JSON.GET user:42 $..hobbies              # 递归找所有 hobbies
```

### 1.2 物理存储

不是简单存 JSON 字符串——内部用一个**树形结构**（C 实现，类似 RapidJSON 的 DOM）：

- 解析一次后 in-memory tree
- JSONPath 操作直接遍历树，无需重解析
- 序列化（JSON.GET）按需

内存代价：约 JSON 文本的 2-3 倍（因为是 DOM 而非压缩字符串）。

### 1.3 主要命令

```
JSON.SET key path value           # 设值
JSON.GET key [path]               # 取值
JSON.DEL key path                 # 删子节点
JSON.TYPE key path                # 看类型
JSON.NUMINCRBY key path n         # 数字字段 +=
JSON.STRAPPEND key path "x"       # 字符串字段 append
JSON.ARRAPPEND key path el        # 数组 push
JSON.ARRINDEX key path val        # 数组中找 index
JSON.OBJKEYS key path             # 取对象的所有 key
```

JSONPath 表达式：

- `$` = root
- `$.name` = root 的 name 字段
- `$.users[*].age` = 所有 user 的 age
- `$..price` = 递归找所有 price

### 1.4 与 Hash 的取舍

```
# Hash 方案
HSET user:42 name alice age 30
HGET user:42 age

# JSON 方案
JSON.SET user:42 $ '{"name":"alice","age":30}'
JSON.GET user:42 $.age
```

| 维度 | Hash | JSON |
|---|---|---|
| 扁平字段（K-V） | ★★★★★ | ★★★ |
| 嵌套结构 | 需自己 flatten | ★★★★★ |
| 数组操作 | 用 List + 联动 | 原生 ARR* |
| 内存效率 | listpack 极省 | DOM 树有 overhead |
| 索引 | 无 | RediSearch 配合 |

**经验值**：纯扁平对象用 Hash；嵌套结构 + 需要 JSONPath 用 JSON。

### 1.5 配合 RediSearch（索引）

Redis 8 还内置了 RediSearch（搜索）。可以对 JSON 字段建索引：

```
FT.CREATE idx ON JSON PREFIX 1 user: SCHEMA
  $.name AS name TEXT
  $.age AS age NUMERIC SORTABLE
  $.hobbies[*] AS hobbies TAG

FT.SEARCH idx '@name:alice @age:[20 40]'
```

支持全文 / 数值范围 / 标签 / 地理 / **向量**——把 Redis 变成"轻量级文档库"。

---

## 第二章：TimeSeries

### 2.1 数据模型

```
TS.CREATE temperature:room1 RETENTION 86400000 LABELS room "1" type "sensor"
TS.ADD temperature:room1 * 23.5            # * = 当前时间
TS.ADD temperature:room1 1715587200000 24.0

TS.RANGE temperature:room1 - +              # 全部
TS.RANGE temperature:room1 1715587000000 1715588000000 AGGREGATION avg 60000  # 按 60s 聚合
```

每个 TimeSeries key 是一个独立的时间序列：(timestamp, value) 对的有序集合，带 labels。

### 2.2 物理存储

- 每个 sample = 16 字节（8 字节 ts + 8 字节 double value）
- 内部按 **chunk** 组织（默认每 chunk 4KB ≈ 250 samples）
- 老 chunk 可压缩（Gorilla 算法）→ 压缩比 3-10x
- 内存中维护 chunk 列表 + 索引

10 亿点（约 100 个 TS × 1000 万点）：原始 16GB，压缩后 ~3-5GB。

### 2.3 Compaction Rule（聚合下采样）

```
TS.CREATERULE temperature:room1 temperature:room1:5m AGGREGATION avg 300000
```

源 TS 的每个 sample 自动按 5 分钟桶聚合到目标 TS。组合多层：

- 原始 1s 粒度，保留 1 天
- 5 分钟聚合，保留 30 天
- 1 小时聚合，保留 1 年

监控类业务标准做法。

### 2.4 多 TS 跨标签查询

```
TS.MRANGE - + AGGREGATION avg 60000 FILTER room=(1,2,3) type=sensor
```

按 labels 选择匹配的多个 TS 并行查询。

### 2.5 与 InfluxDB / Prometheus 对比

| 维度 | Redis TimeSeries | InfluxDB | Prometheus |
|---|---|---|---|
| 单实例吞吐 | 100k samples/s | 500k+ | 1M+ |
| 持久化 | Redis 自己（RDB/AOF） | 自有 TSDB | 自有 TSDB |
| 高基数 cardinality | 中等 | 弱（OSS 版） | 中等（cardinality 爆炸是经典坑） |
| 集成成本 | 已有 Redis 零成本 | 独立部署 | 独立部署 |
| 适用 | <100GB 时序，应用内时序 | 中型时序数据库 | 监控 + alert |

**经验值**：应用内业务时序（用户在线、行为 metrics、IoT 设备）用 Redis TimeSeries 完全够；大规模时序数仓 → 专用 TSDB。

---

## 第三章：Bloom Filter

### 3.1 概率成员判定

**问题**：判断一个元素"是否在集合里"。Set 精确但占用 = N × 元素大小。Bloom Filter 概率（"一定不在"或"可能在"），但占用 = 几 bit / 元素。

```
BF.RESERVE filter 0.001 1000000          # 错误率 0.1%，预期 100 万元素
BF.ADD filter "user_42"
BF.EXISTS filter "user_42"               # 1（可能存在）
BF.EXISTS filter "user_99"               # 0（一定不存在）
BF.MADD filter "a" "b" "c"
BF.MEXISTS filter "a" "x" "b"
```

100 万元素、0.1% 错误率：约 1.4 MB（≈ 1.5 字节/元素）。比 Set 省 100 倍。

### 3.2 算法

- k 个独立哈希函数把元素映射到 m 个 bit
- ADD：把这 k 个 bit 都置 1
- EXISTS：检查这 k 个 bit——全为 1 才"可能在"

错误率随元素数增加上升（设计预期 N 不变）。

### 3.3 应用

- **缓存击穿防护**：查 DB 前先 BF.EXISTS，不存在的 key 直接返回，避免大量 DB 查询
- **海量去重**：UV 统计（HyperLogLog 更便宜但只算 count，不能查"某个 ID 是否见过"）
- **黑名单**：禁言 / 风控

注意：Bloom Filter **不能 delete**——删除某 bit 可能让其他元素也变成"不存在"。要删除场景用 **Cuckoo Filter**。

---

## 第四章：Cuckoo Filter

### 4.1 与 Bloom 的差别

| 维度 | Bloom | Cuckoo |
|---|---|---|
| 假阳性 | ✓ | ✓ |
| 假阴性 | ✗（一定不存在 = 一定不存在） | ✗ |
| 支持 Delete | ✗ | ✓ |
| 空间 | 1-2 字节/元素 | 略大 |
| 性能 | O(k) | O(1) amortized |

```
CF.RESERVE filter 1000000
CF.ADD filter "abc"
CF.EXISTS filter "abc"
CF.DEL filter "abc"        # ★ Cuckoo 独有
```

**唯一需要 Cuckoo 的场景**：需要删除元素 + 概率 OK。否则 Bloom 更省更快。

---

## 第五章：Top-K

### 5.1 频次最高的 N 个元素

```
TOPK.RESERVE topk 10 1000 5 0.9       # top 10、buckets 1000、hashes 5、decay 0.9
TOPK.ADD topk "user_42" "user_42" "user_99" "user_42"
TOPK.LIST topk                          # 返回 top 10 element list
TOPK.QUERY topk "user_42"               # 该元素是否在 top
```

基于 [HeavyKeeper](https://yangtonghome.github.io/uploads/HeavyKeeper_ToN.pdf) 算法。空间 O(K)，**比"维护完整频次 + 取 top"省 100x+ 内存**。

### 5.2 应用

- 实时排行榜（最热商品、最活跃用户）
- 异常检测（哪个 IP 请求最多 → 风控）
- 日志分析（最高频错误）

---

## 第六章：Count-Min Sketch

### 6.1 频次估算

不限元素数 N，估算每个元素的频次（不保证精确，**只多不少**）。

```
CMS.INITBYPROB cms 0.001 0.001            # 误差 0.1%，置信度 99.9%
CMS.INCRBY cms "user_42" 1 "user_99" 3
CMS.QUERY cms "user_42"                   # 估算频次
```

空间常数（如 ~10 KB），无论 N 多大。**用于流式频次统计**（DDOS 检测、TPS 估算）。

---

## 第七章：T-Digest

### 7.1 流式分位数

实时计算 P50 / P95 / P99 而不存全部数据。

```
TDIGEST.CREATE latency COMPRESSION 100
TDIGEST.ADD latency 1.2 3.4 5.6 ... 1000.1
TDIGEST.QUANTILE latency 0.99                 # P99
TDIGEST.RANK latency 100.0                    # 100.0 是第几位
```

空间 O(1)。误差 < 1%。**完美适合 SRE / 性能监控**——实时给业务的"过去 N 秒延迟 P99"。

---

## 第八章：Vector Set（Redis 8.0 Beta）

### 8.1 向量相似度检索

```
# 创建一个 Vector Set：VADD key (FP32 | VALUES num) vector element
# 第一次 VADD 即建立 Vector Set；维度由首次插入的向量决定
VADD myset VALUES 4 1.5 0.3 -0.2 0.8 element1
VADD myset VALUES 4 0.7 1.1 0.5 -0.3 element2

# 搜索：VSIM key (ELE | FP32 | VALUES num) (vector | element) [COUNT n] [WITHSCORES]
VSIM myset VALUES 4 0.9 0.4 -0.1 0.5 COUNT 10           # 用查询向量
VSIM myset ELE element1 COUNT 10 WITHSCORES              # 以已有元素为查询
VCARD myset
```

默认使用 cosine 相似度 + Q8（int8）量化。可在首次 VADD 时用 `NOQUANT`（FP32 全精度）或 `BIN`（二值）切换。底层 HNSW 图（同 Elasticsearch dense_vector / Postgres pgvector 的算法）。

### 8.2 与 RAG 场景

```
# 1. 把每个文档段的 embedding 存入 Vector Set
for doc in corpus:
    embedding = openai_embed(doc.text)        # 假设 dim=1536
    r.execute_command("VADD", "docs", "VALUES", len(embedding), *embedding, doc.id)

# 2. 用户查询
query_emb = openai_embed(user_query)
results = r.execute_command(
    "VSIM", "docs", "VALUES", len(query_emb), *query_emb, "COUNT", 5
)

# 3. 把 top 5 文档段交给 LLM 做 RAG
```

Redis Vector Set 适合**少量到中等规模（< 1000 万向量）+ 低延迟**的向量检索。**超大规模**（亿级）仍推荐专用向量库（Milvus / Qdrant / Pinecone / Elasticsearch dense_vector）。

### 8.3 配合 RediSearch FT 向量字段

更完整的方案：JSON + Vector + RediSearch index：

```
FT.CREATE idx ON JSON PREFIX 1 doc: SCHEMA
  $.text AS text TEXT
  $.embedding AS embedding VECTOR HNSW 6
    TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE

FT.SEARCH idx '*=>[KNN 5 @embedding $vec]' PARAMS 2 vec $blob
```

Hybrid search（全文 + 向量）一站式。

---

## 第九章：实战案例

### 9.1 用户活跃度 + Top-K

```python
# 每次用户活动
r.execute_command("TOPK.ADD", "active_users", user_id)
r.execute_command("CMS.INCRBY", "user_freq", user_id, 1)

# 实时取最活跃用户
top_users = r.execute_command("TOPK.LIST", "active_users")

# 估算某用户访问次数
freq = r.execute_command("CMS.QUERY", "user_freq", user_id)
```

### 9.2 缓存击穿防护（Bloom）

```python
def get_user(user_id):
    # 1. Bloom 预检
    if not r.execute_command("BF.EXISTS", "users_bf", user_id):
        return None    # 一定不在 → 不查 DB

    # 2. Redis 缓存
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # 3. DB
    user = db.query("SELECT ...", user_id)
    if user:
        r.set(f"user:{user_id}", json.dumps(user), ex=3600)
        # （注意：新用户也要 BF.ADD users_bf user_id）
    return user
```

### 9.3 实时 P99 监控

```python
# 应用每次请求结束
latency_ms = request.elapsed_ms
r.execute_command("TDIGEST.ADD", "api:latency:5min", latency_ms)

# 监控系统每 10 秒
p99 = r.execute_command("TDIGEST.QUANTILE", "api:latency:5min", 0.99)
prom_export("api_latency_p99", p99)
```

### 9.4 实时 RAG

参考 §8.2。

---

## 第十章：生产级最佳实践

1. **不需要的不开**——Redis 8 把模块内置但消耗内存。仅业务用到的才使用
2. **JSON 大文档拆**——单 JSON 超 1MB 时考虑拆 key 或换 Hash
3. **TimeSeries 一定配 compaction rule**——否则原始粒度数据占内存爆
4. **Bloom Filter 容量预估**——超过预期 N，错误率上升明显
5. **Vector Set 大规模上 RediSearch + HNSW**——Vector Set 本身 beta
6. **TopK/CMS/T-Digest 误差可接受性能换内存**——监控类业务理想
7. **大 JSON 用 JSON.GET 子路径** 不要 `$`——拿子节点比全文便宜
8. **业务幂等性 + 这些类型的"非精确"特性配合好**——别让概率结构成为业务一致性的关键路径
9. **Valkey 用户单独装 module**——versionkey 8.x 没有自动集成
10. **同一个 key 不要混用**——一个 key 是 JSON 就别 SET 它

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：JSON 大文档反复全量 GET/SET
失去 JSONPath 优势。改用 JSON.GET / JSON.SET path。

### ❌ 陷阱 2：TimeSeries 不修剪
RETENTION 必设，否则永远增长。

### ❌ 陷阱 3：Bloom Filter 删除元素
不支持。需要删用 Cuckoo。

### ❌ 陷阱 4：Bloom Filter 容量算错
预期 100 万实际放进 500 万 → 错误率从 0.1% 飙到 10%+。

### ❌ 陷阱 5：Vector Set 用了"还在 beta"的命令
Redis 8.0 Vector Set 命令仍可能在 8.x 内调整。生产环境锁版本。

### ❌ 陷阱 6：把 T-Digest 当精确分位数
误差 < 1%。如果业务要精确（SLA 计算），用专用统计库。

### ❌ 陷阱 7：Top-K 期望"严格 top N"
Top-K 是估算。低频元素可能错。要严格 top 用 ZSet。

### ❌ 陷阱 8：在 Valkey 上用 Redis 8 命令报错
"unknown command 'JSON.GET'" → Valkey 没装对应 module。

### ❌ 陷阱 9：JSON 字段名带 `.` 或 `[]`
JSONPath 解析歧义。用 `["field.name"]` 或避免特殊字符。

### ❌ 陷阱 10：单 TS 写入太快
单 TS key 是单实例资源（同 hash 等），写入 > 100k/s 会成瓶颈。按 label 切多个 TS。

---

## 第十二章：练习题

**练习 1**：用 JSON 类型存"用户购物车"，包含商品列表 + 总价 + 元数据。设计 schema + 给出"加一个商品 + 总价 +=" 的原子操作。

**练习 2**：网站每秒接收 100k UV 事件。设计一个 30 天去重 + 实时活跃排行 + P99 响应时间的方案。

**练习 3**：1 亿用户的 Bloom Filter，错误率 0.01%。算内存占用、ADD/EXISTS 性能。

**练习 4**：T-Digest 与"传统按桶统计"（如 `HINCRBY latency_buckets <bucket> 1`）算 P99 哪个更准、哪个更快？

**练习 5**：Vector Set vs Elasticsearch dense_vector vs Milvus 在 100 万 768 维向量场景下怎么选？

---

## 参考答案

**练习 1**：

```
JSON.SET cart:42 $ '{"items":[],"total":0,"updated_at":0}'

# 加商品 + 总价 += 单价
JSON.ARRAPPEND cart:42 $.items '{"item_id":1,"qty":2,"price":19.9}'
JSON.NUMINCRBY cart:42 $.total 39.8
JSON.SET cart:42 $.updated_at 1715587200000
```

Lua 包裹保证原子（NUMINCRBY 和 ARRAPPEND 是分开两条命令）：

```lua
redis.call('JSON.ARRAPPEND', KEYS[1], '$.items', ARGV[1])
redis.call('JSON.NUMINCRBY', KEYS[1], '$.total', ARGV[2])
redis.call('JSON.SET', KEYS[1], '$.updated_at', ARGV[3])
return redis.call('JSON.GET', KEYS[1])
```

**练习 2**：
- 去重：Bloom Filter `BF.RESERVE uv_bf 0.001 100000000`（1 亿容量，0.1% 错误）→ 内存 ~180MB
- 排行：TopK `TOPK.RESERVE top_users 100 50000 5 0.9` → 内存 ~几 MB
- P99：T-Digest 每条响应时间 TDIGEST.ADD，每分钟切一个 key（`latency:202605131030`）

代码：

```python
def on_event(user_id, latency_ms):
    r.execute_command("BF.ADD", "uv_bf", user_id)
    r.execute_command("TOPK.ADD", "top_users", user_id)
    key = f"latency:{time.strftime('%Y%m%d%H%M')}"
    r.execute_command("TDIGEST.ADD", key, latency_ms)
    r.expire(key, 86400 * 30)
```

**练习 3**：1 亿元素、0.01% 错误率，约 19 bit / 元素 = **240 MB**。ADD/EXISTS 都是 O(k)（k 个 hash），通常 7-10 个 hash → 单次操作几微秒。

**练习 4**：
- **传统桶**：内存固定 = 桶数 × 8 字节 / 桶。P99 精度取决于桶大小（如 1ms 桶，P99 精度 1ms）；超出桶范围全归入最后一个桶，**长尾不准**。性能：HINCRBY O(1)，查询遍历所有桶 O(B)
- **T-Digest**：内存 O(compression)，自适应分桶（在分位数附近桶密，极值附近桶疏）。**长尾 P99/P999 更准**。性能：ADD O(log compression)，QUANTILE O(log compression)

**结论**：T-Digest 长尾更准、内存自适应；传统桶简单可控。监控类 SLA 计算 → T-Digest。

**练习 5**：100 万 768 维向量
- 内存原始：100 万 × 768 × 4 字节 = 3 GB；int8 量化后 ~770 MB
- **Vector Set / RediSearch**：已有 Redis 集群零成本，吞吐 1-5k QPS，延迟亚秒。适合**轻量 RAG**
- **Elasticsearch dense_vector + HNSW + int8**：与现有 ES 集成；查询性能与 Redis 相近；适合**已有 ES 栈**
- **Milvus**：专用向量库，亿级规模 + 各种 ANN 算法；运维独立；适合**向量为核心业务**

100 万规模都能跑，选型主要看现有技术栈和未来规模规划。

---

## 小结

| 类型 | 算法基础 | 内存效率 | 典型场景 |
|---|---|---|---|
| JSON | DOM 树 | 中 | 嵌套文档 + JSONPath |
| TimeSeries | Chunk + Gorilla | 高 | 时序点 |
| Bloom Filter | 位数组 + 多哈希 | 极高 | 去重、缓存防护 |
| Cuckoo Filter | 多哈希 + 替换 | 高 | 需删除的去重 |
| Top-K | HeavyKeeper | 极高 | 实时排行 |
| Count-Min Sketch | 计数数组 | 极高 | 流式频次 |
| T-Digest | 自适应桶 | 极高 | 流式分位数 |
| Vector Set | HNSW（beta） | 中 | 向量相似度 |

四条铁律：

1. **概率类型换内存**——业务能容忍 0.x% 误差才用
2. **JSON 适合嵌套，Hash 适合扁平**
3. **TimeSeries 必带 RETENTION + Compaction Rule**
4. **Vector Set 仍 beta，大规模上 RediSearch + HNSW**

下一篇 **R10 — 精通 Redis 生产实践与陷阱**：bigkey、hotkey、缓存三大经典问题、ACL 与安全。
