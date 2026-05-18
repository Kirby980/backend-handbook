# 精通查询优化器与 EXPLAIN：cost model、直方图、hash join、字段全解

> 关联章节：[M02 索引](./02-精通-InnoDB-索引.md)、[M03 事务](./03-精通-InnoDB-事务-MVCC.md)、[M07 PS](./07-精通-Performance-Schema.md)

---

## 引言：优化器是怎么"猜"的

经常听到："优化器选错索引了"——背后的事实是：**优化器从来不"知道"，它只是根据统计信息和成本模型估算**。理解优化器的决策逻辑，能让你写出"优化器友好"的 SQL，并在它判错时知道怎么干预。

读完这一章你应该能：

- 解释优化器的成本模型（CPU + I/O 各分多少）
- 用直方图改进低选择性列的估算
- 区分 EXPLAIN 输出的所有 type / Extra 字段
- 用 EXPLAIN ANALYZE 看真实执行情况
- 知道何时该用 Hint 干预优化器
- 排查一次"优化器选错索引"的故障

---

## 第一章：MySQL 优化器架构

### 1.1 阶段划分

```
SQL → Parser (词法/语法) → AST
    → Resolver (列名/表名解析)
    → Logical Optimizer (重写、谓词下推、子查询展开)
    → Cost-Based Optimizer (Plan 选择)
    → Physical Plan
    → Executor
```

5.7 之后优化器代码大幅重构（`sql/range_optimizer/`、`sql/join_optimizer/`），8.0 引入 hypergraph optimizer（`secondary_engine_cost_threshold` 起作用时）。

### 1.2 主要优化能力

| 能力 | 版本 |
|---|---|
| Index Merge | 5.0+ |
| Range Optimizer | 一直有 |
| Multi-Range Read (MRR) | 5.6+ |
| Index Condition Pushdown (ICP) | 5.6+ |
| Block Nested Loop Join | 一直 |
| Hash Join | 8.0.18+（先 inner，8.0.20+ outer/anti/semi） |
| 直方图统计 | 8.0+ |
| Window function | 8.0+ |
| CTE / Recursive CTE | 8.0+ |
| Lateral derived table | 8.0.14+ |
| Hypergraph join optimizer | 8.0.21+（实验性） |

---

## 第二章：成本模型

### 2.1 cost = I/O cost + CPU cost

优化器为每个 plan 估一个 cost 数字（无单位），选最小的。

```
cost ≈ disk_read_pages * io_block_read_cost
     + memory_read_pages * memory_block_read_cost
     + rows_estimated * row_evaluate_cost
     + ...
```

参数表：

```sql
SELECT * FROM mysql.engine_cost;
SELECT * FROM mysql.server_cost;
```

```
io_block_read_cost     = 1.0    -- 从磁盘读一页
memory_block_read_cost = 0.25   -- 从 BP 读一页
row_evaluate_cost      = 0.1    -- 处理一行 CPU
disk_temptable_create_cost = 20.0
disk_temptable_row_cost    = 0.5
key_compare_cost       = 0.05
```

可调整（需重启或 FLUSH OPTIMIZER_COSTS）：

```sql
UPDATE mysql.engine_cost SET cost_value = 0.5 WHERE cost_name = 'io_block_read_cost';
FLUSH OPTIMIZER_COSTS;
```

SSD 用户可以下调 io_block_read_cost（与 memory 接近），让优化器更倾向于走索引大量回表。

### 2.2 行数估算 = 索引 cardinality + 直方图

```sql
SHOW TABLE STATUS LIKE 'users'\G
-- Rows: 1024000   ← 大致行数（InnoDB 是估算！）
```

```sql
SELECT INDEX_NAME, COLUMN_NAME, CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_NAME = 'users';
-- CARDINALITY: 该索引列的不同值数估算
```

cardinality 由 InnoDB 抽样估算（默认抽 20 页），不准。

### 2.3 直方图（Histogram）

8.0+ 用直方图给出列的分布估算（不是索引）：

```sql
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;
```

直方图不影响读写，但优化器用它估算 `WHERE status='paid'` 的行数。

适用：

- 没有索引的列
- 数据分布严重倾斜（如 status='paid' 占 95%）

查看：

```sql
SELECT * FROM information_schema.COLUMN_STATISTICS WHERE TABLE_NAME='orders';
```

删除：

```sql
ANALYZE TABLE orders DROP HISTOGRAM ON status;
```

直方图过期：底层数据大幅变化时优化器仍用旧直方图——周期性 ANALYZE。

---

## 第三章：EXPLAIN

### 3.1 三种格式

```sql
EXPLAIN SELECT ...;                       -- 表格形式
EXPLAIN FORMAT=JSON SELECT ...;           -- JSON，含 cost
EXPLAIN FORMAT=TREE SELECT ...;           -- 树形（8.0+）
EXPLAIN ANALYZE SELECT ...;               -- 真的执行 + 实际 timing（8.0.18+）
```

EXPLAIN ANALYZE 最强——给真实行数 vs 估算行数对比，差距大就是统计信息或直方图问题。

### 3.2 经典表格输出字段

```
+----+-------------+-------+-------+-------+-------+--------+------+----------+--------------------+
| id | select_type | table | type  | key   | ref   | rows   | filtered | Extra              |
+----+-------------+-------+-------+-------+-------+--------+----------+--------------------+
|  1 | SIMPLE      | users | ref   | idx_n | const | 100    | 100.00   | Using where        |
+----+-------------+-------+-------+-------+-------+--------+----------+--------------------+
```

---

## 第四章：type 字段（访问类型）

按性能从最好到最差：

### 4.1 system / const

主键 / 唯一索引等值查询，至多 1 行。

```sql
SELECT * FROM t WHERE id = 1;
-- type: const
```

### 4.2 eq_ref

JOIN 时被驱动表用唯一索引等值匹配。

```sql
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
-- u 的 type: eq_ref
```

### 4.3 ref

非唯一索引等值。

```sql
SELECT * FROM users WHERE name = 'Alice';
-- type: ref（idx_name）
```

### 4.4 range

索引范围扫描。

```sql
SELECT * FROM users WHERE age > 20 AND age < 30;
-- type: range
```

### 4.5 index

全索引扫描——遍历整个二级索引（不回表）。

```sql
SELECT id, name FROM users;
-- type: index（覆盖索引时常见）
```

### 4.6 ALL

全表扫描。**几乎都是要修复的信号**——除非小表（< 1000 行）或要扫全表的统计 SQL。

### 4.7 其他常见

- `index_merge`：合并多个索引（OR / AND 跨索引）
- `index_subquery` / `unique_subquery`：子查询被改写
- `fulltext`：全文索引
- `ref_or_null`：ref + IS NULL

---

## 第五章：Extra 字段（关键执行信息）

### 5.1 性能好的标志

| Extra | 含义 |
|---|---|
| `Using index` | 覆盖索引，不回表 |
| `Using index condition` | ICP 生效 |
| `Using MRR` | 多范围读，回表前排主键 |
| `Using index for group-by` | Loose index scan |

### 5.2 性能差的标志

| Extra | 含义 |
|---|---|
| `Using filesort` | 需要在 sort_buffer / 磁盘排序 |
| `Using temporary` | 需要内部临时表（GROUP BY / DISTINCT 等） |
| `Using join buffer (Block Nested Loop)` | JOIN 没用上索引 |
| `Using join buffer (hash join)` | 8.0+ Hash Join |
| `Range checked for each record` | 每行单独决定能否用索引——糟糕 |

### 5.3 中性

- `Using where`：执行器再过滤一次（不一定坏，看是否能用索引避免）
- `Distinct`：找到第一个匹配就停

### 5.4 filesort 与 temporary 处理

filesort：

- 给排序列加索引
- 减少返回列数（Using filesort 时回表的列越少越好）
- 增 `sort_buffer_size`

temporary：

- 看是否能改成不要 GROUP BY / DISTINCT
- 加索引让 GROUP BY 用 Loose index scan
- 给临时表更多内存：`tmp_table_size` / `max_heap_table_size`
- 临时表落盘转 InnoDB 用：8.0+ 默认 `internal_tmp_disk_storage_engine = InnoDB`

---

## 第六章：rows 与 filtered

### 6.1 rows = 估算扫描行数

不一定准——直方图能改善。

### 6.2 filtered = 行被 WHERE 过滤后剩多少 %

```
rows: 1000, filtered: 10.00
→ 扫 1000 行，预计 100 行匹配
```

filtered 估算依赖统计信息和直方图。

### 6.3 EXPLAIN ANALYZE 看真实

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'paid' AND created_at > '2026-05-01'\G
```

输出：

```
-> Filter: ((status = 'paid') and (created_at > '2026-05-01'))  
   (cost=12345.67 rows=1234) (actual time=0.012..3.456 rows=987 loops=1)
```

`cost=` 估算 / `actual` 实际。差距大 → 统计信息有问题。

---

## 第七章：Join 算法

### 7.1 Block Nested Loop (BNL)

Join 没有索引可用 → 把驱动表行加载到 join buffer，对每个被驱动行做内存匹配。

```
SELECT * FROM a, b WHERE a.x = b.y;   -- b.y 无索引
-- a 是驱动表（小表），加载到 join buffer
-- 扫 b 全表，每行跟 buffer 内 a 比较
```

`Extra: Using join buffer (Block Nested Loop)`

性能：O(N × M) 但带 buffer 优化。

### 7.2 Index Nested Loop

驱动表每行用索引在被驱动表查找。

```sql
SELECT * FROM a JOIN b ON a.x = b.y;   -- b.y 有索引
```

驱动表选行少的；被驱动表必须有索引。性能 O(N × log M)。

### 7.3 Hash Join（8.0.18+）

两表都没合适索引时，构建哈希表 + 探测。

```
build phase: 把驱动表（小）塞进内存哈希表
probe phase: 扫被驱动表，每行 hash 探测
```

`Extra: Using join buffer (hash join)`

性能 O(N + M)，比 BNL 快得多。

8.0.18 只支持 inner join；8.0.20 加 outer / anti / semi。**默认开启**——这是 8.0 性能大跃进之一。

### 7.4 Join Order 优化

优化器尝试不同 join 顺序，选 cost 最小：

```sql
EXPLAIN FORMAT=TREE SELECT * FROM a JOIN b JOIN c WHERE ...;
```

复杂 join（10+ 表）优化器搜索空间爆炸 → 限制：

```ini
optimizer_search_depth = 62      -- 默认 62，建议保留
optimizer_prune_level = 1        -- 启发式剪枝
```

---

## 第八章：子查询与派生表

### 8.1 子查询的两种命运

#### 改写为 Join（理想）

```sql
SELECT * FROM a WHERE x IN (SELECT y FROM b);
-- 优化器可能改写为 semi-join：
-- SELECT * FROM a SEMI JOIN b ON a.x = b.y;
```

`Extra` 看到 `Using semijoin`。

#### Materialized 物化（次优）

子查询结果先算成临时表，再 join。

```sql
SELECT * FROM (SELECT * FROM big GROUP BY x) AS t WHERE t.cnt > 100;
-- 派生表 t 物化为临时表
```

8.0 优化派生表合并（derived merge）—— 把派生表内联到外查询。

### 8.2 EXISTS / NOT EXISTS

```sql
SELECT * FROM a WHERE EXISTS (SELECT 1 FROM b WHERE b.x = a.y);
```

通常被优化为 semi-join（与 IN 类似）。NOT EXISTS → anti-join。

### 8.3 写法陷阱

`IN (SELECT ... )` vs `JOIN`：

- 多数情况优化器能识别等价性
- 但有些 case `JOIN` 更优——尤其旧版本

如果优化器选不好：手动改写 + EXPLAIN 对比。

---

## 第九章：Hint —— 干预优化器

### 9.1 Hint 写在 SELECT 后

```sql
SELECT /*+ INDEX(t idx_name) */ * FROM t WHERE name = 'Alice';
SELECT /*+ NO_INDEX(t idx_a) */ * FROM t WHERE a = 1;
SELECT /*+ JOIN_ORDER(b, a) */ * FROM a, b WHERE a.x = b.y;
SELECT /*+ JOIN_FIXED_ORDER */ * FROM a, b, c WHERE ...;
SELECT /*+ HASH_JOIN(a, b) */ * FROM a, b WHERE ...;
SELECT /*+ NO_HASH_JOIN(a, b) */ * FROM a, b WHERE ...;
SELECT /*+ MAX_EXECUTION_TIME(1000) */ * FROM ...;  -- 1000ms 超时
```

### 9.2 老式 Hint

```sql
SELECT * FROM t USE INDEX (idx_name) WHERE ...;
SELECT * FROM t FORCE INDEX (idx_name) WHERE ...;
SELECT * FROM t IGNORE INDEX (idx_name) WHERE ...;
```

USE 是建议，FORCE 是强制（仍可能走全表）。

### 9.3 何时用 Hint

- 优化器 cost 估算严重偏差（统计信息失准 + 复杂 join）
- 某些查询稳定性优先于"理论最优"
- 紧急救火，先用 Hint 顶住

**长期解决**：调整索引、用直方图、修统计信息。Hint 容易在数据分布变化后变成"反优化"。

---

## 第十章：统计信息与分析

### 10.1 ANALYZE TABLE

```sql
ANALYZE TABLE users;
```

重新采样索引 cardinality。InnoDB 默认 `innodb_stats_persistent=ON`（5.6+），统计信息持久化到 `mysql.innodb_*_stats` 表。

### 10.2 自动统计

```ini
innodb_stats_auto_recalc = ON       -- 表大量变化时自动 ANALYZE
innodb_stats_persistent_sample_pages = 20   -- 采样页数，越大越准但越慢
```

大表（亿级）建议增加采样页数到 100+。

### 10.3 直方图维护

```sql
-- 创建
ANALYZE TABLE orders UPDATE HISTOGRAM ON status, country WITH 100 BUCKETS;

-- 更新
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;

-- 删除
ANALYZE TABLE orders DROP HISTOGRAM ON status;
```

定期跑 ANALYZE（每周或数据量变化 10% 后）。

### 10.4 Optimizer trace

看优化器具体怎么决策：

```sql
SET optimizer_trace = "enabled=on";
SELECT ... ;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G
SET optimizer_trace = "enabled=off";
```

输出 JSON 含每个候选 plan 的 cost 计算——调优深度排查必备。

---

## 第十一章：常见优化器陷阱

### 11.1 优化器误选索引

```sql
-- idx_user(user_id), idx_status(status)
SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
-- 业务知道 status='paid' 占 90% → 应该用 idx_user
-- 但优化器按 cardinality 错选 idx_status
```

修复：

- 加直方图：让优化器知道 status 分布
- 改成联合索引：`(user_id, status)` 一并解决
- Hint：`/*+ INDEX(orders idx_user) */`

### 11.2 隐式类型转换

```sql
-- phone VARCHAR
SELECT * FROM users WHERE phone = 13800138000;  -- 数字
-- → CAST(phone AS DECIMAL) = ... → 索引失效
```

详见 M02。

### 11.3 LIMIT 提前优化

```sql
SELECT * FROM big WHERE col1 = 'x' ORDER BY id LIMIT 10;
-- 优化器看到 LIMIT 10 + 主键排序，可能走 PK 顺序扫
-- 期望命中 col1='x' 时返回 10 个，提前结束
-- 但如果 col1='x' 的行很稀疏 → 实际扫遍全表才凑齐 10 个
```

修复：

- 强制 col1 索引：`USE INDEX(idx_col1)`
- 联合索引 `(col1, id)`

### 11.4 复杂 join 优化器卡死

10+ 表 join 时优化器搜索几秒钟。处理：

- `JOIN_FIXED_ORDER` Hint 跳过搜索
- 拆 SQL
- 8.0+ 用 hypergraph optimizer

### 11.5 NULL 的特殊性

```sql
SELECT * FROM t WHERE col IS NULL;     -- 索引可走
SELECT * FROM t WHERE col != 1;        -- 通常索引不走
```

NULL 影响 cardinality 估算 + 索引可用性。生产建议字段 NOT NULL + 默认值，避免 NULL 引发的优化器困扰。

---

## 第十二章：一次查询调优全流程

### 12.1 案例：业务报表慢

```sql
-- 业务 SQL：统计某用户某月订单
SELECT
    DATE(created_at) AS day,
    COUNT(*) AS cnt,
    SUM(amount) AS total
FROM orders
WHERE user_id = 12345
  AND created_at >= '2026-04-01'
  AND created_at < '2026-05-01'
GROUP BY DATE(created_at);
```

5 秒返回，业务期望 < 200ms。

### 12.2 步骤 1：EXPLAIN 看现状

```sql
EXPLAIN FORMAT=TREE
SELECT ...;
-- key: idx_user (user_id)
-- rows: 50000
-- Extra: Using temporary; Using filesort
```

发现：

- 走 `idx_user`，扫 5w 行
- 用了临时表 + filesort

### 12.3 步骤 2：业务理解

`idx_user` 命中 user_id=12345 的所有订单（50w 行），然后内存里再过滤时间。filesort 来自 `GROUP BY DATE(...)`。

### 12.4 步骤 3：改索引

```sql
CREATE INDEX idx_user_created ON orders(user_id, created_at);
```

```sql
EXPLAIN FORMAT=TREE
SELECT ...;
-- key: idx_user_created
-- rows: 1500   ← 时间过滤已下推
-- Extra: Using temporary; Using filesort   ← 还有
```

行数下降到 1500，但 filesort 还在。

### 12.5 步骤 4：覆盖索引 + 函数索引

```sql
ALTER TABLE orders ADD COLUMN day DATE GENERATED ALWAYS AS (DATE(created_at)) STORED;
CREATE INDEX idx_user_day_amount ON orders(user_id, day, amount);
```

```sql
SELECT day, COUNT(*), SUM(amount) FROM orders
WHERE user_id = 12345 AND day >= '2026-04-01' AND day < '2026-05-01'
GROUP BY day;
-- Extra: Using index; Using index for group-by
-- 完成时间：30ms
```

虚拟列 + 覆盖索引 + Loose index scan。

### 12.6 步骤 5：上线监控

```sql
-- 在 PS 看这条 SQL 的 digest
SELECT DIGEST_TEXT, COUNT_STAR, AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT LIKE '%user_id%amount%' ORDER BY COUNT_STAR DESC LIMIT 5;
```

观察 P95、P99 是否真降下来。

---

## 第十三章：JSON / TREE / ANALYZE 输出对比

### 13.1 表格 vs JSON vs TREE

```sql
-- 表格：紧凑但缺失成本
EXPLAIN SELECT ...;

-- JSON：所有细节
EXPLAIN FORMAT=JSON SELECT ...\G
-- 含每步 cost、estimated/used keys、可读注释

-- TREE：8.0+，最易读
EXPLAIN FORMAT=TREE SELECT ...\G
```

TREE 例：

```
-> Aggregate: count(0)
   -> Filter: (orders.status = 'paid')  (cost=12345 rows=1234)
      -> Index range scan on orders using idx_status  (cost=1000 rows=10000)
```

### 13.2 ANALYZE vs EXPLAIN

EXPLAIN：估算
ANALYZE：实际跑 + 估算 vs 实际行数对比

```
-> Filter: ...  (cost=1000 rows=100) (actual time=0.5..50.2 rows=200 loops=1)
                         ^^                              ^^
                       估算行数                          实际行数
```

差距大 → 统计信息失准。

---

## 第十四章：与其他章节连接

- **与 M02 索引**：优化器选索引的依据是 cardinality + 直方图
- **与 M03 事务**：MVCC 让 SELECT 不阻塞写，但 SELECT FOR UPDATE 走当前读 + 锁
- **与 M07 PS**：events_statements_summary_by_digest 看 SQL digest
- **与 M08 调优**：sort_buffer / join_buffer / tmp_table_size 调整

---

## 总结

优化器 + EXPLAIN 是 SQL 性能的诊断与处方书。关键点：

1. **cost model**：I/O + CPU 加权，可调
2. **统计信息**：CARDINALITY + 直方图（8.0+）
3. **EXPLAIN 字段**：type / key / rows / filtered / Extra
4. **type 性能**：const < eq_ref < ref < range < index < ALL
5. **Extra 警告**：filesort / temporary / Block Nested Loop
6. **Hash Join**：8.0+ 默认，无索引时大幅优于 BNL
7. **Hint**：紧急救火，长期靠索引 + 直方图
8. **优化器 trace**：看具体决策
9. **EXPLAIN ANALYZE**：估算 vs 实际行数对比

---

## 练习题

1. 解释 type 字段从 const 到 ALL 的性能含义。
2. EXPLAIN 显示 `Extra: Using filesort; Using temporary`，给 3 个排查方向。
3. 用直方图改善"status='paid' 占 90%"场景的优化器决策——写出全步骤。
4. Hash Join 比 BNL 快多少？什么场景退回 BNL？
5. 用 optimizer trace 看一次"为什么没选我建的索引"的决策。
6. 改 io_block_read_cost 让优化器更倾向于走索引——SSD 上合理值是多少？
7. 写一个会让 LIMIT 提前优化失效的场景（实际扫全表）。
8. 子查询 IN (SELECT ...) vs JOIN，哪个一定更快？哪个不一定？
9. 设计一个会触发 derived merge 的派生表 SQL。
10. 一个上线后稳定的 SQL 突然慢——可能哪些根因？

---

> 🔁 反馈：每条疑似慢 SQL 都跑一次 EXPLAIN ANALYZE，对比估算与实际
