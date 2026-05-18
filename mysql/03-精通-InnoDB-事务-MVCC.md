# 精通 InnoDB 事务与 MVCC：undo、redo、ReadView、隔离级别底层

> 关联章节：[M02 索引](./02-精通-InnoDB-索引.md)、[M04 锁](./04-精通-MySQL-锁.md)、[M05 复制](./05-精通-Binlog-与复制.md)

---

## 引言：ACID 不是口号——是 4 个机制的联动

每个学过数据库的人都背过 ACID：

- **A**tomicity 原子性
- **C**onsistency 一致性
- **I**solation 隔离性
- **D**urability 持久性

但很少人能解释 **InnoDB 是用哪些机制实现这 4 个性质的**。事实上，这 4 个性质不是 4 个独立模块，而是 4 套机制相互配合的产物：

| 性质 | 主要靠 |
|---|---|
| Atomicity | undo log（失败回滚） |
| Consistency | 上述三者综合 + 应用层约束 |
| Isolation | MVCC（ReadView）+ 锁 |
| Durability | redo log（崩溃恢复）+ binlog（复制持久化） |

绕不开的核心机制：**undo / redo / MVCC / 锁**。本章把前三个讲透，锁单独留 M04。

读完这一章你应该能回答：

- 一次 `UPDATE` 的具体写入顺序是什么？为什么需要"两阶段提交"？
- ReadView 怎么"用一个时间点的快照"来实现 RR 隔离级别？
- RR 下能完全防止幻读吗？什么场景下还会读到？
- undo 不是只用来回滚的——它还干什么？
- crash 后 InnoDB 怎么恢复到一致状态？
- `READ COMMITTED` 比 `REPEATABLE READ` 性能好吗？什么场景该用？

---

## 第一章：事务的基本面

### 1.1 一个事务的生命周期

```
BEGIN;                          ← 取一个事务 ID（trx_id），分配 undo
SELECT * FROM ...               ← 读：构造 ReadView，走 MVCC
UPDATE ... SET ... WHERE ...;   ← 写：先写 undo，改聚簇索引，写 redo
INSERT INTO ...;                ← 写
COMMIT;                         ← redo prepare → 写 binlog → redo commit
                                  ← 释放锁；undo 留作其他事务的 MVCC 历史
```

回滚路径：

```
ROLLBACK;                       ← 用 undo 反向操作回到事务开始前
                                  ← 释放锁
```

### 1.2 事务的隔离级别

SQL 标准 4 个 + InnoDB 默认：

| 级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
|---|---|---|---|---|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 几乎不用（读未提交数据） |
| READ COMMITTED (RC) | ✗ | ✓ | ✓ | 每条 SELECT 重建 ReadView |
| **REPEATABLE READ (RR)** | ✗ | ✗ | 部分 | **InnoDB 默认**；事务首次读时建 ReadView，后续复用 + gap lock 防幻读 |
| SERIALIZABLE | ✗ | ✗ | ✗ | 所有读自动加共享锁；性能最差 |

InnoDB 默认 RR；RC 是另一个常用选择，特别是高并发 OLTP。

### 1.3 隔离级别的实现机制

| | RC | RR |
|---|---|---|
| ReadView 创建 | **每条 SELECT 重建** | **第一条 SELECT 建一次** |
| 看到的快照 | 总是最新已提交 | 事务开始时的快照 |
| Gap lock | 不加 | 加（防幻读） |
| 锁粒度 | Record | Next-key (Record + Gap) |

下面把这套机制拆开讲。

---

## 第二章：undo log

### 2.1 undo 是"反向 SQL"

每次写都先记 undo：

| 操作 | undo 记录 |
|---|---|
| INSERT | DELETE（删除该 PK） |
| UPDATE | UPDATE 回旧值 |
| DELETE | INSERT 回旧行 |

回滚就是按 undo 反向操作。

### 2.2 undo 的两个角色

undo 不只用来回滚——还实现 MVCC 的"历史版本链"。

```
聚簇索引行：(id=10, name='Bob', trx_id=99, roll_ptr → undo entry)
                                                       |
                                              undo (UPDATE):
                                              old_name='Alice', trx_id=88, roll_ptr → 更早 undo
```

每条记录的 `DB_ROLL_PTR` 指向 undo log，undo log 内部也有指向更老版本的指针——形成**版本链**。这就是 MVCC 看历史版本的物理基础。

### 2.3 undo 的物理存储

```
undo log → 存于 undo tablespace（独立 .ibu 文件，5.7+）
```

8.0+ 默认 2 个 undo tablespace，可在线增减。`innodb_undo_log_truncate=ON` 让 undo 不再无限膨胀（自动 truncate）。

监控 undo 大小：

```sql
SELECT * FROM information_schema.INNODB_TABLESPACES WHERE NAME LIKE '%undo%';
```

### 2.4 undo 大小膨胀的常见根因

**长事务**——只要事务还在跑，它需要看到的旧版本就不能 purge → undo 一直保留。

```sql
-- 看长事务
SELECT * FROM information_schema.innodb_trx ORDER BY trx_started ASC;
```

排查清单：

- 业务里 BEGIN 后没及时 COMMIT
- 应用代码异常没 ROLLBACK
- mysqldump（默认开事务做一致性快照）
- 在备库做长查询（备库的事务也阻碍主库 undo purge——除非用 `--single-transaction` 之外的方式）

---

## 第三章：redo log

### 3.1 redo 是"重做日志"——崩溃恢复用

InnoDB 不直接把改动写磁盘（页层面），而是先写 redo log。redo 是**追加写**到一个固定大小的循环文件，速度极快。

```
事务执行：
  改 Buffer Pool 中的页（in-memory）
  追加 redo 记录（"页 X 偏移 Y 改成 Z"）

事务 COMMIT：
  fsync redo log → 持久化保证

崩溃恢复：
  读 redo log → 重放所有未刷盘的页改动
```

这就是 **WAL（Write-Ahead Logging）**——日志先于数据落盘。

### 3.2 redo 的物理结构

```
ib_logfile0, ib_logfile1, ...  循环写入
```

8.0+ 改成动态扩容/缩容（不再固定文件数）。大小由 `innodb_redo_log_capacity` 控制（8.0.30+），默认 100MB，生产建议 4-8GB。

```sql
SET PERSIST innodb_redo_log_capacity = 8589934592;  -- 8GB
```

### 3.3 LSN（Log Sequence Number）

InnoDB 给每次 redo 写入分配一个全局递增的 LSN。

- 每个页有 `FIL_PAGE_LSN`：页最后一次修改对应的 LSN
- redo log 也按 LSN 顺序追加

崩溃恢复时：

```
对每个页 P：
  if P.FIL_PAGE_LSN < 最大 redo LSN：
    重放 (P.FIL_PAGE_LSN, max_lsn] 之间的 redo
```

监控：

```sql
SHOW ENGINE INNODB STATUS\G
-- LOG section：
-- Log sequence number  当前 LSN
-- Log flushed up to    已 fsync 的 LSN
-- Pages flushed up to  已写盘的脏页 LSN
```

`Log sequence number - Pages flushed up to` 就是脏页累积量。

### 3.4 三种 fsync 策略：innodb_flush_log_at_trx_commit

| 值 | 行为 | 性能 | 数据安全 |
|---|---|---|---|
| **1（默认）** | 每次 COMMIT 都 fsync | 慢 | 最多丢 0 行 |
| 2 | COMMIT 时 write 到 OS（不 fsync），后台每秒 fsync | 中 | crash 丢 ≤1s 已提交事务 |
| 0 | 不在 COMMIT 时 write，后台每秒 write+fsync | 快 | crash 丢 ≤1s 已提交事务 |

生产 OLTP 必须 1。秒级日志 / 缓存可以容忍 0/2。

---

## 第四章：两阶段提交（2PC）—— redo 与 binlog 的协同

### 4.1 为什么需要 2PC

InnoDB 写 redo（保证引擎崩溃恢复），Server 层写 binlog（保证主从一致 / 时间点恢复）。如果两者写入顺序错乱，会有数据不一致：

- 只写 redo 不写 binlog → 主库有数据但从库没复制
- 只写 binlog 不写 redo → 从库有数据但主库 crash 后丢了

### 4.2 InnoDB 的 2PC 流程

一次 COMMIT：

```
1. innobase_xa_prepare:
   redo log 写 PREPARE 标记 + fsync
   生成 XID
2. binlog 写入 + fsync
3. innobase_xa_commit:
   redo log 写 COMMIT 标记 + fsync (sync_binlog 与 commit 顺序)
```

崩溃恢复时：

- redo 有 PREPARE 但 binlog 没对应 XID → ROLLBACK
- redo 有 PREPARE 且 binlog 有 XID → COMMIT
- redo 有 COMMIT → 不管，已经完成

### 4.3 sync_binlog 控制 binlog 持久化

| 值 | 行为 |
|---|---|
| 0 | 由 OS 决定 fsync 时机 |
| 1（推荐） | 每次写完 binlog 立刻 fsync |
| N>1 | 每 N 次事务 fsync 一次 |

生产几乎必须 `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1`——双 1，最安全。

### 4.4 Group Commit 优化

如果每事务都 fsync 两次（redo + binlog），TPS 受 IOPS 严重限制。

InnoDB 5.6+ 实现 **group commit**：把同一时刻并发的事务的 fsync 合并成一次磁盘 sync。多个事务的 redo / binlog 一次 fsync 全部落盘 → IOPS 利用率大幅提升。

参数：

```
binlog_group_commit_sync_delay = 10000   -- 微秒，等多少
binlog_group_commit_sync_no_delay_count = 100  -- 累积多少笔强制提交
```

---

## 第五章：MVCC（多版本并发控制）

### 5.1 MVCC 的核心思想

读不阻塞写、写不阻塞读——**靠"读历史版本"实现**。每条记录有版本链，事务看到的是它能"看到"的最新版本。

InnoDB 通过两个机制实现：

1. **隐藏列** `DB_TRX_ID`、`DB_ROLL_PTR` —— 记录"谁改的、上一版本在哪"
2. **ReadView** —— 事务的"快照标尺"，决定能看到哪些版本

### 5.2 ReadView 的结构

```
struct ReadView {
    trx_id_t low_limit_id;      // 当前活跃事务的最大 ID + 1（之后的都看不到）
    trx_id_t up_limit_id;       // 当前活跃事务的最小 ID（之前的都看得到）
    trx_id_t creator_trx_id;    // 当前事务自己的 ID
    vector<trx_id_t> m_ids;     // 所有当前活跃（未提交）事务 ID 列表
};
```

### 5.3 可见性判断算法

对一行 `R`，看 `R.DB_TRX_ID = trx_id`：

```
if trx_id == creator_trx_id:
    可见  // 自己的修改总能看到
elif trx_id < up_limit_id:
    可见  // ReadView 创建前已提交
elif trx_id >= low_limit_id:
    不可见  // ReadView 创建后才开始
elif trx_id in m_ids:
    不可见  // ReadView 创建时还没提交
else:
    可见  // ReadView 创建时已提交
```

如果当前版本不可见，**沿 `roll_ptr` 找上一版本**，重复判断，直到找到可见的（或没有 → 该行对当前事务不存在）。

### 5.4 RR 与 RC 在 ReadView 创建时机的差别

```
RR：事务开始后第一次 SELECT 时建一次 ReadView，后续所有读都用同一个
RC：每次 SELECT 都重建 ReadView
```

这是为什么：

- **RR 实现"可重复读"**：同一事务内多次读同一行结果一致
- **RC 实现"读已提交"**：每次读都能看到最新已提交数据

### 5.5 一个例子

```
事务 A（RR）           事务 B
BEGIN;
SELECT * FROM t WHERE id=1;  → 'Alice'
                       BEGIN;
                       UPDATE t SET name='Bob' WHERE id=1;
                       COMMIT;
SELECT * FROM t WHERE id=1;  → 'Alice'  ← 仍看到旧版本（RR）
COMMIT;

# 同样场景，A 改成 RC：
SELECT * FROM t WHERE id=1;  → 'Bob'   ← 重建 ReadView 看到新提交
```

---

## 第六章：当前读 vs 快照读

### 6.1 快照读（snapshot read）—— 走 MVCC

普通 SELECT：

```sql
SELECT * FROM t WHERE id = 1;
```

通过 ReadView 看历史版本，**不加锁**。这是 MVCC 的舒适场景。

### 6.2 当前读（current read）—— 读最新 + 加锁

以下都是当前读，绕过 MVCC，加行锁：

```sql
SELECT ... FOR UPDATE              -- 加 X 锁
SELECT ... FOR SHARE / LOCK IN SHARE MODE  -- 加 S 锁
INSERT / UPDATE / DELETE           -- 隐式当前读
```

为什么写操作必须当前读？因为不能基于旧版本去改——会丢更新（lost update）。

### 6.3 RR 下幻读问题的解决

```
事务 A（RR）              事务 B
BEGIN;
SELECT * FROM t WHERE age>20;  → 0 行
                          BEGIN;
                          INSERT INTO t (name, age) VALUES ('Bob', 25);
                          COMMIT;
SELECT * FROM t WHERE age>20;  → 0 行（快照读，看不到 B 的）
SELECT * FROM t WHERE age>20 FOR UPDATE;  → 1 行（当前读！看到 B 的）
INSERT INTO t (id, ...) VALUES (B_already_inserted_id, ...);  → 主键冲突
```

最后一行就是经典的"半幻读"——快照看不到，当前读看到。**InnoDB 用 next-key lock 在当前读时锁住范围，避免别的事务插入**——这是 RR 下防幻读的核心手段。

详见 M04 锁。

---

## 第七章：purge 线程

### 7.1 purge 干什么

事务提交后，旧版本（undo）不能立刻删——可能还有其他事务的 ReadView 在看它。需要确认所有 ReadView 都"过去了"，才能 purge 掉。

InnoDB 后台 purge 线程负责：

- 清理可以丢弃的 undo
- 清理被标记 deleted 的记录的真正物理删除（lazy delete）

### 7.2 purge 慢导致的问题

如果有长事务，旧版本 undo 大量积压：

- undo tablespace 膨胀
- 二级索引里"deleted"标记的记录不能真删 → 索引膨胀
- 老版本链很长 → MVCC 可见性判断要遍历更多版本，性能下降

### 7.3 监控

```sql
SELECT TRX_ID, TRX_STARTED FROM information_schema.INNODB_TRX
ORDER BY TRX_STARTED ASC LIMIT 5;

SHOW ENGINE INNODB STATUS\G
-- TRANSACTIONS section
-- History list length 22  ← 未 purge 的版本数；几千 / 几万就告警
```

History list length 持续增长 = purge 跟不上 / 长事务。

---

## 第八章：crash 恢复

### 8.1 InnoDB 启动时的恢复流程

```
1. 找最新 checkpoint LSN
2. 从该 LSN 开始扫 redo log
3. 对每条 redo：
   - 找对应页 → 检查 FIL_PAGE_LSN
   - 如果 redo LSN > FIL_PAGE_LSN → 重放
4. redo 重放完成 → 数据页一致
5. 检查 undo
   - 处于 PREPARE 状态的事务：检查 binlog
     - binlog 有 → COMMIT
     - binlog 无 → ROLLBACK
   - 未提交的事务（active）：ROLLBACK
6. 启动服务，开放连接
```

### 8.2 doublewrite buffer

机制：写脏页时不直接写到目标位置，而是先写到一个集中区域（doublewrite buffer，128 个连续页），fsync 后再写到实际位置。

为什么：InnoDB 16KB 页 vs OS 4KB 页 → "撕裂写"问题。如果一个 16KB 页写到一半挂了，磁盘上是半新半旧的损坏页，redo 重放也救不回来。doublewrite 提供完整副本作恢复源。

代价：每个脏页写两次（doublewrite 一次 + 原位置一次），但 doublewrite 是顺序大块写，开销小。

8.0 引入 `innodb_doublewrite=DETECT_AND_RECOVER` / `DETECT_ONLY` 等更细粒度。

### 8.3 ACID 完整性靠这些机制

| ACID | 机制 |
|---|---|
| Atomicity | undo 回滚 |
| Consistency | undo + redo + 应用约束 |
| Isolation | MVCC + 锁 |
| Durability | redo（崩溃恢复）+ binlog（复制）+ doublewrite |

---

## 第九章：实操：观察事务

### 9.1 查看活跃事务

```sql
SELECT trx_id, trx_state, trx_started, trx_query, trx_rows_locked, trx_rows_modified
FROM information_schema.innodb_trx;
```

看长事务、改动行数、当前在执行的 SQL。

### 9.2 看锁等待

```sql
-- 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- 老版本
SELECT * FROM information_schema.innodb_locks;
SELECT * FROM information_schema.innodb_lock_waits;
```

详细在 M04。

### 9.3 看历史链长度

```sql
SHOW ENGINE INNODB STATUS\G
-- TRANSACTIONS:
-- History list length 1234
```

正常 < 1000；几万 → 必有长事务，去 INNODB_TRX 找到杀掉。

### 9.4 强制结束长事务

```sql
-- 找到 trx_mysql_thread_id
SELECT trx_id, trx_mysql_thread_id, trx_started, trx_query
FROM information_schema.innodb_trx
WHERE TIMESTAMPDIFF(SECOND, trx_started, NOW()) > 60;

-- 杀掉
KILL <thread_id>;
```

---

## 第十章：常见陷阱

### 10.1 应用层"忘 COMMIT"

ORM 默认开 autocommit=ON 时，单条 SQL 自动一个事务。但某些 ORM（Java EE、Spring 老版本）显式开事务后，业务逻辑里 throw 异常没 commit/rollback → 事务一直挂。

排查方法：

```sql
SHOW PROCESSLIST;
-- Time 列大、State="Sleep" → 怀疑事务没结束
```

### 10.2 大事务

`BEGIN; UPDATE 千万行; COMMIT;` 的灾难：

- undo 暴涨
- redo 文件压力（甚至触发 redo wait）
- 锁持有时间长 → 别的事务全等
- binlog 大事件 → 复制延迟

**拆成小批 + 每批 commit**：

```python
for batch in range(0, total, 1000):
    cursor.execute(f"UPDATE t SET ... WHERE id BETWEEN {batch} AND {batch+999}")
    conn.commit()
```

### 10.3 RR 下"以为更新了实际没改"

```sql
-- 事务 A（RR）
BEGIN;
SELECT * FROM t WHERE id=1;     -- 看到旧值 'Alice'，决定改成 'Charlie'
                                  -- 此时事务 B 把 id=1 改成了 'Bob' 并提交
UPDATE t SET name='Charlie' WHERE id=1 AND name='Alice';
                                  -- 当前读看到 'Bob'，WHERE 不匹配，0 行受影响
                                  -- 但应用以为改成功了
```

修复：用 `SELECT ... FOR UPDATE` 或乐观锁版本号。

### 10.4 隔离级别选错

业务上读多写少 + 不在意"看到旧版本"→ 用 RC 比 RR 性能好（无 gap lock，并发好）。

业务上严格依赖"事务内多次读一致"→ 必须 RR。

### 10.5 在事务里不要做外部 RPC

```python
conn.begin()
cursor.execute("UPDATE ...")
external_api_call(...)    # 几百毫秒
conn.commit()
```

锁会被持有几百毫秒 + 长事务问题。**先 COMMIT 再 RPC，或者用消息队列异步化**。

---

## 第十一章：性能调优要点

### 11.1 关键参数

```ini
innodb_flush_log_at_trx_commit = 1   # 双 1 之一
sync_binlog = 1                      # 双 1 之二
innodb_redo_log_capacity = 8G        # 8.0.30+
innodb_log_buffer_size = 64M         # COMMIT 前缓存
innodb_buffer_pool_size = 物理内存 60-80%
innodb_undo_tablespaces = 2          # 默认 2 个，可加
innodb_undo_log_truncate = ON
innodb_purge_threads = 4             # 默认 4
```

### 11.2 监控关键指标

```sql
SHOW GLOBAL STATUS LIKE 'Innodb_rows_%';
-- Innodb_rows_inserted/updated/deleted/read

SHOW ENGINE INNODB STATUS\G
-- 关注：
-- LOG: log buffer hit ratio / write throughput
-- TRANSACTIONS: history list length
-- BUFFER POOL: hit ratio, dirty pages
-- ROW OPERATIONS: pending IO
```

### 11.3 redo / binlog 双 1 损失多大？

OLTP 场景一般：

- 双 0：极致 TPS（如 50k）
- 双 1：约 1/3 - 1/2 TPS（15-25k）
- 中间组合（如 sync_binlog=100）：折中

云 RDS 默认双 1。如果 TPS 卡在 fsync IOPS 上，先确认磁盘是否 SSD / NVMe，再考虑调整。

---

## 第十二章：与其他章节的连接

- **与 M02 索引**：MVCC 读靠版本链，**索引上有 deleted 标记的记录在 purge 前还占空间** → 索引膨胀
- **与 M04 锁**：MVCC 解决"读快照"问题，但写还是要锁；当前读 + gap lock 防幻读
- **与 M05 复制**：binlog 是从库的输入；row format binlog 比 statement 大但更准
- **与 M07 PS**：长事务、history list 都能在 PS 里看
- **与 M08 调优**：buffer pool / log file 大小直接影响事务性能

---

## 总结

InnoDB 事务 = undo + redo + MVCC + 锁。本章关键点：

1. **undo**：原子性 + MVCC 历史版本链；长事务的灾难
2. **redo**：崩溃恢复；WAL；LSN；fsync 策略（双 1）
3. **2PC**：redo prepare → binlog → redo commit；保证主从一致
4. **MVCC**：每事务一个 ReadView 标尺；RR 建一次 / RC 重建
5. **当前读 vs 快照读**：写一定是当前读；FOR UPDATE 也是
6. **purge**：异步清 undo + delete 标记；长事务阻碍
7. **doublewrite**：防撕裂写
8. **大事务危害**：undo + 锁 + binlog 三连击

---

## 练习题

1. 写一段会让 history list length 持续增长的 Python 代码，再写一段健康的对比。
2. 在 RR 隔离级别下，复现"半幻读"——快照读看不到、当前读看到。
3. 解释一次 UPDATE 涉及哪些日志写入，按顺序列出。
4. crash 恢复时如何处理"redo PREPARE + binlog 无对应 XID"的事务？
5. RC 比 RR 在哪些场景性能更好？哪些场景不能换？
6. `SELECT ... FOR UPDATE` 与普通 SELECT 区别？为什么前者会阻塞别的事务的写？
7. 把一个 1000 万行表的 status 字段批量更新，怎么避免大事务？给出两种方法。
8. 监控 history list length 的目的是什么？阈值定多少合理？
9. doublewrite 是为了解决什么问题？SSD 上还需要吗？
10. 业务报"事务超时"，从 `innodb_trx` / `data_locks` / processlist 的诊断步骤是什么？

---

> 🔁 反馈：开两个 mysql client 并发跑 SELECT/UPDATE，亲自感受 RR 与 RC 的差别
