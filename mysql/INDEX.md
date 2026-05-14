# MySQL 深度课程 · 总目录

> 12 篇中文深度课程，聚焦 InnoDB 引擎与生产实践
> 每篇约 10000-15000 字，含源码级原理、性能调优、生产陷阱、练习题
> 适合从中级到高级后端 / DBA / SRE 工程师
>
> **📅 内容基准：MySQL 8.4 LTS**（2024-04 发布，长期支持至 2032）+ **MySQL 9.x 创新版**（2024-07 起持续迭代）
> 默认存储引擎 InnoDB；本课程不涉及 MyISAM / Memory 等过时引擎

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| M01 | [精通 MySQL 整体架构](./01-精通-MySQL-整体架构.md) | ⭐⭐⭐ | 连接器 / 解析器 / 优化器 / 执行器 / 引擎接口 |
| M02 | [精通 InnoDB B+ 树索引](./02-精通-InnoDB-索引.md) | ⭐⭐⭐⭐⭐ | 聚簇 vs 二级 / 页结构 / ICP / MRR / 覆盖索引 |
| M03 | [精通 InnoDB 事务与 MVCC](./03-精通-InnoDB-事务-MVCC.md) | ⭐⭐⭐⭐⭐ | undo / redo / ReadView / 隔离级别实现 |
| M04 | [精通 MySQL 锁机制](./04-精通-MySQL-锁.md) | ⭐⭐⭐⭐⭐ | 行锁 / 间隙锁 / next-key / 意向锁 / 死锁 |
| M05 | [精通 Binlog 与复制](./05-精通-Binlog-与复制.md) | ⭐⭐⭐⭐ | row/statement/mixed / GTID / 半同步 / 并行复制 |
| M06 | [精通查询优化器与 EXPLAIN](./06-精通查询优化与-EXPLAIN.md) | ⭐⭐⭐⭐⭐ | cost model / 直方图 / hash join / type/key/extra |
| M07 | [精通 Performance Schema 与慢日志](./07-精通-Performance-Schema.md) | ⭐⭐⭐⭐ | digest / instruments / sys schema / 慢日志分析 |
| M08 | [精通 InnoDB 参数调优与 Buffer Pool](./08-精通-Buffer-Pool-调优.md) | ⭐⭐⭐⭐ | buffer_pool_size / log file / flush / dirty page |
| M09 | [精通 MySQL 高可用架构](./09-精通-MySQL-高可用.md) | ⭐⭐⭐⭐ | MGR / InnoDB Cluster / Router / ProxySQL |
| M10 | [精通 JSON、窗口函数与 CTE](./10-精通-JSON-窗口函数.md) | ⭐⭐⭐ | json type / json_table / window / recursive CTE |
| M11 | [精通 MySQL 9 新特性](./11-精通-MySQL-9-新特性.md) | ⭐⭐⭐ | vector type / event tracking / privileges |
| M12 | [精通分库分表与中间件](./12-精通-MySQL-分库分表.md) | ⭐⭐⭐⭐ | Vitess / ShardingSphere / 分区表 / 一致性 hash |

---

## 🗺️ 按模块组织

### 🟢 模块一：基础架构（M01-M02）
> SQL 怎么进来、走过哪些层、最终落到 InnoDB 的 B+ 树上。

### 🔵 模块二：事务与并发（M03-M04）
> ACID 不是口号——是 undo/redo/MVCC/锁 四件套联动的具体机制。

### 🟡 模块三：高可用（M05、M09）
> Binlog 是基础，复制是常态，MGR 是 2026 年的主流。

### 🔴 模块四：性能（M06-M08）
> EXPLAIN 读懂 → Performance Schema 定位 → Buffer Pool 调优。

### 🟠 模块五：现代特性与扩展（M10-M12）
> JSON、向量类型、分库分表——MySQL 在云原生时代的延伸。

---

## 🎯 学习路径

### 路径 A：全面进阶（6-8 周）
按编号顺序通读。每篇配套：建表跑 EXPLAIN、采集 Performance Schema、模拟一次故障。

### 路径 B：DBA 速成（2 周）
**M02 索引** + **M03 事务** + **M04 锁** + **M05 复制** + **M08 调优** + **M09 高可用**——6 篇覆盖 DBA 日常 80%。

### 路径 C：开发面试（1 周）
**M02 索引** + **M03 事务/MVCC** + **M04 锁** + **M06 EXPLAIN**——4 篇覆盖后端面试高频题。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[dev.mysql.com/doc/refman/8.4/en/](https://dev.mysql.com/doc/refman/8.4/en/) / [9.x](https://dev.mysql.com/doc/refman/9.0/en/)
- **源码**：[github.com/mysql/mysql-server](https://github.com/mysql/mysql-server)（重点看 `storage/innobase/` 和 `sql/optimizer/`）
- **Percona 博客**：[percona.com/blog](https://www.percona.com/blog/)（业界最权威的 MySQL 实战来源）

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 看版本 / 配置 | `SELECT VERSION();` / `SHOW VARIABLES LIKE '...';` |
| 看引擎状态 | `SHOW ENGINE INNODB STATUS\G` |
| 看锁等待 | `SELECT * FROM performance_schema.data_locks;` |
| 看事务 | `SELECT * FROM information_schema.innodb_trx;` |
| 解读 SQL | `EXPLAIN FORMAT=TREE SELECT ...` (8.0+) / `EXPLAIN ANALYZE` |
| 跑直方图 | `ANALYZE TABLE t UPDATE HISTOGRAM ON col WITH 100 BUCKETS;` |
| 查找慢 SQL | `SELECT * FROM performance_schema.events_statements_summary_by_digest ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;` |
| Binlog dump | `mysqlbinlog --base64-output=DECODE-ROWS -v binlog.000001` |
| GTID 状态 | `SHOW MASTER STATUS\G` / `SHOW REPLICA STATUS\G` |
| Buffer Pool 状态 | `SELECT * FROM information_schema.innodb_buffer_pool_stats;` |
| 在线 DDL | `ALTER TABLE t ADD COLUMN ..., ALGORITHM=INSTANT;` (8.0+) |
| Clone Plugin（搭从库） | `CLONE INSTANCE FROM 'user'@'host':3306 IDENTIFIED BY 'pwd';` |
| pt-toolkit | `pt-query-digest` / `pt-online-schema-change` |

---

## ✅ 完读检查清单

- [ ] 画出 InnoDB 页（16KB）的内部结构（File Header / Page Header / 用户记录 / Page Directory）
- [ ] 解释 RR 隔离级别下幻读如何被 gap lock + next-key lock 阻止
- [ ] 写出一个会产生死锁的两并发事务对，并解释 InnoDB 怎么检测
- [ ] 区分 `EXPLAIN` 输出里的 `Using index` / `Using where` / `Using filesort` / `Using temporary`
- [ ] 解释 GTID 与传统 file/position 复制相比解决了哪些问题
- [ ] 给一个 10 亿行表的常见查询设计索引（含覆盖索引、最左前缀）
- [ ] 区分 statement-based / row-based binlog 各自陷阱
- [ ] 给热点行更新（如秒杀库存）设计一个不引入死锁的方案
- [ ] 用 Performance Schema 定位一次 P99 飙升
- [ ] 评估 MySQL 8.4 LTS vs MySQL 9.x 创新版的选型

---

> 📁 本目录位于 `/data/workspace/dp4/mysql/INDEX.md`
> 🔁 反馈：本地装一个 MySQL 8.4，每节都跑一遍命令再下结论
