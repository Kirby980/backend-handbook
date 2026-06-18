# 精通 PostgreSQL VACUUM 与表膨胀

> 关联章节：[P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)、[P02 堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md)、[P07 事务隔离与锁](./07-精通-事务隔离与锁.md)、[P20 参数调优](./20-精通-参数调优.md)

---

## 引言：VACUUM 是 MVCC 的代价

PG 的 MVCC 是真·多版本——`UPDATE` 不会原地修改，而是把旧 tuple 标记 `xmax`、在堆表（或同页）写一条新 tuple；`DELETE` 也不删，只是标记 `xmax`。这带来惊人的并发能力（读永远不阻塞写），但留下一个甩不掉的后遗症：**旧版本要谁清理？**

答案是 **VACUUM**。这是 PG 与 MySQL InnoDB 最大的运维差异：

- **InnoDB** 把旧版本放进 undo log（系统表空间或独立 undo 表空间），purge 线程后台清理。事务结束后 undo 也基本可丢；旧版本对堆表没影响。
- **PG** 把旧版本直接堆在表里（同一个堆文件），与新版本物理混居。如果不清理，**表越来越大，索引越来越大，磁盘越来越爆**——这就是**表膨胀（bloat）**。

加上 **事务 ID 仅 32 位（约 21 亿）**，PG 用循环计数；旧事务必须在 wraparound 之前被 freeze，否则数据会"穿越时空"。这是 PG DBA 心中的头号红色警告。

读完这章你应该能：

- 解释 VACUUM 做的 4 件事，每件事的成本与产物
- 区分 VACUUM / VACUUM FULL / pg_repack 三者各自的代价与适用场景
- 列出 autovacuum 的所有触发条件，并能针对一张 10TB 表设计 per-table 参数
- 描述 32-bit xid wraparound 危机，画出 freeze 三层阈值
- 找出阻塞 vacuum 的三大杀手（长事务、prepared txn、未消费 slot）
- 用 SQL 监控表膨胀，并给出 200GB 表 80% dead tuple 的在线处理方案

---

## 第一章：为什么需要 VACUUM

### 1.1 PG MVCC 留下的"垃圾"

回顾 [P03](./03-精通-MVCC-与可见性.md) 的关键事实：

| 操作 | 堆表里发生什么 |
|---|---|
| INSERT (id=10) | 写新 tuple T0，`xmin=100, xmax=0` |
| UPDATE (id=10) | T0.xmax=200，再写新 tuple T1，`xmin=200, xmax=0` |
| DELETE (id=10) | T0/T1 中最新可见的那条 `xmax=300` |

事务结束后：

- 已 commit 的 UPDATE：旧版本 T0 对所有事务都不可见（dead），堆里浪费一个槽位
- 已 commit 的 DELETE：所有版本都 dead
- 已 abort：新版本 dead，旧版本仍是"活的"

如果不清理，1 亿次 UPDATE 同一行 → 堆表里有 1 亿个 dead tuple，索引也对应膨胀。

### 1.2 VACUUM 与 InnoDB Purge 对比

| 维度 | PG VACUUM | InnoDB Purge |
|---|---|---|
| 旧版本存储 | 堆表内 | Undo log（独立表空间） |
| 清理对象 | 堆表 + 索引 + visibility map | Undo log + 索引 |
| 频率 | autovacuum 触发，可调 | purge 线程后台跑 |
| 是否回收磁盘 | VACUUM 不回收物理空间，只标记重用；VACUUM FULL 才回收 | 不回收（undo 表空间可单独 truncate） |
| 阻塞性 | 普通 VACUUM 不阻塞 DML；VACUUM FULL 持 AccessExclusive | 几乎不阻塞 |
| 运维负担 | 高（DBA 必须懂） | 低（默认就好） |
| 触发危机 | wraparound、bloat、long-running txn | 极少 |

**简而言之**：PG 用空间换时间（旧版本就在原地，访问快），InnoDB 用时间换空间（旧版本搬到 undo，访问慢但堆表干净）。

### 1.3 dead tuple 的代价

| 影响 | 说明 |
|---|---|
| 堆表膨胀 | 每个 dead tuple 占 24B header + 数据；100 万 dead 约 50-100MB |
| 索引膨胀 | 索引也指向 dead tuple（除非 HOT update），扫描时 visibility check 才发现 dead |
| 顺序扫描慢 | 扫描必须读 dead tuple 才能判断 |
| Index-only scan 失效 | VM 没标记 all-visible → 必须回堆表 |
| 选择性估计偏差 | planner 看 pg_class.reltuples / relpages 算出错误的代价 |
| WAL 暴涨 | 后续 VACUUM 需要写大量 WAL 来清理 |

---

## 第二章：VACUUM 做的 4 件事

### 2.1 任务清单

普通 `VACUUM`（非 FULL）做四件事：

1. **回收 dead tuple 槽位**（line pointer 重用）
2. **更新 visibility map（VM）** → 支持 index-only scan
3. **更新 free space map（FSM）** → 后续 INSERT 知道哪些页有空间
4. **freeze 老 tuple** → 推进 relfrozenxid，远离 wraparound

如果带 `ANALYZE`，再加一件：更新统计信息（pg_statistic）。

### 2.2 工作流程（7 个阶段）

```
[0] initializing
    准备阶段：获取 ShareUpdateExclusive 锁，初始化内部状态

[1] scanning heap
    扫描堆表，找到 dead tuple，收集它们的 CTID
    若使用并行：每个 worker 扫描一段

[2] vacuuming indexes
    对每个索引：读所有叶子，删除指向 dead CTID 的索引条目
    若使用并行：每个 index 一个 worker（PG 13+）

[3] vacuuming heap
    回到堆表，把第一阶段记下的 dead tuple 真正释放
    （line pointer 标记 LP_DEAD，槽位可重用）

[4] cleaning up indexes
    对每个索引做最终清理（B-tree 空页回收等）

[5] truncating heap (可选)
    如果表末尾有大段空页，截断文件返还磁盘给 OS
    持 AccessExclusive 锁瞬间，可能引起业务感知

[6] performing final cleanup
    清理 FSM、更新 pg_class 统计、上报统计信息
```

`pg_stat_progress_vacuum`（PG 9.6+）实时展示当前阶段。

### 2.3 visibility map（VM）

每个堆文件对应一个 `_vm` 后缀的 fork，每页 2 bit：

- `ALL_VISIBLE`：本页所有 tuple 对所有事务可见 → index-only scan 不用回堆表
- `ALL_FROZEN`：本页所有 tuple 都已 freeze → 后续 VACUUM 可跳过本页

`SELECT * FROM pg_visibility('t'::regclass);`（pg_visibility 扩展）可看每页状态。

VACUUM 更新 VM 是 **Index-Only Scan 能否生效** 的关键——没跑过 VACUUM 的表，即使你建了覆盖索引，PG 依然要回堆表。

### 2.4 free space map（FSM）

每个堆文件对应一个 `_fsm` fork，记录每页有多少空闲空间。新 INSERT 用这个图找"有空位的页"。

- VACUUM 不显式跑也会更新 FSM（heap 写入路径）
- VACUUM FREEZE / FULL 重写完成后 FSM 重建
- FSM 损坏 → 新插入只追加到表末尾 → 表暴涨

### 2.5 Freeze

老 tuple（xmin/xmax 远古事务）需要 freeze，即把 xmin 标记 `FrozenTransactionId`（一种特殊标记，对所有事务都"已提交可见"）。Freeze 后即使 xid wraparound，这条 tuple 也永远可见。详见第六章。

---

## 第三章：VACUUM 命令与 ANALYZE

### 3.1 三种调用

```sql
-- 标准 VACUUM（不带 FULL）：在线、不阻塞 DML
VACUUM t;
VACUUM (VERBOSE, ANALYZE) t;   -- 输出详细日志 + 更新统计
VACUUM (FREEZE) t;              -- 主动 freeze 所有 tuple
VACUUM (FULL) t;                -- 重写表，持 AccessExclusive 锁
VACUUM (PARALLEL 4) t;          -- 并行索引清理（PG 13+）
VACUUM (DISABLE_PAGE_SKIPPING) t;  -- 强制扫描所有页（用于损坏修复）
VACUUM (INDEX_CLEANUP OFF) t;   -- 跳过索引清理（紧急加速 wraparound 抢救）
VACUUM (PROCESS_TOAST OFF) t;   -- 不处理 TOAST
VACUUM (SKIP_LOCKED) t1, t2, t3;  -- 表上有冲突锁就跳过（PG 12+）

-- 仅更新统计，不清理
ANALYZE t;
ANALYZE (VERBOSE) t (col1, col2);   -- 仅 col1, col2

-- 强行重组（重写表 + 按索引排序）
CLUSTER t USING idx;            -- 等价于 VACUUM FULL + 按索引顺序
```

### 3.2 VACUUM vs VACUUM FULL

| 维度 | VACUUM | VACUUM FULL |
|---|---|---|
| 锁等级 | ShareUpdateExclusive（不阻塞 DML） | AccessExclusive（阻塞一切，包括 SELECT） |
| 物理空间 | 仅标记可重用，不返还磁盘 | 重写整表，返还磁盘 |
| 时间 | 增量进行 | 与表大小成正比，可能小时级 |
| 中断 | 可随时取消 | 取消后已写的临时文件浪费 |
| 索引 | 原地清理 | 全部重建 |
| 适用 | 日常 | 极端膨胀紧急修复 |

**生产严禁**：在大表上跑 `VACUUM FULL`。请用 `pg_repack` 或 `pg_squeeze`（详见第八章）。

### 3.3 ANALYZE 与 VACUUM 解耦

ANALYZE 独立于 VACUUM：

- ANALYZE 只采样行，更新 pg_statistic（直方图、MCV、相关性、null 比例）
- 默认样本：`default_statistics_target = 100` → 300 × 100 = 30000 行采样
- 大表上 ANALYZE 很快（采样而非全扫）

autoanalyze 触发条件（与 autovacuum 并列）：

```
n_mod_since_analyze > autovacuum_analyze_scale_factor × n_live_tup + autovacuum_analyze_threshold
```

默认 `scale_factor=0.1, threshold=50`。

---

## 第四章：autovacuum —— 后台清理守护

### 4.1 进程模型

```
postmaster
  ├─ autovacuum launcher（常驻，每 autovacuum_naptime 唤醒一次）
  │    └─ 决定哪些库该跑 autovacuum
  └─ autovacuum worker（按需 fork，最多 autovacuum_max_workers 个）
       └─ 对单表执行 VACUUM / ANALYZE
```

每个数据库每 `autovacuum_naptime`（默认 1min）唤醒一次；worker 处理完一张表后退出，launcher 决定下一个目标。

### 4.2 触发条件

`pg_stat_user_tables` 维护四个关键字段：

- `n_live_tup`：估算活 tuple 数
- `n_dead_tup`：估算 dead tuple 数
- `n_ins_since_vacuum`：上次 vacuum 后的插入数（PG 13+）
- `n_mod_since_analyze`：上次 analyze 后的修改数

**VACUUM 触发**：

```
n_dead_tup > autovacuum_vacuum_scale_factor × n_live_tup
           + autovacuum_vacuum_threshold
```

默认：`scale_factor=0.2, threshold=50`。即一张表 1000 万行，需要 200 万 dead 才触发——对热表偏保守。

**INSERT-only VACUUM 触发**（PG 13+，专门针对仅插入的大表）：

```
n_ins_since_vacuum > autovacuum_vacuum_insert_scale_factor × n_live_tup
                   + autovacuum_vacuum_insert_threshold
```

默认：`insert_scale_factor=0.2, insert_threshold=1000`。意义：仅 INSERT 不会产生 dead tuple，但需要 freeze 推进 + 更新 VM 让 index-only scan 生效。PG 13 之前的版本，纯 INSERT 表永远不触发 autovacuum，导致 wraparound 危险。

**ANALYZE 触发**：

```
n_mod_since_analyze > autovacuum_analyze_scale_factor × n_live_tup
                    + autovacuum_analyze_threshold
```

默认：`analyze_scale_factor=0.1, analyze_threshold=50`。

### 4.3 关键参数详解

```ini
# 总开关
autovacuum = on

# launcher 唤醒间隔
autovacuum_naptime = 1min

# 同时跑的 worker 数（不要超过 CPU/4）
autovacuum_max_workers = 3       # 默认 3，大型集群可调 5-10

# VACUUM 触发阈值
autovacuum_vacuum_scale_factor = 0.2     # 默认偏保守
autovacuum_vacuum_threshold = 50

# INSERT-only VACUUM（PG 13+）
autovacuum_vacuum_insert_scale_factor = 0.2
autovacuum_vacuum_insert_threshold = 1000

# ANALYZE 触发阈值
autovacuum_analyze_scale_factor = 0.1
autovacuum_analyze_threshold = 50

# 限流：每次 IO 后休息（避免压垮磁盘）
autovacuum_vacuum_cost_delay = 2ms       # PG 12+ 默认 2ms，之前是 20ms
autovacuum_vacuum_cost_limit = -1        # -1 = 跟 vacuum_cost_limit（默认 200）

# Freeze 相关（详见第六章）
autovacuum_freeze_max_age = 200000000    # 2 亿
autovacuum_multixact_freeze_max_age = 400000000
```

### 4.4 大表上覆盖 per-table 参数

默认参数对 10TB 大表极不合理（要积累 2TB dead 才触发）。生产做法：

```sql
ALTER TABLE big_table SET (
    autovacuum_vacuum_scale_factor = 0.02,         -- 2% dead 就跑
    autovacuum_vacuum_threshold = 10000,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_insert_scale_factor = 0.02,
    autovacuum_vacuum_cost_delay = 0,              -- 不限速
    autovacuum_vacuum_cost_limit = 10000,          -- 大幅放开
    autovacuum_freeze_min_age = 50000000,
    autovacuum_freeze_table_age = 100000000
);
```

**经验法则**：

- 表越大，scale_factor 越小（不能等 20%）
- 表越热，threshold 越大（避免 vacuum 抖动）
- 大表单独提升 cost_limit / 关闭 cost_delay
- 强 SLA 的表（计费表）→ scale_factor 0.01

### 4.5 检查 autovacuum 是否被禁用

```sql
SELECT relname, reloptions
FROM pg_class
WHERE reloptions IS NOT NULL;

-- 看到 {autovacuum_enabled=false} 说明被人手动关了——危险！
```

某些"性能优化"教程教人关 autovacuum 跑批量任务，结果**永远忘了再开回来**——线上有过表膨胀 500% 的案例。

---

## 第五章：cost-based vacuum 限流

### 5.1 为什么需要限流

VACUUM 是后台任务，不能与业务争 IO。PG 用"成本计数器"模型：

- 每次操作积累一个 cost
- 累计达到 `vacuum_cost_limit` → sleep `vacuum_cost_delay` ms → 计数器清零

成本表（默认）：

| 操作 | 成本 |
|---|---|
| `vacuum_cost_page_hit` | 1（页在 shared_buffers） |
| `vacuum_cost_page_miss` | 2（PG 14 前是 10；PG 14+ 改为 2）|
| `vacuum_cost_page_dirty` | 20（弄脏一页） |

### 5.2 计算实际 IO 限速

PG 14+ 默认：`cost_limit=200, cost_delay=2ms` → 每 2ms 处理 200 cost。

如果全是 cache miss（cost=2/page）：
- 200 cost / 2ms = 100 pages / 2ms = 50,000 pages/s
- × 8KB = 400 MB/s

如果全 dirty（cost=20/page）：
- 200 / 20 = 10 pages / 2ms = 5,000 pages/s
- × 8KB = 40 MB/s

PG 14 之前 cost_delay 默认 20ms，慢 10 倍——这是为什么旧版本经常"autovacuum 跑不完"。

### 5.3 调优建议

| 场景 | cost_delay | cost_limit |
|---|---|---|
| 默认 OLTP | 2ms | 200 |
| 大表急救 | 0 | 10000 |
| 弱磁盘服务器 | 5ms | 200 |
| SSD / NVMe 服务器 | 1ms | 1000 |
| ALTER TABLE 单表覆盖 | 0 | 10000 |

---

## 第六章：32-bit xid wraparound 危机

### 6.1 为什么 32 位是炸弹

xid 是 4 字节无符号整数，最大约 42 亿，但 PG 用其中一半做"过去/未来"判断（visibility 比较），有效空间 21 亿。

事务消耗 xid 的速度：

| QPS | 每天消耗 xid |
|---|---|
| 1000 写事务/秒 | 86M |
| 10000 写事务/秒 | 864M |
| 100000 写事务/秒 | 8.6B（超过上限）|

中等负载约 20-50 天就用完 21 亿——所以**必须 freeze**。

### 6.2 Freeze 的含义

`FrozenTransactionId = 2`（特殊值）。当一条 tuple 的 xmin 被改成 frozen，**它对所有未来事务都可见**，不再参与 xid 比较。即使 xid wraparound，frozen tuple 也安全。

PG 9.4 之前直接把 xmin 改成 2；之后引入更优雅的方法：在 tuple 头的 infomask 加 `HEAP_XMIN_FROZEN` 位，保留原始 xmin（方便审计 / 不破坏数据）。

### 6.3 三层 freeze 阈值

```
新事务                                                         老事务
        |————————————————————————————————————————————————|
0                                                          2^31

         freeze_min_age   freeze_table_age      autovacuum_freeze_max_age
              50M              150M                    200M
              ↓                  ↓                       ↓
       VACUUM 时机    aggressive autovacuum     anti-wraparound（强制）
```

**vacuum_freeze_min_age**（默认 5000 万）：
正常 VACUUM 时，xmin 比当前 xid 老 50M 以上的 tuple 会被 freeze。

**vacuum_freeze_table_age**（默认 1.5 亿）：
表的 `relfrozenxid` 距今超过 150M → 下次 autovacuum 会变成"aggressive"，扫描所有页（哪怕 VM 标记 all-visible）。

**autovacuum_freeze_max_age**（默认 2 亿）：
表的 `relfrozenxid` 距今超过 200M → **强制启动 anti-wraparound autovacuum**，**不能取消**，**即使你关了 autovacuum 也会跑**——这是最后防线。

### 6.4 wraparound 警告与停服

接近危险时 PG 会打印：

```
WARNING: database "mydb" must be vacuumed within 10000000 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM.
```

如果触达停服阈值：

```
ERROR: database is not accepting commands to avoid wraparound data loss in database "mydb"
HINT: Stop the postmaster and vacuum that database in single-user mode.
```

数据库**拒绝所有写**——线上故障案例：未 freeze 的 wraparound 让某金融客户 PG 停服 6 小时。

### 6.5 紧急 wraparound 抢救

如果生产已经报错"not accepting commands"：

1. 停 postmaster：`pg_ctl stop`
2. 进入单用户模式：`postgres --single -D /var/lib/pgsql/data mydb`
3. 在单用户模式 prompt 里：`VACUUM (FREEZE);`
4. 等几分钟到几小时（视表大小）
5. 退出，正常启动

不要慌：数据没丢，只是被冻结保护。

### 6.6 多事务 wraparound（MultiXact）

MultiXact ID（多事务行锁）也是 32-bit。共享行锁多的应用（外键、`SELECT FOR SHARE` 频繁）可能先于普通 xid 触发 wraparound。

参数：

- `autovacuum_multixact_freeze_max_age = 400000000`（4 亿，比 xid 宽松）
- `vacuum_multixact_freeze_min_age = 5000000`
- `vacuum_multixact_freeze_table_age = 150000000`

监控：

```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       mxid_age(datminmxid) AS mxid_age
FROM pg_database
ORDER BY xid_age DESC;
```

---

## 第七章：表膨胀检测

### 7.1 三种检测方法

#### 方法 1：pg_stat_user_tables（粗略）

```sql
SELECT
    schemaname, relname,
    n_live_tup, n_dead_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

注意 `n_live_tup / n_dead_tup` 是 **估算值**（由 autoanalyze 维护），未必精确，但是免费的一手指标。

#### 方法 2：pgstattuple 扩展（精确但慢）

```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('big_table');
-- 输出 table_len / tuple_count / dead_tuple_count / free_space / dead_tuple_percent

-- 索引的膨胀
SELECT * FROM pgstattuple('idx_big_table_v');
SELECT * FROM pgstatindex('idx_big_table_v');
```

`pgstattuple` 会全扫表，慢；大表用近似版本：

```sql
SELECT * FROM pgstattuple_approx('big_table');
```

#### 方法 3：估算 SQL（最常用）

```sql
-- pgexperts/pgstattuple 风格的估算 SQL（社区流传版本）
SELECT
    schemaname, tblname, bs * tblpages AS real_size,
    (tblpages - est_tblpages) * bs AS bloat_size,
    round((tblpages - est_tblpages) * 100.0 / NULLIF(tblpages, 0), 2) AS bloat_pct
FROM (
    SELECT
        schemaname, tblname, tblpages, bs,
        ceil((reltuples * (datawidth + 24)) / (bs - 20)) AS est_tblpages
    FROM (
        SELECT
            ns.nspname AS schemaname, tbl.relname AS tblname,
            tbl.reltuples,
            tbl.relpages AS tblpages,
            current_setting('block_size')::int AS bs,
            32 AS datawidth  -- 简化估算
        FROM pg_class tbl
        JOIN pg_namespace ns ON ns.oid = tbl.relnamespace
        WHERE tbl.relkind = 'r' AND ns.nspname NOT IN ('pg_catalog', 'information_schema')
    ) s
) s2
WHERE tblpages > 100
ORDER BY bloat_size DESC;
```

实战推荐：搜 "pgexperts bloat query"，社区有更精确的版本。

### 7.2 index bloat

索引也会膨胀（dead 索引条目 + B-tree 空页）：

```sql
-- 简单看法：未使用索引（idx_scan = 0 长期为 0 → 可能直接删）
SELECT
    schemaname, relname AS table, indexrelname AS index,
    idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND pg_relation_size(indexrelid) > 1024 * 1024 * 10
ORDER BY pg_relation_size(indexrelid) DESC;

-- 精确索引膨胀
SELECT * FROM pgstatindex('idx_big_table_v');
-- 看 leaf_fragmentation / avg_leaf_density
```

PG 14+ 的 B-tree bottom-up deletion 大幅缓解索引膨胀；但仍建议定期 `REINDEX CONCURRENTLY`（PG 12+）。

---

## 第八章：解决膨胀的工具

### 8.1 VACUUM FULL（停机重写）

```sql
VACUUM FULL big_table;
```

- 持 AccessExclusive，业务全停
- 100GB 表可能 1-2 小时
- 中途取消已写文件浪费
- 生产 **几乎不可用**，除非有维护窗口

### 8.2 CLUSTER

```sql
CLUSTER big_table USING idx_created_at;
```

类似 VACUUM FULL 但额外按索引顺序排列。同样持 AccessExclusive。

### 8.3 pg_repack（在线重组首选）

[pg_repack](https://github.com/reorg/pg_repack) 是社区事实标准：

```bash
pg_repack -h localhost -U postgres -d mydb -t big_table -j 4
```

工作原理：

1. 创建影子表（结构相同）
2. 创建触发器把原表的写同步到影子表
3. INSERT INTO 影子表 SELECT FROM 原表（按主键顺序，无碎片）
4. 等同步追上
5. 短暂 AccessExclusive（秒级）切换表名
6. 删除旧表

**缺点**：
- 需要 2× 表大小的磁盘（影子表）
- 表必须有 PRIMARY KEY 或 UNIQUE NOT NULL 索引
- 不支持外键关联表的级联重组
- 极个别场景触发器 race condition（pg_repack 自身记录的 issue）

### 8.4 pg_squeeze

[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze) 类似 pg_repack，但用 logical decoding 同步而非触发器：

- 优点：对 OLTP 影响更小（无触发器开销）
- 缺点：需要 logical replication slot；wal_level=logical

### 8.5 工具对比

| 工具 | 锁等级 | 在线 | 磁盘开销 | 适用 |
|---|---|---|---|---|
| VACUUM（普通） | SUE | ✓ | 无额外 | 日常 |
| VACUUM FULL | AE | ✗ | 临时 1× | 维护窗口 |
| CLUSTER | AE | ✗ | 临时 1× | 维护窗口 + 物理排序 |
| pg_repack | AE（秒级 swap） | ✓ | 2× | 生产首选 |
| pg_squeeze | AE（秒级 swap） | ✓ | 2× | 生产备选 |
| REINDEX CONCURRENTLY | SUE | ✓ | 1× | 仅索引膨胀 |

---

## 第九章：HOT update —— 减少索引膨胀

### 9.1 HOT 是什么

Heap-Only Tuple：UPDATE 时，如果**所有索引列都未变**，且新 tuple 能放在**同一页**，则：

- 新 tuple 直接放在同页
- 旧 tuple 的 line pointer 指向新 tuple（HOT chain）
- **索引不更新**

好处：索引不增长、索引不膨胀、index lookup 通过 HOT chain 找到最新版本。

### 9.2 HOT 触发条件

1. 所有索引列未变更
2. 同页有足够空间（FILLFACTOR 控制）
3. 表本身不是分区父表等

### 9.3 FILLFACTOR

```sql
-- 建表时
CREATE TABLE t (id int PRIMARY KEY, v int) WITH (FILLFACTOR = 80);

-- 改既有表
ALTER TABLE t SET (FILLFACTOR = 80);
```

`FILLFACTOR = 80` 意味着 INSERT 时每页留 20% 空间给 HOT update。

经验：

- 经常 UPDATE 的非索引列的表：FILLFACTOR = 70-80
- INSERT-only 表：FILLFACTOR = 100（默认）
- 索引 fillfactor 类似：B-tree 索引默认 90（叶子页）

### 9.4 监控 HOT 比例

```sql
SELECT
    relname,
    n_tup_upd,                    -- 总更新数
    n_tup_hot_upd,                -- HOT 更新数
    round(n_tup_hot_upd::numeric / NULLIF(n_tup_upd, 0) * 100, 2) AS hot_pct
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC
LIMIT 10;
```

健康指标：HOT 比例 > 70%。低于此值检查：

- 是不是某个索引覆盖了高频更新列（考虑去掉）
- FILLFACTOR 是否太满（页内无空间）

---

## 第十章：阻塞 VACUUM 的三大杀手

### 10.1 杀手一：长事务 / idle in transaction

VACUUM 必须保留所有"对某个活事务可见"的 dead tuple，否则会破坏一致性。"全局 xmin"（即所有活事务里最小的 xmin）就是 vacuum 的截止线。

一个老事务 = 截止线被拽回去 = 所有比它新的 dead tuple 都不能回收。

监控：

```sql
SELECT
    pid, usename, state,
    age(now(), xact_start) AS xact_age,
    backend_xmin,
    query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC NULLS LAST
LIMIT 10;
```

`backend_xmin` 最小的那个 backend 就是 vacuum 截止线的持有者。

防御参数（见 [P07 第十一章](./07-精通-事务隔离与锁.md)）：

```ini
idle_in_transaction_session_timeout = '5min'
transaction_timeout = '30min'    -- PG 17+
```

### 10.2 杀手二：Prepared Transaction（2PC 残留）

```sql
-- 应用做了 PREPARE TRANSACTION 但没 COMMIT PREPARED / ROLLBACK PREPARED
PREPARE TRANSACTION 'foo';
-- ... 应用挂了或代码 bug 忘了清理 ...
```

prepared txn 永远占着 xmin。监控：

```sql
SELECT * FROM pg_prepared_xacts;
-- 看到记录但 age 很老就是残留

-- 清理
ROLLBACK PREPARED 'foo';
```

**生产防御**：除非真要做分布式事务，否则关掉：

```ini
max_prepared_transactions = 0
```

### 10.3 杀手三：未消费的 Replication Slot

物理 slot：

```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE slot_type = 'physical';
```

逻辑 slot：

```sql
SELECT slot_name, active, confirmed_flush_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

`active=false` 且 lag 持续增长 → 主库 WAL 不能 archive、xmin horizon 卡住。

**生产防御**：

```ini
max_slot_wal_keep_size = '10GB'    -- PG 13+：slot 滞后超此值后自动 invalidate（数据丢但保库）
```

逻辑 slot 还有 `hot_standby_feedback` 在 standby 上拽 master 的 xmin——standby 上跑长查询同样会阻塞主库 vacuum。

### 10.4 综合诊断 SQL

```sql
-- 找出"最老 xmin 持有者"
SELECT
    (SELECT max(age(backend_xmin)) FROM pg_stat_activity) AS oldest_backend_xmin_age,
    (SELECT max(age(transaction)) FROM pg_prepared_xacts) AS oldest_prepared_xact_age,
    (SELECT max(age(xmin)) FROM pg_replication_slots WHERE xmin IS NOT NULL) AS oldest_slot_xmin_age;
```

哪个最老，问题就在哪里。

---

## 第十一章：监控 SQL 速查

### 11.1 当前 VACUUM 进度

```sql
SELECT
    p.pid, p.datname, p.relid::regclass,
    p.phase,
    p.heap_blks_scanned, p.heap_blks_total,
    round(p.heap_blks_scanned::numeric / NULLIF(p.heap_blks_total, 0) * 100, 2) AS pct,
    p.index_vacuum_count,
    p.num_dead_tuples,
    a.query
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a USING (pid);
```

`phase` 七个值：initializing / scanning heap / vacuuming indexes / vacuuming heap / cleaning up indexes / truncating heap / performing final cleanup。

### 11.2 表 vacuum / analyze 历史

```sql
SELECT
    schemaname, relname,
    last_vacuum, last_autovacuum,
    last_analyze, last_autoanalyze,
    vacuum_count, autovacuum_count,
    n_dead_tup, n_live_tup
FROM pg_stat_user_tables
WHERE relname = 'big_table';
```

如果 `last_autovacuum` 是几天前，dead tuple 仍很多 → 阈值未触发或 autovacuum 跑不过来。

### 11.3 freeze 健康

```sql
SELECT
    c.relname,
    age(c.relfrozenxid) AS xid_age,
    mxid_age(c.relminmxid) AS mxid_age,
    pg_size_pretty(pg_table_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

`xid_age` 越接近 `autovacuum_freeze_max_age`（默认 2 亿），越快会强制 anti-wraparound。

### 11.4 数据库级 wraparound

```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2^31 - 1000000 - age(datfrozenxid) AS xids_left
FROM pg_database
ORDER BY xid_age DESC;
```

`xids_left` 是"还能消费多少 xid 才停服"。

---

## 第十二章：实战案例

### 12.1 案例：200GB 表，dead_pct 80% 的在线处理

**现状**：

- `users_log` 表 200GB，`n_live_tup=2亿`，`n_dead_tup=8亿`，`dead_pct=80%`
- 业务还在持续写入（约 1000 QPS）
- 表上有 5 个索引，其中 3 个膨胀严重

**诊断**：

```sql
SELECT * FROM pg_stat_user_tables WHERE relname='users_log';
-- 发现 last_autovacuum 是 7 天前；vacuum_count = 0（人为关了）

SELECT reloptions FROM pg_class WHERE relname='users_log';
-- {autovacuum_enabled=false}   ← 元凶
```

**根因**：上次性能优化关了 autovacuum，从未恢复。

**处理方案**：

```sql
-- 第 1 步：恢复 autovacuum 并降低阈值
ALTER TABLE users_log SET (
    autovacuum_enabled = true,
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_vacuum_threshold = 100000,
    autovacuum_vacuum_cost_delay = 0,
    autovacuum_vacuum_cost_limit = 10000
);

-- 第 2 步：人工跑一次 VACUUM（不带 FULL，在线）
VACUUM (VERBOSE, ANALYZE, PARALLEL 4) users_log;
-- 预计跑几小时，期间不阻塞业务；dead tuple 标记可重用
```

VACUUM 完成后 `n_dead_tup` 归零，但**物理空间不还**（仍 200GB）。如果磁盘紧张：

```bash
# 第 3 步：pg_repack 在线收缩
pg_repack -h db -U postgres -d mydb -t users_log -j 4
# 持锁仅切换表名瞬间；表收缩到 ~40GB；索引同时重建
```

**预防**：

```sql
-- 关键：永远不要在生产关 autovacuum
-- 表级覆盖参数比关 autovacuum 更聪明
-- 加监控告警：dead_pct > 30% 时 P2
```

### 12.2 案例：xid wraparound 警告

凌晨 3 点告警："database mydb must be vacuumed within 50 million transactions"。

**诊断**：

```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
-- mydb age = 150M（默认 max_age = 200M 还差 50M）

SELECT relname, age(relfrozenxid)
FROM pg_class
WHERE relkind='r' AND relnamespace IN (
    SELECT oid FROM pg_namespace WHERE nspname NOT LIKE 'pg_%'
)
ORDER BY 2 DESC LIMIT 5;
-- big_event_log 表 age=150M（最老）
```

**根因**：`big_event_log` 是分区父表，autovacuum 默认对分区不友好；当前老分区一直没被 freeze。

**处理**：

```sql
-- 立即手动 freeze
VACUUM (FREEZE, VERBOSE) big_event_log_2020_q1;
VACUUM (FREEZE, VERBOSE) big_event_log_2020_q2;
-- ...

-- 长期：把老分区调成更激进的 freeze
ALTER TABLE big_event_log_2020_q1 SET (
    autovacuum_freeze_min_age = 1000000,
    autovacuum_freeze_table_age = 50000000
);
```

### 12.3 案例：autovacuum 永远跑不完

**现象**：`pg_stat_progress_vacuum` 显示一直在 phase=scanning heap，但进度看起来停滞；同时业务延迟上升。

**诊断**：

```sql
-- 看 autovacuum worker 的 wait event
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker';
-- wait_event = 'VacuumDelay'  ← 限流卡住了
```

**根因**：cost_delay = 20ms（PG 13 老版本默认），表又大；vacuum 跑得比 dead tuple 产生还慢。

**处理**：

```sql
-- 临时解：手动跑一遍不限速
SET vacuum_cost_delay = 0;
SET vacuum_cost_limit = 10000;
VACUUM (VERBOSE, PARALLEL 4) big_table;

-- 长期解：全局降 cost_delay
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 2000;
SELECT pg_reload_conf();
```

---

## 生产实践

### autovacuum 调优 5 条铁律

1. **绝不关闭 autovacuum**——出问题只调 per-table 参数
2. **大表 (>10GB) per-table 覆盖**：scale_factor 0.02-0.05
3. **PG 14+ 默认 cost_delay=2ms 是好事**，旧版本一定降到 2-5ms
4. **autovacuum_max_workers**：CPU 数 / 4，不超过 8
5. **PG 13+ 一定打开 insert-only autovacuum**（默认已开）

### 关键参数推荐（中等服务器：32 vCPU / 128GB / SSD）

```ini
# autovacuum
autovacuum = on
autovacuum_max_workers = 5
autovacuum_naptime = 30s
autovacuum_vacuum_scale_factor = 0.1            # 全局基线（per-table 覆盖大表）
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_insert_scale_factor = 0.1
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 1000

# freeze
autovacuum_freeze_max_age = 200000000
vacuum_freeze_min_age = 5000000
vacuum_freeze_table_age = 100000000

# 防长事务
idle_in_transaction_session_timeout = '5min'
statement_timeout = '60s'

# slot 安全网
max_slot_wal_keep_size = '50GB'

# 维护内存
maintenance_work_mem = '2GB'                    # autovacuum 用，每 worker 上限
autovacuum_work_mem = -1                        # 跟 maintenance_work_mem
```

### 每日 / 每周巡检 SQL

```sql
-- 每天：top 10 膨胀表
\i bloat_check.sql

-- 每天：top 5 vacuum 落后表
SELECT relname, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000
ORDER BY n_dead_tup DESC LIMIT 10;

-- 每天：xid 健康
SELECT datname, age(datfrozenxid) FROM pg_database;

-- 每周：未使用索引
SELECT relname, indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC LIMIT 20;

-- 每月：HOT 比例
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(n_tup_hot_upd::numeric/NULLIF(n_tup_upd,0)*100, 2) AS hot_pct
FROM pg_stat_user_tables WHERE n_tup_upd > 10000 ORDER BY 2 DESC LIMIT 20;
```

### 监控告警阈值

| 指标 | 警告 | 紧急 |
|---|---|---|
| `n_dead_tup / n_live_tup` | > 30% | > 60% |
| `age(datfrozenxid)` | > 1.5 亿 | > 1.8 亿 |
| 最老 backend_xmin age | > 30 分钟 | > 2 小时 |
| `pg_replication_slots.lag` | > 10GB | > 50GB |
| autovacuum 持续时间 | > 6 小时 | > 24 小时 |
| HOT 比例 | < 50% | < 20% |

---

## 陷阱清单

1. **关 autovacuum 跑批**：忘了开 → 数月后表膨胀爆磁盘。
2. **VACUUM FULL 不还磁盘的迷思**：FULL 实际会还（重写新文件、删旧）；普通 VACUUM 才是只标记重用。
3. **TRUNCATE 不释放 dead**：TRUNCATE 重建文件等价于 100% 回收，但持 AE 锁。
4. **VACUUM 不释放表末尾空页**：默认会 truncate 末尾空页，但需要短暂 AE 锁；`vacuum_truncate=off` 可关，避免业务抖动。
5. **REINDEX 持 AccessExclusive**：必须用 `REINDEX CONCURRENTLY`（PG 12+）。
6. **CIC（CREATE INDEX CONCURRENTLY）失败后留下 INVALID 索引**：要手动 DROP 重建；监控 `pg_index.indisvalid=false`。
7. **autovacuum 不跑 prepared statement**：直接 `VACUUM` 也不能在事务里；只能裸跑。
8. **VACUUM (VERBOSE) 在 psql 里输出在 stderr**：脚本捕获要 `2>&1`。
9. **pg_repack 需要 PRIMARY KEY**：无主键表无法 repack；先加主键或用 pg_squeeze。
10. **CLUSTER 不维持物理顺序**：CLUSTER 后续 INSERT/UPDATE 不会保持顺序，几个月后又乱。
11. **冷分区被忽略**：分区表每个子表独立 autovacuum；老分区 INSERT 停止后可能永远不达到触发阈值——必须 INSERT-only vacuum（PG 13+）或人工 vacuum freeze。
12. **idle in transaction (aborted)**：abort 后仍持事务，必须 ROLLBACK 释放——参数 `idle_in_transaction_session_timeout` 同样能 kill。
13. **standby 上长查询拽主库 vacuum**：`hot_standby_feedback=on` 会把 standby 的 xmin 上送主库，长查询直接卡主库 vacuum。
14. **TOAST 表单独 vacuum**：大字段表的 TOAST（`pg_toast.pg_toast_xxx`）单独有膨胀；要单独看。
15. **MultiXact wraparound 比 xid 更隐蔽**：监控 `mxid_age` 不要只看 `xid_age`。
16. **VACUUM FULL 在 archive 模式下产生海量 WAL**：备份和复制带宽暴增。
17. **anti-wraparound vacuum 不可取消**：`pg_cancel_backend` 也不行——只能等。
18. **删除大量行后表不收缩**：DELETE 不归还空间，只标记 dead，必须 vacuum + repack。

---

## 2026 现状

| 主题 | 2026 进展 |
|---|---|
| **PG 17 autovacuum 内存改进** | 删除了 maintenance_work_mem 内对 dead tuple 数组的 1GB 硬上限；vacuum 一次能记更多 dead tuple，减少多轮扫描 |
| **PG 17 并行 VACUUM 改进** | 并行索引清理更稳定；ON COMMIT TRUNCATE 临时表性能提升 |
| **PG 18 异步 IO** | VACUUM 的预读用异步 IO（io_uring on Linux），冷表 vacuum 提速显著 |
| **PG 18 改进 vacuum 反馈** | `pg_stat_progress_vacuum` 增加更多字段；wraparound 风险计算更精确 |
| **xid 64-bit** | 社区讨论中（2024-2026 数次邮件列表大讨论），尚未进 mainline；EDB / GreenPlum / 阿里 PolarDB 等已实现 |
| **pg_repack 工具状态** | 仍是事实标准；维护活跃；2026 兼容 PG 18 |
| **pg_squeeze** | Cybertec 持续维护；与 logical decoding 集成深入 |
| **CloudNativePG / Crunchy / Zalando** | K8s operator 普遍自带 vacuum 告警与自动化 maintenance window |
| **AI 辅助调优** | pganalyze / Datadog DBM 已能根据负载推荐 per-table autovacuum 参数；2025-2026 走向 self-driving |
| **HOT update 改进讨论** | 社区在讨论"global index"和"hot-only column"等扩展机制；尚未落地 |

---

## 练习题

1. 解释 PG 的 MVCC 为什么需要 VACUUM 而 MySQL InnoDB 不需要。从存储模型（堆表 vs undo log）和清理策略两个角度回答。

2. 一张表 `t(id PK, v int, created_at)`，索引仅有 PK。如果你执行 100 次 `UPDATE t SET v=v+1 WHERE id=1`，堆表里会怎么变？index 会膨胀吗？为什么？如果加了 `CREATE INDEX idx_v ON t(v)`，再做同样的 UPDATE，结果如何？

3. 列出 autovacuum 的所有触发条件（VACUUM / INSERT-only VACUUM / ANALYZE），写出它们的精确公式。然后给一张"日插 1000 万行、每月旧分区切走、单分区约 3 亿行"的表设计 per-table 参数。

4. 解释 xid wraparound 的全部流程：xid 用尽 → freeze 机制 → 三层阈值（min_age / table_age / max_age）→ aggressive autovacuum → anti-wraparound autovacuum → 停服。画图说明。

5. 写出三个"阻塞 vacuum 的杀手"的检测 SQL：长事务、prepared txn、未消费 slot。然后给出修复 SQL。

6. 给定 200GB 表 `events`，n_dead_tup=8亿，n_live_tup=2亿。在不停业务的前提下设计完整处理流程（包括预防）。要求：(a) dead tuple 回收；(b) 物理空间归还；(c) 索引去膨胀；(d) 防止再次发生。

7. 解释 HOT update 的触发条件。给一张 `users(id PK, email UNIQUE, name, last_login, login_count)` 表设计 FILLFACTOR，并解释你的取值。

8. 实验：开两个连接。A: `BEGIN; SELECT pg_sleep(60);`。B: `INSERT/DELETE/UPDATE` 同一表 100 次。然后看 `n_dead_tup` 与 `pg_stat_user_tables` 的变化，再 `VACUUM`，对比 A 提交前 vs 提交后的 vacuum 行为。说明为什么 dead tuple 没立刻回收。

---

## 参考答案

1. **为什么 PG 需要 VACUUM**：存储模型上 PG 是堆表多版本——UPDATE/DELETE 不原地改，旧版本物理留在堆页里，必须靠 VACUUM 显式回收死元组并维护可见性映射、FSM；MySQL InnoDB 把旧版本写到 undo log，当前行原地更新，commit 后旧版本由 purge 线程隐式回收，表本身不因 MVCC 膨胀。清理策略：PG 是"延迟批量清理"（autovacuum 按阈值触发），InnoDB 是"后台 purge 持续清理"。

2. **100 次 UPDATE 的膨胀**：仅有 PK、且更新的 `v` 列**无索引**时满足 HOT 条件——新版本写在同页、通过 HOT 链指向，**索引不膨胀**（PK 项不变），堆页内 100 个死版本可被页内整理（HOT pruning）复用。加了 `idx_v` 后，更新了被索引列 → HOT 失效，每次 UPDATE 都新增一个索引项，**index 膨胀**，堆也更易跨页膨胀。

3. **autovacuum 触发公式**：
   - VACUUM：`n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples`（默认 50 + 0.2×N）。
   - INSERT-only VACUUM（PG 13+）：`n_ins_since_vacuum > autovacuum_vacuum_insert_threshold + autovacuum_vacuum_insert_scale_factor × reltuples`（默认 1000 + 0.2×N）。
   - ANALYZE：`n_mod_since_analyze > autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × reltuples`（默认 50 + 0.1×N）。
   3 亿行大表用默认 scale_factor 阈值高达 6000 万，应改为固定小阈值：`ALTER TABLE t SET (autovacuum_vacuum_scale_factor=0, autovacuum_vacuum_threshold=500000, autovacuum_vacuum_insert_scale_factor=0, autovacuum_vacuum_insert_threshold=1000000, autovacuum_vacuum_cost_limit=2000)`，并加大 maintenance_work_mem。

4. **wraparound 流程**：xid 是 32 位环形（约 21 亿可用）→ 需要把老元组 freeze（标记为"永久可见"，不再依赖 xid 比较）→ 三层阈值：`vacuum_freeze_min_age`（多老的 xid 才 freeze）、`vacuum_freeze_table_age`（表 age 超过则下次 VACUUM 全表扫描 aggressive）、`autovacuum_freeze_max_age`（默认 2 亿，超过强制触发 **anti-wraparound autovacuum**，即使 autovacuum 关闭也会跑且不可被普通锁中断）→ 再不处理逼近 `vacuum_failsafe_age`（默认 16 亿）进入 failsafe（跳过索引清理只顾 freeze）→ 接近 21 亿 PG **拒绝新事务**，只能单用户模式 VACUUM。

5. **阻塞 vacuum 三杀手检测与修复**：
   ```sql
   -- 长事务
   SELECT pid, now()-xact_start AS dur, state, query FROM pg_stat_activity
   WHERE state <> 'idle' AND xact_start < now()-interval '5 min' ORDER BY xact_start;
   -- 修复：SELECT pg_terminate_backend(pid);
   -- prepared transaction（两阶段事务残留）
   SELECT gid, prepared, owner FROM pg_prepared_xacts ORDER BY prepared;
   -- 修复：ROLLBACK PREPARED 'gid';
   -- 未消费 / 不活跃复制槽
   SELECT slot_name, active, restart_lsn FROM pg_replication_slots WHERE NOT active;
   -- 修复：SELECT pg_drop_replication_slot('slot_name');
   ```
   三者都会卡住全局 OldestXmin / 保留 WAL，导致 VACUUM 无法推进 horizon。

6. **200GB 大膨胀表处理**：(a) dead tuple 回收——先确认无长事务/旧 slot 卡 horizon，再 `VACUUM (VERBOSE) events` 或调高其 autovacuum 强度让其追上；(b) 物理空间归还——在线用 `pg_repack -t events`（需约 2× 磁盘、瞬时锁），不能停服时避免 `VACUUM FULL`（持 ACCESS EXCLUSIVE）；(c) 索引去膨胀——`REINDEX INDEX CONCURRENTLY`（pg_repack 会一并重建）;(d) 预防——下调该表 scale_factor、设固定阈值、加大 cost_limit、监控 n_dead_tup/膨胀率、消除长事务与未用 slot。

7. **HOT 触发条件**：UPDATE **未修改任何被索引的列** 且 新版本能放进**同一个堆页**（fillfactor 留出空间）。`users(id PK, email UNIQUE, name, last_login, login_count)` 中 `last_login`/`login_count` 是高频更新且无索引的列——典型 HOT 场景，设 `FILLFACTOR=80~90` 给页内留空间让新版本同页存放，最大化 HOT 比例、减少索引与堆膨胀。

8. **实验解释**：A 持有 `pg_sleep(60)` 期间快照活跃，其 snapshot.xmin 卡住全局 OldestXmin。B 制造的 100 个死元组虽然 `n_dead_tup` 上升，但在 A 提交前 `VACUUM` **无法回收**它们——因为这些版本可能对 A 仍可见，VACUUM 只能清理早于 OldestXmin 的死元组。A 提交后 horizon 推进，再次 VACUUM 才真正回收。这演示了"长事务阻塞 vacuum 推进"的核心机制。

---

## 延伸阅读

- 官方文档：[Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- 官方文档：[Preventing Transaction ID Wraparound Failures](https://www.postgresql.org/docs/18/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)
- 官方文档：[VACUUM command](https://www.postgresql.org/docs/18/sql-vacuum.html)
- 经典博客：
  - [pganalyze - The Vacuum Cleaner](https://pganalyze.com/blog/postgres-vacuum) 系列
  - [Crunchy Data - Postgres Bloat](https://www.crunchydata.com/blog)
  - [Cybertec - VACUUM](https://www.cybertec-postgresql.com/en/category/vacuum/)
  - [2ndQuadrant - PostgreSQL Anti-Patterns: Read Modify Write](https://www.2ndquadrant.com/en/blog/)
- 内核源码：
  - `src/backend/access/heap/vacuumlazy.c`（普通 VACUUM 主流程）
  - `src/backend/commands/vacuum.c`（命令入口）
  - `src/backend/postmaster/autovacuum.c`（autovacuum launcher / worker）
  - `src/backend/access/heap/heapam_visibility.c`（HeapTupleSatisfiesVacuum）
- 工具：
  - [pg_repack](https://github.com/reorg/pg_repack)
  - [pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze)
  - `pgstattuple` / `pgstatindex` / `pg_visibility`（contrib 自带）
- 监控查询：
  - [PG bloat queries](https://wiki.postgresql.org/wiki/Show_database_bloat)
  - pganalyze Index Advisor / Bloat View
- 项目内参考：
  - [P02 堆表与 TOAST](./02-精通-堆表存储与-TOAST.md)（dead tuple 的物理形态）
  - [P03 MVCC](./03-精通-MVCC-与可见性.md)（旧版本怎么来的）
  - [P07 事务与锁](./07-精通-事务隔离与锁.md)（长事务危害的上半场）
  - [P20 参数调优](./20-精通-参数调优.md)（maintenance_work_mem 等）
  - [P21 监控诊断](./21-精通-监控与诊断.md)（wait events、阻塞链）
