# 精通 MVCC 与可见性：xmin/xmax、CLOG、Snapshot 与判定算法

> 关联章节：[精通堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md)、[精通 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)、[精通事务隔离与锁](./07-精通-事务隔离与锁.md)

---

## 引言：你真的理解 MVCC 吗

学过数据库的人都听过 MVCC（Multi-Version Concurrency Control，多版本并发控制）。MySQL 文档说它"用 undo log 实现 MVCC"，PostgreSQL 文档说它"是真正的 MVCC"。两个都叫 MVCC，差别在哪？为什么 PostgreSQL 需要 VACUUM、而 MySQL"看起来不需要"？为什么 PostgreSQL 32 位事务 ID 会"耗尽"，但 MySQL 不会有 wraparound 危机？

如果你能 30 秒内回答下面 5–7 个问题，可以跳过本章；否则强烈建议读完：

1. **PG 的 MVCC 和 MySQL InnoDB 的 MVCC 在存储层有什么本质区别？** 提示：旧版本放在哪里。
2. **为什么 PG 需要 VACUUM，而 MySQL 不需要类似的进程？** 提示：dead tuple 物理位置。
3. **一行被 INSERT、UPDATE、DELETE 时，`xmin` 和 `xmax` 各自怎么变化？** 提示：xmin 不变，xmax 才是关键。
4. **PG 的事务 ID 是 32 位，每秒钟几万 TPS 的库一两天就会用光，为什么生产中没炸？** 提示：FrozenXID。
5. **可见性判定算法（HeapTupleSatisfiesMVCC）怎么判断"我的事务能不能看到这个版本"？** 提示：(xmin 是否已提交且在 snapshot 之前) 且 (xmax 未提交或在 snapshot 之后)。
6. **为什么 RC 隔离级别的 PG 在 SELECT 后会"产生脏页"？** 提示：hint bit。
7. **PG 的 RC 和 RR（REPEATABLE READ）在快照管理上的核心差异？** 提示：snapshot 重建时机。
8. **subtransaction（SAVEPOINT）怎么影响 MVCC 可见性？** 提示：subxid 也要走 CLOG。
9. **什么时候一个 dead tuple 可以被回收？** 提示：所有活动事务的 xmin 都已经超过它的 xmax。

本章一次性把这些问题彻底讲透。读完后你应该能：

- 画出 PG 行版本的 Tuple Header 24 字节布局，逐字段解释含义
- 写出 INSERT/UPDATE/DELETE 时 xmin/xmax/t_ctid 的变化时序
- 手写一个简化版的 HeapTupleSatisfiesMVCC 伪代码
- 用 pageinspect 在生产环境观察一行的版本字段
- 解释为什么 PG 是"真 MVCC"，而 MySQL 只是"用 undo log 模拟 MVCC"
- 给出 wraparound 风险的预警监控 SQL
- 用 SAVEPOINT 调试一段含子事务的代码并预判可见性
- 给团队画一张图，说明 hint bit 副作用和 RC vs RR snapshot 差异

---

## 第 1 章：MVCC 的核心思想

### 1.1 一句话定义

> MVCC = **每次写入产生一个新版本**，不覆盖旧版本；**每个读事务看到的是"自己启动那一刻"的版本快照**。

这两句话推出 MVCC 的核心性质：

| 性质 | 含义 |
|---|---|
| 读不阻塞写 | 写新版本不影响旧版本，读旧版本不需要等写 |
| 写不阻塞读 | 同上反过来 |
| 写写互斥 | 同一行同时被两个事务写仍需排队（行锁），否则更新丢失 |
| 一致读 | 长查询看到的是"快照时刻"的全表，即使期间被并发改动 |

### 1.2 没有 MVCC 的世界：两阶段锁

传统 2PL（Two-Phase Locking）数据库（如早期 DB2）的策略：

- 读上 S 锁
- 写上 X 锁
- S/X 互斥

后果：长查询（比如统计报表）会**阻塞所有写**；高频写表的全表 COUNT(*) 在生产基本不可能跑。

### 1.3 MVCC 怎么解决

```
                        时间轴 →
事务 T1（写）:   BEGIN ──UPDATE row──────────────COMMIT
                        ↓ 写新版本 v2
                        ↓ 旧版本 v1 仍在
事务 T2（读）:        BEGIN ──SELECT row──→COMMIT
                        ↑ T2 启动时 T1 未提交
                        ↑ T2 只能看见 v1（旧版本）
```

T1 写新版本不动旧版本，T2 读旧版本不需要等 T1。**关键：两个版本必须同时存在于存储中。**

### 1.4 两种 MVCC 实现路线

MVCC 这个"理论"有两种工程实现：

| 路线 | 旧版本放在哪 | 代表数据库 |
|---|---|---|
| **多版本物理共存（真 MVCC）** | 与新版本一起放在表的数据页里 | PostgreSQL、Firebird、Interbase |
| **当前版本 + undo 回滚链（undo MVCC）** | 当前页只存最新版本；旧版本存在 undo log，按链回放 | MySQL InnoDB、Oracle |

两种路线的细节差异在第 2 章详细对比。先记住：**PG 选了路线 1，MySQL 选了路线 2**——这是后面所有差异的根源。

---

## 第 2 章：PG "真 MVCC" vs MySQL "undo 模拟 MVCC"

### 2.1 总对比表

| 维度 | PostgreSQL（多版本共存） | MySQL InnoDB（undo log） |
|---|---|---|
| **旧版本存储位置** | 与新版本同表数据页 | 当前页只存最新版本；旧版本存 undo log（回滚段） |
| **UPDATE 物理行为** | 在表中插入新版本 tuple，旧 tuple 标记 xmax | 原地更新当前行；把旧值写到 undo log |
| **读旧版本的代价** | 直接读旧 tuple（如果还没被 VACUUM 清掉） | 沿 undo 链回放旧值（链越长越慢） |
| **清理机制** | VACUUM 物理删除旧 tuple（不释放给 OS，只标记复用） | purge 线程清理过期 undo log（自动、不可见） |
| **长事务代价** | dead tuple 堆积、表膨胀、索引膨胀 | undo log 暴涨、磁盘吃紧、回滚段争用 |
| **回滚代价** | 几乎免费（只标记事务 abort，物理数据不动） | 重型（需要回放 undo log，逐行还原） |
| **索引上的可见性** | 索引指向多个版本的 CTID，**必须回表查可见性**（除非 visibility map 全可见） | 索引指向当前行的 PK，回表后再走 undo 链 |
| **wraparound 风险** | 有（xid 32 位，需要 FrozenXID 机制） | 无（trx_id 6 字节 = 48 位，且不参与 wraparound） |
| **DELETE 代价** | 只改 xmax，不动数据 | 标记删除位 + 写 undo |
| **写大事务的内存压力** | 主要在 WAL + 共享缓冲 | undo log 段内存 + redo log |
| **DDL 与 MVCC** | DDL 也走 MVCC（表定义有版本，可在事务内 CREATE/DROP/ALTER 后回滚） | DDL 隐式提交事务，无法回滚 |
| **autovacuum / purge** | 独立后台进程，可看见、可调参 | InnoDB purge 线程，黑盒、调参少 |

### 2.2 视觉对比：UPDATE 一行

**PG（真 MVCC）：**

```
更新前 page:
┌────────────────┐
│ tuple v1       │  xmin=100, xmax=0, ctid=(0,1)
│ id=1, name=A   │
└────────────────┘

执行 UPDATE id=1 SET name=B（事务 200）后 page:
┌────────────────┐
│ tuple v1       │  xmin=100, xmax=200, ctid=(0,2)  ← 指向 v2
│ id=1, name=A   │
├────────────────┤
│ tuple v2       │  xmin=200, xmax=0, ctid=(0,2)    ← 新版本
│ id=1, name=B   │
└────────────────┘

事务 200 commit 后：旧版本仍在，等待 VACUUM 清理
```

**MySQL InnoDB（undo MVCC）：**

```
更新前 page:
┌──────────────────────────┐
│ row v_current            │  trx_id=100, roll_ptr=NULL
│ id=1, name=A             │
└──────────────────────────┘

执行 UPDATE 后 page:
┌──────────────────────────┐
│ row v_current            │  trx_id=200, roll_ptr→undo1
│ id=1, name=B             │
└──────────────────────────┘

undo log:
┌──────────────────────────┐
│ undo1: name=A            │  trx_id=100, prev_ptr=NULL
└──────────────────────────┘

事务 200 commit 后：当前页只剩新值；undo1 等 purge 清理
```

### 2.3 各自的设计后果

**PG（多版本共存）的优势：**

- 回滚廉价（只改事务状态，不动数据）
- 长读不影响写、长写不影响读，并发性好
- DDL 也支持事务（极少数关系库具备）

**PG 的劣势：**

- 必须有 VACUUM（运维负担）
- 索引一定回表才能确认可见性（除非 index-only scan + visibility map）
- 表膨胀风险（dead tuple 不及时清理）
- wraparound 风险（32 位 xid）

**InnoDB（undo MVCC）的优势：**

- 当前页紧凑，扫描效率高
- 索引可以直接信任当前版本（二级索引带 trx_id 标记）
- 无 wraparound
- 没有 VACUUM 的运维概念

**InnoDB 的劣势：**

- 长事务把 undo 撑爆（"history list length" 暴涨）
- 回滚慢（要逐行回放）
- 不支持事务性 DDL
- 不能在同一页同时看到多版本，复杂查询并发能力弱于 PG

记住一句话：**两条路线没有谁绝对好——PG 把"清理"显式化，InnoDB 把"清理"隐式化。** 显式化让 PG 用户必须懂 VACUUM；隐式化让 InnoDB 用户在长事务下死得很惨却不知道怎么调。

---

## 第 3 章：行版本的核心字段（Tuple Header）

### 3.1 完整 Tuple Header 结构

回顾 [P02](./02-精通-堆表存储与-TOAST.md) 第三章：每行数据前面有 23–24 字节的 Tuple Header，源码在 `src/include/access/htup_details.h`：

```c
typedef struct HeapTupleHeaderData
{
    union {
        HeapTupleFields t_heap;       // 普通 tuple 用
        DatumTupleFields t_datum;     // datum tuple 用
    } t_choice;

    ItemPointerData t_ctid;           // 6 字节：当前 tuple 或更新链上的下一个版本
    uint16          t_infomask2;      // 2 字节：列数 + 一些位（HOT_UPDATED 等）
    uint16          t_infomask;       // 2 字节：状态位（含 xmin/xmax 状态、null bitmap 等）
    uint8           t_hoff;           // 1 字节：数据起始偏移
    bits8           t_bits[FLEXIBLE_ARRAY_MEMBER];  // null bitmap
} HeapTupleHeaderData;

typedef struct HeapTupleFields
{
    TransactionId   t_xmin;           // 4 字节：插入此版本的事务 ID
    TransactionId   t_xmax;           // 4 字节：删除/更新此版本的事务 ID（0 表示未删除）
    union {
        CommandId   t_cid;            // 4 字节：cmin 或 cmax（同事务内命令号）
        TransactionId   t_xvac;       // 4 字节：VACUUM FULL 使用
    } t_field3;
} HeapTupleFields;
```

总共 **24 字节**（不含 null bitmap）。MVCC 的全部秘密在 `t_xmin`、`t_xmax`、`t_cid`、`t_ctid`、`t_infomask` 这五个字段里。

### 3.2 五大核心字段详解

#### 3.2.1 t_xmin（插入者事务 ID）

- 这个 tuple **被哪个事务插入**
- 一旦写入永不修改（除非 VACUUM FREEZE 改成 FrozenXID = 2）
- 类型：`TransactionId`（32 位无符号整数）

#### 3.2.2 t_xmax（删除/更新者事务 ID）

- 这个 tuple **被哪个事务删除或更新**
- 0 表示"未被删除/更新"（仍是当前版本）
- UPDATE 时，旧版本的 xmax = 新事务 ID，新版本的 xmax = 0
- DELETE 时，被删 tuple 的 xmax = 删除事务 ID

#### 3.2.3 t_cid（cmin / cmax，命令 ID）

- 同事务内的"第几条命令"产生的此 tuple
- 用来支持**事务内自看见**（同事务内的 UPDATE 不能让后面 SELECT 看不到自己刚写的）
- 区分 cmin（插入它的命令）和 cmax（删除它的命令）；32 位，从 0 开始
- 注意是命令号（SQL 语句号），不是行号

#### 3.2.4 t_ctid（当前/新版本指针）

- 6 字节：`(block_number: 4B, offset: 2B)`
- 通常指向自己（这就是自己的位置）
- **如果被 UPDATE 过，指向新版本**——形成更新链（update chain）
- HOT update 时，旧版本 ctid 指向同页内的新版本，索引不动

```
更新链示例：
v1 ctid=(0,5) → v2 ctid=(1,3) → v3 ctid=(1,3, self)
旧                旧                   当前
```

#### 3.2.5 t_infomask（状态标志位，16 位）

`src/include/access/htup_details.h` 中定义了大量位常量。常见的：

| 位（hex） | 名称 | 含义 |
|---|---|---|
| `0x0100` | HEAP_XMIN_COMMITTED | xmin 已提交（hint bit） |
| `0x0200` | HEAP_XMIN_INVALID | xmin 已 abort（hint bit） |
| `0x0300` | HEAP_XMIN_FROZEN | xmin 已被 freeze（VACUUM FREEZE 后） |
| `0x0400` | HEAP_XMAX_COMMITTED | xmax 已提交（hint bit） |
| `0x0800` | HEAP_XMAX_INVALID | xmax 已 abort 或 xmax=0 |
| `0x1000` | HEAP_XMAX_IS_MULTI | xmax 是 multixact id（多事务共享行锁） |
| `0x2000` | HEAP_UPDATED | 此 tuple 是 UPDATE 产生的新版本 |
| `0x4000` | HEAP_MOVED_OFF | 旧 VACUUM FULL 使用 |
| `0x8000` | HEAP_MOVED_IN | 旧 VACUUM FULL 使用 |

`HEAP_XMIN_COMMITTED` / `HEAP_XMAX_COMMITTED` 就是著名的 **hint bit**（第 9 章详解）。

### 3.3 整体示意图

```
                  Tuple 24-byte Header
┌─────────────────────────────────────────────────────────┐
│ t_xmin (4)       │ t_xmax (4)       │ t_cid / t_xvac (4) │
├──────────────────┴──────────────────┴────────────────────┤
│ t_ctid (6)            │ t_infomask2 (2) │ t_infomask (2) │
├───────────────────────┴─────────────────┴────────────────┤
│ t_hoff (1) │ ... null bitmap ...                         │
├─────────────────────────────────────────────────────────┤
│ user data (列值)                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第 4 章：事务 ID（XID）与 wraparound

### 4.1 XID 的本质

PG 每次开启一个写事务（INSERT/UPDATE/DELETE/DDL）会从一个全局计数器分配一个 32 位无符号整数作为 `xid`：

```c
typedef uint32 TransactionId;

#define InvalidTransactionId        ((TransactionId) 0)
#define BootstrapTransactionId      ((TransactionId) 1)
#define FrozenTransactionId         ((TransactionId) 2)
#define FirstNormalTransactionId    ((TransactionId) 3)
```

特殊值：

- 0 = 无效 xid（用于 xmax = 0 表示"未删除"）
- 1 = bootstrap 事务（initdb 用）
- 2 = FrozenXID（被 VACUUM FREEZE 标记的"远古"事务，永远可见）
- 3 起 = 普通事务

只读事务不会分配 xid（PG 优化：只读不消耗 xid 空间）。

### 4.2 wraparound：32 位的悲剧

32 位 → 约 42 亿（2^32 ≈ 4.29 × 10⁹）。PG 在每次 commit 时 xid 自增 1，对于高频写库：

| TPS | 用尽 xid 时间 |
|---|---|
| 1000 | ~50 天 |
| 10000 | ~5 天 |
| 100000 | ~12 小时 |
| 1000000 | ~70 分钟 |

如果直接到顶，下一个 xid 就回到 0（环绕，wraparound），这时**所有已经写入的 tuple 都会被认为"是未来的事务写的，对当前不可见"**——整张表逻辑上消失，灾难。

### 4.3 PG 的解决方案：FrozenXID + 模运算可见性

PG 用**模 2³² 的环形比较**判断"我 vs 别人"，定义函数：

```c
// 简化伪代码
bool TransactionIdPrecedes(TransactionId id1, TransactionId id2)
{
    int32 diff = (int32)(id1 - id2);
    return (diff < 0);
}
```

把 32 位空间看作一个环，任意时刻"过去"是离当前 xid 最近的 2³¹ 个 xid，"未来"是另外的 2³¹ 个。问题：一个 xid 走过 2³¹ 后，原本"过去"的 xid 现在变成了"未来"。

**FrozenXID 机制：** VACUUM 会把"足够老"（年龄超过 `vacuum_freeze_min_age`，默认 5000 万）的 tuple 的 xmin 改成 `FrozenTransactionId = 2`，并打上 `HEAP_XMIN_FROZEN` 标记位。FrozenXID 在可见性判定中**永远可见**，从此免疫 wraparound。

只要 VACUUM 跟得上 xid 增长速度，wraparound 就不会发生。**跟不上时会怎样：**

1. age 超过 `autovacuum_freeze_max_age`（默认 2 亿），autovacuum 会强制启动 anti-wraparound vacuum（即使表很少修改）
2. age 超过 `vacuum_failsafe_age`（默认 16 亿），VACUUM 切换到 failsafe 模式，跳过索引清理只忙 freeze
3. age 超过 ~20 亿，PG 拒绝新事务，整库只读，强制单用户模式 VACUUM
4. 如果还是不 VACUUM，到达 2³¹ 后**数据可见性翻车**

详细 wraparound 处理见 [P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)。本章只需理解：xmin/xmax 是 32 位、需要 freeze、有 wraparound 危机。

### 4.4 监控 xid 年龄

```sql
-- 数据库级
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2147483647 - age(datfrozenxid) AS xid_remain
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- 表级
SELECT schemaname || '.' || relname AS tbl,
       age(relfrozenxid) AS xid_age,
       n_dead_tup,
       last_autovacuum
FROM pg_stat_user_tables
JOIN pg_class ON pg_class.oid = pg_stat_user_tables.relid
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

`xid_age > 1.5e9` 就该立刻人工介入。

### 4.5 64 位 XID 的社区进展

业界已多次推动 64 位 XID（特别是 EnterpriseDB / Postgres Pro 的分支已经实现），主线一直在评审，2026 年仍未合入。原因是侵入面太大（每个 tuple 24 → 32 字节，所有索引/WAL/复制协议都要改）。短期可见的几年内仍是 32 位 + freeze 这条路。

---

## 第 5 章：CLOG（pg_xact）：事务状态目录

### 5.1 为什么需要 CLOG

Tuple Header 里只存了"哪个事务写的"（xmin/xmax），没存"那个事务最终是 commit 还是 abort"。判可见性必须知道事务状态。把"状态"塞进 tuple 不现实（事务还在跑时还没结果），所以 PG 用一个独立结构 **CLOG（commit log，PG 10+ 改名为 pg_xact）** 记录每个 xid 的最终状态。

### 5.2 4 种事务状态（每 xid 2 位）

```c
#define TRANSACTION_STATUS_IN_PROGRESS  0x00   // 进行中
#define TRANSACTION_STATUS_COMMITTED    0x01   // 已提交
#define TRANSACTION_STATUS_ABORTED      0x02   // 已回滚
#define TRANSACTION_STATUS_SUB_COMMITTED 0x03  // 子事务已提交（最终看父事务）
```

每个 xid 占 2 位。1 个 CLOG 文件 256 KB，能存 `256 * 1024 * 4 = 1048576 个 xid`（每字节 4 个 xid）。

文件路径：

```bash
$PGDATA/pg_xact/
├── 0000        # xid 0 ~ 1048575
├── 0001        # xid 1048576 ~ ...
├── 0002
└── ...
```

xid → 文件偏移：

```
file_no  = xid / 1048576
byte_off = (xid / 4) % 262144
bit_off  = (xid % 4) * 2
```

### 5.3 CLOG 缓存

CLOG 文件存磁盘，但实际访问极频繁（每次可见性判定可能查多次），PG 用 **SLRU buffer** 缓存：

- `transaction_buffers`（PG 17+ 引入，默认 0 = 按 `shared_buffers/512` 自动调，下限 16 页/128 KB、上限 1024 页）
- LRU 替换
- 多 backend 用 lwlock 控制并发

PG 17 起对 SLRU 做了大改造（partitioned），并发能力大幅提升。早期版本 CLOG contention 是热点写入下的常见性能问题。

### 5.4 异步 commit 与 CLOG 写时机

事务 commit 时：

1. WAL 写 commit 记录
2. WAL flush 到磁盘（`fsync`，可被 `synchronous_commit=off` 跳过）
3. CLOG 标记此 xid 为 COMMITTED
4. 释放锁

CLOG 本身**不需要立刻 flush 到磁盘**——崩溃恢复时从 WAL 重放 commit 记录就能重建 CLOG 状态。因此 CLOG 写代价很低。

---

## 第 6 章：Snapshot 结构与构造

### 6.1 什么是 Snapshot

Snapshot 是"一个事务/查询启动时看到的事务全景"。包含三块信息：

| 字段 | 含义 |
|---|---|
| `xmin`（snapshot 的） | 当时活跃事务里**最小**的 xid（小于 snap.xmin 的事务**一定**已完成） |
| `xmax`（snapshot 的） | 下一个将被分配的 xid（≥ snap.xmax 的事务**一定**还没开始） |
| `xip[]` | 在 [xmin, xmax) 区间内**仍活跃**的 xid 数组 |

**注意：** snapshot.xmin/xmax 与 tuple.xmin/xmax 含义不同，别混。下文用 `snap.xmin` / `tup.xmin` 区分。

### 6.2 构造 Snapshot 的过程

PG 在事务开启的第一条 SQL（RC）或显式 BEGIN（RR/SR）时构造 snapshot：

```c
// 简化伪代码
Snapshot GetSnapshotData(Snapshot snap)
{
    LWLockAcquire(ProcArrayLock, LW_SHARED);

    snap.xmax = ShmemVariableCache->nextXid;  // 下一个 xid
    snap.xmin = snap.xmax;
    snap.xcnt = 0;

    for each PROC in ProcArray:
        xid = PROC.xid;
        if (xid is valid && xid < snap.xmax) {
            if (xid < snap.xmin)
                snap.xmin = xid;
            snap.xip[snap.xcnt++] = xid;
        }

    LWLockRelease(ProcArrayLock);
    return snap;
}
```

代价：扫一遍 `MaxBackends` 大小的 ProcArray，持锁时间通常 < 几微秒。高连接数（上千）时这是热点（PG 14+ 引入 ProcArray 改造缓解）。

### 6.3 直观例子

假设事务 A 启动时：

- 活跃事务：xid=100, 105, 108（共 3 个）
- 已提交：xid=99, 101, 102, 103, 104, 106, 107（不在 ProcArray）
- 下一个待分配：xid=109

A 拿到的 snapshot：

```
snap.xmin = 100   // 活跃中最小的
snap.xmax = 109   // 下一个将分配的
snap.xip  = [100, 105, 108]
```

可见性判定规则（不严格，先建立直觉）：

- `tup.xmin < 100`：**必定**已提交（在快照之前），看其状态
- `tup.xmin >= 109`：**必定**是快照后的事务，不可见
- `100 ≤ tup.xmin < 109`：可能在 xip 里活跃，可能已结束，查 CLOG
- `tup.xmin` 在 xip 里 → 不可见（事务还没 commit）
- `tup.xmin` 在区间但不在 xip → 已结束，查 CLOG 看是 commit 还是 abort

### 6.4 查看当前 snapshot

PG 13+：

```sql
SELECT pg_current_snapshot();
-- 结果形如 "100:109:100,105,108"
-- 含义：xmin=100, xmax=109, xip=[100,105,108]
```

PG 12 及以前用 `txid_current_snapshot()`（语义相同）。

---

## 第 7 章：可见性判定算法

### 7.1 核心函数 HeapTupleSatisfiesMVCC

源码：`src/backend/access/heap/heapam_visibility.c`。简化伪代码（保留所有关键分支，但删掉了 multixact、frozen、subxact 的部分细节，本章末尾给完整版）：

```c
// 输入：要看的 tuple、当前事务的 snapshot
// 输出：true=可见 / false=不可见
bool HeapTupleSatisfiesMVCC(HeapTuple tuple, Snapshot snapshot)
{
    TransactionId tup_xmin = HeapTupleHeaderGetXmin(tuple);
    TransactionId tup_xmax = HeapTupleHeaderGetXmax(tuple);
    uint16 infomask = tuple->t_data->t_infomask;

    /* === 第一阶段：判 xmin（这一版本"出生"是否已对我可见） === */

    if (!(infomask & HEAP_XMIN_COMMITTED)) {           // 没有 hint bit
        if (infomask & HEAP_XMIN_INVALID)              // xmin abort 过
            return false;                              //   → 跳过

        if (tup_xmin == MyXid) {                        // 是我自己写的
            if (HeapTupleHeaderGetCmin(tuple) >= snapshot->curcid)
                return false;                          //   未来命令写的，不可见
            // ...省略 cmin 对比逻辑
            goto check_xmax;                           //   通过 xmin，看 xmax
        }

        if (XidInMVCCSnapshot(tup_xmin, snapshot))     // 在 xip[] 里仍活跃
            return false;                              //   → 不可见

        if (TransactionIdDidCommit(tup_xmin)) {         // CLOG: 已 commit
            SetHintBits(tuple, HEAP_XMIN_COMMITTED);   //   写 hint
        } else if (TransactionIdDidAbort(tup_xmin)) {   // CLOG: 已 abort
            SetHintBits(tuple, HEAP_XMIN_INVALID);
            return false;
        } else {
            return false;                              // 进行中但不在我快照 → 见不到
        }
    }
    // 走到这里：tup_xmin 已 commit 且对我可见

    /* === 第二阶段：判 xmax（这一版本"死亡"是否已对我发生） === */

check_xmax:
    if (infomask & HEAP_XMAX_INVALID)                  // xmax 无效（包括=0）
        return true;                                   //   → 可见

    if (!(infomask & HEAP_XMAX_COMMITTED)) {
        if (tup_xmax == MyXid) {
            if (HeapTupleHeaderGetCmax(tuple) >= snapshot->curcid)
                return true;                           //   我后面才删，还可见
            return false;                              //   我之前删了，不可见
        }

        if (XidInMVCCSnapshot(tup_xmax, snapshot))     // 删除者仍活跃
            return true;                               //   → 我看不到它的删除

        if (TransactionIdDidCommit(tup_xmax)) {
            SetHintBits(tuple, HEAP_XMAX_COMMITTED);
        } else {
            SetHintBits(tuple, HEAP_XMAX_INVALID);
            return true;                               //   xmax abort, 此版本仍活
        }
    }

    // 走到这里：tup_xmax 已 commit
    return false;                                       // 被删除了，不可见
}
```

### 7.2 文字版判定流程

整理成口诀：

1. **xmin 必须对我"已经发生"**（已 commit 且在 snap.xmin 之前，或不在 xip[]）
2. **xmax 必须对我"还没发生"**（0、abort、在 xip[] 里活跃、或在 snap.xmax 之后）
3. 我自己写的版本看 cmin/cmax（命令号）

### 7.3 流程图

```
                  HeapTupleSatisfiesMVCC
                          │
            ┌─────────────┴─────────────┐
            ▼                            ▼
       看 tup.xmin                  看 tup.xmax
            │                            │
   ┌────────┼────────┐         ┌─────────┼─────────┐
   ▼        ▼        ▼         ▼         ▼         ▼
 我自己   已commit  abort   xmax=0   abort 或   commit 且
 写的    且对我             或我自己 在我快照   在我快照
 (cmin)   可见     ✗ 不可见  写的    后活跃     之前
   │       │                  ✓         ✓         ✗
   │       └─→ 进入 xmax 阶段                    不可见
   ✓ 通过
```

### 7.4 一个具体例子

假设：

- 我的 snapshot：xmin=100, xmax=109, xip=[100,105,108], MyXid=110
- 看一行 tuple: xmin=99, xmax=105
- CLOG: 99=COMMITTED, 105=IN_PROGRESS

判定：

1. xmin=99 < snap.xmin=100，跳过 xip 检查；CLOG 显示 COMMITTED → xmin 通过
2. xmax=105 在 xip[] 中（活跃）→ 删除尚未对我发生 → 可见

结论：**可见**。即使被 105 删了，从我的视角它仍然存活。

---

## 第 8 章：INSERT/UPDATE/DELETE 时 xmin/xmax 变化

### 8.1 INSERT

```sql
-- 假设当前 xid=200
INSERT INTO t VALUES (1, 'A');
```

| 阶段 | 表内状态 |
|---|---|
| 写入后 | tup{ id=1, name=A, xmin=200, xmax=0, ctid=self } |
| commit 200 | CLOG[200]=COMMITTED；tuple 不变（hint bit 由后续 SELECT 写入） |

ASCII 图：

```
[t row1: xmin=200 xmax=0  ctid=(0,1)]
```

### 8.2 UPDATE

```sql
-- 假设当前 xid=300
UPDATE t SET name='B' WHERE id=1;
```

| 阶段 | 表内状态 |
|---|---|
| UPDATE 中 | 旧 tup{xmin=200, xmax=300, ctid=(0,2)}  + 新 tup{xmin=300, xmax=0, ctid=(0,2)} |
| commit 300 | CLOG[300]=COMMITTED；两条 tuple 都在 |

ASCII 图：

```
更新前:
[t row1: xmin=200 xmax=0  ctid=(0,1)] ← 索引指向这里

更新中（事务 300 写新版本）:
[t row1: xmin=200 xmax=300 ctid=(0,2)] ← 旧版，xmax 指明谁更新
[t row2: xmin=300 xmax=0   ctid=(0,2)] ← 新版

commit 后（VACUUM 之前）:
[t row1: xmin=200 xmax=300 ctid=(0,2)] dead 候选，仍在磁盘
[t row2: xmin=300 xmax=0   ctid=(0,2)] 当前版本

之后某个 SELECT 触发 hint bit:
[t row1: xmin=200 xmax=300 ctid=(0,2)] +HEAP_XMIN_COMMITTED +HEAP_XMAX_COMMITTED
[t row2: xmin=300 xmax=0   ctid=(0,2)] +HEAP_XMIN_COMMITTED
```

### 8.3 DELETE

```sql
-- 假设当前 xid=400
DELETE FROM t WHERE id=1;
```

| 阶段 | 表内状态 |
|---|---|
| DELETE 中 | tup{xmin=300, xmax=400, ctid=self} |
| commit 400 | tup 不动；后续可见性判 = 不可见 |

ASCII 图：

```
删除前:
[t row2: xmin=300 xmax=0   ctid=(0,2)]

DELETE 后（事务 400 未提交）:
[t row2: xmin=300 xmax=400 ctid=(0,2)]  ← 只改 xmax，物理数据原地

commit 400:
[t row2: xmin=300 xmax=400 ctid=(0,2)]  ← 数据仍在！等 VACUUM
```

DELETE 在 PG 是 **最便宜的 DML 之一**——只改 4 字节 xmax。代价推迟到 VACUUM。

### 8.4 ROLLBACK 的行为

```sql
BEGIN;            -- xid=500
INSERT INTO t VALUES (2, 'X');
ROLLBACK;
```

| 阶段 | 状态 |
|---|---|
| INSERT 后 | tup{xmin=500, xmax=0} 已写入数据页 |
| ROLLBACK | CLOG[500]=ABORTED；**tuple 不动** |
| 后续 SELECT | 看到 tup.xmin=500 → 查 CLOG → ABORTED → 跳过 + 写 HEAP_XMIN_INVALID |

ROLLBACK 不撤销物理写入——这是 PG MVCC 的本质特征。代价转嫁到 VACUUM。

### 8.5 HOT UPDATE（同页内更新）

如果 UPDATE 的列**不在任何索引上**，且新版本能放进同一页 → HOT update：

- 旧版本 ctid 指向同页内新版本
- 旧版本 `HEAP_HOT_UPDATED`，新版本 `HEAP_ONLY_TUPLE`
- **索引保持不变**——索引指向旧版本 CTID，沿 ctid 链找到新版本

HOT 的目的是减少索引膨胀。详见 [P02 第 6 章](./02-精通-堆表存储与-TOAST.md)。在 MVCC 视角，HOT update 的 xmin/xmax 变化与普通 UPDATE 完全一致，只是新版本和旧版本在同页。

### 8.6 多次更新形成更新链

```
ctid 链：
(0,1) ─→ (0,5) ─→ (1,3) ─→ (1,3 self)
v1       v2       v3       v3
xmin=100 xmin=200 xmin=300 (current)
xmax=200 xmax=300 xmax=0
```

可见性判定从索引指向的版本开始，沿 ctid 链向后找直到找到对自己可见的版本（或链终止）。

---

## 第 9 章：hint bit 优化与副作用

### 9.1 为什么要 hint bit

每次可见性判定都查 CLOG → 性能爆炸。优化：第一次判定后把结果记在 tuple 自己的 infomask 上：

- `HEAP_XMIN_COMMITTED` / `HEAP_XMIN_INVALID`
- `HEAP_XMAX_COMMITTED` / `HEAP_XMAX_INVALID`

下次判定看到 hint bit 直接信任，不查 CLOG。

### 9.2 hint bit 是谁设置的

**任何**读到这条 tuple 并完成可见性判定的 backend 都会设置。这意味着：

- 一个**只读** SELECT 也会**修改数据页**（设置 hint bit）
- 这一页变成"脏"页，下次 checkpoint 要写盘
- 一个大表第一次大规模 SELECT 后会产生大量"看似无来由"的 IO

### 9.3 副作用清单

| 副作用 | 现象 | 应对 |
|---|---|---|
| SELECT 产生脏页 | 备库流复制延迟、checkpoint 压力 | 接受，或显式跑一次 VACUUM 让 autovacuum 把 hint bit 全部设上 |
| 频繁的 buffer dirty | 缓冲池脏页比例高 | 调大 `bgwriter_lru_maxpages` |
| 物理复制的全页写 | `full_page_writes=on` 时 hint bit 也会触发 FPI | 接受（数据安全大于性能） |
| 不是 WAL 记录 | hint bit 本身不写 WAL（不影响逻辑回放） | — |

注意最后一点：**hint bit 是优化提示，不写 WAL**。崩溃恢复时如果 hint bit 丢了，下次读到时重新查 CLOG 设上即可。所以掉了不影响正确性。

### 9.4 freeze：hint bit 的终极版

VACUUM FREEZE 把 xmin 改成 FrozenXID(2) 并打 `HEAP_XMIN_FROZEN` → 永久免 CLOG、永久可见，wraparound 也奈何不了。

---

## 第 10 章：RC vs RR 隔离级别下 snapshot 的差异

### 10.1 PG 的 4 个隔离级别

PG 标准 4 级，但实际只有 3 个独立行为（READ UNCOMMITTED 在 PG 等价于 READ COMMITTED）：

| 隔离级别 | snapshot 重建时机 | 现象 |
|---|---|---|
| READ UNCOMMITTED | 同 RC（PG 不支持脏读） | — |
| **READ COMMITTED (RC)**（默认） | **每条 SQL 语句开始时重新构造** | 同事务连续读看到的版本可能变 |
| **REPEATABLE READ (RR)** | **事务的第一条 SQL 语句时构造一次** | 同事务任意 SELECT 看到的版本不变 |
| **SERIALIZABLE (SR)** | 同 RR，但加 SSI 检测 | 串行化失败时 ERROR |

详见 [P07 事务隔离与锁](./07-精通-事务隔离与锁.md)。本章只看 snapshot 差异。

### 10.2 RC vs RR 时序对比

**场景：** T1 改了 id=1 的 name 从 A→B 并 commit；与此同时 T2 在跑两个 SELECT。

```
时间轴 →
T1:  BEGIN ─ UPDATE name='B' ─ COMMIT
T2:  BEGIN ─ SELECT name ─────── SELECT name
                ↑                    ↑
                看到什么？           看到什么？
```

**RC：**

- T2 第 1 次 SELECT：在 T1 commit 前，看到 A
- T2 第 2 次 SELECT：T1 已 commit，重新构造快照，看到 B
- 不可重复读

**RR：**

- T2 第 1 次 SELECT：构造快照（xmax 在 T1 commit 之前），看到 A
- T2 第 2 次 SELECT：复用同一快照，仍看到 A
- 可重复读

### 10.3 SR（SSI）

PG 的 SSI（Serializable Snapshot Isolation）是 RR + 谓词锁 + 读写依赖图，在 commit 时检测是否破坏可串行化。检测到则 ERROR `40001 serialization_failure`。详见 [P07](./07-精通-事务隔离与锁.md)。

### 10.4 写冲突在 RC vs RR 下的不同

```sql
-- 两个并发事务都执行：
UPDATE accounts SET balance = balance - 100 WHERE id=1;
```

**RC：**

- T1 拿行锁，更新
- T2 等行锁；T1 commit 后，T2 拿锁
- T2 **重新读 row1 的当前版本**（关键！RC 写时会"重新可见性"），看到 balance=900（已扣 100）
- T2 计算 balance=900-100=800 写入
- 最终结果 800，**正确**

**RR：**

- T1 拿行锁，更新
- T2 等行锁；T1 commit 后，T2 拿锁
- T2 **检查行的当前 xmax**——发现这行被它快照后的事务改了
- T2 报错 `40001 could not serialize access due to concurrent update`
- 应用必须重试

记住：**RR 下的写冲突是显式失败的**。这是 PG 与 InnoDB 在 RR 下的核心差异之一（InnoDB 的 RR 会自动用最新版本继续，类似 PG 的 RC 写冲突处理）。

---

## 第 11 章：subtransaction（SAVEPOINT）对 MVCC 的影响

### 11.1 子事务 = SAVEPOINT

```sql
BEGIN;                    -- 父事务 xid=600
INSERT INTO t VALUES (10, 'A');
SAVEPOINT s1;             -- 子事务 1 xid=601
INSERT INTO t VALUES (11, 'B');
SAVEPOINT s2;             -- 子事务 2 xid=602
INSERT INTO t VALUES (12, 'C');
ROLLBACK TO s2;           -- 子事务 2 abort, 子事务 1 还活着
COMMIT;                   -- 父事务 600 commit
```

每个 SAVEPOINT 在 PG 内部都分配一个新的 xid（"子事务 ID" / subxid）。

### 11.2 子事务在 CLOG 中的状态

- 子事务 commit 时：标记为 `SUB_COMMITTED`（最终看父事务）
- 子事务 abort 时：标记为 `ABORTED`
- 父事务 commit 时：所有 `SUB_COMMITTED` 的子事务被认为 COMMITTED；ABORTED 的子事务不受影响

子事务结构存在 `pg_subtrans/` 目录（也是 SLRU）。

### 11.3 对可见性判定的影响

`HeapTupleSatisfiesMVCC` 看 xmin 时如果是子事务 ID：

1. 查 `pg_subtrans` 找到顶层事务 ID
2. 用顶层事务 ID 去 CLOG 查状态
3. 但如果子事务自己 abort 了，整条 tuple 不可见（即使父事务 commit）

### 11.4 性能陷阱：子事务数量

每个 SAVEPOINT 都消耗一个 xid + 一个 `pg_subtrans` 槽位。snapshot 还要记录所有活跃 subxid（除非超过 `PGPROC_MAX_CACHED_SUBXIDS=64` 后会 overflow 到 lookup 表，导致每次可见性判定都查 pg_subtrans → 严重退化）。

**生产建议：** 单事务内 SAVEPOINT 数量 < 64。ORM（特别是 Hibernate 的 `@Transactional` 嵌套）容易触发 subxact overflow。Django、Rails 在长循环里频繁 SAVEPOINT/RELEASE 也是常见性能陷阱。

### 11.5 一个调试例子

```sql
BEGIN;                            -- xid=700
INSERT INTO t VALUES (20, 'X');   -- t.xmin=700
SAVEPOINT s1;                     -- subxid=701
UPDATE t SET name='Y' WHERE id=20;  -- 新版本 t.xmin=701, 旧 t.xmax=701
ROLLBACK TO s1;                   -- subxid=701 abort
SELECT * FROM t WHERE id=20;
-- 看到的是 INSERT 那条（xmin=700, xmax=0）
-- UPDATE 产生的新版本 xmin=701 已 abort 不可见
-- INSERT 版本的 xmax=701 也是 abort，所以仍然存活
COMMIT;
```

---

## 第 12 章：dead tuple 与回收时机

### 12.1 什么是 dead tuple

一个 tuple 是 dead 当且仅当：**当前所有活动事务都对它不可见**。

形式化：dead ⇔ `tup.xmax != 0` 且 `tup.xmax 已 commit` 且 `tup.xmax < 全局最小 snap.xmin`。

最后一条是关键：必须确保**没有任何当前/未来的事务**还能看到旧版本，才能物理删除。

### 12.2 全局 horizon（OldestXmin）

PG 维护一个全局变量 `OldestXmin`（所有 backend 的 snap.xmin 的最小值，加上长查询、replication slot 的影响）。VACUUM 只能清理 xmax < OldestXmin 的 tuple。

**注意：** 一个长事务（比如一个跑了 1 小时的 SELECT）会"卡住"OldestXmin，所有比它新的更新产生的 dead tuple 都无法回收——这是 PG 表膨胀的最常见根因。

### 12.3 horizon 影响因素清单

让 OldestXmin 卡住的常见原因：

| 原因 | 现象 |
|---|---|
| 长事务（含未关连接的 idle in transaction） | 全表 dead tuple 无法清理 |
| 物理复制的 hot_standby_feedback=on | 备库长查询反压主库 horizon |
| 逻辑复制的 replication slot 未消费 | catalog horizon 卡住 |
| 未释放的快照（prepared transaction、cursor with hold） | 同上 |

监控：

```sql
-- 查最长事务
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
ORDER BY xact_start
LIMIT 10;

-- 查 horizon
SELECT
  pg_snapshot_xmin(pg_current_snapshot()) AS snap_xmin,
  txid_current() AS current_xid;

-- 查复制 slot
SELECT slot_name, restart_lsn, confirmed_flush_lsn, active
FROM pg_replication_slots;
```

### 12.4 VACUUM 做的事

简单总结（详细见 [P08](./08-精通-VACUUM-与表膨胀.md)）：

1. 扫表，找到 dead tuple
2. 删除指向 dead tuple 的索引项
3. 把 dead tuple 标记为 unused（item id 释放）
4. 标记数据页为可重用空间
5. 满足条件的 tuple 做 freeze（xmin → FrozenXID）
6. 更新 visibility map / FSM
7. **VACUUM 不释放磁盘给 OS**（只是回收页内空间）；要彻底回收用 `VACUUM FULL`（重写整张表，需 ACCESS EXCLUSIVE 锁）

---

## 第 13 章：实操章节——观察 MVCC

### 13.1 准备实验环境

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pgstattuple;

CREATE TABLE mvcc_demo (
    id int PRIMARY KEY,
    name text
);
INSERT INTO mvcc_demo VALUES (1, 'A'), (2, 'B'), (3, 'C');
```

### 13.2 看 tuple header

```sql
SELECT
    lp,
    t_xmin,
    t_xmax,
    t_field3 AS t_cid_or_xvac,
    t_ctid,
    to_hex(t_infomask) AS infomask_hex,
    to_hex(t_infomask2) AS infomask2_hex
FROM heap_page_items(get_raw_page('mvcc_demo', 0));
```

输出（示例）：

```
 lp | t_xmin | t_xmax | t_cid_or_xvac | t_ctid | infomask_hex | infomask2_hex
----+--------+--------+---------------+--------+--------------+----------------
  1 |    500 |      0 |             0 | (0,1)  | 800          | 2
  2 |    500 |      0 |             0 | (0,2)  | 800          | 2
  3 |    500 |      0 |             0 | (0,3)  | 800          | 2
```

- `t_xmin=500`：插入事务 ID
- `t_xmax=0`：未删除
- `t_ctid=self`：当前版本

### 13.3 观察 UPDATE 后的变化

```sql
UPDATE mvcc_demo SET name='AA' WHERE id=1;

SELECT lp, t_xmin, t_xmax, t_ctid, to_hex(t_infomask) AS imask
FROM heap_page_items(get_raw_page('mvcc_demo', 0));
```

```
 lp | t_xmin | t_xmax | t_ctid | imask
----+--------+--------+--------+--------
  1 |    500 |    501 | (0,4)  | 100  ← 旧版，xmax=501，ctid 指新版
  2 |    500 |      0 | (0,2)  | 800
  3 |    500 |      0 | (0,3)  | 800
  4 |    501 |      0 | (0,4)  | 8000 ← 新版（HEAP_UPDATED）
```

### 13.4 看当前事务 ID 和 snapshot

```sql
BEGIN;
SELECT txid_current();          -- 当前事务的 xid
SELECT pg_current_snapshot();   -- 当前事务的 snapshot

-- 模拟另一个事务
-- 在另一个会话 BEGIN; INSERT; (不 commit)

-- 回到本会话
SELECT pg_current_snapshot();   -- 已包含对方的 xid
ROLLBACK;
```

### 13.5 看 dead tuple 数量

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       last_vacuum,
       last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'mvcc_demo';

-- 字节级膨胀（需 pgstattuple）
SELECT * FROM pgstattuple('mvcc_demo');
```

### 13.6 模拟长事务卡住 horizon

```sql
-- 会话 A
BEGIN;
SELECT * FROM mvcc_demo;   -- 拿到 snapshot
-- 不 commit

-- 会话 B
DELETE FROM mvcc_demo;
VACUUM mvcc_demo;
-- pg_stat_user_tables.n_dead_tup 仍然非零，因为 A 还能看到旧版本

-- 会话 A
COMMIT;

-- 现在 VACUUM 才能真正清理
VACUUM mvcc_demo;
```

---

## 第 14 章：生产实践

### 14.1 高频写表的注意事项

| 场景 | 影响 | 应对 |
|---|---|---|
| OLTP 高频 UPDATE 同行 | 同一行短时间内 dead tuple 暴涨 | FILLFACTOR 调到 70-80 触发 HOT update；autovacuum 阈值调低 |
| 批量 UPDATE 大表 | 一次产生海量 dead | 分批 + 中间穿插 VACUUM；或使用 `pg_repack` |
| 长查询 + 高写入 | horizon 卡住 → 全库膨胀 | 限制 SELECT 超时（`statement_timeout`）；监控 long transactions |
| 高并发短事务 | xid 消耗快 → 频繁 freeze | autovacuum_freeze_max_age 调大；增加 autovacuum worker |
| 大事务一次提交 | xid 占用单一 | 拆批 |

### 14.2 应用层最佳实践

1. **永远不要"开了 BEGIN 忘 COMMIT"**——idle in transaction 卡 horizon。设置：
   ```ini
   idle_in_transaction_session_timeout = '5min'
   ```
2. **ORM 别滥用 SAVEPOINT**——配置嵌套事务为"扁平模式"（Django: `ATOMIC_REQUESTS=False`；Spring: 默认 `PROPAGATION_REQUIRED` 而非 NESTED）。
3. **报表查询走只读副本**——主库 OldestXmin 不会被报表卡住。
4. **批量任务用游标分页**而非 OFFSET，避免长 SELECT。
5. **大表 ANALYZE 之外定期 `VACUUM (FREEZE)`** 推进 relfrozenxid。

### 14.3 监控指标清单

| 指标 | 来源 | 告警阈值 |
|---|---|---|
| 最长事务时长 | `pg_stat_activity.xact_start` | > 30 min |
| 数据库 xid age | `pg_database.datfrozenxid` + `age()` | > 1.5e9 |
| 表 dead tuple 比例 | `n_dead_tup / n_live_tup` | > 20% |
| autovacuum lag | `last_autovacuum vs now()` | > 24h 且 n_dead_tup 多 |
| replication slot lag | `pg_replication_slots.confirmed_flush_lsn` | 看场景 |
| subxact overflow | `pg_stat_activity.backend_xid` 配合 perf 看 | 极端时主备复制慢 |

---

## 第 15 章：陷阱清单

1. **以为 SELECT 是"只读不写"**——hint bit 让 SELECT 也产生脏页（IO+WAL FPI）。
2. **以为 ROLLBACK 撤销了物理写入**——并没有，dead tuple 等 VACUUM 清。
3. **以为 VACUUM 释放磁盘给 OS**——并没有，只回收页内空间。要真物理回收用 VACUUM FULL（锁表）或 pg_repack。
4. **长事务 + 主库写入 = 全库膨胀**——常见根因。
5. **prepared transactions** 永久卡 horizon（除非显式 COMMIT PREPARED / ROLLBACK PREPARED）。检查 `pg_prepared_xacts`。
6. **hot_standby_feedback** 主备链——备库长查询反压主库 horizon，主库 dead tuple 暴涨。
7. **subxact overflow** > 64 时全库可见性判定变慢。看 `pg_stat_activity.subxact_count`（PG 13+）。
8. **大版本升级跨 wraparound** 风险——pg_upgrade 前必须完成 freeze。
9. **物理复制的备库 vacuum 不主动**——主库的 horizon 也要考虑备库。
10. **以为 RR 等同 InnoDB 的 RR**——PG RR 写冲突直接报错 40001。
11. **以为 PG 没有"幻读"**——RR 下没有"经典幻读"，但更新冲突仍会失败。SSI 才能完全防御串行化异常。
12. **以为 wraparound 是"未来很远的事"**——TPS 1 万的库 5 天就到顶，必须监控 datfrozenxid。
13. **以为 hint bit 写盘是 bug**——这是设计，可以通过显式 VACUUM 把全表 hint bit 提前刷掉。
14. **以为 `n_dead_tup` 准确**——pgstattuple 才精确（但代价是全表扫描）。
15. **以为 VACUUM 是 autovacuum 的事**——大批量删除后建议手动 VACUUM，autovacuum 触发阈值默认是 `n_dead > 0.2 * n_live + 50`，小表反应快、大表反应慢。

---

## 第 16 章：2026 现状

| 主题 | 2026 状态 |
|---|---|
| **64 位 XID** | 社区持续讨论，主线未合入；EnterpriseDB / Postgres Pro 分支已 GA；预计 5+ 年内合入主线 |
| **FrozenXID 自动化** | PG 17+ autovacuum 在 anti-wraparound 时优先级提升 + failsafe 模式（PG 14+）已稳定 |
| **CLOG / SLRU 扩展** | PG 17 把 CLOG / commit_ts / multixact 等 SLRU 改造为可扩展、动态大小、partitioned，缓解大并发下的锁争用 |
| **subxact 性能** | PG 14+ ProcArray 改造缓解 subxact overflow 的全局锁竞争；ORM 端仍建议控制嵌套 |
| **hint bit 替代** | 社区讨论用 "minimum frozen XID + age cache" 减少首次 SELECT 的 hint 写入；尚未稳定 |
| **逻辑复制下 horizon** | PG 17 改进逻辑 slot 过期回收，仍需 DBA 监控 |
| **pgvector / 向量索引下的 MVCC** | HNSW 索引也走标准 CTID + 可见性判定；与堆表一致 |
| **CloudNativePG / 容器化** | autovacuum 调参在 K8s 环境需要特别关注（小 work_mem / 多 worker） |

---

## 第 17 章：练习题

### 题 1（基础）：xmin/xmax 变化

事务 T1（xid=1000）执行以下序列：

```sql
BEGIN;
INSERT INTO t VALUES (1, 'A');   -- 命令 1
UPDATE t SET name='B' WHERE id=1;  -- 命令 2
DELETE FROM t WHERE id=1;        -- 命令 3
COMMIT;
```

问：commit 后表里能看到几条物理 tuple？各自的 xmin、xmax、cmin、cmax 是什么？

**答案要点：**

- 3 条物理 tuple：
  - tup_v1 (INSERT 产物)：xmin=1000, xmax=1000, cmin=0, cmax=1
  - tup_v2 (UPDATE 产物)：xmin=1000, xmax=1000, cmin=1, cmax=2
  - tup_v3 (DELETE 后实际是 v2 被打 xmax，所以没有新 tuple，是 v2 自己被改) — 修正：UPDATE 产生 v2 后，DELETE 只改 v2.xmax
- 实际是 **2 条**：v1（已被 UPDATE 杀掉）+ v2（先 UPDATE 创建，后 DELETE 标记）
- 三条都已"死"，等 VACUUM

### 题 2（基础）：PG vs MySQL

**简述 PG 和 MySQL 在 UPDATE 一行时的物理行为差异，并说明 PG 为什么需要 VACUUM。**

**答案要点：**

- PG：在表里写新版本 tuple，旧版本不动，xmax 标记被谁更新
- MySQL InnoDB：原地修改当前行；把旧值写到 undo log
- PG 旧版本物理留在堆中 → 需要 VACUUM 显式清理
- MySQL 旧版本在 undo log → purge 线程隐式清理

### 题 3（中级）：可见性判定

我的事务 snapshot：xmin=200, xmax=210, xip=[200, 205, 208], MyXid=211, MyCid=2。

CLOG: 199=COMMITTED, 200=IN_PROGRESS, 201=COMMITTED, 205=IN_PROGRESS, 208=IN_PROGRESS, 209=COMMITTED, 210=未分配

判定下列 tuple 是否对我可见：

(a) xmin=199, xmax=0
(b) xmin=201, xmax=205
(c) xmin=205, xmax=0
(d) xmin=209, xmax=0
(e) xmin=211, xmax=0, cmin=1
(f) xmin=211, xmax=0, cmin=3

**答案：**

- (a) xmin=199 < snap.xmin=200，必已 commit；xmax=0 → **可见**
- (b) xmin=201 在区间但不在 xip，CLOG=commit；xmax=205 在 xip → 删除未发生 → **可见**
- (c) xmin=205 在 xip → 还活跃 → **不可见**
- (d) xmin=209 在区间但不在 xip，CLOG=commit；但 209 < snap.xmax=210 ✓，xmax=0 → **可见**
- (e) 我自己写的；cmin=1 < MyCid=2 → 之前命令写的 → **可见**
- (f) cmin=3 ≥ MyCid=2 → 未来命令写的 → **不可见**

### 题 4（中级）：subxact

```sql
BEGIN;
INSERT INTO t VALUES (10, 'A');
SAVEPOINT s1;
INSERT INTO t VALUES (11, 'B');
SAVEPOINT s2;
INSERT INTO t VALUES (12, 'C');
ROLLBACK TO s1;
INSERT INTO t VALUES (13, 'D');
COMMIT;
```

最终表里能看到哪些行？

**答案：** 10 (A) 和 13 (D)。

- `INSERT 10` 在父事务，commit 后存活
- `INSERT 11` 在 s1 子事务，被 ROLLBACK TO s1 撤销
- `INSERT 12` 在 s2 子事务，被 ROLLBACK TO s1 撤销
- `INSERT 13` 在父事务（ROLLBACK TO s1 后回到父），存活

### 题 5（进阶）：hint bit

为什么一个 SELECT 在某些情况下会产生大量 IO？描述完整链路。

**答案要点：**

1. SELECT 触发可见性判定
2. tuple 无 hint bit，需要查 CLOG
3. 完成判定后写 hint bit 到 tuple's infomask
4. 修改了页 → 标脏页
5. 大量页变脏 → bgwriter / checkpointer 写盘
6. 第一次大规模 SELECT 一张刚写入大量数据的表会看到这个现象

应对：批量导入后显式 `VACUUM` 一次，让 autovacuum 把 hint bit 提前刷上。

### 题 6（进阶）：RC vs RR 写冲突

两个事务 T1、T2，都执行 `UPDATE accounts SET balance = balance - 100 WHERE id=1`。RC 和 RR 下的行为差异？

**答案：**

- **RC：** T2 等 T1 的行锁；T1 commit 后 T2 拿锁，重新读 row 当前版本（基于新的 snapshot），用新 balance 计算，正确扣 100
- **RR：** T2 等 T1 的行锁；T1 commit 后 T2 拿锁，但发现行的 xmax 是它快照外的 xid → 报错 `40001 could not serialize access due to concurrent update`，需要应用重试

### 题 7（进阶）：长事务

为什么一个跑了 1 小时的 SELECT 在主库会导致全库膨胀？怎么监控、怎么应对？

**答案要点：**

- 该 SELECT 的 snapshot.xmin 卡住全局 OldestXmin
- 所有比它新的 UPDATE/DELETE 产生的 dead tuple 都无法被 VACUUM 清理
- 全库 n_dead_tup 暴涨 → 表膨胀、索引膨胀

监控：

```sql
SELECT pid, now() - xact_start AS dur, query
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
ORDER BY xact_start;
```

应对：

- `statement_timeout = '10min'` 或类似
- 报表走只读副本
- 使用 `idle_in_transaction_session_timeout`
- 强制 kill 长事务（`pg_terminate_backend(pid)`），让 horizon 推进

### 题 8（进阶）：wraparound

TPS 为 1 万的库，autovacuum 完全停掉一周会发生什么？

**答案：**

- 一周内消耗 xid ≈ 1万 × 86400 × 7 ≈ 6 亿
- 各表 relfrozenxid age 持续增长
- 超过 `autovacuum_freeze_max_age`（默认 2 亿）后，PG 会强制启动 anti-wraparound vacuum（即使 autovacuum 关闭也会启动一种特殊形式；如果完全屏蔽，则继续累积）
- 超过 `vacuum_failsafe_age`（默认 16 亿）后，VACUUM 切换 failsafe 模式
- 接近 21 亿后，PG 拒绝接受新事务（防止 wraparound），强制单用户模式 VACUUM
- 如果继续不处理 → 越过 2³¹（约 21 亿）边界时 wraparound 触发，全库数据"消失"

---

## 第 18 章：延伸阅读

### 官方文档

- [Chapter 13: Concurrency Control](https://www.postgresql.org/docs/18/mvcc.html)（必读）
- [Chapter 73: System Catalogs - pg_xact](https://www.postgresql.org/docs/18/storage-fsm.html)
- [Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)
- [Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)

### 源码导览（PG 18，src/backend）

- `src/backend/access/heap/heapam_visibility.c` — 可见性判定核心，**必读**：
  - `HeapTupleSatisfiesMVCC` 主函数
  - `HeapTupleSatisfiesAny` / `Self` / `Dirty` / `Toast` / `Vacuum` 其他变体
- `src/backend/utils/time/snapmgr.c` — Snapshot 管理：
  - `GetSnapshotData` 构造 snapshot
  - `PushActiveSnapshot` / `PopActiveSnapshot` 栈管理
  - `RegisterSnapshot` / `UnregisterSnapshot`
- `src/backend/access/transam/clog.c` — CLOG 实现
- `src/backend/access/transam/subtrans.c` — 子事务管理
- `src/backend/storage/lmgr/proc.c` — ProcArray
- `src/include/access/htup_details.h` — Tuple Header 定义（必读）

### 经典文章/书

- 《PostgreSQL Internals》(Egor Rogov) 第 5–6 章 — 中文翻译版可在 postgrespro.com 找到
- 《PostgreSQL 14 Internals》中文电子版（postgrespro 出品，免费）
- Bruce Momjian *MVCC Unmasked* slides — postgresql.org 的 wiki 上有，必看
- Robert Haas *Transaction IDs and Wraparound* 系列博客
- pganalyze blog: *Postgres VACUUM Behavior* 系列

### 工具与扩展

- `pageinspect` — 看页 / tuple header
- `pgstattuple` — 字节级膨胀统计
- `pg_visibility` — 看 visibility map
- `pg_buffercache` — 看缓冲池状态
- `pg_repack` / `pg_squeeze` — 在线重排表（替代 VACUUM FULL）

---

> 下一步：读 [P04 B-tree 索引](./04-精通-B-tree-索引.md)，看可见性判定如何被 visibility map 优化为 index-only scan；再读 [P07 事务隔离与锁](./07-精通-事务隔离与锁.md) 把隔离级别 + 锁的全景拼上；最后读 [P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)，把"清理"这一面补齐。
