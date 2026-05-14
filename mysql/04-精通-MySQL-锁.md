# 精通 MySQL 锁机制：行锁、间隙锁、next-key、意向锁、死锁

> 关联章节：[M02 索引](./04-精通-InnoDB-索引.md)、[M03 事务与 MVCC](./05-精通-InnoDB-事务-MVCC.md)

---

## 引言：锁是数据库最容易被误解的部分

锁是一切并发问题的本源。背书"MySQL 有行锁 + 表锁"很容易；解释下面这些问题就难了：

- `UPDATE t SET ... WHERE id=1` 锁的是行还是索引？
- `UPDATE t SET ... WHERE age>20` 在 RR 下到底锁了什么？
- 为什么两条看起来不相关的 SQL 会死锁？
- 间隙锁是什么？为什么 RC 没有它？
- 自增主键的并发写为什么不会冲突？
- 表锁、MDL、自增锁、行锁——分别在什么时候出现？

这一章把 InnoDB 锁体系从 Server 层（MDL）到 Engine 层（行锁、gap）讲完。读完之后你应该能：

- 用 `data_locks` 表读出任意一条 SQL 加了什么锁
- 解释为什么 RR 默认不会幻读
- 给热点更新（秒杀库存）设计不死锁的方案
- 看 `SHOW ENGINE INNODB STATUS` 的死锁日志
- 调试一次锁等待 / 死锁告警

---

## 第一章：锁的全景

### 1.1 两个层次

```
+--------------------------------------+
| Server 层锁                          |
|  - Metadata Lock (MDL)               |
|  - 全局锁（FTWRL）                    |
|  - Table Lock（LOCK TABLES）         |
+--------------------------------------+
| Engine 层（InnoDB）锁                |
|  - 表锁                              |
|     · 意向锁（IS / IX）              |
|     · AUTO-INC 锁                   |
|  - 行锁                              |
|     · Record Lock                    |
|     · Gap Lock                       |
|     · Next-Key Lock                  |
|     · Insert Intention Lock          |
|  - Predicate Lock (空间索引)         |
+--------------------------------------+
```

### 1.2 共享与排他

| 锁模式 | 符号 | 兼容 |
|---|---|---|
| Shared | S | S 可同获 |
| Exclusive | X | X 与其他都不兼容 |

读加 S，写加 X。

### 1.3 兼容矩阵

|  | S | X | IS | IX |
|---|---|---|---|---|
| **S**  | ✓ | ✗ | ✓ | ✗ |
| **X**  | ✗ | ✗ | ✗ | ✗ |
| **IS** | ✓ | ✗ | ✓ | ✓ |
| **IX** | ✗ | ✗ | ✓ | ✓ |

意向锁互不冲突——它们的作用是**快速判断"表里有没有别人在持行级 S/X 锁"**。

---

## 第二章：MDL —— 表结构保护

### 2.1 MDL 是 Server 层的元数据锁

任何操作开始时都加 MDL：

- SELECT / DML → MDL Shared Read
- DDL（ALTER / DROP）→ MDL Exclusive

含义：

- 读 SQL 不阻塞读 SQL
- 但**长 DDL** + **后续读 SQL** → MDL 队列堆积

### 2.2 MDL 引起的雪崩故障

```
T1: SELECT ... FROM huge_table  -- 长查询，持 MDL_SHARED_READ
T2: ALTER TABLE huge_table ADD COLUMN ...  -- 等 MDL_EXCLUSIVE，挂起
T3: SELECT ... FROM huge_table  -- 等 T2，因为 MDL 是 FIFO 队列
T4-Tn: 跟 T3 一样
```

T2 等 T1，T3+ 全等 T2 → **整张表的所有读写都挂起**，直到 T1 完成。

排查：

```sql
SELECT * FROM performance_schema.metadata_locks
WHERE OBJECT_SCHEMA = '<db>' AND OBJECT_NAME = '<table>';
```

### 2.3 MDL 应对手段

- 严控长事务（M03 已讲）
- DDL 加超时：`SET SESSION lock_wait_timeout = 5;` → 拿不到 MDL 5 秒就放弃
- 在线 DDL：`ALGORITHM=INPLACE, LOCK=NONE`（多数 ALTER 支持）
- 8.0+ INSTANT DDL：加列 / 删列在元数据级别瞬间完成
- 生产用 pt-osc / gh-ost 做大表 DDL（影子表 + binlog 同步）

---

## 第三章：意向锁

### 3.1 为什么需要意向锁

事务 A 持有某行的 X 锁。事务 B 想加表锁 → 需要确认"表上有没有行锁"。

挨个扫所有行检查 → 性能崩。**意向锁是"在加行锁前先在表上做个标记"**：

- 加行 S → 先加表 IS
- 加行 X → 先加表 IX

表锁来时：
- LOCK TABLES READ → 想加表 S → 跟 IX 冲突 → 等
- LOCK TABLES WRITE → 想加表 X → 跟 IS/IX 都冲突 → 等

InnoDB 自动加意向锁，不用业务关心。

### 3.2 意向锁互不冲突

两个事务都改不同行：

```
T1: 加行 R1 的 X → 表 IX
T2: 加行 R2 的 X → 表 IX
两个 IX 共存 ✓
```

只跟"显式表锁"冲突。

---

## 第四章：行级锁的三种形式

### 4.1 Record Lock（记录锁）

锁单条索引记录。

```sql
-- RR / RC，主键等值
SELECT * FROM t WHERE id = 10 FOR UPDATE;
-- 锁 id=10 的索引记录
```

### 4.2 Gap Lock（间隙锁）

锁索引区间，**不含端点**。

```sql
-- RR
SELECT * FROM t WHERE id > 10 AND id < 20 FOR UPDATE;
-- 锁 (10, 20) 这个 gap，防止别的事务插入 id=15 之类
```

Gap lock 唯一作用：**阻止 INSERT**。Gap lock 之间**不冲突**——两个事务都可以"持有同一个 gap 的锁"，但都不能往里插。

### 4.3 Next-Key Lock

= Record Lock + Gap Lock，左开右闭区间。

```sql
-- 索引上的记录：10, 15, 20, 25
SELECT * FROM t WHERE id > 10 AND id <= 20 FOR UPDATE;
-- 锁 (10, 15], (15, 20]
-- 即锁 15、20 两条记录 + (10,15) 与 (15,20) 两个 gap
```

**RR 隔离级别的默认行锁就是 next-key**，覆盖范围 + 边界，防幻读。

### 4.4 Insert Intention Lock（插入意向锁）

INSERT 时在目标 gap 上加。多个 INSERT 到同一 gap 但不同位置 → 兼容；与现有 gap lock 冲突 → 等。

---

## 第五章：RR 下的加锁规则

InnoDB 在 RR 下加锁有一套规则——能背下来对死锁分析至关重要。

### 5.1 原则

1. 默认 next-key（前开后闭）
2. 等值查询且唯一索引命中 → 降级为 record lock
3. 等值查询未命中 → 退化为 gap lock
4. 范围查询访问到的第一个不满足的值 → 锁定它的前 gap

### 5.2 例子：表 t(id PK, age, name) 数据 `id ∈ {10, 15, 20, 25, 30}`

#### 等值唯一命中

```sql
SELECT * FROM t WHERE id = 15 FOR UPDATE;
-- 锁：record(id=15)，因为唯一索引命中
```

#### 等值唯一未命中

```sql
SELECT * FROM t WHERE id = 17 FOR UPDATE;
-- 锁：gap(15, 20)，因为未命中，next-key 退化为 gap
```

#### 等值非唯一索引（如 age）

```sql
-- 数据：age=20 出现在 (id=15, age=20), (id=20, age=20)
SELECT * FROM t WHERE age = 20 FOR UPDATE;
-- 锁：next-key(age=20 所有命中行) + 下一条记录的 gap
```

#### 范围

```sql
SELECT * FROM t WHERE id > 15 AND id <= 25 FOR UPDATE;
-- 锁：next-key(15, 20], next-key(20, 25]
-- 即 record(20), record(25), gap(15,20), gap(20,25)

SELECT * FROM t WHERE id > 25 FOR UPDATE;
-- 锁：next-key(25, 30] + next-key(30, +∞) = (25, +∞)
```

### 5.3 工具：查看实际加锁

```sql
-- 终端 A
BEGIN;
SELECT * FROM t WHERE id > 15 AND id < 25 FOR UPDATE;

-- 终端 B
SELECT * FROM performance_schema.data_locks WHERE OBJECT_NAME='t';
```

输出列：

- `INDEX_NAME`：哪个索引
- `LOCK_TYPE`：RECORD / TABLE
- `LOCK_MODE`：S, X, IS, IX, X,GAP, X,REC_NOT_GAP, X,INSERT_INTENTION
- `LOCK_DATA`：锁住哪个值（gap 时是上界，next-key 时是命中记录值）

读这个表是排查锁的金标准。

---

## 第六章：RC 与 RR 加锁差异

### 6.1 RC 几乎不加 gap

```sql
SET SESSION transaction_isolation = 'READ-COMMITTED';

SELECT * FROM t WHERE id > 15 AND id <= 25 FOR UPDATE;
-- RC：仅锁 record(20), record(25)
-- RR：锁 (15, 20], (20, 25]，含 gap
```

RC 不加 gap → 并发好，但**有幻读**。

### 6.2 RC 下 UPDATE 的"半事务可见"

```sql
-- RC
BEGIN;
UPDATE t SET name='X' WHERE age > 20;
-- 此时锁的是 age>20 当前看到的所有行
-- 期间另一事务 INSERT age=25 并 COMMIT
-- 但当前事务不会锁这个新行（RC 无 gap）
COMMIT;
```

业务应用要清楚 RC 的"边界不稳定"行为，避免依赖事务内一致性。

### 6.3 何时选 RC

- 高并发 OLTP，写多读多：RC 锁短，并发好
- 不依赖事务内重复读
- 业务能容忍幻读
- 与 PostgreSQL 默认一致（PG 默认就是 RC）

---

## 第七章：插入与唯一索引

### 7.1 INSERT 触发的锁

```sql
INSERT INTO t (id, name) VALUES (12, 'foo');
```

1. 在 gap (10, 15) 加 insert intention lock
2. 如果有别的事务持有该 gap 的 gap lock → 等
3. 否则插入 + 加 record(12) 的 X 锁（持有到事务结束）

### 7.2 唯一索引冲突时的额外锁

```sql
-- 表已有 (id=10)
INSERT INTO t (id, name) VALUES (10, 'foo');
-- ERROR 1062 Duplicate entry
-- 但 InnoDB 在判定冲突前加了 S 锁
```

冲突时**短暂持 S 锁**——多个事务并发插同一值时，可能引发 S 锁等待 → 死锁。

### 7.3 唯一索引重复冲突死锁

```
事务 A             事务 B
INSERT id=10       
                   INSERT id=10  -- 等 A 的 S 锁释放
INSERT id=20  ←----+
                   INSERT id=20  -- 等
                   
A 和 B 互相等 → 死锁
```

业务上"两台机器同时插入唯一键冲突"导致死锁经常报。解决：业务先 SELECT 一次预判，或者用 `INSERT ... ON DUPLICATE KEY UPDATE`。

---

## 第八章：AUTO-INC 锁

### 8.1 三种自增锁模式

`innodb_autoinc_lock_mode`：

| 值 | 模式 | 行为 |
|---|---|---|
| 0 | traditional | 整个 INSERT 期间持表级自增锁 |
| 1 | consecutive | 批量 INSERT 持锁一次取连续 ID；单条 INSERT 用轻量锁 |
| **2（默认 8.0+）** | interleaved | 不持表锁，多事务交错取 ID（可能 ID 不连续） |

### 8.2 mode=2 的 binlog 限制（已不是问题）

历史上 mode=2 与 statement-based binlog 配合会乱序——5.7 前默认 1。但 8.0 默认 row-based binlog + mode=2，自增 ID 不连续是设计的正常表现。

### 8.3 不连续 ID 的来源

- mode=2 并发
- INSERT 失败（如唯一冲突）已经"领"了 ID
- ROLLBACK 后 ID 不回收

业务设计**不要假设自增 ID 连续**——这是隐形坑。

---

## 第九章：死锁

### 9.1 死锁的形式化定义

两个或多个事务相互持锁等待，形成循环。

```
T1 持 lock A，等 lock B
T2 持 lock B，等 lock A
→ 死锁
```

### 9.2 InnoDB 的死锁检测

每个事务等锁时，InnoDB 走 wait-for graph：

- 看自己等的锁被谁持
- 那个事务又在等谁
- 如果回到自己 → 死锁

检测到死锁，InnoDB 选**回滚代价最小**的事务作为 victim 杀掉，另一个成功。

监控：

```sql
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK
-- TRANSACTION 1: holds, waits ...
-- TRANSACTION 2: holds, waits ...
-- WE ROLL BACK TRANSACTION (1)
```

### 9.3 死锁的常见模式

#### 不同顺序更新

```
T1: UPDATE t SET ... WHERE id=1;  UPDATE t SET ... WHERE id=2;
T2: UPDATE t SET ... WHERE id=2;  UPDATE t SET ... WHERE id=1;
```

修复：**约定全局加锁顺序**（按主键升序）。

#### 二级索引 + 回表

```
-- idx_status
T1: UPDATE t WHERE status='paid';  -- 二级索引加锁 → 回表加聚簇锁
T2: UPDATE t WHERE id=10;           -- 直接聚簇锁

如果 T1 先持二级索引，T2 先持聚簇 → 互等
```

### 9.4 死锁检测的 CPU 代价

`innodb_deadlock_detect` 默认 ON。高并发下，**每次等锁就跑一次 graph 检测**，CPU 飙升。

5.7+ 可以关检测，配合超时（`innodb_lock_wait_timeout`）：

```sql
SET GLOBAL innodb_deadlock_detect = OFF;
SET GLOBAL innodb_lock_wait_timeout = 5;  -- 5s 拿不到锁就报错
```

代价：真死锁需要等 5 秒才报错。

**生产经验**：单机 QPS 几万以内保持检测 ON；几十万以上、热点行竞争激烈 → 关检测 + 调短超时。

---

## 第十章：典型死锁案例

### 10.1 唯一索引并发插入

```
-- T1, T2 并发
INSERT INTO t (uniq_col) VALUES ('x');

-- 一方先插入并 COMMIT，另一方报 Duplicate
-- 期间会持 S 锁判重 → 多事务交叉 → 死锁
```

### 10.2 间隙锁互相阻塞

```
-- RR，数据 id={5, 10}
T1: SELECT * FROM t WHERE id=7 FOR UPDATE;  -- 锁 gap(5, 10)
T2: SELECT * FROM t WHERE id=8 FOR UPDATE;  -- 锁 gap(5, 10)
-- gap lock 互不冲突，OK

T1: INSERT INTO t VALUES (7);   -- 想插 gap(5,10)，但 T2 持 gap lock → 等
T2: INSERT INTO t VALUES (8);   -- 想插 gap(5,10)，但 T1 持 gap lock → 等
→ 死锁
```

### 10.3 二级索引 + 回表死锁

```
-- idx_name
T1: UPDATE t SET age=age+1 WHERE name='Alice';
    -- 在 idx_name 上加 X 锁 → 拿主键 → 在聚簇上加 X 锁

T2: UPDATE t SET name='Bob' WHERE id=10;
    -- 直接在聚簇加 X 锁 → 修改 name 时维护 idx_name → 在 idx_name 加锁

如果 T1 持 idx_name 等聚簇，T2 持聚簇等 idx_name → 死锁
```

修复：在 T1 改造为 `UPDATE t SET age=age+1 WHERE id IN (SELECT id FROM t WHERE name='Alice')`，分离两步。

### 10.4 秒杀库存死锁

```python
# 错误：每个并发请求都 SELECT 然后 UPDATE
def buy():
    cursor.execute("SELECT stock FROM goods WHERE id=1 FOR UPDATE")
    cursor.execute("UPDATE goods SET stock=stock-1 WHERE id=1")
```

并发多个请求 → 互等 → 一堆 deadlock。

修复用原子 UPDATE：

```python
cursor.execute("UPDATE goods SET stock=stock-1 WHERE id=1 AND stock>0")
if cursor.rowcount == 0:
    raise OutOfStock()
```

单条 UPDATE 内部原子，所有并发请求顺序执行单条 SQL，无死锁。

---

## 第十一章：诊断锁等待

### 11.1 看锁等待链

8.0+：

```sql
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_engine_transaction_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_engine_transaction_id;
```

### 11.2 sys.innodb_lock_waits 简化视图

```sql
SELECT * FROM sys.innodb_lock_waits\G
```

更可读，包含等多久、谁阻谁。

### 11.3 杀掉阻塞源

```sql
KILL <blocking_thread_id>;
```

注意：KILL 会回滚该事务，业务可能丢更新。**先排查再操作**。

### 11.4 配置告警

```ini
# 日志记录所有锁等待 > 10s
innodb_print_all_deadlocks = ON
long_lock_wait_time = 10        -- 自己实现监控阈值
```

线上集成 Prometheus 监控：

- `mysql_global_status_innodb_row_lock_time_avg`
- `mysql_global_status_innodb_row_lock_waits`

---

## 第十二章：实战陷阱

### 12.1 用 SELECT FOR UPDATE 当锁不释放

```
BEGIN;
SELECT * FROM t WHERE id=1 FOR UPDATE;
-- 业务逻辑出错没 commit，连接没释放
-- 锁一直持有！别的事务全等
```

修复：业务异常路径必须 ROLLBACK，连接超时主动清理。

### 12.2 没显式 begin，但 autocommit=0

```
autocommit = 0
SELECT * FROM t WHERE id=1 FOR UPDATE;
-- 实际上开了事务，FOR UPDATE 持锁
-- 客户端没意识到要 commit
```

某些 ORM 默认 autocommit=0。仔细配。

### 12.3 索引选错导致全表加锁

```sql
-- 索引：idx_age
UPDATE t SET name='X' WHERE age=25 AND status='paid';
-- 如果优化器选了 idx_age：锁 age=25 的所有行
-- 如果走全表：锁全表所有行！

-- 加索引 (age, status) 让锁更精确
```

EXPLAIN 验证。锁范围 = 走索引时索引上的扫描范围。

### 12.4 RR 下"间隙锁包过头"

```sql
-- 数据 id={1, 100}
SELECT * FROM t WHERE id=50 FOR UPDATE;
-- 锁 gap(1, 100)，禁止 id 2-99 插入
-- 但业务可能完全没想到要锁这么大的范围
```

改 RC 或精确条件。

### 12.5 KILL 事务的延迟

`KILL` 不是瞬时的——InnoDB 要做 ROLLBACK，大事务 KILL 后回滚可能要几分钟（甚至比正常运行还久）。期间 锁仍持有。所以**预防 > 抢救**。

---

## 第十三章：与其他章节的连接

- **与 M02 索引**：锁是基于索引的；选错索引 → 锁范围变大
- **与 M03 事务**：MVCC 解决"读快照"，锁解决"写并发"；当前读 + gap 防幻读
- **与 M06 优化器**：优化器选错索引 → 锁范围错
- **与 M07 PS**：data_locks / data_lock_waits / metadata_locks 都在 PS
- **与 M08 调优**：lock_wait_timeout / deadlock_detect 参数

---

## 总结

InnoDB 锁体系：MDL + 行锁 + gap + 意向锁 + 自增锁。关键点：

1. **MDL**：长事务 + DDL = 雪崩
2. **意向锁**：表级标记，快速判断"有没有行锁"
3. **Record / Gap / Next-Key**：RR 默认 next-key 防幻读
4. **RC 几乎无 gap**：并发好，幻读
5. **AUTO-INC mode 2**：默认，ID 可能不连续
6. **死锁**：自动检测 + 回滚 victim；高热点要关检测
7. **常见死锁模式**：不同顺序更新、唯一冲突、二级索引回表
8. **诊断**：data_lock_waits + innodb_trx + KILL

---

## 练习题

1. 用 RR 在表 t(id PK, age, idx_age) 上执行 `SELECT * FROM t WHERE age=20 FOR UPDATE`，描述加了哪些锁。
2. 改 RC 后再执行同样的 SQL，对比锁差异。
3. 写一组 SQL 故意造成死锁，运行后观察 `SHOW ENGINE INNODB STATUS` 死锁日志。
4. 一个长 SELECT 持 MDL_SHARED_READ，DDL 在后排队，新的 SELECT 也排队——画出等待图。
5. `INSERT ... ON DUPLICATE KEY UPDATE` 与 `INSERT` + 业务判 Duplicate 哪个并发性能更好？为什么？
6. 秒杀场景下，给一段死锁多发的代码并改写成无死锁版本。
7. `data_locks` 表里 `LOCK_MODE = X,GAP` 和 `X,REC_NOT_GAP` 分别表示什么？
8. 关闭 `innodb_deadlock_detect` 的利弊？什么场景该关？
9. 用 SQL 找出当前持有最长锁的事务。
10. 表 t 用了 idx_status，业务大量 UPDATE WHERE status='paid'。10 万行 paid → 整个事务持锁多长？怎么改？

---

> 🔁 反馈：拿两个 mysql client 同时跑 SELECT FOR UPDATE，亲自看到 next-key 锁住 gap
