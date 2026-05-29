# MongoDB 深度课程 · 总目录

> 12 篇中文深度课程，聚焦 WiredTiger 引擎、副本集 / 分片集群与生产实践
> 每篇约 10000-15000 字，含底层结构、查询执行、性能调优、生产陷阱、练习题
> 适合从中级到高级后端 / DBA / SRE 工程师
>
> **📅 内容基准：MongoDB 8.0 LTS**（2024-10 GA，支持至 2029-10）
> 默认存储引擎 WiredTiger；本课程不涉及 MMAPv1 等过时引擎
> 同时覆盖 **MongoDB Atlas** 托管特性、字段级加密 (FLE)、Queryable Encryption、Time Series Collections

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| M01 | [精通文档模型与 BSON](./01-精通-文档模型-与-BSON.md) | ⭐⭐⭐ | document / BSON / 16MB / collection / namespace |
| M02 | [精通 MongoDB 索引体系](./02-精通-索引.md) | ⭐⭐⭐⭐⭐ | B-tree / compound / TTL / 地理 / 文本 / wildcard / vector |
| M03 | [精通 WiredTiger 存储引擎](./03-精通-WiredTiger.md) | ⭐⭐⭐⭐⭐ | LSM vs B-tree / cache / checkpoint / compression |
| M04 | [精通查询与聚合管道](./04-精通-查询与聚合管道.md) | ⭐⭐⭐⭐⭐ | find / aggregation / $lookup / $facet / explain |
| M05 | [精通副本集](./05-精通-副本集.md) | ⭐⭐⭐⭐⭐ | primary / secondary / oplog / election / read preference |
| M06 | [精通分片集群](./06-精通-分片集群.md) | ⭐⭐⭐⭐⭐ | mongos / config server / chunk / balancer / shard key |
| M07 | [精通事务与一致性](./07-精通-事务与一致性.md) | ⭐⭐⭐⭐⭐ | multi-doc tx / causal consistency / write concern / read concern |
| M08 | [精通 MongoDB 性能调优](./08-精通-性能调优.md) | ⭐⭐⭐⭐ | profiler / working set / index selection / hot collection |
| M09 | [精通 Change Streams 与 oplog](./09-精通-Change-Streams-与-oplog.md) | ⭐⭐⭐⭐ | oplog / change stream / resume token / Kafka 集成 |
| M10 | [精通 Schema 设计](./10-精通-Schema-设计.md) | ⭐⭐⭐⭐ | embed vs reference / 16MB / 反范式 / 演化 |
| M11 | [精通 MongoDB 安全](./11-精通-MongoDB-安全.md) | ⭐⭐⭐⭐ | SCRAM / x.509 / TLS / FLE / Queryable Encryption / RBAC |
| M12 | [精通 MongoDB 生产运维 + 替代品](./12-精通-生产运维-与-替代品.md) | ⭐⭐⭐⭐ | Atlas / Ops Manager / 备份 / FerretDB / DocumentDB |

---

## 🗺️ 按模块组织

### 🟢 模块一：数据模型（M01-M02）
> 文档 / BSON / 索引——所有性能与建模问题的物理根因。

### 🔵 模块二：存储与查询（M03-M04）
> WiredTiger + Aggregation Pipeline，是 MongoDB 区别于 MySQL 的核心引擎。

### 🟡 模块三：分布式（M05-M06）
> 副本集是高可用基础，分片集群是水平扩展终极方案。

### 🔴 模块四：一致性与性能（M07-M08）
> 事务 + read/write concern + 调优——生产线上必备。

### 🟠 模块五：现代特性与运维（M09-M12）
> Change Streams、Schema 设计、安全、Atlas / 替代品。

---

## 🎯 学习路径

### 路径 A：全面进阶（5 周）
按编号顺序通读。每篇配套：起一个 docker 副本集 + mongosh，按章节跑实验。

### 路径 B：应用开发速成（2 周）
**M01 文档模型** + **M02 索引** + **M04 查询/聚合** + **M07 事务** + **M10 Schema 设计** —— 5 篇覆盖应用侧 80%。

### 路径 C：DBA / SRE 特化（2 周）
**M03 WiredTiger** + **M05 副本集** + **M06 分片** + **M08 性能** + **M12 运维** —— 5 篇覆盖容量、稳定性、故障排查。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[mongodb.com/docs](https://www.mongodb.com/docs/manual/)
- **WiredTiger 文档**：[source.wiredtiger.com](http://source.wiredtiger.com/)
- **源码**：[github.com/mongodb/mongo](https://github.com/mongodb/mongo)
- **MongoDB University**：[learn.mongodb.com](https://learn.mongodb.com/)（官方免费课程）

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 连数据库 | `mongosh "mongodb://localhost:27017"` |
| 看集合大小 | `db.users.stats()` |
| 看索引 | `db.users.getIndexes()` |
| 看执行计划 | `db.users.find({age:25}).explain("executionStats")` |
| 创建索引 | `db.users.createIndex({email:1},{unique:true})` |
| TTL 索引 | `db.sessions.createIndex({createdAt:1},{expireAfterSeconds:3600})` |
| 副本集状态 | `rs.status()` |
| 副本集配置 | `rs.conf()` |
| 当前 op | `db.currentOp()` |
| 杀掉 op | `db.killOp(opid)` |
| 慢查询开启 | `db.setProfilingLevel(1, { slowms: 100 })` |
| 查慢查询 | `db.system.profile.find().sort({ts:-1}).limit(10)` |
| 分片状态 | `sh.status()` |
| 启动分片 | `sh.enableSharding("mydb")` + `sh.shardCollection("mydb.coll", {key:1})` |
| 看 chunks | `db.getSiblingDB("config").chunks.find()` |
| change stream | `db.users.watch([{$match:{operationType:"insert"}}])` |
| 备份 | `mongodump --uri="..." --out=/backup` |
| 恢复 | `mongorestore --uri="..." /backup` |
| 集合压缩 | `db.runCommand({compact: "coll"})` |

---

## ✅ 完读检查清单

- [ ] 解释 BSON 与 JSON 的关键差异（类型、二进制）
- [ ] 区分覆盖索引、复合索引前缀、ESR 规则
- [ ] 说明 WiredTiger cache、checkpoint、journal 三层关系
- [ ] 设计一个 read-mostly 业务的副本集 + read preference 配置
- [ ] 选 shard key 的三大原则（基数 / 频率 / 单调性）
- [ ] 解释 write concern w:majority + read concern majority/snapshot 的协同
- [ ] 用 $lookup + $facet 实现 ES-like 多维聚合
- [ ] 设计一个 oplog window 30 天的生产配置
- [ ] 区分 embed / reference / bucket / 反范式
- [ ] 启用 Queryable Encryption 做"加密下仍能查询"
- [ ] 选 MongoDB Atlas / DocumentDB / FerretDB 并说出理由

---

> 🔁 反馈：`docker run mongo:8.0` 单实例先跑通，再起 3 节点副本集
