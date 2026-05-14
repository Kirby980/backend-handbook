# 精通 NoSQL 选型

> 课程编号：B14
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — NoSQL Databases
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：没有银弹

```
"我们用 MongoDB"     → 嗨，是因为关系建模复杂吗？
"我们用 Cassandra"    → 真要全球分布 + 高写入吗？
"我们用 Redis"        → 缓存还是主存？
```

NoSQL 包含至少 5 大类，每类内又有多种产品。错选 = 几年的技术债。本章按数据模型分类，讲清各家擅长什么、为什么、何时选。

---

## 第一章：NoSQL 类别

### 1.1 五大类

| 类别 | 代表 | 数据模型 |
|---|---|---|
| **Key-Value** | Redis、Memcached、DynamoDB | key → value |
| **文档** | MongoDB、CouchDB | key → JSON-like 文档 |
| **列族** | Cassandra、HBase、ScyllaDB | (row, column) → value |
| **图** | Neo4j、ArangoDB、Dgraph | 节点 + 边 |
| **时间序列** | InfluxDB、TimescaleDB、Prometheus | timestamp → 度量 |

### 1.2 NoSQL vs SQL

NoSQL **不**是"没有 SQL"——是"不只是 SQL"或"非关系"。共同特征（多数）：
- 弱化 schema（schemaless 或 schema-on-read）
- 水平扩展友好
- 牺牲 ACID 换可用性 / 性能
- 没有跨表 JOIN（或弱）

---

## 第二章：Key-Value 存储

### 2.1 Redis

详见 B16。核心：内存数据库 + 丰富数据结构（string / list / set / hash / sorted set / stream）+ 持久化选项。

适合：
- 缓存
- session
- 排行榜（sorted set）
- 限流（INCR + EXPIRE）
- pub/sub
- 简单队列（stream）

不适合：
- 主数据库（持久化保证不如关系 DB）
- 数据量超过 RAM
- 复杂查询

### 2.2 Memcached

最纯粹的 KV cache。LRU、过期、无持久化。比 Redis 更简单——除了"分布式 KV cache"没别的事。

现在多数项目直接 Redis。Memcached 仍在大公司纯 cache 场景（极致性能）。

### 2.3 DynamoDB

AWS 托管。海量水平扩展 + per-request 计费 + 单 ms 延迟。

特性：
- 主键 = partition key (+ sort key)
- secondary index（local / global）
- streams（CDC）
- TTL
- DAX 内存加速

陷阱：
- 设计要绕着 access pattern 转（"单表设计"）
- 跨 partition query 慢 / 贵
- 强一致 read 额外费用

---

## 第三章：文档 DB

### 3.1 MongoDB

最流行。JSON-like BSON 文档。

```javascript
db.users.insertOne({
    name: "Alice",
    age: 30,
    addresses: [
        { city: "NYC", zip: "10001" },
        { city: "SF", zip: "94101" }
    ],
    metadata: { /* any nested */ }
});

db.users.find({ "addresses.city": "NYC" });
```

适合：
- 模式频繁演进
- 嵌套结构自然
- 读为主、按 ID 查整个对象
- 中等规模（< 数十 TB）

不适合：
- 复杂 JOIN / 多表事务
- 强一致严格要求
- schema 真的需要刚性约束

特性：
- 4.0+ 单文档事务
- 4.2+ 跨文档事务（性能较差）
- replica set + sharding
- aggregation pipeline（强大但 SQL 用户陡）

### 3.2 CouchDB

MVCC + HTTP API。同步双向（适合离线优先 app）。少数派但有 niche 用途（PouchDB 配合）。

### 3.3 PostgreSQL JSONB

技术上不是 NoSQL，但 jsonb + GIN 索引能做"文档 DB"的事。
**好处**：兼得关系 + 文档；事务、外键、JOIN 都在。
**何时不选 MongoDB 选 jsonb**：业务有强关系部分 + 灵活 metadata。

---

## 第四章：列族（Wide Column）

### 4.1 Cassandra

灵感来自 Google Bigtable + Dynamo。

数据模型：
```
keyspace
  └── table (column family)
        └── partition key → row(s)
              └── clustering keys
                    └── columns (values)
```

```sql
CREATE TABLE events (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);
```

- partition key 分片
- clustering key 排序
- 单 partition 内强排序

适合：
- 大写入吞吐（百万 QPS）
- 时间序列
- 已知 access pattern
- 多 DC 部署、最终一致
- 数据量 PB 级

不适合：
- ad-hoc 查询（必须按 primary key）
- 强一致
- 关系建模

特性：
- 无主架构（任何节点接读写）
- 一致性可调（ONE / QUORUM / ALL）
- LSM tree 存储（写快，读偶有 compaction 延迟）

### 4.2 HBase

Hadoop 生态。强一致 + 高吞吐。复杂度高，适合 Hadoop 已部署的场景。

### 4.3 ScyllaDB

Cassandra 兼容协议，C++ 实现 + shard-per-core 架构。性能 5-10x。

---

## 第五章：图 DB

### 5.1 Neo4j

```cypher
CREATE (alice:Person {name: "Alice"})-[:KNOWS]->(bob:Person {name: "Bob"})

MATCH (a:Person {name: "Alice"})-[:KNOWS*1..3]-(friend)
RETURN friend
```

适合：
- 社交网络
- 推荐系统（"喜欢这个的人还喜欢..."）
- 欺诈检测（关联分析）
- 知识图谱

特点：
- 关系是一等公民——存的是邻接，查询沿边跳
- 关系 DB 多表 JOIN 跳跃 5+ 跳 → 不可行；Neo4j 几毫秒

不适合：
- 简单 CRUD
- 大量普通业务数据
- 海量写入

### 5.2 ArangoDB / Dgraph

多模型（图 + 文档）。Dgraph 用 GraphQL 接口。

### 5.3 RedisGraph

Redis 模块，简单图查询。性能高，功能少。

---

## 第六章：时间序列 DB

### 6.1 InfluxDB

```
measurement,tag_key=tag_val field_key=field_val timestamp
cpu,host=server01 value=64.5 1652345678
```

适合：
- 监控指标
- IoT 传感器
- 金融 tick 数据
- 应用日志

特性：
- 写入压缩极高（按 timestamp 排序 + delta encoding）
- 自动 retention policy（30 天数据自动 drop）
- 内置 downsampling

### 6.2 TimescaleDB

PostgreSQL 扩展。继承 SQL + 关系——加时间序列优化。

```sql
CREATE TABLE metrics (
    time TIMESTAMPTZ,
    host TEXT,
    value DOUBLE PRECISION
);
SELECT create_hypertable('metrics', 'time');
```

适合：已用 PostgreSQL + 加入时间序列；不想换栈。

### 6.3 Prometheus

监控专用。pull 模式 + 自己 query 语言（PromQL）。短期存储（默认 15 天）。配合 Grafana 主流。

### 6.4 ClickHouse

技术上是列式 OLAP DB，但常用于时间序列（高写入 + 聚合查询）。强大但复杂。

---

## 第七章：搜索引擎

### 7.1 Elasticsearch

Lucene 之上的分布式全文搜索 + 分析。

适合：
- 全文搜索
- 日志聚合（ELK stack）
- 实时分析（聚合 query）

特性：
- 倒排索引
- 复杂 query DSL
- 接近实时（NRT, ~1s 延迟）
- 集群 + 分片

不适合：
- 主数据库（一致性弱）
- 事务

### 7.2 OpenSearch

Elasticsearch 的开源 fork（AWS 主导）。

### 7.3 Meilisearch / Typesense

轻量、单机搜索引擎。中小项目易部署。

### 7.4 Algolia

托管搜索。极快、贵。

---

## 第八章：选型决策树

### 8.1 流程

```
1. 数据有强关系吗？
   是 → 关系 DB（PostgreSQL）。结束。
   不确定 → 也选 PostgreSQL，jsonb 处理半结构化。
2. 量级？
   < 1 TB → 关系 DB
   ~ 数 TB → 关系 DB + 分片 或 NoSQL
   > PB → NoSQL（Cassandra、DynamoDB）
3. 访问模式？
   ad-hoc → 关系
   按 key 取整体 → 文档 DB / KV
   时间序列 → 专门 TS DB
   社交关系 → 图 DB
4. 一致性要求？
   强一致 → SQL / DynamoDB Strong / CockroachDB
   最终 → Cassandra / DynamoDB Eventually
5. 团队熟悉度？
   不要为了"新潮"上没人懂的技术
```

### 8.2 常见组合

实际系统常多技术栈共存：

```
PostgreSQL  - 主业务数据（用户、订单、产品）
Redis       - 缓存 + session + rate limit
Elasticsearch - 全文搜索 + 日志
InfluxDB / Prometheus - 监控
S3          - 文件、备份
Kafka       - 事件流
```

不要"all in 一个 DB"——但也别加新 DB 没必要。

---

## 第九章：迁移与共存

### 9.1 从 PostgreSQL 加 Redis

经典第一步——加缓存。读路径优先 Redis，miss 走 DB。

### 9.2 PostgreSQL → MongoDB

慎做。MongoDB 不支持 JOIN，业务有关系 → 各种打补丁。多数 case 加 jsonb 就够。

### 9.3 加搜索

Postgres full-text search 够用到几百万行。再大上 Elasticsearch。同步用 Debezium / Logstash 监听 binlog。

### 9.4 加分析

主库不胜分析查询 → ETL 到 OLAP（ClickHouse / BigQuery / Snowflake）。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：跟风
"X 公司用 Cassandra"——你不是 Netflix。先 PostgreSQL。

### ❌ 陷阱 2：MongoDB 当关系用
"users.orders[]" 嵌入数千订单 → 文档膨胀 + 16MB 限制。

### ❌ 陷阱 3：Cassandra 没设计 access pattern
"我后期会 JOIN"——Cassandra 不行。设计阶段就要明确所有 query。

### ❌ 陷阱 4：Redis 当主存
RAM 装不下 + 持久化保证弱 → 丢数据。

### ❌ 陷阱 5：Elasticsearch 当事务库
弱一致 + 偶尔丢消息。仅做搜索副本。

### ❌ 陷阱 6：用图 DB 处理 CRUD
关系 DB 跑得好的事，图 DB 复杂 10x。

### ❌ 陷阱 7：多 DB 同步噩梦
3 个数据库需同步 → CDC + Outbox 才不乱。

---

## 第十一章：生产级最佳实践

1. **默认 PostgreSQL**：90% 项目够。
2. **Redis 加在 PG 前**：缓存挡 90% 读。
3. **专用问题用专用 DB**：搜索 ES、监控 Prometheus、时间序列 InfluxDB。
4. **不要在生产玩新 DB**：先用 1 年 staging。
5. **JOIN 多的关系业务 → SQL**。
6. **设计阶段就定 access pattern**：Cassandra / DynamoDB 必须。
7. **多 DB 之间数据流明确**：Source of truth 是哪个？
8. **备份 + 恢复演练**：每种 DB 都要。
9. **监控存储增长 + 查询延迟**：早发现瓶颈。
10. **不要 lock-in 云厂商专有**：除非彻底承诺该云。

---

## 第十二章：常见陷阱清单（续）

### ❌ 陷阱 1：选型 PPT 漂亮
benchmark 是销售用，自己 POC。

### ❌ 陷阱 2：忽略运维
Cassandra 维护成本远超 PostgreSQL。

### ❌ 陷阱 3：以为 NoSQL 一定快
访问模式匹配才快；不匹配 NoSQL 比 SQL 更慢。

### ❌ 陷阱 4：跨 DB 一致性靠业务代码
缺事务 → 不一致状态。用 Outbox / Saga。

---

## 第十三章：练习题

**练习 1**：购物车 + 商品 + 订单 + 评论的系统，主 DB 选什么？

**练习 2**：用户登录场景需要什么数据库支持？

**练习 3**：IoT 传感器每秒 100 万写入，30 天保留。选什么？

**练习 4**：LinkedIn"X 度好友"功能用什么 DB？为什么？

**练习 5**：电商搜索"包含'red dress'+ 5 颗星好评 + 价格 < 100"的商品。怎么实现？

---

## 参考答案

**练习 1**：PostgreSQL。这是经典关系业务——用户/订单/商品多对多、JOIN、事务、强一致。再加 Redis 做 session / cart cache。

**练习 2**：
- 用户表 → PostgreSQL（强一致、关系）
- session token → Redis（短期、快、自带 TTL）
- rate limit → Redis（INCR + EXPIRE）

**练习 3**：InfluxDB / TimescaleDB / ClickHouse。原因：
- 时间序列优化压缩
- 自动 retention
- 高写入吞吐
- 时间窗口聚合 query 快

**练习 4**：Neo4j（图 DB）。关系 DB 跑 "find friends within 3 hops" 要 3 次自 JOIN，million 用户根本跑不通。图 DB 按邻接遍历几毫秒。

**练习 5**：Elasticsearch。多字段过滤 + 全文 + 排序 + 高亮 → ES 的 query DSL 强项。主库 PostgreSQL 存权威数据 + 通过 Debezium CDC 同步到 ES 搜索副本。

---

## 小结

| 类别 | 代表 | 何时选 |
|---|---|---|
| KV | Redis | 缓存、session、限流 |
| 文档 | MongoDB | 嵌套结构、模式演进 |
| 列族 | Cassandra | 高写入、海量、TS |
| 图 | Neo4j | 关系跳跃查询 |
| 时间序列 | InfluxDB / Timescale | 监控、IoT |
| 搜索 | Elasticsearch | 全文 + 复杂过滤 |
| 默认 | PostgreSQL | 90% 业务 |

下一篇 **B15 — 缓存策略** 将拆开 cache-aside / write-through / TTL / 失效策略 / 缓存雪崩穿透击穿。

---

