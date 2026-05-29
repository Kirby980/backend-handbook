# 精通 MySQL 整体架构：从连接到磁盘的一句 SQL

> 关联章节：[M02 InnoDB 索引](./02-精通-InnoDB-索引.md)、[M03 事务与 MVCC](./03-精通-InnoDB-事务-MVCC.md)、[M06 优化器](./06-精通查询优化与-EXPLAIN.md)

---

## 引言：一句 `SELECT` 到底经过了什么

"MySQL 慢" 是后端工程师最常听到的吐槽。但绝大多数人对一句 SQL 在 MySQL 里"经过了哪些层"只有模糊印象。这一章把 MySQL 当一个分层系统来拆开看。

一句 `SELECT * FROM users WHERE id = 42` 从你的 client 进来，到磁盘 read I/O，再到结果返回，会穿越**至少 6 层**：网络协议层、连接器、解析器、优化器、执行器、存储引擎（InnoDB）。其中任何一层卡住，都会被你测到"MySQL 慢"——但不同层的瓶颈，需要完全不同的优化手段。

读完这一章，你应该能回答：

- 一个连接占多少内存？连接池为什么有意义？
- Server 层和 Engine 层是怎么解耦的？为什么 MyISAM 和 InnoDB 表现差这么多？
- 优化器是怎么"猜"统计信息的？怎么覆盖它？
- 二级索引"回表"具体经过哪几层？为什么覆盖索引能省一次磁盘 I/O？
- redo / undo / binlog 在一次 UPDATE 里的写入顺序，以及 two-phase commit 的存在意义

后续每一章都是某一层的纵深；本章是地图。

---

## 第一章：高层架构图

```
+--------------------------------------------------+
|                客户端 / Driver                   |
+---------------------|----------------------------+
                      | MySQL 协议（TCP / Socket）
                      v
+--------------------------------------------------+
| Server 层（与存储引擎无关，所有引擎共享）           |
|                                                  |
|  ┌──────────────┐                                |
|  │  连接器       │  鉴权 / 线程绑定 / 权限缓存       |
|  └──────┬───────┘                                |
|         v                                        |
|  ┌──────────────┐                                |
|  │  解析器       │  词法 / 语法 → AST              |
|  └──────┬───────┘                                |
|         v                                        |
|  ┌──────────────┐                                |
|  │  预处理器     │  对象名解析 / 权限检查           |
|  └──────┬───────┘                                |
|         v                                        |
|  ┌──────────────┐                                |
|  │  优化器       │  成本估算 / 索引选择 / Join 顺序 |
|  └──────┬───────┘                                |
|         v                                        |
|  ┌──────────────┐                                |
|  │  执行器       │  调用 handler API 拿数据         |
|  └──────┬───────┘                                |
+---------┼----------------------------------------+
          | handler::index_read / handler::rnd_next ...
          v
+--------------------------------------------------+
| 存储引擎层（InnoDB / MyISAM / NDB / RocksDB ...）   |
|                                                  |
|  ┌──────────────┐  ┌──────────────┐              |
|  │ Buffer Pool  │  │ Lock System  │              |
|  └──────┬───────┘  └──────────────┘              |
|         v                                        |
|  ┌──────────────┐  ┌──────────────┐              |
|  │   B+ 树索引  │  │ Undo / MVCC  │              |
|  └──────┬───────┘  └──────────────┘              |
|         v                                        |
|  ┌──────────────┐                                |
|  │ Redo Log     │                                |
|  └──────┬───────┘                                |
+---------┼----------------------------------------+
          v
+--------------------------------------------------+
| 文件系统 / 磁盘（ibd / redo / undo / binlog）       |
+--------------------------------------------------+
```

**关键观察**：

1. Server 层和存储引擎之间通过 `handler` 抽象类解耦。MyISAM、InnoDB、NDB、RocksDB 实现同一组接口（约 30 个虚函数）。
2. **binlog 在 Server 层；redo / undo 在 Engine 层（InnoDB）**。这就是 UPDATE 涉及"两套日志"的根因。
3. 优化器**不知道** InnoDB 的内部结构——它通过 handler 获取的"统计信息估算"做决策。这是为什么优化器偶尔会做出反直觉的选择。

---

## 第二章：连接器

### 2.1 协议与握手

MySQL 监听 3306（TCP）或 socket 文件。客户端发起连接：

1. 服务端发 `Server Greeting`（含 server 版本、challenge bytes、能力位）
2. 客户端发 `Login Request`（含 username、SHA-256 加密的密码、要连的 db、能力位）
3. 服务端 `mysql_native_password` / `caching_sha2_password` 校验
4. 成功后服务端发 `OK packet`

MySQL 8.0+ 默认 `caching_sha2_password`——比 `mysql_native_password` 安全，但要求客户端支持。**很多老 driver（PHP 5、老 Java JDBC）默认不支持**，连接失败常常错错在这里。临时回退方法：

```sql
ALTER USER 'app'@'%' IDENTIFIED WITH mysql_native_password BY 'pwd';
```

但 MySQL 8.4 LTS 又**默认禁用了** `mysql_native_password`（要 `--mysql-native-password=ON` 才启用）。生产请直接升 driver。

### 2.2 一个连接的代价

每个连接 = 一个**线程**（默认） + 一个**线程局部缓冲区**：

| 内存项 | 默认 | 说明 |
|---|---|---|
| thread_stack | 1 MB | 线程栈（MySQL 8.0.27+ 64 位默认 1048576 字节；8.0.26 及更早为 280 KB） |
| net_buffer_length | 16 KB | 协议接收缓冲 |
| sort_buffer_size | 256 KB | 排序用（按需翻倍直到上限） |
| join_buffer_size | 256 KB | 无索引 Join 用 |
| read_buffer_size | 128 KB | 全表扫描缓冲 |
| read_rnd_buffer_size | 256 KB | 随机读缓冲 |
| tmp_table_size | 16 MB | 内存临时表上限 |

理论"一个 idle 连接 ≈ 1.2 MB"（线程栈一项就占 1 MB），但**一个执行复杂 Join + Order By 的连接**很容易瞬间用到几十 MB。如果你有 1000 个并发执行复杂 SQL 的连接，总内存可能上 GB——这是为什么生产要用**连接池 + 合理上限**。

### 2.3 thread_pool（企业版 / Percona / MariaDB）

社区版 MySQL 默认"一个连接一个线程"（thread-per-connection）。当连接数 > CPU 核数太多时，**上下文切换成本**显著。

企业版的 thread_pool 插件：

- 固定线程数（默认 `thread_pool_size = CPU 核数`）
- 连接到来时排队等待线程
- 短查询走快通道，长查询走慢通道

类似 Nginx 的事件驱动模型。**社区版没有这个插件**——大量短查询场景下，企业版 / Percona 性能能高 30%+。

### 2.4 wait_timeout 与连接老化

```
wait_timeout = 28800        # 默认 8 小时，空闲超时
interactive_timeout = 28800 # 交互式（mysql cli）超时
max_execution_time = 0      # SELECT 执行最大时长（毫秒），0 = 无限
```

**生产配置建议**：

- `wait_timeout = 600`（10 分钟）—— 防止应用 bug 导致连接泄漏
- 连接池侧设 `idleTimeout < wait_timeout` —— 别等服务端主动断
- 用 `SET GLOBAL max_execution_time = 30000`（30 秒）兜底长 SELECT —— 防止误查表把 CPU 打满

### 2.5 客户端常踩的坑

- **TCP keepalive vs MySQL wait_timeout**：TCP 层 keepalive 默认 7200 秒，MySQL `wait_timeout` 默认 28800 秒。如果 LB / NAT 在 wait_timeout 之前先断连，应用拿到一个"看似有效"的连接发 SQL 就 `MySQL server has gone away`。**修法**：连接池设 `validationQuery = SELECT 1` 或 `testOnBorrow=true`。
- **认证插件不兼容**：上面 2.1 提到的 `caching_sha2_password` 问题。
- **max_connections 打满**：默认 151，生产太低。设 `max_connections = 1000` 起步，配合连接池上限。

---

## 第三章：解析器与预处理

### 3.1 词法 + 语法 → AST

解析器是 yacc 生成的（`sql/sql_yacc.yy`），把 SQL 字符串拆成 token，再按 SQL 语法构建抽象语法树。这一层只关心**语法对错**——`SELEC * FROM t` 的报错 `You have an error in your SQL syntax` 就来自这里。

### 3.2 预处理：对象名解析 + 权限检查

解析完成后进入预处理：

1. **对象名解析**：把 `users` 解析成具体 schema 的 `mydb.users`，把列名 `id` 解析成具体表的列。涉及当前 db、search_path、视图展开。
2. **权限检查**：当前用户有没有 `SELECT` 这张表的权限？没有就报 `ERROR 1142: SELECT command denied`。

权限是 **per-table、per-column 级别**的，存在 `mysql.user` / `mysql.db` / `mysql.tables_priv` / `mysql.columns_priv` 四张表，权限缓存在内存（`FLUSH PRIVILEGES` 重载）。

### 3.3 查询缓存的退场

MySQL 8.0 之前有 Query Cache：把 `SELECT ... = "key"` 做 hash 缓存结果（5.7.20 起废弃）。MySQL 8.0 **彻底移除**了它，原因：

- 缓存命中率在并发更新场景下极低（任何写都让该表所有缓存失效）
- 缓存维护本身是高竞争点，写多读少环境反而拖慢
- 高级应用应在 Redis 层做缓存，而非 SQL 层

**今天没有 Query Cache**——这是为什么"一模一样的 SQL 第二次跑也不会更快"的根因。重复 SQL 加速靠 InnoDB Buffer Pool（页缓存）+ optimizer 的 prepared statement plan cache。

---

## 第四章：优化器

### 4.1 优化器在做什么

输入：parsed AST。输出：**执行计划**（access path + join order）。

执行计划的本质是回答：

1. 每张表用哪个索引？（access method：const / eq_ref / ref / range / index / ALL）
2. 多表 Join 的顺序？
3. 临时表 / 排序需要吗？
4. 用 hash join 还是 nested loop join？（8.0.18+）

### 4.2 成本模型

优化器对每种候选 plan 估算成本：

```
cost = io_cost + cpu_cost
io_cost  = 估算读多少页 × per_page_io_cost
cpu_cost = 估算处理多少行 × per_row_cpu_cost
```

成本系数可调（`mysql.server_cost` / `mysql.engine_cost` 系统表），但 99% 场景用默认就好。

**关键输入**：

- **基数估算**（cardinality）—— 估算"过滤后剩多少行"
- **唯一性**——主键 = 1，二级索引看采样的离散度
- **直方图**（histogram，8.0+）—— 显式记录列值分布

### 4.3 直方图：让优化器看懂数据倾斜

老版本 MySQL 估算"列值分布"靠**采样几条 record**——遇到数据高度倾斜（如 status='active' 占 99%）会严重错估。

MySQL 8.0+ 引入直方图：

```sql
ANALYZE TABLE users UPDATE HISTOGRAM ON status WITH 100 BUCKETS;
ANALYZE TABLE users DROP HISTOGRAM ON status;

-- 看现有直方图
SELECT * FROM information_schema.column_statistics WHERE table_name='users';
```

**何时该建**：

- 列有大量重复值且分布不均（status、is_deleted 等）
- 列无索引但常用于 WHERE
- 优化器选错索引（看 EXPLAIN type=ALL 但其实有可用索引）

直方图 **不自动维护**——表数据大变后要重新 ANALYZE。

### 4.4 优化器开关

```sql
SET optimizer_switch='index_merge=on,index_merge_union=on,...';
```

可以单独开关 25+ 个优化器特性。生产偶尔需要——比如 hash join 在某些 SQL 上反而慢，可以 `SET optimizer_switch='hash_join=off'` 强行回 nested loop。

### 4.5 强制索引

```sql
SELECT * FROM t USE INDEX (idx_name)   WHERE ...;  -- 建议用
SELECT * FROM t FORCE INDEX (idx_name) WHERE ...;  -- 强制必须用
SELECT * FROM t IGNORE INDEX (idx_a)   WHERE ...;  -- 禁用某个
```

**慎用**——一旦数据分布或新增索引，hint 可能反而劣化。生产里更推荐：通过 `OPTIMIZER_TRACE` 找根因，必要时调直方图 / 加索引 / 改 SQL。

```sql
SET optimizer_trace='enabled=on';
SELECT ...;
SELECT * FROM information_schema.optimizer_trace\G
```

输出是一段 JSON，详尽到每个 cost 怎么算出来的。

---

## 第五章：执行器

### 5.1 调用 handler API

执行器拿到 plan 后，按 plan 调用存储引擎的 `handler` 接口：

```cpp
// sql/handler.h 的关键虚函数（简化）
class handler {
    virtual int ha_index_read_map(uchar *buf, const uchar *key, ...);  // 索引等值
    virtual int ha_index_next(uchar *buf);                              // 范围扫描下一行
    virtual int ha_rnd_next(uchar *buf);                                // 全表扫描下一行
    virtual int ha_write_row(uchar *buf);                               // 插入
    virtual int ha_update_row(const uchar *old_data, uchar *new_data);  // 更新
    virtual int ha_delete_row(const uchar *buf);                        // 删除
};
```

InnoDB 的 `ha_innobase` 实现这些接口。**Server 层完全不知道 B+ 树**——它只调"给我下一行"。

### 5.2 WHERE 在哪一层过滤

经典疑问：`SELECT * FROM users WHERE id > 100 AND name LIKE '%abc%'` 的两个条件分别在哪里执行？

答：

1. `id > 100` → **下推到引擎层**（用主键索引扫描）
2. `name LIKE '%abc%'` → 有可能在 **引擎层**（Index Condition Pushdown，ICP）或 **Server 层**

ICP（MySQL 5.6+）是一个关键优化：让能用索引部分过滤的条件**在引擎层就过滤掉**，减少回表次数。EXPLAIN Extra 显示 `Using index condition` 就是 ICP 生效。

详见 M02 / M06 章节。

### 5.3 临时表 vs 内存表

执行器需要排序 / 聚合时会建临时表：

- 优先建在内存（`MEMORY` 引擎，`tmp_table_size` 限制大小）
- 超限或包含 BLOB / TEXT 列 → 转磁盘临时表（`InnoDB` 引擎，MySQL 8.0+；之前是 MyISAM）
- 磁盘临时表写入 `tmpdir`

`SHOW STATUS LIKE 'Created_tmp%'` 看：

- `Created_tmp_tables` = 总共建了多少
- `Created_tmp_disk_tables` = 其中转磁盘的数量

**`Created_tmp_disk_tables` 持续高 = 严重信号**——某条 SQL 在每次执行都生成 GB 级临时表写磁盘，要立刻定位（用 Performance Schema 看哪条 SQL）。

---

## 第六章：存储引擎层 InnoDB（速览）

详细见 M02–M04。这里给整体印象：

### 6.1 InnoDB 的核心组件

```
+----------------------------------+
|       Buffer Pool（内存）          |
| LRU 链表 + free list + flush list |
+----------------------------------+
       |          |          |
       v          v          v
+----------+ +----------+ +----------+
| 索引页   | | 数据页    | | undo 页  |
+----------+ +----------+ +----------+
       ^          ^          ^
       |          |          |
       +----------+----------+
                  |
       +----------------------------------+
       | Redo Log Buffer + Group Commit   |
       +----------------------------------+
                  |
                  v
       +----------------------------------+
       | ibd 表空间 / redo / undo / binlog |
       +----------------------------------+
```

### 6.2 一次 UPDATE 的写日志顺序

经典面试题。`UPDATE users SET age=30 WHERE id=42`：

1. 用主键索引找到行，**读到 Buffer Pool**（如果不在）
2. **生成 undo log**（记录旧 age 值，写到 undo log buffer）
3. **修改 Buffer Pool 中的数据页**（"脏页"，未落盘）
4. **生成 redo log**（记录"id=42 的页 offset X 处改成 30"，写到 redo log buffer）
5. **redo log 写到 OS Page Cache**（fsync 取决于 `innodb_flush_log_at_trx_commit`）
6. **Server 层写 binlog**
7. **InnoDB 提交**（标记事务为 committed）
8. 之后（异步）：**dirty page flush 到 ibd 文件**

**关键的 two-phase commit**：步骤 5（redo prepare）→ 步骤 6（binlog write）→ 步骤 7（redo commit）。这套机制保证：

- 崩溃时如果 binlog 已写 → redo log 也已 prepare → 重启后能把事务重做或回滚一致
- 崩溃时如果 redo log prepare 但 binlog 未写 → 重启时回滚

详见 M03 章节。

### 6.3 三种关键日志

| 日志 | 位置 | 作用 | 大小控制 |
|---|---|---|---|
| **Redo log**（ib_logfile） | Engine 层 | crash recovery：把已 commit 但未 flush 的页重新应用 | `innodb_redo_log_capacity`（8.0.30+，单参替代多文件） |
| **Undo log**（undo tablespace） | Engine 层 | 事务回滚 + MVCC 旧版本读 | `innodb_undo_log_truncate=ON` 自动 truncate |
| **Binlog**（mysql-bin.xxx） | Server 层 | 主从复制 + 时间点恢复 | `binlog_expire_logs_seconds`（默认 30 天） |

---

## 第七章：完整生命周期实战

跑一次 `SELECT * FROM users WHERE id = 42` 在 MySQL 8.4 上：

1. **连接器**（< 1ms）：复用线程；权限缓存命中
2. **解析器**（< 1ms）：构建 SELECT 的 AST
3. **预处理**（< 1ms）：解析 `users.id` 列
4. **优化器**（< 1ms）：唯一索引等值查询 → access=const，cost 极低，不犹豫
5. **执行器**：调用 `ha_innobase::index_read_map(PRIMARY, key=42, HA_READ_KEY_EXACT)`
6. **InnoDB**：
   - 在 Buffer Pool 找 root page → 内部节点 → 叶子页
   - 命中？返回行；不命中？发 read I/O 从 ibd 读 16 KB
7. **返回**：执行器把 row 通过协议层发回 client

**端到端理想耗时**（Buffer Pool 命中）：< 0.1ms。**磁盘 miss**：1-10ms（SSD），10-100ms（机械盘）。

跑一次 `SELECT COUNT(*) FROM huge_table`：

1-5 步同上
6. InnoDB：用 `index_first` + `index_next` 全索引扫描（默认走最小的索引）
7. 千万行 → 主线程秒级 CPU + 大量 page read

**这是为什么 `COUNT(*)` 在大表慢**——InnoDB 不维护"行数"元数据（不像 MyISAM）。要快读 count，用 `SHOW TABLE STATUS LIKE 'huge_table'` 拿估算值（约 ± 30% 误差），或自己维护 counter 表。

---

## 第八章：观察工具

### 8.1 SHOW PROCESSLIST

```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
SELECT * FROM information_schema.processlist;
```

看每个连接当前在干什么、状态、耗时。常见 State：

| State | 含义 |
|---|---|
| `Sending data` | 在执行查询并发送结果（最常见，不一定是慢） |
| `Sorting result` | 在排序 |
| `Creating sort index` | 创建排序临时表 |
| `Waiting for table metadata lock` | 等元数据锁（被 DDL 阻塞） |
| `Sending to client` | 在写网络 |

`KILL <id>` 终止一个会话（注意：DDL 语句 KILL 后不会立刻回滚，且锁可能仍持有一段时间）。

### 8.2 SHOW ENGINE INNODB STATUS

InnoDB 内部的"诊断快照"，**一切 InnoDB 问题第一时间看这个**：

```
=====================================
LATEST DETECTED DEADLOCK
=====================================
...

----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: ...

------------
TRANSACTIONS
------------
Trx id counter ...
Trx ACTIVE 123 sec ...   ← 长事务！

--------
FILE I/O
--------
Pending reads ..., writes ...

----------
BUFFER POOL AND MEMORY
----------
Total memory allocated ...
Buffer pool size ...
Free buffers ...
Database pages ...
Modified db pages ...
```

死锁 / 长事务 / I/O 抖动都能从这里第一时间看到。

### 8.3 Performance Schema

MySQL 8.0+ 默认开启，详细在 M07。最常用：

```sql
-- 当前正在执行的 SQL
SELECT * FROM performance_schema.events_statements_current;

-- 按 digest 聚合的 SQL 性能 Top
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT/1e9 AS sum_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;

-- 当前锁等待
SELECT * FROM performance_schema.data_lock_waits;
```

---

## 第九章：生产级最佳实践

1. **`max_connections` 起 1000**，配合连接池——别让应用直接打到上限。
2. **`wait_timeout = 600`**（10 分钟）——快速回收泄漏连接。
3. **关 Query Cache**（MySQL 8.0+ 已自动）—— 别误以为是问题原因。
4. **直方图主动建**——对倾斜列 + 优化器选错索引的场景。
5. **永远 EXPLAIN 复杂 SQL**——上线前必跑；用 `EXPLAIN FORMAT=TREE`（8.0+）看 join 顺序。
6. **大查询用 `max_execution_time` 兜底**——`SET SESSION max_execution_time = 30000` 防误查表。
7. **`Created_tmp_disk_tables` 加监控**——突增即定位。
8. **`SHOW ENGINE INNODB STATUS` 是免费的金矿**——出问题 5 分钟内必看。
9. **认证插件先升 driver**——`caching_sha2_password` 是 2026 年的默认，老 driver 都不支持。
10. **生产参数三件套**：`innodb_buffer_pool_size`（机器内存 50-70%）、`innodb_redo_log_capacity`（8.0.30+，建议 4-8 GB）、`innodb_flush_log_at_trx_commit=1`（数据安全）/ `=2`（性能高一档但断电丢 1 秒）。详见 M08。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：以为 SELECT 1 检查连接是免费的
JDBC 默认 `testOnBorrow=true` + `validationQuery=SELECT 1` → 每次借连接都跑一次 SQL。高频小事务下 SELECT 1 也是成本。改用 `testWhileIdle=true` + `timeBetweenEvictionRunsMillis=30000` 改为后台检查。

### ❌ 陷阱 2：`SELECT *` 的代价不只是网络
`SELECT *` 让优化器**几乎一定回表**（无法用覆盖索引），列出具体列名后优化器可能选个更小的二级索引一次拿完。

### ❌ 陷阱 3：长事务 + GTID = 主从延迟雪崩
一个开了 30 秒未提交的事务，在主库占着 undo log；切到从库时这条 binlog 也要 30 秒重放，导致 lag。生产配 `SET SESSION max_execution_time = 5000` 上限保护。

### ❌ 陷阱 4：临时表满到磁盘
`tmp_table_size` 默认 16 MB，复杂 GROUP BY 一上来就转磁盘。生产建议改 64-128 MB。

### ❌ 陷阱 5：DDL 期间 SELECT 等元数据锁
即使是"轻量" DDL，期间任何 SELECT/INSERT/UPDATE 都拿不到 MDL。**8.0+ 多数 DDL 都是 INSTANT 或 INPLACE**，但有些（如改列类型）仍是 COPY。生产 DDL 用 `pt-online-schema-change` / `gh-ost`。

### ❌ 陷阱 6：以为 EXPLAIN 的 rows 是准确的
那是基于直方图 + 索引 cardinality 的估算，可能差几个数量级。`EXPLAIN ANALYZE`（8.0.18+）会真跑一遍 SQL 给出实际 rows，更准但更慢。

### ❌ 陷阱 7：优化器 hint 写错位置
```sql
SELECT * FROM t USE INDEX (idx) WHERE x=1;            -- ✓
SELECT * FROM t WHERE x=1 USE INDEX (idx);             -- ❌ 语法错误
SELECT /*+ INDEX(t idx) */ * FROM t WHERE x=1;         -- ✓ Optimizer hint（8.0+ 推荐）
```

### ❌ 陷阱 8：错把 binlog_format=STATEMENT 用在 8.x
8.x 默认 ROW，老配置文件迁移过来可能仍是 STATEMENT。某些非确定性函数（NOW、UUID、AUTO_INCREMENT）在 STATEMENT 模式下主从不一致。**统一改 ROW**（或 MIXED）。

### ❌ 陷阱 9：max_connections 仍是默认 151
默认 151，生产太低。改 1000+ 起步。但**注意操作系统 file descriptor 限制**：每个连接占一个 FD，要 `ulimit -n` 配合调到 65535+。

### ❌ 陷阱 10：authentication_policy 升级踩坑
MySQL 8.4 默认 `caching_sha2_password`。从 5.7 迁的老用户密码 hash 不兼容，必须 `ALTER USER ... IDENTIFIED BY 'pwd'`（重新生成）。

---

## 第十一章：练习题

**练习 1**：解释为什么 MySQL 一句 SQL 涉及"两套日志"（redo + binlog），它们各自的作用，以及为什么需要 two-phase commit。

**练习 2**：以下 SQL 在 MySQL 8.4 上每次执行都需要解析 + 优化吗？

```sql
PREPARE stmt FROM 'SELECT * FROM users WHERE id=?';
SET @id = 42;
EXECUTE stmt USING @id;
EXECUTE stmt USING @id;
```

**练习 3**：在 100 GB 表上跑 `COUNT(*)` 需要多久？为什么 MyISAM 上是 O(1) 而 InnoDB 不是？

**练习 4**：用 Performance Schema 找出过去一小时执行次数最多的 SQL（按 digest 聚合）。写出 SQL。

**练习 5**：某个客户报"我的连接经常掉，错误是 `MySQL server has gone away`"。给出排查路径（至少 4 条假设和验证方法）。

---

## 参考答案

**练习 1**：
- **Redo log**（Engine 层）：保证 crash 后已 commit 事务能恢复（已 flush 到 redo 的页若未持久化数据页，重启后重放 redo）。物理日志，记录"页 X offset Y 改成 Z"。
- **Binlog**（Server 层）：主从复制 + 时间点恢复。逻辑日志（ROW 模式记录"行改成什么"）。
- **two-phase commit**：保证 redo 与 binlog 状态一致。如果只有 redo 没有 binlog，从库追不上；只有 binlog 没有 redo，主库恢复时数据丢。流程：prepare redo → write binlog → commit redo。崩溃恢复时若 binlog 已写 → 提交事务；binlog 未写 → 回滚。

**练习 2**：不需要重新解析+优化。`PREPARE` 时已解析+优化并缓存 plan。后续 `EXECUTE` 只走执行器。这是 prepared statement 的核心价值——尤其在高并发场景下省 CPU。**注意**：plan cache 是 per-session 的；连接池如果每次还连接前 reset，plan 会失效。

**练习 3**：100 GB 表 InnoDB COUNT(*) ≈ 几十秒到几分钟（取决于 buffer pool 命中率和 disk）。InnoDB 不存 row count 元数据，因为 MVCC 让"准确 count"依赖事务 snapshot——不同事务看到的行数不同。MyISAM 没 MVCC，单值即可。**生产替代方案**：维护 counter 表（每次插入/删除 UPDATE counter SET cnt = cnt ± 1）或近似值 `SHOW TABLE STATUS`。

**练习 4**：
```sql
SELECT DIGEST_TEXT,
       COUNT_STAR AS exec_count,
       SUM_TIMER_WAIT/1e12 AS total_sec,
       AVG_TIMER_WAIT/1e9 AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE LAST_SEEN > NOW() - INTERVAL 1 HOUR
ORDER BY COUNT_STAR DESC
LIMIT 10;
```

**练习 5**：4 条假设：
1. **`wait_timeout` 触发**：MySQL 主动断了空闲连接。验证：`SHOW VARIABLES LIKE 'wait_timeout'`；连接池设 idleTimeout 更小。
2. **LB / NAT 中间断**：网络中间设备断了 TCP。验证：tcpdump 抓 RST；连接池开 `testOnBorrow` 或 keepalive。
3. **max_allowed_packet 被超**：单条 SQL 数据超 16MB（默认）。验证：`SHOW VARIABLES LIKE 'max_allowed_packet'`，看错误日志是否有 `Got a packet bigger than 'max_allowed_packet' bytes`。
4. **服务器 OOM/重启**：`uptime` 看 MySQL 进程；`dmesg` 找 OOM-Killer；监控系统看 MySQL CPU/内存。

---

## 小结

| 层 | 主要职责 | 关键问题 |
|---|---|---|
| 客户端 / Driver | 协议封装、连接池 | 认证插件、idle 超时 |
| 连接器 | 鉴权 / 线程绑定 / 权限 | max_connections、wait_timeout |
| 解析器 | SQL → AST | 语法错误位置 |
| 预处理 | 对象名 + 权限 | 视图展开、列名歧义 |
| 优化器 | 索引选择 / Join 顺序 | 直方图、optimizer trace |
| 执行器 | 调用 handler API | 临时表、ICP、回表 |
| InnoDB | B+树 / MVCC / 锁 / Buffer Pool | 三种日志、Buffer Pool 命中率 |
| 文件系统 | ibd / redo / undo / binlog | I/O 调度、fsync 策略 |

记住四个原则：

1. **Server 层和 Engine 层解耦**——优化器不知道 B+ 树细节，所以偶尔会做反直觉决定
2. **redo / undo / binlog 各司其职**——分别管 crash recovery、MVCC、复制
3. **慢的原因可能在任何一层**——观察工具（SHOW PROCESSLIST、ENGINE INNODB STATUS、Performance Schema）是分层定位的关键
4. **不要在 SQL 层做应用层的事**——缓存让 Redis、加锁让分布式锁、消息让 Kafka

下一篇 **M02 — 精通 InnoDB B+ 树索引** 将拆开 16 KB 页结构、聚簇索引 vs 二级索引、ICP / MRR / 覆盖索引——把"为什么我建了索引却没走" 讲清楚。
