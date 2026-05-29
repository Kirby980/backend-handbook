# 精通 PostgreSQL EXPLAIN 与执行计划：ANALYZE、BUFFERS、所有 Scan/Join 节点详解

> 关联章节：[P04 B-tree 索引](./04-精通-B-tree-索引.md)、[P05 多类型索引](./05-精通-多类型索引.md)、[P09 查询规划器](./09-精通-查询规划器.md)、[P11 查询调优实战](./11-精通-查询调优实战.md)、[P14 分区表](./14-精通-分区表.md)

---

## 引言：EXPLAIN 是 PostgreSQL 工程师的"听诊器"

如果说 [P09](./09-精通-查询规划器.md) 讲的是"planner 怎么想"，那么这一章讲的是"如何听见 planner 在想什么"。EXPLAIN 是 PG 提供的最强诊断工具——一条命令告诉你 cost 估算、行数估算、实际执行时间、共享缓冲区命中、临时文件落盘量、并行 worker 行为、JIT 编译耗时等几十个维度的细节。

但 EXPLAIN 输出对新手有巨大的认知门槛：节点类型 20+ 种、ANALYZE 之后的 actual loops 怎么读、为什么 estimate 是 5 行 actual 是 50 万、Bitmap Heap Scan 和 Index Scan 的区别在哪、Memoize 节点是干嘛的、Buffers 输出里 hit/read/dirtied/written 各代表什么。这一章会把这些字段全部讲清楚。

读完这一章你应该能：

- 列出 EXPLAIN 的所有选项（ANALYZE / BUFFERS / VERBOSE / SETTINGS / WAL / TIMING / SUMMARY / FORMAT / GENERIC_PLAN）的用途
- 区分 Seq Scan / Index Scan / Index Only Scan / Bitmap Scan / Tid Scan 各自适合的场景
- 区分 Nested Loop / Hash Join / Merge Join 的代价模型与选择条件
- 解读 `cost=0.00..1234.56 rows=100 width=42` 的每个数字
- 看出 `actual time=...` 里的 `loops` 是什么意思，正确计算总时间
- 用 BUFFERS 选项区分"内存命中"和"磁盘读取"，定位冷热数据问题
- 找出 estimate vs actual 偏差超过 10 倍的"高危节点"
- 用 auto_explain 自动记录线上慢查询的真实 plan
- 至少诊断 4 类典型问题：N+1 嵌套、Sort 落盘、行数估算偏差、并行未触发

---

## 第一章：EXPLAIN 全套选项

### 1.1 基础语法

```sql
EXPLAIN [ ( option [, ...] ) ] statement
EXPLAIN [ ANALYZE ] [ VERBOSE ] statement   -- 旧语法，2026 仍兼容
```

推荐用括号语法（PG 9.0+），可以任意组合：

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, FORMAT JSON)
SELECT ...;
```

### 1.2 全部选项一览

| 选项 | 默认 | 作用 | 自版本 |
|---|---|---|---|
| `ANALYZE` | off | **真正执行 SQL**，输出真实时间和行数 | 老版本 |
| `VERBOSE` | off | 显示输出列、schema 名、触发器等 | 老版本 |
| `COSTS` | on | 显示 cost 估算（关掉只看 plan 结构） | 9.0+ |
| `BUFFERS` | off | 显示 shared/local/temp buffer 命中/读取 | 9.0+；PG 13+ 默认 ANALYZE 时打开了一部分 |
| `WAL` | off | 显示产生的 WAL 字节数（写操作必看） | 13+ |
| `TIMING` | on(ANALYZE) | 测每个节点耗时（关掉减少 gettimeofday 开销） | 9.2+ |
| `SUMMARY` | auto | 显示 planning time / execution time 汇总 | 10+ |
| `SETTINGS` | off | 显示偏离默认值的 GUC 参数 | 12+ |
| `GENERIC_PLAN` | off | 看 prepared 语句的 generic plan，**不执行** | 16+ |
| `MEMORY` | off | 显示 plan 阶段的内存消耗 | 17+ |
| `SERIALIZE` | off | 模拟序列化输出到客户端的时间 | 17+ |
| `FORMAT` | TEXT | TEXT / XML / JSON / YAML 输出格式 | 9.0+ |

### 1.3 ANALYZE：真实执行 vs 估算

```sql
-- 不加 ANALYZE：只 plan 不执行
EXPLAIN SELECT * FROM orders WHERE status='paid';
-- 输出：Seq Scan ... (cost=0.00..1234.56 rows=950 width=42)

-- 加 ANALYZE：真的执行，多出 actual 数据
EXPLAIN (ANALYZE) SELECT * FROM orders WHERE status='paid';
-- 输出：Seq Scan ... (cost=0.00..1234.56 rows=950 width=42)
--                  (actual time=0.020..2.350 rows=948 loops=1)
-- Planning Time: 0.123 ms
-- Execution Time: 2.450 ms
```

**ANALYZE 的坑**：

1. **真的执行 SQL**——`EXPLAIN (ANALYZE) DELETE ...` 会删数据！必须包在事务里：
   ```sql
   BEGIN;
   EXPLAIN (ANALYZE) DELETE FROM orders WHERE created_at < '2020-01-01';
   ROLLBACK;
   ```
2. UPDATE/INSERT/DELETE 也都是真执行——同样包事务
3. EXPLAIN ANALYZE 有 `gettimeofday()` 开销，可能让查询慢 10%-30%。生产追查特别敏感的查询时可以 `EXPLAIN (ANALYZE, TIMING OFF)`

### 1.4 BUFFERS：内存命中分析

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE id = 42;
-- Index Scan using orders_pkey on orders ...
--   Buffers: shared hit=3
-- Execution Time: 0.080 ms
```

`Buffers:` 输出字段：

| 字段 | 含义 |
|---|---|
| `shared hit` | shared_buffers 命中页数（最快） |
| `shared read` | 从 OS / 磁盘读的页数（OS page cache 也算 read） |
| `shared dirtied` | 本次执行变脏的页 |
| `shared written` | 本次执行写回的页（一般是 backend 写脏页，bgwriter 也算） |
| `local hit/read/dirtied/written` | 临时表（CREATE TEMP TABLE）的缓冲 |
| `temp read` / `temp written` | 排序、hash 落盘到 `pgsql_tmp/` 的临时文件 |

**冷热判断**：
- 第一次执行：`shared read` 多 → 数据在磁盘 / OS cache
- 第二次执行：`shared hit` 多 → 已在 shared_buffers

**实战**：

```sql
-- 清空 OS cache 看真实冷数据性能（仅测试环境）
DISCARD ALL;
-- + 重启 / drop_caches 操作系统层面

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big WHERE x=1;
-- Buffers: shared read=50000        ← 5 万页磁盘读

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big WHERE x=1;
-- Buffers: shared hit=50000          ← 全部命中
```

### 1.5 VERBOSE：详细信息

```sql
EXPLAIN (VERBOSE) SELECT id, name FROM users WHERE city='NYC';
-- Index Scan using idx_city on public.users  ...
--   Output: id, name
--   Index Cond: (city = 'NYC'::text)
```

加了 VERBOSE 显示：

- schema 限定的表名
- 每个节点的 Output 列
- 表达式细节
- 触发器信息（INSERT/UPDATE/DELETE 时）

### 1.6 SETTINGS：相关的非默认参数

```sql
SET work_mem = '256MB';
EXPLAIN (SETTINGS) SELECT * FROM big ORDER BY name;
-- Sort ...
-- Settings: work_mem = '256MB'
```

诊断"昨天和今天 plan 不同"——是不是有人改了 GUC。

### 1.7 WAL：写操作必看

```sql
EXPLAIN (ANALYZE, WAL) UPDATE orders SET status='paid' WHERE id=42;
-- Update on orders ...
--   WAL: records=2  bytes=200
```

- `records`：产生多少 WAL 记录
- `bytes`：WAL 字节数
- `fpi`：full-page image 数量（checkpoint 后第一次修改页时产生，意味着大量 WAL）

### 1.8 GENERIC_PLAN（PG 16+）

prepared statement 的 generic plan 在没执行前看不到——PG 16 解决：

```sql
EXPLAIN (GENERIC_PLAN) SELECT * FROM orders WHERE status = $1;
-- 注意：参数用 $1，不能 EXECUTE 真正传值
```

定位"long-lived connection 的 prepared statement plan 异常"问题非常有用。

### 1.9 FORMAT：JSON 给工具吃

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

输出 JSON 适合输入到工具：
- [explain.dalibo.com](https://explain.dalibo.com)（可视化树）
- [explain.depesz.com](https://explain.depesz.com)（彩色 + 高亮慢节点）
- [pgmustard.com](https://www.pgmustard.com)（商业，AI 给建议）
- [pev2](https://github.com/dalibo/pev2)（开源可视化）

---

## 第二章：读 EXPLAIN 输出的基本功

### 2.1 节点缩进 = 树结构

```
Limit
  ->  Sort
        Sort Key: orders.created_at DESC
        ->  Hash Join
              Hash Cond: (orders.user_id = users.id)
              ->  Seq Scan on orders
              ->  Hash
                    ->  Seq Scan on users
```

执行顺序：**从最深的子节点开始，往上推**。Seq Scan on users → Hash → Seq Scan on orders → Hash Join → Sort → Limit。

### 2.2 cost / rows / width 的含义

```
Seq Scan on orders  (cost=0.00..1234.56 rows=950 width=42)
```

| 字段 | 含义 |
|---|---|
| `cost=0.00..1234.56` | startup_cost..total_cost（无单位，与 seq_page_cost=1.0 为基准） |
| `rows=950` | 估算返回行数 |
| `width=42` | 每行平均字节数（用于估算后续 sort / hash 内存） |

- **startup_cost**：返回第一行前的代价。`Sort` 节点的 startup 是 total（必须读完才能输出）；`Seq Scan` 的 startup 接近 0。
- **total_cost**：返回最后一行的代价
- **rows**：planner 估算；ANALYZE 时会多出 `actual rows`

### 2.3 actual time / loops

```
Index Scan using idx_user on orders  (cost=0.42..8.45 rows=3 width=42)
                                     (actual time=0.020..0.045 rows=3 loops=5)
```

| 字段 | 含义 |
|---|---|
| `actual time=0.020..0.045` | 单次执行的 startup..total 实际毫秒 |
| `rows=3` | 单次执行的实际行数 |
| `loops=5` | 这个节点被执行了几次 |

**关键陷阱**：`actual time` 是**单次**执行时间，**不是总时间**！

```
总耗时 ≈ actual time (total) × loops
```

例：Nested Loop 内层 Index Scan 显示 `actual time=0.020..0.050 rows=3 loops=10000` —— 实际总耗时 ≈ 0.050 × 10000 = 500ms，不是 0.050ms！

很多新手看到 0.050ms 觉得很快，没注意 loops=10000——这正是 N+1 问题的典型表征。

### 2.4 Planning Time vs Execution Time

```
Planning Time: 0.123 ms
Execution Time: 5.420 ms
```

- **Planning Time**：parser + analyzer + planner 全程，复杂查询可能秒级（24 表 join）
- **Execution Time**：executor 真正跑的时间

如果 Planning Time >> Execution Time（如 200ms plan + 5ms 跑），说明：

- 表太多，planner 枚举 join 顺序耗时
- prepared statement 应该启用（avoid 每次重 plan）
- 或者 SQL 复杂度过高，考虑拆解

---

## 第三章：Scan 节点详解

### 3.1 Seq Scan（顺序扫描）

```
Seq Scan on orders  (cost=0.00..1875.00 rows=100000 width=42)
  Filter: (status = 'paid'::text)
  Rows Removed by Filter: 5000
```

- 全表扫描，从第一页到最后一页
- Filter 是扫完后再过滤；`Rows Removed by Filter` 显示扫了多少但被过滤掉

**何时是好选择**：
- 小表（<= 一页或几页）
- 需要扫大比例（> 20-30%）行的查询
- 没有合适索引

**何时是坏选择**：
- 大表 + 高选择性条件 + 有索引 → 应该走 Index Scan
- 通常意味着 random_page_cost 太高或统计过时

### 3.2 Index Scan

```
Index Scan using idx_user on orders  (cost=0.42..8.45 rows=3 width=42)
  Index Cond: (user_id = 42)
```

- 通过 B-tree 索引找到匹配的 TID，再回表读 heap tuple
- `Index Cond`：能用索引解决的条件
- `Filter`：索引拿到 tuple 后再过滤的条件（不能下推到索引）

```
Index Scan using idx_user on orders
  Index Cond: (user_id = 42)
  Filter: (status = 'paid'::text)            ← 没在索引里
  Rows Removed by Filter: 50
```

复合索引 `(user_id, status)` 能把 Filter 也变成 Index Cond，避免回表后过滤。

### 3.3 Index Only Scan（只索引扫描）

```
Index Only Scan using idx_user_status on orders
  Index Cond: (user_id = 42)
  Heap Fetches: 0
```

- 只读索引，不访问 heap
- 前提：所有 SELECT 列都在索引里（覆盖索引 / INCLUDE 列）
- `Heap Fetches=0`：理想情况，全部从索引返回
- `Heap Fetches>0`：因为 **visibility map** 标记某些页有未冻结的更新，必须回 heap 检查可见性

**为什么需要 visibility map**：PG 的 MVCC 让索引不知道某行是否对当前事务可见，必须查 heap 的 xmin/xmax。VM 是一个 bit 数组，标记"这页所有元组都对所有事务可见"，避免回表。

**优化建议**：
- 经常 VACUUM 让 VM 更新（autovacuum 通常足够）
- INCLUDE 列把 SELECT 列加进索引：
  ```sql
  CREATE INDEX idx_user_status_inc ON orders(user_id) INCLUDE(status, amount);
  ```

### 3.4 Bitmap Index Scan + Bitmap Heap Scan

```
Bitmap Heap Scan on orders  (cost=4.16..23.40 rows=10 width=42)
  Recheck Cond: ((status = 'paid') OR (status = 'pending'))
  Heap Blocks: exact=8
  ->  BitmapOr
        ->  Bitmap Index Scan on idx_status  (rows=8)
              Index Cond: (status = 'paid')
        ->  Bitmap Index Scan on idx_status  (rows=2)
              Index Cond: (status = 'pending')
```

工作原理：

1. **Bitmap Index Scan**：扫索引但不立即回表，构建一个 TID bitmap（位图）
2. **BitmapOr / BitmapAnd**：合并多个 bitmap
3. **Bitmap Heap Scan**：按 heap 物理顺序读 bitmap 标记的页（顺序 IO 比 Index Scan 的随机 IO 快）
4. **Recheck Cond**：bitmap 可能"有损"（lossy，当行多时只记页号），所以读到页后还要 recheck

**何时选 bitmap**：
- 多个索引的 OR / AND 组合
- 选择性中等（不算高也不算低，比如 5%-30%）
- 单索引但行数足够多，让随机 IO 转顺序 IO 更划算

**Heap Blocks 字段**：
- `exact=N`：精确模式，N 个页
- `lossy=M`：lossy 模式（work_mem 不够时降级），M 个页只记了页号没记 tuple offset

```
Heap Blocks: exact=10000 lossy=50000
```

出现 lossy 大量时，调大 work_mem。

### 3.5 Index Scan vs Bitmap Scan 对比

| 维度 | Index Scan | Bitmap Heap Scan |
|---|---|---|
| 回表顺序 | 索引序（可能乱） | heap 物理序（顺序） |
| 多索引组合 | 不支持 | 支持 BitmapAnd / BitmapOr |
| 选择性低 | 适合（找几行） | 不适合（启动成本高） |
| 选择性中等 | 大量随机 IO | 顺序读，更快 |
| 选择性高 | 不如 Seq Scan | 同样不如 Seq Scan |
| LIMIT N | 拿到 N 行就停 | 必须先建 bitmap 才能扫 heap |
| ORDER BY 索引列 | 索引序匹配，能跳过 sort | 失去顺序，要重新 sort |

### 3.6 Tid Scan

```
Tid Scan on orders  (cost=0.00..4.01 rows=1 width=42)
  TID Cond: (ctid = '(0,1)')
```

- 通过 `WHERE ctid = '(page,offset)'` 直接访问指定物理位置
- 极少手写，多用于内核工具 / 高级运维（如分批清理）

### 3.7 Function Scan / Values Scan / Subquery Scan / CTE Scan

- **Function Scan**：`SELECT * FROM generate_series(1,100)`
- **Values Scan**：`VALUES ((1,'a'),(2,'b'))`
- **Subquery Scan**：嵌套子查询没能 pull-up 时
- **CTE Scan**：`WITH ... MATERIALIZED` 物化后扫描

### 3.8 Foreign Scan / Custom Scan

- **Foreign Scan**：postgres_fdw / file_fdw 等外部表
- **Custom Scan**：扩展实现的扫描节点（如 Citus 的分布式扫描、TimescaleDB 的 ChunkAppend）

---

## 第四章：Join 节点与算法

### 4.1 Nested Loop Join

```
Nested Loop  (cost=0.42..1234.56 rows=100 width=84)
  ->  Seq Scan on users u  (rows=10)
  ->  Index Scan using idx_user on orders o
        Index Cond: (user_id = u.id)
```

工作原理：对外表每一行，去内表找匹配。

```
for each row in outer:
    for each matching row in inner (via index or scan):
        emit join
```

**代价模型**：
```
cost ≈ outer_rows × inner_scan_cost_per_row
```

**适合场景**：
- 外表小（几十到几千行）
- 内表有高效查找（B-tree 等值 / 范围）
- LATERAL JOIN 通常走 nested loop

**不适合**：
- 外表大 + 内表无索引 → O(N×M) 灾难
- 外表大 + 内表索引但回表多 → 大量随机 IO

### 4.2 Nested Loop + Materialize

```
Nested Loop
  ->  Seq Scan on a
  ->  Materialize
        ->  Seq Scan on b
```

外表多次扫内表时，把内表先物化到内存 / 临时文件，避免重复 scan。出现 Materialize 节点通常是因为内表没法走索引但需要被外表多次访问。

### 4.3 Nested Loop + Memoize（PG 14+）

```
Nested Loop
  ->  Seq Scan on a  (rows=1000)
  ->  Memoize  (Cache Key: a.user_id, Hits: 950, Misses: 50, Evictions: 0)
        ->  Index Scan using idx_user on b
              Index Cond: (user_id = a.user_id)
```

Memoize 是 PG 14 引入的"参数化结果缓存"：

- 对内层扫描的输入参数缓存结果
- 下次相同参数直接命中缓存
- 类似 SQL 层面的 memoization

**关键指标**：
- `Hits`：缓存命中次数
- `Misses`：未命中（真正扫了内表）
- `Evictions`：因 work_mem 限制被驱逐的条目数

外层有大量重复 join key 时（如 N+1 模式、子查询关联），Memoize 收益巨大。

### 4.4 Hash Join

```
Hash Join  (cost=10.50..2345.67 rows=10000 width=84)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (rows=100000)
  ->  Hash  (rows=1000)
        Buckets: 1024  Batches: 1  Memory Usage: 50kB
        ->  Seq Scan on users u
```

工作原理：

1. **Build 阶段**：把内表（hash 输入）读完，构建 hash 表（key=join column，value=tuple）
2. **Probe 阶段**：扫外表，每行用 join key 在 hash 表查找

**代价模型**：
```
cost ≈ build_cost + probe_cost
     = inner_scan + outer_scan + hash_compute + lookups
```

**Batches > 1**：work_mem 装不下 hash 表，分批落盘
```
Buckets: 1024  Batches: 8  Memory Usage: 128MB
```

Batches=8 意味着 hash 表被分成 8 批，对外表也要分批 probe——慢 5-10 倍。调大 work_mem 或减少 build 端。

**适合场景**：
- 两表都较大
- 等值连接（hash 只支持 =）
- 内表能装进 work_mem

### 4.5 Parallel Hash Join（PG 11+）

```
Parallel Hash Join  (cost=...)
  Hash Cond: (...)
  ->  Parallel Seq Scan on orders
  ->  Parallel Hash  ...
        ->  Parallel Seq Scan on users
```

build 阶段也并行——hash 表存在动态共享内存（DSM）。需要 work_mem × workers 的空间。

### 4.6 Merge Join

```
Merge Join  (cost=...)
  Merge Cond: (o.user_id = u.id)
  ->  Index Scan using idx_orders_user on orders
  ->  Index Scan using users_pkey on users
```

工作原理：两表按 join key 排序后归并扫描。

```
i = j = 0
while i < N and j < M:
    if a[i].key < b[j].key: i++
    elif a[i].key > b[j].key: j++
    else: emit join; i++; j++
```

**代价模型**：两表 sort（若没有顺序）+ 一次归并扫描。

**适合场景**：
- 两表都极大
- 两表 join key 都已经有序（自然顺序或索引顺序）
- 等值或范围连接

**不适合**：
- 需要额外 sort（除非已经有 ORDER BY 用上同一顺序）

### 4.7 三大 join 算法对比

| 维度 | Nested Loop | Hash Join | Merge Join |
|---|---|---|---|
| 外表大小 | 外小 | 任意 | 任意 |
| 内表查找 | 索引 / 全扫 | hash 查找 | 顺序扫描 |
| join 类型 | 等值 / 范围 / 任意 | 仅等值 | 等值 / 范围 |
| 是否需要排序 | 否 | 否 | 是（除非已序） |
| 内存使用 | 几乎不用 | work_mem 装 hash | 排序 work_mem |
| 启动成本 | 低 | 中（build） | 高（sort） |
| 大表+大表 | 差 | 好 | 好（需排序） |
| 与 LIMIT 配合 | 极好 | 一般 | 一般 |

---

## 第五章：Sort 与 Aggregate 节点

### 5.1 Sort 节点

```
Sort  (cost=...) (actual ... rows=1000 loops=1)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 230kB
```

`Sort Method` 三种：

- **quicksort**：全在内存，最快
- **top-N heapsort**：有 LIMIT 时只保留前 N，O(N×log(K))
- **external merge Disk: 500MB**：放不下，落盘

```
Sort Method: external merge  Disk: 500MB
```

意味着 `work_mem` 不够，sort 慢一个数量级。修复：

```sql
SET work_mem = '512MB';
```

注意 work_mem 是单 sort/hash 的上限，全局并发开太大会爆内存。

### 5.2 Incremental Sort（PG 13+）

当输入已经按部分键排序时（如索引顺序），可以增量排序而不是全部排序。

```
Incremental Sort  (cost=...)
  Sort Key: a, b
  Presorted Key: a
  Full-sort Groups: 1000  Sort Method: quicksort  Average Memory: 25kB  Peak Memory: 30kB
  ->  Index Scan using idx_a on t
```

输入按 a 有序，每个相同 a 的小组内再排 b——内存友好，且能跳过 LIMIT 之外的部分。

PG 17 改进了 incremental sort 的选择条件，更主动用上。

### 5.3 Aggregate 节点家族

#### Plain Aggregate

```
Aggregate  (cost=...) (actual ... rows=1)
  ->  Seq Scan on orders
```

没有 GROUP BY 的聚合（如 `SELECT count(*) FROM ...`），单值。

#### HashAggregate

```
HashAggregate  (cost=...)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on orders
```

每组维护一个 hash 桶。PG 13+ 支持 disk spill（Batches > 1 时落盘）。

#### GroupAggregate

```
GroupAggregate  (cost=...)
  Group Key: status
  ->  Sort
        Sort Key: status
        ->  Seq Scan on orders
```

输入有序时使用——每读到新 key 就 emit 前一组聚合值。比 HashAggregate 内存友好。

PG planner 在两者之间选择，看输入是否已经有序、estimate 是否能在 work_mem 装下 hash 表。

#### Partial Aggregate + Finalize Aggregate（并行）

```
Finalize Aggregate
  ->  Gather
        Workers Planned: 4
        ->  Partial HashAggregate
              Group Key: status
              ->  Parallel Seq Scan on orders
```

并行聚合分两阶段：

1. 各 worker 局部 HashAggregate
2. leader 合并（Finalize）

### 5.4 WindowAgg 节点

```
WindowAgg  (cost=...)
  ->  Sort
        Sort Key: department, salary DESC
        ->  Seq Scan on employees
```

执行窗口函数（`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`）。需要按 PARTITION BY + ORDER BY 排序后扫描，维护窗口状态。

详见 [P13 高级 SQL](./13-精通-高级-SQL.md)。

---

## 第六章：Append / Gather / Limit 等"组合"节点

### 6.1 Append（分区表 / UNION ALL）

```
Append
  ->  Seq Scan on orders_2024_01
  ->  Seq Scan on orders_2024_02
  ->  Seq Scan on orders_2024_03
```

来自分区表 / UNION ALL 各分支。子节点结果直接拼接。

PG 11+ 有 **Parallel Append**：分支可以并行扫描。

### 6.2 MergeAppend（保留顺序）

```
Merge Append
  Sort Key: created_at DESC
  ->  Index Scan using ... on orders_2024_01
  ->  Index Scan using ... on orders_2024_02
```

各分支已按 sort key 有序，归并保留顺序——用于 `SELECT ... FROM partitioned_table ORDER BY created_at LIMIT 10`。

### 6.3 Gather / Gather Merge（并行）

```
Gather
  Workers Planned: 4
  Workers Launched: 4
  ->  Parallel Seq Scan on big
```

- **Gather**：leader 收集 worker 输出（不保留顺序，先到先 emit）
- **Gather Merge**：保留各 worker 的局部顺序，做归并

`Workers Planned` vs `Workers Launched`：如果 launched < planned，说明并行 worker 不够（`max_parallel_workers` / `max_parallel_workers_per_gather` 限制 / 其他查询占用）。

### 6.4 Limit

```
Limit  (cost=0.00..0.50 rows=10 width=42)
  ->  Index Scan using idx_created_at on orders
```

LIMIT 是一个独立节点——一旦上层拿够行就停止从下层拉取。这是为什么 LIMIT + ORDER BY 索引能极快。

### 6.5 SubPlan / InitPlan

```
Seq Scan on orders
  Filter: (user_id = $0)
  SubPlan 1
    ->  Aggregate
        ...
InitPlan 1 (returns $0)
  ->  Aggregate
        ...
```

- **InitPlan**：查询前算一次的子查询（标量值），结果存为 `$N` 参数
- **SubPlan**：每次外层调用都重新执行的子查询（相关子查询）

InitPlan 是优化版（只执行一次），SubPlan 是 N+1 的潜在风险。

---

## 第七章：BUFFERS 深入

### 7.1 全字段表

```
Buffers: shared hit=1000 read=50 dirtied=5 written=2
         local hit=0 read=0 dirtied=0 written=0
         temp read=100 written=100
```

| 字段组 | 字段 | 含义 |
|---|---|---|
| shared | hit | 在 shared_buffers 命中 |
| shared | read | 从 OS / 磁盘读到 shared_buffers |
| shared | dirtied | 本次执行变脏的页 |
| shared | written | 本次执行 backend 写出去的脏页 |
| local | * | 临时表（CREATE TEMP TABLE）专用 buffer pool |
| temp | read/written | sort/hash 落盘的临时文件 |

### 7.2 read 不等于"磁盘"

`shared read` 包括：

1. 磁盘真实物理读
2. OS page cache 命中（PG 看不到 OS cache，统一算 read）

要区分二者需要看 OS 层（iostat、blktrace）。但相对来说：
- shared hit：肯定是 RAM（shared_buffers）
- shared read：可能 OS cache，可能磁盘

### 7.3 temp 文件警告

```
Buffers: ... temp read=10000 written=10000
Sort Method: external merge  Disk: 80MB
```

`temp written` > 0 表示有 sort/hash 落盘，慢一个数量级。修复：

- 增大 work_mem
- 减少 sort 输入（加 WHERE / LIMIT / 索引利用）
- 检查 hash join 的 build 端是否选错

### 7.4 WAL 与 BUFFERS 配合

写操作必看 WAL：

```sql
EXPLAIN (ANALYZE, BUFFERS, WAL) UPDATE orders SET status='paid' WHERE id=42;
-- Update on orders
--   Buffers: shared hit=4 dirtied=1
--   WAL: records=2  bytes=200  fpi=1
```

- `WAL records=2`：xlog 记录数
- `bytes=200`：WAL 字节
- `fpi=1`：full-page image 数（checkpoint 后第一次写整页）

frequent FPI 意味着 checkpoint 太频繁，调 `checkpoint_timeout` / `max_wal_size`。

---

## 第八章：诊断方法论——找偏差

### 8.1 寻找"高危节点"

> 一句话原则：**estimate vs actual 偏差 > 10× 的节点几乎一定是问题根源**。

```
Seq Scan on orders  (cost=... rows=10 width=42)
                    (actual ... rows=500000 loops=1)
                    ↑ 估算 10，实际 500000 → 偏差 50000 倍
```

planner 基于错误估算选了 Seq Scan，但实际行数巨大——可能本应该走 Index Scan + LIMIT 早停。

### 8.2 找"loops 巨大"的节点

```
Index Scan using idx_user on orders
  (actual time=0.030..0.050 rows=3 loops=10000)
```

loops=10000 意味着外层迭代 1 万次，每次扫一遍内表——这是 N+1 的典型表征。即使每次 0.05ms，总耗时 500ms。

修复：
- 改写为 Hash Join 或 Merge Join（条件允许的话）
- 加 Memoize 提示（PG 14+ 自动）
- 减少外层行数

### 8.3 找"落盘"节点

```
Sort Method: external merge  Disk: 500MB
```
或
```
Buckets: ... Batches: 8 Memory Usage: 256MB
```
或
```
Buffers: ... temp written=100000
```

都意味着 work_mem 不够。调大 work_mem 或减少输入。

### 8.4 完整诊断流程

```
1. 跑 EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) 拿真实计划
2. 从顶往下找 actual time 最大的节点
3. 检查 rows estimate vs actual 偏差
4. 检查 loops 异常大的节点（N+1）
5. 检查 Sort/Hash 是否落盘（temp written / Disk: ...）
6. 检查 shared read >> hit 的节点（冷数据 / 缺缓存）
7. 根据问题定位修复：
   - 估算偏差 → ANALYZE / CREATE STATISTICS / target
   - N+1 → 改写为 join / 加 Memoize
   - 落盘 → work_mem
   - 冷数据 → 增大 shared_buffers / 预热
   - 索引未用 → random_page_cost / 加索引 / 改写 SQL
```

---

## 第九章：四个经典案例

### 9.1 案例 1：诊断 N+1（高 loops）

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name,
       (SELECT count(*) FROM orders o WHERE o.user_id = u.id) AS cnt
FROM users u;
```

输出：

```
Seq Scan on users u  (cost=0.00..10000 rows=10000 width=120)
                     (actual time=0.020..15000.000 rows=10000 loops=1)
  SubPlan 1
    ->  Aggregate  (actual ... rows=1 loops=10000)
          ->  Index Scan using idx_user on orders o
                (actual time=0.030..0.080 rows=5 loops=10000)
                Buffers: shared hit=50000
Execution Time: 15050 ms
```

- SubPlan loops=10000 → 每行外表执行一次
- Buffers shared hit=50000 = 10000 × 5 → 5 万次 buffer 读
- 总耗时 15 秒

**修复**：改写为 LEFT JOIN GROUP BY

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

```
HashAggregate  (actual ... rows=10000 loops=1)
  ->  Hash Right Join  (actual ... rows=50000 loops=1)
        ->  Seq Scan on orders o
        ->  Hash
              ->  Seq Scan on users u
Execution Time: 250 ms
```

60 倍提升。

### 9.2 案例 2：Sort 落盘

```
Sort  (cost=...) (actual time=8000.000..8500.000 rows=1000000 loops=1)
  Sort Key: amount DESC
  Sort Method: external merge  Disk: 1024MB
  ->  Seq Scan on big_orders
```

诊断：1 GB 排序落盘，慢 10 倍。

```sql
SHOW work_mem;   -- 4MB（默认）

-- 当前会话调大
SET work_mem = '2GB';
EXPLAIN ANALYZE SELECT * FROM big_orders ORDER BY amount DESC LIMIT 100;
```

```
Limit  (actual ... rows=100 loops=1)
  ->  Sort  (actual ... rows=100 loops=1)
        Sort Method: top-N heapsort  Memory: 30kB
```

加 LIMIT + top-N heapsort，只用 30KB 内存。如果不能加 LIMIT，至少调 work_mem 到能装下数据：

```
Sort Method: quicksort  Memory: 1.0GB
```

### 9.3 案例 3：行数估算偏差导致选错索引

```
Seq Scan on orders  (cost=0.00..2000 rows=10 width=42)
                    (actual rows=50000 loops=1)
  Filter: (city='Beijing' AND country='China')
  Rows Removed by Filter: 50000
```

estimate 10，actual 50000——5000 倍偏差。原因：planner 假设 city 和 country 独立，导致选择 Seq Scan + Filter 而非 Index。

修复：

```sql
CREATE STATISTICS stat_city_country (mcv, dependencies)
    ON city, country FROM orders;
ANALYZE orders;

EXPLAIN ANALYZE ...;  -- rows 估算修正
```

如果 plan 仍未改善，检查是否有合适索引：

```sql
CREATE INDEX idx_city_country ON orders(city, country);
```

### 9.4 案例 4：用 BUFFERS 看冷热数据

```
-- 第一次执行
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big WHERE id BETWEEN 1 AND 10000;
-- Index Scan ... (actual time=200.000..1500.000 rows=10000 loops=1)
--   Buffers: shared hit=2 read=10000
-- Execution Time: 1500 ms

-- 第二次执行
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big WHERE id BETWEEN 1 AND 10000;
-- Index Scan ... (actual time=2.000..15.000 rows=10000 loops=1)
--   Buffers: shared hit=10002
-- Execution Time: 15 ms
```

第一次 read=10000，磁盘读 10000 页；第二次全部 hit。100 倍加速来自 shared_buffers 命中。

实战推论：

- 热数据应该常驻 shared_buffers，调大它（建议 25% 物理内存）
- 冷数据偶尔被查（如月度报表），可以提前 `pg_prewarm` 预热

```sql
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('big');  -- 加载到 shared_buffers
```

---

## 第十章：auto_explain 与可视化工具

### 10.1 auto_explain：自动记录慢查询 plan

PG 自带扩展，自动记录超过阈值的查询计划到日志。

`postgresql.conf`：

```conf
shared_preload_libraries = 'auto_explain'   # 需重启
session_preload_libraries = 'auto_explain'  # 或会话级（无需重启）

auto_explain.log_min_duration = 100ms       # 慢于 100ms 记录
auto_explain.log_analyze = on               # 真实执行数据
auto_explain.log_buffers = on               # buffers 信息
auto_explain.log_timing = off               # 高负载关 timing 减少开销
auto_explain.log_verbose = on
auto_explain.log_format = json              # 适合工具处理
auto_explain.log_nested_statements = on     # 函数内 SQL 也记录
auto_explain.sample_rate = 1.0              # 1.0 = 全采样；0.1 = 10% 采样
```

**生产建议**：

- `log_analyze=on` 加 `log_timing=off` 减少开销
- `sample_rate=0.1` 高 QPS 时降采样
- 配合 pgBadger / `pg_stat_statements` 分析

### 10.2 可视化工具对比

| 工具 | 类型 | 特点 |
|---|---|---|
| [explain.depesz.com](https://explain.depesz.com) | 在线 | 彩色高亮慢节点、Buffers 表格化 |
| [explain.dalibo.com](https://explain.dalibo.com) | 在线 | 树形可视化（pev2 在线版） |
| [pgmustard.com](https://www.pgmustard.com) | 在线 / API | 商业，AI 给具体建议 |
| [pev2](https://github.com/dalibo/pev2) | 开源前端组件 | 可嵌入运维平台 |
| pgBadger | 离线日志分析 | log 聚合 + 慢查询统计 |
| [pganalyze](https://pganalyze.com) | SaaS | 长期 plan 历史、统计、报警 |

推荐组合：日常排查用 depesz / dalibo，生产监控装 pganalyze 或 pgmustard。

---

## 生产实践

### 标准排查模板

```sql
-- 1. 关键参数检查
SHOW work_mem;
SHOW random_page_cost;
SHOW effective_cache_size;
SHOW max_parallel_workers_per_gather;

-- 2. 完整 EXPLAIN
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) <slow SQL>;

-- 3. 看历史 plan
SELECT
    query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
WHERE query LIKE '%大致 SQL 形态%'
ORDER BY mean_exec_time DESC LIMIT 5;

-- 4. 看表统计
SELECT * FROM pg_stats WHERE tablename='orders' AND attname='status';

-- 5. 看索引使用
SELECT * FROM pg_stat_user_indexes WHERE relname='orders';
```

### 写操作排查

```sql
EXPLAIN (ANALYZE, BUFFERS, WAL)
UPDATE orders SET status='paid' WHERE id IN (1,2,3);
```

关注：
- `Rows Removed by Filter`（多扫了多少）
- `WAL records / bytes / fpi`
- 触发器（VERBOSE 显示）

### 定期 review

- 每周用 pg_stat_statements 找出 top 20 慢 SQL
- 跑 `EXPLAIN (ANALYZE, BUFFERS)` 各一次
- 用 explain.depesz.com 上传 plan 找出"红色"节点
- 对 estimate/actual 偏差 > 10x 的写 ticket 跟进

### auto_explain + pgBadger

线上常驻 auto_explain，定期 pgBadger 分析日志：

```
pgbadger -j 4 -f stderr /var/log/postgresql/postgresql-*.log -o report.html
```

报告包含慢查询 top N + 自动聚类 + plan 节选。

---

## 陷阱清单

1. **EXPLAIN ANALYZE 真的执行**——DELETE/UPDATE/INSERT 必须包事务 + ROLLBACK。
2. **actual time 是单次时间**，总耗时 = actual × loops。Nested Loop 内层这个数字最容易误读。
3. **shared read 不等于磁盘**——OS cache 也算 read。要区分需要看 OS 工具。
4. **temp written > 0 = 落盘**——sort / hash 内存不够，慢一个数量级。
5. **Bitmap Heap Scan 的 lossy=N** 表示 work_mem 不够 bitmap 退化。看到大量 lossy 调 work_mem。
6. **Heap Fetches > 0 的 Index Only Scan** 不够"only"——VACUUM 没跟上 visibility map。
7. **生产 SET work_mem = '2GB'** 全局这样设会爆内存（每个 sort/hash 都能用这么多 × 并发数）。
8. **Workers Launched < Planned**——max_parallel_workers 不够或被其他查询占用，并行没真正生效。
9. **rows estimate=1** 时 planner 倾向 Nested Loop——估算 1 行时 Nested Loop 永远最便宜。如果实际几千行就是灾难。这是 estimate 偏差最危险的形式。
10. **PG 12 之前 CTE 是 fence**——老 SQL 升级到 PG 12+ 后 plan 行为可能变化（CTE 内联）。
11. **prepared statement 的 plan 看不到** ——用 PG 16 的 `EXPLAIN (GENERIC_PLAN)`。
12. **JIT 编译时间计入 Execution Time**（单独的 JIT 块统计）——OLTP 小查询可能因为 JIT 反而变慢，调高 `jit_above_cost`。
13. **EXPLAIN 输出与实际执行不一致**——VERBOSE / SETTINGS 帮你发现某些 GUC 被 session 覆盖了。
14. **看 EXPLAIN 不看 BUFFERS**——很多"慢"是因为冷数据 IO，不是 plan 错了。

---

## 2026 现状

- **PG 17 新选项**：`MEMORY`（plan 阶段内存）/ `SERIALIZE`（模拟序列化到客户端）
- **PG 16 GENERIC_PLAN**：终于能不执行就看 prepared statement 的 generic plan
- **PG 16 改进 parallel hash join**：build 也并行，大表 join 进一步加速
- **PG 14 Memoize**：相关子查询和 LATERAL JOIN 的 N+1 杀手锏
- **PG 13 incremental sort**：与索引部分顺序匹配时的优化
- **PG 13 HashAggregate disk spill**：再也不会因为 group by 内存不够而失败
- **PG 18 异步 IO (io_uring)**：影响 Buffers read 的实际开销模型；EXPLAIN 输出格式不变，但底层"read"快了
- **pganalyze / pgmustard 商业化**：基于 plan + 历史趋势的 AI 建议进入主流
- **EXPLAIN ANALYZE 仍是真执行**——这一条永远不会变，写操作永远先 BEGIN ... ROLLBACK
- **pgvector / pgvectorscale 的 plan**：新增 Custom Scan 节点（HNSW Index Scan 等）

---

## 练习题

1. 给一条 SQL `SELECT * FROM orders WHERE user_id = 42 AND status = 'paid' ORDER BY created_at DESC LIMIT 10`。请写出三种可能的 plan 形态（Seq Scan / Index Scan / Bitmap Scan），并分析各自适合的统计/数据特征。

2. EXPLAIN ANALYZE 输出 `actual time=0.030..0.080 rows=3 loops=10000`，总耗时是多少？这是什么 plan node 的典型表征？如何修复？

3. 看到 `Sort Method: external merge Disk: 800MB`，给出至少 3 种修复方案，按"对其他查询影响小到大"排序。

4. 比较 Index Scan、Index Only Scan、Bitmap Heap Scan 三种节点。请用真实表举例（描述索引定义 + SQL + 各节点何时被选）。

5. `EXPLAIN (ANALYZE, BUFFERS)` 输出 `shared hit=500 read=10000`。第二次执行同样 SQL，输出 `shared hit=10500 read=0`。这说明什么？如何利用这个特性优化报表查询？

6. 解释 Nested Loop + Memoize（PG 14+）的工作原理。在什么场景下它能把 Nested Loop 从灾难变成优秀方案？写一个具体 SQL 例子。

7. 写一段 postgresql.conf 片段配置 auto_explain，要求：记录 > 200ms 的查询，包含真实数据但不开 timing，采样率 50%，记录嵌套语句。

8. `EXPLAIN` 显示 `Workers Planned: 4` 但 `Workers Launched: 1`。可能原因有哪些？给出排查步骤。

---

## 延伸阅读

- 官方文档：[Using EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html)、[Planner Cost Constants](https://www.postgresql.org/docs/18/runtime-config-query.html)
- 源码：`src/backend/commands/explain.c`、`src/backend/executor/`
- depesz 博客：[Explaining the unexplainable](https://www.depesz.com/2013/04/16/explaining-the-unexplainable/)（系列 6 篇，必读）
- Egor Rogov《PostgreSQL 14 Internals》Part III Query Execution
- pganalyze 博客：[Understanding PostgreSQL EXPLAIN ANALYZE Output](https://pganalyze.com/)
- pgmustard Indexing Advisor：[pgmustard.com](https://www.pgmustard.com)
- pev2 文档与示例：[github.com/dalibo/pev2](https://github.com/dalibo/pev2)
- auto_explain：[官方文档](https://www.postgresql.org/docs/18/auto-explain.html)
