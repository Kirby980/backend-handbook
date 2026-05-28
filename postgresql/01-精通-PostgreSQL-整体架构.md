# 精通 PostgreSQL 整体架构：进程模型、共享内存、后台进程与一条 SQL 的旅程

> 关联章节：[P02 堆表存储与 TOAST](./02-精通-堆表存储与-TOAST.md)、[P03 MVCC 与可见性](./03-精通-MVCC-与可见性.md)、[P15 WAL 与物理流复制](./15-精通-WAL-与物理流复制.md)、[P17 高可用与连接池](./17-精通-高可用与连接池.md)

---

## 引言：为什么 PostgreSQL 的架构必须从"进程"开始讲

如果你只用过 MySQL，第一次部署 PostgreSQL 看 `ps -ef | grep postgres` 一定会愣一下：

```
postgres  1234   1     0  postgres: postmaster -D /pgdata
postgres  1235  1234  0  postgres: logger
postgres  1236  1234  0  postgres: checkpointer
postgres  1237  1234  0  postgres: background writer
postgres  1238  1234  0  postgres: walwriter
postgres  1239  1234  0  postgres: autovacuum launcher
postgres  1240  1234  0  postgres: archiver
postgres  1241  1234  0  postgres: stats collector       # PG 14-（PG 15+ 移除）
postgres  1242  1234  0  postgres: logical replication launcher
postgres  2001  1234  0  postgres: app app_db 10.0.0.5(54231) idle
postgres  2002  1234  0  postgres: app app_db 10.0.0.6(54232) SELECT
postgres  2003  1234  0  postgres: app app_db 10.0.0.7(54233) idle in transaction
postgres  2004  1234  0  postgres: app app_db 10.0.0.8(54234) idle
...（每个客户端连接一个进程）
```

PostgreSQL 的进程模型是 **"一个 postmaster 守护进程 + 一组后台进程 + 每个客户端连接 fork 一个 backend"**。这与 MySQL "一个 mysqld 主进程 + 大量线程"是根本不同的两种世界观。理解这个差异是理解 PG 一切运维行为（为什么必须 PgBouncer、为什么 prepared plan 不能跨连接复用、为什么 1000 并发会爆 OOM）的起点。

读完这一章，你应该能：

- 画出 PostgreSQL 完整进程架构图，列出每个后台进程的职责与触发条件
- 解释共享内存中 `shared_buffers / WAL Buffer / CLOG / Lock Table / ProcArray / MultiXact` 各自的作用
- 用一条 7 层路径走通 SQL 从客户端到磁盘的全流程
- 计算 "为什么 1000 并发裸接 PG 会需要 10 GB 内存" 并理解 PgBouncer 的必然性
- 看懂 `$PGDATA` 目录结构，能从目录布局反推出引擎的设计
- 用 `pg_stat_activity / pg_stat_bgwriter / pg_settings` 在生产中观察架构运作

---

## 第一章：postmaster 守护进程与每连接 1 进程模型

### 1.1 postmaster 的职责

`postmaster`（即主 `postgres` 进程，PID 通常是 `$PGDATA/postmaster.pid` 里那个）做且只做四件事：

| 职责 | 说明 |
|---|---|
| **监听客户端连接** | TCP（默认 5432）+ Unix domain socket（默认 `/tmp/.s.PGSQL.5432`） |
| **fork backend 进程** | 每个新连接 fork 一个 backend，把 fd 传过去 |
| **启动后台进程** | 启动时拉起 bgwriter / checkpointer / walwriter / autovacuum launcher 等 |
| **监控子进程崩溃** | 任一 backend 崩溃 → 主动 kill 所有 backend → 重做 crash recovery |

postmaster 本身**不处理 SQL**——它只是一个 supervisor。这个设计借鉴自 Berkeley 的 Postgres 项目（1986），目标是**故障隔离**：一个 backend 段错误不会污染其他 backend 的内存。

### 1.2 与 MySQL 多线程模型的对比

| 维度 | PostgreSQL（多进程） | MySQL（多线程） |
|---|---|---|
| 并发模型 | postmaster fork() 每个连接一个进程 | mysqld 主进程 + 线程池（thread_handling=pool-of-threads）或一线程一连接 |
| 单连接开销 | 10-15 MB RSS（含 catalog cache、relcache、SQL plan cache） | 256 KB - 2 MB（线程栈 + per-thread buffer） |
| 1000 并发内存 | 10-15 GB（裸接 PG） | ~2-4 GB |
| 隔离性 | 强（地址空间隔离，一个 backend 崩溃只影响自己） | 弱（一个线程崩溃可能拖死整个 mysqld） |
| Plan Cache | per-backend，不能跨连接复用 prepared statement | 全局 Query Cache（已废弃）/ 每线程 prepared stmt |
| 上下文切换成本 | fork() 慢（10-100ms），上下文切换 ~ 5μs | 线程创建 ~ 50μs，上下文切换 ~ 1μs |
| 必须连接池 | **是**（PgBouncer 几乎是标配） | 否（但高并发也建议 ProxySQL） |
| 共享数据 | shared_buffers 显式映射（System V shm / mmap） | InnoDB Buffer Pool 进程内堆 |
| 跨连接通信 | 信号 + 共享内存 + LISTEN/NOTIFY | 进程内锁、消息队列 |

PG 选多进程的核心理由是 **1986 年 Unix 还没有成熟的线程支持**（pthread 1995 才标准化）。今天再看，多进程有得有失：

- ✅ 隔离强，运行 30 年的生产数据库没有"一个 bug 搞挂整个实例"的传说
- ✅ 与现代云原生（cgroup / namespace）天然契合，每个 backend 可独立限额
- ✅ 编程简单（不用考虑 false sharing、内存重排序、TLB 抖动那么多事）
- ❌ 单连接开销大 → 必须连接池
- ❌ fork() 在 NUMA 大机器上慢，连接突发风暴会引爆 postmaster
- ❌ 共享数据必须显式放共享内存，开发新功能门槛高

### 1.3 fork 一个 backend 的代价

```c
// src/backend/postmaster/postmaster.c 简化
static int BackendStartup(Port *port) {
    pid_t pid = fork_process();
    if (pid == 0) {
        // 子进程：调用 BackendInitialize() → BackendRun()
        InitProcess();              // 注册到 ProcArray（共享内存）
        InitPostgres(dbname, ...);  // 加载 catalog cache、relcache
        PostgresMain(...);          // 主循环：读 SQL、Parse / Plan / Execute
        exit(0);
    }
    return pid;
}
```

`InitPostgres` 是大头——要从磁盘读 `pg_class`、`pg_attribute`、`pg_proc` 等 70+ 张系统表，构建 catalog cache。在中等规模 schema（几千张表）下 fork+init 耗时 30-100ms。这就是为什么短连接频繁（每秒上千次 connect/disconnect）会让 PG 飙到 100% CPU 而 SQL 本身一行都没跑。

**结论：所有生产 PG 都应该走 PgBouncer**。这不是建议，是基础设施要求。

---

## 第二章：共享内存详解

### 2.1 共享内存总览

PostgreSQL 启动时通过 `shmget()`（System V）或 POSIX `mmap()`（PG 9.3+ 默认）申请一大块共享内存，分给所有后台进程和 backend 使用。

```
┌────────────────────────────────────────────────────────────┐
│                  PostgreSQL Shared Memory                  │
├────────────────────┬───────────────────────────────────────┤
│ Shared Buffers     │ 数据/索引页缓冲池（最大区域，通常 25% RAM）│
│ (shared_buffers)   │                                       │
├────────────────────┼───────────────────────────────────────┤
│ WAL Buffer         │ WAL 写盘前的缓冲（默认 -1 = 1/32       │
│ (wal_buffers)      │ shared_buffers，常见 16 MB）           │
├────────────────────┼───────────────────────────────────────┤
│ CLOG               │ 事务提交状态位图（2 bit / xid）        │
│ (pg_xact)          │ 32 MB 分页，按需常驻                  │
├────────────────────┼───────────────────────────────────────┤
│ MultiXact          │ 行锁组合事务（多事务持锁同一行）        │
├────────────────────┼───────────────────────────────────────┤
│ Lock Table         │ 重量级锁哈希表（关系锁、advisory lock） │
├────────────────────┼───────────────────────────────────────┤
│ Predicate Lock     │ SSI 谓词锁（可串行化隔离）             │
├────────────────────┼───────────────────────────────────────┤
│ ProcArray          │ 所有活动 backend 的 PGPROC 数组        │
│                    │ （MVCC Snapshot 构造关键）            │
├────────────────────┼───────────────────────────────────────┤
│ Subtrans / Notify  │ 子事务、LISTEN/NOTIFY 队列            │
├────────────────────┼───────────────────────────────────────┤
│ Stats Shared (PG15+)│ 累积统计的共享内存（替代 stats collector）│
├────────────────────┼───────────────────────────────────────┤
│ Replication Slots  │ 复制槽元数据                          │
└────────────────────┴───────────────────────────────────────┘
```

### 2.2 shared_buffers：核心缓冲池

**作用**：缓存数据页（堆表、索引、TOAST、visibility map、free space map）。是性能的第一影响因素。

```sql
SHOW shared_buffers;
-- 默认 128 MB（适合开发机），生产至少 4-8 GB

SHOW effective_cache_size;
-- planner 估算 OS page cache 多大；不分配内存，仅参与代价模型
```

**配置经验**：

| 总内存 | shared_buffers 推荐 | effective_cache_size 推荐 |
|---|---|---|
| 8 GB | 2 GB（25%） | 6 GB（75%） |
| 32 GB | 8 GB | 24 GB |
| 64 GB | 16 GB | 48 GB |
| 128 GB | 16-32 GB | 96 GB |
| 256 GB+ | **不再线性增长**，常见 32-64 GB | 192 GB |

为什么 shared_buffers 不无脑设到 50%-75%？原因是 PG 的 buffer manager 在 **超大 buffer pool 下 LRU 锁竞争剧增**，且与 OS page cache **双层缓存**（同一页在 PG buffer 和 OS cache 各占一份）。社区共识是中等规模（16-64 GB），剩下交给 OS。

**观察**：

```sql
-- 缓冲池命中率（应 > 99%）
SELECT
  sum(heap_blks_hit) * 100.0 /
  nullif(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS hit_ratio
FROM pg_statio_user_tables;

-- 哪些表占用 buffer 最多
SELECT c.relname,
       count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname
ORDER BY buffers DESC
LIMIT 10;
-- 需要 CREATE EXTENSION pg_buffercache;
```

### 2.3 WAL Buffer

**作用**：所有 INSERT/UPDATE/DELETE 先写 WAL 记录到 WAL Buffer（共享内存），由 walwriter 异步刷盘；commit 时强制 fsync（除非 `synchronous_commit = off`）。

```sql
SHOW wal_buffers;       -- 默认 -1 表示 1/32 shared_buffers
SHOW wal_writer_delay;  -- walwriter 默认 200ms 唤醒一次
```

**调优要点**：

- 高写入负载下 `wal_buffers` 设到 64 MB（默认即可，PG 已经自动算）
- `commit_delay + commit_siblings` 可以让小事务批量 commit（吞吐换延迟）
- 极端写场景可用 `synchronous_commit = off`（异步提交），代价是崩溃丢失最近 ~600ms 事务

### 2.4 CLOG (Commit Log / pg_xact)

**作用**：记录每个事务 ID 的状态（in_progress / committed / aborted / sub-committed），每个 xid 2 bit。

```
pg_xact/
  0000   ← 第 0 个 CLOG 段，32 KB（PG 16- 是 256 KB，PG 17 改为可配置，PG 18 默认 32 KB）
  0001
  0002
  ...
```

文件名是十六进制段号。一个段覆盖 `262144` 个 xid（256K × 8 bit / 2 bit）。事务 ID 一共 32 位（约 42 亿），一年高写入实例可能用掉几亿 xid。

CLOG 的访问极其频繁——**每次可见性判定都要查 xmin/xmax 的事务状态**。所以 CLOG 页**常驻共享内存**（CLOG SLRU buffer，默认 128 个 8KB 页 = 1 MB；PG 17 起可调 `transaction_buffers`）。

### 2.5 Lock Table（重量级锁）

存储所有 **表级锁、advisory lock、关系级锁**。每个锁项 ~200 字节。

```sql
-- 当前所有重量级锁
SELECT pid, locktype, mode, granted, relation::regclass
FROM pg_locks
WHERE locktype IN ('relation', 'transactionid', 'advisory');
```

行锁**不在** Lock Table——行锁是写在 Tuple Header 的 xmax 字段里（详见 [P03 MVCC](./03-精通-MVCC-与可见性.md)），不占共享内存。这是 PG 行锁能"无限多"的原因（不像某些数据库行锁会升级到表锁）。

### 2.6 ProcArray：所有活动事务的目录

**ProcArray** 是一个 PGPROC 结构数组，每个活动 backend 一项，记录：

- backend PID
- 当前事务 xid
- 当前事务的 xmin（最早活动事务 ID）
- 复制状态、wait event、应用名等

**用途**：构造 MVCC Snapshot 时遍历 ProcArray 拿到所有"正在活动"的 xid，组成 `xip[]` 数组。

**痛点**：ProcArray 是全局结构，每次 `GetSnapshotData()` 要遍历所有 backend——**连接数过多（>500）时 ProcArrayLock 成为竞争热点**。PG 14 引入 `procarray_truncate` 优化，PG 16 又有 `LSN-based Snapshot` 实验性改进。

### 2.7 MultiXact

当多个事务同时持有同一行的共享锁（`SELECT FOR SHARE` 或外键引用），单一 xmax 装不下，PG 创建一个 **MultiXact ID** 代替单个 xid 写入 xmax。MultiXact 自己有独立的状态文件（`pg_multixact/`）和 wraparound 风险。

```
pg_multixact/
  members/   ← 每个 MultiXact 的成员事务列表
  offsets/   ← 成员列表的索引
```

生产经验：高并发外键场景会大量产生 MultiXact，监控 `pg_stat_database.multixact_id` 不要逼近 wraparound 阈值。

---

## 第三章：后台进程详解

### 3.1 进程清单速查

| 进程 | 作用 | 触发条件 | 失败影响 |
|---|---|---|---|
| **postmaster** | 主守护，fork backend | 启动时唯一 | 致命 |
| **logger** | 写 stderr 日志到文件 | `logging_collector=on` | 日志丢失 |
| **bgwriter** | 持续把脏页刷出（不强制） | 总在跑（每 200ms 唤醒一次） | 性能下降，checkpoint 压力大 |
| **checkpointer** | 周期性 checkpoint（强制刷脏 + 推进 redo 点） | `checkpoint_timeout` 或 `max_wal_size` 触发 | WAL 堆积，崩溃恢复变慢 |
| **walwriter** | WAL Buffer 刷盘 | `wal_writer_delay`（200ms） | 提交延迟上升 |
| **autovacuum launcher** | 决定何时启动 vacuum worker | 持续运行 | 表膨胀失控，wraparound |
| **autovacuum worker** | 实际跑 VACUUM/ANALYZE | launcher 派发，最多 `autovacuum_max_workers`（默认 3） | 同上 |
| **archiver** | 把完整 WAL 段拷到归档目录 | `archive_mode=on` | PITR 不可用 |
| **stats collector**（PG 14-） | 收集 pg_stat_* 统计 | 总在跑 | pg_stat 视图过期 |
| **logical replication launcher** | 启动 logical replication apply worker | `wal_level=logical` 且有订阅 | 逻辑复制中断 |
| **startup** | 崩溃恢复重放 WAL，备库持续应用 WAL | 启动初期或 standby 持续运行 | 启动失败 / 复制停止 |
| **walreceiver**（standby） | 从主库流式拉 WAL | standby 模式 | 复制延迟无限增长 |
| **walsender**（primary） | 推 WAL 给 standby / 逻辑订阅 | 有 replication 连接 | 复制中断 |
| **io worker**（PG 18+） | 异步 IO（io_uring）专用 worker | `io_method=io_uring` | 退化为同步 IO |

### 3.2 bgwriter：连续刷脏的"温柔工"

bgwriter 的策略不是"把所有脏页刷掉"，而是 **"每轮挑 N 个最近最少使用的脏页刷掉"**，目的是让 backend 在淘汰页时**不需要自己同步写**。

```sql
SHOW bgwriter_delay;          -- 200ms 唤醒一次
SHOW bgwriter_lru_maxpages;   -- 每轮最多刷 100 页
SHOW bgwriter_lru_multiplier; -- 自适应系数（默认 2.0）
```

观察：

```sql
SELECT * FROM pg_stat_bgwriter;
-- buffers_clean    : bgwriter 刷的页数（健康）
-- buffers_backend  : backend 自己刷的页数（不健康，说明 bgwriter 不够快）
-- buffers_checkpoint: checkpoint 刷的页数
-- buffers_alloc    : 总分配次数
```

判断 bgwriter 是否够用：`buffers_backend / (buffers_clean + buffers_backend + buffers_checkpoint)` 应 < 10%。如果高，调高 `bgwriter_lru_maxpages` 或缩短 `bgwriter_delay`。

> PG 17 起 `pg_stat_bgwriter` 部分指标拆到 `pg_stat_checkpointer` 和 `pg_stat_io`，名称略变。

### 3.3 checkpointer：周期性大刷

**Checkpoint** 是把当前共享内存里所有脏页强制写入数据文件、然后在 WAL 中写一条 `CHECKPOINT` 记录，以便崩溃恢复时知道"从哪条 WAL 开始重放"。

触发条件（任一）：

1. 距上次 checkpoint 时间 ≥ `checkpoint_timeout`（默认 5min，生产推荐 15-30min）
2. 上次 checkpoint 后产生的 WAL ≥ `max_wal_size`（默认 1GB，生产推荐 16-64GB）
3. 手动 `CHECKPOINT;` 或 shutdown
4. 切换 WAL level、`pg_start_backup` 等

```
        WAL stream:  | seg N | seg N+1 | seg N+2 | seg N+3 | ...
                       ↑                              ↑
                  上次 checkpoint                  本次 checkpoint
                  (redo point)
                       └── 崩溃恢复从这里开始重放 ──┘
```

**Spread Checkpoint**：默认 `checkpoint_completion_target = 0.9`，意味着 checkpointer 把刷脏分摊到 90% 的 timeout 区间，避免一次性 IO 风暴。

观察：

```sql
SELECT * FROM pg_stat_checkpointer;  -- PG 17+
-- 或 pg_stat_bgwriter (PG 16-)
-- num_timed: 按时间触发的次数
-- num_requested: 被 max_wal_size 强制触发的次数（这个高说明 max_wal_size 太小）
-- write_time / sync_time: 刷脏耗时
```

**关键经验**：如果 `num_requested > num_timed`，说明 WAL 增长速度超过 checkpoint 节奏，应调大 `max_wal_size`。否则会陷入"checkpoint 紧急触发 → IO 风暴 → 业务延迟尖刺"。

### 3.4 walwriter

把 WAL Buffer 里的内容 `write()` 到 WAL 段文件（不强制 fsync）。`COMMIT` 时由 backend 自己 fsync（除非 `synchronous_commit = off`）。

```sql
SHOW wal_writer_delay;  -- 200ms 唤醒一次
SHOW wal_writer_flush_after;  -- 累积 1 MB 后强制 fsync
```

### 3.5 autovacuum launcher / worker

PG 的 MVCC 留下 dead tuple（旧版本），必须 VACUUM 才能回收。autovacuum 系统自动做这件事。

```
autovacuum launcher（一个）
    │ 持续检查所有数据库的 pg_stat_user_tables
    │ 决定哪个表需要 VACUUM / ANALYZE
    │
    ↓ fork
autovacuum worker（最多 N 个，N = autovacuum_max_workers，默认 3）
    │
    └── 对单表执行 VACUUM / ANALYZE
```

触发条件（任一）：

```
dead_tuples > autovacuum_vacuum_threshold (50)
              + autovacuum_vacuum_scale_factor (0.2) × n_live_tup

inserts > autovacuum_vacuum_insert_threshold (1000)        -- PG 13+
        + autovacuum_vacuum_insert_scale_factor (0.2) × n_live_tup

age(relfrozenxid) > autovacuum_freeze_max_age (200M)       -- 防 wraparound 强制 VACUUM
```

详细见 [P08 VACUUM 与表膨胀](./08-精通-VACUUM-与表膨胀.md)。

### 3.6 stats collector → 共享内存统计（PG 15+ 革命）

**PG 14 及以前**：所有 backend 通过 UDP 发统计包给 stats collector 进程，collector 写临时文件 → 其他 backend 读文件构建 `pg_stat_*` 视图。这套机制有两个老问题：

1. UDP 包丢失 → 统计数据偶尔不准
2. 临时文件 `pg_stat_tmp/` 写入频繁，SSD 寿命加速折损

**PG 15 起**移除 stats collector 进程，改为 **共享内存统计**（`pgstat.c` 重写）：

- 每个 backend 直接更新共享内存里的统计计数器
- `pg_stat_*` 视图直接读共享内存
- 持久化到 `pg_stat/pgstat.stat` 文件（仅 shutdown 时写）

效果：统计无丢失、零临时文件、性能更好。`ps -ef` 里不再有 `stats collector` 进程。

### 3.7 logical replication launcher / worker

`wal_level = logical` 且存在 subscription 时启动。每个订阅一个 apply worker，负责拉解码后的逻辑变更并 apply 到本地。详见 [P16 逻辑复制](./16-精通-逻辑复制.md)。

### 3.8 archiver

```
archive_mode = on
archive_command = 'cp %p /mnt/wal_archive/%f'
```

WAL 段写满（默认 16 MB）后，archiver 调用 `archive_command` 拷贝到归档目录。失败会无限重试（直到 `$PGDATA/pg_wal/` 被填满）——这是生产常见的"WAL 占满磁盘"故障根因。

PG 15+ 推出 **archive_library** 替代 shell 命令，可加载共享库直接调用云对象存储（如 pgBackRest）。

### 3.9 startup 与 walreceiver（standby 链路）

主库崩溃恢复时由 startup 进程重放 WAL；standby 模式下 startup 持续从 `pg_wal/` 应用 WAL，walreceiver 从主库流式拉取新 WAL 段。详见 [P15 WAL 与物理流复制](./15-精通-WAL-与物理流复制.md)。

### 3.10 PG 18 新进程：io worker

PG 18 引入异步 IO（`io_method = io_uring | worker`），新增独立的 io worker 池（`io_workers` 默认 3）专门处理异步读请求。这是十多年来 PG IO 子系统最大的改动，目的是消除 backend 在 I/O 上的同步阻塞。

---

## 第四章：一条 SQL 从客户端到磁盘的 7 层路径

### 4.1 全景图

```
┌──────────┐
│  Client  │  psql / pgx / JDBC
└─────┬────┘
      │ 1. TCP / Unix Socket（postgres wire protocol v3）
      ↓
┌──────────────────────────────────────────────────────────────┐
│  postmaster   ─── fork ───→   Backend Process               │
└──────────────────────────────────────────────────────────────┘
                                       │
                                       ↓
       ┌────────────────────────────────────────────────────┐
       │ Backend 内部 7 层处理路径                          │
       │                                                    │
       │ ① Parser     ─→ Raw Parse Tree                    │
       │     │  (gram.y / scan.l, flex+bison)               │
       │ ② Analyzer   ─→ Query (resolved 引用)             │
       │     │  (analyze.c, 查 catalog cache)               │
       │ ③ Rewriter   ─→ Query'（视图展开、规则改写）       │
       │     │  (rewriteHandler.c)                          │
       │ ④ Planner    ─→ Plan Tree（最优执行计划）         │
       │     │  (planner.c, 代价模型 + 动态规划)            │
       │ ⑤ Executor   ─→ 节点流（拉取式 volcano model）    │
       │     │  (execMain.c, executor.c)                    │
       │ ⑥ Access     ─→ heap / index AM 接口              │
       │     │  (heapam.c, nbtree.c, ...)                   │
       │ ⑦ Storage    ─→ Shared Buffer Manager → 磁盘      │
       │       (bufmgr.c, smgr.c, md.c)                     │
       └────────────────────────────────────────────────────┘
```

### 4.2 各层细节

**第 1 层：网络协议（libpq / wire protocol）**

客户端发送 `Q` 消息（Simple Query）或 `P/B/E/S` 序列（Extended Query / Prepared Statement）。`pgx` 默认走 Extended Query，能复用 plan。

```go
// pgx 示例
conn, _ := pgx.Connect(ctx, "postgres://app:pwd@db:5432/app_db")
var name string
err := conn.QueryRow(ctx, "SELECT name FROM users WHERE id=$1", 42).Scan(&name)
// pgx 内部发送 Parse + Bind + Execute 三条消息
```

**第 2 层：Parser**

`scan.l`（flex）做词法分析，`gram.y`（bison）做语法分析，产生 RawStmt 树。这一层**不查 catalog**，只检查语法。

**第 3 层：Analyzer**

`parse_analyze()` 把 `RawStmt` 转成 `Query`，过程中查 catalog cache（`pg_class / pg_attribute / pg_type`）解析每个标识符。例如 `SELECT name FROM users` 会确定 `users` 是哪个 OID、`name` 是第几列、类型是什么。

**第 4 层：Rewriter**

应用 RULE 规则（古老的特性，今天主要用于实现视图）和视图展开：

```sql
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
SELECT name FROM active_users WHERE id = 1;
-- Rewriter 改写成：
-- SELECT name FROM users WHERE deleted_at IS NULL AND id = 1;
```

**第 5 层：Planner**（PG 的灵魂）

基于代价模型选择最优执行路径。对每个连接节点用 **动态规划**（< 12 表）或 **GEQO 遗传算法**（≥ 12 表，可配 `geqo_threshold`）。

详见 [P09 查询规划器](./09-精通-查询规划器.md)。

**第 6 层：Executor**

Volcano（拉取）模型：上层节点调用 `ExecProcNode(下层)` 一次拿一行。PG 11+ 支持 JIT（LLVM 编译表达式）；PG 14+ 支持并行哈希连接 spilling。

**第 7 层：Storage**

Access Method（heap / btree / gin / ...）通过 BufferManager 拿到页（在 shared_buffers 中），更改后标脏，写 WAL。WAL fsync 完成才返回 commit ack。

### 4.3 一个完整例子的时序

```
Client: BEGIN;
  └── backend: 分配 xid（懒分配，第一次写时才分配）

Client: UPDATE users SET name='Bob' WHERE id=1;
  ├── 1. Parse → Analyze → Plan → Execute
  ├── 2. heapam: 找到 ctid (block 5, offset 3) 的元组
  ├── 3. 在 shared_buffers 找 block 5（命中或读盘）
  ├── 4. 写新版本（xmin=本事务xid, xmax=0）到同页空闲处（HOT update）
  │      或写到其它页（普通 UPDATE）
  ├── 5. 旧版本 xmax 改为本事务 xid（行锁同时获得）
  ├── 6. 写 WAL 记录到 WAL Buffer
  └── 7. 标记 page 为 dirty（不立即刷盘）

Client: COMMIT;
  ├── 1. 在 CLOG 中标记本事务为 committed
  ├── 2. WAL Buffer flush + fsync（synchronous_commit=on 默认）
  └── 3. 返回 CommandComplete 给客户端

[后台]
  ├── bgwriter 后续把 dirty page 刷到磁盘
  ├── checkpointer 周期性把所有 dirty 刷掉 + 推进 redo 点
  └── autovacuum 之后清理 dead 旧版本
```

注意几个关键点：

- 提交时**只需 WAL fsync**，数据页可以晚很久才刷（WAL-First / WAL-Ahead 原则）
- 崩溃恢复时 startup 进程从 redo 点重放 WAL 重建 shared_buffers
- 这就是为什么 PG 写入很快——单次提交只有 1 次顺序 fsync

---

## 第五章：为什么必须 PgBouncer

### 5.1 数学题：1000 并发要多少内存

```
单 backend 内存占用（估算）：
  catalog cache         ~3-5 MB
  relcache              ~2-3 MB
  plan cache            ~1-2 MB（每个 prepared stmt 几 KB）
  work_mem 临时分配     可达 work_mem 上限（默认 4 MB，可调高）
  per-stmt 内存         1-3 MB
  ───────────────────────────────
  典型 RSS              10-15 MB
  繁忙复杂查询          可达 50-100 MB
```

1000 并发：

```
1000 × 15 MB = 15 GB（最低）
1000 × 50 MB = 50 GB（繁忙）
```

加上 `shared_buffers`（假设 16 GB）、OS page cache、其它系统开销，需要至少 64 GB 内存的机器才能勉强裸接 1000 并发——而且很可能 ProcArrayLock 已经成为瓶颈。

### 5.2 PgBouncer 的三种 pool_mode

| Mode | 复用粒度 | 客户端连接 ↔ backend 关系 | 限制 |
|---|---|---|---|
| **session** | 整个客户端 session | 1 : 1 | 不省连接，等于没用 |
| **transaction** | 每个事务 | 多客户端轮流用一个 backend，事务内独占 | 不能用 prepared stmt（PG 14+ 起 PgBouncer 1.21+ 支持 protocol-level prepared）、`SET LOCAL` |
| **statement** | 每条 SQL | 极致复用 | 不允许多语句事务 |

**生产推荐：transaction mode**。一个 PgBouncer 把 1000 客户端连接复用到 50 个 backend，资源占用降到 1/20。

### 5.3 PgBouncer 部署拓扑

```
                    ┌──────────────┐
   client × 1000 ──→│  PgBouncer   │←── 50 个 backend ─→ PostgreSQL
                    │  (单进程)    │
                    └──────────────┘
                       │
                       ├─ 配置 pool_size = 50
                       ├─ max_client_conn = 2000
                       └─ pool_mode = transaction
```

PgBouncer 是单线程异步事件驱动（libevent），单进程能跑几万 QPS。极大并发可以多实例（每个连接到不同 IP，或前面挂 HAProxy）。

更激进的方案：**pgcat**（Rust 实现，2024 兴起，多线程 + 内置分片）、**Odyssey**（Yandex，多线程）。

### 5.4 Go pgx 中的连接池配置

```go
import "github.com/jackc/pgx/v5/pgxpool"

config, _ := pgxpool.ParseConfig("postgres://app:pwd@pgbouncer:6432/app_db")
config.MaxConns = 20            // 客户端到 pgbouncer 的最大连接
config.MinConns = 5
config.MaxConnLifetime = 30 * time.Minute
config.MaxConnIdleTime = 5 * time.Minute
config.HealthCheckPeriod = 1 * time.Minute

// 关键：transaction mode 下要禁用 prepared statement cache
// 或 PgBouncer 1.21+ 启用 protocol-level prepared statement support
config.ConnConfig.DefaultQueryExecMode = pgx.QueryExecModeExec

pool, err := pgxpool.NewWithConfig(ctx, config)
```

**两级池子的死亡陷阱**：app pool max=20 × 10 个 app 实例 = 200 个客户端连接到 PgBouncer，PgBouncer pool_size=50，PG max_connections=100。看似没问题，但如果某个 SQL 慢了，200 个客户端连接全部 hang 在 PgBouncer 队列里——业务雪崩。

经验：

- app pool max 算总连接数 ≤ PgBouncer max_client_conn × 0.8
- PgBouncer pool_size ≤ PG max_connections / PgBouncer 实例数 - 预留管理连接
- 始终保留 5-10 个 superuser 连接（`reserved_connections` / `superuser_reserved_connections`）

---

## 第六章：$PGDATA 目录结构

### 6.1 主要目录与文件

```
$PGDATA/                          # 默认 /var/lib/postgresql/18/data
├── PG_VERSION                    # 主版本号（"18"）
├── postgresql.conf               # 主配置
├── postgresql.auto.conf          # ALTER SYSTEM 写的覆盖配置
├── pg_hba.conf                   # 客户端认证规则
├── pg_ident.conf                 # OS 用户 → PG 角色映射
├── postmaster.pid                # postmaster PID + 端口 + 启动时间
├── postmaster.opts               # 启动参数
│
├── base/                         # ★ 用户数据库目录
│   ├── 1/                        # template1 数据库（OID=1）
│   ├── 4/                        # template0
│   ├── 5/                        # postgres
│   └── 16384/                    # 你创建的数据库
│       ├── 2619                  # 一张表的主 fork（文件名 = relfilenode）
│       ├── 2619_fsm              # Free Space Map（自由空间图）
│       ├── 2619_vm               # Visibility Map（可见性图）
│       ├── 2619.1                # 同一关系超过 1 GB 的第 2 段
│       └── ...
│
├── global/                       # 集群共享对象（pg_database, pg_authid 等）
│   ├── pg_control                # ★ 集群核心元数据（LSN, system_id, ...）
│   └── ...
│
├── pg_wal/                       # ★ WAL 段（默认 16 MB 一个）
│   ├── 000000010000000000000001  # 第 1 个段
│   ├── 000000010000000000000002
│   ├── archive_status/           # 归档状态
│   └── ...
│
├── pg_xact/                      # ★ CLOG 事务状态（PG 10 前叫 pg_clog）
│   ├── 0000
│   ├── 0001
│   └── ...
│
├── pg_multixact/                 # MultiXact 状态
│   ├── members/
│   └── offsets/
│
├── pg_tblspc/                    # 表空间符号链接
│   └── 16385 -> /mnt/ssd2/pg_data_18
│
├── pg_replslot/                  # 复制槽元数据
│
├── pg_subtrans/                  # 子事务父子关系
│
├── pg_notify/                    # LISTEN/NOTIFY 队列
│
├── pg_serial/                    # SSI 谓词锁
│
├── pg_snapshots/                 # 显式 export 的 snapshot
│
├── pg_logical/                   # 逻辑复制临时文件
│
├── pg_stat/                      # 累积统计（PG 15+ shutdown 持久化）
├── pg_stat_tmp/                  # PG 14- 的临时统计文件（PG 15 起为空）
│
└── log/                          # 日志（如果 logging_collector=on）
    └── postgresql-2026-05-28_000000.log
```

### 6.2 关键文件深挖

#### pg_control（global/pg_control）

集群的"灵魂"文件，存集群级元数据：

```bash
$ pg_controldata $PGDATA
pg_control version number:            1700
Catalog version number:               202405141
Database system identifier:           7392011892341023456   # ← 唯一 ID，复制必须一致
Database cluster state:               in production
pg_control last modified:             Wed 28 May 2026 10:30:45 AM UTC
Latest checkpoint location:           1/3A8E2D40
Latest checkpoint's REDO location:    1/3A8E2D08
Latest checkpoint's TimeLineID:       1
Latest checkpoint's NextXID:          0:34892341          # ← 下一个分配的事务 ID
Latest checkpoint's NextOID:          16400
Latest checkpoint's oldestXID:        739                 # ← 最老活跃 xid（防 wraparound）
Latest checkpoint's oldestXID's DB:   16384
Latest checkpoint's oldestActiveXID:  34892341
WAL block size:                       8192
WAL segment size:                     16777216            # 16 MB
```

`pg_control` 损坏 = 集群无法启动。所以备份必须包含它（`pg_basebackup` 自动）。

#### relfilenode → 表名

```sql
-- 一张表的物理文件名是 relfilenode（不是 OID，UPDATE 后会变）
SELECT oid, relname, relfilenode, pg_relation_filepath(oid)
FROM pg_class WHERE relname = 'users';
--  oid  | relname |  relfilenode  |     pg_relation_filepath
-- ------+---------+---------------+------------------------------
--  16400| users   |  16400        | base/16384/16400
```

`VACUUM FULL / CLUSTER / TRUNCATE / REINDEX` 会改变 relfilenode（创建新文件再 swap），所以**绝不要用 OID 拼路径**，必须用 `pg_relation_filepath()`。

#### 三类 fork

每张表/索引有最多 4 个物理文件：

| Fork | 文件名后缀 | 内容 |
|---|---|---|
| **main** | （无） | 实际数据页 |
| **fsm** | `_fsm` | Free Space Map：每页剩余空间（用于 INSERT 找位置） |
| **vm** | `_vm` | Visibility Map：每页是否"全部可见"（index-only scan 关键） |
| **init** | `_init` | unlogged table 的初始化模板（crash 后用它重建） |

### 6.3 表文件大小限制：1 GB 分段

PG 的表/索引单文件最大 **1 GB**（编译时 `RELSEG_SIZE`），超过自动分段：

```
16400        # 第 1 段
16400.1      # 第 2 段
16400.2      # 第 3 段
...
```

这是为了兼容老文件系统（ext3 的 2 TB 限制等），现代文件系统其实可以编译时调到 4 GB / 32 GB，但社区版默认仍是 1 GB。

### 6.4 操作场景对照

| 场景 | 看哪里 |
|---|---|
| 磁盘满 | `du -sh $PGDATA/pg_wal/` 看 WAL 是否堆积；`du -sh $PGDATA/base/*/` 看哪个表大 |
| 哪张表占空间 | `SELECT pg_size_pretty(pg_total_relation_size('t'));` |
| 复制延迟 | `pg_replslot/` 是否堆积；`SELECT * FROM pg_stat_replication;` |
| autovacuum 没在跑 | `ls pg_stat_progress_vacuum`；`SELECT * FROM pg_stat_user_tables WHERE n_dead_tup > 10000;` |
| 日志在哪 | `SHOW log_destination;` `SHOW logging_collector;` `SHOW log_directory;` |

---

## 第七章：观察架构运作的 SQL 速查

```sql
-- 1. 版本与编译信息
SELECT version();
SHOW server_version_num;       -- 180001 表示 18.1
SHOW data_directory;
SHOW config_file;

-- 2. 共享内存关键参数
SHOW shared_buffers;
SHOW wal_buffers;
SHOW effective_cache_size;
SHOW max_connections;
SHOW work_mem;
SHOW maintenance_work_mem;

-- 3. 当前活动 backend
SELECT pid, usename, application_name, client_addr,
       state, wait_event_type, wait_event,
       backend_start, xact_start, query_start,
       LEFT(query, 80) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- 4. 后台进程
SELECT pid, backend_type, wait_event_type, wait_event
FROM pg_stat_activity
WHERE backend_type != 'client backend';

-- 5. 缓冲池命中率（应 > 99%）
SELECT
  sum(blks_hit) * 100.0 / nullif(sum(blks_hit) + sum(blks_read), 0) AS hit_ratio
FROM pg_stat_database;

-- 6. checkpoint 状态
SELECT * FROM pg_stat_checkpointer;  -- PG 17+
-- SELECT * FROM pg_stat_bgwriter;   -- PG 16-

-- 7. WAL 位置与活动
SELECT
  pg_current_wal_lsn() AS current_lsn,
  pg_walfile_name(pg_current_wal_lsn()) AS wal_file,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0')) AS total_wal;

-- 8. 复制状态（主库）
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;

-- 9. 锁等待（看 NOT granted）
SELECT pid, relation::regclass, mode, locktype, granted
FROM pg_locks
WHERE NOT granted;

-- 10. 阻塞链
SELECT
  blocking.pid AS blocking_pid,
  blocked.pid AS blocked_pid,
  blocking.query AS blocking_query,
  blocked.query AS blocked_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

---

## 生产实践

1. **始终用 PgBouncer**。生产 PG 没有 PgBouncer 就像 K8s 没有 Service —— 短期能跑，长期出事。`pool_mode = transaction`，`pool_size` 按 CPU 核心数 × 2-4 起步。
2. **shared_buffers 设到 25%**。32 GB 机器给 8 GB；64 GB 机器给 16 GB。再大边际收益急剧下降，把内存留给 OS page cache。
3. **`work_mem` 不能设太大**。`work_mem` 是 per-operation（不是 per-session、不是 per-query），一个复杂查询可能消耗 N 倍。1000 并发 × work_mem 16MB × 平均 2 个 sort/hash = 32 GB。生产 4-16 MB 起步，特定慢查询 SET LOCAL 临时拉高。
4. **`max_connections` 不要超过 500**。每个 backend 占 PGPROC 槽位 + 共享内存开销 + ProcArrayLock 竞争。真正能扛 1000+ 并发的是 PgBouncer 后面的 50 个 backend。
5. **`max_wal_size` 调到 16-64 GB**。默认 1 GB 在写入负载下会频繁触发"requested checkpoint"导致 IO 风暴。
6. **`archive_command` 失败会撑满磁盘**。务必 `archive_command` 严格 `set -eo pipefail`，且监控 `archive_status/` 文件数。
7. **监控 `pg_stat_activity.wait_event_type`**。`LWLock / Lock / IO / Client` 等占比能定位瓶颈类型。
8. **不要禁用 autovacuum**。"autovacuum 慢就关掉"是新手最大的灾难——表会膨胀到原 5-10 倍，wraparound 危机会在某个深夜爆发。

---

## 陷阱清单

1. **`fork() flood`**：每秒上百个新连接会让 postmaster CPU 100%。解决：PgBouncer。
2. **`shared_buffers` 设过大**：超过物理内存的 40-50% 会导致 OS 频繁 swap PG buffer，性能崩盘。
3. **`work_mem` 全局调高**：把单查询 OOM 风险放大到全集群。
4. **PgBouncer transaction mode + prepared statement**：旧版 PgBouncer 不支持，连接报错 `prepared statement does not exist`。升级到 1.21+ 或改用 `pgx.QueryExecModeExec`。
5. **`archive_command` 写本地磁盘后忘了清理**：`pg_wal/` 不会膨胀，但归档目录会爆。
6. **`max_connections` 改了忘 reload**：postgres.conf 写了 500，但 PG 仍用旧的 100，新连接被拒。`SHOW max_connections;` 验证。
7. **`pg_control` 损坏**：`pg_resetwal` 是最后的稻草——会丢数据但能启动。备份永远要 `pg_basebackup`，不要直接拷 `$PGDATA`。
8. **kill -9 postgres backend**：会触发整个集群崩溃恢复（postmaster 看到有 backend 异常退出，会主动重启全部）。优雅终止用 `SELECT pg_terminate_backend(pid);`。
9. **OOM killer 杀的不是 postmaster**：Linux 默认 OOM killer 倾向于杀大内存进程（往往是 backend），但 postmaster 看到子进程 SIGKILL 仍会触发崩溃恢复。把 postmaster 设 `oom_score_adj = -1000`。
10. **HugePages 配错**：`huge_pages = on` 但内核未预留足够 HugePages → PG 启动失败。`huge_pages = try` 是安全默认。

---

## 2026 现状

- **PG 18（2025-09 发布，2026 主流）**：异步 IO（`io_method = io_uring`）大幅降低读放大下的等待；UUIDv7 内置；虚拟生成列；改进的统计信息。
- **PG 17 LTS（2024-09）**：incremental backup（`pg_basebackup --incremental`）；改进的 VACUUM 内存管理；MERGE...RETURNING。
- **stats collector 已死（PG 15+）**：所有累积统计走共享内存，性能好一个数量级。
- **PgBouncer 1.21+**：protocol-level prepared statement support，终于摆脱 transaction mode 的痛点。
- **pgcat / Odyssey 崛起**：Rust / 多线程实现的连接池，PgBouncer 单线程瓶颈下的替代选项。
- **CloudNativePG GA**：CNCF Incubation，K8s 上跑 PG 的事实标准。
- **进程模型仍是多进程**：社区讨论了十年"是否改线程"，主流意见仍是保持进程模型——隔离性、与容器化的契合度都是优势。

---

## 练习题

**1. 你的 PG 实例 `max_connections = 200`，实际只有 50 个客户端连接，但 `ps` 看到 80 个 `postgres:` 进程。这正常吗？**

答案要点：正常。除了 50 个客户端 backend，还有 postmaster + bgwriter + checkpointer + walwriter + autovacuum launcher + 最多 3 个 autovacuum worker + logical replication launcher + archiver + walsender（若有 standby） + walreceiver（若是 standby） + logger 等后台进程，加起来 ~10-30 个。

**2. 为什么 `synchronous_commit = off` 不会丢失已 commit 的数据，但会丢失"刚 commit 几百毫秒"的数据？**

答案要点：`off` 模式下 COMMIT 不等 WAL fsync 就返回，WAL 数据仍在 OS page cache。崩溃前 fsync 完成的 WAL 不丢；未 fsync 的会丢——窗口约 `wal_writer_delay × 3 ≈ 600ms`。所有数据一致性约束（约束、外键、触发器）仍然生效，"已 commit 的数据"指 fsync 落盘的部分。

**3. 一台 32 GB 内存的服务器，PgBouncer pool_size 应设多少？为什么不能简单设到 max_connections=200？**

答案要点：pool_size 反映的是 PG 后端实际并发执行能力，受限于 CPU 核数（典型 8-16 核 → pool_size 20-40）和内存（每 backend 10-50 MB → 32 GB 内存 - 16 GB shared_buffers - 4 GB OS = 12 GB / 30 MB ≈ 400 上限，但 CPU 早瓶颈）。`max_connections=200` 是硬性最大，pool_size 应远小于它，留余地给管理连接和突发。

**4. `pg_stat_bgwriter.buffers_backend` 占 `buffers_alloc` 的 40%，意味着什么？怎么改？**

答案要点：意味着 40% 的页淘汰是 backend 自己同步写出的（影响业务延迟），bgwriter 没跟上节奏。调高 `bgwriter_lru_maxpages`（默认 100，可到 500-1000）或降低 `bgwriter_delay`（默认 200ms，可到 50ms）。同时检查是不是 checkpoint 太频繁（`max_wal_size` 太小）。

**5. 一条 SQL 走完全程，哪一步是最慢的？哪一步消耗最多 CPU？**

答案要点：最慢通常是 Storage 层（磁盘 IO，毫秒级，比内存慢千倍）；CPU 消耗最多通常是 Planner（复杂 join 的代价估算 + 路径搜索）或 Executor（hash 大表）。简单点查命中 buffer 时全程 < 1ms。

**6. `kill -9 backend_pid` 和 `SELECT pg_terminate_backend(pid)` 有什么区别？为什么前者很危险？**

答案要点：`pg_terminate_backend` 发送 SIGTERM，backend 优雅清理后退出，不影响其它进程。`kill -9` 发 SIGKILL，postmaster 检测到子进程异常退出（非正常 exit），认为可能损坏了 shared_buffers，**主动重启所有 backend 并执行 crash recovery**——业务全部断连，恢复期间不可服务。生产严禁。

**7. `$PGDATA/base/16384/2619` 这个文件是什么？怎么从文件名反查表名？**

答案要点：是数据库 OID=16384 中 relfilenode=2619 的表/索引主 fork。反查：

```sql
\c db_name
SELECT relname, relkind FROM pg_class WHERE pg_relation_filenode(oid) = 2619;
```

注意 relfilenode 在 VACUUM FULL / CLUSTER / TRUNCATE / REINDEX 后会变。

**8. 为什么 PG 必须有 autovacuum 而 MySQL 不需要？**

答案要点：PG 的 MVCC 是"多版本物理共存"（旧版本留在堆表中等待清理），不清理就膨胀且会触发 wraparound 危机。MySQL InnoDB 的 MVCC 通过 undo log，commit 时旧版本"逻辑上"立即可回收（实际由 purge thread 处理），表本身不会因为 MVCC 膨胀。详见 [P03](./03-精通-MVCC-与可见性.md) 与 [P08](./08-精通-VACUUM-与表膨胀.md)。

---

## 延伸阅读

- 官方文档：[PostgreSQL 18 Internals - Server Programming](https://www.postgresql.org/docs/18/runtime.html)、[Backup and Restore](https://www.postgresql.org/docs/18/backup.html)
- 必读书：《PostgreSQL Internals》Egor Rogov, [免费在线](https://postgrespro.com/community/books/internals)，第 1-2 章
- 源码路径：
  - `src/backend/postmaster/postmaster.c` ：postmaster 主循环、fork backend
  - `src/backend/postmaster/bgwriter.c / checkpointer.c / walwriter.c` ：核心后台进程
  - `src/backend/storage/ipc/shmem.c` ：共享内存分配
  - `src/backend/storage/lmgr/proc.c` ：PGPROC 与 ProcArray
  - `src/backend/utils/init/postinit.c` ：InitPostgres、catalog cache 加载
  - `src/backend/tcop/postgres.c` ：PostgresMain 主循环（SQL 处理 7 层）
- 博客：[pganalyze: Inside Postgres](https://pganalyze.com/blog/postgres-internals-architecture)、[Crunchy Data: Memory Management](https://www.crunchydata.com/blog/postgres-memory-management)
- 工具：`pg_buffercache` 扩展（看共享缓冲池内容）、`pg_proctab` 扩展（看后台进程资源消耗）
- 视频：PGConf.EU 2024 - "PostgreSQL Process Architecture" by Bruce Momjian
