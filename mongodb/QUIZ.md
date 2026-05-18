# MongoDB 深度课程 · 自测题库

> 配合 [INDEX.md](./INDEX.md) 使用。每章 ~10 题，含答案与展开解析。
> 难度标记：⭐ 概念 ⭐⭐ 进阶 ⭐⭐⭐ 源码级 / 容易踩坑

> 答题建议：先盖住答案自答，再展开核对。能讲出"为什么"比答对结论更重要。

---

## M01 文档模型与 BSON

### Q1.1 ⭐⭐ BSON 与 JSON 的 3 个关键差异？

<details><summary>答案</summary>

1. **二进制 + 每字段类型 tag**：JSON 是文本无类型；BSON 每字段带 type ID（int32 / int64 / double / decimal128 / date / binary / ObjectId 等）
2. **字段长度前缀**：可跳过整个值不用解析
3. **丰富类型**：date / binary / ObjectId / decimal128 / regex 等 JSON 没有

附带：内部用小端字节序。

</details>

### Q1.2 ⭐⭐ 16MB 文档上限怎么影响 schema 设计？

<details><summary>答案</summary>

- 单文档不能无限增长 → 数组 / 嵌套结构有 cap
- 评论列表、订单列表等不能全 embed
- 超过 16MB 的内容用 GridFS 切片
- 实际经验：单文档 < 100KB 健康，> 1MB 警觉

</details>

### Q1.3 ⭐⭐⭐ ObjectId 的 12 字节如何构成？

<details><summary>答案</summary>

```
| 4 bytes timestamp（秒） | 5 bytes random | 3 bytes counter |
```

- 时间戳前缀让 ObjectId 大致按生成时间排序（不严格单调，多机器并发）
- random + counter 防冲突
- 8.0 把内部 counter 改为更 random 化，改善高并发热点

</details>

### Q1.4 ⭐⭐ 自定义 _id 用 UUID v4 vs ObjectId 的取舍？

<details><summary>答案</summary>

| 维度 | ObjectId | UUID v4 |
|---|---|---|
| 大小 | 12 字节 | 16 字节 |
| 时间排序 | 大致单调（按时间） | 完全随机 |
| 写入热点 | 同 page 集中（高 QPS 时） | 散得开但 cache 局部性差 |
| 跨系统唯一 | 同进程内唯一 | 全球唯一 |

ObjectId 适合多数场景；UUID v7（时间排序）是更好的现代选择。

</details>

### Q1.5 ⭐⭐ schema validator 的 strict / moderate 区别？

<details><summary>答案</summary>

- **strict**：所有写都验证（默认）
- **moderate**：只对"已经符合 validator 的文档"的更新进行验证；对老文档的修改不验证
- **off**：完全关

迁移期用 moderate 让老文档可以继续工作。

</details>

### Q1.6 ⭐⭐ 数组无限增长的具体危害？

<details><summary>答案</summary>

1. 文档最终撑爆 16MB
2. 每次 $push 重写整文档（写放大）
3. multi-key 索引占用大
4. 数组遍历 / 排序变慢
5. 复制 oplog entry 变大

</details>

### Q1.7 ⭐ Decimal128 vs double

<details><summary>答案</summary>

- double：IEEE 754 64-bit 浮点。会有精度问题（`0.1 + 0.2 ≠ 0.3`）
- Decimal128：128-bit 高精度十进制，34 位有效数字。**财务 / 金融必须用**

```js
NumberDecimal("0.1").add(NumberDecimal("0.2"))   // 0.3 精确
```

</details>

### Q1.8 ⭐⭐ multi-key index

<details><summary>答案</summary>

数组字段的索引——为数组中**每个元素**建索引项：

```js
{tags: ["red", "small"]}
// 索引项："red" → docid, "small" → docid
```

特点：

- 自动开启（不需要 explicit）
- 同一复合索引最多 1 个数组字段
- 占空间大（数组长度 × 文档数）

</details>

### Q1.9 ⭐⭐ collation 解决什么？默认为什么不开

<details><summary>答案</summary>

默认按字节序比较 → "Apple" / "Banana" / "apple"（大写在前）。

collation 让按语言规则比（中文拼音、大小写不敏感等）：

```js
{ collation: { locale: "zh", strength: 1 } }
```

默认不开是因为：

- 字节序最快
- 大部分业务不需要

要语言敏感的 query（搜索 / 排序）才开。

</details>

### Q1.10 ⭐⭐ GridFS 适合什么不适合什么？

<details><summary>答案</summary>

**适合**：

- 单文件 > 16MB
- 需要随机读（按 offset 取某段）
- 文件元数据要查询
- 已经在用 MongoDB，文件不多不大

**不适合**：

- 大量大文件（用 S3 / 对象存储）
- 高吞吐分发（CDN 不如对象存储）
- 频繁修改文件内容

</details>

---

## M02 索引

### Q2.1 ⭐⭐ ESR 规则

<details><summary>答案</summary>

复合索引字段顺序：**Equality → Sort → Range**

- Equality 最先：选择性最强
- Sort 中间：复用索引顺序，避免 in-memory sort
- Range 最后：范围扫描发生在最后

例：`find({status:"paid", amount:{$gt:100}}).sort({createdAt:-1})`
→ 索引 `{status:1, createdAt:-1, amount:1}`

</details>

### Q2.2 ⭐ 复合索引 {a:1,b:1,c:1} 加速哪些查询

<details><summary>答案</summary>

- `{a}` ✅
- `{a, b}` ✅
- `{a, b, c}` ✅
- `{a, c}` ⚠️（a 走索引 c 不走）
- `{b}` ❌
- `{b, c}` ❌

**只能用前缀**。

</details>

### Q2.3 ⭐⭐ 覆盖索引的两个必要条件

<details><summary>答案</summary>

1. 查询字段 + projection 字段全在索引中
2. 显式排除 `_id`（或 _id 也在索引中）

```js
db.users.createIndex({email:1, name:1})
db.users.find({email:"a@x.com"}, {_id:0, name:1})   // 覆盖
db.users.find({email:"a@x.com"}, {name:1})           // 不覆盖（要 _id）
```

</details>

### Q2.4 ⭐⭐ multi-key 为什么不能两个数组字段

<details><summary>答案</summary>

如果一个文档同时在两个数组字段：

```js
{tags: ["a","b"], cats: ["X","Y"]}
```

索引项就要笛卡尔积：(a,X)(a,Y)(b,X)(b,Y)。

爆炸性增长，MongoDB 拒绝。所以**同一复合索引最多 1 个数组字段**。

</details>

### Q2.5 ⭐⭐ TTL 索引字段类型 + 删除频率

<details><summary>答案</summary>

- 字段类型必须是 **BSON Date**（int / long / string 都不行）
- 后台任务每 **60 秒**扫一次
- 删除有滞后（可能晚 60+ 秒）
- 只在 primary 上删，secondary 通过复制

```js
db.sessions.createIndex({createdAt: 1}, {expireAfterSeconds: 3600})
```

</details>

### Q2.6 ⭐⭐ partial vs sparse

<details><summary>答案</summary>

- **sparse**：字段不存在的文档不索引（老语法）
- **partial**：任意条件不满足的文档不索引（更灵活）

```js
// partial
db.users.createIndex(
  {email:1},
  {unique:true, partialFilterExpression: {email: {$type:"string"}}}
)
```

新版本一律用 partial 替代 sparse。

</details>

### Q2.7 ⭐⭐ wildcard 索引适用

<details><summary>答案</summary>

适合：**真正动态 schema**，字段路径不固定（如产品属性、事件 props）。

```js
db.events.createIndex({"props.$**": 1})
```

不适合：

- 字段固定时浪费空间（用普通索引）
- 7.0 前不能与复合索引中其他字段组合（7.0+ 支持 compound wildcard index，但仍只能含一个 wildcard 字段）
- 查询语法对单字段（不支持 `$or` 跨不固定字段）

</details>

### Q2.8 ⭐⭐⭐ unique 索引下 null 字段为什么会"重复"

<details><summary>答案</summary>

unique 索引把"字段缺失"视为 null。多个文档都没这字段 → 索引项都是 null → duplicate key。

修复：用 partial filter：

```js
db.users.createIndex(
  {email:1},
  {unique:true, partialFilterExpression: {email: {$type:"string"}}}
)
```

只对有 email 的文档建唯一索引。

</details>

### Q2.9 ⭐⭐ hashed 索引为什么不支持范围

<details><summary>答案</summary>

hashed 索引存的是 `hash(field_value)`，不是原值。hash 后**顺序无意义**（相邻值的 hash 完全无关）。

→ 范围查询无法走 hashed 索引，必须全 scan。

hashed 适合：

- 等值查询
- 分片场景（散写入热点）

</details>

### Q2.10 ⭐⭐⭐ MongoDB vector search vs ES dense_vector

<details><summary>答案</summary>

| 维度 | MongoDB Vector | ES dense_vector |
|---|---|---|
| 算法 | HNSW | HNSW（Lucene） |
| 量化 | binary / scalar | int8/int4/bbq |
| 与过滤结合 | $vectorSearch + $match | knn pre-filter |
| 一站式 | ✅ | ✅ |
| 数据量上限 | 看 cache（一般 < 1000 万） | 类似 |

底层都是 HNSW。MongoDB 集成更紧凑（aggregation pipeline），ES 选项更丰富。

</details>

---

## M03 WiredTiger

### Q3.1 ⭐ WT cache 默认大小

<details><summary>答案</summary>

```
cache = max(50% × (RAM - 1GB), 256MB)
```

例：

- 16GB RAM → 7.5GB cache
- 64GB RAM → 31.5GB cache

专用机器用默认；共享 / 容器化需要降低。

</details>

### Q3.2 ⭐ checkpoint 与 journal 各自解决什么

<details><summary>答案</summary>

- **journal**：每次写 op 都 append + 周期 fsync（默认 100ms）。保证**单次崩溃恢复**。
- **checkpoint**：默认 60 秒把所有 dirty page 刷盘，建立一致快照点。**恢复时只需从最近 checkpoint + journal replay**。

journal 不能少，否则崩溃后只能丢上次 checkpoint 之后的所有写。

</details>

### Q3.3 ⭐⭐ cache dirty page > 20% 会怎样

<details><summary>答案</summary>

触发 **eviction**：

- WT 启动更激进 eviction 线程
- application threads 也帮忙 evict（业务延迟突增）
- 写入吞吐下降

正常应该 < 20% dirty。持续高说明 cache 不够 / 写入太快 / 磁盘 IO 慢。

</details>

### Q3.4 ⭐⭐ snappy / zstd / zlib 选择

<details><summary>答案</summary>

| 算法 | 压缩比 | CPU |
|---|---|---|
| snappy（默认） | 中 | 低 |
| zstd（4.2+） | 高 | 中 |
| zlib | 高 | 高 |

实战：

- 默认 snappy 90% 场景够
- 磁盘紧张 + CPU 富裕（日志） → zstd
- zlib 几乎被 zstd 全面替代

</details>

### Q3.5 ⭐⭐⭐ 长事务为什么让 history store 暴涨

<details><summary>答案</summary>

WT MVCC 在 cache 中保留**所有正在被读的版本**。长事务（事务开始时间老）必须持有那个时刻的快照 → 之后所有版本都不能丢。

in-memory 老版本满 → 下沉到 `WiredTigerHS.wt`（history store）。

修复：

- 业务短事务（< 1 分钟）
- `transactionLifetimeLimitSeconds` 限制（默认 60s）

</details>

### Q3.6 ⭐⭐ journal 关掉的代价

<details><summary>答案</summary>

技术可关：

```yaml
storage:
  journal:
    enabled: false
```

代价：

- 单节点崩溃 → 必须 **resync**（从其他副本全量复制）
- 大数据量 resync 极慢
- 总体来说"省的一点 IO" 远不如"恢复时的灾难"

**永远不要关 journal**。

</details>

### Q3.7 ⭐⭐ B-tree vs LSM 在 MongoDB 选择

<details><summary>答案</summary>

WT 支持两种，但 MongoDB **默认用 B-tree**。

- B-tree：读取性能高、范围查询好、写入有 page 分裂代价
- LSM：写吞吐极高、读放大（多层 SSTable）、compaction IO 重

MongoDB 工作负载偏读 + 范围 + 索引精准 → B-tree 更稳。LSM 适合写极重场景，但 MongoDB 实践中几乎不用。

</details>

### Q3.8 ⭐⭐ Working Set 怎么估算

<details><summary>答案</summary>

```
Working Set ≈ 经常访问的文档大小 + 索引大小
```

观察：

```js
db.serverStatus().wiredTiger.cache["bytes currently in the cache"]
// 长期观察这个值
```

如果 cache used / configured 持续 95%+ → working set 接近 cache 上限。

性能黄金法则：**Working Set ≤ cache 大小**。

</details>

### Q3.9 ⭐⭐ eviction 跟不上的症状

<details><summary>答案</summary>

- application thread 也被拉去 evict（业务延迟暴增）
- `modified pages evicted` 速率高
- `pages read into cache` 速率高（miss 多）
- 写入吞吐降
- mongostat 看 dirty / used 双高

修复：加 cache / 降低写入 / 升级磁盘 IO。

</details>

### Q3.10 ⭐⭐ fsyncLock 是什么

<details><summary>答案</summary>

```js
db.fsyncLock()    // flush dirty + 阻塞写
// 备份操作
db.fsyncUnlock()
```

- 刷新所有 dirty page 到磁盘
- **阻塞所有写**（读仍可）
- 保证文件级 snapshot 一致

用途：文件级备份前必须 lock 才能保证一致性。生产用副本集 + 在 secondary 上 fsyncLock 减少对 primary 影响。

</details>

---

## M04 查询与聚合

### Q4.1 ⭐ find projection 中 _id 默认行为

<details><summary>答案</summary>

`_id` 默认**包含**。不要的话显式排除：

```js
db.users.find({}, {name: 1})           // 含 _id
db.users.find({}, {_id: 0, name: 1})   // 不含 _id
```

</details>

### Q4.2 ⭐⭐ $elemMatch vs 多个独立条件

<details><summary>答案</summary>

```js
// 不用 $elemMatch
db.users.find({"scores.subject":"math", "scores.score":{$gt:90}})
// 数组里 subject=math 的元素 + 数组里 score>90 的元素（可能不同元素）

// 用 $elemMatch
db.users.find({scores: {$elemMatch: {subject:"math", score:{$gt:90}}}})
// 数组里"既 subject=math 又 score>90"的同一个元素
```

精髓：让多个条件作用于**同一个元素**。

</details>

### Q4.3 ⭐⭐⭐ $match 放 $group 后能不能用索引

<details><summary>答案</summary>

**不能**。索引只对原始 collection 文档有意义。$group 输出的是聚合结果（虚拟文档），没有索引。

实战：

- $match 永远尽量提前
- 优化器会自动 push $match，但只能对相邻 stage
- 业务侧也要主动 reorder

</details>

### Q4.4 ⭐⭐⭐ $lookup 慢的两个原因

<details><summary>答案</summary>

1. **右侧字段无索引**：每次外侧文档都要 collection scan 右侧
2. **左侧文档太多**：N × M 复杂度

修复：

- 右侧 join 字段建索引
- 左侧先 $match 缩小
- 极端情况下反范式（embed）替代 $lookup

</details>

### Q4.5 ⭐⭐ $facet 并行 vs 串行

<details><summary>答案</summary>

**串行**。各分支按顺序执行。

但比"跑 3 次独立 query"快——只扫一次输入。

适合：仪表盘多维度统计。

</details>

### Q4.6 ⭐⭐ $out vs $merge

<details><summary>答案</summary>

| Stage | 行为 |
|---|---|
| `$out` | 整 collection 替换（先 drop 再写） |
| `$merge` | 灵活合并（upsert / append / replace） |

`$merge` 支持：

```js
{whenMatched: "merge|replace|keepExisting|fail|pipeline"}
{whenNotMatched: "insert|discard|fail"}
```

适合**增量物化视图**（每天追加新数据，不动旧的）。

</details>

### Q4.7 ⭐⭐ 聚合 100MB 限制怎么绕过

<details><summary>答案</summary>

```js
db.orders.aggregate([...], { allowDiskUse: true })
```

每个 stage（如 $sort / $group）超 100MB 时溢出到磁盘。

代价：磁盘 swap 慢。最好优化 pipeline 让中间结果小。

</details>

### Q4.8 ⭐⭐⭐ $unwind 后 $sort 不走索引

<details><summary>答案</summary>

$unwind 把数组拆成多行 → 输出顺序与原 collection 无关 → 索引顺序失效。

后续 $sort 必须 in-memory（或 disk）做。

修复：

- 把 $sort 放到 $unwind 之前（如果可能）
- 或接受 in-memory sort（小数据 OK）

</details>

### Q4.9 ⭐ findOneAndUpdate 的原子性

<details><summary>答案</summary>

- 查找 + 更新 + 返回 在单一文档操作内**原子完成**
- 不会有"查到 → 别人改了 → 我覆盖"竞态
- 返回更新前 or 更新后的文档（returnNewDocument 选择）

适合：

- 计数器取唯一值
- 队列模式（fetch + claim）
- 状态机原子推进

</details>

### Q4.10 ⭐⭐ upsert + $inc 场景

<details><summary>答案</summary>

```js
db.counters.updateOne(
  {_id: "today_views"},
  {$inc: {count: 1}},
  {upsert: true}
)
```

不存在则插入 + count=1；存在则 count++。

适合：

- 第一次访问的计数器创建
- 状态聚合（updateOrCreate 模式）
- 幂等写入

</details>

---

## M05 副本集

### Q5.1 ⭐ PSS 比 PSA 好在哪

<details><summary>答案</summary>

- PSS：3 数据节点。挂 1 个，剩 2 个仍能 quorum + 服务 w:majority
- PSA：2 数据 + 1 arbiter。挂 1 个数据节点 → ISR 只剩 P + A → A 不存数据 → w:majority 写阻塞

PSA 节省一台机器但牺牲了"挂 1 仍能强写"，**不推荐生产**。

</details>

### Q5.2 ⭐⭐ oplog 为什么必须幂等

<details><summary>答案</summary>

secondary 可能：

- 重启后 replay 重叠部分
- 网络抖动重复拉
- 从其他 secondary 拉链式复制

每条 oplog entry 必须可以 replay 多次结果一样。

实现：

```js
{$inc: {count: 1}}   // 不是幂等
// 但 oplog 记录为：
{op:"u", o:{$v:2, diff:{u:{count: new_value}}}}   // 直接设置为最终值，幂等
```

</details>

### Q5.3 ⭐⭐ replication lag 60 秒 3 个可能原因

<details><summary>答案</summary>

1. **secondary 资源不足**（CPU / IO 跟不上 apply）
2. **secondary 上跑大查询**（占用资源）
3. **网络带宽 / 跨 region RTT 高**
4. **大批量写入超过 secondary apply 能力**
5. **secondary 索引重建 / TTL 清理 / 大 backup**

</details>

### Q5.4 ⭐⭐ priority 0 效果

<details><summary>答案</summary>

- 永远**不能成为 primary**
- 可以参与投票
- 可以服务读
- 可以做 hidden / DR

适合：

- 备份专用节点
- 跨 region DR（不希望 DR 节点意外成 primary）
- 报表节点

</details>

### Q5.5 ⭐ read preference 5 种适用

<details><summary>答案</summary>

| 模式 | 适用 |
|---|---|
| primary（默认） | 通用业务，强一致 |
| primaryPreferred | primary 挂时可读 secondary |
| secondary | 报表 / 离线 |
| secondaryPreferred | 读密集 + 容忍 stale |
| nearest | 跨地域降延迟 |

</details>

### Q5.6 ⭐⭐⭐ w:majority + wtimeout: 1000 失败回滚吗

<details><summary>答案</summary>

**不回滚**。wtimeout 只是"等多数 ack"的超时，不是事务。

行为：

- primary 已经 apply 了
- 1 秒内多数 secondary 没 ack → 抛 WriteConcernError
- 但写已经存在，secondary 会继续追赶

应用要么：

- 业务上视为"成功"（最终会一致）
- 业务上视为"未知"，retry idempotent

建议 wtimeout 长（5-10s）或不设。

</details>

### Q5.7 ⭐⭐ read concern majority vs linearizable

<details><summary>答案</summary>

- **majority**：读已 committed 到多数节点的版本。不会回滚。能在 secondary 用。
- **linearizable**：只能在 primary。等当前 majority commit 完成 + 再次确认 primary 身份。**最强一致**。

代价 linearizable 大：多次往返 + 必须 primary。99% 业务 majority 够。

</details>

### Q5.8 ⭐⭐ retryWrites 对什么 op 生效

<details><summary>答案</summary>

**对幂等单文档 op 生效**：

- insertOne
- updateOne（不含 multi-field $inc 等？实际是支持的）
- replaceOne
- findOneAndUpdate
- deleteOne

**不生效**：

- updateMany / deleteMany（不幂等，可能改了一部分后 failover）
- bulk write 中的非幂等部分

failover 时自动重试一次。

</details>

### Q5.9 ⭐⭐⭐ rollback 何时发生

<details><summary>答案</summary>

```
T0: primary A 接受写 W
T1: A 网络分区，W 还没复制到多数
T2: B 当选新 primary，接受新写
T3: A 恢复 → 发现自己有"多数没确认的"W → rollback
```

A 上的 W 数据被回滚到 `rollback/` 目录（不丢但下线）。

预防：用 **w:majority**——保证 ack 的写已经在多数节点，不会 rollback。

</details>

### Q5.10 ⭐⭐ 跨 region priority 设置

<details><summary>答案</summary>

主 region 节点：priority 高（如 2）
DR region 节点：priority 0（不希望自动 failover 到 DR）

```
us-east-1: P=2, S=2
us-west-1 (DR): S=0
```

要 DR 接管 → 手动 reconfig priority。避免网络分区时 DR 误当选。

</details>

---

## M06 分片集群

### Q6.1 ⭐ mongos / config server / shard 各自角色

<details><summary>答案</summary>

- **mongos**：客户端入口，无状态路由
- **config server**：3 节点副本集，存集群元数据（chunks、shards、databases）
- **shard**：每个是一个副本集，存实际数据

最小生产：3 mongos + 3 config + N × 3 shard mongod。

</details>

### Q6.2 ⭐⭐ shard key 三大原则

<details><summary>答案</summary>

1. **基数高**：值的种类多（至少几千几万种）
2. **频率均匀**：没有大热点（如某一 user 占 99% 流量）
3. **不单调递增**：避免新数据集中在最大 chunk 的 shard（hot shard）

经典反例：用 `timestamp` 作 shard key → 所有新数据集中 → hot shard。

</details>

### Q6.3 ⭐⭐ ranged vs hashed 适用

<details><summary>答案</summary>

| 类型 | 适用 |
|---|---|
| ranged | 范围查询友好；分布均匀的自然字段（如 customerId） |
| hashed | 单调字段（如 timestamp / ObjectId）需要散开写入 |
| compound hashed（4.4+） | 前缀 ranged 等值 + 后缀 hashed |

</details>

### Q6.4 ⭐⭐⭐ compound hashed 解决什么

<details><summary>答案</summary>

```js
sh.shardCollection("events", {region: 1, userId: "hashed"})
```

- 第一字段 ranged：同 region 数据集中（zone sharding / 地域查询友好）
- 第二字段 hashed：region 内分散，避免热点

最佳分片实践组合。

</details>

### Q6.5 ⭐⭐ chunk 默认大小 + 分裂 / 迁移触发

<details><summary>答案</summary>

- 默认 **128 MB**（6.0+；6.0 前 64 MB）
- 超过 → 自动分裂（仅元数据，不复制）
- shard 间 chunk 差太多 → balancer 迁移

</details>

### Q6.6 ⭐⭐⭐ jumbo chunk

<details><summary>答案</summary>

某个 chunk 长大但**无法分裂**——shard key 取值都相同（如同 user 几亿订单）。

特征：

- 大小可能几 GB
- balancer 无法迁移
- 该 shard 持续负载不均

修复：

- 紧急：手动清除 jumbo 标记 + 分裂（如果可能）
- 长期：reshardCollection 改 shard key

预防：shard key 频率均匀（这是三大原则之一）。

</details>

### Q6.7 ⭐⭐ broadcast query 代价

<details><summary>答案</summary>

不带 shard key 的查询 → mongos 给所有 shard 发请求 → 合并结果。

代价：

- 网络放大 N 倍
- 最慢 shard 拖整体
- mongos 内存合并大量结果

修复：业务侧让查询带 shard key。

</details>

### Q6.8 ⭐⭐ zone sharding 数据本地化

<details><summary>答案</summary>

```js
sh.addShardToZone("sh-us", "US")
sh.addShardToZone("sh-eu", "EU")
sh.updateZoneKeyRange("mydb.users",
  {region:"US", _id:MinKey},
  {region:"US", _id:MaxKey},
  "US")
```

效果：region=US 的用户数据**强制**落到 tag US 的 shard（部署在美国机房）。

适合：

- 地理合规（GDPR）
- 跨 region 降延迟
- 冷热分层（旧数据移到便宜 shard）

</details>

### Q6.9 ⭐⭐⭐ reshardCollection 适用场景

<details><summary>答案</summary>

5.0+ 引入。在线改 shard key：

```js
sh.reshardCollection("mydb.orders", {newKey: "hashed"})
```

代价：

- 全量复制（旧 chunk → 新 chunk）
- 巨大网络 + IO 压力
- 持续几小时到几天
- 业务流量受影响

适用：

- shard key 选错了无法承受
- 业务低峰长时间窗口能跑
- 没其他选择

</details>

### Q6.10 ⭐⭐ 单 shard 写 90% 根因

<details><summary>答案</summary>

可能原因：

1. **shard key 频率不均**：某段 shard key 集中（如某大客户）
2. **shard key 单调**：所有新写入同一最大 chunk
3. **balancer 没跑**：被 stop / window 配置错
4. **balancer 跟不上**（极端写入速度）

诊断：

```js
sh.status()   // 看 chunk 分布
```

如果 chunk 数均匀但流量集中 → 1 / 2。如果 chunk 数也不均 → 3 / 4。

</details>

---

## M07 事务与一致性

### Q7.1 ⭐⭐ 单文档原子 vs 多文档事务边界

<details><summary>答案</summary>

**单文档原子**：

- 一个 updateOne / replaceOne / findOneAndUpdate 内
- 所有字段 / 嵌套 / 数组操作都原子
- 不需要事务包装

**多文档事务**：

- 修改多个文档（如银行转账）
- 跨 collection
- 副本集 4.0+ / 分片 4.2+

建议优先 schema 设计成单文档原子；事务只在必要时用。

</details>

### Q7.2 ⭐⭐⭐ retry transaction 时区分 retry 整个 vs retry commit

<details><summary>答案</summary>

看 error label：

- `TransientTransactionError` → 整个事务 retry
- `UnknownTransactionCommitResult` → 只 retry commit
- 都没有 → 永久错误，不要 retry

```js
if (err.errorLabels.includes("TransientTransactionError")) restartAll()
else if (err.errorLabels.includes("UnknownTransactionCommitResult")) retryCommit()
else throw err
```

</details>

### Q7.3 ⭐⭐ read concern majority vs snapshot

<details><summary>答案</summary>

- **majority**：读多数 commit 的最新版本（每次读看当时的 commit 点）
- **snapshot**：事务开始时拍快照，session 内所有读基于这快照（一致视图）

snapshot 用于事务，提供 repeatable-read 级别。

</details>

### Q7.4 ⭐⭐ 事务 60 秒上限原因

<details><summary>答案</summary>

设计上鼓励**小事务**：

- WT MVCC 需要持有老版本，长事务让 history store 爆
- 跨 partition 长事务 LSO 卡（虽然 MongoDB 没有这术语，但概念类似）
- 失败重试代价大（长事务白干）

实战：99% 事务都该在几百 ms 内完成。

</details>

### Q7.5 ⭐⭐ linearizable 在事务中能用吗

<details><summary>答案</summary>

**不能**。事务用 snapshot 隔离。

linearizable 只能用于事务外的单 read，且必须 primary。

</details>

### Q7.6 ⭐⭐⭐ WriteConflict 本质 + 减少

<details><summary>答案</summary>

两个事务同时改同一文档 → 后到的事务 abort。

MongoDB 是**乐观并发**（不锁文档），冲突时整个事务回滚 retry。

减少：

- 减小事务范围（少 doc）
- 缩短事务时间
- 分散热点（计数器拆分）
- 业务侧避免高并发改同 doc

</details>

### Q7.7 ⭐⭐⭐ 分片事务 2PC 两阶段

<details><summary>答案</summary>

```
Phase 1: Prepare
  coordinator → 各参与 shard：prepare
  shard 持久化 prepare oplog
  shard 回 OK

Phase 2: Commit
  coordinator 写 commit decision 到 config
  coordinator → 各 shard：commit
  shard apply + ack
```

代价：

- 跨 shard 事务比单 shard 慢一个数量级
- 任一 shard 不可达 → 事务挂起

业务设计：让事务范围落在同 shard（shard key 合理）。

</details>

### Q7.8 ⭐⭐ causal consistency 解决 stale read

<details><summary>答案</summary>

```js
const session = client.startSession({causalConsistency: true})
db.users.updateOne({_id:1}, {$set:{name:"Alice"}}, {session})
db.users.findOne({_id:1}, {session})   // driver 等 secondary 追上才返回
```

原理：

- 每 op 返回 operationTime
- session 记住最新 operationTime
- 后续读 `afterClusterTime: operationTime`
- secondary 等 lastApplied >= 才返回

保证 "see-your-own-write"。

</details>

### Q7.9 ⭐⭐ Saga 适用场景

<details><summary>答案</summary>

跨服务 / 跨数据库的"分布式事务"，无法用 MongoDB 单集群事务覆盖：

```
1. 扣库存
2. 扣余额（不同 service / DB）
3. 创建订单
任一失败 → 反向执行已执行的步骤
```

事件驱动 + 业务层补偿。适合微服务架构。

</details>

### Q7.10 ⭐⭐ 单点计数器热点修复

<details><summary>答案</summary>

```js
// 反例：所有写到一个 _id
db.counters.updateOne({_id:"global"}, {$inc:{n:1}})   // 高并发 WriteConflict

// 改：分散到 N 个 sub-counter
const shardId = Math.floor(Math.random() * 100)
db.counters.updateOne(
  {_id:"global", shard: shardId},
  {$inc:{n:1}},
  {upsert: true}
)

// 读时聚合
db.counters.aggregate([
  {$match:{_id:"global"}},
  {$group:{_id:"$_id", total:{$sum:"$n"}}}
])
```

写分散到 100 个 doc → WriteConflict 极少。读时多一次 group。

</details>

---

## M08 性能调优

### Q8.1 ⭐ profiler level 1 / 2

<details><summary>答案</summary>

- **level 1**：只记 > slowms 的操作。生产可开。
- **level 2**：记所有。**生产慎用**（开销大、system.profile 爆炸）。

```js
db.setProfilingLevel(1, {slowms: 100})
```

</details>

### Q8.2 ⭐⭐ explain 健康比例

<details><summary>答案</summary>

```
totalKeysExamined / totalDocsExamined / nReturned

理想：1 : 1 : 1（每个索引项对应一个返回文档）
警觉：100 : 100 : 1（索引选择性差，多扫了 100 倍）
最差：N : N : 1（全集合扫）
```

</details>

### Q8.3 ⭐⭐ Working Set 估算

<details><summary>答案</summary>

```
Working Set ≈ 经常访问的文档总大小 + 经常访问的索引总大小
```

观察 `wiredTiger.cache.bytes currently in the cache` 长期趋势：

- 持续上涨 → working set 在增长
- 接近 95% → 接近上限
- 加 RAM 或开始下降意味着已经超过

</details>

### Q8.4 ⭐⭐ cache used 95% 怎么办

<details><summary>答案</summary>

按代价从低到高：

1. **加索引**：少 FETCH，cache 占用降
2. **删冷数据**：TTL / retention
3. **改 schema**：减小单文档
4. **加 RAM**
5. **分片**

不要急着调 cacheSizeGB——根本原因往往是 working set 过大。

</details>

### Q8.5 ⭐⭐⭐ 全局计数器热点修复

<details><summary>答案</summary>

见 Q7.10：分散到 N 个 sub-counter，写散开，读聚合。

也可以用 **approximation pattern**：

- 应用本地累加 100 → batch $inc 100
- 减少写次数 100 倍
- 容忍小延迟 / 计数不精确

</details>

### Q8.6 ⭐⭐ 查询慢 5 步排查

<details><summary>答案</summary>

1. 用 `explain("executionStats")` 看是不是 COLLSCAN
2. 看 `totalKeysExamined / nReturned` 是否合理
3. 看是不是 SORT 在内存（缺索引）
4. 看 cache miss 是不是高（working set 超）
5. 看是不是有锁竞争 / 长事务

</details>

### Q8.7 ⭐⭐ allowDiskUse 适用

<details><summary>答案</summary>

聚合 stage（如 $sort / $group）中间结果超 100MB：

```js
db.coll.aggregate([...], {allowDiskUse: true})
```

代价：磁盘 swap 慢。优先优化 pipeline 让中间结果小。

</details>

### Q8.8 ⭐⭐ drop collection 后磁盘没释放

<details><summary>答案</summary>

WT 文件**不缩**（保留空 page 给未来用）。

修复：

```js
db.runCommand({compact: "coll"})   // 阻塞 collection 一段时间
```

或：

- drop database / collection 后重建
- resync secondary 让它从头建（更彻底）

</details>

### Q8.9 ⭐⭐⭐ 连接数飙到上限根因

<details><summary>答案</summary>

常见：

- 应用每请求开 new MongoClient（没用连接池）
- 连接泄露（没 close）
- 连接池配太大
- 客户端实例太多（每微服务每 pod 独立 100 连接）

修复：

- 应用复用 MongoClient（启动一次）
- 设合理 maxPoolSize（50-100 per app instance）
- 加 mongos / 增加 mongod ulimit

</details>

### Q8.10 ⭐⭐ 单 collection 2TB 该不该分片

<details><summary>答案</summary>

**先试垂直扩**：

- 升级到更大机器（RAM、SSD）
- 副本集仍然能跑

如果：

- working set 仍 > cache 几倍
- 写吞吐 > 单机上限
- 一年内还会涨 3-5×

→ 才考虑分片。

分片代价高（运维、shard key 选错），不要轻易做。

</details>

---

## M09 Change Streams

### Q9.1 ⭐ Change Streams vs 直接读 oplog

<details><summary>答案</summary>

- **Change Streams**：高层 API，应用友好，过滤聚合管道，resume token，跨 shard 自动合并
- **直接读 oplog**：低层，需要自己处理乱序 / 重复 / shard 合并

应用一律用 Change Streams。读 oplog 是 driver / Connector 的事。

</details>

### Q9.2 ⭐⭐ resume token 过期

<details><summary>答案</summary>

原因：oplog 已经 rotate 掉对应 entry（应用离线时间 > oplog window）。

预防：

- oplog window 设大（默认 24h，关键业务 7 天）
- 应用 HA 部署，少 downtime
- 监控 lag

修复：用 startAtOperationTime 重新开始（接受跳过中间）。

</details>

### Q9.3 ⭐⭐ fullDocument updateLookup vs required

<details><summary>答案</summary>

- **updateLookup**：事件被消费时再查 collection 拿当前文档。**可能是后续又被改过的版本**。
- **required**（6.0+ + changeStreamPreAndPostImages）：mongod 记录事件时刻的 pre/post image，**严格事件时刻版本**。

代价：required 让 mongod 额外存历史，占空间。

</details>

### Q9.4 ⭐⭐ invalidate 事件

<details><summary>答案</summary>

触发：

- 监听的 collection 被 drop
- 数据库被 drop
- collection 被 rename

行为：stream 关闭，应用必须**重启** stream（不能从同 token 续）。

处理：

```js
if (event.operationType === "invalidate") {
  recreateStream()
}
```

</details>

### Q9.5 ⭐⭐ 单 collection 顺序 vs 跨 collection 顺序

<details><summary>答案</summary>

- **单 collection**：事件按 cluster time 严格有序
- **跨 collection / 跨 shard**：mongos 合并后按 cluster time 排序，但**同时刻不同来源顺序不保证**

业务设计：不要依赖跨 collection 的严格顺序。

</details>

### Q9.6 ⭐⭐ at-least-once + 幂等

<details><summary>答案</summary>

```js
while (stream.hasNext()) {
  const event = stream.next()
  await processEvent(event)         // 1. 处理（必须幂等）
  await saveToken(event._id)         // 2. 持久化 token
}
```

- 处理在 token 保存之前 → 重启可能重复处理
- 业务侧必须**幂等**（用 doc id 覆盖式写、upsert）

</details>

### Q9.7 ⭐⭐⭐ 分片集群 watch 工作机制

<details><summary>答案</summary>

```
client → mongos
mongos → 各 shard 各开一个 oplog 流
mongos 内部合并 → 按 cluster time 排序 → 输出全集群有序流给 client
```

mongos 是无状态合并节点。client 不直接连各 shard。

性能：shard 数多 → mongos 工作量增加，但 client 透明。

</details>

### Q9.8 ⭐⭐ updateLookup 拿的不是事件时版本

<details><summary>答案</summary>

```
T0: doc A.field = 1
T1: update set field = 2 → event 1
T2: update set field = 3 → event 2
T3: consumer 处理 event 1，updateLookup 拿当前文档 → 看到 field = 3 ❌
```

修复：用 fullDocumentBeforeChange / fullDocument: required（6.0+ + collMod）。

</details>

### Q9.9 ⭐⭐ 初始全量同步 + 增量切换

<details><summary>答案</summary>

```js
// 1. 记录当前 cluster time
const now = (await db.admin().command({hello:1})).$clusterTime.clusterTime

// 2. 全量扫
await db.users.find().forEach(syncDoc)

// 3. 从 now 开始增量
const stream = db.users.watch([], {startAtOperationTime: now})
```

要点：先记 timestamp 再全量扫，保证不漏（增量从全量开始的时间点续上）。

</details>

### Q9.10 ⭐⭐⭐ 单进程 Change Streams 吞吐瓶颈

<details><summary>答案</summary>

单 stream 是**单 cursor**，应用单线程消费。瓶颈：

- 处理单事件耗时
- 网络 RTT
- 下游写入速度

扩展：

- 单进程内 worker pool 并发处理
- 多 stream（不同 collection / db）
- 转发到 Kafka 让多消费者并行

不能"分片消费同一个 stream"。

</details>

---

## M10 Schema 设计

### Q10.1 ⭐ embed vs reference 三维度

<details><summary>答案</summary>

1. **关系基数**：1:1/1:few embed；1:massive / N:M reference
2. **访问模式**：常一起读 embed；常单独读 reference
3. **更新频率**：sub 静态 embed；sub 频繁单独改 reference

</details>

### Q10.2 ⭐⭐ Subset Pattern 解决

<details><summary>答案</summary>

商品有几千条评价，但首屏只显示 top 5：

```js
// 商品文档 embed top 5
{ _id: "prod-1", name: "...", reviewsTop5: [...] }
// 全量评价单独 collection
db.reviews.find({productId: "prod-1"})
```

效果：

- 商品页一次查询 + 5 条评价
- 详情页才走 reference 拉全部
- 不撑爆主文档

</details>

### Q10.3 ⭐⭐ Bucket Pattern 时序节省

<details><summary>答案</summary>

```
反例：1M 条 / 天的单独文档
推荐：每小时 / 每天聚合一个 bucket（3600 条进一个文档）
```

效果：

- 文档数从百万降到千
- 索引大小 1/3600
- 范围扫描快得多
- 单文档预聚合（min/max/avg）

</details>

### Q10.4 ⭐⭐ Outlier Pattern 适用

<details><summary>答案</summary>

99% 文档小、1% 巨大：

```js
// 普通用户：embed followers
// 巨人用户：followers 单独 collection
{ _id: "influencer-1", isOutlier: true, followerCollection: "followers_1" }
```

避免巨人撑爆主集合索引 / cache。

</details>

### Q10.5 ⭐ user 文档不应该 embed 什么

<details><summary>答案</summary>

- 订单列表（无限增长）
- 行为日志
- 大型文件 / 图片
- 频繁单独更新的子数据（导致写放大）

应该 embed：

- 用户基本信息
- 偏好设置（小且稳定）
- 最近 N 条订单（subset，可选）

</details>

### Q10.6 ⭐⭐⭐ 无限增长数组问题

<details><summary>答案</summary>

1. 16MB 撑爆
2. $push 重写整文档（写放大）
3. multi-key 索引爆
4. 数组遍历 / 排序慢
5. oplog entry 大（影响复制）

</details>

### Q10.7 ⭐⭐ computed pattern 写放大

<details><summary>答案</summary>

```js
// user 文档维护 totalSpent
db.users.updateOne(
  {_id: customerId},
  {$inc: {totalSpent: amount}}
)
```

每订单 → user doc 写一次。

代价：

- 高频订单时 user doc 热点
- 多 worker 并发 update 同 user → WriteConflict

权衡：读频繁 + 写不极端时，computed 是甜区。

</details>

### Q10.8 ⭐⭐ schema_version 演化

<details><summary>答案</summary>

```js
{ _id: 1, schemaVersion: 2, ... }
```

应用按 version 分支处理：

```js
function process(doc) {
  switch (doc.schemaVersion || 1) {
    case 1: return migrate(doc)
    case 2: return processV2(doc)
  }
}
```

效果：

- 不同时期写入的文档可共存
- 渐进式后台迁移
- 灰度发布 schema 升级

</details>

### Q10.9 ⭐⭐ validator strict / moderate 在迁移期

<details><summary>答案</summary>

迁移时改 schema：

```js
db.runCommand({
  collMod: "users",
  validator: { /* 新规则 */ },
  validationLevel: "moderate"   // 只验证已符合的文档的更新
})
```

老文档（不符合新规则）：

- 不影响（不触发验证）
- 直接更新仍可（如果改的字段符合验证）

后台慢慢迁老文档 → 全迁完后切回 strict。

</details>

### Q10.10 ⭐⭐⭐ 多租户 SaaS 3 种方案

<details><summary>答案</summary>

| 方案 | 适合 |
|---|---|
| 每租户独立 db | 大租户（< 100）/ 强隔离 |
| 共享 collection + tenantId 字段 | 海量小租户 / 节省管理 |
| Database 分组 + 共享集群 | 中等规模 |

实战：

- 几个大客户：独立 db
- SaaS 平台几千客户：共享 collection + shard key tenantId
- 极大租户（SAP-like）：独立集群

</details>

---

## M11 安全

### Q11.1 ⭐⭐ SCRAM vs x.509 适用

<details><summary>答案</summary>

- **SCRAM**：用户名 + 密码。人 / 应用账号
- **x.509**：证书认证。服务对服务（mTLS 配合）

经典组合：

- broker 间 mTLS（自动）
- 人 / kafkactl：SCRAM
- 服务 client：mTLS 或 SCRAM 看场景

</details>

### Q11.2 ⭐⭐ mTLS 比单向 TLS 多了

<details><summary>答案</summary>

- 单向 TLS：客户端验证服务端（防中间人）
- mTLS：双向都验证，**服务端也验证客户端身份**

mTLS 下证书 CN 直接做用户身份。

</details>

### Q11.3 ⭐⭐⭐ 最小授权具体做法

<details><summary>答案</summary>

1. 业务应用：只 readWrite 自己的 db
2. 报表应用：只 read
3. 管理员：分 userAdmin（管用户）+ dbAdmin（管 schema）+ root（紧急）
4. 不同业务模块用不同账号（出问题能审计到）
5. 定期 review 用户列表，删除离职 / 废弃账号
6. 别让 app 用 root（一旦泄露全集群完蛋）

</details>

### Q11.4 ⭐⭐ FLE Deterministic vs Random

<details><summary>答案</summary>

- **Deterministic**：同明文 → 同密文。**可等值查询**，能建索引。但密文模式可能泄露 frequency。
- **Random**：同明文 → 不同密文。**不能查询**，安全更高。

实战：

- 需要查询的字段（如 SSN 查询）→ Deterministic
- 完全只存不查的字段（如审计内容）→ Random
- 都想要 → Queryable Encryption（6.0+）

</details>

### Q11.5 ⭐⭐⭐ Queryable Encryption 解决 FLE 问题

<details><summary>答案</summary>

FLE Deterministic 同明文 → 同密文，server 看到密文 frequency 仍可推测明文（"这个密文出现 1 万次，可能是某常见值"）。

Queryable Encryption：

- 同明文 → 不同密文（防 frequency analysis）
- 但**仍支持等值 / 范围查询**（内部 secure index）

代价：性能慢 2-5×、存储 1.5×、6.0 Preview / 7.0 GA。

</details>

### Q11.6 ⭐ requireTLS vs preferTLS

<details><summary>答案</summary>

- **requireTLS**：必须 TLS（不允许明文）
- **preferTLS**：内部连接 TLS，外部可以明文（迁移期用）
- **allowTLS**：接受 TLS 和明文都可（启动期）
- **disabled**：完全无 TLS

生产 **requireTLS**。

</details>

### Q11.7 ⭐⭐ 静态加密 3 种取舍

<details><summary>答案</summary>

| 方式 | 优势 | 劣势 |
|---|---|---|
| OS / 磁盘加密 | 简单、对 DB 透明 | 内存中是明文 |
| WT 加密引擎 | DB 层加密，符合合规 | 企业版 |
| FLE / QE | 字段级，DB 都看不到明文 | 复杂、性能代价 |

实战：

- 起步：OS 加密
- 合规要求：+ WT 加密
- 内部威胁：+ FLE

</details>

### Q11.8 ⭐ 证书过期不轮换

<details><summary>答案</summary>

- mongod 证书过期 → 客户端连接失败
- mTLS 节点间证书过期 → 副本集自身通信挂

→ 集群业务全停。

预防：

- cert-manager 自动 renew
- 监控有效期 < 60 天 / 30 天 / 7 天告警
- 每年一次"模拟过期"演练

</details>

### Q11.9 ⭐⭐⭐ zone sharding 帮 GDPR

<details><summary>答案</summary>

GDPR 要求欧洲用户数据**驻留在欧洲**：

```js
sh.addShardToZone("sh-eu-1", "EU")
sh.updateZoneKeyRange("mydb.users",
  {region:"EU", _id:MinKey},
  {region:"EU", _id:MaxKey},
  "EU")
```

效果：region=EU 的用户数据强制落到 EU shard（部署在欧洲数据中心）。

满足"数据不出欧洲"的合规要求。

</details>

### Q11.10 ⭐⭐ 上线 5 个必做安全项

<details><summary>答案</summary>

1. **启用 authorization**（不开 auth 等于裸奔）
2. **bindIp 限制**（不暴露公网）
3. **TLS 强制**（requireTLS）
4. **最小授权**（app 用户只 readWrite 自己的 db）
5. **备份加密**（备份本身可能泄露）

附加：

- 防火墙规则
- 证书自动 renew
- 监控认证失败

</details>

---

## M12 运维 + 替代品

### Q12.1 ⭐ 日常巡检 5 个指标

<details><summary>答案</summary>

1. 副本集状态（rs.status，无 DOWN / ROLLBACK）
2. 复制 lag（< 10s）
3. 长跑 op（无 > 60s）
4. 连接数（不接近上限）
5. 磁盘使用率（< 80%）

</details>

### Q12.2 ⭐⭐ 3-2-1 备份法则

<details><summary>答案</summary>

- **3 份副本**
- **2 种介质**（如磁盘 + 对象存储）
- **1 份异地**（不同 region / 物理位置）

防灾难性丢失。

</details>

### Q12.3 ⭐⭐ mongodump vs 文件 snapshot

<details><summary>答案</summary>

| 方式 | 适合 |
|---|---|
| mongodump | 小集群 < 100GB / 单集合 export / 跨版本迁移 |
| 文件 snapshot | 中大集群、整集群备份、配合 PITR |

mongodump 慢但灵活；snapshot 快但需要文件级一致性（fsyncLock 或 LVM）。

</details>

### Q12.4 ⭐⭐⭐ PITR 实现 + 前提

<details><summary>答案</summary>

```
全量 snapshot（每天）+ oplog 持续保存 = 任意时间点恢复
```

前提：

- 持续 oplog 流（推到 S3 / Atlas / PBM）
- snapshot 与 oplog 时间对齐
- 工具支持（PBM / Atlas Backup）

恢复时：

1. 恢复最近 snapshot
2. apply oplog 到目标时间点

适合：误删 / 误改的恢复（"恢复到删除前 5 分钟"）。

</details>

### Q12.5 ⭐⭐ 副本集滚动升级顺序

<details><summary>答案</summary>

```
1. secondaries 一个一个（stop / 升级 / start / 等 SECONDARY）
2. stepDown primary → 让 secondary 接管
3. 升级原 primary
4. 升完所有节点后提 featureCompatibilityVersion
```

分片集群：先 config server → shards → mongos。

</details>

### Q12.6 ⭐⭐⭐ featureCompatibilityVersion 是什么

<details><summary>答案</summary>

fCV 控制 mongod 能用哪些"特定版本的功能"。即使升级了 binary，fCV 不提升的话仍按老版本行为运行（不开启新特性、新存储格式）。

```js
db.adminCommand({setFeatureCompatibilityVersion: "8.0"})
```

提升时机：

- 升级完所有节点
- 业务验证 1-2 周稳定
- 不需要回滚到老版本

提了之后不能轻易降。

</details>

### Q12.7 ⭐⭐ Atlas vs 自建

<details><summary>答案</summary>

| 维度 | 自建 | Atlas |
|---|---|---|
| 成本（云 VM） | 低 | 2-5× 溢价 |
| 运维 | 团队自己 | Atlas 团队 |
| 备份 / PITR / 升级 | 自己做 | 内置 |
| 灵活性 | 高 | 受限 |
| 适合 | 有 DBA / 大规模 | 没 DBA / 中等规模 |

</details>

### Q12.8 ⭐⭐⭐ AWS DocumentDB 与 MongoDB 差异

<details><summary>答案</summary>

- **底层不同**：DocumentDB 是 AWS 自研引擎运行在 Aurora 分布式存储上，不是真 MongoDB 代码（社区推测计算层基于 PostgreSQL，但未官方确认）
- **API 兼容性**：约 80%（兼容到 5.0 子集），部分聚合 stage / Change Streams 不支持或行为不同
- **运维**：AWS 托管，与 RDS 类似
- **价格**：中等

适合：已在 AWS、用 MongoDB 子集。

不适合：要新 MongoDB 特性（最新版本聚合 / vector search 等）、要 MongoDB 社区支持。

</details>

### Q12.9 ⭐⭐ FerretDB 核心优势

<details><summary>答案</summary>

- **完全开源 Apache 2.0**（避开 MongoDB SSPL）
- Wire protocol 兼容（驱动代码不改）
- 底层 PostgreSQL / SQLite（成熟 + 灵活）

适合：

- SSPL 协议不可接受（如商业转售 / SaaS 内嵌）
- 想统一 PostgreSQL 栈
- 中小规模

劣势：性能 / 功能完整性不如 MongoDB。

</details>

### Q12.10 ⭐ 被勒索 3 个根因

<details><summary>答案</summary>

1. **没开 authorization**（默认裸奔）
2. **bindIp 0.0.0.0 暴露公网**
3. **防火墙没限制 / 端口公开**

历史上几次 MongoDB 大规模被勒索都是这三点。

预防：永远不要"省事"——auth + TLS + 内网 是底线。

</details>

---

## 综合大题

### Q.综合.1 ⭐⭐⭐ 设计 1000 万用户 + 1 亿订单的 schema

<details><summary>答案</summary>

**集合**：

```js
// users（10M doc）
{
  _id: ObjectId("..."),
  email: "a@x.com",
  name: "Alice",
  profile: { /* 基本信息 */ },
  stats: { totalOrders: 42, totalSpent: 1500, lastOrderAt: ISODate("...") },   // computed
  createdAt: ISODate("..."), updatedAt: ISODate("...")
}

// orders（100M doc）
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  status: "paid",
  items: [...],      // embed
  total: NumberDecimal("..."),
  createdAt: ISODate("...")
}
```

**索引**：

```js
db.users.createIndex({email: 1}, {unique: true})
db.users.createIndex({"profile.country": 1, createdAt: -1})

db.orders.createIndex({userId: 1, createdAt: -1})    // 按用户查历史
db.orders.createIndex({status: 1, createdAt: -1})    // 按状态查
db.orders.createIndex({createdAt: -1})                // 全局时间
```

**部署**：

- 副本集 PSS（3 节点 × 大机器 64GB RAM）
- 数据量未到分片门槛

**ID 生成**：

- ObjectId（默认）
- 业务订单号单独字段（如 `orderNo: "2026051300001"`）+ unique index

**未来扩展**：

- orders 涨到 5 亿 → shard key `{customerId: 1, _id: 1}`
- users 涨到 1 亿 → shard key `{_id: "hashed"}`

</details>

### Q.综合.2 ⭐⭐⭐ 实现一个新闻 feed 实时推送

<details><summary>答案</summary>

**架构**：

```
User publishes → MongoDB posts collection
                    ↓ Change Stream
                 Fan-out Worker → user inbox / WebSocket push
```

**Schema**：

```js
// posts
{ _id: ObjectId("..."), authorId: ..., content: "...", at: ISODate("...") }

// inbox（per user）
{ _id: ObjectId(), userId: ..., postId: ..., readAt: null, at: ISODate("...") }
```

**Change Stream 消费**：

```js
const stream = db.posts.watch([{$match: {operationType: "insert"}}])
while (stream.hasNext()) {
  const event = stream.next()
  const post = event.fullDocument
  
  const followers = await getFollowers(post.authorId)
  
  // 写 inbox
  await db.inbox.bulkWrite(
    followers.map(uid => ({
      insertOne: { document: { userId: uid, postId: post._id, at: post.at } }
    })),
    { ordered: false }
  )
  
  // WebSocket 推送（在线用户）
  for (const uid of followers) {
    if (isOnline(uid)) io.to(uid).emit("post", post)
  }
  
  await saveToken(event._id)
}
```

**关键**：

- Change Stream resume token 持久化
- 大 V（million followers）用混合：fan-out on read（详见 M10）
- inbox 加 TTL 30 天自动清理

</details>

### Q.综合.3 ⭐⭐⭐ 集群突然 P99 飙升 10×，排查步骤

<details><summary>答案</summary>

**第一分钟**：

```js
db.serverStatus()   // overview
db.currentOp({secs_running:{$gt:5}})   // 长跑 op
rs.status()         // 复制状态
mongostat           // 实时
```

看：

- 是否有长跑慢查询占资源
- 是否在 election 切换
- 是否 cache pressure（dirty / used 高）

**第二步**：

```js
db.system.profile.find().sort({ts:-1}).limit(10)
// 看最近慢操作
```

**可能根因**：

1. **新业务上线 + 缺索引** → COLLSCAN
2. **数据增长 working set > cache** → cache miss 高
3. **后台报表跑大聚合** → 占 IO
4. **secondary failover** → 选举期间业务受影响
5. **磁盘 IO 满**（其他进程 / 共享存储抖动）
6. **网络问题**

**修复**：

- 慢查询 kill + 加索引
- 后台任务移到 secondary
- 资源不足 → 加 RAM / 升 SSD
- 长期：监控告警提前发现 working set 增长趋势

</details>

---

## 答题统计

| 章节 | 题数 | 难度分布 |
|---|---|---|
| M01 文档模型 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M02 索引 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M03 WiredTiger | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| M04 查询/聚合 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M05 副本集 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M06 分片 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M07 事务 | 10 | ⭐ 0 / ⭐⭐ 6 / ⭐⭐⭐ 4 |
| M08 性能 | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M09 Change Streams | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M10 Schema | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| M11 安全 | 10 | ⭐ 2 / ⭐⭐ 5 / ⭐⭐⭐ 3 |
| M12 运维+替代 | 10 | ⭐ 2 / ⭐⭐ 5 / ⭐⭐⭐ 3 |
| 综合 | 3 | ⭐⭐⭐ 3 |
| **合计** | **123** | — |

---

## 自评标准

- 答对 > 100 题：能独立设计 / 运维生产 MongoDB 集群
- 80-100：核心能力扎实，部分边角细节缺
- 60-80：还在中级，建议把不会的逐条研究
- < 60：从 M01-M02 重新读，把概念和实操结合

---

> 🔁 反馈：把不会的题写到笔记里，3 个月后再答一遍
