# Elasticsearch 深度课程 · 自测题库

> 配合 [INDEX.md](./INDEX.md) 使用。每章 ~10 题，含答案与展开解析。
> 难度标记：⭐ 概念 ⭐⭐ 进阶 ⭐⭐⭐ 源码级 / 容易踩坑

> 答题建议：先盖住答案自答，再展开核对。能讲出"为什么"比答对结论更重要。

---

## E01 倒排索引与 Lucene

### Q1.1 ⭐⭐ 倒排索引的两个核心数据结构是什么？

<details><summary>答案</summary>

**Term dictionary（词典）** + **Posting list（倒排链）**。

- Term dictionary：term → posting list 入口指针。Lucene 用 **FST（Finite State Transducer）** 压缩，把所有 term 表达为最小有限状态自动机。优势：磁盘占用极小、共享前缀/后缀、O(len(term)) 查找。
- Posting list：每个 term 对应的文档列表，记 `docId, freq, positions, offsets, payload`，Lucene 用 **PFor delta**（Patched Frame-of-Reference）压缩。

两层结构让搜索是 O(查询 term 数 × len(term) + 命中文档数) 复杂度，不是 O(文档总数)。

</details>

### Q1.2 ⭐⭐ FST 是什么？为什么 Lucene 选它而不是哈希表？

<details><summary>答案</summary>

**FST = Finite State Transducer**，有限状态转换机。把"输入字符串 → 输出值"压成一个 DAG。

为什么选 FST 不选哈希：

1. **内存占用极小**：共享前缀和后缀，对自然语言词汇尤其有效。1 亿英文词 FST 可压到几百 MB，哈希表要几 GB。
2. **顺序遍历友好**：支持范围查询 / 前缀查询（哈希不行）。
3. **mmap 友好**：FST 是只读结构、连续内存，mmap 进进程地址空间不用反序列化。

代价：构建慢（O(N) 但常数大），但 Lucene 是**写一次读多次**模型，能承受。

</details>

### Q1.3 ⭐⭐⭐ 一个查询 `match: {"name": "redis cluster"}` 在倒排索引中走多少步？

<details><summary>答案</summary>

1. 分词器把 "redis cluster" 切成 `["redis", "cluster"]`
2. 在 term dict（FST）查 "redis" → 拿到 posting list 偏移
3. 解码 posting list `redis` → docId 序列 + tf/positions
4. 在 term dict 查 "cluster" → 拿到偏移
5. 解码 posting list `cluster`
6. 求两个 posting list 的**布尔 OR / AND**（默认 OR，可改）
7. 对每个候选 doc 计算 BM25 评分
8. 维护 top-K 堆，返回 K 个结果

注意 positions 用于 **phrase query**（要求相邻），普通 match 用不到。

</details>

### Q1.4 ⭐ stored fields、doc_values、source 三者区别？

<details><summary>答案</summary>

| 维度 | source | stored_fields | doc_values |
|---|---|---|---|
| 内容 | 原始 JSON 全文 | 单个字段值 | 单个字段值（列式） |
| 用途 | fetch 返回结果 | 同 source 但单字段 | 排序 / agg / script |
| 默认 | 默认开 | 默认关 | 默认开（除 text） |
| 编码 | 行式（一行一文档） | 行式 | 列式 |

实战：

- 大部分场景留 source 关 stored_fields
- 只对**单大字段**（PDF 内容、大 JSON）独立 stored，配 `_source` excludes 不存到 source
- doc_values 是聚合 / 排序的物理基础

</details>

### Q1.5 ⭐⭐ `keyword` 和 `text` 的区别？

<details><summary>答案</summary>

| 字段 | 分词 | 倒排 | doc_values | 用途 |
|---|---|---|---|---|
| `keyword` | ❌ 不分词整体存 | ✅ 单 term | ✅ | term 查、agg、sort、精确匹配 |
| `text` | ✅ 经 analyzer | ✅ 多 term | ❌ | 全文匹配，BM25 评分 |

常见做法：业务字段同时定义 `text` + `.keyword` 子字段：

```json
"name": {
  "type":"text",
  "fields":{"keyword":{"type":"keyword","ignore_above":256}}
}
```

全文搜走 `name`，精确匹配 / 聚合走 `name.keyword`。

</details>

### Q1.6 ⭐⭐ Posting list 用什么压缩？

<details><summary>答案</summary>

Lucene 用 **PFor delta**（Patched Frame-of-Reference）：

1. **Delta encoding**：docId 单调递增，存差值（更小的数字）
2. **Frame of Reference**：一批数字找 base 和 bit width，每个数用 `b` 位表示（如果差值都 < 2^b）
3. **Patched**：少数 outlier（远大于其他）单独存，主流用紧凑 b 位

对 docId 序列，原始 32-bit 整数能压到平均 3-5 bit/doc。

</details>

### Q1.7 ⭐⭐⭐ 为什么 Lucene 设计成 segment 不可变？

<details><summary>答案</summary>

不可变带来：

1. **无锁并发读**：读时不用加锁，segment 内容固定
2. **强缓存友好**：FST、posting list、doc_values 文件能放 OS page cache，命中率高
3. **写入高吞吐**：append-only，没有更新冲突
4. **崩溃恢复简单**：每个 segment 完整即可用

代价：

- 删 = 标记删除位（liveDocs），物理删要等 merge
- 更新 = 删 + 新建
- 段越多查询要遍历越多 → 需要 merge

设计取舍：把"写"的复杂性放到 merge 异步过程，"读"路径简单纯净。

</details>

### Q1.8 ⭐⭐ Lucene 的 norm 字段是什么？

<details><summary>答案</summary>

`norm` 存每个 doc 每个 field 的**字段长度归一化值**，BM25 评分的 `fieldNorm` 来自这里。

- 1 字节存放（量化压缩）
- 用于 BM25 公式里的 `(1 - b + b * fieldLength / avgFieldLength)`
- 可以 `"norms": false` 关闭（用于完全不参与评分的字段，省内存）

注意：`keyword` 字段默认不存 norm；`text` 默认存。

</details>

### Q1.9 ⭐⭐⭐ 数值范围查询（`range`）在倒排索引里怎么走？

<details><summary>答案</summary>

数值 / 时间 / IP 不存在倒排表里——它们走 **BKD tree**（多维 KD 树的磁盘版）。

- 数值字段（long、double）和 date、ip 都用 BKD tree
- 支持精确 + 范围 + 多维（geo_point 也走它）
- 大数据集下范围查询 O(log N + 命中数)

如果一个数值字段同时要走 `term`（精确）+ `range` 查询，Lucene 一般同时存：

- BKD（用于范围）
- doc_values（用于排序 / agg）
- 部分场景也可以加 `index: true` 把数值变成 keyword 进倒排（少用）

</details>

### Q1.10 ⭐ Lucene 一个 segment 物理上有哪些文件？

<details><summary>答案</summary>

```
_0.fdt    field data（stored fields）
_0.fdx    field data 索引
_0.fnm    field info
_0.tip    term index（FST）
_0.tim    term dictionary
_0.doc    posting list（docId + freq）
_0.pos    positions
_0.pay    payloads / offsets
_0.dvd    doc_values data
_0.dvm    doc_values metadata
_0.nvd    norms data
_0.nvm    norms metadata
_0.kdd    BKD data
_0.kdi    BKD index
_0.kdm    BKD metadata
_0.cfs/.cfe  复合文件（小段合并存）
_0.si     segment info
write.lock
```

熟悉这些扩展名在排查 `Too many open files` / 磁盘碎片时很有用。

</details>

---

## E02 Segment 与合并

### Q2.1 ⭐ refresh、flush、commit 三者区别？

<details><summary>答案</summary>

| 操作 | 频率 | 持久 | 可搜索 |
|---|---|---|---|
| **refresh** | 默认 1s | ❌（segment 在 page cache） | ✅ |
| **flush** | translog 达 10GB（上限磁盘 1%，下限 10MB） | ✅（fsync segment + 清 translog） | ✅ |
| **commit** | Lucene 概念，等同 ES flush | ✅ | ✅ |

关键：**refresh 让数据可搜，flush 让数据持久**。崩溃后丢的是 `flush 之后 + translog` 之外的数据。

</details>

### Q2.2 ⭐⭐ 为什么 ES "近实时"而不是实时？

<details><summary>答案</summary>

写入到 indexing buffer → refresh（默认 1s）→ 新 segment 才可见。

这 1 秒延迟是吞吐和实时性的折衷：

- 缩到 0 → 每次写都开新 segment → segment 爆炸、merge 跟不上
- 拉到 30s → 写入吞吐更高，但延迟容忍要 30s

可以通过 `?refresh=true` 强制 refresh 但**严重影响吞吐**——只对调试或低频写场景用。

</details>

### Q2.3 ⭐⭐ `_forcemerge?max_num_segments=1` 慎用的原因？

<details><summary>答案</summary>

1. **IO 重**：把所有 segment 读出 + 写回，磁盘和网络压力大
2. **同步操作**：在 segment merge 完成前 API 一直阻塞
3. **段太大后果**：合后 1 个超大段（几十 GB），page cache 命中下降、热数据 cache evict 增多
4. **不可逆**：合完无法分裂

**只对只读索引做**。对活跃写入索引 force_merge 等于白干（新数据会立刻产生新段）。

</details>

### Q2.4 ⭐⭐ Tiered Merge Policy 的核心思想？

<details><summary>答案</summary>

Lucene 默认合并策略。**把段按大小分层**：每层有 N 个段，超过就触发合并到下一层。

- `index.merge.policy.segments_per_tier`：默认 10
- `index.merge.policy.max_merge_at_once`：默认 10
- `index.merge.policy.max_merged_segment`：默认 5GB

目标：**避免合并代价剧增**（如果总是合很小的段或合很大的段，性能都不好）。

类似 LSM 数据库的 size-tiered compaction。

</details>

### Q2.5 ⭐⭐⭐ 删除一个文档的物理过程？

<details><summary>答案</summary>

1. 找到文档在哪个 segment（按 _id 路由）
2. 在该 segment 的 **`.liveDocs`**（位图）里把对应 bit 置 0
3. translog 记录删除操作
4. refresh 后查询自动跳过 liveDocs=0 的 doc
5. **物理删除要等 merge**：merge 时丢弃 liveDocs=0 的 doc，新 segment 才真的没有它

代价：

- 删多了 → liveDocs 位图大 → 查询要做"是否活着"的额外判断
- 物理空间不释放（直到 merge）
- 大量 delete 后 forcemerge 才能真的回收

</details>

### Q2.6 ⭐⭐ 更新一个文档（PUT _doc/{id}）的物理过程？

<details><summary>答案</summary>

**ES 没有真正的"原地更新"**。一次 update 实际：

1. 找旧文档 → 标记删除（同删除流程）
2. 写新文档 → 进 indexing buffer
3. refresh 后两个版本并存（旧的不可见）
4. merge 后老版本物理消失

所以"高频更新"工作集对 ES 来说很贵（每次产生标删 + 新写）。

</details>

### Q2.7 ⭐⭐ `_version` / `_seq_no` / `_primary_term` 的作用？

<details><summary>答案</summary>

ES 内部的乐观并发控制：

- 每次写自增 seq_no
- primary_term 在主分片切换时递增
- 客户端可以 `?if_seq_no=10&if_primary_term=2` 强制乐观锁
- ES 7+ 推荐用 seq_no / primary_term 不要直接用 _version

防 read-modify-write 竞态：客户端读→改→写，期间别人改过 → seq_no 不匹配 → 409 Conflict → 重试。

</details>

### Q2.8 ⭐⭐⭐ `best_compression` codec 与默认 codec 的差异？

<details><summary>答案</summary>

| codec | 算法 | 写吞吐 | 压缩比 | 解压速度 |
|---|---|---|---|---|
| `default` | LZ4 | 高 | 1× | 极快 |
| `best_compression` | DEFLATE | 中 | 2-4× 节省 | 慢 ~30% |
| ES 8.14 起 stored fields 默认使用 ZSTD | ZSTD | 中-高 | 类似 best_compression | 比 DEFLATE 快 |

**适用**：

- 默认 LZ4：热数据、查询频繁
- best_compression：日志、warm/cold 数据（磁盘成本主导）

切 codec 是**只在新 segment 生效**，老段要 force_merge 才会重写。

</details>

### Q2.9 ⭐⭐ translog 与 fsync 策略

<details><summary>答案</summary>

translog 是 WAL（Write-Ahead Log），ES 写入路径：

```
请求 → append translog → 更 in-memory buffer → ack
              ↓ fsync 策略
       request: 每请求 fsync（默认）
       async: 每 sync_interval (5s) fsync
```

`index.translog.durability`：

- `request`（默认）：丢失上限 = 单请求（最安全但慢）
- `async`：可能丢最近 sync_interval 内的数据（更快）

崩溃恢复时 replay translog 到 in-memory buffer，refresh / flush 后生效。

</details>

### Q2.10 ⭐ segment count 多少算多？

<details><summary>答案</summary>

经验阈值：**单 shard 50 segment 以下健康**。超过 100 要警觉。

太多原因：

- 写入速度 > merge 速度（IO / CPU 瓶颈）
- merge 被 throttle 卡（`indices.store.throttle.type=none` 可放开）
- 刚做完大量小批量写入

修复：

```bash
POST myidx/_forcemerge?max_num_segments=5
# 注意只对相对稳定的索引做
```

</details>

---

## E03 集群拓扑

### Q3.1 ⭐ master / data / coordinating 三种节点的职责？

<details><summary>答案</summary>

| 角色 | 职责 |
|---|---|
| **master** | 维护 cluster state（mapping、shard 分配、节点存活）；不存数据 |
| **data** | 存储 shard，处理读写 IO |
| **coordinating** | 接客户端请求 → 路由 → 聚合 fan-out 结果。每个节点天然都是 coord（只是程度不同） |
| ingest | ingest pipeline 预处理 |
| ml | ML 任务 |

一个节点可以是多角色（小集群），生产推荐**至少专门 3 个 master**。

</details>

### Q3.2 ⭐⭐ master quorum 是什么？为什么至少 3 个 master？

<details><summary>答案</summary>

ES 用 **Raft-like 协议**（叫 cluster coordination subsystem，ES 7 重写）选主。

quorum = `master_eligible / 2 + 1`：

- 1 个 master：单点，挂了集群挂
- 2 个 master：quorum 是 2，挂任一就选不出 → 双脑风险
- **3 个 master**：quorum 2，挂 1 还能选主
- 5 个 master：quorum 3，挂 2 还能选主

**永远奇数个 master，最少 3 个**。

</details>

### Q3.3 ⭐⭐⭐ Split brain（脑裂）问题怎么解决？

<details><summary>答案</summary>

旧版 ES（< 7.0）靠 `discovery.zen.minimum_master_nodes` 防脑裂——必须配等于 quorum 的值，配错就裂。

ES 7+ 完全重写了集群协调（cluster coordination subsystem），自动管理 quorum，**不再手动配 minimum_master_nodes**。

核心机制：

- 第一次启动需要 `cluster.initial_master_nodes` 引导（之后摘掉）
- voting configuration 动态调整，master 节点加入退出时自动加减
- 没法形成 quorum 时拒绝服务（read-only），不会脑裂

</details>

### Q3.4 ⭐⭐ Hot / Warm / Cold / Frozen 节点用什么区分？

<details><summary>答案</summary>

`node.attr.data: hot|warm|cold|frozen` 标签，ES 7.10+ 改用专门角色：

```yaml
node.roles: [data_hot]
node.roles: [data_warm]
node.roles: [data_cold]
node.roles: [data_frozen]
```

索引侧通过 `index.routing.allocation.include._tier_preference` 指定。

- **hot**：NVMe SSD，写入 + 近期查询
- **warm**：SSD，近期查询不写
- **cold**：HDD，少查询
- **frozen**：对象存储 + 本地 cache（searchable snapshot）

</details>

### Q3.5 ⭐⭐ coordinating-only 节点的两个典型场景？

<details><summary>答案</summary>

1. **大聚合 / 复杂查询**：聚合归并阶段在 coord 进行，吃 heap+CPU。专 coord 隔离避免影响 data 节点。
2. **Kibana / 大量客户端长连接入口**：作为负载均衡层，长连接和 SSL 终止集中处理。

配置：

```yaml
node.roles: []   # 既不是 master 也不是 data 也不是 ingest
```

heap 给 16-30GB。

</details>

### Q3.6 ⭐⭐ master 节点要多少 heap？

<details><summary>答案</summary>

4-8GB 即可。master 主要存 cluster state（mapping、shard 分配表），通常几十 MB 到几百 MB。除非有几万索引/几十万 shard 才需要更大。

但**集群规模大（shard 数 > 5w）时 cluster state 几 GB 是可能的**，要监控 cluster state size：

```bash
GET _cluster/state?filter_path=metadata,routing_table | wc -c
```

</details>

### Q3.7 ⭐⭐ allocation awareness 是什么？

<details><summary>答案</summary>

让 ES 在分配 shard 时**考虑节点的物理位置**，避免一个 rack/zone 挂了同时丢主副本。

```yaml
# 每个节点
node.attr.zone: us-east-1a   # 或 1b/1c

# 集群
cluster.routing.allocation.awareness.attributes: zone
cluster.routing.allocation.awareness.force.zone.values: us-east-1a,us-east-1b,us-east-1c
```

效果：每个分片的主和副本必须分布在不同的 zone。

forced awareness：即使少了某个 zone，也不在剩余 zone 上挤压副本（避免单 zone 故障后全部副本压在剩余 zone）。

</details>

### Q3.8 ⭐ ingest 节点和 ingest pipeline 关系？

<details><summary>答案</summary>

ingest pipeline 是写入前的预处理链（grok、convert、enrich、user_agent 等 processor）。

- ingest 节点是有 `node.roles: [ingest]` 角色的节点
- 默认所有节点都能做 ingest（最小角色集 `master, data, ingest, ...`）
- 大集群专门拆出 ingest 节点：CPU 强、不存数据
- 通过 `index.default_pipeline` 让索引自动经过 pipeline

</details>

### Q3.9 ⭐⭐⭐ cluster state 太大的危害？

<details><summary>答案</summary>

cluster state 在 master 内存里，**每次变更要全量广播到所有节点**。

cluster state 太大（> 几百 MB）：

- master GC 抖动
- 节点 join 慢
- 改任何 mapping / index 都让全集群短暂卡顿
- pending_tasks 堆积

主因：

- 索引太多（几万个）
- mapping 太大（几千字段 / 索引）
- dynamic mapping 不控制爆字段

修复：

- 合并小索引（rollover）
- 限制字段数：`index.mapping.total_fields.limit: 1000`
- 关 dynamic mapping：`"dynamic": "strict"`

</details>

### Q3.10 ⭐⭐ 一个节点 down 后多久 shard 才重新分配？

<details><summary>答案</summary>

默认 **1 分钟延迟**（`index.unassigned.node_left.delayed_timeout`）。

设计：避免短暂网络抖动 / 节点重启就立刻搬数据（数据搬运极重）。

```bash
# 滚动重启时建议先延长
PUT _all/_settings
{"settings": {"index.unassigned.node_left.delayed_timeout": "30m"}}
# 完事改回
PUT _all/_settings
{"settings": {"index.unassigned.node_left.delayed_timeout": "1m"}}
```

</details>

---

## E04 分片与路由

### Q4.1 ⭐ 主分片 vs 副本分片的区别？

<details><summary>答案</summary>

| 维度 | primary | replica |
|---|---|---|
| 写入 | 接收原始写入 | 同步 primary 的写 |
| 读 | 可读 | 可读 |
| 数量 | 索引创建时定，**不可改**（除非 split / reindex） | `_settings` 随时改 |
| 故障容忍 | primary 丢 → replica 升级 | replica 丢 → 重建 |

读写都不强制走 primary，replica 也能服务读（吞吐线性扩展）。

</details>

### Q4.2 ⭐⭐ 默认路由公式？

<details><summary>答案</summary>

`shard = hash(_id) % number_of_primary_shards`

- hash 是 Murmur3
- **primary 分片数不能改**就是因为这个公式——改了所有文档要重路由 → reindex
- 自定义 routing：`POST idx/_doc?routing=user_42`，让指定 routing 值的所有文档落同一 shard（用于热点查询场景）

</details>

### Q4.3 ⭐⭐ shard 数怎么选？

<details><summary>答案</summary>

核心约束：

- 单 shard 大小 **10-50 GB 是甜区**
- 单节点 shard 数 < 600（每 GB heap < 20）
- shard 数 < 数据节点数 → 浪费
- shard 数 >> 数据节点数 → 元数据开销 + 大查询 fan-out

经验：

- 小数据（< 50GB）：1 个 shard 就够
- 中等（50GB-1TB）：3-10 个 shard
- 大（> 1TB）：按时间 rollover 拆多个索引，每索引 5-10 shard

时间序列推荐：**按天/周 rollover + 单分片 30-50 GB**。

</details>

### Q4.4 ⭐⭐ shrink 与 split API 的区别？

<details><summary>答案</summary>

| API | 方向 | 约束 |
|---|---|---|
| `shrink` | 减少 shard 数 | 目标数必须整除当前；前置：所有 shard 同节点 + 索引 read-only |
| `split` | 增加 shard 数 | 目标数必须是当前的整数倍；前置：索引 read-only |

两者都是 **hard link + 修改 mapping**（不复制数据），秒级完成。

唯一情况两者都不行：要把 shard 数变成"奇怪"的（不整除/不倍数）—— 必须 reindex。

</details>

### Q4.5 ⭐⭐⭐ 路由热点（hot shard）的原因和解决？

<details><summary>答案</summary>

**原因**：

- 自定义 routing 选错（如所有最热 user 都路到一个 shard）
- 时间序列 _id 单调递增 + 默认 routing 落同一 shard（很罕见，Murmur3 已经很均匀）

**症状**：单 shard CPU/IO 100%，其他 shard 闲

**修复**：

- 改 routing 策略（更均匀 hash）
- 用 `routing.allocation.total_shards_per_node` 强制每节点 shard 数上限
- partitioned routing（`index.routing_partition_size`）让同 routing 落 N 个 shard 子集
- 长期：重新设计 mapping / 拆索引

</details>

### Q4.6 ⭐⭐ 一个搜索如何分发到 shards？

<details><summary>答案</summary>

默认 **query then fetch** 两阶段：

1. **Query phase**：coord 选 N 个 shard（primary 或 replica，按 adaptive replica selection 算），并行发查询。每 shard 返回 top K 的 doc_id + score。
2. **Coord merge**：归并所有 shard 的 doc_id + score，全局取 top K。
3. **Fetch phase**：coord 按归并后的 doc_id 列表去对应 shard 取 _source。

为什么不一次性 fetch？— 每 shard 都返回大文档会爆带宽。先选出 top K 再 fetch K 个文档省 N 倍流量。

</details>

### Q4.7 ⭐⭐ adaptive replica selection 是什么？

<details><summary>答案</summary>

ES 6.1+ 引入。coord 给 shard 发查询时不固定 RR，而是**动态选最快的副本**：

- 综合考虑：副本节点的负载、过去 RTT、past errors、search queue depth
- 自动避开慢节点 / 高负载节点

默认开启。极少需要关。

</details>

### Q4.8 ⭐⭐⭐ Cross-Cluster Replication (CCR) 跟 reindex 远程的区别？

<details><summary>答案</summary>

| 维度 | CCR | reindex remote |
|---|---|---|
| 实时性 | 准实时（基于 changes API） | 一次性快照 |
| 方向 | 单向（leader → follower） | 单向 pull |
| 协议 | ES 内部 transport | HTTP |
| 设置 | 需要 Platinum 许可 | 免费 |
| 用途 | 灾备、跨地域、读分流 | 数据迁移 |

CCR 类似数据库的物理复制；reindex 是逻辑导出导入。

</details>

### Q4.9 ⭐⭐ shard rebalance 触发条件？

<details><summary>答案</summary>

ES 看几个维度：

- 每节点 shard 数（`cluster.routing.allocation.balance.shard`）
- 每节点 disk 使用率（`cluster.routing.allocation.balance.disk_usage`，8.6+）
- 每索引每节点 shard 数（`cluster.routing.allocation.balance.index`）

触发条件：

- 新节点加入
- 节点离开
- 索引创建 / 删除
- 集群参数变更
- 手动 reroute API

**rebalance 会搬数据**，搬过程 IO + 网络重。生产高负载时段建议禁：

```bash
PUT _cluster/settings
{"transient":{"cluster.routing.rebalance.enable":"none"}}
```

</details>

### Q4.10 ⭐ 副本数怎么选？

<details><summary>答案</summary>

- **0**：能容忍单节点故障丢数据 → 不推荐生产
- **1**：默认，单节点故障不丢数据 → 主流
- **2**：高可用 + 读 QPS 高 → 在乎容忍 2 节点同时挂
- **≥ 3**：极特殊（写吞吐很低 + 读极高）

代价：写吞吐 ≈ `1 / (1 + replica)`，磁盘 = `(1 + replica) × 原始大小`。

</details>

---

## E05 Mapping 与 Analyzer

### Q5.1 ⭐ dynamic mapping 是什么？为什么生产要关？

<details><summary>答案</summary>

新字段第一次出现时 ES 自动推断类型加 mapping。

**生产坑**：

- 字段爆炸：日志的 `id`、`uuid` 全成 keyword → mapping 万字段
- 类型错猜：第一次出现是数字第二次出现是字符串 → 后续写失败
- cluster state 膨胀

生产推荐 `"dynamic": "strict"` 或 `"dynamic": "false"`：

- `strict`：新字段 → 拒绝写入
- `false`：新字段 → 不索引但存到 _source

</details>

### Q5.2 ⭐⭐ runtime fields 解决什么问题？

<details><summary>答案</summary>

Schema-on-read。运行时定义虚拟字段（Painless 脚本），不重新索引：

```json
PUT logs/_mapping
{
  "runtime": {
    "duration_ms": {
      "type":"long",
      "script": {"source": "emit(doc['end'].value.millis - doc['start'].value.millis)"}
    }
  }
}
```

特点：

- 不占磁盘（计算在查询时）
- 可用于 query/agg/sort
- 慢（每次查询都算一遍）

适合：临时探索、字段 schema 变更测试、低 QPS 字段。

</details>

### Q5.3 ⭐⭐⭐ analyzer 的三层结构？

<details><summary>答案</summary>

```
char filter → tokenizer → token filter
```

- **char filter**：在分词前过滤字符（HTML strip、模式替换）
- **tokenizer**：切词（standard、whitespace、ngram、ik_max_word）— 必有一个
- **token filter**：处理 token（lowercase、stop、stemmer、synonym）— 0 个或多个

例子：standard analyzer = `standard tokenizer + lowercase filter + stop filter`。

</details>

### Q5.4 ⭐⭐ 中文分词为什么不能用 standard？

<details><summary>答案</summary>

standard tokenizer 用 Unicode 文本分割算法：对英文按空格 + 标点切，对中文**一字一 token**：

```
"Elasticsearch 中文分词"
→ ["elasticsearch", "中", "文", "分", "词"]
```

查询 "中文" 会变成 `"中" AND "文"`，匹配大量噪声。

正确做法：用中文分词器（IK / Smartcn / ICU / jieba），按词切分：

```
→ ["elasticsearch", "中文", "分词"]
```

</details>

### Q5.5 ⭐⭐ IK 分词器的 `ik_smart` 和 `ik_max_word` 区别？

<details><summary>答案</summary>

| 模式 | 行为 | 用途 |
|---|---|---|
| `ik_smart` | 最少切分（最长匹配） | 索引 + 查询，**查询时用** |
| `ik_max_word` | 最细粒度（穷举所有可能词） | **索引时用** |

经典配置：

```json
"content": {
  "type":"text",
  "analyzer":"ik_max_word",
  "search_analyzer":"ik_smart"
}
```

让索引时切的 token 多（提高召回），查询时切的 token 少（避免歧义）。

</details>

### Q5.6 ⭐⭐ normalizer 是什么？

<details><summary>答案</summary>

只能用在 `keyword` 字段上的"轻量 analyzer"——**只能用 char filter 和 token filter，不能用 tokenizer**（keyword 不分词）。

用途：
- 大小写归一化
- 去除尾部空格
- 特殊字符替换

```json
"settings": {
  "analysis": {
    "normalizer": {
      "lowercase_norm": {"type":"custom","filter":["lowercase","asciifolding"]}
    }
  }
},
"mappings": {
  "properties": {
    "email": {"type":"keyword","normalizer":"lowercase_norm"}
  }
}
```

让 `Foo@bar.com` 写入 / 查询时都变 `foo@bar.com`。

</details>

### Q5.7 ⭐⭐⭐ 修改 mapping 字段类型为什么不行？

<details><summary>答案</summary>

字段类型决定底层物理存储：

- text → 倒排 + 分词信息
- keyword → 倒排（单 term）+ doc_values
- long → BKD tree + doc_values
- ...

改字段类型 = 改物理结构，已存数据没法兼容。

唯一办法：**新建索引 + reindex**：

```bash
POST _reindex
{
  "source": {"index":"old_idx"},
  "dest": {"index":"new_idx"}
}
```

或借助 alias 做双索引切换，零停机迁移。

</details>

### Q5.8 ⭐⭐ multi-fields 是什么？

<details><summary>答案</summary>

一个字段以多种方式索引：

```json
"name": {
  "type":"text",
  "analyzer":"ik_max_word",
  "fields": {
    "keyword": {"type":"keyword","ignore_above":256},
    "english": {"type":"text","analyzer":"english"},
    "pinyin": {"type":"text","analyzer":"pinyin"}
  }
}
```

存一次，三种索引：
- `name`：中文搜（IK）
- `name.keyword`：精确 / agg
- `name.english`：英文 stemming
- `name.pinyin`：拼音搜

代价：磁盘 N 倍，但读写 API 不变。

</details>

### Q5.9 ⭐⭐ ignore_above 是什么？

<details><summary>答案</summary>

`keyword` 字段属性。超过这个长度的值**不索引**（但还在 _source 里）：

```json
"url": {"type":"keyword", "ignore_above":2048}
```

URL 字符串可能很长，超过 2KB 的不参与倒排，避免 term dict 爆炸。

注意：**值还在 source 里能 fetch 出来**，但不能 term 查询。

</details>

### Q5.10 ⭐⭐ index_options 各级别区别？

<details><summary>答案</summary>

控制倒排里存什么：

| 级别 | 存内容 | 用途 |
|---|---|---|
| `docs` | 只 docId | 仅出现判断 |
| `freqs` | docId + tf | BM25 评分（但不能 phrase） |
| `positions`（默认 text） | docId + tf + position | phrase 查询 |
| `offsets` | + offset | highlight |

精打细算：

- `keyword` 默认 docs（够用）
- `text` 默认 positions（支持 phrase）
- 要 highlight 加 offsets（多磁盘）

</details>

---

## E06 Query DSL

### Q6.1 ⭐ filter context 和 query context 区别？

<details><summary>答案</summary>

| 维度 | filter | query |
|---|---|---|
| 评分 | ❌ 不算 | ✅ 算 BM25 |
| 缓存 | ✅ 可缓存 | ❌ 不缓存 |
| 用途 | 是/否判断 | 相关度排序 |

**经验**：能用 filter 就用 filter，速度快、能缓存。`bool.filter` 内的 term/range 都是 filter context。

</details>

### Q6.2 ⭐⭐ `term` 和 `match` 区别？

<details><summary>答案</summary>

| 维度 | term | match |
|---|---|---|
| 分词 | ❌ 不分词 | ✅ 经 analyzer |
| 适用字段 | keyword、数值、ip | text |
| 评分 | constant_score | BM25 |

陷阱：对 text 字段用 term 查询 → **查的是分词后的某个 token**：

```json
// 文档 name: "Elasticsearch"，经 analyzer 后变 "elasticsearch"
{"term":{"name":"Elasticsearch"}}   // ❌ 不命中（大小写）
{"term":{"name":"elasticsearch"}}   // ✅ 命中
{"match":{"name":"Elasticsearch"}}  // ✅ 命中（match 也走 analyzer）
```

</details>

### Q6.3 ⭐⭐ `bool` 查询的 must / should / filter / must_not 评分行为？

<details><summary>答案</summary>

| 子句 | 必须命中 | 参与评分 | 缓存 |
|---|---|---|---|
| must | ✅ | ✅ | ❌ |
| should | minimum_should_match | ✅ | ❌ |
| filter | ✅ | ❌ | ✅ |
| must_not | 必须不命中 | ❌ | ✅ |

**最佳实践**：能扔 filter 的扔 filter，仅排序相关的留 must / should。

</details>

### Q6.4 ⭐⭐⭐ 深度分页（from + size）为什么慢？

<details><summary>答案</summary>

`from: 10000, size: 10` 实际要求 **coord 拿 10010 × shard 数 的 docId** 归并，丢弃前 10000 个。

每个 shard 必须排序 top 10010，**线性增长 + N 倍 shard**。

**正确分页**：

1. **search_after**：基于上一页的 sort 值续传
   ```json
   { "size":10, "sort":[{"timestamp":"asc"},{"_id":"asc"}], "search_after":[1683475200000,"id_99"] }
   ```
2. **scroll**：拿快照游标（适合一次性导出，不适合实时）
3. **PIT + search_after**：稳定的快照 + 续传

</details>

### Q6.5 ⭐⭐ multi_match 的几种 type？

<details><summary>答案</summary>

| type | 行为 |
|---|---|
| `best_fields`（默认） | 取所有字段中最高分（适合搜短描述） |
| `most_fields` | 累加所有字段分（多字段都命中加分） |
| `cross_fields` | 把多字段当一个大字段（人名 + 邮箱 + ID 这种）|
| `phrase` | 各字段做 phrase |
| `phrase_prefix` | 前缀 phrase |

```json
{"multi_match": {"query": "redis cluster","fields": ["title^3","content"],"type":"best_fields"}}
```

</details>

### Q6.6 ⭐⭐ aggs 的 cardinality 准吗？

<details><summary>答案</summary>

**不准，是 HLL 近似**。

参数：
- `precision_threshold`（默认 3000）：低于这值近似精确，超过会有 ~1% 误差
- 上限：40000（极限准确但内存高）

```json
{"aggs": {"u": {"cardinality": {"field": "user_id", "precision_threshold": 40000}}}}
```

精确 distinct 要靠 `terms` agg + size（但桶爆）或 `terms_set` 过滤。生产 99% 场景 1% 误差能接受。

</details>

### Q6.7 ⭐⭐⭐ composite agg vs terms agg

<details><summary>答案</summary>

| 维度 | terms | composite |
|---|---|---|
| 返回数量 | size 限定（默认 10） | 一次 size，可分页 |
| 桶上限 | search.max_buckets | 同上但能续传 |
| 多维 | sub-agg 嵌套 | 多 source 笛卡尔 |
| 续页 | ❌ | ✅ after_key |
| 典型用途 | 仪表盘 top N | 数据导出 |

**关键**：要把"所有桶"拉出来，必须用 composite + after_key。

</details>

### Q6.8 ⭐ highlight 的 type 有哪些？

<details><summary>答案</summary>

- `unified`（默认 ES 6+）：通用，平衡速度和准确
- `plain`：经典，慢，准确度高
- `fvh`（fast vector highlighter）：极快，但要求字段有 `term_vector: with_positions_offsets`

```json
"highlight": {
  "fields": {"content": {"type":"unified","number_of_fragments":3}}
}
```

</details>

### Q6.9 ⭐⭐⭐ ESQL 与 _sql 的区别？

<details><summary>答案</summary>

| 维度 | _sql | ESQL |
|---|---|---|
| 语法 | 标准 SQL | 管道式（类 SPL） |
| 引擎 | 解释执行 | 编译 + 向量化 |
| Join | 有限（lookup） | ES|QL LOOKUP JOIN：8.18/9.0 预览，8.19/9.1 GA |
| 性能 | 一般 | 比 _sql 快很多 |
| ES 9 推荐 | 不再积极发展 | 主推 |

ESQL 示例：

```
FROM logs-*
| WHERE level == "error"
| STATS count = COUNT(*) BY service
| SORT count DESC
| LIMIT 10
```

</details>

### Q6.10 ⭐⭐ suggester 三种 type？

<details><summary>答案</summary>

| type | 用途 |
|---|---|
| `term` | 单 token 拼写校正 |
| `phrase` | 短语级别拼写校正 |
| `completion` | 自动补全（搜索框 typeahead） |

`completion` 用 FST 极快但**要单独的 mapping 类型**：

```json
"suggest": {"type":"completion"}
// 写入: {"suggest":{"input":["redis","cache"], "weight":10}}
// 查询: 输入 "re" 返回 "redis"
```

</details>

---

## E07 BM25 与 Reranking

### Q7.1 ⭐ BM25 公式核心三个因子？

<details><summary>答案</summary>

```
score = IDF(term) × TF_norm × FieldNorm
```

- **IDF**：`log((N - n + 0.5) / (n + 0.5) + 1)`。term 出现的文档越少 IDF 越大。
- **TF_norm**：`tf × (k1 + 1) / (tf + k1 × (1 - b + b × |D| / avgdl))`。tf 饱和 + 长度归一。
- **fieldNorm**：1 字节量化的长度归一（norm 字段）。

参数 `k1`（默认 1.2）控制 TF 饱和速度，`b`（默认 0.75）控制长度归一强度。

</details>

### Q7.2 ⭐⭐ BM25 比 TF-IDF 好在哪？

<details><summary>答案</summary>

1. **TF 饱和**：TF-IDF 的 tf 线性增长 → 长文档命中 N 次比短文档命中 1 次评分高 100×。BM25 用 `tf × (k1+1) / (tf + k1)`，tf=10 和 tf=100 评分差很小。
2. **长度归一更细**：BM25 的 b 参数能调节"长文档惩罚"程度，TF-IDF 用粗暴的开方归一。
3. **概率模型基础**：从 Robertson 的 BIM 模型推出，理论更严谨。

</details>

### Q7.3 ⭐⭐⭐ explain API 输出什么？

<details><summary>答案</summary>

`POST idx/_explain/{id} {"query":...}` 返回某文档对某 query 的**逐子句评分树**：

```json
{
  "matched": true,
  "explanation": {
    "value": 1.234,
    "description": "sum of:",
    "details": [
      {"value":0.5,"description":"weight(name:redis in 12) [PerFieldSimilarity]","details":[...]},
      {"value":0.7,"description":"weight(name:cluster in 12) [PerFieldSimilarity]","details":[...]}
    ]
  }
}
```

详细到每个 term 的 IDF / TF / fieldNorm 数值，是调评分的核心工具。

</details>

### Q7.4 ⭐⭐ function_score 与 rank_feature 的区别？

<details><summary>答案</summary>

| 维度 | function_score | rank_feature |
|---|---|---|
| 工作方式 | 包裹 query，对每个命中算 function | 字段类型，写入时记录 boost 信号 |
| 灵活性 | 高（可写 script） | 低（固定值/log/saturation） |
| 性能 | 慢（每命中算 script） | 快（rank_feature 类型加速 retrieval） |
| 用法 | 业务复杂信号 | 文档级 boost（pagerank、popularity） |

```json
// rank_feature
"mappings": {"properties": {"popularity":{"type":"rank_feature"}}}
// 查询
{"rank_feature":{"field":"popularity","saturation":{"pivot":10}}}
```

</details>

### Q7.5 ⭐⭐⭐ multi-stage 排序

<details><summary>答案</summary>

工业级搜索通常 3 阶段：

1. **Retrieval（粗排）**：BM25 / kNN 召回 top 1000
2. **First-pass ranking**：function_score / rank_feature 加业务信号，top 100
3. **Reranking（精排）**：cross-encoder / LTR 模型对 100 个重新评分，最终 top 10

```json
POST idx/_search
{
  "size": 10,
  "query": { /* BM25 retrieval */ },
  "rescore": [
    {"window_size":100,"query":{"rescore_query":{ /* 精排 */ },"query_weight":0.3,"rescore_query_weight":0.7}}
  ]
}
```

</details>

### Q7.6 ⭐⭐ LTR（Learning to Rank）的训练过程？

<details><summary>答案</summary>

1. **采集训练数据**：搜索日志 → query / doc / 用户行为（点击 / 购买 / 停留时长）
2. **构特征**：每对 (query, doc) 提取特征（BM25 分、向量相似度、点击率、CTR、人工 boost）
3. **训练模型**：LambdaMART / XGBoost / LightGBM（rank-aware loss）
4. **导入 ES**：用 LTR 插件或 inference processor
5. **rescore 阶段调用模型**

挑战：

- 标注数据贵
- 训练 / 在线特征对齐
- 模型更新频率

</details>

### Q7.7 ⭐⭐ cross-encoder 比 bi-encoder 好在哪？

<details><summary>答案</summary>

| 维度 | bi-encoder（向量） | cross-encoder |
|---|---|---|
| 编码方式 | query 和 doc 分别编码成向量 | (query, doc) 一起送进 BERT |
| 召回 | 适合（向量库 ANN） | ❌ 太慢 |
| 精排 | 一般 | ✅ 业界 SOTA |
| 速度 | 极快 | 慢（每对推理一次） |

实战：bi-encoder 召回 top 100 → cross-encoder rerank top 10。

</details>

### Q7.8 ⭐⭐⭐ BM25 的 b 参数对什么场景敏感？

<details><summary>答案</summary>

`b` 控制长度归一强度（默认 0.75）：

- `b=0`：完全忽略文档长度
- `b=1`：完全用长度归一

调小 b（如 0.3）适合：

- 长文档（文章、论文）：不要因为长就过分降权
- 标题字段：长度差别不大，无需强归一

调大 b 适合：

- 短描述（商品名）：明显的长度优势应该被惩罚

</details>

### Q7.9 ⭐⭐ 评测搜索质量的指标？

<details><summary>答案</summary>

| 指标 | 含义 |
|---|---|
| **MRR**（Mean Reciprocal Rank） | 第一个相关结果的位置倒数的平均 |
| **NDCG@k** | 排序质量，考虑位置折扣 |
| **MAP**（Mean Average Precision） | 多关键词相关的平均精度 |
| **Precision@k** | top-k 中相关的比例 |
| **Recall@k** | top-k 召回率（要标全集相关） |

ES 提供 **rank_eval API** 计算 MRR / NDCG。

</details>

### Q7.10 ⭐⭐⭐ explain 显示分数 1.0 表示什么？

<details><summary>答案</summary>

- `1.0` 多半是 **constant_score** 或 **filter context**（不算 BM25 分）
- 真 BM25 分通常是浮点小数（如 1.234）

如果你发现 `match` 查询出 1.0，可能：

- 字段 norms 关了（`"norms": false`）→ 长度归一缺失
- 用了 `bool.filter` 包裹（filter 不评分）
- 用了 `constant_score`

</details>

---

## E08 向量检索

### Q8.1 ⭐ 余弦相似度 vs 点积 vs L2

<details><summary>答案</summary>

| 度量 | 公式 | 何时用 |
|---|---|---|
| cosine | `A·B / (|A||B|)` | 关心方向，不关心模长 |
| dot_product | `A·B` | 模长有信息（如 ESLER），且向量都已归一化 |
| L2 | `sqrt(Σ(A-B)²)` | 几何距离场景（图像、地理） |

实战：embedding 模型通常输出**已归一化向量**，cosine 和 dot 结果一样，但 dot 计算快（无除法）。

</details>

### Q8.2 ⭐⭐ HNSW 是什么？

<details><summary>答案</summary>

**Hierarchical Navigable Small World**。多层图近似最近邻索引：

- 节点是向量，边连"距离近"的节点
- 多层：上层稀疏（用于粗导航），下层密（精搜）
- 查询从顶层入口 greedy 走到接近 query 的位置，逐层下降到底层

参数：

- `m`：每节点最大连接数（默认 16）。大 → 精度高 / 内存大
- `ef_construction`：构图时候选邻居数（默认 100）。大 → 构图慢 / 质量高
- `num_candidates`（查询时）：候选池大小。大 → 召回高 / 查询慢

</details>

### Q8.3 ⭐⭐⭐ dense_vector 的 element_type 三种？

<details><summary>答案</summary>

| element_type | 位宽 | 内存 | 精度 |
|---|---|---|---|
| `float` (默认) | 32-bit | 4 bytes × dims | 高 |
| `byte` | 8-bit (-128~127) | 1 byte × dims | 中（要归一化处理） |
| `bit` | 1-bit | dims/8 bytes | 低（仅特定模型 BBQ） |

768 维向量比较：
- float: 3072 bytes
- byte (int8): 768 bytes（节省 4×）
- bit: 96 bytes（节省 32×）

适用：
- float：高精度（小数据集 / 关键场景）
- byte：99% 生产场景（int8 量化精度损失通常 < 2%）
- bit：超大规模 + 模型支持二值化（如 BBQ）

</details>

### Q8.4 ⭐⭐ knn 查询的 num_candidates 怎么选？

<details><summary>答案</summary>

`num_candidates` 是 HNSW 搜索的候选池大小。返回 k 个结果，但内部检查 num_candidates 个。

经验：

- `num_candidates = max(100, k × 10)`
- 召回不够 → 调大（同步影响延迟）
- 延迟敏感 → 调小

监控：开 `profile: true` 看 knn 阶段实际访问的节点数。

</details>

### Q8.5 ⭐⭐⭐ Hybrid Search 与 RRF 融合

<details><summary>答案</summary>

混合 BM25 + 向量结果。**RRF（Reciprocal Rank Fusion）**：

```
score(doc) = Σ 1 / (k + rank_i(doc))
```

ES 8.8+ 内置 RRF：

```json
POST idx/_search
{
  "size": 10,
  "retriever": {
    "rrf": {
      "retrievers": [
        {"standard":{"query":{"match":{"text":"foo bar"}}}},
        {"knn":{"field":"embedding","query_vector":[...],"k":50,"num_candidates":500}}
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  }
}
```

为什么不直接加权 BM25 + cosine？两个分数量级不同（BM25 几到几十、cosine 0-1），加权很难调。RRF 用 rank 而非 score，量级无关。

</details>

### Q8.6 ⭐⭐ 向量索引 vs 倒排索引的存储差异？

<details><summary>答案</summary>

| 维度 | 倒排 | HNSW |
|---|---|---|
| 主结构 | term → posting | 多层图 |
| 文件 | term dict + posting | hnsw graph + raw vectors |
| 内存 | 部分到内存（FST） | **整图常驻内存** |
| 构建 | 写时构 | 写时构 |
| 更新 | 段合并 | 段合并（不太友好，要重 link） |

**核心痛点**：HNSW 整图常驻内存。10M 768d float 向量 = 30GB heap，要 int8 量化才能省 4×。

</details>

### Q8.7 ⭐⭐⭐ ELSER 是什么？

<details><summary>答案</summary>

**Elastic Learned Sparse EncodeR**。Elastic 训练的稀疏向量模型：

- 输入文本 → 输出稀疏向量（实际是 term-weight 映射）
- 不依赖外部 LLM，ES 内 inference
- 不需要训练自己的模型
- 召回质量在多数英文场景接近 dense + reranker

字段类型 `sparse_vector`，查询 `text_expansion` 或 ES 8.16 后的 `semantic_query`。

劣势：

- 英文优化好，中文一般（要 ELSER v2 + 中文模型支持，仍在演进）
- 模型固定，定制化空间小

</details>

### Q8.8 ⭐⭐ kNN 查询的 filter 是 pre-filter 还是 post-filter？

<details><summary>答案</summary>

ES 7.16+ 的 kNN 是 **pre-filter**：先用 filter 缩小候选集，再在子集上做 HNSW 搜索。

```json
{
  "knn": {
    "field":"embedding","query_vector":[...],"k":10,"num_candidates":100,
    "filter": {"term":{"category":"books"}}
  }
}
```

关键：filter 过严 → 候选集小 → HNSW 走不出来 → 召回降。

解决：

- 增大 num_candidates
- 用 `boost` 软过滤（不强行排除）
- 重新设计：分类索引（不同 category 分索引）

</details>

### Q8.9 ⭐⭐ RAG 落地的关键工程问题？

<details><summary>答案</summary>

1. **chunking 策略**：固定 200 token、按段落、滑动窗口、按语义边界
2. **embedding 选型**：text-embedding-3、bge-large、cohere、ELSER
3. **检索质量评估**：人工标注 + NDCG / Recall@k
4. **rerank**：cross-encoder 精排
5. **citation**：让 LLM 引用具体段落
6. **更新策略**：增量索引、全量重建
7. **多语言**：中英混合的 embedding
8. **冷启动**：BM25 fallback

</details>

### Q8.10 ⭐⭐⭐ 向量字段一定要 `index: true` 吗？

<details><summary>答案</summary>

不必。`index: false`：

- 字段以 doc_values 形式存（仍能 fetch / script）
- **不构建 HNSW 索引**
- knn 查询不能用，但 brute force script_score 可以

适合：
- 向量字段做"次要数据"，主要不用 ANN
- 大向量集（百万级）但查询频率很低
- 用其他系统做 ANN，ES 只存

```json
"vec": {"type":"dense_vector","dims":768,"index":false}
```

</details>

---

## E09 写入与近实时

### Q9.1 ⭐ bulk 一次多少行最合适？

<details><summary>答案</summary>

经验值 **5-30 MB / batch 或 1000-5000 doc**：

- 太小 → 网络 RTT 占比高，吞吐低
- 太大 → 占用 indexing buffer / pressure 大，单批失败重试代价高

**测**：从 1000 doc 开始翻倍，直到出现 429 或延迟陡升 → 退一档。

</details>

### Q9.2 ⭐⭐ translog durability `request` vs `async` 的取舍？

<details><summary>答案</summary>

| 选项 | 行为 | 丢失上限 | 写吞吐 |
|---|---|---|---|
| `request`（默认） | 每请求 fsync | 0 | 低 |
| `async` | 每 sync_interval（默认 5s）fsync | sync_interval 之内 | 高 ~2-5× |

实战：

- 关键业务（订单、交易）：`request`
- 日志、监控数据：`async`（可丢 5s 数据，节省 IO）

</details>

### Q9.3 ⭐⭐⭐ indexing pressure 触发 429 的本质？

<details><summary>答案</summary>

ES 7.9+ 反压机制。**每个节点正在处理的写**占用内存有上限：

```yaml
indexing_pressure.memory.limit: 10%   # 默认 10% heap
```

请求大小算法：

- coordinator: `单请求大小`
- primary: `单请求大小 × 1`
- replica: `单请求大小 × number_of_replicas`

**所以副本越多，indexing pressure 越大**。10GB 写入 + 3 副本 → coord 内存计算 4 × 10GB → 触发反压。

修复：

- 副本降 1
- 加节点（分摊 pressure）
- 客户端降并发

</details>

### Q9.4 ⭐⭐ refresh_interval 设 -1 后批量导入完要做什么？

<details><summary>答案</summary>

```bash
# 1. 导入前
PUT myidx/_settings
{"index.refresh_interval":"-1","index.number_of_replicas":0}

# 2. 大量 bulk 写入
# ...

# 3. 完成后
POST myidx/_refresh
PUT myidx/_settings
{"index.refresh_interval":"1s","index.number_of_replicas":1}

# 4. 等副本同步
GET _cluster/health/myidx?wait_for_status=green
```

一定要恢复 refresh_interval 和副本——否则新数据永远不可见，且没冗余。

</details>

### Q9.5 ⭐⭐ 写入路径完整时序？

<details><summary>答案</summary>

```
Client → Coordinator
  ↓ routing
Primary 节点：
  - 解析 + mapping 检查
  - append translog
  - 更 in-memory buffer (Lucene IndexWriter)
  - 并行转发给所有 in-sync replica
Replica 节点（并行）：
  - 同样处理
  - ack 给 primary
Primary 收齐所有 ack → 回 Coord → 回 Client
```

ack 后**数据已在 primary 和所有 replica 的 translog 中**，refresh 后可搜索，flush 后落盘。

</details>

### Q9.6 ⭐⭐ 写入慢的诊断顺序？

<details><summary>答案</summary>

按概率：

1. `_nodes/stats/thread_pool` 看 write 队列 / rejected
2. `_nodes/stats/indexing_pressure` 看内存压力
3. `_cluster/pending_tasks` 看 mapping 变更阻塞
4. `_nodes/hot_threads` 看是在 merge / analyzer / GC
5. `iostat` 看磁盘
6. 客户端侧网络

</details>

### Q9.7 ⭐⭐⭐ `index` vs `create` vs `update` 操作的差异？

<details><summary>答案</summary>

| 操作 | _id 存在时 | 用途 |
|---|---|---|
| `index` | 覆盖 | 默认 upsert |
| `create` | 报 409 | 防重复（要求 _id 唯一） |
| `update` | 部分更新 + 脚本 | RMW 模式 |
| `delete` | 标删 | — |
| `update_by_query` | 批量改 | — |
| `delete_by_query` | 批量删 | — |

**`update` 内部 = 读旧 doc + 改 + 重写**（不是真原地）→ 性能比 index 差。

</details>

### Q9.8 ⭐⭐ 大批量导入设副本数 0 的原因？

<details><summary>答案</summary>

副本数 N 时，每次 bulk 写都要 primary + N 个 replica 都 ack 才算完成，吞吐 ≈ `1 / (1 + N)`。

导入阶段先设 0，结束再设回 1（或 2）：

```bash
# 副本数 0：纯 primary 写，最快
PUT idx/_settings {"index.number_of_replicas": 0}
# 导入完
PUT idx/_settings {"index.number_of_replicas": 1}
# replica 重建（恢复）期间集群 status yellow 是正常的
```

代价：导入阶段 primary 单节点故障会丢数据。所以**只对可重做的导入**（如 reindex）用。

</details>

### Q9.9 ⭐⭐ ingest pipeline 与 update 的关系？

<details><summary>答案</summary>

ingest pipeline 在写入前预处理（grok、enrich、user_agent）。

陷阱：`POST _update` 默认**不走 ingest pipeline**——只有 index / bulk 走。要让 update 走 pipeline：

```bash
POST idx/_update/{id}?pipeline=my_pipeline
{"doc":{...}}
```

也可以让索引默认走 pipeline：

```bash
PUT idx/_settings
{"index.default_pipeline": "my_pipeline"}
```

</details>

### Q9.10 ⭐ search 与 indexing 在 thread_pool 上的隔离？

<details><summary>答案</summary>

ES 内部维护独立线程池：

| pool | 用途 | 默认大小 |
|---|---|---|
| `write` | indexing/bulk/delete | cores |
| `search` | 查询 | int((cores * 3) / 2) + 1 |
| `get` | get/mget | cores |
| `analyze` | 分析 | 1 |
| `refresh` | refresh | min(cores/2, 10) |
| `flush` | flush | min(cores/2, 5) |
| `force_merge` | merge | 1 |
| `management` | 集群管理 | 5 |

监控 `GET _cat/thread_pool?v`，看 `queue` 和 `rejected`。

</details>

---

## E10 性能调优

### Q10.1 ⭐ 节点 64GB 物理内存，JVM heap 给多少？

<details><summary>答案</summary>

**30GB**。理由：

1. **32GB 是 Compressed OOPs 上限**——超过 → 64-bit 指针 → 实际可用 heap 反而下降。
2. 留 ~34GB 给 OS page cache（Lucene mmap segment 文件靠它）。
3. 不给 50%（32GB）是为留 2GB 安全 buffer。

`-Xms30g -Xmx30g`，必须等大避免动态扩容。

</details>

### Q10.2 ⭐⭐ 单节点 600 shard 上限的依据？

<details><summary>答案</summary>

经验阈值 **20 shard / GB heap**。

- 30GB heap → ~600 shard
- 每 shard 元数据约 50KB 起步，加上 segment 内存映射的开销，多了直接撑爆 heap

集群级别有硬上限：`cluster.max_shards_per_node`（默认 1000）。

</details>

### Q10.3 ⭐⭐ indexing pressure 与 write 线程池 rejected 的关系？

<details><summary>答案</summary>

两套机制都防过载，但侧重不同：

| 机制 | 检查 | 单位 |
|---|---|---|
| write 线程池 | 队列长度 | 任务数 |
| indexing pressure | 内存占用 | 字节 |

任一触发都返回 429：
- write 线程池：`EsRejectedExecutionException`
- indexing pressure：`EsRejectedExecutionException, reason: indexing_pressure`

监控两者：
```bash
GET _cat/thread_pool/write?v&h=node,active,queue,rejected
GET _nodes/stats/indexing_pressure
```

</details>

### Q10.4 ⭐⭐ text 字段做聚合慢的原因？

<details><summary>答案</summary>

text 字段**默认没有 doc_values**——聚合要走 **fielddata**（运行时把倒排倒装成列存，全部加载进 heap）。

代价：

- 慢（第一次查询要构 fielddata）
- 吃 heap（基数大时几 GB 起）
- 可能触发 fielddata circuit breaker

修复：加 `.keyword` 子字段，agg 走 keyword：

```json
"name": {
  "type":"text",
  "fields":{"keyword":{"type":"keyword","ignore_above":256}}
}
// agg
{"terms":{"field":"name.keyword"}}
```

</details>

### Q10.5 ⭐⭐⭐ profile API 显示 80% 时间在 build_scorer，可能原因？

<details><summary>答案</summary>

build_scorer 是查询执行前的"构造打分器"阶段，慢通常因为：

1. **wildcard / regex / prefix**：term enum 阶段要遍历大量 term，构 ConstantScoreScorer 慢
2. **大量 should**：bool should 子句多，每个都要 build
3. **terms 查询的列表过长**：`terms: {field: [10000 个值]}`
4. **range 在低基数字段**：枚举大量 term

修复：

- 避免 leading wildcard（`*foo`）—— 给 reverse 索引
- 用 keyword 而非 ngram
- terms_set / lookup 替代超长 terms

</details>

### Q10.6 ⭐⭐⭐ 集群 heap 持续 85%，列 3 个根因和验证方法？

<details><summary>答案</summary>

| 根因 | 验证 |
|---|---|
| **shard 过多** | `_cat/shards` 数节点分布；每 GB heap > 20 shard 警觉 |
| **mapping 字段爆** | `_cluster/state/metadata` 大小；某索引几千字段 |
| **大聚合 / fielddata** | `_nodes/stats/breaker` 看 fielddata used；`_cat/fielddata` |
| **field data 加载** | `_nodes/stats/indices/fielddata` 看 memory_size |
| **deep paging / scroll** | `_cat/tasks` 看长 task；scroll context 占内存 |
| **inflight bulk** | indexing pressure 监控 |

</details>

### Q10.7 ⭐⭐ ILM hot → warm 为什么要 shrink + forcemerge？

<details><summary>答案</summary>

- **shrink**：hot 阶段为了写吞吐多分片（如 5），到 warm 阶段不再写入，分片合并到 1 个减少元数据 + 文件句柄
- **forcemerge**：把 segment 合到 1 个，提高查询缓存命中、减少 segment 数

两者都是**只读阶段才能做**——hot 阶段做会有写入冲突。

</details>

### Q10.8 ⭐⭐ parent circuit breaker 触发返回什么 HTTP code？

<details><summary>答案</summary>

**HTTP 429**（Too Many Requests）+ body 写明 `circuit_breaking_exception`：

```json
{
  "error":{
    "type":"circuit_breaking_exception",
    "reason":"[parent] Data too large, data for [...] would be [...]",
    "bytes_wanted":...,"bytes_limit":...,"durability":"TRANSIENT"
  },
  "status":429
}
```

客户端**应该重试**（指数退避）—— 是临时压力，不是永久错误。

</details>

### Q10.9 ⭐⭐ coordinating-only 节点的两个典型场景？

<details><summary>答案</summary>

1. **大聚合归并**：avg/sum/percentile 在 coord 完成最后合并，吃 heap + CPU
2. **Kibana 后端**：长连接 + HTTPS + 查询 fan-out 入口，专门一个 coord 给 Kibana

heap 给 16-30GB。

```yaml
node.roles: []
```

</details>

### Q10.10 ⭐⭐⭐ 跨 region 部署 ES 为什么不能放一个集群？

<details><summary>答案</summary>

1. **心跳 / quorum**：master 心跳超时容忍 ~10s，跨 region RTT 50-100ms 让选主变脆
2. **写入 RTT**：primary→replica 同步要全 RTT，跨 region 写入延迟 100ms+
3. **cluster state 广播**：每次变更全集群广播，跨 region 链路压力大
4. **网络分区**：跨 region 比单 region 网络故障频繁，触发 split-brain 风险

正确做法：**每 region 一个集群 + CCR（Cross-Cluster Replication）异步复制**。

</details>

---

## E11 ES vs OpenSearch

### Q11.1 ⭐ 2021 年 1 月 Elastic 改许可证的动机是什么？

<details><summary>答案</summary>

阻止云厂商（特别是 AWS）把 ES 包装成托管服务赚钱。

SSPL 要求"提供 ES 为 SaaS 必须开源整个服务栈"——AWS 不可接受，**fork 出 OpenSearch**（Apache 2.0）。

为什么是 SSPL 不是 GPL：
- GPL 没明确"SaaS 也要开源"
- SSPL 是 MongoDB 2018 发明的，专门补这个洞
- AGPLv3 后来也加进来作为第三选项

</details>

### Q11.2 ⭐⭐ ELv2 下能做什么不能做什么？

<details><summary>答案</summary>

**可以**：

- 公司内部企业搜索
- 自用 ES 跑业务、SaaS 公司内部存数据
- 产品中嵌入搜索（搜索是手段，不是核心商品）
- 修改源码自用

**不能**：

- 把 ES 包装成"managed Elasticsearch service"对外卖
- 提供"hosted ES" 给客户用
- 移除 / 篡改 ES 内部的许可证检查

99% 公司其实根本不受 ELv2 影响——除非主营是 ES 托管。

</details>

### Q11.3 ⭐⭐ 为什么 ES 8 client 默认连不上 OpenSearch？

<details><summary>答案</summary>

ES 8 client 校验响应 header `X-Elastic-Product: Elasticsearch`，OpenSearch 不返回这个 header → 直接拒绝连接。

补救：

1. 降级用 ES 7.x client（不校验 product header）
2. 用 OpenSearch 官方 client（`opensearch-py` / `opensearch-java`），API 与 ES 7 几乎一致

</details>

### Q11.4 ⭐⭐ ILM 和 ISM 的差异？

<details><summary>答案</summary>

| 维度 | ES ILM | OS ISM |
|---|---|---|
| API | `_ilm/policy` | `_plugins/_ism/policies` |
| 名称 | Index Lifecycle Management | Index State Management |
| 阶段命名 | hot/warm/cold/frozen/delete | 用户自定义 state |
| Action | rollover/shrink/forcemerge/freeze/searchable_snapshot/delete | 类似但用户配置 |
| Frozen tier | 与 searchable snapshot 集成 | 不一定有同等抽象 |

DSL 不一致，迁移要重写策略。

</details>

### Q11.5 ⭐⭐⭐ semantic_text 解决什么问题？OS 上怎么做？

<details><summary>答案</summary>

ES 8.16+ 的 `semantic_text` 字段类型让 RAG 极其简单：

- 写入：自动 embed → 写入向量
- 查询：自动 embed 查询文本 → kNN

```json
"properties": {"content":{"type":"semantic_text","inference_id":".elser_model_2"}}
```

OS 对应：

- 用 ingest pipeline + neural search 插件
- 要手动配 embed 模型
- 查询要明确 `neural` 子句

工程量差几倍。

</details>

### Q11.6 ⭐⭐ 跨集群 reindex 的前置配置？

<details><summary>答案</summary>

目标集群 `elasticsearch.yml`（或 `opensearch.yml`）：

```yaml
reindex.remote.whitelist: source-cluster.example.com:9200, *.internal:9200
```

然后：

```bash
POST _reindex
{
  "source": {
    "remote": {"host":"https://source:9200","username":"...","password":"..."},
    "index":"logs-*"
  },
  "dest":{"index":"logs-migrated"}
}
```

要点：

- 大索引分批做（按时间或 _id 范围）
- size 5000 起调
- 监控 source 集群负载（reindex 读会拖慢）

</details>

### Q11.7 ⭐⭐ 列出至少 3 个 ES 与 OS 在 mapping / DSL 上的差异？

<details><summary>答案</summary>

| 差异 | ES | OS |
|---|---|---|
| 向量字段类型 | `dense_vector` | `knn_vector` |
| 向量引擎 | Lucene HNSW | Lucene / FAISS / NMSLIB |
| 量化 | int8/int4/bbq | PQ/SQ |
| Semantic | `semantic_text` | 无（要 Neural Search） |
| Sparse | ELSER / `sparse_vector` | Neural Sparse 插件 |
| SQL | _sql 或 ESQL | _sql（PPL） |
| 安全 API | `_security/*` | `_plugins/_security/*` |
| ML | `_ml/*` | `_plugins/_ml/*` |

</details>

### Q11.8 ⭐⭐ SaaS 公司用 ES 做产品内嵌搜索，要不要担心 SSPL？

<details><summary>答案</summary>

**多半不用担心**。

判断标准：你卖的是**产品内嵌的搜索功能**还是 **ES 托管服务本身**？

- 卖 SaaS 业务系统，里面有搜索框 → ✅ 内嵌使用，不违反
- 卖"我帮你跑 ES 集群"的托管服务 → ❌ 违反
- 暴露 ES API 给客户直接用 → ❌ 接近违反，需要法务评估

灰色地带：你的客户能通过 API 直接发 ES query。这时建议法务先咨询。

</details>

### Q11.9 ⭐⭐⭐ AGPLv3 与 SSPL 的相同点和不同点？

<details><summary>答案</summary>

**相同点**：

- 都是 copyleft（强约束）
- 都要求"在网络上提供服务"也必须开源代码

**不同点**：

| 维度 | AGPLv3 | SSPL |
|---|---|---|
| 历史 | 2007 FSF | 2018 MongoDB |
| OSI 认证 | ✅ | ❌ |
| 范围 | 修改后的程序 | 整个服务栈（包括周边管理工具） |
| 商业 / 开源界共识 | 开源 | 商业偏见，被多数大企业拒绝 |

ES 8.16+ 把 AGPLv3 加进选项 → "可以叫开源了"，但商业约束基本一样严苛。

</details>

### Q11.10 ⭐⭐ AWS 跑 5 年 OpenSearch 的电商团队要不要迁 ES？

<details><summary>答案</summary>

**默认建议：不迁**。

迁的成本：

- 客户端代码改造（OS-py → ES-py）
- 仪表盘迁（OS Dashboards → Kibana）
- 监控告警重搭
- 数据迁移（reindex 远程或快照）
- 灰度切流期间 + 监控 + 回滚机制
- 总成本 6-12 月 + 多团队投入

迁的收益：

- 用 ELSER → 召回提升 5-10%（但要测才知道）
- ESQL → 查询体验更好
- 一些商业 ML 特性

判断：除非有**明确量化收益**（如 ELSER 实测提升 GMV 5%），否则**留在 OS 更省事**。

</details>

---

## 综合大题

### Q.综合.1 ⭐⭐⭐ 给一个集群配置：1TB/天日志，保留 90 天，写入 1w QPS，查询低频。

<details><summary>答案</summary>

容量估算：
- 90 天总量 = 90 TB（原始）
- best_compression codec + warm/cold → 物理 30-50 TB

集群拓扑：
- **master**: 3 节点，4C/8G，专用
- **hot**: 3 节点，16C/64G，NVMe SSD（写入 + 近 1 天查询）
- **warm**: 2 节点，8C/64G，SATA SSD（1-7 天）
- **cold**: 2 节点，4C/32G，HDD（7-30 天）
- **frozen**: searchable snapshot 到 S3（30-90 天）

索引策略：
- 按天 rollover：`logs-2026.05.13`
- 每日索引：6 shard × 1 replica（hot 节点 = 6）
- ILM：1d hot → 7d warm（shrink to 1 shard + forcemerge）→ 23d cold → 60d frozen → 90d delete

写入：
- bulk 10MB
- refresh_interval 30s
- translog async / sync_interval 30s
- replica 1
- indexing pressure 监控

查询：
- 主要查 7 天内（hot + warm）
- 大查询走 coordinating-only 节点（2 个，独立 16G heap）

监控：
- 元集群（独立 3 节点）接收 monitoring 数据
- heap 80% 告警 / disk 80% 告警 / rejected > 0 告警

</details>

### Q.综合.2 ⭐⭐⭐ 给一个 RAG 系统：百万文档，要 hybrid search + rerank。

<details><summary>答案</summary>

mapping：

```json
{
  "mappings": {
    "properties": {
      "title": {"type":"text","analyzer":"ik_max_word"},
      "content": {"type":"text","analyzer":"ik_max_word"},
      "embedding": {
        "type":"dense_vector",
        "dims": 1024,
        "index": true,
        "similarity":"dot_product",
        "element_type":"byte",
        "index_options":{"type":"int8_hnsw","m":16,"ef_construction":100}
      },
      "category": {"type":"keyword"},
      "popularity": {"type":"rank_feature"}
    }
  }
}
```

写入：

- ingest pipeline 调外部 embedding 模型（bge-large-zh / text-embedding-3-small）
- 按 200 token chunk + 50 overlap
- 一 doc 多 chunk → parent-child 或 nested

查询：

```json
POST docs/_search
{
  "size": 10,
  "retriever": {
    "rrf": {
      "retrievers": [
        {"standard":{"query":{"multi_match":{"query":"...","fields":["title^3","content"]}}}},
        {"knn":{"field":"embedding","query_vector_builder":{...},"k":50,"num_candidates":500}}
      ]
    }
  },
  "rescore": [{
    "window_size":50,
    "query":{"rescore_query":{ /* cross-encoder via inference processor */ }}
  }]
}
```

调优：
- HNSW int8 量化 → 内存省 4×
- pre-filter 走 category 缩小搜索空间
- BM25 b=0.5（文档长度不齐）
- rerank 用 bge-reranker-large

容量：100w doc × 1024 dim × 1 byte = 1 GB raw vector + HNSW 图 ~1.5×

集群：3 数据节点 64GB heap，单分片 30GB，2 副本。

</details>

### Q.综合.3 ⭐⭐⭐ 集群突然每秒几百个 429，怎么排查？

<details><summary>答案</summary>

**第一分钟（确认是什么错）**：

```bash
GET _cat/thread_pool?v&h=node,name,active,queue,rejected
GET _nodes/stats/indexing_pressure
GET _nodes/stats/breaker
GET _cluster/health
```

看到 write 队列爆 or indexing pressure 接近 limit or breaker tripped。

**第二步（找触发条件）**：

```bash
GET _cat/tasks?v&detailed=true | head -30
# 看有没有大 bulk / reindex 在跑

GET _cat/recovery?v
# 看是不是 shard 重建
```

**第三步（业务侧）**：
- 客户端是否同时跑批量任务（cron 触发）
- 副本数最近是否调过
- 是否新加索引 / 删索引导致 rebalance

**第四步（修复）**：

- 短期：客户端降并发 / 限流；副本调到 1（如果调过 2 以上）
- 中期：加节点 / 加 heap / 加 SSD
- 长期：分流；区分高优 / 低优队列

</details>

---

## 答题统计

| 章节 | 题数 | 难度分布 |
|---|---|---|
| E01 倒排索引 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| E02 Segment | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| E03 集群拓扑 | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| E04 分片路由 | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| E05 Mapping | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| E06 DSL | 10 | ⭐ 2 / ⭐⭐ 5 / ⭐⭐⭐ 3 |
| E07 BM25 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| E08 向量 | 10 | ⭐ 1 / ⭐⭐ 5 / ⭐⭐⭐ 4 |
| E09 写入 | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| E10 性能 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| E11 vs OS | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| 综合 | 3 | ⭐⭐⭐ 3 |
| **合计** | **113** | — |

---

## 自评标准

- 答对 > 90 题：你已经能独立设计 / 调优生产 ES 集群
- 70-90：核心能力扎实，缺少边角细节
- 50-70：还在中级，建议把不会的逐条研究
- < 50：从 E01-E02 重新读起，把概念和实操结合

---

> 🔁 反馈：把不会的题写到笔记里，下个月再答一遍——记得住的才是真知识
