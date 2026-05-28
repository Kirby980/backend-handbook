# 精通 WAL 与物理流复制：从 LSN 到 Hot Standby 的完整链路

> 关联章节：[P01 整体架构](./01-精通-PostgreSQL-整体架构.md)、[P08 VACUUM](./08-精通-VACUUM-与表膨胀.md)、[P16 逻辑复制](./16-精通-逻辑复制.md)、[P17 高可用](./17-精通-高可用与连接池.md)

---

## 引言：WAL 是 PostgreSQL 的"生命线"

WAL（Write-Ahead Logging，预写日志）是 PostgreSQL 的根基。没有 WAL，PostgreSQL 既无法崩溃恢复，也无法做物理复制、PITR、incremental backup。理解 WAL 不是"DBA 才需要的奢侈品"——只要你的服务跑在 PostgreSQL 上，你就必须懂：

- 一次 `COMMIT` 到底等了什么
- 主备延迟 5 秒到底是网络问题还是磁盘问题
- `pg_wal` 目录涨到 50 GB 不肯回收，谁是凶手
- standby 启动失败，提示 "requested WAL segment ... has already been removed" 怎么办
- PG 17 新出的 incremental backup 到底比 `pg_basebackup` 节省多少

读完本章你应该能：

- 解释 LSN、WAL 段、timeline、checkpoint、redo point 之间的关系
- 区分 `wal_level` 三档对复制能力的影响
- 配置 streaming replication + Hot Standby + replication slot 的完整链路
- 解读 `pg_stat_replication` 五个 LSN 字段的物理含义
- 知道同步复制 ANY 2 / FIRST 2 在故障切换语义上的差别
- 用 `pg_basebackup --incremental` + `pg_combinebackup` 做 PG 17 增量备份
- 设计一个跨机房 1 主 2 备级联复制 + WAL 归档 + 同步组的拓扑

---

## 第 1 章：WAL 的本质——为什么必须先写日志

### 1.1 数据库的"持久性"难题

考虑一次最朴素的 `UPDATE`：

```
1. backend 读 page → shared_buffers
2. 修改 page 上某个 tuple（xmax = 当前 xid）
3. 标记 page 为 dirty
4. COMMIT
```

如果第 4 步之后立刻断电，dirty page 还没刷盘——数据就丢了。

最直白的方案：**每次 COMMIT 都把所有 dirty page 刷盘**。但 16 KB 一页，一个事务可能改 100 页，每页都 fsync 一次——TPS 直接降到 100 以下。

WAL 的核心思想：**只刷一段顺序追加的日志（小、顺序、快），数据页延迟刷盘**。

```
传统：随机写多个数据页 → fsync 慢
WAL：顺序追加 WAL → fsync 快；数据页交给后台慢慢刷
```

只要 WAL 落盘，崩溃后就能"redo"重建出最新页面——这就是 **Write-Ahead** 的含义：**先写日志，再改数据**。

### 1.2 WAL 的 ACID 保障

| ACID 字母 | WAL 怎么保 |
|---|---|
| **A**tomicity | undo 信息（在 PG 是 MVCC 多版本，不是单独 undo log）；崩溃恢复时未提交事务回滚 |
| **C**onsistency | redo 重放保证最终一致 |
| **I**solation | 跟 WAL 关系不大（靠 MVCC + 锁） |
| **D**urability | COMMIT 时 fsync WAL 保证已提交事务不丢 |

PostgreSQL 的 WAL 既是 redo log 也兼任 binlog 角色——MySQL 是 redo（InnoDB）+ binlog（Server）双日志，PG 只有一份 WAL，复制和归档都从它派生。

### 1.3 redo / undo / 物理 vs 逻辑日志

| 日志类型 | 例子 | PG 怎么做 |
|---|---|---|
| 物理日志（physical） | "page 13 的 offset 200 的字节改为 0xA1" | PG WAL 主体是物理（精确到页面字节） |
| 逻辑日志（logical） | "UPDATE users SET name='Bob' WHERE id=1" | PG 在 WAL 里还内嵌部分逻辑信息，供 logical decoding 使用（PG 9.4+） |
| undo | "把 page 13 改回原值" | PG 没有独立 undo（旧版本就是堆表里的旧行，靠 VACUUM 清理） |

物理日志 redo 极快（不用解析 SQL，按 offset 改字节），但**主备必须二进制完全一致**——这就是物理复制为什么不能跨大版本、不能跨架构的根本原因。

---

## 第 2 章：WAL 段、LSN、Timeline

### 2.1 WAL 段文件

WAL 不是一个大文件，而是切成 16 MB 一段（默认，编译时可改 `--with-wal-segsize=N`，PG 11+ 运行时 `initdb --wal-segsize` 也可设）。

```
$PGDATA/pg_wal/
├── 000000010000000000000001    # 16 MB
├── 000000010000000000000002
├── 000000010000000000000003
├── ...
└── archive_status/             # 归档状态目录
```

文件名 24 位 16 进制，含义：

```
00000001  0000000000000003
└── 1 ──┘ └──── 2 ────────┘
1. timeline（8 位）
2. LSN 高 64 位 / 16MB → 文件序号（16 位）
```

把文件名拆开就是 timeline + LSN。

### 2.2 LSN（Log Sequence Number）

LSN 是 **64 位字节偏移**，指向 WAL 流中某个位置。文本表示：

```
SELECT pg_current_wal_lsn();
-- 0/16B374A
-- ↑   ↑
-- 高32  低32（16 进制字节偏移）
```

LSN 单调递增，可以做减法得字节数：

```sql
SELECT pg_wal_lsn_diff('0/2000000', '0/1000000');
-- 16777216  （刚好 16 MB = 一段）
```

LSN 和 WAL 段的映射：

```
LSN 0/0000000 ~ 0/0FFFFFF → 段 000000010000000000000000
LSN 0/1000000 ~ 0/1FFFFFF → 段 000000010000000000000001
...
```

LSN 在很多地方使用：

| LSN 名 | 含义 | 查询 |
|---|---|---|
| `pg_current_wal_lsn()` | 主库当前写入位置 | 主库 |
| `pg_current_wal_insert_lsn()` | WAL buffer 中已分配但可能未 flush | 主库 |
| `pg_current_wal_flush_lsn()` | 已 fsync 到盘 | 主库 |
| `pg_last_wal_receive_lsn()` | 备库 walreceiver 收到位置 | 备库 |
| `pg_last_wal_replay_lsn()` | 备库 startup 进程已重放位置 | 备库 |
| `sent_lsn` / `write_lsn` / `flush_lsn` / `replay_lsn` | `pg_stat_replication` 中主库视角看备库 | 主库 |

### 2.3 Timeline（时间线）

Timeline 是 WAL 历史的"分支编号"。

```
timeline 1: ────────────────── (主库)
                   │
                   ↓ failover
timeline 2:        └─────────── (原备库切主后)
```

每次 promote standby 都产生新 timeline，避免新主和旧主写出不同内容到同一 LSN 造成混乱。

切换文件 `.history`：

```
$PGDATA/pg_wal/00000002.history
1   0/16B374A   no recovery target specified
```

含义：timeline 2 从 timeline 1 的 LSN `0/16B374A` 分叉。

旧备库要重新接到新主库，可以靠 `pg_rewind` 在 timeline 上回退到分叉点再追。

---

## 第 3 章：WAL 记录结构

### 3.1 物理结构

每个 WAL 段切成多个 8 KB 的 page（XLog page），每个 page 内含多个 WAL record。

```
+-------------+
| XLogPageHdr |  24 字节
+-------------+
| XLogRecord1 |
| XLogRecord2 |
| ...         |
+-------------+
```

`XLogRecord` 头部（PG 14+ 大致结构，源码 `src/include/access/xlogrecord.h`）：

```c
typedef struct XLogRecord {
    uint32      xl_tot_len;     /* 总长度 */
    TransactionId xl_xid;       /* 事务 ID */
    XLogRecPtr  xl_prev;        /* 前一条 record LSN */
    uint8       xl_info;        /* 标志位 */
    RmgrId      xl_rmid;        /* resource manager ID */
    /* 后面跟 per-rmgr data + block references */
    pg_crc32c   xl_crc;         /* 全记录 CRC */
} XLogRecord;
```

### 3.2 Resource Manager（rmgr）

WAL 不仅记录 heap 的变化，每种"东西"由对应的 rmgr 处理：

| rmgr | 负责的操作 |
|---|---|
| Heap / Heap2 | 堆表 INSERT/UPDATE/DELETE/VACUUM |
| Btree | B-tree 索引插入、分裂、删除 |
| Gin / Gist / SPGist / Brin / Hash | 各类索引 |
| XLOG | checkpoint, full-page write, parameter change |
| Transaction | COMMIT, ABORT, prepared transaction |
| CLOG | 事务状态变化 |
| MultiXact / Subtrans | 子事务和多事务 |
| Standby | running xacts snapshot（hot standby 用） |
| LogicalMessage | logical decoding 内嵌消息 |
| Generic | 扩展自定义 WAL |

查 WAL 内容（PG 10+ 内置 `pg_waldump`）：

```bash
$ pg_waldump 000000010000000000000003 | head -3
rmgr: Heap        len (rec/tot):     54/   150, tx:        750, lsn: 0/16B3748, prev 0/16B3700, desc: INSERT off 8 flags 0x00, blkref #0: rel 1663/13404/16384 blk 0 FPW
rmgr: Btree       len (rec/tot):     53/    53, tx:        750, lsn: 0/16B37E0, prev 0/16B3748, desc: INSERT_LEAF off 1, blkref #0: rel 1663/13404/16386 blk 1
rmgr: Transaction len (rec/tot):     34/    34, tx:        750, lsn: 0/16B3818, prev 0/16B37E0, desc: COMMIT 2026-05-15 10:30:00.123456 UTC
```

线上分析 WAL 风暴必备工具。

### 3.3 Full Page Write（FPW）

考虑这个场景：一次 8 KB 数据页正在写盘时断电——大部分文件系统不保证原子写，可能写了一半。这叫 **torn page**（断页）。

恢复时拿一个半新半旧的页面去 apply 物理日志（"在 offset 200 改 4 个字节"），结果完全错乱。

PostgreSQL 的解决：**每次 checkpoint 后，每个页第一次被修改时，在 WAL 里写入整页镜像**。这就是 `full_page_writes = on`（默认开）。

```
checkpoint
  ↓
page X 第一次改：WAL = [整个 8 KB page 镜像] + [delta]
page X 第二次改：WAL = [仅 delta]
  ↓
next checkpoint
  ↓
page X 又是第一次：WAL = [整个 8 KB page 镜像] + [delta]
```

**代价**：checkpoint 后的 WAL 量会暴涨（一次写入可能 8 KB + delta）。这就是为什么 checkpoint 触发后能看到 WAL 量明显上升。

**关闭风险**：torn page → 不可恢复的数据损坏。**生产永远 ON**，除非你的文件系统/硬件保证 8 KB 原子写（ZFS / 部分企业 SSD），但收益不大风险极大，**不推荐关闭**。

`wal_compression`（PG 15+ 支持 lz4/zstd）能压缩 FPW，主流压 50%+。

---

## 第 4 章：wal_level —— 决定一切能力的开关

### 4.1 三档

| wal_level | 写入内容 | 支持能力 |
|---|---|---|
| **minimal** | 仅崩溃恢复必需 | 单机恢复；**不支持复制、归档** |
| **replica**（默认 PG 9.6+） | 上面 + 主从复制需要的信息 | 流复制、归档、PITR、物理 standby |
| **logical** | 上面 + 逻辑解码需要的额外信息（如 REPLICA IDENTITY） | logical replication、CDC（Debezium 等） |

**生产建议**：

- 默认 `replica`（90% 场景够用）
- 用到逻辑复制 / CDC / 跨大版本升级 → `logical`
- `minimal` 仅适合一次性 ETL 临时实例（写 WAL 少一些，性能好一点）

### 4.2 设置方式

```ini
# postgresql.conf
wal_level = replica         # 改这个要重启
```

```sql
SHOW wal_level;
SELECT pg_reload_conf();    -- 不够，必须重启
```

### 4.3 关联参数

| 参数 | 含义 |
|---|---|
| `max_wal_senders` | 最大并行 walsender 进程（默认 10）。包括备库 + 逻辑订阅 + pg_basebackup |
| `max_replication_slots` | 最大复制槽数（默认 10） |
| `wal_keep_size`（PG 13+ 替代 `wal_keep_segments`） | 保留多少 MB WAL 给备库追（默认 0，靠 slot） |
| `wal_sender_timeout` | walsender 检测空闲超时（默认 60s） |

---

## 第 5 章：Checkpoint —— 推进 redo point

### 5.1 为什么需要 checkpoint

如果不做 checkpoint，崩溃恢复就得从最古老的 WAL 一路 redo 到最新——可能几小时。

Checkpoint 做两件事：

1. **把所有 dirty shared_buffers 刷盘**
2. **在 WAL 中写一条 CHECKPOINT record**，记录此时的 LSN 为新的 **redo point**

崩溃恢复只需从 redo point 开始 replay。

```
WAL: ─[T1]─[T2]─[CHECKPOINT@LSN_A]─[T3]─[T4]─[T5]─[T6 crashed]
                       ↑
                   redo point
                       ↓
                redo 只需从 LSN_A 开始
```

### 5.2 触发条件

| 触发 | 参数 / 时机 |
|---|---|
| 时间到 | `checkpoint_timeout`（默认 5min） |
| WAL 量到 | 写入的 WAL 接近 `max_wal_size`（默认 1 GB） |
| 手动 | `CHECKPOINT;` |
| shutdown | smart/fast 关库前必做 |
| pg_basebackup 开始时 | 强制做 |

### 5.3 关键参数

```ini
checkpoint_timeout = 15min              # 拉长能减少 FPW
max_wal_size = 16GB                     # 大写入业务必须调大
min_wal_size = 2GB                      # 复用 WAL 段数下限
checkpoint_completion_target = 0.9      # 在 timeout * 0.9 时间内匀速刷完（默认 0.9）
checkpoint_warning = 30s                # 太频繁警告
```

**调优经验**：

- 写入越大的库，`checkpoint_timeout` 越长越好（30min 也常见），减少 FPW
- 但是 `max_wal_size` 要大到容得下两次 checkpoint 之间的 WAL 量，否则会被 WAL 强制触发
- `checkpoint_completion_target` 0.9 = 慢节奏匀速刷盘，避免突发 IO 风暴

### 5.4 看 checkpoint 实际行为

```sql
SELECT * FROM pg_stat_bgwriter;
-- checkpoints_timed   1024    -- 时间触发
-- checkpoints_req     12      -- WAL 量触发（**不应远大于 timed**）
-- checkpoint_write_time   ...
-- buffers_checkpoint  ...
```

**判定**：`checkpoints_req >> checkpoints_timed` → `max_wal_size` 太小，被 WAL 量逼着提前 checkpoint，FPW 飙升。

PG 17 中部分 bgwriter 统计被拆到 `pg_stat_checkpointer`（独立 view），字段含义不变。

---

## 第 6 章：WAL 归档（PITR 基础）

### 6.1 为什么归档

`pg_wal/` 的 WAL 段在 checkpoint 之后会被回收/重命名复用——如果没归档走，过去的 WAL 就丢了，做不了 PITR（Point-in-Time Recovery）。

归档 = 把每个完成的 WAL 段拷贝到外部存储（NFS / S3 / 磁带）。

### 6.2 配置

```ini
# postgresql.conf
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
# %p = 源 WAL 段完整路径
# %f = 文件名
```

`archive_command` 返回 0 表示成功；失败 PG 会一直重试（**不删 WAL 段**，会撑爆 pg_wal）。

生产更推荐 S3 上传（用 `pgbackrest` / `wal-g` / `barman` 包装）：

```ini
archive_command = 'wal-g wal-push %p'
```

`archive_library`（PG 15+ 引入）用动态库代替 shell 命令，性能更高：

```ini
archive_library = 'basic_archive'
basic_archive.archive_directory = '/archive'
```

### 6.3 恢复（restore）

```ini
# 备库或 PITR 实例的 postgresql.conf
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-05-15 14:30:00'
recovery_target_action = 'promote'
```

PG 12+ 起，配合 `recovery.signal`（PITR）或 `standby.signal`（备库）放在 `$PGDATA` 下指示恢复模式。

PITR 完整流程：

```bash
# 1. 用最近的全备恢复 PGDATA
tar -xzf base_backup_2026-05-15.tgz -C $PGDATA

# 2. 配置 restore_command + recovery_target_time
echo "restore_command = 'cp /archive/%f %p'" >> $PGDATA/postgresql.conf
echo "recovery_target_time = '2026-05-15 14:30:00'" >> $PGDATA/postgresql.conf

# 3. 标记 recovery
touch $PGDATA/recovery.signal

# 4. 启动；PG 会重放 WAL 到指定时刻
pg_ctl start
```

---

## 第 7 章：Streaming Replication —— 流复制

### 7.1 进程模型

```
主库                            备库
  ├── walwriter                  ├── walreceiver (拉 WAL)
  ├── checkpointer               ├── startup (replay WAL)
  ├── bgwriter                   ├── bgwriter
  ├── walsender [→ 备库]        └── ...
  └── ...
```

主库每个备库连接对应一个 `walsender` 进程；备库一个 `walreceiver` 拉、一个 `startup` 重放。

```
[主 backend] → 写 WAL → [walsender] → 网络 → [walreceiver] → pg_wal/ → [startup] → 应用
                                                                            ↑
                                                                       和主库二进制一致
```

### 7.2 与传统 log shipping 的差异

| 方案 | 触发 | 延迟 | 复杂度 |
|---|---|---|---|
| **log shipping**（archive_command） | WAL 段（16 MB）写满才发 | 高（至少满一段） | 低 |
| **streaming replication** | 实时按 record 流 | 低（毫秒级） | 中 |

生产标准：**streaming + 归档兜底**（streaming 断了能从归档追）。

### 7.3 同步 vs 异步

```ini
# 主库 postgresql.conf
synchronous_commit = on              # 默认。COMMIT 等本地 WAL fsync
synchronous_standby_names = ''       # 空 = 异步复制
```

`synchronous_commit` 取值：

| 值 | COMMIT 等什么 |
|---|---|
| `off` | 不等本地 fsync（性能高，可能丢提交） |
| `local` | 等本地 fsync（默认行为，不等备库） |
| `remote_write` | 等备库 walreceiver write 到 OS（备库 OS 没崩就不丢） |
| `on`（默认） | 等备库 walreceiver fsync 到盘 |
| `remote_apply` | 等备库 startup 进程 replay 完（**备库查询能立即看到主库刚提交**） |

`synchronous_standby_names` 指定哪些备库参与同步。PG 9.6+ 支持 ANY / FIRST 语法：

```ini
synchronous_standby_names = 'ANY 2 (standby1, standby2, standby3)'
# 任意 2 个 ACK 即可

synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
# 优先 standby1；standby1 挂才轮到 standby2
```

**生产推荐**：3 备库时 `ANY 2`，写入要 2/3 确认，1 个备库挂不影响。

### 7.4 配置完整流程

**主库**：

```ini
# postgresql.conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB       # 兜底，避免 slot 失败时 WAL 被清

# pg_hba.conf
host replication repl_user 10.0.0.0/8 scram-sha-256
```

```sql
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'xxx';
```

**备库**（用 `pg_basebackup` 直接拉一份）：

```bash
pg_basebackup -h primary.example.com -U repl_user -D /var/lib/postgresql/18/data \
    -Fp -Xs -P -R -S standby1_slot
# -Fp  plain 格式
# -Xs  stream WAL（同时拉 WAL 保证一致）
# -P   显示进度
# -R   自动生成 standby.signal + postgresql.auto.conf
# -S   使用指定 slot
```

`-R` 在 PG 12+ 会自动写入：

```ini
# postgresql.auto.conf
primary_conninfo = 'host=primary.example.com port=5432 user=repl_user ...'
primary_slot_name = 'standby1_slot'
```

并创建空文件 `$PGDATA/standby.signal`（**PG 12+ 替代 `recovery.conf`** 的标志）。

启动：

```bash
pg_ctl start -D /var/lib/postgresql/18/data
```

---

## 第 8 章：Hot Standby —— 备库只读

### 8.1 历史

- PG 9.0+ 引入 Hot Standby（之前 standby 完全不可读）
- 默认开启（`hot_standby = on`）
- 备库可执行 SELECT、prepared queries、explain

不可做：INSERT/UPDATE/DELETE/DDL、序列推进、`pg_advisory_lock`（exclusive 模式）、`SELECT ... FOR UPDATE`（部分受限）。

### 8.2 Hot Standby 冲突

备库一边重放 WAL 一边响应查询，会发生冲突：

**例 1：查询持有某行 snapshot，主库已 VACUUM 清掉旧版本**

```
主库 t=10: UPDATE row
主库 t=20: DELETE row
主库 t=30: VACUUM 物理删除
       ↓ WAL ↓
备库 t=8 启动一个长查询，需要旧 snapshot
备库 t=30 收到 VACUUM record，要清掉旧行——但查询还需要
       ↓
冲突！
```

PG 的策略由 `max_standby_streaming_delay` 控制：

```ini
max_standby_streaming_delay = 30s   # 备库最多延迟 30s 等查询完成
```

- 等 30s 查询还没完 → **取消查询**（报 `ERROR: canceling statement due to conflict with recovery`）
- 等超时仍要继续 replay，否则主备延迟无限增大

**解决方案**：

```ini
hot_standby_feedback = on    # 备库定期告诉主库"我还在用 xmin=X"，主库 VACUUM 不清更早的版本
```

代价：主库膨胀风险（备库长查询会卡 VACUUM）。

**例 2：备库重放 DDL（DROP TABLE）时主库查询还在读那张表** → 取消查询。

### 8.3 备库的查询语义

| 隔离级别 | 备库支持 |
|---|---|
| READ COMMITTED | 支持 |
| REPEATABLE READ | 支持 |
| SERIALIZABLE | **不支持**（需要 SERIALIZABLE READ ONLY DEFERRABLE 替代） |

```sql
-- 备库长报表
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT ...;
COMMIT;
```

DEFERRABLE 会等到能保证可串行化的 snapshot 才执行，避免冲突。

---

## 第 9 章：Replication Slot —— 保护 WAL 不被清

### 9.1 没有 slot 的问题

```
主库不断生成 WAL，checkpoint 后旧 WAL 段会被 recycle
       ↓
备库断线 1 小时
       ↓
备库恢复连接：找不到自己上次 LSN 对应的 WAL 段 → 报错
```

解决思路 1（老方案）：`wal_keep_size = 10GB`——主库强制保留最近 10 GB WAL。

缺点：

- 设小了备库可能赶不上
- 设大了浪费空间（即使备库一直在线也保留）

### 9.2 物理 Slot

```sql
-- 主库
SELECT pg_create_physical_replication_slot('standby1_slot');

SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

备库连接时指定 `primary_slot_name`，walsender 会**记录这个备库最远没确认的 LSN**——主库的 WAL 永远保留到那之后。

```
slot.restart_lsn = 0/3000000
主库当前 WAL = 0/5000000
       ↓
0/3000000 之前的 WAL 才能被回收
```

### 9.3 Slot 的危险

如果备库**永久挂了**或者**用 slot 但配错没连**，主库 WAL 会无限增长，撑爆磁盘。

**监控必须**：

```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots
WHERE NOT temporary;
```

PG 13+ 引入 `max_slot_wal_keep_size`：

```ini
max_slot_wal_keep_size = 100GB    # slot 最多让主库保留 100 GB
```

超过这个值，主库会强制丢弃 WAL（slot 失效，备库要重建）。**生产强烈推荐设置**，避免把主库撑死。

### 9.4 临时 vs 永久 slot

```sql
-- 临时 slot：会话结束自动删除
SELECT pg_create_physical_replication_slot('tmp_slot', true);
```

`pg_basebackup -X stream` 默认创建临时 slot，传完就删——既保护备份期间 WAL 不被清，又不会遗留。

---

## 第 10 章：监控复制状态

### 10.1 主库视角 `pg_stat_replication`

```sql
SELECT pid, usename, application_name, client_addr, state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag,
       sync_state, sync_priority
FROM pg_stat_replication;
```

四个 LSN 含义：

```
[主库 walsender]
    └─ sent_lsn       已发送到网络
        └─ write_lsn   备库收到并 write 到 OS
            └─ flush_lsn   备库 fsync 到盘
                └─ replay_lsn  备库 startup 进程已 replay
```

正常情况：`sent ≥ write ≥ flush ≥ replay`，差距越小越实时。

`sync_state`：

| 值 | 含义 |
|---|---|
| `sync` | 当前同步备库 |
| `potential` | 候选（FIRST 模式下排队） |
| `quorum` | ANY 模式下投票备库 |
| `async` | 异步 |

### 10.2 备库视角 `pg_stat_wal_receiver`

```sql
SELECT pid, status, received_lsn, last_msg_send_time, last_msg_receipt_time,
       latest_end_lsn, latest_end_time
FROM pg_stat_wal_receiver;
```

### 10.3 延迟计算

时间延迟（最常用，直接告诉你"备库落后多少秒"）：

```sql
-- 在备库执行
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

字节延迟（在主库执行）：

```sql
SELECT application_name,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_behind
FROM pg_stat_replication;
```

主备完全实时（空库）时 `pg_last_xact_replay_timestamp()` 可能为 NULL，主库无写入 → 备库无延迟数据。

### 10.4 看 walsender 在干什么

```sql
SELECT pid, state, query  -- query 列对 walsender 显示其活动状态
FROM pg_stat_activity
WHERE backend_type = 'walsender';
```

---

## 第 11 章：级联复制

```
        Primary (主)
           │
        ┌──┴──┐
        ↓     ↓
   Standby1  Standby2
        │
        ↓
   Standby3 (级联，从 Standby1 拉)
```

Standby3 配置：

```ini
primary_conninfo = 'host=standby1.example.com user=repl_user ...'
primary_slot_name = 'standby3_slot'   # slot 建在 standby1 上
```

用途：

- 主库负载重，让二级 standby 从一级 standby 拉
- 跨机房：北京主，上海一级，深圳二级（深圳只跟上海通信）

注意：**级联 standby 必须等中间 standby 重放完才能拿到 WAL**，延迟会叠加。

---

## 第 12 章：故障切换（手动 promote）

### 12.1 promote 流程

```bash
# 备库执行
pg_ctl promote -D /var/lib/postgresql/18/data
# 或 SQL：SELECT pg_promote();
```

发生什么：

1. 备库停止重放 WAL
2. 升级到 timeline+1
3. 删除 `standby.signal`
4. 开放写入
5. 原备库提供完整服务

### 12.2 旧主库怎么办

直接重启旧主库当备库 → 几乎肯定失败：两边 timeline 不同，WAL 字节不一致。

工具 `pg_rewind`：

```bash
pg_rewind --target-pgdata=/var/lib/postgresql/18/data \
          --source-server='host=new-primary user=repl_user'
```

`pg_rewind` 找出旧主和新主的分叉点，把旧主上之后的页"回退"到分叉点，再补 WAL。条件：旧主必须**干净关闭**（不能 kill -9）或者开了 `wal_log_hints` / data checksums。

### 12.3 split-brain 风险

如果故障切换时主库其实没死（网络分区），出现：

- 旧主仍在写
- 新主也在写
- 应用一部分连旧主、一部分连新主 → 数据混乱

防护手段（手动 promote 没有）：

- **fencing**（强制隔离）：通过电源/IP 屏蔽旧主
- **DCS quorum**（Patroni + etcd 多数派）：失败的主库自己 demote

详见 P17。

---

## 第 13 章：pg_basebackup 全量备份

### 13.1 基本用法

```bash
pg_basebackup -h primary -U repl_user -D /backup/2026-05-25 \
              -Ft -Z 9 -X fetch -P
```

| 选项 | 说明 |
|---|---|
| `-D` | 输出目录 |
| `-Fp` / `-Ft` | plain / tar 格式 |
| `-Z 9` | gzip 压缩等级 |
| `-X stream` | 同时拉 WAL（保证一致） |
| `-X fetch` | 备份结束后一次性拉所需 WAL |
| `-X none` | 不拉 WAL（自行归档） |
| `-c fast` | 立即触发 checkpoint（默认 spread，慢） |
| `-S slot_name` | 使用 replication slot |
| `-R` | 自动生成 standby.signal + primary_conninfo |
| `-P` | 显示进度 |

### 13.2 原理

pg_basebackup 内部走 streaming replication 协议：

```
1. 调 pg_backup_start()（PG 15+，旧版叫 pg_start_backup）
2. 触发 checkpoint
3. 流式拉 PGDATA 所有文件
4. 同时启动 walsender 拉 WAL（如果 -X stream）
5. 调 pg_backup_stop()
6. 拉完最后的 WAL
```

备份期间允许业务写入——这是物理热备份的核心能力。

---

## 第 14 章：PG 17 Incremental Backup

### 14.1 痛点

每周全备 500 GB，下班高峰 IO 占满。增量备份能省 80%+ 流量。

PG 17 之前：靠 `pgbackrest` / `wal-g` 等外部工具实现块级增量。

PG 17（2024-09）：**内置 `pg_basebackup --incremental`**。

### 14.2 原理

PG 17 引入 **WAL summary**：后台进程 `walsummarizer` 持续扫描 WAL，把"哪些块被改过"汇总成 `pg_wal/summaries/*.summary` 小文件。

下次 `--incremental` 备份时，主库扫这些 summary 知道某个数据文件的哪些 block 改了，只拷改了的块。

### 14.3 用法

```bash
# 1. 启用
echo "summarize_wal = on" >> postgresql.conf
pg_ctl reload

# 2. 先做一次全备（manifest 必须保留）
pg_basebackup -D /backup/full -Fp -X stream

# 3. 之后每天做增量
pg_basebackup -D /backup/inc1 -Fp -X stream \
              --incremental=/backup/full/backup_manifest

# 4. 第二次增量基于第一次
pg_basebackup -D /backup/inc2 -Fp -X stream \
              --incremental=/backup/inc1/backup_manifest
```

### 14.4 还原：`pg_combinebackup`

增量备份不能直接启动——需要先合并成完整目录：

```bash
pg_combinebackup -o /restore /backup/full /backup/inc1 /backup/inc2
# 现在 /restore 可以直接当 PGDATA 启动
```

工具自动校验 manifest 一致性，缺一个增量就拒绝合并。

### 14.5 收益（实测样例）

| 场景 | 全备 | 增量 |
|---|---|---|
| 100 GB 库，日变 1 GB | 100 GB | 1-2 GB |
| 1 TB 库，日变 10 GB | 1 TB | 10-15 GB |
| 时序表（只插入） | 1 TB | 几乎 = 新插入量 |

---

## 生产实践

### 实践 1：典型 1 主 2 备 + 同步组 + 归档

```
      ┌─────────────────────┐
      │     Primary         │
      │  wal_level=replica  │
      │  sync ANY 1 (s1,s2) │
      │  archive_command    │ ──→  S3 (wal-g)
      └────────┬────────────┘
               │ streaming
       ┌───────┴───────┐
       ↓               ↓
  ┌────────┐      ┌────────┐
  │ Standby1│      │ Standby2│
  │ Hot Stby│      │ Hot Stby│
  │  slot1  │      │  slot2  │
  └────────┘      └────────┘
```

关键配置（主库）：

```ini
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
synchronous_commit = on
synchronous_standby_names = 'ANY 1 (standby1, standby2)'
archive_mode = on
archive_command = 'wal-g wal-push %p'
max_slot_wal_keep_size = 200GB
checkpoint_timeout = 15min
max_wal_size = 16GB
full_page_writes = on
wal_compression = lz4
```

### 实践 2：监控核心查询

```sql
-- 1. 复制延迟
SELECT application_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS lag_bytes,
       replay_lag
FROM pg_stat_replication;

-- 2. slot 累积
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;

-- 3. checkpoint 健康度
SELECT checkpoints_timed, checkpoints_req,
       round(checkpoint_write_time::numeric / 1000, 1) AS write_s,
       round(checkpoint_sync_time::numeric / 1000, 1) AS sync_s
FROM pg_stat_bgwriter;

-- 4. archive 失败
SELECT archived_count, failed_count, last_failed_wal, last_failed_time
FROM pg_stat_archiver;
```

### 实践 3：故障演练手册

每季度做一次（生产前必排：先在 staging 做）：

1. 停一个备库 30 分钟，看 WAL 累积是否触发 `max_slot_wal_keep_size`
2. kill -9 主库进程，备库 promote，应用切换 DSN，统计 RTO
3. 用 `pg_rewind` 把旧主接回新主拓扑
4. 模拟网络分区（iptables drop），确认 sync 备库切换语义
5. 用一周前的全备 + WAL 归档做 PITR 到指定时刻

### 实践 4：跨机房同步

```
北京主 ─→ 北京备1（sync ANY 1）
       └→ 上海备2（async，远程容灾）
       └→ 深圳备3（async，远程只读）
```

绝不要把跨机房备库设为 sync——网络抖一下主库就 hang 了。同城容灾用 sync，异地用 async。

---

## 陷阱清单

1. **`max_wal_senders` 算不够**——`pg_basebackup`（含 -X stream）、备库、逻辑订阅都各占 1 个。10 个上限在中等集群常爆。
2. **`replication slot` 配了但备库永久下线**——主库 pg_wal 涨满 → 主库 panic shutdown。**必须配 `max_slot_wal_keep_size`**。
3. **关闭 `full_page_writes`**——torn page 风险，存量数据可能不可恢复。除非你 100% 确定文件系统/硬件保证 8 KB 原子写。
4. **`hot_standby_feedback = on` + 备库长报表**——主库 dead tuple 飙升、表膨胀。生产报表库可以开，但要监控膨胀。
5. **`synchronous_standby_names = 'standby1'`（FIRST 1 默认）**——standby1 一挂，主库写入全 hang。改成 `ANY 1 (standby1, standby2)`。
6. **跨大版本物理复制**——PG 16 主和 PG 17 备库**根本不能搭**，物理复制要求严格同版本。跨大版本要走逻辑复制。
7. **`pg_rewind` 没开 `wal_log_hints`**——多数情况 rewind 直接报错。建议主库永远 `wal_log_hints = on`（或开 data checksums，自带 hint logging）。
8. **`archive_command` 失败但没监控**——WAL 段一直留着，pg_wal 涨。`pg_stat_archiver.failed_count` 必须告警。
9. **故障切换没 fence 旧主**——split-brain 双写。手动切换没保护，靠人确认；Patroni 走 DCS quorum + watchdog。
10. **`pg_basebackup` 不带 `-X stream`**——备份期间 WAL 可能被回收，备份失败。**永远带 -X stream 或 -X fetch**。
11. **incremental backup 删了中间任一份**——`pg_combinebackup` 拒绝合并；增量链断了从最后一个完整全备重做。
12. **standby 配 `recovery.conf`**（PG 11 写法）——PG 12+ 启动直接拒绝。改用 `postgresql.auto.conf` + `standby.signal`。

---

## 2026 现状

- **PG 18（2025-09）**：异步 IO（io_uring/posix_aio）可显著降低 WAL 写盘和数据页 IO 等待；改进了 streaming 协议的 keepalive。
- **PG 17（2024-09 LTS）**：内置 incremental backup（`--incremental`）和 `pg_combinebackup`；`walsummarizer` 进程；failover slot 实验性，订阅在主库故障后能继续。
- **WAL 压缩 `wal_compression`** 默认仍 off，可选 `pglz`/`lz4`/`zstd`（PG 15+），FPW 多的场景压缩比通常 50%+。
- **`pg_walinspect` 扩展**（PG 15+）允许 SQL 查询 WAL record 内容，调试更方便（不用每次 `pg_waldump`）。
- **`pg_basebackup` 默认开启 manifest**（PG 13+），含 SHA-256 校验。
- **CloudNativePG / Crunchy / Zalando Operator** 几乎所有 K8s PG 部署都用 streaming + slot + S3 归档（wal-g/pgbackrest）三件套。
- **行业趋势**：异步复制 + 同城 sync 1 备 + 异地 async 2 备 + S3 归档兜底，是金融/电商标准拓扑。

---

## 练习题

1. **LSN 计算**：`pg_current_wal_lsn()` 返回 `1/A2C3D4E5`，`pg_wal/` 目录下当前 WAL 段文件名是什么？（提示：timeline + LSN 高 / 16MB）

2. **全页写**：解释为什么 `full_page_writes = off` 时，主库崩溃后某些数据页可能"半新半旧"，并说明 PostgreSQL 通过 FPW 怎么避免。

3. **slot 风险**：你创建了 `standby1_slot` 但忘了配置备库。一周后 `pg_wal/` 涨到 800 GB。说出：为什么会涨？怎么紧急回收？如何防止再发生？

4. **同步语义**：`synchronous_commit = remote_write` 和 `synchronous_commit = on` 的差别是什么？分别可能丢什么场景的数据？

5. **故障切换**：1 主 1 同步备库 + 1 异步备库。同步备库突然死机，主库写入会发生什么？是阻塞、降级还是报错？怎么配避免这个？

6. **PITR**：你需要把数据库恢复到"2026-05-15 14:30:00 UTC，事务 ID 接近 1000000 的那次 UPDATE 之前的状态"。列出 PG 18 上你会用到的所有配置项和文件。

7. **incremental backup**：解释 PG 17 `walsummarizer` 怎么知道某个数据块改过；增量链 full → inc1 → inc2 → inc3，删了 inc1 还能恢复到 inc3 吗？为什么？

8. **Hot Standby 冲突**：备库上跑一个 30 分钟报表，主库一直 VACUUM 老表。`max_standby_streaming_delay = 30s`，会发生什么？怎么平衡报表和复制延迟？

---

## 延伸阅读

- 官方文档：[Chapter 30. High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/18/high-availability.html)
- 官方文档：[Chapter 31. Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html)（对照物理理解）
- 官方文档：[Chapter 25. Reliability and the Write-Ahead Log](https://www.postgresql.org/docs/18/wal.html)
- Egor Rogov, _PostgreSQL 14 Internals_, Chapter 5 (WAL), Chapter 23 (Replication)
- pganalyze blog: ["Five mistakes beginners make with replication slots"](https://pganalyze.com/blog)
- Crunchy Data: ["WAL Compression Comparison: pglz vs lz4 vs zstd"](https://www.crunchydata.com/blog)
- Postgres weekly: PG 17 incremental backup 系列分析
- 源码：`src/backend/access/transam/xlog.c`、`src/backend/replication/walsender.c`、`src/backend/replication/walreceiver.c`、`src/backend/postmaster/walsummarizer.c`（PG 17+）
- 工具：`pgbackrest`、`wal-g`、`barman` 三大归档方案对比
