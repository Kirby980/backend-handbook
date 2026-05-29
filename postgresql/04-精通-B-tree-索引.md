# 精通 PostgreSQL B-tree 索引：页结构、INCLUDE 覆盖、Deduplication、Bottom-up Deletion

> 关联章节：[P02 堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md)、[P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)、[P05 多类型索引](./05-精通-多类型索引.md)、[P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)

---

## 引言：B-tree 是 PostgreSQL 索引宇宙的中心

如果 PostgreSQL 只能保留一种索引，那一定是 B-tree。它是 `CREATE INDEX` 不指定 `USING` 时的默认类型，是主键、唯一约束、外键校验的底层数据结构，也是绝大多数等值、范围、ORDER BY、LIMIT 优化的依据。生产环境里你会用到 GIN/GiST/BRIN/HNSW 等"花式索引"，但 B-tree 的占比通常仍在 80% 以上。

很多工程师对 PG B-tree 的认识停留在"和 MySQL B+ 树差不多"，这是一个**严重低估**：

- PG 的 B-tree 没有"聚簇索引"概念——索引和堆表是**完全分离**的两份存储
- PG 的二级索引指向 **CTID（堆中物理位置）**，而非 MySQL 的主键
- 因此 PG 索引扫描总是要**回堆**取行（heap fetch），即使是覆盖索引也得检查可见性
- PG 13 引入 **Deduplication**（重复键合并 posting list），让重复值密集的索引体积大幅缩小
- PG 14 引入 **Bottom-up Deletion**（叶子页级别的逻辑删除回收），缓解了 MVCC 导致的索引膨胀
- PG 11 起的 **INCLUDE 覆盖索引**让你能把"非键列"塞进叶子页用于 index-only scan
- PG 没有 hint，但有 `CREATE INDEX CONCURRENTLY` 在线建索引（**不阻塞 DML**），其工作原理是两阶段扫描 + 等待事务

这一章会从 PG 的 `src/backend/access/nbtree/` 源码出发，逐层拆解：

1. 一个 8KB B-tree 页的字节布局
2. 元页 / 根页 / 内部页 / 叶子页的分工
3. 高度估算与"为什么 PG B-tree 通常 3-4 层"
4. 唯一索引、NULL 的特殊处理（PG 把多个 NULL 视为**不重复**）
5. 表达式索引、部分索引——PG 的两个"杀器"
6. INCLUDE / Deduplication / Bottom-up Deletion 三个现代特性
7. `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY` 的内部机制
8. 索引膨胀的诊断与治理
9. PG vs MySQL：聚簇索引 vs 堆表 + 独立索引的根本差异

读完这一章你应该能：

- 在 `psql` 中用 `pageinspect` 扩展打开任意一个 B-tree 页看字节
- 解释为什么 PG 的二级索引比 MySQL 大得多（因为存 CTID 而非主键）
- 说出 INCLUDE 索引、Deduplication、Bottom-up Deletion 之间的关系
- 给一个 100GB 表估算建索引耗时和磁盘占用
- 看 `pg_stat_user_indexes` 发现"建了没用"的索引
- 在不停服的情况下重建一个 50GB 的膨胀索引

---

## 第一章：PostgreSQL B-tree 总览

### 1.1 为什么 PG 也选 B-tree

PostgreSQL 的 B-tree 实现源自 1990 年代 Lehman & Yao 的论文 *Efficient Locking for Concurrent Operations on B-Trees*（这就是为什么 PG 源码中叫 `nbtree`，N 表示"new"或者"non-blocking"）。它是一个**高并发版本**的 B+ 树变体，具备：

| 特性 | 说明 |
|---|---|
| **多叉低高度** | 每个节点 8KB，可存 数百-上千 个 key；3 层可管几亿行 |
| **叶子链表** | 叶子页之间双向链接，支持高效正/反向范围扫描 |
| **Lehman-Yao 高并发** | 通过"右兄弟链接 + high key"实现读不加锁 |
| **可扩展** | 通过 opclass 适配任意可比较类型（字符串、数组、自定义类型） |
| **WAL 支持** | 所有变更通过 WAL 持久化、可崩溃恢复、可复制 |
| **MVCC 感知** | 索引元组带有 LP_DEAD 提示位，配合 VACUUM 清理 |

PG B-tree 与 MySQL InnoDB B+ 树的最大形式差异：**叶子节点的 value 不是行数据，而是堆指针 CTID**（6 字节，由 4 字节 BlockNumber + 2 字节 OffsetNumber 组成）。

### 1.2 与 MySQL/InnoDB 的根本差异速览

| 维度 | PostgreSQL B-tree | MySQL InnoDB B+ 树 |
|---|---|---|
| 表的物理组织 | **堆表**（无聚簇索引）+ 独立 B-tree 索引 | **索引组织表**，主键索引即数据本身 |
| 主键索引叶子 | 存 `(主键, CTID)` | 存**整行数据** |
| 二级索引叶子 | 存 `(二级 key, CTID)` | 存 `(二级 key, 主键)` |
| 回表 | 必须（PG 没有"主键索引即数据"概念）| 主键索引不回表；二级索引需要回表 |
| Index-only scan | 需要满足 visibility map "全可见" | 主键索引天然 index-only |
| 重排数据 | `CLUSTER` 命令（一次性物理排序） | 主键决定物理顺序 |
| 页大小 | 8 KB（编译期可改 4-32 KB） | 16 KB（默认） |
| 主键缺省 | 没强制（可全表无主键，但生产强烈推荐有） | 必有主键（隐式 6 字节 ROW_ID） |
| 重复键处理 | PG 13+ deduplication 合并 | InnoDB 把 PK 作为 tie-breaker |
| NULL 在唯一索引 | 多个 NULL **不冲突**（标准 SQL 行为） | 多个 NULL **不冲突**（与 PG 一致） |
| 在线建索引 | `CREATE INDEX CONCURRENTLY`（两阶段扫描）| `ALTER TABLE ... ALGORITHM=INPLACE`（基于 online log） |

**结论**：PG 把"表"和"索引"看作两个相互独立的对象，这带来一种特别的灵活——同一张表可以有 6 种索引类型并存。但代价是：**所有索引扫描都要回堆**，因此 PG 比 MySQL 更依赖 `effective_cache_size` 和 OS page cache。

### 1.3 源码导航：`src/backend/access/nbtree/`

```
src/backend/access/nbtree/
├── nbtree.c            // AM (Access Method) 接口实现
├── nbtinsert.c         // 插入逻辑（含分裂、bottom-up deletion）
├── nbtsearch.c         // 搜索逻辑（_bt_search / _bt_first / _bt_next）
├── nbtpage.c           // 页操作（split / delete / page recycling）
├── nbtxlog.c           // WAL 解码与重放
├── nbtutils.c          // 工具函数
├── nbtsort.c           // 批量构建（CREATE INDEX）
├── nbtdedup.c          // Deduplication（PG 13+）
├── nbtcompare.c        // 内建类型的比较函数（btoidcmp / btint4cmp ...）
└── README              // **必读**：作者自述设计思路（含 Lehman-Yao 实现要点）
```

阅读建议：先看 `README`（这是 PG 源码里写得最详细的 README 之一），再按 `_bt_search → _bt_first → _bt_next → _bt_insertonpg → _bt_split` 的调用链跟进。

---

## 第二章：B-tree 页的内部结构（8 KB）

### 2.1 一切从 8 KB 页开始

PG 的所有存储都是页（page，又叫 block）为单位，B-tree 索引也不例外。`BLCKSZ` 默认 8192 字节（8 KB），编译时常量。

```c
#define BLCKSZ  8192    /* src/include/pg_config.h */
```

为什么是 8 KB 而非 InnoDB 的 16 KB？

- **历史原因**：1990 年代 UNIX 文件系统块多为 4-8 KB
- **WAL 友好**：单个 page 写入对应小一些的 WAL record
- **OS page cache 友好**：8KB = 2 个 Linux page（4KB）
- **代价**：树更高一点（因为每页存的 key 少）

**改不改 BLCKSZ？** 极少有人改。把它改成 16 KB 需要重新编译 PG，所有 PITR/复制兼容性都会受影响。生产环境基本默认 8 KB。

### 2.2 B-tree 页的总体布局

```
Offset 0                                                Offset 8192
+--------------------------------------------------------------+
| PageHeaderData    | 24 字节   pd_lsn / pd_checksum / pd_lower / pd_upper / ... |
+--------------------------------------------------------------+ 24
| ItemIdData[]      | line pointers（slot 数组，向下生长）       |
+--------------------------------------------------------------+ pd_lower
|                                                              |
|     free space   ← 向中间收拢                                 |
|                                                              |
+--------------------------------------------------------------+ pd_upper
| IndexTuple[]      | 真实的索引元组（向上生长）                  |
+--------------------------------------------------------------+ 8192 - sizeof(BTPageOpaqueData)
| BTPageOpaqueData  | 16 字节   B-tree 专属页尾（左右指针、level、flags） |
+--------------------------------------------------------------+ 8192
```

关键点：

- **ItemIdData 是 slot 数组**：每个 slot 4 字节，记录该索引元组在页内的偏移和长度。slot 数组**逆序对应** index tuple——slot[0] 指向 key 最小的元组，slot[N-1] 指向最大。
- **double-ended growth**：ItemIdData 从上往下写、IndexTuple 从下往上写、free space 在中间。这是 PG 的一个通用页布局，所有 access method 共享。
- **BTPageOpaqueData 是 B-tree 专属的**：存放左右兄弟指针、层级、标志位。

### 2.3 BTPageOpaqueData（B-tree 页尾元数据）

```c
/* src/include/access/nbtree.h */
typedef struct BTPageOpaqueData
{
    BlockNumber btpo_prev;     /* 左兄弟页号；INVALID 表示最左 */
    BlockNumber btpo_next;     /* 右兄弟页号；INVALID 表示最右 */
    uint32      btpo_level;    /* 层级：0=叶子，越大越靠根 */
    uint16      btpo_flags;    /* 标志位 */
    BTCycleId   btpo_cycleid;  /* VACUUM 用 */
} BTPageOpaqueData;

/* btpo_flags 关键位 */
#define BTP_LEAF        (1 << 0)   /* 叶子页 */
#define BTP_ROOT        (1 << 1)   /* 根页 */
#define BTP_DELETED     (1 << 2)   /* 已删除，等待回收 */
#define BTP_META        (1 << 3)   /* 元页（第 0 块） */
#define BTP_HALF_DEAD   (1 << 4)   /* 半死页（删除过程中间态） */
#define BTP_SPLIT_END   (1 << 5)   /* split 链尾（已完成） */
#define BTP_HAS_GARBAGE (1 << 6)   /* 有 LP_DEAD 标记的死元组 */
#define BTP_INCOMPLETE_SPLIT (1 << 7)  /* split 未完成 */
```

`btpo_prev / btpo_next` 把同一层的所有页串成双向链表——这就是 PG B-tree 的 **Lehman-Yao 关键设计**：读操作可以沿着 next 指针追上分裂中的页，不需要加锁。

### 2.4 元页（Metapage，Block 0）

每个 B-tree 索引的第 0 块永远是**元页**（Meta Page），它不存任何索引元组，只存"我这棵树的根在哪里、有几层"。

```c
typedef struct BTMetaPageData
{
    uint32      btm_magic;          /* 魔数 0x053162 */
    uint32      btm_version;        /* 版本号（PG 12+ = BTREE_VERSION 4） */
    BlockNumber btm_root;           /* 当前根页号 */
    uint32      btm_level;          /* 根页层级（即树高 - 1） */
    BlockNumber btm_fastroot;       /* 快速根（避免遍历多余的"瘦"层） */
    uint32      btm_fastlevel;
    /* PG 11+ 字段 */
    TransactionId btm_oldest_btpo_xact;
    float8      btm_last_cleanup_num_heap_tuples;
    /* PG 13+ 字段 */
    bool        btm_allequalimage;  /* 是否支持 deduplication */
} BTMetaPageData;
```

为什么需要元页？因为根页**可能会变**——当根分裂时，会产生一个新的根；元页提供了一个**稳定入口**让访问总能从这里出发。

### 2.5 根 / 内部 / 叶子页的分工

```
                 +---------------+
                 |  Metapage (0) |   永远是 block 0
                 +-------+-------+
                         | btm_root
                         v
                   +------------+
                   |  Root      |   level = N-1
                   +-+--------+-+
                     |        |
            +--------+        +--------+
            v                          v
       +---------+               +---------+
       | Internal|               | Internal|   level = N-2 ... 1
       +--+---+--+               +--+---+--+
          |   |                     |   |
     +----+   +----+           +----+   +----+
     v        v   v            v        v   v
   [Leaf]  [Leaf][Leaf] ... [Leaf]  [Leaf][Leaf]   level = 0
     ↓        ↓                       ↓        ↓
   (CTID)   (CTID)                 (CTID)   (CTID)  → 堆元组
     ← ← ← ← ← ← ← ← 双向链表 → → → → → → → → →
```

- **叶子页**（level 0）：存 `(index key, CTID)`，叶子之间双向链表
- **内部页**（level > 0）：存 `(下界 key, 子页号)`，每个 key 是"该子页的最小键"
- **根页**：内部页的特例，元页指向它

### 2.6 索引元组的格式（IndexTuple）

```c
/* src/include/access/itup.h */
typedef struct IndexTupleData
{
    ItemPointerData t_tid;     /* 6 字节：CTID (block, offset) */
    unsigned short  t_info;    /* 大小 + NULL 位图存在标志 */
    /* 然后是可选的 NULL bitmap */
    /* 然后是 key 列的实际数据 */
} IndexTupleData;
```

- `t_tid` **6 字节** = `BlockNumber (4B) + OffsetNumber (2B)`——这就是这个索引元组**指向的堆行**。在内部页中，`t_tid` 被复用为"指向下一层的页号"。
- `t_info` 高 3 位是 flags（含 `INDEX_NULL_MASK`），低 13 位是元组总大小（最大 8K）。

每个 IndexTuple 至少 8 字节（不含 key）。一个 8 KB 页大约能放：

- 全空元组：8192 / 8 = 1024 个（不现实）
- bigint key + CTID：8192 / (8 + 8) ≈ 512 个
- 64 字节 varchar key：8192 / (8 + 64) ≈ 113 个

### 2.7 用 pageinspect 实地查看页

`pageinspect` 是 PG 自带的 contrib 扩展，能让你直接读页字节：

```sql
CREATE EXTENSION pageinspect;

-- 看元页
SELECT * FROM bt_metap('idx_users_email');
--  magic  | version |  root  | level | fastroot | fastlevel | allequalimage
-- --------+---------+--------+-------+----------+-----------+---------------
--  340322 |       4 |     412|     2 |      412 |         2 | t

-- 看某一层的统计
SELECT * FROM bt_page_stats('idx_users_email', 412);
-- blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_prev | btpo_next | btpo_level | btpo_flags
-- ------+------+------------+------------+---------------+-----------+-----------+-----------+-----------+------------+-----------
--   412 | r    |          5 |          0 |          1432 |      8192 |       768 |         0 |         0 |          2 |          2

-- 看某页所有 items
SELECT itemoffset, ctid, itemlen, data FROM bt_page_items('idx_users_email', 412);
```

`type` 字段：`r` = root，`l` = leaf，`i` = internal，`d` = deleted。

---

## 第三章：高度估算 —— PG 8KB 页 vs MySQL 16KB 的影响

### 3.1 算一棵 B-tree 的高度

假设：

- 主键 `bigint`：8 字节 key + 8 字节 IndexTuple 头 = 16 字节
- 内部页一页能放 8192 / 16 = **512 个 key**
- 叶子页一行 IndexTuple 同样 16 字节，能放 **512 个元组**

| 树高 | 容量（行） |
|---|---|
| 1 层（只有叶子+根） | 512 |
| 2 层 | 512² = 26 万 |
| 3 层 | 512³ = 1.3 亿 |
| 4 层 | 512⁴ = 680 亿 |

**实际生产中 PG B-tree 高度通常 3-4 层**。即使 100 亿行表，索引也只需 4 次页 I/O 就能找到任一 key。

### 3.2 与 MySQL 16KB 页对比

MySQL InnoDB 16KB 页，相同假设下：

| 树高 | InnoDB（16KB）容量 | PG（8KB）容量 |
|---|---|---|
| 3 层 | 1170³ ≈ 16 亿 | 512³ ≈ 1.3 亿 |
| 4 层 | 1170⁴ ≈ 1.9 万亿 | 512⁴ ≈ 680 亿 |

InnoDB 用更大页换来更扁的树。表面看 PG"吃亏"，但实际：

1. **PG 叶子页只有 16 字节/索引元组，但堆行可能 100+ 字节** → 索引的总大小远小于表
2. **8KB 页和 OS page cache 对齐更好**
3. **多 1-2 层的代价是 1-2 次 buffer cache 命中**——上层页（根、内部）几乎永远在 cache 里

### 3.3 索引大小估算公式

```
索引大小 ≈ 行数 × (avg_key_len + 12) / fillfactor

12 字节 = 6 字节 CTID + 2 字节 t_info + 平均 4 字节对齐/null bitmap
fillfactor 默认 90%（B-tree）
```

例：1 亿行表，索引列是 `varchar(36) UUID`：

```
1e8 × (36 + 12) / 0.9 = 5.33 GB
```

例：同样 1 亿行，索引列是 `bigint`：

```
1e8 × (8 + 12) / 0.9 = 2.22 GB
```

**主键选择对索引大小有 2-3 倍影响**——这也是为什么 PG 18 引入 `uuidv7()`（时序友好）但仍推荐用 `bigint` 主键，除非有特殊分布式需求。

### 3.4 用 `pg_relation_size` 量索引

```sql
SELECT
  indexrelname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan AS scans,
  idx_tup_read AS tup_read,
  idx_tup_fetch AS tup_fetched
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

`idx_tup_read` vs `idx_tup_fetch`：

- `idx_tup_read`：索引扫描返回的元组数
- `idx_tup_fetch`：实际回堆取的行数（index-only scan 时这个值 = 0）

二者差距大 = 索引大量 LP_DEAD / VM 不可见，回堆很多。

---

## 第四章：唯一索引与 NULL 的处理

### 4.1 唯一索引的实现

`CREATE UNIQUE INDEX idx ON t(col)` 在结构上与普通 B-tree **完全一样**，只是插入时多一次"check before insert"：

```c
/* 简化版伪代码 src/backend/access/nbtree/nbtinsert.c */
_bt_doinsert(rel, itup, checkUnique) {
    stack = _bt_search(...);                    // 找到目标叶子
    if (checkUnique) {
        _bt_check_unique(rel, stack, itup);    // 在 page + 右兄弟扫描重复 key
    }
    _bt_insertonpg(...);                       // 实际插入
}
```

`_bt_check_unique` 的精彩之处：它必须**跨页扫描**，因为可能有等值键在右兄弟页上。同时它**对死元组（LP_DEAD）容忍**——VACUUM 清理过的可以复用 key。

### 4.2 NULL 在唯一索引中的特殊处理

**关键点：PostgreSQL 的唯一索引允许多个 NULL 共存**（这是 SQL 标准行为，MySQL 也一样）：

```sql
CREATE TABLE t(email TEXT UNIQUE);
INSERT INTO t VALUES (NULL), (NULL), (NULL);  -- 全部成功
SELECT count(*) FROM t;  -- 3
```

为什么？因为 SQL 标准中 `NULL = NULL` 的结果是**未知**（UNKNOWN），不是 TRUE，所以"唯一性"对 NULL 不生效。

### 4.3 PG 15+：NULLS NOT DISTINCT

PostgreSQL 15 引入了 `NULLS NOT DISTINCT` 子句，反转默认行为：

```sql
-- PG 15+
CREATE UNIQUE INDEX idx ON t(email) NULLS NOT DISTINCT;
-- 现在多个 NULL 视为冲突
INSERT INTO t VALUES (NULL);  -- OK
INSERT INTO t VALUES (NULL);  -- ERROR: duplicate key
```

适用场景：业务要求"未提供邮箱"也唯一（如手机号字段——一个用户必须给一个手机号，但允许部分用户没填手机号时只允许 1 个空）。

也可以通过部分索引达成相同效果（PG 14 及以前）：

```sql
CREATE UNIQUE INDEX idx_email ON t(email) WHERE email IS NOT NULL;
-- 联合一个：
CREATE UNIQUE INDEX idx_email_null ON t((1)) WHERE email IS NULL;
```

但语义略不同（部分索引允许多个非空 + 多个 NULL，PG 15 NULLS NOT DISTINCT 是"NULL 也唯一"）。

### 4.4 主键 = NOT NULL + UNIQUE + 唯一索引

`PRIMARY KEY` 在 PG 中是 `UNIQUE NOT NULL` 的语法糖，会**自动创建一个 B-tree 唯一索引**：

```sql
CREATE TABLE users(id BIGINT PRIMARY KEY);
-- 等价于：
-- CREATE TABLE users(id BIGINT NOT NULL);
-- CREATE UNIQUE INDEX users_pkey ON users(id);
-- ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY USING INDEX users_pkey;

\d users
--  Indexes:
--    "users_pkey" PRIMARY KEY, btree (id)
```

值得注意的是，PG 16+ 起允许 `ALTER TABLE ... ADD PRIMARY KEY USING INDEX` 把已存在的唯一索引"提升"为主键，避免重复扫描建索引。

---

## 第五章：多列索引与最左前缀

### 5.1 多列 B-tree 的存储顺序

```sql
CREATE INDEX idx ON orders(user_id, status, created_at);
```

索引按 `(user_id, status, created_at)` 三元组**字典序**排列。可以想象成：

> 按 user_id 升序；user_id 相同时按 status 升序；status 相同时按 created_at 升序。

### 5.2 最左前缀匹配规则

只有从最左列开始**连续等值**的查询能完整用上索引：

| 查询条件 | 用了索引几列 |
|---|---|
| `WHERE user_id = 1` | 1（user_id） |
| `WHERE user_id = 1 AND status = 'paid'` | 2（user_id, status） |
| `WHERE user_id = 1 AND status = 'paid' AND created_at > '2026-01-01'` | 3（全部） |
| `WHERE user_id = 1 AND created_at > '2026-01-01'` | 1（仅 user_id，created_at 在内存里过滤） |
| `WHERE status = 'paid'` | 0（不能用索引；除非 PG 17+ 的 skip scan） |
| `WHERE user_id = 1 AND status > 'a' AND created_at > '2026-01-01'` | 2（user_id 等值 + status 范围，created_at 在内存过滤） |

### 5.3 PG 17 Skip Scan（部分支持）

PostgreSQL 17 引入了**多列 B-tree skip scan**（也叫 loose index scan），允许跳过中间列的等值条件：

```sql
-- PG 17+
EXPLAIN SELECT * FROM orders
 WHERE status = 'paid' AND created_at > '2026-01-01';
-- 即使没指定 user_id，规划器会枚举所有 user_id 值进行"循环扫描"
-- 适合 user_id 基数小（< 1000）的场景
```

这部分支持目前还在演进——PG 18 进一步扩展了 skip scan 能力。但**不要依赖**它：设计索引时仍应优先考虑最左前缀。

### 5.4 列顺序设计原则

```
等值条件列 → 排序/分组列 → 范围条件列 → 仅过滤的列（候选 INCLUDE）
```

举例：

```sql
-- 业务：查某用户某状态下按时间倒序的订单，并返回订单金额
SELECT id, amount FROM orders
 WHERE user_id = ? AND status = 'paid'
 ORDER BY created_at DESC LIMIT 20;

-- 推荐索引（PG 11+）：
CREATE INDEX idx_orders_user_status_time
  ON orders (user_id, status, created_at DESC)
  INCLUDE (amount);
--   ↑ 等值 ↑ 等值 ↑ 排序          ↑ 仅过滤/返回，进 INCLUDE
```

### 5.5 ORDER BY 的索引利用

B-tree 索引天然有序，可以同时支持升降序扫描：

```sql
-- 索引 (a ASC, b DESC)
ORDER BY a ASC, b DESC      -- 用索引（正向扫）
ORDER BY a DESC, b ASC      -- 用索引（反向扫）
ORDER BY a ASC, b ASC       -- 不能直接用，需要排序
```

PG 的 B-tree 支持任意 `ASC`/`DESC` + `NULLS FIRST`/`NULLS LAST` 的组合：

```sql
CREATE INDEX idx ON t(a ASC NULLS FIRST, b DESC NULLS LAST);
```

但只有**正向**或**完全反向**扫描能直接利用——混合方向需要额外排序。

---

## 第六章：INCLUDE 覆盖索引（PG 11+）

### 6.1 什么是 INCLUDE

PostgreSQL 11 引入了 `INCLUDE` 子句，让你把"非键列"塞进索引的**叶子页**：

```sql
CREATE INDEX idx ON orders(user_id, status) INCLUDE (amount, created_at);
```

- `(user_id, status)` 是**键列**，参与 B-tree 排序、可用于 WHERE/ORDER BY
- `(amount, created_at)` 是**包含列**，**只**存在于叶子页，不参与排序、不能用于 WHERE

### 6.2 INCLUDE 解决的问题

```sql
-- 需求：查某用户某状态的所有订单金额
SELECT amount FROM orders WHERE user_id = 42 AND status = 'paid';

-- 索引 A：CREATE INDEX ON orders(user_id, status);
-- 执行：索引扫 → 拿 CTID → 回堆取 amount → 返回
-- 每行至少 2 次 I/O（索引 + 堆）

-- 索引 B：CREATE INDEX ON orders(user_id, status) INCLUDE (amount);
-- 执行：索引扫 → 直接拿 amount → 返回（如果 VM 全可见，免回堆）
-- 每行 1 次 I/O，可触发 Index-Only Scan
```

### 6.3 INCLUDE vs 把列加进键

为什么不直接 `CREATE INDEX ON orders(user_id, status, amount)`？

| 方案 | 优点 | 缺点 |
|---|---|---|
| **(user_id, status, amount)** 全键 | 可按 amount 范围查询/排序 | amount 进入排序、影响 split；唯一索引时影响唯一性判定 |
| **(user_id, status) INCLUDE (amount)** | 不影响排序、不参与唯一性、内部页更瘦 | amount 不能用于 WHERE/ORDER BY |

**典型场景：唯一索引 + 想覆盖额外列**

```sql
-- 错误：把 email 加入唯一约束会让唯一性以 (user_id, email) 判定
CREATE UNIQUE INDEX ON users(user_id, email);

-- 正确：唯一性仅按 user_id，但叶子页带上 email
CREATE UNIQUE INDEX ON users(user_id) INCLUDE (email);
```

### 6.4 INCLUDE 列的内部存储

INCLUDE 列只存在叶子页的 IndexTuple 中，**内部页不存**。这样的设计：

- 内部页保持瘦（高度不变）
- 叶子页变胖（每个元组多了 INCLUDE 列的字节）

举例：原索引每元组 16 字节，加 `INCLUDE (amount BIGINT, created_at TIMESTAMP)` 后每元组 32 字节，叶子页能放的元组数减半，**索引整体大小约翻倍**。

### 6.5 Index-Only Scan 与 Visibility Map

PG 的 Index-Only Scan **需要 VM（Visibility Map）配合**——VM 标记"这一页所有元组对所有事务都可见"，才能跳过回堆。

```
对每行：
  if (VM.all_visible(ctid.block)) {
      return index_tuple_value;     // ← Index-Only！
  } else {
      heap_tuple = heap_fetch(ctid);
      if (visible_to_snapshot(heap_tuple))
          return heap_tuple_value;
  }
```

`pg_stat_user_indexes.idx_tup_fetch = 0` 时说明该索引最近的扫描全部走了 Index-Only Scan，效率最高。

**保持 VM 新鲜**靠 VACUUM——所以"索引能不能 index-only"也是 VACUUM 是否及时的问题。

### 6.6 EXPLAIN 看 Index-Only Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT amount FROM orders WHERE user_id = 42 AND status = 'paid';

-- Index Only Scan using idx_orders_user_status on orders
--   Index Cond: ((user_id = 42) AND (status = 'paid'))
--   Heap Fetches: 0                                    ← 全部命中 VM
--   Buffers: shared hit=4
```

`Heap Fetches: 0` 是完美 index-only。如果 `Heap Fetches > 0` 说明部分行 VM 未标记可见，仍回了堆。

---

## 第七章：Deduplication（PG 13+）

### 7.1 重复键的痛点

设想一个索引 `CREATE INDEX ON orders(status)`，`status` 只有 5 个枚举值。1 亿行表中有 2000 万行 `status = 'paid'`。

PG 12 及之前：每行一个 IndexTuple，**2000 万行 = 2000 万个 IndexTuple**，每个 16 字节 → 索引膨胀。

PG 13 引入 **Deduplication**：连续相同 key 的元组被合并成一个 **posting list** 元组：

```
PG 12：[paid, ctid1][paid, ctid2][paid, ctid3]...[paid, ctid20M]
       ↑ 20M 个 IndexTuple

PG 13+：[paid, posting_list(ctid1, ctid2, ctid3, ..., ctid20M)]
        ↑ 1 个 posting tuple，内含 20M 个 CTID
```

### 7.2 Posting List 元组的结构

```c
/* posting list 元组共享一个 t_tid，但用高位 flag 标记 */
#define INDEX_ALT_TID_MASK     0x2000   /* t_info 高位：这是 posting 元组 */
/* 然后 t_tid 后面紧跟一个数组：TID[1], TID[2], ..., TID[N] */
```

特点：

- 同 key 的多个 CTID 合并到一个 IndexTuple
- 一个 posting list 最多包含 **441 个 TID**（PG 13），超过会拆成多个
- 完全向后兼容：旧版本能读，但不会主动合并

### 7.3 何时触发 Deduplication

**插入时不会**主动 dedup——因为开销大。Deduplication 在以下情况触发：

1. **页接近满**，将要分裂前——尝试 dedup 看能不能省下分裂
2. **bottom-up deletion 触发时**（PG 14+）的辅助过程

```c
/* src/backend/access/nbtree/nbtdedup.c */
_bt_dedup_pass(rel, page, ...) {
    /* 扫描叶子页所有元组 */
    /* 对相同 key 的连续元组合并 */
    /* 把空间还给页 */
}
```

### 7.4 Deduplication 的效果

实测案例：

```sql
-- 表：1 亿行，status 5 个枚举值（80% 是 'pending'）
-- 索引：(status) B-tree

-- PG 12（无 dedup）
SELECT pg_size_pretty(pg_relation_size('idx_status'));
-- 2.1 GB

-- PG 13+（dedup 后）
SELECT pg_size_pretty(pg_relation_size('idx_status'));
-- 380 MB   ← 5.5x 缩小
```

**适用判定**：

- 重复值越多，效果越好（10 重复以上明显）
- 唯一索引、布尔索引、低基数列受益最大
- 数字、UUID 等几乎没重复的索引，dedup 没意义

### 7.5 是否启用 Deduplication

默认所有支持的 opclass 自动启用。可在建索引时显式关闭：

```sql
CREATE INDEX idx ON t(col) WITH (deduplicate_items = off);
```

什么情况关闭？

- 索引值高基数（接近全唯一），dedup 浪费 CPU
- 用一些不支持 dedup 的自定义类型（不常见）

可通过 `pg_index.indoption` 或 `bt_metap('idx').allequalimage` 查看是否启用。

### 7.6 Dedup 与 Bottom-up Deletion 的关系

二者是 PG 13/14 "B-tree 现代化"的连环组合：

- **Dedup（13）**：减少**新插入**带来的 IndexTuple 数
- **Bottom-up Deletion（14）**：清理 MVCC 留下的**旧 IndexTuple**

PG 13 之前，UPDATE 即使 HOT-eligible，只要修改的列在索引上就会产生新 IndexTuple → 旧 IndexTuple 由 VACUUM 异步清理 → 在 VACUUM 跟不上时索引疯狂膨胀。Dedup + Bottom-up Deletion 大幅缓解了这个问题。

---

## 第八章：Bottom-up Index Deletion（PG 14+）

### 8.1 MVCC 导致的索引膨胀

复习 MVCC：UPDATE 在 PG 中**不是真正更新**，而是"插入新行 + 标记旧行 dead"（除非 HOT update）。每次 UPDATE 涉及索引列时：

```
原行：(id=1, name='Alice', age=30)
UPDATE users SET age = 31 WHERE id = 1;

堆：旧元组 (xmin=t1, xmax=t2, age=30)  ← 死元组
   新元组 (xmin=t2, xmax=∞,  age=31)

索引：旧索引项 (age=30) → 旧 CTID
     新索引项 (age=31) → 新 CTID    ← 多了一个！
```

如果 `age` 上有 5 个索引，每次 UPDATE 会产生**5 个新索引项**。索引膨胀比表更快。

### 8.2 Bottom-up Deletion 的核心思想

PG 14 引入的 bottom-up deletion：**当一个叶子页即将分裂时，先尝试清理 LP_DEAD 死元组和"可推断为死"的元组，可能就不需要分裂了**。

```c
/* src/backend/access/nbtree/nbtinsert.c  _bt_findinsertloc */
if (page_almost_full) {
    /* 触发 bottom-up deletion */
    _bt_simpledel_pass(rel, page);  // 简单删除已知 LP_DEAD
    _bt_bottomupdel_pass(rel, page); // 高级版：跨页推断"可能死"的元组
}
```

**关键创新**：bottom-up deletion 可以**主动询问堆**——"这个 CTID 指向的堆元组是不是死的？"，然后批量清理对应索引项。

### 8.3 LP_DEAD 提示位

每个 IndexTuple 的 line pointer 有一个 `LP_DEAD` 位：

```c
#define LP_DEAD     2   /* dead, may be replaced */
```

当索引扫描读到这个元组、并且发现堆上对应行**对当前快照不可见且 xmax 已提交**时，会**顺手**把 LP_DEAD 设上。这是一个"乘客式"清理：

```
SELECT * FROM users WHERE name = 'Alice';
→ 扫到索引项 (name='Alice', ctid=(5,3))
→ 回堆读 (5,3)：发现 xmax 已提交且 < 当前快照 xmin
→ 这个堆元组对所有事务都死了！
→ 顺手把索引项的 LP_DEAD 设上（不需要 WAL，性能高）
```

下次 bottom-up deletion / VACUUM 看到这些 LP_DEAD 就清理。

### 8.4 Bottom-up Deletion vs VACUUM

| 维度 | VACUUM | Bottom-up Deletion |
|---|---|---|
| 触发 | 后台周期 / 显式调用 | 叶子页即将分裂时 |
| 范围 | 整个表的所有索引 | **单个叶子页** |
| 开销 | 大（扫整表/整索引） | 小（一页 + 部分堆查找） |
| 清理对象 | 所有死元组 + 所有死索引项 | 当前叶子页 LP_DEAD + 可推断为死的元组 |
| 引入版本 | 一直有 | PG 14+ |

二者**配合工作**：

- Bottom-up deletion 解决"VACUUM 跟不上时局部页膨胀"
- VACUUM 解决"全局清理"

### 8.5 实测效果

PG 14 release notes 提到：

> Bottom-up deletion can prevent index bloat by 80%+ in some UPDATE-heavy workloads.

实战观察：

- 频繁 UPDATE 同一行（如计数器）的索引膨胀几乎消失
- 重复键多的索引（dedup + bottom-up 联动）尤其受益
- 全表大批量 UPDATE 仍会膨胀（因为大量页同时分裂，bottom-up 来不及）

### 8.6 监控 Bottom-up Deletion

`pg_stat_user_indexes` 没有直接指标，但可以通过 `pgstattuple` 看膨胀程度：

```sql
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('idx_users_email');
--  table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count | dead_tuple_len | dead_tuple_percent | free_space | free_percent
-- -----------+-------------+-----------+---------------+------------------+----------------+--------------------+------------+--------------
--    8388608 |        5023 |    160736 |          1.92 |               12 |            384 |               0.00 |    1245312 |        14.85
```

`free_percent + dead_tuple_percent < 30%` 通常健康。> 50% 应该考虑 `REINDEX CONCURRENTLY`。

---

## 第九章：表达式索引

### 9.1 什么是表达式索引

PG 允许在**任意表达式**上建索引，而不只是列：

```sql
CREATE INDEX idx_lower_email ON users(lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com';  -- 用索引
```

被索引的不是 `email` 列，而是 `lower(email)` 的计算结果。每次 INSERT/UPDATE 会计算表达式、存进索引。

### 9.2 用途场景

#### 场景 1：大小写不敏感搜索

```sql
CREATE INDEX idx_lower_email ON users(lower(email));
SELECT * FROM users WHERE lower(email) = lower('Alice@EXAMPLE.com');
```

#### 场景 2：JSON 字段索引（PG JSONB 的核心用法）

```sql
CREATE TABLE events(id BIGSERIAL PRIMARY KEY, data JSONB);

-- 索引 data 中 'user_id' 路径
CREATE INDEX idx_data_user ON events((data ->> 'user_id'));

SELECT * FROM events WHERE data ->> 'user_id' = '42';  -- 用索引
```

#### 场景 3：函数索引（trigram、bm25 等）

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_email_trgm ON users USING GIN (email gin_trgm_ops);
-- 严格说这是 GIN + 表达式（运算符类），但概念相通
```

#### 场景 4：计算列

```sql
CREATE INDEX idx_age ON users((date_part('year', age(birth_date))));
SELECT * FROM users WHERE date_part('year', age(birth_date)) > 18;
```

### 9.3 表达式必须是 IMMUTABLE

被索引的表达式必须是 **IMMUTABLE** 函数——给定相同输入永远返回相同输出。

```sql
-- 报错：now() 是 STABLE，不是 IMMUTABLE
CREATE INDEX ON events((now() - created_at));
-- ERROR: functions in index expression must be marked IMMUTABLE

-- 也报错：to_char(timestamp, 'YYYY-MM-DD') 受 lc_time / DateStyle 影响，被标 STABLE
CREATE INDEX ON events((to_char(created_at, 'YYYY-MM-DD')));
```

变通方案：自己包一层标 IMMUTABLE（**自担风险**——如果包错了会查询失败）：

```sql
CREATE FUNCTION my_to_date(t TIMESTAMPTZ)
RETURNS DATE LANGUAGE SQL IMMUTABLE
AS $$ SELECT t::DATE $$;

CREATE INDEX ON events(my_to_date(created_at));
```

### 9.4 PG 18 虚拟生成列 + 表达式索引

PG 12 引入 STORED 生成列，PG 18 引入 VIRTUAL 生成列。结合表达式索引能做出很 elegant 的设计：

```sql
-- PG 18+
CREATE TABLE events(
  data JSONB,
  user_id INT GENERATED ALWAYS AS ((data ->> 'user_id')::INT) VIRTUAL
);
CREATE INDEX ON events(user_id);
SELECT * FROM events WHERE user_id = 42;  -- 优雅
```

虚拟生成列**不占存储**，索引时按需计算。

---

## 第十章：部分索引

### 10.1 什么是部分索引

只索引**满足某条件**的行：

```sql
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

只有 `status = 'active'` 的行进入这个索引。其他行（如 status = 'deleted'）不占索引空间。

### 10.2 用途场景

#### 场景 1：去除大量"无意义"行

```sql
-- 100M 用户，99% deleted/banned，只有 1M active
CREATE INDEX idx_email ON users(email) WHERE status = 'active';
-- 索引大小：100x 缩小
```

#### 场景 2：唯一约束限定范围

```sql
-- 同一 user_id 下，只能有 1 个 status='active' 订单
CREATE UNIQUE INDEX ON orders(user_id) WHERE status = 'active';
-- 其他 status 的可以多个
```

#### 场景 3：NULL 处理

```sql
-- 只索引非空 email（与 PG 15 NULLS DISTINCT 不同）
CREATE INDEX ON users(email) WHERE email IS NOT NULL;
```

#### 场景 4：按时间窗口热数据

```sql
-- 只索引最近 30 天，旧数据靠分区/全表扫
CREATE INDEX idx_recent ON events(user_id)
  WHERE created_at > '2026-04-28';
-- 注意：WHERE 必须是 IMMUTABLE，所以不能用 now() - interval '30 days'
-- 需要定期重建索引
```

### 10.3 规划器如何用部分索引

规划器会**证明**查询条件**蕴含**索引的 WHERE 条件，才会使用：

```sql
CREATE INDEX idx ON t(a) WHERE status = 'active';

-- 用：因为 WHERE status = 'active' 已经匹配
SELECT * FROM t WHERE status = 'active' AND a = 1;

-- 不用：规划器无法证明 status 一定是 'active'
SELECT * FROM t WHERE a = 1;

-- 用：因为 status IN ('active') 蕴含 status = 'active'
SELECT * FROM t WHERE status IN ('active') AND a = 1;
```

### 10.4 部分索引 + 表达式索引

二者可以叠加：

```sql
CREATE INDEX ON orders(lower(buyer_email))
 WHERE status = 'paid' AND created_at > '2026-01-01';
```

适合"热数据 + 灵活搜索"。

### 10.5 部分索引的限制

- WHERE 必须 IMMUTABLE
- 不能用于强制唯一性的主键（PG 不允许 PRIMARY KEY 是 partial）
- 无法被外键 REFERENCES 使用

---

## 第十一章：CREATE INDEX CONCURRENTLY

### 11.1 普通 CREATE INDEX 的问题

```sql
CREATE INDEX idx ON huge_table(col);
```

会获取表的 `SHARE` 锁——**阻塞所有 UPDATE / INSERT / DELETE**（SELECT 仍可），直到索引建完。在一个 100GB 表上可能持续小时级。

### 11.2 CONCURRENTLY 的工作原理

`CREATE INDEX CONCURRENTLY`（CIC）通过**两阶段扫描 + 等待事务**实现"不阻塞 DML"：

```
阶段 1：扫描表第一遍
  - 取 SHARE UPDATE EXCLUSIVE 锁（与 DML 不冲突，但与其他 DDL 冲突）
  - 拿一个 snapshot S1
  - 全表扫描，把 S1 可见的所有行加入索引
  - 此期间发生的 DML（>= S1 之后的事务）不在索引里

阶段 2：等待 S1 之前的所有事务结束
  - 等待所有 "S1 开始时尚未结束" 的事务提交或回滚
  - 保证：之后启动的事务都能看到"索引存在"这件事

阶段 3：扫描表第二遍
  - 拿一个新 snapshot S2
  - 扫描所有"在 S1 之后被修改过的行"，加入索引
  - 此时索引已包含所有 S2 可见的行

阶段 4：等待 S2 之前的事务结束
  - 防止有些事务以 S2 之前的 snapshot 启动、不知道索引存在

阶段 5：将索引标记为 valid，可供查询使用
```

**关键点**：CIC 全程不阻塞 DML，但**会阻塞别的 DDL**。CIC 也会**慢得多**——比普通 CREATE INDEX 慢 2-3 倍（要扫两遍 + 等事务）。

### 11.3 CIC 的失败处理

如果 CIC 中途失败（系统重启、CTRL-C、约束冲突），索引会进入 `INVALID` 状态：

```sql
\d users
--  Indexes:
--    "idx_email" btree (email) INVALID

-- 解决方案：
DROP INDEX CONCURRENTLY idx_email;
CREATE INDEX CONCURRENTLY idx_email ON users(email);

-- 或者尝试 REINDEX：
REINDEX INDEX CONCURRENTLY idx_email;
```

### 11.4 CIC 的关键限制

| 限制 | 详情 |
|---|---|
| 不能在事务中 | CIC 自身就是一个隐式事务，不能嵌入到 BEGIN...COMMIT |
| 不能并发 CIC | 同一表上的两个 CIC 会冲突 |
| 比普通 CIC 慢 2-3x | 两次扫描 |
| 失败留 INVALID | 必须手动清理 |
| 阻塞其他 DDL | ALTER TABLE 等会等 CIC 完成 |

### 11.5 实战：在生产 100GB 表上建索引

```sql
-- 1. 先在 replica 上测试估算时间
EXPLAIN (BUFFERS) SELECT count(*) FROM huge_table;  -- 看页数

-- 2. 主库执行（建议在低峰期）
CREATE INDEX CONCURRENTLY idx_email ON huge_table(email);
-- 这会运行很久（可能小时级），但不阻塞业务

-- 3. 监控进度（PG 12+）
SELECT
  pid, phase, blocks_done, blocks_total,
  tuples_done, tuples_total, partitions_done, partitions_total
FROM pg_stat_progress_create_index;

-- 4. 验证
\d huge_table
-- 确认索引不是 INVALID 状态
SELECT indexrelname, indisvalid FROM pg_stat_user_indexes
  JOIN pg_index USING (indexrelid)
WHERE schemaname = 'public';
```

---

## 第十二章：REINDEX 与索引膨胀治理

### 12.1 索引膨胀的来源

1. **大量 UPDATE/DELETE**：留下 LP_DEAD，VACUUM 跟不上
2. **TXID wraparound 前 anti-wraparound vacuum**：可能短期不清理 dead 索引项
3. **bulk load 后 fillfactor 不合理**：批量插入产生大量分裂
4. **VACUUM 的 dead_tuples 内存不够**：要扫多次堆才能回收索引

### 12.2 诊断膨胀

```sql
-- pgstattuple 扩展
CREATE EXTENSION pgstattuple;

SELECT pg_size_pretty(pg_relation_size('idx_users_email')) AS size,
       leaf_fragmentation, avg_leaf_density
FROM pgstatindex('idx_users_email');
--    size   | leaf_fragmentation | avg_leaf_density
-- ----------+--------------------+------------------
--    2.1 GB |              45.32 |            48.21
--   ↑ 健康值：fragmentation < 30，density > 70
```

`avg_leaf_density < 50` 通常表示明显膨胀，建议重建。

也可以用纯 SQL 查询估算（社区有现成脚本，比如 `btree_bloat.sql`，pganalyze 的脚本）。

### 12.3 REINDEX vs REINDEX CONCURRENTLY

```sql
-- 阻塞写，但快
REINDEX INDEX idx_email;

-- PG 12+ 不阻塞写，但慢 2-3x，需要额外磁盘空间
REINDEX INDEX CONCURRENTLY idx_email;

-- 重建整个表所有索引
REINDEX TABLE CONCURRENTLY users;

-- 重建整个 schema
REINDEX SCHEMA CONCURRENTLY public;

-- 重建整个数据库
REINDEX DATABASE CONCURRENTLY mydb;
```

### 12.4 REINDEX CONCURRENTLY 的工作原理

类似 CIC：

```
1. 用 CIC 模式建一个"影子索引"（idx_email_ccnew）
2. 等待事务、刷新 VM
3. 原子地 swap：把 idx_email 重命名为 idx_email_ccold，把 idx_email_ccnew 重命名为 idx_email
4. 等待依赖 idx_email_ccold 的查询结束
5. 删除 idx_email_ccold
```

需要的额外磁盘空间：**约等于原索引大小**（因为新旧索引同时存在）。

### 12.5 自动化膨胀监控

生产建议设置 cron / pg_cron 任务每周扫描：

```sql
WITH bloated_indexes AS (
  SELECT
    schemaname || '.' || indexrelname AS idx,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size,
    pgstatindex(indexrelid::text)
  FROM pg_stat_user_indexes
  WHERE pg_relation_size(indexrelid) > 1 * 1024 * 1024 * 1024   -- > 1 GB
)
SELECT * FROM bloated_indexes
WHERE (pgstatindex).avg_leaf_density < 50
ORDER BY pg_relation_size(idx::regclass) DESC;
```

---

## 第十三章：与 MySQL B+ 树的核心差异

### 13.1 聚簇索引 vs 堆表 + 独立索引

| 维度 | PostgreSQL | MySQL/InnoDB |
|---|---|---|
| 数据组织 | 堆表（heap），无序 | 索引组织表，主键索引即数据 |
| 主键索引叶子 | 仅 `(主键, CTID)` | 完整行数据 |
| 二级索引叶子 | `(二级 key, CTID)` | `(二级 key, 主键)` |
| 二级索引回表 | 回堆（通过 CTID）| 回主键索引（通过主键） |
| 主键查询 | 索引扫 + 1 次堆 I/O | 索引扫即数据 |
| Index-Only Scan | 需要 VM 全可见 | 主键索引天然 index-only |
| 物理顺序 | 任意（除非 CLUSTER） | 按主键升序 |

### 13.2 实战影响

#### 影响 1：PG 的主键查询多 1 次 I/O

```sql
-- PG: SELECT * FROM users WHERE id = 42;
-- 1. B-tree 找到 id=42 → CTID = (5, 3)
-- 2. heap_fetch((5, 3)) → 完整行
-- 共 2 次 I/O（理论）

-- MySQL: SELECT * FROM users WHERE id = 42;
-- 1. 主键 B+ 树找到 id=42 → 叶子页直接返回完整行
-- 共 1 次 I/O
```

但实际 PG 主键查询并不比 MySQL 慢——因为：

1. 索引页和堆页都很可能在 buffer pool 里
2. PG 的 buffer 替换算法（clock-sweep）对热索引页友好
3. 现代 SSD 上单次 I/O 几乎可忽略

#### 影响 2：PG 二级索引比 MySQL 大

PG 二级索引存 `(key, CTID)` = `key + 6B`；
MySQL 二级索引存 `(key, 主键)` = `key + 8B`（假设 bigint 主键）。

看似 PG 小？**不**——MySQL 主键即数据，所以主键索引很大；PG 主键索引和数据分开，**总大小（数据 + 索引）通常 PG 更大**。

#### 影响 3：PG 范围扫描没有物理顺序保证

```sql
-- PG: 1 亿行表，建表时 id=1 行可能在文件结尾
SELECT * FROM users WHERE id BETWEEN 1 AND 100;
-- 索引按 id 升序找到 100 个 CTID
-- 但这 100 个 CTID 可能散布在堆的任意位置 → 随机 I/O

-- MySQL: 数据按主键物理有序
SELECT * FROM users WHERE id BETWEEN 1 AND 100;
-- 连续读，顺序 I/O
```

**对策**：

- PG 有 `CLUSTER` 命令一次性按某索引重排数据（但之后又会乱）
- 配合 `pg_repack` 在线重排
- 或用分区表按时间分区，间接维持顺序

### 13.3 PG 18 异步 IO 对索引扫描的影响

PG 18 引入异步 IO（io_uring）后，对**索引扫描后的回堆**带来巨大改进：

- 以前：每个 CTID 串行 heap_fetch，等 I/O
- PG 18：批量提交 I/O 请求，io_uring 内核并发执行

实测显示，BitmapHeapScan 在 io_uring 后吞吐提升 2-5 倍。这有效缓解了 PG"堆 + 独立索引"模型的一部分劣势。

---

## 第十四章：完整 SQL 与 Go pgx 示例

### 14.1 建索引完整流程

```sql
-- 1. 准备一个测试表
CREATE TABLE orders (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT NOT NULL,
  status      TEXT NOT NULL CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
  amount      NUMERIC(12, 2) NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
INSERT INTO orders (user_id, status, amount)
SELECT
  (random() * 1000000)::BIGINT,
  (ARRAY['pending', 'paid', 'shipped', 'cancelled'])[(random() * 4)::INT + 1],
  (random() * 1000)::NUMERIC(12, 2)
FROM generate_series(1, 10000000);

-- 2. 基础索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 3. 联合索引 + INCLUDE
CREATE INDEX CONCURRENTLY idx_orders_user_status_time
  ON orders(user_id, status, created_at DESC)
  INCLUDE (amount);

-- 4. 部分索引：只索引 'pending' 订单（占比小，常被高频访问）
CREATE INDEX CONCURRENTLY idx_orders_pending
  ON orders(created_at)
  WHERE status = 'pending';

-- 5. 表达式索引
CREATE INDEX CONCURRENTLY idx_orders_year_month
  ON orders((to_char(created_at, 'YYYY-MM')));

-- 6. 唯一索引
CREATE UNIQUE INDEX CONCURRENTLY idx_orders_unique_idem
  ON orders(user_id, status) WHERE status = 'pending';

-- 7. 验证
ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, amount FROM orders
WHERE user_id = 42 AND status = 'paid'
ORDER BY created_at DESC LIMIT 20;
```

### 14.2 Go pgx 客户端示例

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

type Order struct {
    ID        int64
    UserID    int64
    Status    string
    Amount    float64
    CreatedAt time.Time
}

func main() {
    ctx := context.Background()
    pool, err := pgxpool.New(ctx, "postgres://user:pass@localhost:5432/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()

    // 1. 联合索引 + INCLUDE 适合的查询
    rows, err := pool.Query(ctx, `
        SELECT id, amount, created_at
          FROM orders
         WHERE user_id = $1 AND status = $2
         ORDER BY created_at DESC LIMIT 20`,
        42, "paid")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    for rows.Next() {
        var o Order
        if err := rows.Scan(&o.ID, &o.Amount, &o.CreatedAt); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("Order: %+v\n", o)
    }

    // 2. 用 Prepared Statement 命中 prepared plan
    _, err = pool.Exec(ctx, `
        PREPARE get_orders AS
        SELECT id, amount FROM orders
         WHERE user_id = $1 AND status = $2
         ORDER BY created_at DESC LIMIT 20`)
    if err != nil {
        log.Fatal(err)
    }

    // 3. 索引膨胀监控
    var size int64
    var leafDensity float64
    err = pool.QueryRow(ctx, `
        SELECT pg_relation_size('idx_orders_user_status_time'),
               (pgstatindex('idx_orders_user_status_time')).avg_leaf_density`).
        Scan(&size, &leafDensity)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Index size: %d MB, leaf density: %.2f%%\n", size/1024/1024, leafDensity)

    if leafDensity < 50 {
        log.Println("Index needs REINDEX CONCURRENTLY")
    }
}
```

### 14.3 在线 REINDEX 脚本

```go
// 生产可用的 REINDEX CONCURRENTLY 调度器
func ReindexBloatedIndexes(ctx context.Context, pool *pgxpool.Pool) error {
    rows, err := pool.Query(ctx, `
        SELECT schemaname || '.' || indexrelname AS idx_name
          FROM pg_stat_user_indexes
          JOIN pg_index USING (indexrelid)
         WHERE NOT indisunique  -- 暂不动唯一索引
           AND pg_relation_size(indexrelid) > 1024 * 1024 * 100  -- > 100 MB
           AND (pgstatindex(indexrelid::text)).avg_leaf_density < 50`)
    if err != nil {
        return err
    }
    defer rows.Close()

    var indexes []string
    for rows.Next() {
        var name string
        if err := rows.Scan(&name); err != nil {
            return err
        }
        indexes = append(indexes, name)
    }

    for _, idx := range indexes {
        log.Printf("REINDEX CONCURRENTLY %s ...", idx)
        // 注意：REINDEX CONCURRENTLY 必须单独提交，不能在事务中
        sql := fmt.Sprintf("REINDEX INDEX CONCURRENTLY %s", idx)
        if _, err := pool.Exec(ctx, sql); err != nil {
            log.Printf("failed: %v", err)
            continue
        }
        log.Printf("done %s", idx)
    }
    return nil
}
```

---

## 生产实践

### 索引设计原则清单

1. **每张表必有主键**：用 `BIGINT GENERATED ALWAYS AS IDENTITY` 或 `BIGSERIAL`，避免 UUID
2. **WHERE 高频列 + ORDER BY 列 → 联合索引**，遵循"等值 → 排序 → 范围"
3. **覆盖列用 INCLUDE**，不要塞进键
4. **稀疏值用部分索引**：status='active' / deleted_at IS NULL 这类筛选
5. **JSONB 查询用表达式索引或 GIN**（详见 P05/P12）
6. **超大表用 CONCURRENTLY**：所有 CREATE INDEX、REINDEX 都加 CONCURRENTLY
7. **定期监控膨胀**：每周扫一次 `pgstatindex`，对 density < 50 的考虑 REINDEX
8. **删除未使用索引**：`pg_stat_user_indexes.idx_scan = 0` 的可以删（用 1-2 周观察期）
9. **prepared statement 缓存 plan**：减少规划开销
10. **ANALYZE 跟上**：大批量 import 后立即 `ANALYZE`，保证规划器估算准确

### 索引压力测试模板

```sql
-- 1. 加载数据
CREATE TABLE bench (id BIGINT, data TEXT);
INSERT INTO bench SELECT i, repeat('x', 100) FROM generate_series(1, 10000000) i;

-- 2. 不同索引方案 A/B 测试
CREATE INDEX idx_a ON bench(id);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM bench WHERE id = 5000000;
DROP INDEX idx_a;

CREATE INDEX idx_b ON bench(id) INCLUDE (data);
EXPLAIN (ANALYZE, BUFFERS) SELECT data FROM bench WHERE id = 5000000;
```

### 灾难案例：误删生产索引

```sql
-- 错误：DROP INDEX 会获取 ACCESS EXCLUSIVE 锁，阻塞所有查询
DROP INDEX idx_critical;

-- 正确：先 DROP INDEX CONCURRENTLY（不阻塞）
DROP INDEX CONCURRENTLY idx_critical;
```

如果不小心 DROP 了，需要立即 `CREATE INDEX CONCURRENTLY` 重建——期间相关查询会**全表扫描**，QPS 可能崩 100x。在生产环境，**DROP INDEX 必须在维护窗口**或始终带 CONCURRENTLY。

---

## 陷阱清单

### 1. 索引列上加函数 = 索引失效

```sql
CREATE INDEX ON users(email);
SELECT * FROM users WHERE lower(email) = 'alice@example.com';  -- ✗ 不用索引

-- 正确：
CREATE INDEX ON users(lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com';  -- ✓
```

### 2. 隐式类型转换 = 索引失效

```sql
CREATE INDEX ON users(phone);  -- phone 是 TEXT
SELECT * FROM users WHERE phone = 13800000000;  -- 整数比较
-- ✗ 隐式 TEXT → BIGINT 转换，索引可能失效

-- 正确：
SELECT * FROM users WHERE phone = '13800000000';
```

### 3. `IS NOT DISTINCT FROM` 不走标准 B-tree

```sql
SELECT * FROM users WHERE name IS NOT DISTINCT FROM NULL;  -- 可能走 seq scan
SELECT * FROM users WHERE name IS NULL;  -- 走索引（如果建了的话）
```

### 4. CONCURRENTLY 失败留 INVALID 索引

```sql
\d users
--  "idx_email" btree (email) INVALID
-- → 必须 DROP INDEX CONCURRENTLY + 重建
```

### 5. 在事务中 CIC

```sql
BEGIN;
CREATE INDEX CONCURRENTLY idx ON t(col);  -- ERROR
COMMIT;
```

### 6. partial index 的 WHERE 用了 STABLE 函数

```sql
CREATE INDEX ON events(user_id) WHERE created_at > now() - interval '7 days';
-- ERROR: functions in index predicate must be marked IMMUTABLE
```

### 7. 全表 UPDATE 让 bottom-up deletion 失效

`UPDATE huge_table SET col = col + 1` 这种全表 UPDATE 会同时触发大量页分裂，bottom-up deletion 处理不过来 → 索引膨胀。改用**分批 UPDATE**（`LIMIT N` + WHERE id BETWEEN）。

### 8. 双写主键（OLD ON DUPLICATE KEY）使用错误

PG 没有 MySQL 的 `INSERT ... ON DUPLICATE KEY UPDATE`，但有 `INSERT ... ON CONFLICT`，需要明确指定冲突的索引：

```sql
INSERT INTO users (email, name) VALUES ('a@b.com', 'A')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

冲突的列必须有唯一约束/索引。

### 9. fillfactor 默认 90 在高 UPDATE 场景偏高

```sql
-- 高 UPDATE 场景建议
CREATE INDEX ON t(col) WITH (fillfactor = 70);
-- 留更多空间给 bottom-up deletion 和后续插入
```

### 10. 一表索引 > 10 个

每个索引都增加 INSERT/UPDATE/DELETE 开销。生产经验：

- 5 个以下：理想
- 5-10 个：警惕
- 10+：必须审视，多半有未使用的或冗余的

用 `pg_stat_user_indexes.idx_scan` 看哪些没人用。

---

## 2026 现状

### PG 18（2025-09 发布）

- **UUIDv7 内置**：`uuidv7()` 函数，时序友好的 UUID。用 UUIDv7 做主键不再像 UUIDv4 那样导致写放大
- **虚拟生成列（VIRTUAL）**：可用于表达式索引的等价替代
- **改进的索引扫描**：异步 IO 让 BitmapHeapScan 后的堆访问大幅加速
- **B-tree skip scan 扩展**：更多场景受益

### PG 17（2024-09 发布）

- **Improved parallel VACUUM**：索引并行清理
- **incremental backup**：减小重建副本的 I/O
- **MERGE...RETURNING**：UPSERT 语义增强

### 第三方索引扩展

- **pgvector 0.7+**：HNSW 主推（详见 P06）
- **pgvectorscale**：Timescale 出品的 StreamingDiskANN，向量索引大表加速
- **rum**：高级全文搜索索引
- **pg_bm25 / paradedb**：Tantivy 风格 BM25 索引

### 业界趋势

- **HOT update 受限的现状未变**：大字段 UPDATE 仍触发索引膨胀，bottom-up deletion 是当前最佳缓解
- **xid 64 位社区讨论**：未来可能彻底消除 wraparound 风险，影响索引 freezing
- **AWS Aurora Postgres / Google AlloyDB**：自家修改的 B-tree 实现（如 Aurora 的"日志即数据库"），生产经验逐渐积累
- **CockroachDB / YugabyteDB**：兼容 PG 协议但用 LSM 而非 B-tree，索引行为有差异

---

## 练习题

### 基础题

1. **PG B-tree 页有几种角色？** 元页、根、内部、叶子（外加 deleted、half-dead 等过渡态）。元页永远是 block 0，存树高和当前 root 块号。

2. **PG 8KB 页和 MySQL 16KB 页对索引高度的影响？** PG 同样树高能管的行数约为 MySQL 的 1/4，但实际 PG 树高通常 3-4 层、MySQL 3 层，相差有限。

3. **PG 唯一索引中多个 NULL 冲突吗？** 默认不冲突（SQL 标准行为）。PG 15+ 可用 `NULLS NOT DISTINCT` 反转。

### 进阶题

4. **解释 INCLUDE 索引和把列加进键的区别。** INCLUDE 列只存叶子页、不参与排序、不影响唯一性；键列影响排序、可用于 WHERE/ORDER BY/唯一性。

5. **Deduplication 和 Bottom-up Deletion 各解决什么问题？** Dedup 减少新插入造成的 IndexTuple 数（合并相同 key 的多个 CTID）；Bottom-up Deletion 在叶子页将分裂前主动清理 MVCC 留下的死索引项，缓解索引膨胀。

6. **CREATE INDEX CONCURRENTLY 比普通 CREATE INDEX 慢多少？为什么？** 通常 2-3 倍。原因：需要两遍扫描表 + 两次等待相关事务结束。

### 实战题

7. **给一张订单表（1 亿行，列 id/user_id/status/amount/created_at），写出最优的 4 个索引方案，并解释每个的覆盖场景。**

```sql
-- 主键
PRIMARY KEY (id)
-- 用户视角：按时间倒序的订单（最高频）
CREATE INDEX ON orders(user_id, created_at DESC) INCLUDE (status, amount);
-- 运营视角：按 status + 时间
CREATE INDEX ON orders(status, created_at DESC) WHERE status IN ('pending', 'paid');
-- 月度统计
CREATE INDEX ON orders((to_char(created_at, 'YYYY-MM')));
```

8. **一张 50GB 表上的 idx_email 索引 avg_leaf_density = 32%，如何在线重建？给出完整 SQL。**

```sql
REINDEX INDEX CONCURRENTLY idx_email;
-- 监控
SELECT * FROM pg_stat_progress_create_index;
-- 验证
SELECT indisvalid FROM pg_index WHERE indexrelid = 'idx_email'::regclass;
SELECT (pgstatindex('idx_email')).avg_leaf_density;
```

9. **如何检测一个索引被建了但从来没用过？给出 SQL 和判定标准。**

```sql
SELECT
  schemaname || '.' || indexrelname AS idx,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan
FROM pg_stat_user_indexes
WHERE NOT indisunique  -- 唯一索引可能用于约束
  AND idx_scan = 0
  AND now() - stats_reset > interval '14 days'  -- 观察 2 周
ORDER BY pg_relation_size(indexrelid) DESC;
```

判定：观察期 ≥ 14 天、`idx_scan = 0`、非唯一索引 → 候选删除。

10. **解释为什么 `WHERE name LIKE 'foo%'` 能用 B-tree 索引，而 `WHERE name LIKE '%foo'` 不能。**

B-tree 按字典序排列。`LIKE 'foo%'` 等价于 `name >= 'foo' AND name < 'fop'`，是范围查询，能用索引。`LIKE '%foo'` 没有"以什么开头"的限定，B-tree 无法定位起点。如需后者，用 trigram 索引（`pg_trgm` + GIN，详见 P05）。

---

## 延伸阅读

- **PG 官方文档**
  - [Index Types](https://www.postgresql.org/docs/18/indexes-types.html)
  - [B-Tree Indexes](https://www.postgresql.org/docs/18/btree.html)
  - [Building Indexes Concurrently](https://www.postgresql.org/docs/18/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)
  - [pgstattuple](https://www.postgresql.org/docs/18/pgstattuple.html)
  - [pageinspect](https://www.postgresql.org/docs/18/pageinspect.html)
- **源码**
  - `src/backend/access/nbtree/README`（必读）
  - `src/backend/access/nbtree/nbtinsert.c`
  - `src/backend/access/nbtree/nbtdedup.c`
- **论文**
  - Lehman & Yao, *Efficient Locking for Concurrent Operations on B-Trees* (1981)
  - Peter Geoghegan, *Anatomy of a Database System: B-Tree Indexes in PostgreSQL*
- **博客**
  - [pganalyze: A Look at PostgreSQL B-Tree Internals](https://pganalyze.com/blog/postgres-b-tree)
  - [Peter Geoghegan's blog](https://pgeoghegan.blogspot.com/) — PG 13/14 索引优化作者
  - [Crunchy Data: How To: Reindex](https://www.crunchydata.com/blog/reindex)
- **关联章节**
  - [P02 堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md)
  - [P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)
  - [P05 GIN/GiST/SP-GiST/BRIN/Hash 索引](./05-精通-多类型索引.md)
  - [P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)
  - [P10 EXPLAIN 与执行计划](./10-精通-EXPLAIN-与执行计划.md)
  - [MySQL M02 InnoDB B+ 树索引](../mysql/02-精通-InnoDB-索引.md)（差异对比）
