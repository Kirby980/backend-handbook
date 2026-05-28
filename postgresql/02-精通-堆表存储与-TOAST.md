# 精通堆表存储与 TOAST：Page 布局、Tuple Header、CTID、HOT update 与大字段存储

> 关联章节：[P01 整体架构](./01-精通-PostgreSQL-整体架构.md)、[P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)、[P04 B-tree 索引](./04-精通-B-tree-索引.md)、[P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)

---

## 引言：PostgreSQL 的"堆表"到底"堆"在哪

学过 MySQL InnoDB 的人看 PostgreSQL 第一个困惑是：**为什么 PG 没有"聚簇索引"这个概念？**

InnoDB 表本身就是一棵 B+ 树（按主键组织，叶子页是完整行）；PG 的表是 **堆（heap）**——行无序地、追加式地堆放在数据文件里，所有索引都是独立的 B-tree / GIN / GiST...，叶子页里存的是 `(索引键, CTID)`，CTID 指向堆表中的物理位置。

这个差异是 PG 与 MySQL 在存储层最根本的分歧，由此衍生出：

- PG 行有"物理位置"概念（CTID），但 **CTID 不稳定**（UPDATE / VACUUM FULL 后会变）
- PG 不能"按主键物理排序"（CLUSTER 命令只是一次性重排）
- PG 的 UPDATE 是 **写新版本不动旧版本**（堆中物理共存），自然支持真 MVCC
- PG 必须有 VACUUM（清理旧版本），MySQL 不需要
- PG 大字段（>2KB）走 TOAST 独立表，行内只留指针；InnoDB 走 off-page BLOB 页

读完这一章，你应该能：

- 画出 PG 8KB 数据页的内部布局（PageHeader / ItemId 数组 / 数据区 / Special）
- 说出 Tuple Header 24 字节的每个字段含义
- 解释 CTID = (block, offset) 的物理意义和为什么不稳定
- 选择合适的 TOAST 策略（PLAIN / EXTENDED / EXTERNAL / MAIN）
- 用 FILLFACTOR + 列设计触发 HOT update，把更新性能提升 5 倍
- 用 `pageinspect` 在生产中观察一页的字节级布局
- 对比 PG heap vs InnoDB 聚簇索引的取舍

---

## 第一章：为什么 PG 没有聚簇索引

### 1.1 InnoDB 聚簇索引回顾

InnoDB 中，表的数据**就是**主键 B+ 树的叶子页：

```
                    InnoDB 表 = 主键 B+ 树
                          [50, 100]
                         /    |    \
                  [1..49]  [50..99]  [100..]
                  (完整行)  (完整行)   (完整行)
```

- 主键索引叶子 = 完整行
- 二级索引叶子 = `(二级键, 主键)` → 回表用主键再查一次 B+ 树

### 1.2 PostgreSQL 堆表 + 独立索引

```
   堆表（heap）：行无序追加
   ┌─────────────────────────┐
   │ block 0: [row1][row2]   │
   │ block 1: [row3][row4]   │
   │ block 2: [row5][row6]   │  CTID = (block_num, offset_in_block)
   │ ...                     │
   └─────────────────────────┘
              ↑ ↑ ↑
              │ │ │ 索引叶子里存 (key, CTID) 指过来
              │ │ │
   ┌──────────┴─┴─┴──────┐    ┌──────────────────┐
   │ B-tree on id (PK)   │    │ B-tree on email  │
   │   (1, (2,1))        │    │ ('a@x', (0,1))   │
   │   (2, (0,1))        │    │ ('b@x', (2,2))   │
   │   ...               │    │   ...            │
   └─────────────────────┘    └──────────────────┘
```

PG 的**所有索引都是二级索引**——包括主键索引。主键索引和邮箱索引在 PG 看来没有本质区别，都是 `B-tree(key) → CTID`。

### 1.3 为什么这么设计：CTID 不稳定的代价

如果像 InnoDB 一样把数据组织到 B+ 树叶子里，一旦更新主键、行迁移、页分裂，**所有二级索引都要更新指针**。InnoDB 解决办法：二级索引存主键值（而不是物理指针），用主键回表。代价是二级索引膨胀（主键越宽，二级索引越大）+ 回表两次树搜索。

PG 选择堆表：索引指针 = CTID（6 字节物理地址），最小化索引大小。代价是：

- **CTID 在 UPDATE 后会变**（除非 HOT update，详见第六章）
- **VACUUM FULL / CLUSTER 会改所有 CTID**
- 没有"主键物理顺序"，范围扫主键不一定比扫别的列快

### 1.4 对比总表

| 维度 | InnoDB（聚簇索引） | PostgreSQL（堆表+独立索引） |
|---|---|---|
| 表结构 | 主键 B+ 树叶子 = 完整行 | 堆文件，行无序追加 |
| 主键索引 | 即表本身 | 独立 B-tree，叶子是 (key, CTID) |
| 二级索引叶子 | (二级键, 主键) | (二级键, CTID) |
| 二级索引回表 | 必须（查主键再查 PK 树） | 直接 CTID 寻址，1 次随机 IO |
| 二级索引大小 | 包含主键，较大 | 只含 CTID(6B)，较小 |
| 主键范围扫 | 极快（物理顺序） | 不一定快（堆无序） |
| 是否需要回表 | 一定要 | 看情况（visibility map 全可见时 index-only） |
| 主键改值 | 触发整行迁移（重组 B+ 树） | 等于 DELETE + INSERT |
| 大字段 | off-page BLOB 页（同表空间） | TOAST（独立辅表） |
| MVCC 实现 | undo log（旧版本逻辑回滚） | 多版本物理共存堆中（真 MVCC） |
| 表碎片清理 | 自动 + OPTIMIZE TABLE 重建 | VACUUM（标记复用） + VACUUM FULL（重建） |
| 写入放大 | 主键 + 二级索引都要写 | 堆 + 每个索引都要写 |
| HOT update | 不适用 | 同页内更新不动索引 → 大幅减少索引膨胀 |

### 1.5 PG 也有"伪聚簇"：CLUSTER 命令

```sql
CLUSTER users USING idx_users_created_at;
-- 按 idx_users_created_at 的顺序重写整张表
```

执行后表的物理顺序 = 索引顺序，**但后续的 INSERT / UPDATE 不会维护这个顺序**——堆表本性如此。需要定期重新 `CLUSTER`（持有 ACCESS EXCLUSIVE 锁，相当于停服务）。生产很少用，反倒是 `pg_repack` 扩展更常见（在线重排）。

---

## 第二章：8 KB 数据页内部结构

### 2.1 块大小固定 8 KB

```bash
$ pg_controldata $PGDATA | grep block
Database block size:    8192
WAL block size:         8192
```

`BLCKSZ` 是编译时常量（默认 8192）。和 MySQL InnoDB 默认 16 KB 不同——PG 选 8 KB 的历史原因是与早期 Unix 文件系统块对齐。今天可以编译时改成 16 KB 或 32 KB，但极少有人这么做（破坏了所有现成的扩展二进制兼容）。

8 KB 的代价与收益：

| 优势 | 劣势 |
|---|---|
| 单页读写代价小 | 一行最大 ~8 KB（更大要 TOAST） |
| 全页写（FPW）开销小（vs InnoDB 16KB） | TOAST 阈值 2 KB，比 InnoDB 8 KB 更早触发 |
| 索引页能放更多 key（绝对值少但相对密集） | 表大时页数多（10 GB 表 = 130 万页） |

### 2.2 页的整体布局

```
偏移 0
┌───────────────────────────────────┐
│  PageHeaderData (24 B)            │  ← 校验、LSN、空闲空间指针
├───────────────────────────────────┤
│  ItemIdData [0]   (4 B 每项)      │  ← 行指针数组，向下生长
│  ItemIdData [1]                   │
│  ItemIdData [2]                   │
│  ...                              │
├───────────────────────────────────┤
│                                   │
│       FREE SPACE                  │  ← 空闲区（向两端收缩）
│                                   │
├───────────────────────────────────┤
│  HeapTuple [N-1]   ↑              │  ← 数据区，向上生长
│  HeapTuple [N-2]                  │
│  ...                              │
│  HeapTuple [0]                    │
├───────────────────────────────────┤
│  Special Space (可选, 通常 0 B)   │  ← 索引页才用，堆页没有
└───────────────────────────────────┘
偏移 8192
```

关键设计：**ItemId 数组向下生长，Tuple 数据向上生长**，中间是空闲空间。新插入行时：

1. 在空闲区底部分配 Tuple 空间
2. 在 ItemId 数组末尾追加一个 ItemId 指向新 Tuple

这样**Tuple 物理移动时 ItemId 可以保持位置不变**——这就是 CTID 的稳定性来源（同页内）。

### 2.3 PageHeader 详细字段（24 B）

```c
// src/include/storage/bufpage.h
typedef struct PageHeaderData {
    PageXLogRecPtr pd_lsn;        // 8 B  WAL LSN（页最后修改）
    uint16         pd_checksum;   // 2 B  校验（data_checksums=on 时）
    uint16         pd_flags;      // 2 B  标志位
    LocationIndex  pd_lower;      // 2 B  空闲区起点（=ItemId 数组末尾）
    LocationIndex  pd_upper;      // 2 B  空闲区终点（=最低 Tuple 起点）
    LocationIndex  pd_special;    // 2 B  Special 区起点
    uint16         pd_pagesize_version;  // 2 B  页大小+版本号
    TransactionId  pd_prune_xid;  // 4 B  下一次可裁剪的 xid（HOT 清理用）
} PageHeaderData;                  // 共 24 B
```

| 字段 | 含义 |
|---|---|
| `pd_lsn` | 这一页最后写入对应的 WAL 位置；用于"先 WAL 后数据页"的崩溃恢复 |
| `pd_checksum` | 启用 `data_checksums` 时的页校验值，检测磁盘静默损坏 |
| `pd_flags` | PD_HAS_FREE_LINES / PD_PAGE_FULL / PD_ALL_VISIBLE 等 |
| `pd_lower / pd_upper` | 空闲区边界：`空闲字节 = pd_upper - pd_lower` |
| `pd_special` | Special 区起点；堆页 = 8192（无 Special），索引页 < 8192 |
| `pd_pagesize_version` | 高 8 位 = pagesize（8192）；低 8 位 = page format version（PG 8.3+ 都是 4） |
| `pd_prune_xid` | "裁剪提示"：如果 xmax < pd_prune_xid，该 tuple 可能可清理 |

### 2.4 ItemId 数组（每项 4 B）

```c
typedef struct ItemIdData {
    unsigned   lp_off:15,    // 15 bit: tuple 在页内偏移
               lp_flags:2,   //  2 bit: 状态
               lp_len:15;    // 15 bit: tuple 长度
} ItemIdData;                 // 4 B
```

`lp_flags` 四种状态：

| 值 | 名称 | 含义 |
|---|---|---|
| 0 | LP_UNUSED | 该 slot 空闲（可复用） |
| 1 | LP_NORMAL | 指向正常的 Tuple |
| 2 | LP_REDIRECT | HOT chain 中间节点，转发到另一个 ItemId |
| 3 | LP_DEAD | 标记为死亡（可清理），用于 index-only scan 提前告知 |

**LP_REDIRECT 是 HOT 更新的关键**——当一个行被 HOT 更新多次后形成链：

```
旧索引指针 → ItemId[3] (LP_REDIRECT 指向 ItemId[7])
                            ↓
                       ItemId[7] (LP_NORMAL 指向最新 tuple)
```

索引只需指向原始 ItemId[3]，HOT 更新换内容不换索引。

### 2.5 CTID 的物理意义

```
CTID = (块号, ItemId 序号)
     = (block_number, item_offset)

CTID 占 6 字节：4 B 块号 + 2 B item 序号（其中 16 位实际只用 11）
```

```sql
SELECT ctid, id, name FROM users LIMIT 5;
--   ctid  | id |  name
-- --------+----+-------
--  (0,1)  |  1 | Alice    ← 第 0 块的第 1 项
--  (0,2)  |  2 | Bob
--  (0,3)  |  3 | Carol
--  (0,4)  |  4 | David
--  (0,5)  |  5 | Eve
```

注意：

- **ItemId 序号从 1 开始**（不是 0）
- 同一块内项数取决于行宽，典型 ~50-200 项/块
- CTID 在 UPDATE 后**通常会变**（除 HOT update 同块内更新）；DELETE 后 ItemId 标 LP_UNUSED，可被新行复用

---

## 第三章：Tuple Header 详解（24 字节）

### 3.1 完整结构

```c
// src/include/access/htup_details.h
typedef struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;                  // 12 B

    ItemPointerData t_ctid;      //  6 B  自指或指向新版本
    uint16          t_infomask2; //  2 B  属性数 + 标志
    uint16          t_infomask;  //  2 B  类型/状态标志
    uint8           t_hoff;      //  1 B  到数据区的偏移

    bits8           t_bits[];    //  变长 NULL 位图（每列 1 bit）

    /* 后面跟数据列 */
} HeapTupleHeaderData;
```

Header **固定 23 B**（最后 1 B 是 `t_hoff` 自身），加上对齐通常占 24 B。可变长部分是 NULL bitmap（每 8 列 1 字节，全部 NOT NULL 时省略）。

### 3.2 12 B 的 t_choice（事务信息）

```c
typedef struct HeapTupleFields {
    TransactionId t_xmin;       // 4 B  插入此版本的事务 ID
    TransactionId t_xmax;       // 4 B  删除/更新此版本的事务 ID（0 表示未删）
    union {
        CommandId t_cid;        // 4 B  command ID（同事务内多语句区分）
        TransactionId t_xvac;   // 4 B  VACUUM 移动事务 ID（VACUUM FULL 旧版本用）
    } t_field3;
} HeapTupleFields;
```

| 字段 | 用途 |
|---|---|
| `t_xmin` | 谁创建了这个版本（INSERT/UPDATE 的事务） |
| `t_xmax` | 谁删除/更新了这个版本（0 表示活着；非 0 表示被某事务覆盖） |
| `t_cid` | 同一事务内的命令 ID，用于 cursor 内可见性（自己刚改的行下一条 SELECT 能不能看见） |
| `t_xvac` | 旧式 VACUUM FULL（PG 9.0 前）的迁移事务标记，现已基本不用 |

**xmin + xmax 是 MVCC 的基石**。详见 [P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)。

### 3.3 t_ctid（6 B 自指 / 转发）

正常活跃行：`t_ctid = 自己的 CTID`。

UPDATE 后：旧版本的 `t_ctid` 指向新版本的 CTID（形成 update chain）。VACUUM 沿着这条链清理。

DELETE 后：`t_ctid = (0, 0)` 或保留指向自己（不再链接到新版本，因为没有）。

### 3.4 t_infomask2（2 B）

低 11 bit：列数（`HEAP_NATTS_MASK = 0x07FF`，最多 1664 列）

高 5 bit 是标志：

```c
#define HEAP_KEYS_UPDATED     0x2000  // UPDATE 修改了 key（影响 HOT 判定）
#define HEAP_HOT_UPDATED      0x4000  // 此版本被 HOT 更新过
#define HEAP_ONLY_TUPLE       0x8000  // 此版本是 HOT chain 中的非首节点（索引未指向它）
```

### 3.5 t_infomask（2 B 状态标志，最关键）

```c
#define HEAP_HASNULL          0x0001  // 行内有 NULL 列
#define HEAP_HASVARWIDTH      0x0002  // 行内有变长列
#define HEAP_HASEXTERNAL      0x0004  // 行内有 TOAST 外部存储字段
#define HEAP_HASOID_OLD       0x0008  // 行内有 OID（PG 12 移除）
#define HEAP_XMAX_KEYSHR_LOCK 0x0010  // xmax 是 KEY SHARE 锁
#define HEAP_COMBOCID         0x0020  // t_cid 是 ComboCid
#define HEAP_XMAX_EXCL_LOCK   0x0040  // xmax 是排他行锁
#define HEAP_XMAX_LOCK_ONLY   0x0080  // xmax 只表示锁定（非删除）
#define HEAP_XMIN_COMMITTED   0x0100  // ★ xmin 已确认提交（hint bit）
#define HEAP_XMIN_INVALID     0x0200  // ★ xmin 已确认回滚（hint bit）
#define HEAP_XMAX_COMMITTED   0x0400  // ★ xmax 已确认提交
#define HEAP_XMAX_INVALID     0x0800  // ★ xmax 已确认回滚
#define HEAP_XMAX_IS_MULTI    0x1000  // xmax 是 MultiXactId
#define HEAP_UPDATED          0x2000  // 这是 UPDATE 产生的版本
#define HEAP_MOVED_OFF        0x4000  // VACUUM FULL 旧式移动标记
#define HEAP_MOVED_IN         0x8000  // 同上
```

**最重要的是 4 个 hint bits**（HEAP_XMIN/XMAX_COMMITTED/INVALID）。

### 3.6 t_hoff（数据区偏移）

Tuple Header + NULL bitmap + 对齐填充后，从哪个字节开始是真正的列数据。一般 24-32 字节。

---

## 第四章：行格式与对齐

### 4.1 整体布局

```
┌──────────────────────────────────────────────────────────────┐
│ Tuple                                                        │
├──────────────────────────────────────────────────────────────┤
│  HeapTupleHeader (23 B)                                      │
│    ├─ t_xmin (4)                                             │
│    ├─ t_xmax (4)                                             │
│    ├─ t_cid (4)                                              │
│    ├─ t_ctid (6)                                             │
│    ├─ t_infomask2 (2)                                        │
│    ├─ t_infomask (2)                                         │
│    └─ t_hoff (1)                                             │
├──────────────────────────────────────────────────────────────┤
│  NULL Bitmap (可选，每 8 列 1 字节)                          │
├──────────────────────────────────────────────────────────────┤
│  对齐填充 (到 MAXALIGN，通常 8 字节边界)                     │
├──────────────────────────────────────────────────────────────┤
│  数据区                                                       │
│    列 1（按类型对齐）                                         │
│    列 2                                                       │
│    ...                                                        │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 列对齐规则

| 类型 | typalign | 对齐字节 |
|---|---|---|
| `char`, `bool` | 'c' | 1 |
| `smallint` (int2) | 's' | 2 |
| `integer` (int4), `date`, `real` | 'i' | 4 |
| `bigint` (int8), `double precision`, `timestamp` | 'd' | 8 |
| `text`, `varchar`, `numeric`, `bytea` | 'i' | 4（变长头对齐到 4） |

变长字段头有两种：

- **短头 1 B**：长度 ≤ 126 字节时
- **长头 4 B**：长度 > 126 字节或需要 TOAST 时

### 4.3 列顺序影响行大小

考虑两种等价 schema：

```sql
-- 版本 A（差）
CREATE TABLE t (
  a smallint,    -- 2 B + 6 B 填充对齐到 8（因为下一列是 bigint）
  b bigint,      -- 8 B
  c smallint     -- 2 B + 6 B 填充对齐到行尾
);
-- 实际占用：2 + 6 + 8 + 2 + 6 = 24 B 数据区

-- 版本 B（好）
CREATE TABLE t (
  b bigint,      -- 8 B
  a smallint,    -- 2 B
  c smallint     -- 2 B + 4 B 填充对齐到行尾（MAXALIGN=8）
);
-- 实际占用：8 + 2 + 2 + 4 = 16 B 数据区
```

**经验：把大对齐字段（bigint / timestamp / double）放前面，小对齐字段（smallint / bool）放后面。** 10 亿行表能省 8 GB+。

```sql
-- 检查行大小
SELECT pg_column_size(row(*)) AS row_bytes FROM t LIMIT 1;
```

### 4.4 NULL 的代价

NOT NULL 列 + 全行无 NULL → `HEAP_HASNULL = 0`，无 NULL bitmap，省 1 B+。

NULL 列在数据区**不占空间**，只在 bitmap 中标位。但需要 NULL bitmap（每 8 列 1 B），且 `pg_column_size` 不算 bitmap，实际行大小要加上。

经验：能 NOT NULL 就 NOT NULL，既省存储又利于优化器。

---

## 第五章：TOAST —— 大字段独立存储

### 5.1 TOAST 是什么

**The Oversized-Attribute Storage Technique**：当一行超过约 2 KB 时，PG 自动把大字段压缩并搬到独立的 TOAST 辅表中，行内只留 18 字节指针。

为什么需要：

- 单行不能跨页（行 ≤ 页大小 = 8 KB）
- 即使一行能放进 8 KB，"宽行"也会拖慢页扫描
- 大字段（JSON / TEXT / BYTEA / 数组）需要被压缩、被分块

### 5.2 TOAST 表的位置

每张包含可能 TOAST 字段（变长类型）的表自动有一张 TOAST 表：

```sql
SELECT
  c.relname AS table_name,
  t.relname AS toast_name,
  pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
WHERE c.relname = 'documents';
--  table_name  |    toast_name      | toast_size
-- -------------+--------------------+------------
--  documents   | pg_toast_16400     | 1234 MB
```

TOAST 表的 schema 固定：

```sql
\d pg_toast.pg_toast_16400
--   chunk_id  | oid     | 一个大字段的唯一标识
--   chunk_seq | integer | 第几个分块
--   chunk_data| bytea   | 分块数据（默认每块 ~2000 B）
-- 索引: (chunk_id, chunk_seq)
```

一个大字段被切成多个 chunk_data（约 2 KB / 块），按 `(chunk_id, chunk_seq)` 索引能高效拼回。

### 5.3 触发阈值

```c
// src/include/access/heaptoast.h
#define TOAST_TUPLE_THRESHOLD  2032   // 行 > 2032 B 时尝试 TOAST
#define TOAST_TUPLE_TARGET     2032   // TOAST 后行目标 ≤ 2032 B
#define TOAST_MAX_CHUNK_SIZE   1996   // 每块约 2 KB
```

可调（per-table）：

```sql
ALTER TABLE t SET (toast_tuple_target = 4096);   -- 提高阈值减少 TOAST
```

### 5.4 4 种存储策略

每个变长列可独立设置：

```sql
ALTER TABLE t ALTER COLUMN body SET STORAGE EXTENDED;  -- 默认
```

| 策略 | 行为 | 默认场景 |
|---|---|---|
| **PLAIN** | 不压缩、不 out-of-line | 定长类型（int / timestamp） |
| **EXTENDED** | 先压缩，仍超阈值再 out-of-line | 大部分变长（text / jsonb / bytea），**默认** |
| **EXTERNAL** | 不压缩，超阈值 out-of-line | 想保留可子串/索引时（大 text） |
| **MAIN** | 先压缩，尽量留行内（除非实在塞不下） | 想最大化行内 IO 局部性 |

经验：

- JSONB → 默认 EXTENDED 即可，但如果经常用 `->` 取部分字段，**用 EXTERNAL 避免解压代价**
- 短字符串（< 100 B 占多数）→ 改 PLAIN 省 TOAST 元数据开销
- 几 MB 的 BYTEA → EXTERNAL 配合压缩列（应用层压缩 + DB 不再压缩）

### 5.5 压缩算法：pglz vs lz4

```sql
SHOW default_toast_compression;  -- PG 14+
--  pglz   （默认，向后兼容）
--  lz4    （PG 14+，更快更省）
```

| 算法 | 压缩率 | 压缩速度 | 解压速度 | 何时选 |
|---|---|---|---|---|
| **pglz** | 一般 | 慢 | 中 | 与老版本兼容 |
| **lz4** | 略低 | 快 5-10× | 快 2-5× | **PG 14+ 强烈推荐** |

调整方法：

```sql
-- 全局
ALTER SYSTEM SET default_toast_compression = 'lz4';
SELECT pg_reload_conf();

-- 单列
ALTER TABLE t ALTER COLUMN body SET COMPRESSION lz4;

-- 新插入数据会用 lz4；已有 TOAST 行不会重压，需要 VACUUM FULL 才重新编码
```

实测：日志类 JSONB 表从 pglz 改 lz4，CPU 下降 30%，TOAST 大小略增 5-10%。

### 5.6 TOAST 的隐藏代价

- **每个大字段一次额外索引查找**：取 jsonb 字段时要查 TOAST 表的 (chunk_id, chunk_seq) 索引 + 多次 chunk 读取
- **TOAST 表自己也有 dead tuple**：大字段 UPDATE 会在 TOAST 表中新写 chunk，旧 chunk 等 VACUUM
- **VACUUM 必须扫 TOAST 表**：表 + TOAST 两份 IO
- **EXPLAIN 看不到 TOAST 解压时间**：表现为 "Seq Scan 估算 1ms 实际 1s" 时往往是 TOAST 解压
- **HOT update 不支持改 TOAST 字段**：改 toasted 列必然触发非 HOT 更新

### 5.7 观察 TOAST

```sql
-- 1. 哪些字段被 toasted
SELECT
  pg_column_size(body) AS stored_size,    -- 包含 TOAST 指针
  octet_length(body) AS logical_size,     -- 解压后逻辑大小
  (pg_column_size(body) < 100)::int AS likely_toasted
FROM documents LIMIT 10;

-- 2. TOAST 表大小占比
SELECT
  relname,
  pg_size_pretty(pg_relation_size(oid)) AS main_size,
  pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_size,
  pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relkind = 'r' AND reltoastrelid != 0
ORDER BY pg_relation_size(reltoastrelid) DESC
LIMIT 10;

-- 3. TOAST 膨胀（dead chunks）
SELECT * FROM pgstattuple('pg_toast.pg_toast_16400');
-- 需要 CREATE EXTENSION pgstattuple;
```

---

## 第六章：HOT Update —— PG 写性能的灵魂技巧

### 6.1 普通 UPDATE 的代价

```sql
UPDATE users SET last_login = now() WHERE id = 1;
```

PG 不就地修改（堆表保留所有版本），而是：

1. 旧 tuple 的 `t_xmax = current_xid`（不删除，只标记）
2. **写一个新 tuple**（新 CTID）含修改后的列
3. **更新所有索引指针**：每个索引都新增一项 `(key, 新CTID)`，旧索引项保留待 VACUUM

如果表有 5 个索引，一次 UPDATE 产生 1 个旧 tuple + 1 个新 tuple + 5 个新索引项 = **大量写 IO + 索引膨胀**。

### 6.2 HOT (Heap-Only Tuple) Update

**触发条件（必须全部满足）**：

1. 新版本写在**同一页**（同 block）
2. 修改的列**没有任何索引**
3. 新版本不大幅改变行宽（同一页放得下）

**效果**：

1. 旧 tuple `t_xmax = current_xid`，`t_ctid → 新 tuple 的 CTID`
2. 新 tuple 标记 `HEAP_ONLY_TUPLE`，索引**不更新**（仍指向旧 ItemId）
3. 旧 ItemId 升级为 LP_REDIRECT，指向新 ItemId
4. 查询时索引 → 旧 ItemId → 沿 ctid 链找到最新可见版本

```
更新前：
  Index: (key=1) → ItemId[3] → tuple_v1 {id=1, x=5}

UPDATE t SET x=6 WHERE id=1;  -- x 没有索引！

更新后（HOT update 成功）：
  Index: (key=1) → ItemId[3] (LP_REDIRECT) → ItemId[7]
                                                ↓
                                          tuple_v2 {id=1, x=6, HEAP_ONLY_TUPLE}
  原 tuple_v1 仍在原位（xmax=current_xid, t_ctid→ItemId[7] 的 (block, 7)）
```

**收益**：

- 索引完全没动 → 索引几乎不膨胀
- 单次 UPDATE 的 WAL 量降到 1/3 - 1/5
- HOT chain 内的旧版本可以被 **"page-level prune"** 在普通 SELECT 时顺手清理（不用等 VACUUM）

### 6.3 FILLFACTOR：为 HOT 留余地

默认 `fillfactor = 100`，意味着页装到 100% 才换页。HOT update 需要新版本与旧版本同页——页都满了哪有空间？

```sql
-- 更新频繁的表降低 fillfactor，留 20% 空闲给 HOT
ALTER TABLE users SET (fillfactor = 80);

-- 几乎只读的表保持 100（不浪费空间）
ALTER TABLE archive_logs SET (fillfactor = 100);

-- 索引也有 fillfactor（B-tree 默认 90）
ALTER INDEX idx_users_name SET (fillfactor = 70);
```

经验取值：

| 表类型 | fillfactor |
|---|---|
| 几乎只读 | 100（默认） |
| 偶尔 UPDATE | 90 |
| 频繁 UPDATE（HOT 候选） | 80 |
| 几乎全 UPDATE | 70 |

`ALTER TABLE ... SET (fillfactor=80)` 只影响**新页**——历史页不会重新分布。要全表生效需 `VACUUM FULL` 或 `pg_repack`。

### 6.4 设计原则：避免在频繁更新的列上建索引

```sql
-- 反例：last_login 频繁更新，又被索引
CREATE INDEX idx_users_last_login ON users(last_login);
UPDATE users SET last_login = now() WHERE id = 1;
-- HOT update 失效 → 索引膨胀
```

如果业务确实要按 last_login 查，常见对策：

1. **拆表**：把 last_login 拆到 `user_activity` 表，主表 `users` 不动
2. **降频**：last_login 每 5 分钟更新一次（用 WHERE 减少更新）
3. **接受**：定期 REINDEX

### 6.5 验证 HOT update 比例

```sql
SELECT
  relname,
  n_tup_upd AS total_updates,
  n_tup_hot_upd AS hot_updates,
  round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 1) AS hot_pct
FROM pg_stat_user_tables
ORDER BY n_tup_upd DESC
LIMIT 10;
--  relname |  total_updates  | hot_updates | hot_pct
-- ---------+-----------------+-------------+---------
--  users   |     1234567     |   1100000   |   89.1
--  orders  |      456789     |    50000    |   10.9   ← 太低！查索引
```

`hot_pct < 50%` 的更新密集表需要调查为什么 HOT 失败：

1. fillfactor 是否过高（页满了）
2. 是否更新了被索引的列
3. 行是否太宽（新版本同页放不下）

---

## 第七章：用 pageinspect 在生产观察页内容

### 7.1 安装与基本用法

```sql
CREATE EXTENSION pageinspect;
```

### 7.2 看 PageHeader

```sql
SELECT * FROM page_header(get_raw_page('users', 0));
--    lsn      | checksum | flags | lower | upper | special | pagesize | version | prune_xid
-- ------------+----------+-------+-------+-------+---------+----------+---------+-----------
--  1/3A8E2D40 |        0 |     1 |   144 |   192 |    8192 |     8192 |       4 |   34892100
```

`(upper - lower) = 48` 字节空闲空间。

### 7.3 看所有 ItemId

```sql
SELECT lp, lp_off, lp_flags, lp_len
FROM heap_page_items(get_raw_page('users', 0));
--  lp | lp_off | lp_flags | lp_len
-- ----+--------+----------+--------
--   1 |   8136 |        1 |     56     ← LP_NORMAL
--   2 |   8080 |        1 |     56
--   3 |      0 |        0 |      0     ← LP_UNUSED（被 VACUUM 回收）
--   4 |   8024 |        2 |      0     ← LP_REDIRECT
--   5 |   7968 |        1 |     56
```

### 7.4 看完整 tuple 字段

```sql
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2,
       to_hex(t_infomask) AS infomask_hex
FROM heap_page_items(get_raw_page('users', 0))
WHERE lp_flags = 1;
--  lp | t_xmin | t_xmax | t_ctid | t_infomask | t_infomask2 | infomask_hex
-- ----+--------+--------+--------+------------+-------------+--------------
--   1 |  12345 |      0 |  (0,1) |       2304 |           3 | 900           ← HEAP_XMIN_COMMITTED|HEAP_XMAX_INVALID
--   2 |  12346 |  12350 |  (0,5) |       2816 |           3 | b00           ← 已被更新（指向 (0,5)）
--   5 |  12350 |      0 |  (0,5) |        256 |       32771 | 100           ← HEAP_ONLY_TUPLE
```

通过 t_infomask 比对常量值可以推断 hint bits 状态、HOT chain 结构、行锁状态。

### 7.5 看 B-tree 索引页

```sql
SELECT * FROM bt_metap('idx_users_name');     -- 元页（根、级别等）
SELECT * FROM bt_page_stats('idx_users_name', 1);  -- 单页统计
SELECT * FROM bt_page_items('idx_users_name', 1);  -- 单页项
```

### 7.6 pgstattuple：表/索引膨胀诊断

```sql
CREATE EXTENSION pgstattuple;

SELECT * FROM pgstattuple('users');
--  table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count |
--  dead_tuple_len | dead_tuple_percent | free_space | free_percent
-- ----------+-------------+-----------+---------------+------------------+
--  10485760 |       50000 |   3500000 |          33.4 |            10000 |
--    700000 |               6.7 |    5400000 |         51.5    ← 51% 空间浪费！
```

`dead_tuple_percent > 20%` 或 `free_percent > 40%` 是膨胀信号，需要 VACUUM 或 pg_repack。

---

## 第八章：堆表与 InnoDB 的取舍总结

### 8.1 写入场景对比

```sql
INSERT INTO users(id, name, email) VALUES (1, 'A', 'a@x');
```

| 步骤 | PostgreSQL | InnoDB |
|---|---|---|
| 1 | 找堆表有空闲的页（FSM） | 找 PK B+ 树中 id=1 应插入的叶子页 |
| 2 | 写 tuple 到页 | 写完整行到叶子页（可能页分裂） |
| 3 | 每个索引各加一项 | 每个二级索引各加一项（含主键） |
| 4 | 写 WAL | 写 redo log + undo log |

PG INSERT 通常**少一次写**（不用维护"按主键物理顺序"），但**索引项更小**（CTID 6B vs InnoDB 主键变长）。

### 8.2 主键范围扫对比

```sql
SELECT * FROM users WHERE id BETWEEN 1000 AND 2000;
```

- **InnoDB**：扫主键 B+ 树叶子，物理顺序，**1 次连续 IO**
- **PostgreSQL**：扫主键索引拿到 1000 个 CTID，逐个回堆，**1000 次随机 IO**（除非堆刚 CLUSTER 过）

这就是为什么 PG 偏好**索引覆盖**（INCLUDE）和**index-only scan**——避免回堆。

### 8.3 UPDATE 场景对比

```sql
UPDATE users SET last_login = now() WHERE id = 1;  -- last_login 无索引
```

| 步骤 | PostgreSQL (HOT) | InnoDB |
|---|---|---|
| 1 | 同页写新 tuple，旧 tuple 标 xmax | 就地更新（in-place），写 undo log |
| 2 | 索引不动 | 索引不动（非索引列） |
| 3 | 写 WAL | 写 redo + undo |
| 索引维护 | 0 | 0 |
| 旧版本 | 物理共存堆中，等 VACUUM | undo log，purge thread 清理 |

HOT update 让 PG 在"非索引列更新"场景与 InnoDB 性能持平。

### 8.4 大字段对比

| 维度 | PostgreSQL TOAST | InnoDB BLOB |
|---|---|---|
| 阈值 | ~2 KB | ~半页（~8 KB） |
| 独立存储 | 独立 TOAST 表（每张主表对应一张） | off-page 页（同表空间） |
| 压缩 | pglz / lz4（PG 14+） | 无（应用层压缩） |
| 索引子串 | 可以（pg_trgm、btree on substring） | 受限 |
| MVCC | 旧 TOAST chunk 留待 VACUUM | undo log |
| 取部分字段 | 必须读完整 chunks 解压 | 类似 |

---

## 生产实践

1. **关键表设 fillfactor=80**。频繁更新表（用户表、订单状态表）务必降低 fillfactor 给 HOT 留余地。新表创建时直接指定，避免上线后改。
2. **监控 hot_pct**。`pg_stat_user_tables.n_tup_hot_upd / n_tup_upd > 80%` 是健康水位；持续 < 50% 说明设计有问题（索引列被更新、行太宽、fillfactor 太高）。
3. **避免在更新频繁列建索引**。"为了好查"加的索引可能让全表写吞吐降到 1/5。先确认查询频率，再决定是否值得。
4. **列顺序按对齐排**。大对齐字段在前（bigint / timestamp / text 头部），小对齐字段在后（smallint / bool）。10 亿行表能省几 GB。
5. **JSONB 默认就好，但 PG 14+ 改 lz4**。`ALTER SYSTEM SET default_toast_compression = 'lz4';` —— CPU 降 30%，吞吐升。
6. **大 BYTEA 用 EXTERNAL**。已经压过的二进制（图片、PDF）用 PLAIN/EXTERNAL，避免 PG 再做无用功的压缩。
7. **定期检查 TOAST 表大小**。TOAST 也会膨胀，VACUUM FULL 的时候记得它也会被重建（耗时翻倍）。
8. **不要追求"理想 fillfactor"**。表大小 < 1 GB 时 fillfactor 影响极小，不值得调；几十 GB+ 的热表才值得精调。
9. **用 pgstattuple 而不是 pg_stat_user_tables**。前者真实扫表给出精确膨胀率，后者只是估算（n_dead_tup 可能偏离实际几十倍）。
10. **`pg_repack` 替代 VACUUM FULL**。VACUUM FULL 持 ACCESS EXCLUSIVE 锁（停服务）；pg_repack 在线重建（短暂锁），生产首选。

---

## 陷阱清单

1. **CTID 不稳定还到处用**：基于 CTID 做去重、做分页（`WHERE ctid > '(100,0)'`）会在 VACUUM FULL 后崩。CTID 仅用于"当下"，跨事务不可靠。
2. **fillfactor 改完没生效**：只影响新页。改完要 `VACUUM FULL` 或 `pg_repack` 才全表生效。
3. **HOT chain 过长导致 SELECT 变慢**：连续 100+ 次 HOT update 形成 100 节点链，每次 SELECT 走到最新版本要追链。VACUUM 会折叠链，但 autovacuum 慢时链会拉长。
4. **TOAST 字段的 EXPLAIN 误导**：EXPLAIN 显示 1 行 1ms，实际读 JSONB 字段触发 50 次 TOAST chunk 读取，实际 50ms。EXPLAIN (ANALYZE, BUFFERS) 才能看到 TOAST 读取的 buffers。
5. **JSONB 字段 SELECT \*** **慢得离谱**：N 行 × 每行 TOAST 解压。改为只选需要字段（`SELECT data->>'name'`）能用 TOAST 的 chunk substring 优化。
6. **default_toast_compression 改了但旧数据不变**：只影响新插入。要让历史数据用新算法需 `VACUUM FULL` 或 `pg_repack`。
7. **NOT NULL 漏写**：可空列 + 实际全部有值 = 浪费一个 NULL bitmap byte（每 8 列一字节，看似小，但 1 亿行表浪费 12 MB+，且影响优化器）。
8. **`pg_column_size` 不包含 Tuple Header**：算总行大小要加 24 B + NULL bitmap + 对齐。
9. **`ALTER TABLE` 改列类型可能不需要重写表**：PG 12+ 同二进制兼容的类型（如 text → varchar(N)）不重写表；改 int → bigint 则要重写整表。提前 EXPLAIN 验证。
10. **TOAST 表 OID 不能直接 `DROP`**：必须先 DROP 主表，TOAST 自动消失。手动操作 pg_toast schema 是给自己挖坑。

---

## 2026 现状

- **PG 18 引入 `uuidv7()` 内置函数**：UUIDv7 时序友好（前 48 bit 是 unix epoch ms），用作主键时**不会像 UUIDv4 那样让索引随机插入**，HOT 友好度大幅提升。
- **PG 14+ TOAST lz4 默认推荐**：吞吐提升明显，但生产升级时注意：新数据 lz4 / 老数据 pglz，混合存在，pg_dump 跨版本恢复可能略变化。
- **PG 16+ vacuum 内存管理改进**：autovacuum 在大表上不再无谓占用 maintenance_work_mem 上限，缓解 HOT chain 清理延迟问题。
- **pg_repack 4.x**：与 PG 17/18 兼容，是 VACUUM FULL 的事实替代品。
- **fillfactor 在 SSD 时代仍有意义**：不是节省 IO（SSD 随机读廉价），而是**让 HOT update 成立**（节省索引膨胀、WAL 量、VACUUM 工作量）。
- **TOAST 在 AI 时代被压力测试**：长 prompt / embedding 数组（768/1536 维 float4 ≈ 3-6 KB）频繁触发 TOAST，pgvector 0.7+ 引入二进制量化部分缓解。
- **社区讨论中：行存可选 ZSTD 压缩**：PG 18 尚未合入，但 patch 已在 reviewing。

---

## 练习题

**1. 你的表 `users` 行平均 1.5 KB，单页能存几行？INSERT 100 万行后表多大？**

答案要点：8192 - 24（PageHeader）- 一些 ItemId（每行 4 B）≈ 8000 B 可用。1.5 KB / 行 → 一页 ~5 行。100 万行 / 5 = 20 万页 = 1.6 GB。考虑 fillfactor=100 和无空闲。

**2. `UPDATE users SET email='new@x' WHERE id=1`，email 上有索引，HOT update 会发生吗？为什么？**

答案要点：不会。HOT 的硬性条件是"修改的列没有任何索引"。email 有索引 → 必须更新索引指针 → 走普通 UPDATE，写新 tuple + 新索引项。

**3. 一张表 `n_tup_upd = 100 万`、`n_tup_hot_upd = 30 万`，HOT 比例 30%。三个可能原因？**

答案要点：(a) fillfactor 太高（100），页满了新版本只能写到别的页；(b) 更新了被索引的列；(c) 行太宽，新版本同页放不下。诊断顺序：先看 fillfactor → 再看哪些索引覆盖的列被更新 → 最后看行宽变化。

**4. `pg_column_size('hello'::text)` 返回什么？为什么不是 5？**

答案要点：返回 6（5 字节内容 + 1 字节变长短头）。长度 > 126 时变长头变 4 B，返回 5+4=9。`octet_length` 才是纯内容长度。

**5. 行内有一个 5 MB 的 JSONB 字段，存储策略 EXTENDED，pg_column_size 返回多少？解释。**

答案要点：返回 18（TOAST 指针大小）。5 MB 数据被压缩并切块存到 pg_toast.pg_toast_xxx 表，行内只有 18 字节的 `varatt_external` 结构（含 toast OID、原长度、压缩长度、value OID）。

**6. 你想知道某个 UPDATE 后 `(0,5)` 这条行被指向哪里。怎么查？**

答案要点：

```sql
SELECT lp, t_ctid, t_xmin, t_xmax,
       (t_infomask & 32) != 0 AS hot_updated,
       (t_infomask2 & 16384) != 0 AS heap_only_tuple
FROM heap_page_items(get_raw_page('users', 0));
```
查看 lp=5 的 t_ctid 字段，若指向 (0, 7) 表示后续版本在同页 ItemId 7；若指向其它块表示行被迁移。

**7. 100 GB 表 fillfactor 从 100 改到 80，怎么让历史数据生效？代价？**

答案要点：方法 1：`VACUUM FULL`，持 ACCESS EXCLUSIVE 锁，停服务，耗时与表大小成正比（百 GB 级要数小时）。方法 2：`pg_repack`，在线重建（创建新表 + 触发器同步 + swap），仅瞬时锁，但需要 ~2× 磁盘空间。生产几乎都选 pg_repack。

**8. PG 18 的 `uuidv7()` 为什么对 HOT 友好？UUIDv4 不行吗？**

答案要点：UUIDv4 完全随机 → 作为主键导致索引"到处插入"，几乎不会触发"同页 HOT 更新前提"（行会被分散到各种页）。UUIDv7 前 48 位是 unix epoch ms → 时序递增 → 新行追加到表尾、新索引项追加到 B-tree 右侧 → 与 bigserial 性能接近，并保留 UUID 全局唯一性。

---

## 延伸阅读

- 官方文档：
  - [Database Physical Storage](https://www.postgresql.org/docs/18/storage.html)
  - [TOAST](https://www.postgresql.org/docs/18/storage-toast.html)
  - [Heap-Only Tuples](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT)（README in source tree, must read）
- 必读书：《PostgreSQL 14 Internals》第 1 章（Tables, Tuples）+ 第 2 章（Bufferer）+ 第 6 章（HOT Updates）
- 源码路径：
  - `src/include/access/htup_details.h` ：HeapTupleHeader 完整结构
  - `src/backend/access/heap/heapam.c` ：插入/更新/删除主逻辑
  - `src/backend/access/heap/pruneheap.c` ：HOT chain 清理（page-level prune）
  - `src/backend/access/heap/heaptoast.c` ：TOAST 入口
  - `src/include/storage/bufpage.h` ：PageHeader / ItemId
- 扩展：`pageinspect`（看页字节）、`pgstattuple`（精确膨胀）、`pg_visibility`（VM 状态）、`pg_repack`（在线重建）
- 博客：
  - [Crunchy Data: HOT Updates Deep Dive](https://www.crunchydata.com/blog/heap-only-tuples-and-the-postgresql-mvcc-system)
  - [pganalyze: TOAST and JSON performance](https://pganalyze.com/blog/5mins-postgres-jsonb-toast-compression-lz4)
  - [Postgres.AI: pg_repack vs VACUUM FULL](https://postgres.ai/blog/20230223-postgresql-vacuum-full-bloat)
- 视频：PGCon 2023 - "Tuples, the Hard Way" by Andres Freund
