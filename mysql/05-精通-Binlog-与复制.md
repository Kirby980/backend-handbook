# 精通 Binlog 与复制：行格式、GTID、半同步、并行复制

> 关联章节：[M03 事务](./03-精通-InnoDB-事务-MVCC.md)、[M09 高可用](./09-精通-MySQL-高可用.md)

---

## 引言：复制是 MySQL 高可用的物理基础

单实例 MySQL = 单点故障。生产 MySQL 一定要主从复制：

- 故障转移（master 挂 → replica 顶）
- 读扩展（写主 / 读从）
- 异地容灾（跨机房 replica）
- 时间点恢复（binlog + 备份）

但"主从复制"这四个字背后有一长串机制：binlog 格式、复制线程、GTID、半同步、并行复制、延迟、丢失、错乱。这章把它们讲清。

读完之后你应该能：

- 解释 row / statement / mixed 三种 binlog 格式各自陷阱
- 区分异步、半同步、组复制
- 用 GTID 在故障后续接
- 解释为什么从库会延迟，怎么并行
- 一次复制中断的标准排查流程
- 设计一个跨机房 + 一主多从 + 时间点恢复的拓扑

---

## 第一章：binlog 是什么

### 1.1 binlog 在 Server 层

```
[InnoDB] 写 redo（崩溃恢复）
[Server] 写 binlog（复制 + PITR）
```

binlog 跟引擎无关——所有引擎（InnoDB / MyISAM / NDB）都写 binlog。redo 是 InnoDB 私有。

事务执行时：

1. 改 buffer pool 中页
2. 写 undo
3. 写 redo（PREPARE 阶段）
4. **写 binlog**
5. 写 redo（COMMIT 阶段）

binlog 在 2PC 中跟 redo 协同（M03 已讲）。

### 1.2 binlog 用途

| 用途 | 说明 |
|---|---|
| 主从复制 | replica 拉 binlog 应用 |
| 时间点恢复 (PITR) | 基于全备 + binlog 重放到指定时间 |
| 数据订阅 | Canal / Debezium 解析 binlog 发到 Kafka |
| 审计 | 完整记录所有变更（DML） |

### 1.3 文件结构

```
binlog.000001
binlog.000002
...
binlog.index    所有 binlog 文件的目录
```

每次 mysqld 重启、`FLUSH LOGS` 或文件达到 `max_binlog_size`（默认 1GB）就新开一个。

清理：

```sql
SHOW BINARY LOGS;
PURGE BINARY LOGS TO 'binlog.000010';   -- 删 010 之前的
PURGE BINARY LOGS BEFORE '2026-05-01 00:00:00';

-- 自动过期
SET PERSIST binlog_expire_logs_seconds = 604800;  -- 7 天
```

---

## 第二章：三种 binlog 格式

### 2.1 STATEMENT —— 记录 SQL 语句

```
UPDATE users SET name='Bob' WHERE id=1;
```

binlog 里就是这条 SQL 文本。

优点：

- binlog 小
- 可读性好

缺点（不可控的陷阱）：

```sql
-- 不安全语句
UPDATE t SET col = NOW();    -- 主从执行时 NOW() 不同
UPDATE t SET col = RAND();   -- 随机
UPDATE t SET col = UUID();   -- 不确定
UPDATE t LIMIT 10;           -- LIMIT 没 ORDER BY → 主从行序不同
INSERT INTO t SELECT ... FROM big_table;  -- 多个并发可能死锁不一致
```

若 `binlog_format=MIXED`，MySQL 检测到上述不安全语句会自动改用 ROW 记录该条；纯 `STATEMENT` 模式下只发 warning，仍按文本写入，复现风险得由调用方承担。

### 2.2 ROW —— 记录行的变化（默认）

```
Table_map: `mydb`.`users`
Update_rows:
  WHERE id=1, name='Alice', age=20
  SET id=1, name='Bob', age=20
```

二进制记录变化前后两份镜像。

优点：

- **精确**——主从必定一致
- 安全：不存在不确定函数问题
- 工具友好：Canal / Debezium 直接解析

缺点：

- binlog 大（一次 `UPDATE 100w 行` → 100w 行 image 写进 binlog）
- 排查不直观（要用 `mysqlbinlog --base64-output=DECODE-ROWS -vv` 解码）

**生产标配**：`binlog_format = ROW`（8.0 默认就是 ROW）。

### 2.3 MIXED —— 自动切换

DML 默认 STATEMENT，遇到不安全语句切 ROW。

实践上 MIXED 很少用——8.0 直接 ROW，逻辑简单。

### 2.4 binlog_row_image

ROW 格式下记录 image 的详细度：

| 值 | 行为 |
|---|---|
| **FULL（默认）** | 前后镜像都记全部列 |
| MINIMAL | 前镜像只记 PK + 改动列；后镜像只记改动列 |
| NOBLOB | 同 FULL 但 BLOB/TEXT 不变时省略 |

MINIMAL 省 binlog 50%+ 体积，但**下游 CDC 拿不到完整数据**——Canal、Debezium 同步到 Elasticsearch 需要完整 image 才能更新。生产一般 FULL。

---

## 第三章：复制拓扑

### 3.1 经典异步复制

```
Master ─── binlog ───> Replica
```

Master 提交不等 replica，性能最高，可能丢数据。

### 3.2 半同步复制（semi-sync）

```
Master ─── binlog ───> Replica
       ←── ACK ─────── Replica
```

Master 提交前等至少一个 replica ACK 收到 binlog。性能略降，**已提交事务不丢**（前提：replica 没崩）。

```sql
INSTALL PLUGIN rpl_semi_sync_source SONAME 'semisync_source.so';
INSTALL PLUGIN rpl_semi_sync_replica SONAME 'semisync_replica.so';
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_replica_enabled = 1;
```

超时退化（`rpl_semi_sync_source_timeout`，默认 10s）：如果 replica 没 ACK，master 退化为异步。

### 3.3 全同步（Group Replication / MGR）

```
Master ── propose ─→ Replicas  (Paxos)
       ←── majority ACK
       commit
```

Paxos 协议保证多数派一致。强一致但写入吞吐受网络 RT 限制。

详见 M09。

### 3.4 多源复制（multi-source）

```
Source A ──┐
Source B ──┼─→ Replica（应用多份 binlog）
Source C ──┘
```

5.7+ 支持。常见用法：把多个业务库聚合到一个分析库。

---

## 第四章：传统 file/position 复制

### 4.1 配置流程

```sql
-- 在 master 创建复制账号
CREATE USER 'repl'@'%' IDENTIFIED BY 'pwd';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- 在 master 查 binlog 位置
SHOW MASTER STATUS;  -- file: binlog.000005, position: 1234

-- 在 replica 配置
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='<master-ip>',
    SOURCE_USER='repl',
    SOURCE_PASSWORD='pwd',
    SOURCE_LOG_FILE='binlog.000005',
    SOURCE_LOG_POS=1234;
START REPLICA;
```

### 4.2 复制线程

Replica 端两个线程：

- **IO Thread**：连 master 拉 binlog event，写本地 relay log
- **SQL Thread**：读 relay log 应用到本地

```
[Master binlog] →← IO Thread → [Relay log] → SQL Thread → [Replica data]
```

### 4.3 文件位置的痛点

- 故障转移时新 master 的 file/position 跟旧 master 不同，replica 接新 master 要算 offset
- 计算极易出错
- 错位一次就数据不一致

GTID 解决这些问题。

---

## 第五章：GTID 复制

### 5.1 GTID 是什么

`Global Transaction ID` = `<server_uuid>:<transaction_id>`。

```
3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

每个事务在主库分配一个全局唯一 ID。从库知道自己"看过哪些 GTID"，主从故障切换时**replica 自动从未看过的 GTID 接续**。

### 5.2 启用 GTID

```ini
gtid_mode = ON
enforce_gtid_consistency = ON
log_replica_updates = ON    # 从库执行的也写 binlog（多级复制需要）
```

切换是滚动的（OFF → OFF_PERMISSIVE → ON_PERMISSIVE → ON），生产升级要走完。

### 5.3 GTID 复制配置

```sql
CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='...',
    SOURCE_USER='repl', SOURCE_PASSWORD='...',
    SOURCE_AUTO_POSITION=1;     -- 关键：自动从未执行的 GTID 开始
START REPLICA;
```

不再传 file/position——GTID 自己算。

### 5.4 GTID 的便利

```
Master A 挂 → 提升 Replica B 为新 Master
旧的 Replica C 怎么接 B？
```

传统 file/position：

- 在 B 上查 SHOW MASTER STATUS
- 在 C 上停 replica，CHANGE TO 新 file/pos
- 不知道 C 已经执行到 A 的哪条 → 错位风险

GTID：

- C 知道自己 Executed_Gtid_Set 是 `A_uuid:1-1000`
- C 接 B，B 把 1001 开始的发给 C
- B 自己已经是 A_uuid:1-1500，C 自动补 1001-1500，然后跟上 B 的新 GTID

零配置错位。

### 5.5 GTID 限制

- **CREATE TABLE ... SELECT** 不支持（事务边界混乱）
- 临时表 + 事务混用受限
- 跨多个事务的 DDL 受限

`enforce_gtid_consistency=ON` 在违规时直接报错而非默默不一致。

---

## 第六章：从库延迟

### 6.1 延迟的根因

- **大事务**：主库 1 秒执行的 10w 行 UPDATE，binlog 1 个事务，从库串行回放也 1 秒（这是最佳情况）
- **慢 SQL**：从库执行某条 SQL 比主库慢（如索引不一致、配置差异）
- **单线程回放**：5.6 前 SQL Thread 串行 → 主库多线程 vs 从库单线程 → 必然延迟
- **网络**：跨机房带宽 / 丢包

### 6.2 看延迟

```sql
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: 234
```

**Seconds_Behind_Source 的陷阱**——它衡量"replica 当前在执行的 binlog event 距离 master 当前时间多久"：

- 如果 replica 完全没收到新 binlog，**显示 0**（虽然实际延迟）
- 不准确，仅作粗略参考

更准方法：在 master 写个心跳表 `INSERT INTO heartbeat VALUES (NOW())`，在 replica 看心跳表与本地 NOW() 的差。`pt-heartbeat` 工具就是干这个。

### 6.3 并行复制（MTS, Multi-Threaded Slave）

5.6：按 database 并行（同 database 内仍串行）。
5.7+：**基于 group commit 的逻辑时钟并行** —— 主库同一组 group commit 内的事务在 replica 可以并行执行。
8.0：**WRITESET 并行** —— 分析事务改动的行集合，无冲突则并行。

启用：

```ini
replica_parallel_workers = 16
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = ON   -- 保持提交顺序与 master 一致
binlog_transaction_dependency_tracking = WRITESET  -- 8.0 主库这边
```

效果：从库吞吐基本能跟上主库。

### 6.4 心跳表（无延迟的从库）

如果从库延迟监控显示 0 而你怀疑——心跳表能给出真相：

```sql
-- master
CREATE TABLE heartbeat (ts DATETIME(6));
INSERT INTO heartbeat VALUES (NOW(6)) ON DUPLICATE KEY UPDATE ts=NOW(6);
-- 每秒一次（用 event scheduler 或外部）

-- replica
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6))/1e6 AS lag_seconds FROM heartbeat;
```

---

## 第七章：复制中断

### 7.1 中断的常见原因

- 主库 row 中包含从库不存在的表 / 列
- 主从字符集 / 排序规则差异
- 从库本地有违反唯一约束的数据
- 从库参数（如 `sql_mode`）跟主库不同
- 网络断开 IO Thread 重连后位置错乱
- replica 自己当 master 给下游用，下游写入污染了 replica（log_replica_updates）

### 7.2 排查流程

```sql
SHOW REPLICA STATUS\G
-- Replica_IO_Running: Yes/No
-- Replica_SQL_Running: Yes/No
-- Last_IO_Error / Last_SQL_Error
-- Last_Error 文本（具体哪条 SQL 报什么错）
-- Exec_Source_Log_Pos / Relay_Log_Pos
```

看哪个线程挂了。

### 7.3 跳过错误

**慎用**——会引入主从不一致。

```sql
SET GLOBAL sql_replica_skip_counter = 1;
START REPLICA;
```

或基于 GTID 跳过：

```sql
SET GTID_NEXT = '<server-uuid>:42';
BEGIN; COMMIT;
SET GTID_NEXT = AUTOMATIC;
START REPLICA;
```

跳过后**一定要用 pt-table-checksum + pt-table-sync 验证差异并修复**。

### 7.4 用 mysqlbinlog 查具体哪条 SQL

```bash
mysqlbinlog --start-position=<pos> --stop-position=<pos+1000> \
    --base64-output=DECODE-ROWS -vv binlog.000005
```

decode 出来看 ROW 模式下的具体 before/after image。

---

## 第八章：时间点恢复（PITR）

### 8.1 全备 + binlog 重放

```
00:00 全备 (mysqldump / Percona XtraBackup / Clone)
00:00 - 12:00 binlog 累积
12:00 ：发现误操作 DROP TABLE x;
```

恢复到 11:59:

1. 还原全备
2. 用 binlog 重放到 11:59 那一秒

```bash
mysqlbinlog --start-datetime='2026-05-13 00:00:00' \
            --stop-datetime='2026-05-13 11:59:00' \
            binlog.000001 binlog.000002 ... | mysql
```

或基于 GTID：

```bash
mysqlbinlog --exclude-gtids='3E11FA47-...:1234' ... | mysql
```

### 8.2 误操作 DROP/DELETE 的应急

- **DROP TABLE**：binlog 是 DDL，找到那条之前的位置 PITR 恢复
- **DELETE FROM t WHERE ...**：ROW binlog 里有完整 before image，可解析反转成 INSERT
- 工具：`binlog2sql`、`my2sql`、`mysql-replay`

### 8.3 备份策略

- mysqldump：逻辑备份，慢但跨版本兼容
- Percona XtraBackup：物理备份，快
- MySQL Enterprise Backup：官方付费
- Clone Plugin（8.0+）：内置在线克隆

定期演练恢复——**没演练的备份等于没备份**。

---

## 第九章：复制 + 故障转移

### 9.1 手动 failover

```
Master A 挂
-> 选 Replica B（最新的）
-> B: STOP REPLICA; RESET REPLICA ALL; SET GLOBAL read_only=OFF;
-> 把流量切到 B
-> 老的 Replica C: CHANGE REPLICATION SOURCE TO ... SOURCE_HOST='B'
```

复杂、易错。生产用工具：

- **MHA**（老牌，5.5 时代）
- **Orchestrator**（GitHub 出品，强）
- **MySQL Router + MGR**（官方栈）

### 9.2 数据丢失风险

异步复制下，master 挂时可能 binlog 还没传到 replica。提升 replica 为 master → 这部分事务永久丢。

半同步缓解但不彻底（多机房网络差时半同步退化为异步）。

强一致用 MGR。

### 9.3 split-brain

主库网络隔离，被认为挂了 → 选新主。旧主网络恢复 → 两边都接受写。

防御：

- failover 工具自带 fencing（强制旧主只读 / 杀掉）
- 应用层 VIP / DNS 切换
- 客户端订阅故障转移事件

---

## 第十章：监控

### 10.1 关键指标

```sql
SHOW REPLICA STATUS\G

-- Replica_IO_Running / Replica_SQL_Running
-- Seconds_Behind_Source（参考）
-- Last_Error
-- Retrieved_Gtid_Set (IO 拉到哪)
-- Executed_Gtid_Set (SQL 执行到哪)
-- Auto_Position
```

### 10.2 心跳延迟

```sql
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6))/1e6 FROM heartbeat;
```

### 10.3 Prometheus 监控项

- `mysql_slave_status_seconds_behind_master`
- `mysql_slave_status_slave_io_running`
- `mysql_slave_status_slave_sql_running`

告警：

- 延迟 > 30 秒
- IO/SQL Thread 任一停止

### 10.4 主从一致性校验

```bash
pt-table-checksum --replicate=mydb.checksums --host=master ...
pt-table-sync --replicate=mydb.checksums --execute master_host slave_host
```

定期跑一次（每周）。

---

## 第十一章：生产实战

### 11.1 拓扑设计

简单业务：

```
       Master
      /   |   \
    R1   R2   R3 (其中一个是热备)
```

跨机房：

```
机房 A:        机房 B:
Master ── 半同步 ── Replica(热备, 跨机房)
  |                    |
  R1                  R2(本地读)
```

写量大 + 强一致：

```
MGR Cluster:
[Member A] [Member B] [Member C]
    |          |          |
    └──────────┴──────────┘
       MySQL Router 路由
```

### 11.2 大事务限制

```ini
binlog_row_event_max_size = 8K   # 控制单 event 大小
```

应用层拆批：每批 1000-5000 行 commit。

### 11.3 在线 DDL 与复制

```sql
ALTER TABLE huge_table ADD COLUMN x INT;
```

如果在 master 跑，从库会**串行回放**这个 ALTER，期间从库该表读阻塞。

工具：gh-ost / pt-online-schema-change 用影子表 + 触发器（或解析 binlog），写完后 atomic rename，**主从都不阻塞读**。

### 11.4 binlog 占盘

ROW + binlog_row_image=FULL + 高写入 = binlog 飞涨。

- 设 `binlog_expire_logs_seconds` 7 天
- 用专门盘装 binlog（与数据盘分离）
- 监控 `binary_log_files` 总大小

### 11.5 复制账号最小权限

```sql
CREATE USER 'repl'@'%' IDENTIFIED BY '...';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
-- 只给 REPLICATION SLAVE
```

很多人把复制账号给了 ALL PRIVILEGES，被 replica 主机入侵 = 主库失守。

---

## 第十二章：与其他章节连接

- **与 M03 事务**：binlog 跟 redo 的 2PC
- **与 M04 锁**：大事务持锁，binlog 也大
- **与 M07 PS**：Performance Schema 监控复制线程状态
- **与 M09 高可用**：MGR / InnoDB Cluster 基于复制
- **与 M12 分库分表**：每个分片仍要主从

---

## 总结

binlog 与复制：

1. **格式**：8.0 默认 ROW；MINIMAL 省体积但 CDC 不友好
2. **2PC**：redo prepare → binlog → redo commit
3. **GTID**：替代 file/position，故障转移友好
4. **半同步**：至少一 replica ACK，超时退化
5. **MGR**：Paxos 强一致，多写不便
6. **并行复制**：8.0 WRITESET，从库不延迟
7. **PITR**：全备 + binlog 重放
8. **故障转移**：用 Orchestrator / MGR + Router
9. **校验**：pt-table-checksum 定期跑
10. **监控**：心跳表 + Prometheus

---

## 练习题

1. 解释 row / statement / mixed 各自的不安全场景，举 2 个具体例子。
2. 启用 GTID 后，从库 `Executed_Gtid_Set` 含义？怎么用它跳过单个事务？
3. 半同步退化为异步的条件是什么？业务能感知吗？
4. 一个 100w 行的 UPDATE 在 master 跑了 10 秒，从库延迟会怎样？怎么优化？
5. `pt-heartbeat` 比 `Seconds_Behind_Source` 准确，原因是什么？
6. 设计一个跨 3 机房 + 1 主 6 从的拓扑，并标注半同步 / 异步关系。
7. 误删了 1000 行数据，binlog 是 ROW 模式 → 给出反向恢复的步骤。
8. master crash 时，半同步 vs MGR 各有什么数据安全保证？
9. 在线 DDL 加列对从库延迟的影响？用 gh-ost 如何避免？
10. binlog 占盘 80% 报警 → 排查与处理流程？

---

> 🔁 反馈：本地起 master+replica 双节点，故意制造一次复制错误，亲手修复一次
