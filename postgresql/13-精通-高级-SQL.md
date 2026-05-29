# 精通高级 SQL：窗口函数、CTE、LATERAL、MERGE、GROUPING SETS

> 关联章节：[P10 EXPLAIN 与执行计划](./10-精通-EXPLAIN-与执行计划.md)、[P11 查询调优实战](./11-精通-查询调优实战.md)、[P12 JSONB 与全文检索](./12-精通-JSONB-与全文检索.md)

---

## 引言：现代 SQL 不是 1992 年的 SQL

很多后端工程师对 SQL 的认知停留在"SELECT/JOIN/GROUP BY/UNION"——这是 SQL-92。如果你只用这些，你会把大量本属于数据库的逻辑搬到应用层，写一堆胶水代码，往返数据库 N 次，性能差、代码丑、Bug 多。

过去三十年标准在持续演进，PostgreSQL 是这些标准最早最完整的实现：

| 特性 | 标准 | PG 支持版本 |
|---|---|---|
| 窗口函数 | SQL:2003 | 8.4 |
| CTE / WITH | SQL:1999 | 8.4 |
| RECURSIVE | SQL:1999 | 8.4 |
| LATERAL JOIN | SQL:1999 | 9.3 |
| ROLLUP / CUBE / GROUPING SETS | SQL:1999 | 9.5 |
| INSERT ... ON CONFLICT (upsert) | PG 扩展 | 9.5 |
| Frame: GROUPS | SQL:2011 | 11 |
| MERGE | SQL:2003 | 15 |
| MERGE...RETURNING | PG 扩展 | 17 |
| WITH RECURSIVE ... CYCLE | SQL:1999 | 14 |
| DISTINCT ON | PG 扩展 | 远古 |
| TABLESAMPLE | SQL:2003 | 9.5 |

这一章覆盖的就是这些"现代 SQL 工具箱"——重点不是语法手册，而是**什么场景该用、什么场景反例、如何配合 EXPLAIN 验证执行计划**。

读完之后你应该能：

- 默写 ROW_NUMBER / RANK / DENSE_RANK / NTILE / PERCENT_RANK 语义
- 用 LAG / LEAD 写"环比同比"，用 FIRST_VALUE / NTH_VALUE 写"首单/复购"
- 区分 ROWS / RANGE / GROUPS 三种 frame 的差异
- 用 LATERAL 优雅地写"每个用户 Top 3 订单"
- 用 WITH RECURSIVE 写图遍历，并用 CYCLE 防环
- 区分 CTE 的 MATERIALIZED / NOT MATERIALIZED 行为
- 用 MERGE 写 upsert + delete 的三态合并
- 用 GROUPING SETS / ROLLUP / CUBE 写多维报表
- 用 INSERT ... ON CONFLICT 写并发安全的 upsert
- 用 DISTINCT ON 取每组首行（PG 独门技）
- 用 TABLESAMPLE 抽样大表

---

## 第一章：窗口函数（Window Functions）

### 1.1 一句话定义

窗口函数 = **不聚合行的聚合**——保留所有行，给每行额外算一个跨行的值。

```sql
-- GROUP BY：3 行变 1 行
SELECT dept, AVG(salary) FROM emp GROUP BY dept;

-- 窗口函数：保留所有行，每行多一列"该部门平均薪资"
SELECT name, dept, salary,
       AVG(salary) OVER (PARTITION BY dept) AS dept_avg
FROM emp;
```

### 1.2 完整语法

```sql
function() OVER (
  [PARTITION BY col1, col2, ...]   -- 分窗
  [ORDER BY col3 ASC|DESC NULLS FIRST|LAST]  -- 窗内排序
  [frame_clause]                    -- 窗内子范围
)
```

或用**命名窗口**（多函数共享窗口定义）：

```sql
SELECT
  AVG(salary) OVER w AS avg_sal,
  RANK() OVER w AS rank
FROM emp
WINDOW w AS (PARTITION BY dept ORDER BY salary DESC);
```

### 1.3 三种 Frame：ROWS / RANGE / GROUPS

Frame 决定窗内"当前行的邻居"——这是窗口函数最容易出错的地方。

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW       -- 物理行
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW      -- 值范围
GROUPS BETWEEN 2 PRECEDING AND 2 FOLLOWING             -- 同值组（PG 11+）
```

三者差异：

| 维度 | ROWS | RANGE | GROUPS（PG 11+） |
|---|---|---|---|
| "前 N" 含义 | 前 N 物理行 | ORDER BY 值差 N（仅数值/时间） | 前 N 个不同值组 |
| 相同 ORDER BY 值如何处理 | 视为独立行 | 视为同一行（绑定）| 视为同一组 |
| 默认值（带 ORDER BY） | — | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | — |
| 默认值（无 ORDER BY） | 整个分区 | 整个分区 | — |
| 典型用途 | 移动窗口（精确 N 行） | 时间窗口（最近 7 天） | 同分类聚合 |

**关键陷阱**：默认 frame 是 `RANGE`，对重复值的处理可能与你想的不同。

```sql
-- emp 中有多个 salary=5000 的人
SELECT name, salary,
       SUM(salary) OVER (ORDER BY salary) AS s_default,  -- RANGE
       SUM(salary) OVER (ORDER BY salary ROWS UNBOUNDED PRECEDING) AS s_rows
FROM emp;

-- 对于 salary=5000 的所有行，s_default 都相同（绑定累加）
-- s_rows 每行不同（一行一行累加）
```

### 1.4 排名类函数

| 函数 | 行为 |
|---|---|
| `ROW_NUMBER()` | 1, 2, 3, 4, 5（永不重复，相同值任意排） |
| `RANK()` | 1, 2, 2, 4, 5（并列后跳号） |
| `DENSE_RANK()` | 1, 2, 2, 3, 4（并列后不跳） |
| `NTILE(N)` | 把数据均分 N 桶，返回桶号 1..N |
| `PERCENT_RANK()` | `(rank - 1) / (count - 1)`，0..1 |
| `CUME_DIST()` | `小于等于当前值的行数 / 总行数`，0..1 |

经典："每部门薪资 Top 3"

```sql
SELECT dept, name, salary
FROM (
  SELECT dept, name, salary,
         ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
  FROM emp
) t
WHERE rn <= 3;
```

注意 PostgreSQL **不支持 QUALIFY** 子句（Snowflake/Trino 才有）。永远要用子查询或 CTE 包一层。

### 1.5 偏移类函数

| 函数 | 行为 |
|---|---|
| `LAG(col, n, default)` | 前 n 行的值（默认 n=1） |
| `LEAD(col, n, default)` | 后 n 行的值 |
| `FIRST_VALUE(col)` | 窗内第一个值 |
| `LAST_VALUE(col)` | 窗内最后一个值（注意 frame！） |
| `NTH_VALUE(col, n)` | 第 n 个值 |

**`LAST_VALUE` 的经典陷阱**：

```sql
-- 默认 frame 是 RANGE UNBOUNDED PRECEDING AND CURRENT ROW
-- LAST_VALUE 永远 = 当前行的值！
SELECT day, sales,
       LAST_VALUE(sales) OVER (ORDER BY day) AS wrong  -- 错
FROM daily_sales;

-- 正确写法：扩到整个分区
SELECT day, sales,
       LAST_VALUE(sales) OVER (ORDER BY day
                              ROWS BETWEEN UNBOUNDED PRECEDING
                                   AND UNBOUNDED FOLLOWING) AS ok
FROM daily_sales;
```

环比同比示例：

```sql
SELECT day, sales,
       sales - LAG(sales, 1, 0) OVER (ORDER BY day) AS d_o_d,
       sales - LAG(sales, 7, 0) OVER (ORDER BY day) AS w_o_w,
       sales - LAG(sales, 30, 0) OVER (ORDER BY day) AS m_o_m
FROM daily_sales;
```

### 1.6 聚合 + OVER

任何聚合函数都可以加 `OVER`：

```sql
-- 累计销售
SELECT day, sales,
       SUM(sales) OVER (ORDER BY day) AS cum_sales
FROM daily_sales;

-- 7 日移动平均
SELECT day, sales,
       AVG(sales) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;

-- 时间窗口：最近 30 天滚动求和
SELECT day, sales,
       SUM(sales) OVER (ORDER BY day
                        RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW) AS r30
FROM daily_sales;
```

### 1.7 性能与执行计划

窗口函数底层通常需要：

1. 按 PARTITION BY + ORDER BY 排序（或 hash 分组）
2. 单次扫描窗内计算
3. 输出

ASCII 示意：

```
        ┌───────────┐
        │ Sort by   │  ← 如果索引匹配 PARTITION BY + ORDER BY 可省
        │ part/ord  │
        └─────┬─────┘
              ↓
        ┌───────────┐
        │ WindowAgg │  ← Frame 在此计算
        └─────┬─────┘
              ↓
            output
```

EXPLAIN 输出节点：`WindowAgg`。给 PARTITION BY + ORDER BY 列建组合索引可以省去 Sort。

**WHERE 不能直接用窗口函数**：

```sql
-- 错：窗口函数在 SELECT 之后才求值
SELECT * FROM emp WHERE ROW_NUMBER() OVER (...) <= 3;

-- 对：包子查询
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (...) rn FROM emp) t WHERE rn <= 3;
```

### 1.8 常用模式速查

**找连续登录 N 天的用户**：

```sql
SELECT user_id, MIN(login_day) AS start_day, COUNT(*) AS streak
FROM (
  SELECT user_id, login_day,
         login_day - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_day))::int AS grp
  FROM user_login
) t
GROUP BY user_id, grp
HAVING COUNT(*) >= 7;
```

**每组取最新一行**：

```sql
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY updated_at DESC) AS rn
  FROM events
) t
WHERE rn = 1;
```

**累计占比**：

```sql
SELECT product, sales,
       SUM(sales) OVER (ORDER BY sales DESC) * 1.0 / SUM(sales) OVER () AS cum_pct
FROM products;
```

**Gap 检测**（找出连续 ID 中的断号）：

```sql
SELECT id + 1 AS gap_start, next_id - 1 AS gap_end
FROM (
  SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id FROM t
) s
WHERE next_id - id > 1;
```

---

## 第二章：LATERAL JOIN

### 2.1 概念

`LATERAL` 让 JOIN 的右侧子查询/函数**能引用左侧表的列**，等价于"对左侧每一行做一次右侧子查询"。SQL-92 不允许这种引用。

```sql
SELECT u.id, u.name, t.top_orders
FROM users u
JOIN LATERAL (
  SELECT jsonb_agg(o ORDER BY o.amount DESC) FILTER (WHERE rn <= 3) AS top_orders
  FROM (SELECT o.*, ROW_NUMBER() OVER (ORDER BY amount DESC) rn FROM orders o WHERE o.user_id = u.id) o
) t ON true;
```

或更典型——**每个用户 Top 3 订单**：

```sql
SELECT u.id, u.name, o.id AS order_id, o.amount
FROM users u
JOIN LATERAL (
  SELECT id, amount FROM orders
  WHERE user_id = u.id
  ORDER BY amount DESC
  LIMIT 3
) o ON true;
```

如果不用 LATERAL，要写成窗口函数 + 子查询过滤，可读性差且优化器有时给不出好计划。

### 2.2 LATERAL 与函数

集合返回函数自动 LATERAL（无需写关键字）：

```sql
SELECT u.id, e.value
FROM users u, jsonb_array_elements_text(u.tags) AS e(value);
-- 等价于
SELECT u.id, e.value
FROM users u, LATERAL jsonb_array_elements_text(u.tags) AS e(value);
```

### 2.3 LATERAL vs 关联子查询

关联子查询写在 SELECT 中只能返回一个值；LATERAL 可以返回多行多列：

```sql
-- 关联子查询：只能一个标量
SELECT u.id, (SELECT MAX(amount) FROM orders WHERE user_id = u.id) AS max_amt
FROM users u;

-- LATERAL：可以返回多列多行
SELECT u.id, m.max_amt, m.last_day
FROM users u,
LATERAL (
  SELECT MAX(amount) AS max_amt, MAX(created_at)::date AS last_day
  FROM orders WHERE user_id = u.id
) m;
```

### 2.4 LATERAL 的执行计划

EXPLAIN 中通常体现为 `Nested Loop` + `SubPlan`/`Function Scan`：

```
Nested Loop
  -> Seq Scan on users u
  -> Limit
       -> Index Scan on orders_user_idx
            Index Cond: (user_id = u.id)
```

要点：右侧子查询通常需要有 `user_id` 上的索引才不会爆炸。

---

## 第三章：CTE 与递归

### 3.1 普通 CTE

```sql
WITH dept_avg AS (
  SELECT dept, AVG(salary) AS avg_sal FROM emp GROUP BY dept
),
top_depts AS (
  SELECT dept FROM dept_avg WHERE avg_sal > 10000
)
SELECT * FROM emp e
JOIN top_depts USING (dept)
JOIN dept_avg USING (dept);
```

可读性 + 复用性：同一 CTE 在主查询多处引用一次定义即可。

### 3.2 MATERIALIZED / NOT MATERIALIZED（PG 12+ 关键变化）

PG 12 之前：CTE 总是"优化栅栏"——独立物化执行，规划器不能下推谓词。
PG 12 之后：**默认自动 inline（只引用一次且无副作用时）**；可以显式控制。

```sql
WITH a AS              (SELECT * FROM big_table)  -- PG 12+ 自动 inline
WITH a AS MATERIALIZED (SELECT * FROM big_table)  -- 强制物化
WITH a AS NOT MATERIALIZED (SELECT * FROM big_table)  -- 强制 inline
```

**实战意义**：

- 想强制把复杂子查询变成"先算一次再用"（防止 planner 重复展开）→ `MATERIALIZED`
- 想让 planner 把 CTE 当子查询优化（下推谓词）→ `NOT MATERIALIZED` 或依赖默认

| 场景 | 应该用 |
|---|---|
| CTE 包含 INSERT/UPDATE/DELETE（有副作用） | 自动 MATERIALIZED（强制） |
| 引用 ≥ 2 次的 CTE | 自动 MATERIALIZED |
| 引用 1 次的纯 SELECT CTE | 自动 inline（PG 12+） |
| CTE 内有 LIMIT，希望先 LIMIT 再 JOIN | MATERIALIZED |
| CTE 内带过滤，希望 planner 把外部条件下推进去 | NOT MATERIALIZED |

### 3.3 与派生表对比

```sql
-- 派生表
SELECT * FROM (SELECT ...) t1 JOIN (SELECT ...) t2 ON ...

-- CTE
WITH t1 AS (SELECT ...), t2 AS (SELECT ...) SELECT * FROM t1 JOIN t2 ON ...
```

CTE 优势：可读、可复用、可递归。

### 3.4 递归 CTE

语法：

```sql
WITH RECURSIVE cte_name AS (
  -- anchor: 基础查询（不引用自己）
  SELECT ...
  UNION [ALL]
  -- recursive: 引用自己
  SELECT ... FROM cte_name WHERE ...
)
SELECT * FROM cte_name;
```

执行流程：

```
   ┌─────────────┐
   │  Anchor     │ ← 执行一次，结果放工作表
   └──────┬──────┘
          ↓
   ┌─────────────┐
   │ Recursive   │ ← 引用上一轮工作表，迭代直到工作表为空
   └──────┬──────┘
          ↓
   ┌─────────────┐
   │   UNION     │ ← UNION 去重 / UNION ALL 不去重
   └─────────────┘
```

### 3.5 经典案例：组织树

```sql
CREATE TABLE org (id INT PRIMARY KEY, name TEXT, parent_id INT REFERENCES org);

INSERT INTO org VALUES
  (1, 'CEO', NULL),
  (2, 'CTO', 1), (3, 'CFO', 1),
  (4, '架构师', 2), (5, '后端', 2), (6, '前端', 2);

WITH RECURSIVE tree AS (
  SELECT id, name, parent_id, 0 AS depth, ARRAY[id] AS path
  FROM org WHERE parent_id IS NULL
  UNION ALL
  SELECT o.id, o.name, o.parent_id, t.depth + 1, t.path || o.id
  FROM org o JOIN tree t ON o.parent_id = t.id
)
SELECT REPEAT('  ', depth) || name AS hier, path FROM tree ORDER BY path;
```

输出：

```
CEO
  CTO
    架构师
    后端
    前端
  CFO
```

### 3.6 图遍历：找两点间所有路径

```sql
WITH RECURSIVE paths AS (
  SELECT src, dst, ARRAY[src, dst] AS path FROM edges WHERE src = 1
  UNION ALL
  SELECT p.src, e.dst, p.path || e.dst
  FROM paths p JOIN edges e ON e.src = p.dst
  WHERE NOT e.dst = ANY(p.path)   -- 手动防环
)
SELECT * FROM paths WHERE dst = 10;
```

### 3.7 CYCLE 子句（PG 14+）

PG 14 引入标准 `CYCLE` 子句，自动检测环：

```sql
WITH RECURSIVE paths AS (
  SELECT src, dst FROM edges WHERE src = 1
  UNION ALL
  SELECT p.src, e.dst FROM paths p JOIN edges e ON e.src = p.dst
)
CYCLE dst SET is_cycle USING cycle_path
SELECT * FROM paths;
```

`CYCLE col SET flag USING path` 自动维护：
- `cycle_path`：路径数组
- `is_cycle`：当前行是否形成环

### 3.8 Mandelbrot 集

递归 CTE 甚至可以画 ASCII Mandelbrot 集（社区经典 demo）：

```sql
WITH RECURSIVE x(i) AS (SELECT 0 UNION ALL SELECT i+1 FROM x WHERE i < 100),
Z(Ix, Iy, Cx, Cy, X, Y, I) AS (
  SELECT Ix, Iy, X::FLOAT, Y::FLOAT, X::FLOAT, Y::FLOAT, 0
  FROM (SELECT -2.2 + 0.031 * i AS X, i AS Ix FROM x) AS xgen
  CROSS JOIN (SELECT -1.5 + 0.031 * i AS Y, i AS Iy FROM x WHERE i < 50) AS ygen
  UNION ALL
  SELECT Ix, Iy, Cx, Cy, X*X - Y*Y + Cx AS X, 2*X*Y + Cy, I + 1
  FROM Z WHERE X*X + Y*Y < 16.0 AND I < 27
)
SELECT array_to_string(
         array_agg(SUBSTRING(' .,,,-----++++%%%%@@@@#### ', 1 + LEAST(I, 26), 1)
                  ORDER BY Ix), '') AS Mandel
FROM (SELECT Ix, Iy, MAX(I) AS I FROM Z GROUP BY Iy, Ix) AS t GROUP BY Iy ORDER BY Iy;
```

（运行试试，神奇）

### 3.9 递归 CTE 的爆炸防御

- 优先 `UNION`（自动去重），仅在确认数据无环时用 `UNION ALL`
- 加 `WHERE depth < N` 限制最大层数
- 路径数组 + `NOT = ANY()` 防环
- PG 14+ 用 `CYCLE` 子句

---

## 第四章：MERGE 与 ON CONFLICT

### 4.1 ON CONFLICT（PG 9.5+）——单表 upsert 神器

```sql
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES (42, 1, now())
ON CONFLICT (user_id) DO UPDATE
  SET login_count = user_stats.login_count + 1,
      last_login  = EXCLUDED.last_login;
```

要点：
- `EXCLUDED.*` 指代试图插入的那一行
- `ON CONFLICT DO NOTHING` 用于"插入或跳过"
- 必须有匹配的唯一索引/PK
- 并发安全（不会出现两个 INSERT 都成功的双写）

### 4.2 部分索引 + ON CONFLICT

```sql
CREATE UNIQUE INDEX ux_active_email ON users (email) WHERE deleted_at IS NULL;

INSERT INTO users (email, name) VALUES ('a@b.com', 'Alice')
ON CONFLICT (email) WHERE deleted_at IS NULL DO UPDATE SET name = EXCLUDED.name;
```

### 4.3 MERGE（PG 15+）

`MERGE` 是 SQL:2003 标准，比 ON CONFLICT 更强大：

- 可以同时 INSERT / UPDATE / DELETE
- 可以基于复杂条件分支
- 不限于唯一索引冲突

```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED AND s.qty = 0 THEN DELETE
WHEN MATCHED               THEN UPDATE SET qty = t.qty + s.qty
WHEN NOT MATCHED           THEN INSERT (id, qty) VALUES (s.id, s.qty);
```

适用场景：
- ETL 增量同步
- 库存对账
- CDC 应用变更
- 报表批量更新

### 4.4 MERGE...RETURNING（PG 17+）

PG 17 增加了 `RETURNING`，可以拿回每行 MERGE 的操作类型：

```sql
MERGE INTO target t
USING source s ON t.id = s.id
WHEN MATCHED THEN UPDATE SET qty = s.qty
WHEN NOT MATCHED THEN INSERT VALUES (s.id, s.qty)
RETURNING merge_action() AS action, t.*;
-- action 列返回 'INSERT' / 'UPDATE' / 'DELETE'
```

这是过去几年 MERGE 最缺的能力——以前要 RETURNING 必须用 CTE 包装。

### 4.5 ON CONFLICT vs MERGE 选择

| 维度 | ON CONFLICT | MERGE |
|---|---|---|
| 标准 | PG 扩展 | SQL:2003 |
| 单表 / 多表源 | 单表 INSERT | INSERT/UPDATE/DELETE + 任意 USING |
| 并发安全 | 强（基于唯一索引） | 弱（可能产生重复，需 SERIALIZABLE 或唯一索引） |
| 简单 upsert | 推荐 | 杀鸡用牛刀 |
| ETL 批量同步 | 不灵活 | 推荐 |
| 性能（小批量 upsert） | 略快 | 略慢 |
| RETURNING | 直接支持 | PG 17+ 支持 |

经验：
- **行级 upsert** → ON CONFLICT
- **批量合并** → MERGE
- **跨平台兼容** → MERGE（标准）

### 4.6 RETURNING 子句

PG 在 INSERT/UPDATE/DELETE/MERGE 都支持 RETURNING，返回受影响的行：

```sql
INSERT INTO orders (user_id, amount) VALUES (1, 100) RETURNING id, created_at;

UPDATE orders SET status = 'paid' WHERE id = 1
RETURNING id, status, updated_at;

DELETE FROM cart WHERE user_id = 1 RETURNING jsonb_agg(item_id) AS removed;

WITH deleted AS (
  DELETE FROM cart WHERE user_id = 1 RETURNING item_id
)
INSERT INTO purchases (user_id, item_id) SELECT 1, item_id FROM deleted;
```

最后一个是 PG 独门——**WITH 子句中可以包含 DML**，把 DELETE 的结果直接喂给 INSERT（一个 SQL 完成事务级数据移动）。

---

## 第五章：GROUPING SETS、ROLLUP、CUBE

### 5.1 一句话

- `GROUPING SETS ((a,b), (a), (b), ())`：手动列出每个组合
- `ROLLUP (a, b, c)`：层级汇总 = `GROUPING SETS ((a,b,c), (a,b), (a), ())`
- `CUBE (a, b)`：所有组合 = `GROUPING SETS ((a,b), (a), (b), ())`

### 5.2 示例：销售报表

```sql
-- 按 region、product 汇总，再 region 小计、再总计
SELECT region, product, SUM(amount) AS total
FROM sales
GROUP BY ROLLUP (region, product);

-- 结果：
-- region | product | total
-- 华南   | A       | 1000
-- 华南   | B       |  500
-- 华南   | NULL    | 1500   <- region 小计
-- 华北   | A       |  800
-- 华北   | NULL    |  800
-- NULL   | NULL    | 2300   <- 总计
```

### 5.3 GROUPING() 函数——区分 NULL 是真 NULL 还是汇总

```sql
SELECT
  COALESCE(region, '总计') AS region,
  COALESCE(product, '小计') AS product,
  SUM(amount) AS total,
  GROUPING(region) AS g_region,
  GROUPING(product) AS g_product
FROM sales
GROUP BY ROLLUP (region, product);
-- GROUPING(col) = 0 表示该列参与分组（真值），1 表示该列是汇总（NULL 是 ROLLUP 产生的）
```

### 5.4 CUBE

```sql
SELECT region, product, SUM(amount)
FROM sales GROUP BY CUBE (region, product);
-- 4 种组合：(region,product), (region), (product), ()
```

适用：多维 OLAP 报表（区域 × 渠道 × 时间 …）

### 5.5 GROUPING SETS 通用版

```sql
SELECT region, product, channel, SUM(amount)
FROM sales
GROUP BY GROUPING SETS (
  (region, product),
  (region, channel),
  (product),
  ()
);
```

---

## 第六章：DISTINCT ON（PG 独门）

### 6.1 用法

```sql
SELECT DISTINCT ON (user_id) user_id, created_at, amount
FROM orders
ORDER BY user_id, created_at DESC;
-- 每个 user_id 取 ORDER BY 后的第一行
```

等价于窗口函数 `ROW_NUMBER() = 1`，但**更快、更简洁**：

```sql
-- 等价但啰嗦
SELECT user_id, created_at, amount FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) rn
  FROM orders
) t WHERE rn = 1;
```

### 6.2 关键要求

`DISTINCT ON (cols)` 的 cols 必须是 `ORDER BY` 的**前缀**。否则结果不确定（PG 不会报错但行为未定义）。

### 6.3 性能

如果有索引 `(user_id, created_at DESC)`，PG 可以直接 **Skip Scan**（跳跃扫描），秒返。

### 6.4 与 GROUP BY 对比

```sql
-- GROUP BY：丢失非聚合列
SELECT user_id, MAX(created_at) FROM orders GROUP BY user_id;
-- 想知道那行的 amount？还要 JOIN 回去

-- DISTINCT ON：直接拿到整行
SELECT DISTINCT ON (user_id) * FROM orders ORDER BY user_id, created_at DESC;
```

---

## 第七章：TABLESAMPLE 采样

### 7.1 两种采样方法

```sql
SELECT * FROM big_table TABLESAMPLE BERNOULLI (1);   -- 每行独立 1% 概率
SELECT * FROM big_table TABLESAMPLE SYSTEM (1);      -- 每个数据块 1% 概率（快得多）
```

| 方法 | 速度 | 均匀度 |
|---|---|---|
| BERNOULLI | 慢（全表扫） | 行级均匀 |
| SYSTEM | 快（块级） | 同块行高度相关 |

### 7.2 可复现采样（种子）

```sql
SELECT * FROM big_table TABLESAMPLE SYSTEM (1) REPEATABLE (42);
-- REPEATABLE 给定种子，多次执行结果相同
```

### 7.3 自定义采样：tsm_system_rows

```sql
CREATE EXTENSION tsm_system_rows;
SELECT * FROM big_table TABLESAMPLE SYSTEM_ROWS (1000);  -- 抽 1000 行
```

### 7.4 应用场景

- 大表上做粗略统计（avg、count distinct 近似）
- 给 ML 模型采训练集
- 慢查询调优时拿"代表性子集"
- A/B 实验抽样

---

## 第八章：综合实战案例

### 8.1 留存分析

```sql
WITH first_login AS (
  SELECT user_id, MIN(login_day) AS f_day FROM user_login GROUP BY user_id
),
cohort AS (
  SELECT user_id, date_trunc('week', f_day)::date AS cohort_week FROM first_login
),
activity AS (
  SELECT l.user_id, c.cohort_week,
         (date_trunc('week', l.login_day)::date - c.cohort_week) / 7 AS week_offset
  FROM user_login l JOIN cohort c USING (user_id)
)
SELECT cohort_week,
       COUNT(DISTINCT user_id) FILTER (WHERE week_offset = 0) AS w0,
       COUNT(DISTINCT user_id) FILTER (WHERE week_offset = 1) AS w1,
       COUNT(DISTINCT user_id) FILTER (WHERE week_offset = 2) AS w2,
       COUNT(DISTINCT user_id) FILTER (WHERE week_offset = 4) AS w4
FROM activity GROUP BY cohort_week ORDER BY cohort_week;
```

### 8.2 漏斗分析

```sql
WITH funnel AS (
  SELECT user_id,
         MAX(CASE WHEN event = 'view'    THEN 1 ELSE 0 END) AS s1,
         MAX(CASE WHEN event = 'add_cart' THEN 1 ELSE 0 END) AS s2,
         MAX(CASE WHEN event = 'pay'     THEN 1 ELSE 0 END) AS s3
  FROM events GROUP BY user_id
)
SELECT SUM(s1) AS view, SUM(s2) AS add_cart, SUM(s3) AS pay,
       SUM(s2) * 100.0 / NULLIF(SUM(s1), 0) AS view2cart,
       SUM(s3) * 100.0 / NULLIF(SUM(s2), 0) AS cart2pay
FROM funnel;
```

### 8.3 累计 RFM

```sql
WITH rfm AS (
  SELECT user_id,
         (CURRENT_DATE - MAX(order_day))::int AS recency,
         COUNT(*) AS frequency,
         SUM(amount) AS monetary
  FROM orders GROUP BY user_id
)
SELECT user_id, recency, frequency, monetary,
       NTILE(5) OVER (ORDER BY recency)   AS r,
       NTILE(5) OVER (ORDER BY frequency DESC) AS f,
       NTILE(5) OVER (ORDER BY monetary DESC)  AS m
FROM rfm;
```

### 8.4 时序断点检测（Gap & Island）

```sql
-- 找出每个设备连续 ON 的时段
WITH labeled AS (
  SELECT device_id, ts, status,
         ROW_NUMBER() OVER (PARTITION BY device_id ORDER BY ts) -
         ROW_NUMBER() OVER (PARTITION BY device_id, status ORDER BY ts) AS grp
  FROM device_status
)
SELECT device_id, MIN(ts) AS start_ts, MAX(ts) AS end_ts
FROM labeled WHERE status = 'ON'
GROUP BY device_id, grp;
```

### 8.5 复合榜单（CTE + LATERAL + 窗口）

```sql
WITH active_users AS (
  SELECT id FROM users WHERE last_login > now() - interval '30 days'
)
SELECT u.id, u.name, t.rank, t.amount, t.product
FROM active_users a
JOIN users u ON u.id = a.id
JOIN LATERAL (
  SELECT id, amount, product,
         RANK() OVER (ORDER BY amount DESC) AS rank
  FROM orders WHERE user_id = u.id AND created_at > now() - interval '7 days'
  ORDER BY amount DESC LIMIT 3
) t ON true;
```

---

## 生产实践

### 实践 1：窗口函数与索引

`PARTITION BY a ORDER BY b` 的窗口能用 `(a, b)` 组合索引 **避免 Sort**。EXPLAIN 看到 `WindowAgg` 上方没有 `Sort` 节点即为命中。

### 实践 2：CTE 性能 trap

PG 12 之前的代码迁移到 12+ 时，**CTE 行为可能改变**。如果发现某查询变慢/变快，先看 CTE 是被 inline 还是 materialize 了：

```sql
EXPLAIN (ANALYZE, VERBOSE) WITH t AS (SELECT ...) SELECT ... ;
```

需要保持旧行为时显式加 `MATERIALIZED`。

### 实践 3：MERGE 的并发安全

`MERGE` 在 RC 隔离下**不保证唯一性**——两个并发 MERGE 都判定"NOT MATCHED"然后都 INSERT 会冲突。两种方案：

- 提升到 SERIALIZABLE（性能代价大）
- 表上加 UNIQUE 索引兜底，INSERT 失败时显式重试

### 实践 4：ON CONFLICT 的"幽灵更新"

`DO UPDATE` 在并发场景下可能"覆盖一个尚未 commit 的版本"（PG 通过 update chain 解决，但行为不直观）。

最佳实践：`WHERE` 子句保护：

```sql
... ON CONFLICT (id) DO UPDATE
SET version = EXCLUDED.version
WHERE target.version < EXCLUDED.version;
```

### 实践 5：递归 CTE 的内存

递归 CTE 会把"工作集 + 结果集"放在内存（`work_mem` 不够则用 temp file）。深图遍历前先估算工作集大小，必要时切片处理。

### 实践 6：报表 SQL 的可维护性

复杂报表用多个命名 CTE 把每个逻辑步骤拆成块；用 `MATERIALIZED` 防止 planner 把它们合并产生天书计划。

### 实践 7：LATERAL 替代 N+1

应用层最常见的 N+1：

```go
for _, u := range users { db.Query("SELECT ... FROM orders WHERE user_id = ?", u.ID) }
```

一条 LATERAL 解决：

```sql
SELECT u.*, o.*
FROM users u
JOIN LATERAL (SELECT ... FROM orders WHERE user_id = u.id ORDER BY ... LIMIT 5) o ON true;
```

### 实践 8：DISTINCT ON 取代窗口函数

PG 独有的 `DISTINCT ON` 比通用 `ROW_NUMBER() = 1` 快、可读、可用 Skip Scan。Postgres 项目中优先用 DISTINCT ON。

---

## 陷阱清单

| # | 陷阱 | 后果 | 解决 |
|---|---|---|---|
| 1 | `LAST_VALUE` 默认 frame 是 RANGE→CURRENT，永远 = 当前行 | 结果错 | 显式 `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| 2 | WHERE 直接用窗口函数 | 语法错 | 包子查询 |
| 3 | RANGE frame 对重复值"绑定" | 累加跳跃 | 改 ROWS 或用 GROUPS（PG 11+） |
| 4 | DISTINCT ON 列不是 ORDER BY 前缀 | 结果不确定 | 修正 ORDER BY |
| 5 | PG 12 后 CTE 自动 inline 改变执行计划 | 老查询变慢 | 显式 MATERIALIZED |
| 6 | 递归 CTE 无环检测炸内存 | OOM | UNION 去重 / WHERE 深度限制 / CYCLE 子句 |
| 7 | MERGE 在 RC 下并发不保证唯一 | 重复插入 | 唯一索引 + 重试 或 SERIALIZABLE |
| 8 | ON CONFLICT 未利用唯一索引 | 语法错 | 必须有匹配的 UNIQUE/PK |
| 9 | GROUPING SETS 中 NULL 与汇总 NULL 混淆 | 报表错乱 | 用 GROUPING() 函数区分 |
| 10 | ROLLUP/CUBE 列过多组合爆炸 | 慢/OOM | 用 GROUPING SETS 手选组合 |
| 11 | LATERAL 右侧子查询无索引 | 全表扫 × N | 给关联列建索引 |
| 12 | RETURNING 与触发器交互 | 触发器修改后的值 vs 原值 | 阅读触发器行为 |
| 13 | WITH 内 DML 副作用顺序未定义 | 数据不确定 | 一个 WITH 不要混 INSERT/UPDATE/DELETE 同一表 |
| 14 | NTILE 在重复值上分桶不均 | 桶大小相差 1 | 接受或改用 PERCENT_RANK |
| 15 | 窗口函数 Sort 内存不足 | 落盘很慢 | 加大 work_mem 或建组合索引 |

---

## 2026 现状

| 主题 | 2026 状态 |
|---|---|
| 窗口函数 | 标准齐全；GROUPS frame 成熟 |
| LATERAL | 优化器对其支持稳定，是 N+1 终极解 |
| 递归 CTE | CYCLE 子句普及（PG 14+）；图分析仍推荐专用图库 |
| CTE inline | PG 12+ 默认 inline 已成主流共识 |
| MERGE | PG 15 引入，PG 17 加 RETURNING，PG 18 持续优化 |
| ON CONFLICT | 仍是单行 upsert 首选 |
| GROUPING SETS | 简单 OLAP 够用；复杂仍走专用 OLAP 引擎 |
| DISTINCT ON | PG 独门稳定特性 |
| TABLESAMPLE | 不常用但偶有亮点（ML 抽样） |
| 反向预期 | QUALIFY 仍未进入 PG，需子查询 |

---

## 练习题

> 答案见 [QUIZ.md](./QUIZ.md)。

1. **frame 差异**：表 `t(v)` 有值 `1,1,2,3,3,3,4`，写出三个查询：
   ```sql
   SELECT v, SUM(v) OVER (ORDER BY v) FROM t;                    -- RANGE
   SELECT v, SUM(v) OVER (ORDER BY v ROWS UNBOUNDED PRECEDING) FROM t;
   SELECT v, SUM(v) OVER (ORDER BY v GROUPS UNBOUNDED PRECEDING) FROM t;  -- PG 11+
   ```
   预测每个的输出。

2. **环比同比**：写一个 SQL 同时计算"对比昨天 / 上周同日 / 去年同日"三个增长率。

3. **每组首单**：用 `DISTINCT ON` 写"每个用户的首单"。再用窗口函数写一遍，比较 EXPLAIN。

4. **LATERAL 改 N+1**：你有一段应用代码遍历 1000 个用户每个取最近 3 条订单，用一条 LATERAL JOIN 重写。

5. **CTE materialize**：以下 CTE 在 PG 12+ 会被 inline 还是 materialize？
   ```sql
   WITH t AS (SELECT * FROM big WHERE flag)
   SELECT * FROM t WHERE name = 'x';
   ```
   如果想强制 planner 把 `name='x'` 下推进 CTE 应该怎么改？

6. **递归 CTE**：表 `friends(a, b)` 表示无向好友关系，写一个递归 CTE 找出"用户 1 的 N 度好友列表"，N 作为参数。注意防环。

7. **MERGE**：写一个 MERGE 语句完成"库存对账"——source 是当日实际盘点，target 是系统库存：
   - 当 MATCHED 且数量不同 → UPDATE
   - 当 MATCHED 且 source 数量 = 0 → DELETE
   - 当 NOT MATCHED → INSERT
   并用 PG 17 的 RETURNING 输出每行的操作类型。

8. **ROLLUP**：销售表 `sales(year, quarter, region, amount)`，写一个 ROLLUP 同时输出：年度合计、年度+季度合计、年度+季度+region 合计、总合计。

---

## 延伸阅读

- 官方文档：[Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- 官方文档：[WITH Queries (CTE)](https://www.postgresql.org/docs/current/queries-with.html)
- 官方文档：[MERGE](https://www.postgresql.org/docs/current/sql-merge.html)
- 官方文档：[GROUP BY clause with ROLLUP/CUBE/GROUPING SETS](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPING-SETS)
- 博客：[Crunchy Data — Recursive CTEs Explained](https://www.crunchydata.com/blog/recursive-ctes-explained)
- 博客：[pganalyze — Lateral joins and PostgreSQL](https://pganalyze.com/blog/lateral-joins-postgresql)
- 博客：[Depesz — Waiting for PostgreSQL 17 — MERGE...RETURNING](https://www.depesz.com/2024/04/15/waiting-for-postgresql-17-add-returning-support-to-merge/)
- 论文：[Modern Window Functions in SQL](https://15721.courses.cs.cmu.edu/spring2017/papers/12-windowfunctions/leis-vldb2015.pdf)
- 工具：[explain.dalibo.com](https://explain.dalibo.com/) — 可视化解析 EXPLAIN
- 关联章节：[P10 EXPLAIN](./10-精通-EXPLAIN-与执行计划.md) / [P11 查询调优](./11-精通-查询调优实战.md) / [P12 JSONB 与全文检索](./12-精通-JSONB-与全文检索.md)
