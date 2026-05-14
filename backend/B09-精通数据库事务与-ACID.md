# 精通数据库事务与 ACID

> 课程编号：B09
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Transactions & ACID
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：从转账说起

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- from
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- to
COMMIT;
```

如果中间崩溃？两个并发转账操作同一账户？10000 个并发账单生成？ACID 是数据库 60 年来回答这些问题的核心。本章拆开 ACID 每个字母、隔离级别的实际差异、MVCC、锁、死锁、分布式事务。

---

## 第一章：ACID

### 1.1 四个保证

- **Atomicity**（原子性）：整个事务要么全成功要么全回滚——不可能"半个完成"
- **Consistency**（一致性）：事务把 DB 从一个合法状态变到另一个；约束、触发器必须满足
- **Isolation**（隔离性）：并发事务互不干扰，看起来像串行执行
- **Durability**（持久性）：commit 后数据永久保存，即使断电

### 1.2 Consistency 的歧义

"一致性"在数据库 ACID 中是**应用层概念**——只要 transaction 满足业务规则（FK、check constraint、唯一约束等），不是说"所有节点最新"（这是 CAP 中的 C，参见 B12）。

### 1.3 Atomicity 怎么实现

WAL（Write-Ahead Log）：所有变更先写日志再改实际数据页。崩溃后重放日志 redo 或撤销 undo。

---

## 第二章：隔离级别

### 2.1 SQL 标准四级

由弱到强：

| 级别 | 脏读 | 不可重复读 | 幻读 |
|---|---|---|---|
| READ UNCOMMITTED | ❌ | ❌ | ❌ |
| READ COMMITTED | ✅ | ❌ | ❌ |
| REPEATABLE READ | ✅ | ✅ | ❌ |
| SERIALIZABLE | ✅ | ✅ | ✅ |

✅ = 防止；❌ = 可能发生。

### 2.2 现象解释

**脏读（dirty read）**：读到其他事务未 commit 的数据。极少 DB 默认开（Oracle 完全不支持）。

**不可重复读（non-repeatable read）**：同一事务内读两次同一行，结果不同（其他事务 commit 了 UPDATE）。

**幻读（phantom read）**：同一事务内两次范围查询结果集大小不同（其他事务 INSERT 新行）。

### 2.3 各 DB 默认

| DB | 默认 |
|---|---|
| PostgreSQL | READ COMMITTED |
| MySQL InnoDB | REPEATABLE READ |
| Oracle | READ COMMITTED |
| SQL Server | READ COMMITTED |

`SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;` 改级别。

### 2.4 SERIALIZABLE 实现

- **悲观**：行锁、范围锁（gap lock）
- **乐观（PostgreSQL SSI）**：执行时跟踪依赖；提交时检测冲突 → 失败重试

SERIALIZABLE 性能代价大，且会出现 retry。一般用 READ COMMITTED + 应用层 SELECT FOR UPDATE 控制。

---

## 第三章：MVCC（多版本并发控制）

### 3.1 思路

每次 UPDATE/DELETE 不覆盖原数据，**写新版本**；每个事务看到一个一致的快照。

```
事务 T1 (snapshot at time S1):
  读 user 42 → version_2 (created < S1)

事务 T2 在 T1 中间 commit 了 user 42 → version_3

T1 仍读 version_2 (不可见 version_3)
```

### 3.2 优点

- **读不阻塞写**：读取读旧版本
- **写不阻塞读**：写新版本
- 高并发友好

### 3.3 代价

- 旧版本要清理（PostgreSQL VACUUM、MySQL purge）
- 表空间可能膨胀
- 长事务持有旧 snapshot 阻止清理

### 3.4 PostgreSQL vs MySQL InnoDB

**PostgreSQL**：每行 (xmin, xmax) 标记创建/删除事务 ID。VACUUM 定期清理。

**InnoDB**：聚集索引的叶节点存当前版本；旧版本在 undo log。purge 线程清理。

---

## 第四章：锁

### 4.1 行锁

```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

锁定该行直到事务结束，其他事务等待。

### 4.2 共享锁 vs 排他锁

| 锁 | 名 | 兼容其他 share | 兼容 exclusive |
|---|---|---|---|
| 共享（S） | `FOR SHARE` | ✅ | ❌ |
| 排他（X） | `FOR UPDATE` | ❌ | ❌ |

多事务可同时读 S 锁；写需要 X 锁排他。

### 4.3 范围锁（gap lock）

防幻读。`WHERE age > 18 FOR UPDATE` 锁定整个 `age > 18` 区间，新 INSERT 阻塞。

InnoDB 在 REPEATABLE READ 下自动加 gap lock；PostgreSQL 用 SSI 不需要显式 gap lock。

### 4.4 表锁

`LOCK TABLE foo IN ACCESS EXCLUSIVE MODE;` 锁整张表——一般避免。

### 4.5 锁等待与死锁

```
T1: locks A, waits for B
T2: locks B, waits for A
→ 死锁
```

DB 自动检测：选一个 victim 回滚，让另一个继续。`ERROR: deadlock detected`。

**应用代码必须能 retry**——死锁不可完全避免。

### 4.6 减少死锁

- **固定加锁顺序**：所有事务按相同顺序锁多个资源
- **缩短事务**：拿锁后快速完成
- **降低隔离级别**：READ COMMITTED 锁更少
- **行锁粒度**：避免 LOCK TABLE

---

## 第五章：事务模式

### 5.1 显式

```sql
BEGIN;
-- ...
COMMIT;   -- or ROLLBACK;
```

### 5.2 隐式（auto-commit）

默认每条 SQL 都是独立事务，自动 commit。`BEGIN` 关闭它。

### 5.3 SAVEPOINT

部分回滚：

```sql
BEGIN;
INSERT ...;
SAVEPOINT before_risky;
INSERT ...;
ROLLBACK TO before_risky;   -- 回到 savepoint，外层事务继续
COMMIT;
```

### 5.4 应用层 wrapper

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()
    if err := fn(tx); err != nil { return err }
    return tx.Commit()
}
```

`defer Rollback`：成功 commit 后 rollback 是 no-op；失败时自动回滚。

---

## 第六章：锁与并发实例

### 6.1 经典 race condition

```
T1: SELECT balance FROM accounts WHERE id = 1;   -- 100
T2: SELECT balance FROM accounts WHERE id = 1;   -- 100
T1: UPDATE accounts SET balance = 100 - 50;       -- 50
T2: UPDATE accounts SET balance = 100 - 30;       -- 70
```

两次扣款，但 T1 的扣款被覆盖。**Lost update**——典型并发 bug。

### 6.2 解法 A：SELECT FOR UPDATE

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;   -- 加 X 锁
-- 应用判断
UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;
```

### 6.3 解法 B：原子 UPDATE

```sql
UPDATE accounts SET balance = balance - 50 WHERE id = 1 AND balance >= 50;
```

DB 内部加 X 锁；返回 rows affected = 0 表示余额不足。

### 6.4 解法 C：乐观锁（version 字段）

```sql
UPDATE accounts SET balance = ?, version = version + 1
WHERE id = ? AND version = ?;
```

返回 0 → 别的事务改过 → retry。

### 6.5 三种选择

- 强冲突场景 → 悲观（FOR UPDATE）
- 弱冲突 + 高并发 → 乐观（version）
- 简单状态机 → 原子 UPDATE

---

## 第七章：分布式事务

### 7.1 跨数据库

订单服务（DB A）+ 支付服务（DB B）。怎么保证两边一致？

### 7.2 2PC（两阶段提交）

```
协调者 → 准备阶段：所有参与者 "可以 commit 吗？"
所有 YES → 提交阶段："commit!"
有 NO → "rollback!"
```

问题：
- 协调者崩溃 → 参与者阻塞
- 慢，所有参与者等
- 不适合微服务

### 7.3 Saga 模式

把分布式事务拆为本地事务序列 + 补偿操作：

```
1. order-service: 创建订单 (pending)
2. payment-service: 扣款
3. inventory-service: 减库存
   ↓ 失败
2'. payment-service: 退款（补偿）
1'. order-service: 取消订单
```

每步可独立 commit，失败时逆序 compensate。**最终一致性**，不是 ACID。

### 7.4 Outbox 模式

事务内同时写业务表 + outbox 表（待发送消息）；后台 worker 读 outbox 发到消息队列。

```sql
BEGIN;
INSERT INTO orders ...;
INSERT INTO outbox (event_type, payload) VALUES ('order_created', ...);
COMMIT;
```

保证"业务变更 + 事件发布"原子。Kafka Connect 或 Debezium 自动化。

### 7.5 idempotency

下游处理消息必须幂等——同一事件可能重发。

---

## 第八章：性能与监控

### 8.1 锁等待

PostgreSQL：
```sql
SELECT * FROM pg_locks WHERE NOT granted;
SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';
```

MySQL：
```sql
SELECT * FROM information_schema.innodb_lock_waits;
SHOW ENGINE INNODB STATUS;
```

### 8.2 长事务

长事务持有锁、阻止 VACUUM、占内存。监控并 kill：

```sql
-- PostgreSQL
SELECT pid, age(now(), xact_start) AS dur, query
FROM pg_stat_activity
WHERE state = 'active' AND xact_start < now() - interval '5 minutes';
```

### 8.3 死锁日志

PostgreSQL：默认 log；通过 `log_lock_waits = on` 看锁等待。
MySQL：`SHOW ENGINE INNODB STATUS` 看最近一次死锁。

---

## 第九章：生产级最佳实践

1. **事务尽量短**：减少锁持有、减少死锁。
2. **事务边界清晰**：业务层 wrapper（WithTx）。
3. **优先原子 UPDATE**：避免读-改-写。
4. **加锁顺序固定**：防死锁。
5. **接受死锁 + retry**：业务代码包 retry with backoff。
6. **READ COMMITTED 默认**：性能 + 实际够用。
7. **应用层乐观锁**：高并发 + 弱冲突。
8. **分布式用 Saga 而非 2PC**：可扩展。
9. **Outbox 模式发事件**：业务 + 事件原子。
10. **监控锁等待 + 长事务**：及时报警。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：忘 commit 或 rollback
长事务持锁 + 占连接。永远 `defer tx.Rollback()`。

### ❌ 陷阱 2：读-改-写无锁
```python
balance = db.query("SELECT balance FROM ...")
balance -= 100
db.execute("UPDATE ... SET balance = ?", balance)
```
经典 lost update。

### ❌ 陷阱 3：交互式事务
人类点击驱动的事务（界面等用户确认）→ 锁数分钟。改设计。

### ❌ 陷阱 4：用 SERIALIZABLE 不 retry
PostgreSQL SSI 可能 throw serialization error；不处理直接 500。

### ❌ 陷阱 5：在事务内调外部服务
HTTP/RPC 慢、可能挂 → 长事务。把外部调用移出事务，用 outbox 异步。

### ❌ 陷阱 6：嵌套 begin
```sql
BEGIN;
  BEGIN;   -- 第二个 BEGIN 在多数 DB 忽略或报错
  ...
COMMIT;
```
用 SAVEPOINT 表达"子事务"。

### ❌ 陷阱 7：跨服务事务依赖 2PC
微服务环境 2PC 复杂、慢、单点故障。Saga 更现实。

---

## 第十一章：练习题

**练习 1**：解释 READ COMMITTED 与 REPEATABLE READ 在以下场景的差异：
```
T1: BEGIN; SELECT balance FROM a WHERE id=1;  -- 100
T2: BEGIN; UPDATE a SET balance=200 WHERE id=1; COMMIT;
T1: SELECT balance FROM a WHERE id=1;  -- ?
```

**练习 2**：以下转账代码会发生什么并发问题？怎么改？
```python
def transfer(from_id, to_id, amount):
    with db.transaction():
        bal = db.query("SELECT balance FROM a WHERE id=?", from_id)
        if bal < amount: raise Insufficient
        db.execute("UPDATE a SET balance=balance-? WHERE id=?", amount, from_id)
        db.execute("UPDATE a SET balance=balance+? WHERE id=?", amount, to_id)
```

**练习 3**：为何长事务阻止 PostgreSQL VACUUM 清理旧版本？

**练习 4**：设计一个 Saga：电商下单需要 (1) 创建订单 (2) 扣库存 (3) 扣账户余额。

**练习 5**：解释 outbox 模式如何保证"业务变更 + 消息发送"原子。

---

## 参考答案

**练习 1**：
- READ COMMITTED：T1 第二次 SELECT 返回 200（看到了 T2 的 commit）—— 不可重复读
- REPEATABLE READ：T1 第二次 SELECT 仍返回 100（事务初始快照）

**练习 2**：lost update 风险 + race condition（两个并发转账可能 over-draft）。修：
```python
# 用原子 UPDATE + 条件
n = db.execute("UPDATE a SET balance=balance-? WHERE id=? AND balance>=?",
               amount, from_id, amount)
if n == 0: raise Insufficient
```
或 `SELECT ... FOR UPDATE` 加锁。

**练习 3**：旧版本要被 PostgreSQL 看作"没有事务还需要看它"才能清理。长事务持有一个 snapshot，可能看旧版本，所以 VACUUM 不能清。

**练习 4**：
```
正向：
1. order-service: INSERT order (status=pending)
2. inventory-service: 减库存
3. account-service: 扣余额
4. order-service: UPDATE order SET status=confirmed

任一失败 → 逆序补偿：
4'. (n/a，状态在 1 之后才改)
3'. account-service: 退余额
2'. inventory-service: 加库存
1'. order-service: UPDATE order SET status=cancelled
```

**练习 5**：本地事务原子写两个表：业务表 + outbox 表。要么都成功要么都失败。后台 worker 异步读 outbox 推 Kafka——这个失败 → 重试，因为消息已在 outbox 持久化。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| ACID | Atomicity / Consistency / Isolation / Durability |
| 隔离级别 | RC < RR < SERIALIZABLE |
| MVCC | 多版本快照；读不阻塞写 |
| 锁 | FOR UPDATE / FOR SHARE / gap lock |
| 死锁 | 自动检测 + retry |
| 原子 UPDATE | 防 lost update 首选 |
| 乐观锁 | version 字段 + retry |
| 分布式 | Saga 优于 2PC；outbox 模式 |

下一篇 **B10 — 规范化 vs 反规范化** 将拆 1NF/2NF/3NF/BCNF、反规范化时机、典型 trade-off。

---

