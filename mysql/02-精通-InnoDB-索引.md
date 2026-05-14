# 精通 InnoDB B+ 树索引：页结构、聚簇索引、覆盖索引、ICP/MRR

> 关联章节：[M01 整体架构](./03-精通-MySQL-整体架构.md)、[M03 事务与 MVCC](./05-精通-InnoDB-事务-MVCC.md)、[M06 优化器](./08-精通查询优化与-EXPLAIN.md)

---

## 引言：为什么所有的 SQL 性能问题最后都回到索引

如果让你只学一章关于 MySQL 的知识，应该是这一章。**生产环境 80% 的 SQL 性能问题都能用"索引选错"或"索引设计错"来解释**。剩下 20% 中的一半也跟索引有关——只是表现为锁、回表、临时表。

但工程师对索引的理解经常停在"加 index on (col)"这一层。深入一点的问题立刻露馅：

- 联合索引 `(a, b, c)` 上的 `WHERE b=1 AND c=2` 能用吗？为什么？
- `WHERE name LIKE '%foo'` 永远走不了索引吗？
- 二级索引等于扫描两次 B+ 树吗？什么场景能省一次？
- `OFFSET 100000 LIMIT 10` 为什么慢？怎么改？
- 100 GB 表加索引会锁表多久？
- `IN (1,2,3,...,1000)` 还能走索引吗？

这一章把 InnoDB 索引从最底层（页内字节布局）讲到最上层（设计原则），中间贯穿优化器视角与生产经验。读完之后你应该能：

- 画出 InnoDB 一个 16KB 页的内部结构
- 解释为什么聚簇索引主键最好"自增整数"
- 区分 ICP（Index Condition Pushdown）和 MRR（Multi-Range Read）
- 给一张 10 亿行表设计联合索引
- 看 EXPLAIN 时知道索引用了多少前缀（`key_len`）

---

## 第一章：B+ 树为什么是数据库的"自然之选"

### 1.1 候选数据结构对比

| 结构 | 等值查询 | 范围查询 | 排序 | 磁盘友好 |
|---|---|---|---|---|
| 哈希表 | O(1) | ✗ 不支持 | ✗ | ✗ 散列分布，磁盘随机 |
| 跳表 | O(log N) | ✓ | ✓ | ✗ 链表节点散落 |
| 平衡二叉树 | O(log N) | ✓ | ✓ | ✗ 树高大，I/O 多 |
| **B+ 树** | O(log N) | ✓ | ✓ | ✓ **多叉低高度** |
| LSM-Tree | ~O(log N) | ✓ | ✓ | ✓ **写优化**（RocksDB） |

InnoDB 选 B+ 树的核心理由：

1. **多叉低高度**：每个节点 16KB，可存几百到上千个 key——3 层 B+ 树就能管 2000 万行
2. **范围查询友好**：叶子节点之间双向链表
3. **顺序写入对磁盘友好**（聚簇索引时）

### 1.2 为什么是 B+ 树而不是 B 树

B 树每个节点存 (key + data)；B+ 树**只有叶子节点存 data**，内部节点只存 key + 子树指针。

带来的好处：

- 内部节点更小 → 一页能放更多 key → 树更扁
- 叶子节点用链表串起来 → 范围扫高效
- 全表扫描只需扫叶子层

### 1.3 B+ 树的高度估算

InnoDB 默认页 16KB。假设：

- 主键 8 字节（bigint）
- 内部节点指针 6 字节
- 一个内部节点 ≈ 16 KB / (8 + 6) ≈ 1170 个 key
- 每个叶子页存 1KB/行 × 16 行 = 16 行

3 层 B+ 树容纳：1170 × 1170 × 16 ≈ **2200 万行**
4 层 B+ 树：1170^3 × 16 ≈ **256 亿行**

**InnoDB 中常见的 B+ 树高度只有 2-4 层**——这是为什么索引查询通常在毫秒级。

---

## 第二章：InnoDB 页（Page）的内部布局

### 2.1 页是 InnoDB 的最小 I/O 单位

```
innodb_page_size = 16384    # 默认 16 KB
```

所有读写都以页为单位，即使你只查一行也读整个页。这是为什么"宽行"（一行 1KB 以上）会显著伤性能——一页能放的行数变少。

### 2.2 一页 16KB 的结构

```
+------------------+ 0
| File Header      | 38 字节   页号、上下页指针、LSN、checksum
+------------------+ 38
| Page Header      | 56 字节   页内记录数、最后一条记录、Slot 数
+------------------+ 94
| Infimum + Supremum| 26 字节  虚拟最小/最大记录（边界）
+------------------+ 120
|                  |
|  User Records    | ↓ 向下生长
|  (实际数据)       |
|                  |
| ... 空闲空间 ...  |
|                  |
|  Page Directory  | ↑ 向上生长（slot 数组，二分查找用）
+------------------+ 16384 - 8
| File Trailer     | 8 字节   checksum + LSN，写入完整性验证
+------------------+ 16384
```

关键点：

- **Page Directory** 是稀疏索引（每 4-8 条记录一个 slot），让页内记录用二分查找而非顺序扫
- **Infimum / Supremum** 是两个虚拟记录，所有真实记录夹在中间，链表的头尾哨兵
- 用户记录之间通过 `next_record` 字段串成单向链表（按主键升序）

### 2.3 单条记录的格式（COMPACT row format）

```
+-----------+--------+--------+----------+-----+----+----+----+
| 变长长度列表 | NULL 位图 | 记录头(5B) | 隐藏列(13B) | 列1 | 列2 | ... |
+-----------+--------+--------+----------+-----+----+----+----+
                     <- 信息区 ->         <- 数据区 ->
```

3 个隐藏列：

- `DB_ROW_ID`（6B）：如果没显式主键时自动生成
- `DB_TRX_ID`（6B）：最后修改该行的事务 ID（MVCC 用）
- `DB_ROLL_PTR`（7B）：指向 undo log（MVCC 历史版本）

记录头 5 字节里有：

- `delete_mask`（1 bit）：是否被 delete 标记
- `record_type`（3 bit）：0=普通行 / 1=B+ 节点指针 / 2=infimum / 3=supremum
- `next_record`（16 bit）：下一行偏移（链表）

### 2.4 列存储顺序的优化

InnoDB 把**变长列**（VARCHAR / TEXT / BLOB）放到记录尾部，前面是**变长长度数组**。原因：

- 定长字段固定偏移 → 直接寻址
- 变长字段长度集中放，遍历记录头时不打扰数据访问

如果一行的 VARCHAR 总长 > 半页（约 8KB），InnoDB 触发 **off-page 存储**：把超长字段放外部页，行内只留 20 字节指针。这就是 BLOB / TEXT 的实现方式。

---

## 第三章：聚簇索引 vs 二级索引

### 3.1 聚簇索引（Clustered Index）

**InnoDB 表的数据本身就是 B+ 树**——叶子页存的是**完整行数据**，按主键顺序组织。这棵 B+ 树就是聚簇索引（也叫主键索引）。

```
                  [主键: 1, 50, 100]
                 /         |        \
         [1,5,10]     [50,55,60]   [100,105,110]
         (完整行)      (完整行)      (完整行)
            |             |             |
         双向链表 -- 双向链表 -- 双向链表
```

含义：

- **每张 InnoDB 表都有且仅有一个聚簇索引**
- 没显式主键时，InnoDB 用第一个 NOT NULL UNIQUE 索引；都没有则用隐藏的 `DB_ROW_ID`
- 数据**物理上**按主键顺序存储 → 主键范围查询最快

### 3.2 二级索引（Secondary Index）

`CREATE INDEX idx_name ON users(name)` 建的是二级索引。它也是 B+ 树，但叶子页存的是 `(索引列, 主键)`，**不是完整行**。

```
              二级索引 (name)
                  [b, m, t]
                /      |     \
         [a,b]      [m,n,o]    [t,u,z]
         {主键 1,2}  {主键 6,7,8} {主键 12,13,15}
```

含义：

- 通过二级索引找到主键，然后**回到聚簇索引**查完整行——这一步叫**回表**（lookup）
- 一张表可以有多个二级索引（一般不超过 5-6 个）

### 3.3 回表的代价

```sql
-- users(id PK, name, age, city)
-- 索引：idx_name(name)

SELECT * FROM users WHERE name = 'Alice';
-- 1. 在 idx_name 上找到 (name='Alice') → 拿到主键 id=42
-- 2. 在聚簇索引上用 id=42 找到完整行
-- 共 2 次 B+ 树查找
```

如果 `name='Alice'` 匹配 1000 行，就要回表 1000 次——每次都是新的 B+ 树查找。这是为什么"等值少结果"的查询用二级索引爽，"匹配很多"的反而不如全表扫。

### 3.4 主键设计原则

**用自增整数（BIGINT AUTO_INCREMENT）当主键**——经典推荐，原因：

1. **顺序写入**：B+ 树按主键插入。自增主键总是追加到右边，不分裂中间页
2. **空间紧凑**：8 字节 vs UUID 36 字节（字符串）
3. **二级索引短**：所有二级索引都包含主键 → 主键越短，二级索引越小
4. **范围查询友好**：物理顺序 = 逻辑顺序

**反面教材**：用 UUID 做主键

- 36 字节 → 二级索引膨胀 4-5 倍
- 完全随机 → 每次插入触发**页分裂**
- 写入吞吐降到 1/5 - 1/10

如果业务要 UUID（如分布式生成 ID），常见折中：用 BIGINT 做主键 + UUID 单独建唯一二级索引。

---

## 第四章：联合索引与最左前缀

### 4.1 联合索引的存储顺序

`CREATE INDEX idx ON t(a, b, c)` 按 `(a, b, c)` 三元组排序。B+ 树里 key 是 `(a, b, c)` 的组合。

可以把 B+ 树想成"按 a 排序；a 相同时按 b 排序；b 相同时按 c 排序"。

### 4.2 最左前缀匹配规则

只有从最左列开始连续匹配的查询能完整用上索引：

| 查询条件 | 用了索引几列 |
|---|---|
| `WHERE a=1` | 1（a） |
| `WHERE a=1 AND b=2` | 2（a, b） |
| `WHERE a=1 AND b=2 AND c=3` | 3（a, b, c） |
| `WHERE a=1 AND c=3` | 1（仅 a；c 在内存里过滤） |
| `WHERE b=2 AND c=3` | 0（不能用） |
| `WHERE a=1 AND b>2 AND c=3` | 2（a 等值 + b 范围；c 在内存过滤）|

第 4 行是常见误区——**等值列必须连续**才能继续用索引。第 6 行是"范围列后断"——一旦遇到范围（`>`, `<`, `BETWEEN`, `LIKE 'foo%'`），后续列就不能再用索引排序。

### 4.3 列顺序设计原则

设计联合索引时，**等值条件 → 排序列 → 范围条件**：

```sql
-- 业务：查某用户某状态下按时间倒序的订单
SELECT * FROM orders
WHERE user_id = ? AND status = 'paid'
ORDER BY created_at DESC LIMIT 20;

-- 推荐索引：(user_id, status, created_at)
--   user_id 等值 → status 等值 → created_at 范围排序
```

如果业务还要支持 `WHERE user_id=? AND created_at>?`，那索引可改成 `(user_id, created_at, status)`，让范围更有效；但 status 等值就不能进索引了。**没有完美索引——按业务最高频查询设计**。

### 4.4 EXPLAIN 看 key_len 验证用了几列

```sql
EXPLAIN SELECT * FROM t WHERE a=1 AND b=2;
-- key: idx_abc
-- key_len: 8         ← 用了 a (4B) + b (4B) = 8B
```

key_len 计算（int = 4 字节，varchar 视字符集与长度）：

- `int NOT NULL` → 4
- `int NULL` → 4 + 1
- `varchar(20)` utf8mb4 → 20 × 4 + 2 = 82

key_len 越大代表用了越多列。

---

## 第五章：覆盖索引

### 5.1 覆盖索引的定义

**查询所需的所有列都包含在索引内 → 不需要回表**。

```sql
-- 表：users(id PK, name, age, city)
-- 索引：idx_name(name)

EXPLAIN SELECT id, name FROM users WHERE name = 'Alice';
-- Extra: Using index   ← 覆盖索引！
```

为什么 OK？因为二级索引的叶子页存了 `(name, id)`，查询要的两列都在里面。

```sql
EXPLAIN SELECT name, age FROM users WHERE name = 'Alice';
-- Extra: 无 "Using index"  ← 需要回表查 age
```

### 5.2 设计覆盖索引

把高频查询的"select 列"加入索引尾部（在范围列之后）：

```sql
-- 高频查询
SELECT name, age FROM users WHERE city = 'Beijing';

-- 普通索引：idx_city(city) → 要回表查 name, age
-- 覆盖索引：idx_city_cover(city, name, age) → 不回表
```

代价：索引变大、写入更慢。一般加 1-2 个常用列即可，不要把整张表的列都塞进去。

### 5.3 INCLUDE 索引（PostgreSQL 有，MySQL 没有）

PG 有 `CREATE INDEX ... INCLUDE (col1, col2)` 显式声明"加进叶子但不参与排序"。MySQL 无此语法，要把列加到 `(...)` 里参与排序——索引会更大。

---

## 第六章：Index Condition Pushdown (ICP)

### 6.1 5.6 之前的痛点

```sql
-- idx_age_name(age, name)
SELECT * FROM users WHERE age = 30 AND name LIKE '%Bob%';

-- 5.6 之前：
-- 1. 在索引上找 age = 30 的所有记录（如 1000 个）
-- 2. 全部回表取 name
-- 3. Server 层对 1000 个 name 做 LIKE 过滤
```

`LIKE '%Bob%'` 不能走索引（`%` 在前），但 `name` 已经在二级索引叶子页里了——完全可以在引擎层过滤再决定回不回表。

### 6.2 ICP 把过滤下推到引擎层

5.6+ 启用 ICP：

```
1. 在索引上找 age = 30
2. 取叶子页里的 (name, id)，先在引擎层过滤 name LIKE '%Bob%'
3. 通过的才回表取完整行
```

效果：减少大量回表 I/O。

EXPLAIN 标志：

```
Extra: Using index condition
```

### 6.3 关闭 ICP

```sql
SET optimizer_switch = 'index_condition_pushdown=off';
```

调试用。生产保持默认 on。

---

## 第七章：Multi-Range Read (MRR)

### 7.1 范围查询回表的随机 I/O

```sql
-- idx_age(age)
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
```

二级索引按 age 排序，回表时拿到的主键是**乱序**的（age 与 id 无关）→ 回表对聚簇索引是随机 I/O，磁盘性能差。

### 7.2 MRR 的优化

5.6+：

1. 先扫二级索引取所有目标主键
2. **对主键排序**
3. 按排序后顺序回表 → 顺序 I/O

EXPLAIN 标志：

```
Extra: Using MRR
```

### 7.3 SSD 时代 MRR 的意义减弱

机械盘上 MRR 收益巨大；SSD 时代随机 I/O 代价小，MRR 收益变小，默认有时关闭：

```
optimizer_switch = 'mrr=on,mrr_cost_based=on'
```

`mrr_cost_based=on` 让优化器按成本决定用不用 MRR。

---

## 第八章：索引下推与排序、分组

### 8.1 索引天然有序——可以省 sort

```sql
-- idx_created(created_at)
SELECT * FROM orders ORDER BY created_at LIMIT 10;
-- Extra: Using index 或 无 filesort
```

二级索引叶子双向链表 → 直接顺序读 10 个就出来。

如果 `ORDER BY created_at DESC LIMIT 10`，8.0 之前要"反向扫描"，性能差；8.0+ 支持降序索引：

```sql
CREATE INDEX idx_created_desc ON orders(created_at DESC);
```

### 8.2 索引覆盖 GROUP BY

```sql
-- idx_status(status)
SELECT status, COUNT(*) FROM orders GROUP BY status;
-- Extra: Using index for group-by   ← Loose index scan
```

InnoDB 的 **loose index scan** 直接跳到每个分组的第一条，不读组内其他行。

### 8.3 filesort

`Extra: Using filesort` 表示要在内存或磁盘做排序。处理：

- 给排序列加索引
- 增大 `sort_buffer_size`（线程级，谨慎）
- 改 SQL 减少排序需求

---

## 第九章：前缀索引与函数索引

### 9.1 长字符串的前缀索引

```sql
-- email VARCHAR(100)
CREATE INDEX idx_email ON users(email(10));   -- 只索引前 10 字符
```

- 节省空间
- 牺牲选择性（前缀重复多的 email 区分度低）

选择性度量：

```sql
SELECT COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) FROM users;
-- 接近 1 才合适
```

前缀索引**不能用于覆盖索引、ORDER BY、GROUP BY**（因为只存了前缀，不知道完整值的顺序）。

### 9.2 函数索引（8.0+）

```sql
-- 业务：按月份统计
SELECT * FROM orders WHERE MONTH(created_at) = 5;
-- 5.7：MONTH(...) 不能走索引

-- 8.0+ 函数索引：
CREATE INDEX idx_month ON orders((MONTH(created_at)));
-- 现在可以走索引
```

或更通用的虚拟列 + 索引：

```sql
ALTER TABLE orders ADD COLUMN month INT GENERATED ALWAYS AS (MONTH(created_at)) STORED;
CREATE INDEX idx_month ON orders(month);
```

### 9.3 表达式索引下的查询匹配

注意：函数索引只对**完全相同表达式**生效。

```sql
CREATE INDEX idx ON t((LOWER(name)));

WHERE LOWER(name) = 'alice';   -- 用索引
WHERE LOWER(name) LIKE 'a%';   -- 用索引（范围）
WHERE name = 'Alice';          -- 不用！表达式不匹配
```

---

## 第十章：哈希索引与全文索引

### 10.1 InnoDB 的自适应哈希索引（AHI）

InnoDB 内部对**热点 B+ 树节点**自动建哈希索引，让等值查询不必每次走 B+ 树。

```
SHOW ENGINE INNODB STATUS\G
-- Hash table size N, node heap has K buffer(s)
-- 0.00 hash searches/s, 0.00 non-hash searches/s
```

AHI 在内存里维护，查询透明。**但对锁竞争重的工作负载有时是负担**——监控 hash searches 比例，必要时关闭：

```sql
SET GLOBAL innodb_adaptive_hash_index = OFF;
```

### 10.2 全文索引（FULLTEXT）

```sql
CREATE FULLTEXT INDEX ft_idx ON articles(title, body);
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('mysql innodb');
```

InnoDB 5.6+ 支持。但功能远不如 Elasticsearch / Meilisearch / Manticore。**生产几乎不用 MySQL 全文检索做严肃搜索**——分词、相关性、排序、模糊匹配都弱。

---

## 第十一章：索引失效的常见场景

### 11.1 隐式类型转换

```sql
-- phone VARCHAR(20)
WHERE phone = 13800138000   -- 数字！
```

字符串列 = 数字 → 触发函数转换 `CAST(phone AS BIGINT) = 13800138000` → 索引失效。

修复：用字符串 `WHERE phone = '13800138000'`。

### 11.2 函数包裹列

```sql
WHERE DATE(created_at) = '2026-05-13'   -- 索引失效
```

改成范围查询：

```sql
WHERE created_at >= '2026-05-13 00:00:00' AND created_at < '2026-05-14';
```

### 11.3 OR 跨多列

```sql
-- idx_a(a), idx_b(b)
WHERE a = 1 OR b = 2
```

5.x 不能合并两个索引；5.6+ 引入 **index merge**（union / intersection / sort_union）能合并，但仍不如改 SQL：

```sql
SELECT * FROM t WHERE a = 1
UNION
SELECT * FROM t WHERE b = 2;
```

### 11.4 LIKE '%foo%'

`%` 在前 → 不能走索引（B+ 树需要从前缀开始匹配）。如果一定要做模糊搜索，用全文索引或外部搜索引擎。

`LIKE 'foo%'` 可以走索引（前缀匹配 = 范围扫描）。

### 11.5 NOT IN / != 

```sql
WHERE status != 'paid'
```

不等查询通常不能走索引（要扫除 paid 之外的所有），优化器一般选全表扫。如果"等于的值"占比小，反着写：

```sql
WHERE status IN ('pending', 'cancelled', 'refunded')
```

### 11.6 排序列与 WHERE 列方向冲突

```sql
-- idx(a ASC, b ASC)
WHERE a > 5 ORDER BY a ASC, b DESC
```

a 范围 + b 反向 → 不能用索引排序（要 filesort）。

8.0+ 可以建混合方向：

```sql
CREATE INDEX idx ON t(a ASC, b DESC);
```

---

## 第十二章：分页深翻问题

### 12.1 LIMIT 100000, 10 为什么慢

```sql
SELECT * FROM orders ORDER BY id LIMIT 100000, 10;
```

MySQL 实际**扫描 100010 行**，丢弃前 100000，返回最后 10。深翻越深越慢。

### 12.2 优化：游标分页（keyset pagination）

```sql
-- 翻第 N 页时记下当前页最后一行的 id
SELECT * FROM orders WHERE id > <last_id> ORDER BY id LIMIT 10;
```

每页都是 O(log N) 索引查找 + 顺序读 10 行，与页码无关。

### 12.3 优化：延迟关联

如果不能改业务为游标分页：

```sql
SELECT o.* FROM orders o
JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 10
) AS t ON o.id = t.id;
```

子查询只走索引（不回表），主查询只回表 10 次，比直接 `SELECT *` 快很多。

---

## 第十三章：索引的代价

### 13.1 写入开销

每次 INSERT/UPDATE/DELETE 都要更新所有相关索引。索引越多，写入越慢。

经验值：**5-6 个索引是上限**。超过的话写入吞吐显著下降。

### 13.2 空间开销

二级索引大小 ≈ `(索引列宽度 + 主键宽度) × 行数 × 1.5`（B+ 树有冗余）。

10 亿行表 + 10 个二级索引 → 索引可能比数据还大。

### 13.3 在线 DDL 加索引的方式

```sql
ALTER TABLE t ADD INDEX idx(col), ALGORITHM=INPLACE, LOCK=NONE;
```

5.6+ 支持在线加索引，**不锁表**（仅 metadata 短锁）。但仍：

- 占用磁盘 I/O
- 大表（百 GB 级）耗时小时级
- 期间 redo log 持续累积

**生产实践**：用 pt-online-schema-change 或 gh-ost，更可控。

---

## 第十四章：索引设计实战

### 14.1 案例 1：订单表

```sql
-- 业务查询
1. SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20;
2. SELECT * FROM orders WHERE order_no = ?;
3. SELECT COUNT(*) FROM orders WHERE status = 'paid' AND created_at > ?;

-- 索引设计
PRIMARY KEY (id)
UNIQUE KEY uk_order_no (order_no)
KEY idx_user_created (user_id, created_at)
KEY idx_status_created (status, created_at)
```

### 14.2 案例 2：消息表

```sql
-- 业务：查某用户最近 50 条收件
SELECT * FROM messages WHERE to_user = ? ORDER BY id DESC LIMIT 50;

-- 索引：(to_user, id) 联合
KEY idx_to_user (to_user, id)
-- 排序按 id 降序、to_user 等值 → 直接走索引，无 filesort
```

如果还要按未读筛选：

```sql
WHERE to_user = ? AND is_read = 0 ORDER BY id DESC

-- 索引：(to_user, is_read, id)
KEY idx_to_user_unread (to_user, is_read, id)
```

### 14.3 案例 3：日志表（高写入）

写入是主要负载，索引尽量少：

```sql
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    level VARCHAR(10),
    body TEXT
) PARTITION BY RANGE (UNIX_TIMESTAMP(ts)) (...);

-- 仅一个时间索引就够，配合分区
KEY idx_ts (ts)
```

定期 DROP 老分区清理。

---

## 第十五章：诊断与排查

### 15.1 检查索引是否被使用

```sql
EXPLAIN FORMAT=TREE SELECT ... ;       -- 8.0+
EXPLAIN ANALYZE SELECT ... ;           -- 8.0.18+
```

ANALYZE 真的运行 SQL，给出实际行数 vs 估算行数对比。

### 15.2 索引选择性低的 key 没用

```sql
-- 看每个索引的选择性
SELECT
    INDEX_NAME, CARDINALITY,
    (SELECT COUNT(*) FROM users) AS total_rows,
    CARDINALITY / (SELECT COUNT(*) FROM users) AS selectivity
FROM information_schema.STATISTICS
WHERE TABLE_NAME = 'users';
```

selectivity < 0.1 的索引（如 status='paid' 占 90%）几乎无用，徒增写入开销。

### 15.3 找出未使用的索引

```sql
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema NOT IN ('mysql', 'performance_schema', 'sys');
```

实例运行一段时间后查这个表能看到哪些索引完全没被用过 → 可以删。

### 15.4 直方图（8.0+）

低选择性列的等值估算可以用直方图改进：

```sql
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;
```

适用：

- 数据分布不均（如 status 90% 是 paid）
- 没有索引的列也能有统计信息
- 不影响写入（只是统计信息）

详见 M06 优化器章。

---

## 总结

InnoDB 索引的核心：**B+ 树 + 聚簇 + 16KB 页**。本章关键点：

1. **聚簇索引**：表数据本身按主键组织成 B+ 树；二级索引叶子存"列+主键"，要回表
2. **页结构**：16KB 页 + Page Directory + 双向链表
3. **联合索引最左前缀**：等值列必须连续，遇范围则停
4. **覆盖索引**：把查询所需列全放索引里，免回表
5. **ICP / MRR**：引擎层过滤 + 主键排序回表，减少 I/O
6. **索引失效**：隐式转换、函数包裹、`%` 在前的 LIKE
7. **深翻**：游标分页 / 延迟关联
8. **写入代价**：索引越多写越慢，5-6 个为上限
9. **在线 DDL**：8.0+ 大多支持 INPLACE/INSTANT，仍建议 pt-osc / gh-ost

---

## 练习题

1. 给一张 1 亿行的 `orders(id PK, user_id, status, created_at, amount)`，业务有以下高频查询，设计最合适的索引：
   - 按用户查最近 20 单
   - 按订单号精确查
   - 按状态 + 时间范围统计金额
   - 按时间区间导出全部订单
2. 解释为什么 `WHERE a=1 AND b>2 AND c=3` 在索引 `(a,b,c)` 上只能用前两列。
3. 一张 1KB 行宽的表，10 亿行，主键 `BIGINT`，估算 B+ 树高度与单次主键查询的预期 I/O 次数。
4. 写一个 SQL 故意用上 ICP，再写一个明显需要 MRR 的，对比 EXPLAIN。
5. 用 UUID 做主键时插入吞吐变慢，给出至少 2 个数据层面的根因。
6. 一段 SQL 慢，EXPLAIN 显示 `Using filesort; Using temporary`。给出 3 个排查方向。
7. 设计一个会触发"前缀索引选择性不足"问题的场景，给出量化判断标准。
8. `SELECT id FROM users WHERE name LIKE 'foo%' LIMIT 100` 与 `SELECT * FROM users WHERE name LIKE 'foo%' LIMIT 100`，性能差几倍？为什么？
9. 业务有一个 `WHERE city='Beijing' OR phone='13...'` 的查询，索引怎么设计？
10. 100GB 表加二级索引，给出在线方案与离线方案的差别（耗时、风险、回滚）。

---

> 🔁 反馈：每个 EXPLAIN 字段都自己跑一遍才有体感
