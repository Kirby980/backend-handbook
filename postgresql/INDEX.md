# PostgreSQL 深度课程 · 总目录

> 22 篇中文深度课程，聚焦 PostgreSQL 引擎内核与生产实践
> 每篇约 10000-15000 字，含源码级原理、Go 客户端示例、性能调优、生产陷阱、练习题
> 适合从中级到高级后端 / DBA / SRE / 平台工程师
>
> **📅 内容基准：PostgreSQL 18**（2025-09-25 发布，2026 主流稳定版）+ **PostgreSQL 17**（2024-09 发布，上一稳定版）
> ⚠️ PostgreSQL **无官方 "LTS" 概念**——每个大版本统一获 5 年支持（PG 17 至 2029-11、PG 18 至 2030-11）；"LTS" 仅见于第三方托管服务（如 AWS Aurora）。**2026 新项目应直接上 PG 18。**
> 涵盖 PG 18 异步 IO、incremental backup、UUIDv7、SQL/JSON、MERGE...RETURNING、虚拟生成列等新特性

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| P01 | [精通 PostgreSQL 整体架构](./01-精通-PostgreSQL-整体架构.md) | ⭐⭐⭐ | 进程模型 / postmaster / backend / 共享内存 / WAL / autovacuum |
| P02 | [精通堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md) | ⭐⭐⭐⭐ | heap / CTID / Tuple 格式 / TOAST / FILLFACTOR / HOT update |
| P03 | [精通 MVCC 与可见性](./03-精通-MVCC-与可见性.md) | ⭐⭐⭐⭐⭐ | xmin/xmax / CLOG / Snapshot / 可见性判定 / PG vs MySQL |
| P04 | [精通 B-tree 索引](./04-精通-B-tree-索引.md) | ⭐⭐⭐⭐⭐ | 页结构 / 唯一性 / INCLUDE / deduplication / bottom-up deletion |
| P05 | [精通 GIN/GiST/SP-GiST/BRIN/Hash 索引](./05-精通-多类型索引.md) | ⭐⭐⭐⭐⭐ | GIN / GiST / SP-GiST / BRIN / Hash / Bloom / 适用场景 |
| P06 | [精通 pgvector 与向量检索](./06-精通-pgvector-与向量检索.md) | ⭐⭐⭐⭐ | HNSW / IVFFlat / pgvectorscale / 量化 / RAG 实战 |
| P07 | [精通事务、隔离级别与锁](./07-精通-事务隔离与锁.md) | ⭐⭐⭐⭐⭐ | 4 隔离级别 / SSI / 行锁 / Advisory Lock / 死锁检测 |
| P08 | [精通 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md) | ⭐⭐⭐⭐⭐ | autovacuum / VACUUM FULL / wraparound / freezing / 膨胀监控 |
| P09 | [精通查询规划器](./09-精通-查询规划器.md) | ⭐⭐⭐⭐⭐ | cost model / pg_statistic / GEQO / planner method / 关联限制 |
| P10 | [精通 EXPLAIN 与执行计划](./10-精通-EXPLAIN-与执行计划.md) | ⭐⭐⭐⭐⭐ | EXPLAIN ANALYZE / BUFFERS / 各种 Scan / Join 算法 |
| P11 | [精通查询调优实战](./11-精通-查询调优实战.md) | ⭐⭐⭐⭐ | pg_stat_statements / 慢查询 / 并行查询 / JIT / 索引选择 |
| P12 | [精通 JSONB 与全文检索](./12-精通-JSONB-与全文检索.md) | ⭐⭐⭐⭐ | JSONB 索引 / jsonpath / tsvector / tsquery / ts_rank |
| P13 | [精通窗口函数、CTE 与高级 SQL](./13-精通-高级-SQL.md) | ⭐⭐⭐⭐ | 窗口函数 / LATERAL / WITH RECURSIVE / MERGE / GROUPING SETS |
| P14 | [精通分区表](./14-精通-分区表.md) | ⭐⭐⭐⭐ | 声明式分区 / 分区裁剪 / pg_partman / 分区维护 |
| P15 | [精通 WAL 与物理流复制](./15-精通-WAL-与物理流复制.md) | ⭐⭐⭐⭐⭐ | WAL 段 / Streaming Replication / Hot Standby / replication slot |
| P16 | [精通逻辑复制](./16-精通-逻辑复制.md) | ⭐⭐⭐⭐ | Logical Decoding / Pub/Sub / Debezium / 跨版本升级 |
| P17 | [精通高可用与连接池](./17-精通-高可用与连接池.md) | ⭐⭐⭐⭐ | Patroni / repmgr / PgBouncer / HAProxy / 故障切换 |
| P18 | [精通扩展生态](./18-精通-扩展生态.md) | ⭐⭐⭐⭐ | PostGIS / TimescaleDB / Citus / pg_partman / pg_cron / 扩展开发 |
| P19 | [精通 FDW 与异构集成](./19-精通-FDW-与异构集成.md) | ⭐⭐⭐ | postgres_fdw / 异构数据源 / 外部表 / 跨库查询 |
| P20 | [精通参数调优](./20-精通-参数调优.md) | ⭐⭐⭐⭐ | shared_buffers / work_mem / autovacuum / WAL / checkpoint |
| P21 | [精通监控与诊断](./21-精通-监控与诊断.md) | ⭐⭐⭐⭐ | pg_stat_* / wait events / pgBadger / 锁等待 / Prometheus exporter |
| P22 | [精通 PostgreSQL 18 / 17 新特性](./22-精通-PG18-17-新特性.md) | ⭐⭐⭐ | 异步 IO / incremental backup / UUIDv7 / SQL/JSON / 虚拟生成列 |

---

## 🗺️ 按模块组织

### 🟢 模块一：架构基础（P01-P03）

> PostgreSQL 是一个**多进程**、**堆表 + 独立索引**、**真正的 MVCC**（多版本物理共存）数据库——与 MySQL/InnoDB 的差异从这三点开始。

- **P01 整体架构**：postmaster + backend per connection、共享内存、WAL、autovacuum/bgwriter/checkpointer
- **P02 堆表与 TOAST**：堆表无聚簇索引、CTID、Tuple Header（24B）、TOAST 大字段存储、HOT update
- **P03 MVCC**：xmin/xmax 双版本号、CLOG 事务状态、Snapshot 构造、可见性判定（PG 真正的 MVCC vs MySQL undo-log MVCC）

### 🔵 模块二：索引体系（P04-P06）

> PG 拥有数据库界最丰富的索引家族——6 大类索引 + 表达式索引 + 部分索引 + 覆盖索引。

- **P04 B-tree**：页结构、唯一索引、INCLUDE 覆盖、PG 13 deduplication、PG 14 bottom-up deletion
- **P05 多类型索引**：GIN（倒排，JSONB/数组/全文）、GiST（通用，几何/范围/全文）、SP-GiST（空间分区）、BRIN（块范围，时序）、Hash（PG 10+ WAL 支持）、Bloom
- **P06 pgvector**：HNSW（PG 16+ 主推）、IVFFlat、pgvectorscale（StreamingDiskANN）、量化、与 RAG 集成

### 🟡 模块三：事务与 VACUUM（P07-P08）

> MVCC 的另一面：旧版本必须被清理，否则膨胀、wraparound 危机随之而来。

- **P07 事务与锁**：4 个隔离级别（含 PG 独有的 SSI 真可串行化）、行锁/表锁/Advisory Lock、死锁检测
- **P08 VACUUM**：autovacuum 触发条件、VACUUM (FULL/FREEZE/ANALYZE)、txid wraparound（"32 亿事务"危机）、可视化膨胀监控

### 🔴 模块四：查询优化（P09-P11）

> Planner 是 PostgreSQL 的灵魂——cost-based、统计驱动、无 hint 哲学。

- **P09 查询规划器**：cost model、pg_statistic（直方图 + MCV）、GEQO 遗传算法、planner 开关参数、为什么 PG 不支持 hint
- **P10 EXPLAIN**：EXPLAIN (ANALYZE, BUFFERS, VERBOSE)、各种 Scan Node、Join 算法（Nested Loop / Hash / Merge）
- **P11 查询调优**：pg_stat_statements、PARALLEL 查询、JIT 编译（PG 11+）、auto_explain、索引选择实战

### 🟣 模块五：数据类型与高级 SQL（P12-P14）

- **P12 JSONB 与全文检索**：JSONB 二进制存储、GIN 表达式索引、jsonpath（PG 12+ SQL/JSON）、tsvector/tsquery、相关性排序
- **P13 高级 SQL**：窗口函数全套、LATERAL、WITH RECURSIVE（图遍历/CTE 物化）、MERGE（PG 15+）、GROUPING SETS/ROLLUP/CUBE
- **P14 分区表**：声明式分区（PG 10+）、分区裁剪、分区维护（pg_partman）、与时序数据结合

### 🟠 模块六：复制与高可用（P15-P17）

- **P15 WAL 与物理复制**：WAL 段（16MB）、Streaming Replication、Hot Standby、同步/异步、replication slot、`pg_basebackup`
- **P16 逻辑复制**：Logical Decoding（PG 10+）、Publications/Subscriptions、CDC 与 Debezium、跨大版本零停机升级
- **P17 高可用**：Patroni（etcd/consul + raft）、repmgr、PgBouncer 连接池、HAProxy、故障切换演练

### 🟤 模块七：扩展生态（P18-P19）

> 扩展是 PostgreSQL 区别于一切关系库的杀手锏——`CREATE EXTENSION` 把 GIS、时序、列存、向量、分布式一键加入。

- **P18 扩展生态**：PostGIS（GIS 事实标准）、TimescaleDB（时序）、Citus（分布式分片）、pg_partman、pg_cron、自定义扩展开发
- **P19 FDW 与异构集成**：postgres_fdw、mysql_fdw、file_fdw、跨库 join、外部表 vs 物化视图

### 🔘 模块八：生产化（P20-P22）

- **P20 参数调优**：shared_buffers / work_mem / maintenance_work_mem / autovacuum / WAL / checkpoint / planner 参数
- **P21 监控与诊断**：pg_stat_database/activity/replication/io、wait events（PG 9.6+）、pgBadger 日志分析、Prometheus exporter
- **P22 PG 18/17 新特性**：异步 IO（PG 18）、incremental backup（PG 17）、UUIDv7（PG 18）、SQL/JSON 完整子集、虚拟生成列（PG 18）、MERGE...RETURNING（PG 17）

---

## 🎯 学习路径

### 路径 A：全面进阶（8-10 周）
按编号顺序通读。每篇配套：建表跑 EXPLAIN、采集 pg_stat、模拟一次故障/膨胀/复制中断。

### 路径 B：DBA 速成（2-3 周）
**P03 MVCC** + **P07 事务/锁** + **P08 VACUUM** + **P15 WAL/物理复制** + **P17 HA** + **P20 参数** + **P21 监控**——7 篇覆盖 DBA 日常 80%。

### 路径 C：应用开发者（1-2 周）
**P02 存储** + **P04 B-tree** + **P05 多类型索引** + **P10 EXPLAIN** + **P12 JSONB** + **P13 高级 SQL**——6 篇足以写出"专业"的应用代码。

### 路径 D：AI / RAG 工程师（1 周）
**P05 GIN/GiST** + **P06 pgvector** + **P12 JSONB** + **P18 扩展（pgvectorscale/TimescaleDB）** + 配合 ai-backend/A06-A07 RAG 章节。

### 路径 E：从 MySQL 迁移到 PostgreSQL（2 周）
**P01 架构对比** + **P03 MVCC**（关键差异） + **P08 VACUUM**（MySQL 没有的运维负担） + **P09-P10 Planner/EXPLAIN** + **P16 逻辑复制**（在线迁移路径） + **P22 18/17 新特性**。

### 路径 F：Operator / 平台工程师（2 周）
**P15-P17 复制 / HA** + **P18 扩展（Citus 分布式）** + **P20-P21 调优 / 监控** + 配合 cloud-native/C09 Operator 在 K8s 上跑 PostgreSQL（CloudNativePG / Crunchy / Zalando）。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[postgresql.org/docs/18/](https://www.postgresql.org/docs/18/) / [17](https://www.postgresql.org/docs/17/)
- **源码**：[github.com/postgres/postgres](https://github.com/postgres/postgres)（重点看 `src/backend/storage/`、`src/backend/access/heap/`、`src/backend/optimizer/`）
- **必读书**：
  - 《PostgreSQL Internals》（Egor Rogov，免费英文版，必读）
  - 《PostgreSQL 14 Internals》（中文译本良好）
  - 《Database Internals》（Alex Petrov，跨数据库视角）
- **社区博客**：
  - [pganalyze blog](https://pganalyze.com/blog)（业界顶级 PG 实战来源）
  - [Crunchy Data blog](https://www.crunchydata.com/blog)
  - [Citus Data blog](https://www.citusdata.com/blog/)
- **会议**：PGConf.EU / PGConf.NYC / PgCon

---

## 🛠️ 工具速查

| 任务 | 命令 / 工具 |
|---|---|
| 版本 | `SELECT version();` |
| 当前活动 | `SELECT * FROM pg_stat_activity;` |
| 锁等待 | `SELECT * FROM pg_locks WHERE NOT granted;` |
| 阻塞链 | `SELECT pg_blocking_pids(pid) FROM pg_stat_activity;` |
| 慢查询 | `pg_stat_statements` 扩展 + `SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;` |
| 解读 SQL | `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) SELECT ...` |
| 索引使用 | `SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0;`（未使用索引） |
| 表大小 | `SELECT pg_size_pretty(pg_total_relation_size('t'));` |
| 膨胀检测 | `pgstattuple` 扩展 / `pg_stat_user_tables.n_dead_tup` |
| autovacuum 状态 | `SELECT * FROM pg_stat_progress_vacuum;`（PG 9.6+） |
| 复制状态 | `SELECT * FROM pg_stat_replication;`（主库）/ `pg_stat_wal_receiver`（备库） |
| 复制延迟 | `SELECT now() - pg_last_xact_replay_timestamp();`（备库） |
| WAL 段位置 | `SELECT pg_current_wal_lsn();` |
| 物理备份 | `pg_basebackup -h primary -D /backup -X stream -P` |
| 逻辑备份 | `pg_dump -Fc -d mydb -f mydb.dump` / `pg_restore` |
| 重启服务 | `pg_ctl restart -D $PGDATA` / `systemctl restart postgresql` |
| 客户端 | `psql` / `pgcli`（自动补全增强） |
| 连接池 | `pgbouncer` / `pgcat`（Rust 实现，2024 兴起） |
| HA | `Patroni` / `repmgr` / `pg_auto_failover` |
| 监控 | `pg_stat_statements` / `auto_explain` / Prometheus `postgres_exporter` |
| K8s 部署 | `CloudNativePG`（CNCF Sandbox） / `Zalando postgres-operator` / `Crunchy PGO` |
| 模式迁移 | `Flyway` / `Liquibase` / `golang-migrate` / `Atlas` |
| 客户端库（Go） | `pgx`（推荐）/ `database/sql + lib/pq` / `gorm` |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 画出 PostgreSQL 进程模型（postmaster fork backend、共享内存区、各后台进程职责）
- [ ] 解释为什么 PG 是"真 MVCC"而 MySQL 是"undo log 模拟 MVCC"
- [ ] 写出 xmin/xmax 双版本号在 INSERT / UPDATE / DELETE 时的变化
- [ ] 描述堆表为什么没有聚簇索引（CTID 不稳定）
- [ ] 解释 HOT update 的触发条件和它如何减少索引膨胀
- [ ] 给一个 100 GB JSONB 表设计 GIN 表达式索引（按高频路径）
- [ ] 区分 GIN / GiST / SP-GiST / BRIN 各自的最佳场景
- [ ] 写一个 pgvector HNSW 索引并解释为什么向量推荐用 HNSW 而非 IVFFlat
- [ ] 解释 PG 的 SSI（可串行化快照隔离）和传统两阶段锁的区别
- [ ] 写一个会触发死锁的两并发事务对，并解释 PG 的检测机制
- [ ] 列出 autovacuum 的所有触发条件，解释 wraparound 危机
- [ ] 用 EXPLAIN (ANALYZE, BUFFERS) 读出一个 SQL 的实际执行情况，区分 estimate vs actual
- [ ] 解释为什么 PG 不支持 query hint（设计哲学）
- [ ] 用 LATERAL JOIN 重写一个 N+1 子查询为高效连接
- [ ] 用 WITH RECURSIVE 写出一个图遍历查询
- [ ] 用声明式分区把一个时序表按月切片，配合 pg_partman 自动维护
- [ ] 解释 WAL 段、checkpoint、archive_mode、replication slot 之间的关系
- [ ] 用 pg_basebackup + recovery.signal 搭建一个 Hot Standby
- [ ] 用逻辑复制做 PG 14 → PG 18 的零停机升级
- [ ] 用 Patroni + etcd 搭建一个自动故障切换的高可用集群
- [ ] 解释 PgBouncer 三种 pool_mode（session/transaction/statement）的差异和适用场景
- [ ] 用 PostGIS 写一个"找最近的 100 个 POI"查询
- [ ] 用 TimescaleDB 把一个亿级时序表压缩到原大小的 10%
- [ ] 给出 16 核 / 64 GB 服务器的合理 `shared_buffers` / `work_mem` / `effective_cache_size` 配置
- [ ] 通过 wait events 定位"某个时段慢"的瓶颈（IO/Lock/CPU/WAL）

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **P01 架构** | PG 18 异步 IO（io_uring）；PG 17 改进的并行 VACUUM |
| **P02 存储** | UUIDv7（PG 18 内置 `uuidv7()`，时序友好的 UUID）；TOAST 压缩可选 lz4（PG 14+ 默认 pglz） |
| **P03 MVCC** | xid 64bit 推进中（社区讨论）；FrozenXID 优化 |
| **P04 B-tree** | PG 17 改进的多列 B-tree skip scan（部分支持） |
| **P05 索引** | BRIN 进化（PG 14+ 多范围 BRIN）；Bloom 索引扩展成熟 |
| **P06 向量** | pgvector 0.7+（2024）支持 HNSW、二进制量化；pgvectorscale（Timescale 出品的 StreamingDiskANN）；与 RAG 主流栈集成 |
| **P07 事务** | SSI（PG 9.1+）已稳定，被 CockroachDB / YugabyteDB 借鉴 |
| **P08 VACUUM** | PG 17 改进 autovacuum 内存管理；wraparound 仍是 PG 头号"死亡警告"|
| **P09 Planner** | PG 16+ 改进的并行哈希连接；CTE 默认 inline（PG 12+） |
| **P10 EXPLAIN** | PG 16+ `EXPLAIN (GENERIC_PLAN)` 看 prepared plan |
| **P12 JSONB** | SQL/JSON 路径表达式（PG 12+）；PG 17 完整 SQL/JSON 子集（JSON_TABLE/JSON_VALUE/JSON_QUERY） |
| **P13 高级 SQL** | MERGE（PG 15+）；MERGE...RETURNING（PG 17） |
| **P14 分区** | 默认分区裁剪持续优化；Hash 分区（PG 11+）成熟 |
| **P15 WAL** | PG 17 incremental backup（`pg_basebackup --incremental`）；PG 16 双向流复制 |
| **P16 逻辑复制** | PG 16 双向逻辑复制；PG 17 故障切换不中断订阅；CDC 主流（Debezium + Kafka） |
| **P17 HA** | Patroni 4.x；CloudNativePG GA（CNCF Incubation 路上）；pgcat（Rust 连接池）崛起 |
| **P18 扩展** | TimescaleDB Hyperfunctions；Citus 12（PG 16 集成）；pgvector / pgvectorscale 火热 |
| **P20 参数** | PG 16+ `recovery_min_apply_delay`；jit_above_cost 调优经验积累 |
| **P22 新特性** | **PG 18 (2025-09)**：异步 IO、UUIDv7、虚拟生成列、改进的索引扫描；**PG 17 (2024-09)**：incremental backup、MERGE...RETURNING、SQL/JSON 完整子集 |

---

> 准备好你的 psql 和一个空集群（Docker：`docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=pwd postgres:18`），开始学习吧。
