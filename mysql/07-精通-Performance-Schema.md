# 精通 Performance Schema 与慢日志：digest、instruments、sys schema

> 关联章节：[M03 事务](./03-精通-InnoDB-事务-MVCC.md)、[M06 优化器](./06-精通查询优化与-EXPLAIN.md)、[M08 调优](./08-精通-Buffer-Pool-调优.md)

---

## 引言：MySQL 内置的"火焰图"

很多 MySQL 用户排查问题止步于 `SHOW PROCESSLIST` 和 `slow_log`。但 5.7+ 的 **Performance Schema (PS)** 已经能给出几乎所有内部细节：每条 SQL 跑了多久、等了什么锁、I/O 多少次、用了什么 buffer。

但 PS 表很多（5.7 有 80+，8.0 有 110+），表名长得像数据库内部仪表盘——大多数人看一眼就放弃。这章把 PS 拆开讲，配合 `sys` schema（5.7+ 自带的"翻译层"），让你能在 30 秒内定位一次慢查询。

读完之后你应该能：

- 区分 PS 的几大类 instruments（statements / waits / stages / memory）
- 用 `events_statements_summary_by_digest` 找 top SQL
- 用 `events_waits_summary_global_by_event_name` 找 I/O / 锁等待热点
- 用 sys schema 一句 `SELECT * FROM sys.statements_with_runtimes_in_95th_percentile`
- 配合慢日志做"补遗式"分析
- 排查一次 P99 飙升

---

## 第一章：Performance Schema 是什么

### 1.1 一句话定义

PS 是**MySQL 自身的仪表盘**——不像 INFORMATION_SCHEMA（数据库元信息），PS 记录的是"运行时事件"：

- 每条 SQL 执行了多久
- 哪些线程在等什么锁 / I/O
- 内存分配多少
- 文件 I/O 多少次

PS 表是内存表（PERFORMANCE_SCHEMA 引擎），重启清零。

### 1.2 启用与开销

```ini
performance_schema = ON   # 默认 ON（5.7+）
```

开销：

- 5.6 之前：5-20% 性能损失
- 5.7+：常用 instruments 默认开，损失 < 5%
- 8.0：进一步优化，可控制单 instrument 开关

不要因为"性能损失"关 PS——损失远小于排不出问题的代价。

### 1.3 instruments 与 consumers

PS 是**生产者-消费者**模型：

- **instruments**（探针）：埋点位置（如 `wait/io/file/innodb/innodb_data_file`）
- **consumers**（消费者）：要不要把这些事件存下来

```sql
SELECT * FROM performance_schema.setup_instruments LIMIT 10;
SELECT * FROM performance_schema.setup_consumers;
```

要某个细分数据，得对应 instrument + consumer 都开。

---

## 第二章：核心表分类

### 2.1 setup_* —— 配置

```sql
SHOW TABLES LIKE 'setup%';
-- setup_actors / setup_consumers / setup_instruments
-- setup_objects / setup_threads
```

### 2.2 events_*_current —— 当前事件

每个线程当前正在做什么：

```sql
SELECT * FROM performance_schema.events_statements_current LIMIT 5;
```

### 2.3 events_*_history —— 最近 N 条事件

```sql
SELECT * FROM performance_schema.events_statements_history;
-- 默认每线程保留 10 条
```

### 2.4 events_*_history_long —— 最近 N 条全局

跨线程的更长历史。

### 2.5 events_*_summary_by_*  —— 按某维度聚合

最常用——按 digest / by host / by user 等维度聚合。

```sql
SELECT * FROM performance_schema.events_statements_summary_by_digest;
SELECT * FROM performance_schema.events_waits_summary_global_by_event_name;
```

### 2.6 表的"全名结构"

```
events_<类别>_<时态>_by_<维度>
       statements / waits / stages / transactions
       current / history / history_long / summary
       digest / thread / host / user / event_name
```

---

## 第三章：Statement 事件 —— SQL 执行分析

### 3.1 当前正在执行的 SQL

```sql
SELECT
    THREAD_ID, EVENT_NAME, SQL_TEXT,
    TIMER_WAIT/1e9 AS elapsed_ms
FROM performance_schema.events_statements_current
WHERE SQL_TEXT IS NOT NULL;
```

比 `SHOW PROCESSLIST` 的 `INFO` 列更全。

### 3.2 Top SQL by 总耗时

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR AS exec_count,
    SUM_TIMER_WAIT/1e12 AS total_sec,
    AVG_TIMER_WAIT/1e9 AS avg_ms,
    SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0) AS rows_ratio
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### 3.3 Top SQL by 单次耗时

```sql
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    AVG_TIMER_WAIT/1e9 AS avg_ms,
    MAX_TIMER_WAIT/1e9 AS max_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE COUNT_STAR > 100
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

### 3.4 digest 是什么

DIGEST = SQL 的"模板哈希"。同一模板不同参数算同一 digest：

```
SELECT * FROM users WHERE id = 1
SELECT * FROM users WHERE id = 2
SELECT * FROM users WHERE id = 100
                   → 同一 digest
```

让你能聚合"同一类 SQL 的总开销"，非常关键。

### 3.5 排查"为什么慢"

```sql
SELECT
    DIGEST_TEXT,
    AVG_ROWS_EXAMINED,
    AVG_ROWS_SENT,
    AVG_TIMER_WAIT/1e9 AS avg_ms,
    SUM_NO_INDEX_USED,    -- 没用索引的次数
    SUM_NO_GOOD_INDEX_USED,
    SUM_SELECT_FULL_JOIN, -- 全表 JOIN
    SUM_SORT_MERGE_PASSES -- 排序合并次数
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE 'SELECT%'
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

---

## 第四章：Wait 事件 —— 等待分析

### 4.1 全局等待 top

```sql
SELECT
    EVENT_NAME,
    COUNT_STAR,
    SUM_TIMER_WAIT/1e12 AS total_sec,
    AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle'
  AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC LIMIT 20;
```

输出 EVENT_NAME 形如：

- `wait/io/file/innodb/innodb_data_file` — InnoDB 数据文件 I/O
- `wait/io/file/innodb/innodb_log_file` — redo I/O
- `wait/synch/mutex/innodb/buf_pool_mutex` — buffer pool 互斥锁
- `wait/synch/rwlock/innodb/btr_search_latch` — 自适应哈希索引锁

### 4.2 大类汇总

```sql
SELECT SUBSTRING_INDEX(EVENT_NAME, '/', 3) AS category,
       SUM(SUM_TIMER_WAIT)/1e12 AS total_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle'
GROUP BY category
ORDER BY total_sec DESC;
```

得到 I/O / 锁 / 互斥锁 哪类是最大瓶颈。

### 4.3 单线程等待 detail

```sql
SELECT
    THREAD_ID, EVENT_NAME,
    TIMER_WAIT/1e9 AS ms
FROM performance_schema.events_waits_current
WHERE THREAD_ID = <your_thread>;
```

排查"这个连接现在卡在哪"。

---

## 第五章：File I/O 事件

### 5.1 哪个表 I/O 最重

```sql
SELECT
    OBJECT_SCHEMA, OBJECT_NAME,
    COUNT_READ, SUM_TIMER_READ/1e9 AS read_ms,
    COUNT_WRITE, SUM_TIMER_WRITE/1e9 AS write_ms
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY (SUM_TIMER_READ + SUM_TIMER_WRITE) DESC LIMIT 10;
```

### 5.2 哪个文件 I/O 最重

```sql
SELECT
    FILE_NAME,
    COUNT_READ, SUM_NUMBER_OF_BYTES_READ / 1024 / 1024 AS read_MB,
    COUNT_WRITE, SUM_NUMBER_OF_BYTES_WRITE / 1024 / 1024 AS write_MB
FROM performance_schema.file_summary_by_instance
ORDER BY (SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE) DESC LIMIT 10;
```

输出可能：

```
ibdata1                       2 GB read,    100 MB write
ibtmp1                        500 MB read,  100 MB write
binlog.000123                 0 read,         5 GB write
ib_logfile0                   100 MB read, 10 GB write
```

binlog / redo 写入量大是正常；ibtmp（临时表落盘）大是异常信号。

### 5.3 Index 使用情况

```sql
SELECT
    OBJECT_SCHEMA, OBJECT_NAME, INDEX_NAME,
    COUNT_STAR AS uses,
    COUNT_FETCH, COUNT_INSERT, COUNT_UPDATE, COUNT_DELETE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA NOT IN ('mysql','performance_schema','sys')
  AND INDEX_NAME IS NOT NULL
ORDER BY COUNT_STAR DESC;
```

`COUNT_STAR=0` 的 index → 候选删除。

---

## 第六章：Stage 事件 —— SQL 内部阶段

```sql
SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT/1e9 AS total_ms
FROM performance_schema.events_stages_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

输出 stage 形如：

- `stage/sql/optimizing` — 优化器
- `stage/sql/executing` — 执行
- `stage/sql/Sending data` — 发送结果
- `stage/sql/converting HEAP to ondisk` — 临时表落盘
- `stage/innodb/alter table (read PK and internal sort)` — DDL 阶段

DDL 慢时 stage 能告诉你卡在哪一步。

---

## 第七章：Transaction 事件

```sql
-- 当前活跃事务
SELECT
    THREAD_ID, EVENT_ID, STATE, TRX_ID,
    GTID, ISOLATION_LEVEL, ACCESS_MODE,
    TIMER_WAIT/1e9 AS elapsed_ms
FROM performance_schema.events_transactions_current;

-- 事务汇总
SELECT
    COUNT_STAR, SUM_TIMER_WAIT/1e12 AS total_sec
FROM performance_schema.events_transactions_summary_global_by_event_name;
```

适合排查长事务、隔离级别使用情况。

---

## 第八章：锁与元数据

### 8.1 当前锁

```sql
-- 8.0+
SELECT * FROM performance_schema.data_locks WHERE OBJECT_NAME = 'orders';
SELECT * FROM performance_schema.data_lock_waits;
```

### 8.2 元数据锁

```sql
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = 'mydb';
```

### 8.3 看锁等待链

```sql
SELECT
    requesting_engine_transaction_id AS waiting_trx,
    blocking_engine_transaction_id AS blocking_trx,
    requesting_thread_id, blocking_thread_id,
    locked_table, locked_index
FROM performance_schema.data_lock_waits w
JOIN performance_schema.data_locks bl ON bl.engine_lock_id = w.blocking_engine_lock_id
JOIN performance_schema.data_locks rq ON rq.engine_lock_id = w.requesting_engine_lock_id;
```

---

## 第九章：内存与连接

### 9.1 全局内存使用

```sql
SELECT
    EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED / 1024 / 1024 AS MB
FROM performance_schema.memory_summary_global_by_event_name
WHERE CURRENT_NUMBER_OF_BYTES_USED > 1024 * 1024
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 20;
```

输出：

```
memory/innodb/buf_buf_pool          16384 MB
memory/innodb/log_buffer              64 MB
memory/sql/THD::main_mem_root         48 MB
memory/sql/User_var_entry              8 MB
memory/temptable/physical_disk        5 MB
```

排查 OOM 的金指标。

### 9.2 连接 / 线程

```sql
SELECT
    USER, HOST, COUNT_STAR,
    SUM_CONNECTIONS, CURRENT_CONNECTIONS
FROM performance_schema.accounts;

SELECT * FROM performance_schema.threads WHERE TYPE = 'FOREGROUND';
```

---

## 第十章：sys schema —— PS 的人性化封装

### 10.1 sys schema 是什么

5.7+ 自带，是一组"基于 PS 的视图 + 存储过程"，把繁琐的 PS 表查询包装成可读 SQL。

### 10.2 常用视图

```sql
-- Top SQL
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile;
SELECT * FROM sys.statement_analysis;
SELECT * FROM sys.statements_with_full_table_scans;
SELECT * FROM sys.statements_with_temp_tables;
SELECT * FROM sys.statements_with_sorting;
SELECT * FROM sys.statements_with_errors_or_warnings;

-- 表 / 索引
SELECT * FROM sys.schema_table_statistics;
SELECT * FROM sys.schema_unused_indexes;     -- 没用过的索引
SELECT * FROM sys.schema_redundant_indexes;  -- 冗余索引

-- IO
SELECT * FROM sys.io_global_by_file_by_bytes;
SELECT * FROM sys.io_global_by_wait_by_latency;

-- 锁
SELECT * FROM sys.innodb_lock_waits;

-- 内存
SELECT * FROM sys.memory_global_by_current_bytes;
SELECT * FROM sys.memory_global_total;

-- 主机 / 用户
SELECT * FROM sys.host_summary;
SELECT * FROM sys.user_summary;
```

### 10.3 实战：5 条速查

```sql
-- 1. 最近最慢的 SQL
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 5;

-- 2. 没用过的索引（candidates to drop）
SELECT * FROM sys.schema_unused_indexes;

-- 3. 哪个表 I/O 最重
SELECT * FROM sys.io_global_by_file_by_bytes ORDER BY total DESC LIMIT 5;

-- 4. 当前锁等待
SELECT * FROM sys.innodb_lock_waits;

-- 5. 连接 + 资源用量
SELECT * FROM sys.host_summary;
```

5 条 SQL 涵盖大部分日常诊断。

---

## 第十一章：慢查询日志（slow_log）

### 11.1 配置

```ini
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 0.5            # 0.5 秒以上算慢
log_queries_not_using_indexes = ON
log_throttle_queries_not_using_indexes = 60   # 同 digest 1 分钟最多记一次
log_slow_admin_statements = ON
log_slow_replica_statements = ON
```

### 11.2 vs Performance Schema

| | slow_log | PS digest |
|---|---|---|
| 实时性 | 写文件，离线分析 | 在线 SQL 查 |
| 完整性 | 单条文本 + 详细指标 | 聚合 |
| 容易聚合 | 用 pt-query-digest 解析 | 直接 SQL |
| 性能影响 | 写盘 | 内存表 |

**两者互补**——PS 看趋势 / 排名，slow log 看具体某条 SQL 的全文 + 指标。

### 11.3 pt-query-digest 分析

```bash
pt-query-digest /var/log/mysql/slow.log > report.txt
```

输出 top SQL by 总耗时 + 详细 explain / 时间分布。比手动看 slow log 强 10 倍。

---

## 第十二章：典型排查流程

### 12.1 P99 飙升

```sql
-- 1. 确认是哪类 SQL
SELECT DIGEST_TEXT, COUNT_STAR, AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE LAST_SEEN > NOW() - INTERVAL 5 MINUTE
ORDER BY MAX_TIMER_WAIT DESC LIMIT 5;

-- 2. 当前等待事件
SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT/1e9
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME != 'idle' AND COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC LIMIT 5;

-- 3. 锁等待
SELECT * FROM sys.innodb_lock_waits;

-- 4. 内存 / 连接
SELECT * FROM sys.memory_global_total;
SHOW STATUS LIKE 'Threads_connected';
```

### 12.2 内存涨

```sql
-- 哪类内存涨
SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED / 1024 / 1024 AS MB
FROM performance_schema.memory_summary_global_by_event_name
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC LIMIT 10;

-- 是不是某个连接特别多
SELECT * FROM sys.memory_by_thread_by_current_bytes
ORDER BY current_allocated DESC LIMIT 5;
```

### 12.3 磁盘 I/O 高

```sql
-- 哪个文件
SELECT * FROM sys.io_global_by_file_by_bytes ORDER BY total DESC LIMIT 10;

-- 哪个 SQL 触发大量 I/O
SELECT DIGEST_TEXT, SUM_ROWS_EXAMINED, SUM_NO_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_ROWS_EXAMINED DESC LIMIT 5;
```

### 12.4 长事务排查

```sql
SELECT trx_id, trx_started, trx_query, trx_rows_locked, trx_rows_modified
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 30;
```

---

## 第十三章：监控集成

### 13.1 Prometheus mysqld_exporter

主流监控栈直接拿 PS 的关键指标，无需手动 SQL：

- `mysql_perf_schema_events_statements_total`
- `mysql_perf_schema_events_waits_seconds_total`
- `mysql_perf_schema_table_io_waits_total`
- `mysql_perf_schema_file_events_total`
- `mysql_perf_schema_memory_events_alloc_bytes`

Grafana dashboard 直接可视化 top SQL / I/O / 锁等待。

### 13.2 PMM (Percona Monitoring and Management)

集成 PS + slow log + OS 指标的开源方案，比自建简单。

### 13.3 自定义指标采集

```sql
SELECT
    'slow_sql_count' AS metric,
    COUNT(*) AS value
FROM performance_schema.events_statements_summary_by_digest
WHERE AVG_TIMER_WAIT > 1e9;   -- > 1 秒
```

接入 Prometheus pushgateway。

---

## 第十四章：注意事项

### 14.1 PS 是内存表，重启清零

排查"昨天的某次卡顿"——PS 帮不了你，得靠：

- 慢日志归档
- 监控系统的历史数据
- binlog（如果是 DML）

### 14.2 Statement digest 数量上限

```ini
performance_schema_digests_size = 10000   # 默认
```

超过 → 新 digest 进 `STATEMENT_DIGEST_LOST`。SQL 多样性高（如 SQL 拼字符串而非 prepared）→ digest 表会无效溢出。

修复：

- 用 prepared statement
- 增大 `performance_schema_digests_size`

### 14.3 events_statements_history_long 写满

默认 10000 条循环。要更长历史用 `_history_long_size` 调大或外部归档。

### 14.4 不要在 PS 上加复杂 SQL

PS 表本身就是性能数据，对它跑复杂 SQL 反而拖慢监控。**简单 SELECT + LIMIT**。

---

## 第十五章：一次完整诊断 demo

### 15.1 业务报警："某 API P99 从 200ms 飙到 2s"

```sql
-- 1. 看慢 SQL
SELECT DIGEST_TEXT, AVG_TIMER_WAIT/1e9 AS avg_ms, COUNT_STAR
FROM performance_schema.events_statements_summary_by_digest
WHERE LAST_SEEN > NOW() - INTERVAL 5 MINUTE
ORDER BY MAX_TIMER_WAIT DESC LIMIT 5;

-- 找到嫌疑 SQL：
-- SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20
-- avg_ms = 1500, count = 1000
```

### 15.2 EXPLAIN

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 20\G

-> Limit: 20 row(s)  (cost=...) (actual time=1500.0..1500.5 rows=20)
   -> Sort: orders.created_at DESC, limit input to 20 rows per chunk
      -> Filter: orders.user_id = 12345
         -> Table scan on orders  (rows=10000000)  ← 全表扫！
```

发现：走全表扫 + filesort。

### 15.3 看索引

```sql
SHOW INDEX FROM orders;
-- 没有 idx_user_created
-- 只有 PRIMARY 和 idx_status
```

### 15.4 看 PS 印证

```sql
SELECT INDEX_NAME, COUNT_STAR
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_NAME = 'orders';
-- PRIMARY: 几亿
-- idx_status: 几千
-- (NULL 全表): 1000  ← 全表扫
```

### 15.5 修复

```sql
CREATE INDEX idx_user_created ON orders(user_id, created_at DESC);
```

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 12345 ORDER BY created_at DESC LIMIT 20\G

-> Limit: 20 row(s)
   -> Index range scan on orders using idx_user_created (cost=...) (actual time=0.5..1.5 rows=20)
```

P99 回到 30ms。

### 15.6 持续监控

```sql
SELECT DIGEST_TEXT, AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%orders%user_id%'
ORDER BY LAST_SEEN DESC LIMIT 5;
```

---

## 总结

Performance Schema + sys schema + slow log 是 MySQL 的诊断三件套。关键点：

1. **PS 表分类**：events_*_summary_by_*  最常用
2. **digest**：SQL 模板哈希，聚合的核心
3. **wait events**：I/O / 锁 / 互斥锁等待 top
4. **sys schema**：人性化视图，30 秒诊断
5. **slow log + pt-query-digest**：补遗
6. **典型流程**：digest → EXPLAIN ANALYZE → 索引 → 验证
7. **监控集成**：mysqld_exporter / PMM
8. **PS 是内存的**：重启清零，长期靠监控系统

---

## 练习题

1. 用 `events_statements_summary_by_digest` 找出全库 top 5 慢 SQL。
2. 对每个慢 SQL，从 PS 找出"扫描行 / 返回行"比，判断是否走错索引。
3. 用 `sys.schema_unused_indexes` 找出未用索引，估算删除收益。
4. 区分 wait/io/file 与 wait/synch/mutex，举出每类的代表事件。
5. 用 `events_waits_summary_global_by_event_name` 判断当前瓶颈是 I/O 还是锁。
6. 慢日志和 PS digest 的差异？什么场景该用 slow log 不用 PS？
7. 配置 `performance_schema_digests_size` 不够时怎么排查？
8. 一段 SQL 用 prepared 与不用 prepared，PS 里的 digest 数量差异？
9. 排查"内存涨而 buffer pool 不变"——具体看哪个 PS 表？
10. 写一个查询：找出最近 1 小时内 SQL 执行次数 / 总耗时增长最快的 5 个 digest。

---

## 参考答案

**1.** top 5 慢 SQL（按总耗时）：
```sql
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT/1e12 AS total_sec,
       AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 5;
```
按单次耗时则改 `ORDER BY AVG_TIMER_WAIT DESC`。

**2.** 用扫描行/返回行比判断是否走错索引：
```sql
SELECT DIGEST_TEXT,
       SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT,0) AS rows_ratio,
       SUM_NO_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 5;
```
`rows_ratio` 远大于 1（如扫 10 万返回 10）或 `SUM_NO_INDEX_USED > 0` → 大概率没走/走错索引，需 EXPLAIN 进一步确认。

**3.** `SELECT * FROM sys.schema_unused_indexes` 列出运行期间 `count_star=0` 的索引。删除收益：减少每次 DML 维护该索引的写开销、节省索引磁盘与 Buffer Pool 空间。需注意：实例运行时间要足够长、覆盖完整业务周期（含月底报表等低频查询），避免误删。

**4.** `wait/io/file/...` 是**文件 I/O 等待**（代表事件 `wait/io/file/innodb/innodb_data_file` 数据文件读写、`.../innodb_log_file` redo 写）；`wait/synch/mutex/...` 是**互斥锁等待**（代表 `wait/synch/mutex/innodb/buf_pool_mutex` buffer pool 锁、`wait/synch/rwlock/innodb/btr_search_latch` AHI 锁）。前者反映磁盘瓶颈，后者反映内部并发竞争。

**5.** 判断瓶颈是 I/O 还是锁：
```sql
SELECT SUBSTRING_INDEX(EVENT_NAME,'/',3) AS category,
       SUM(SUM_TIMER_WAIT)/1e12 AS total_sec
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE EVENT_NAME!='idle'
GROUP BY category ORDER BY total_sec DESC;
```
看 `wait/io/...` 类总耗时大 → I/O 瓶颈；`wait/synch/mutex|rwlock/...` 大 → 锁/互斥竞争瓶颈。

**6.** slow log vs PS digest 差异：slow log 写文件、记录单条 SQL 全文 + 完整执行指标、可离线用 pt-query-digest 分析、能保留历史（归档后重启不丢）；PS digest 是内存中按模板聚合的实时排名、重启清零。**该用 slow log 不用 PS 的场景**：需要回溯历史（如分析昨天某次卡顿）、需要单条 SQL 的完整文本与上下文、或要用 pt-query-digest 做离线深度报告。

**7.** `performance_schema_digests_size` 不够排查：查 `SHOW GLOBAL STATUS LIKE 'Performance_schema_digest_lost'`（或看 digest 表是否出现汇总到 `STATEMENT_DIGEST` 溢出行），若 lost 持续增长说明 digest 种类超上限。根因常是 SQL 未用 prepared statement、把字面值拼进 SQL 导致模板爆炸。修复：改用参数化/prepared statement，或调大 `performance_schema_digests_size`。

**8.** prepared vs 非 prepared 的 digest 数量：用 prepared statement（`WHERE id=?`）时不同参数归并为**同一个 digest**；不用 prepared、把值拼进 SQL（`WHERE id=1`、`id=2`…）虽然 digest 归一化也会把字面值替换为 `?`，但若 SQL 文本结构因拼接产生差异（如 IN 列表长度不同、拼了不同表名/列名）则产生**大量不同 digest**，易撑爆 digest 表。参数化能稳定收敛 digest 数量。

**9.** "内存涨而 buffer pool 不变"看 `performance_schema.memory_summary_global_by_event_name`（按 EVENT_NAME 找哪类内存增长，如 `memory/sql/...`、`memory/temptable/...`），再用 `sys.memory_by_thread_by_current_bytes` 看是不是某个连接/线程占用激增。常见元凶：临时表、用户变量、连接级缓冲、JS/UDF。

**10.** 最近增长最快的 digest——PS 是累计值，需两次快照求差，或用 `LAST_SEEN`/`FIRST_SEEN` 近似：
```sql
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT/1e12 AS total_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE LAST_SEEN > NOW() - INTERVAL 1 HOUR
ORDER BY COUNT_STAR DESC LIMIT 5;
```
更精确做法：定时把该表 COUNT_STAR / SUM_TIMER_WAIT 落到历史表，按时间窗口做差值排序，得到真正的"增长最快"。

---

> 🔁 反馈：把这章里的 SQL 全部跑一遍，挑出 3 条最有用的写到团队 wiki
