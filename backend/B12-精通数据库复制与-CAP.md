# 精通数据库复制与 CAP 定理

> 课程编号：B12
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Replication & CAP
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：分布式的代价

```
Primary DB 挂了
30 秒后告警
2 分钟后人工介入
5 分钟后服务恢复
```

可用性 99.9% 一年允许约 8.7 小时宕机；99.99% 仅 52 分钟。要做到，必须**复制**。但复制引入新问题：副本与主库不一致时怎么办？跨数据中心的网络抖动怎么办？分布式 ACID 还可能吗？本章拆开复制模式、共识算法、CAP 定理、典型选型。

---

## 第一章：为什么复制

### 1.1 三大目标

1. **高可用**：主挂了副本顶上
2. **读扩展**：读流量分散到多副本
3. **地理就近**：副本部署到用户附近

### 1.2 代价

- 复杂度：故障切换、路由、同步延迟
- 数据延迟：副本可能滞后几毫秒到几秒
- 一致性 trade-off：见 CAP

---

## 第二章：复制拓扑

### 2.1 Single-Leader（主从）

```
Primary  ───→  Replica 1
         ───→  Replica 2
         ───→  Replica 3
```

- 写只能去 primary
- 读可以从任何副本
- primary 挂 → failover 一个 replica

最常见。PostgreSQL、MySQL、MongoDB 默认模式。

### 2.2 Multi-Leader（多主）

```
Leader A ←──→ Leader B
   │             │
   ↓             ↓
Replica A    Replica B
```

- 任何 leader 接写
- leader 间互相复制
- 写冲突需解决

适合多数据中心写入需求。复杂度高（冲突解决）。

### 2.3 Leaderless

```
        Client
       /  |  \
      W   W   W
     N1  N2  N3
       同时写到 N 个
       quorum 读取
```

Dynamo 系列（Cassandra、Riak、ScyllaDB）。无固定 leader，每节点对等。

---

## 第三章：同步策略

### 3.1 Synchronous（同步）

```
Client → Primary → Replica（写 + ack）→ Primary 才 ack Client
```

- 数据零丢失保证
- 慢（等所有副本）
- 副本挂 → primary 挂

### 3.2 Asynchronous（异步）

```
Client → Primary → ack
         ↓ 异步
         Replica
```

- 快
- 副本可能滞后
- primary 挂 + 副本未追上 → 数据丢失

### 3.3 Semi-Synchronous

```
Primary 等至少 1 个副本 ack 后才 ack Client
其他副本异步
```

平衡：保证至少一份副本有数据 + 单副本挂不阻塞。MySQL 复制默认是异步；semi-sync 是可选插件，需在主从两端显式启用，超时（默认 10s）会自动降级回异步。

### 3.4 PostgreSQL 配置

```
synchronous_commit = on              ← WAL 落盘
synchronous_standby_names = 'rep1'   ← 等 rep1 ack
```

可调到 `remote_apply` 等更严格级别。

---

## 第四章：复制实现机制

### 4.1 基于 WAL（write-ahead log）

PostgreSQL 用流复制：primary 把 WAL 推给副本，副本 replay。

```
PostgreSQL primary:
WAL: [INSERT user 42, UPDATE balance, ...]
  → stream to replica
Replica: replay WAL → 同样的状态
```

物理复制——字节级一致。

### 4.2 基于 binlog（MySQL）

```
MySQL primary:
binlog: 行变更 (statement / row / mixed)
  → stream to replica
Replica: replay
```

逻辑复制（按行）或语句复制。

### 4.3 逻辑复制

按业务事件（INSERT、UPDATE 高层抽象）传输。优势：
- 跨版本 / 跨引擎
- 选择性复制（仅某些表）
- 反规范化下游

PostgreSQL 10+ 内置 logical replication；Debezium 把 binlog 转成 Kafka 事件。

### 4.4 PostgreSQL 18 关键升级（2025-09 发布）

> 2026 年新建 PostgreSQL 集群应直接用 18，老集群规划 17 → 18 升级路径。

- **Asynchronous I/O**：批量异步发起 read 请求（worker / io_uring / sync 三种 io_method），顺序扫描 / bitmap heap scan / vacuum 性能可达 3x。Linux 5.1+ 上 `io_method = io_uring` 最优。
- **UUIDv7** 内置：`uuidv7()` 函数生成 timestamp-ordered UUID，B-tree 索引插入更友好（少 page split），主键终于可以"随机 + 有序"两全。
- **Virtual generated columns 默认**：generated column 默认按"读时计算"，不再写时 materialize（除非显式 `STORED`）。和 MySQL/SQL Server 行为一致。
- **OAuth 2.0 客户端认证**：直接对接 IdP / SSO，不再需要 plug-in。
- **OLD / NEW in RETURNING**：`UPDATE ... RETURNING OLD.*, NEW.*` 拿改前改后值，省一轮 SELECT。
- **Skip scan**：多列 B-tree 索引在前缀列基数低时也能用上。
- **Major upgrade 加速**：pg_upgrade 大幅缩短停机时间。

参考：[PostgreSQL 18 release notes](https://www.postgresql.org/docs/18/release-18.html)。

### 4.5 分布式 SQL 替代品（2026 选型）

如果你 outgrew 单 PostgreSQL（写入分片、跨地域、强一致 + 弹性扩展），主流分布式 SQL 选项：

| 产品 | 兼容性 | 一致性 | 适用 |
|---|---|---|---|
| **CockroachDB** | PostgreSQL 协议 | Serializable + multi-region | OLTP，跨地域强一致 |
| **YugabyteDB** | PostgreSQL + Cassandra | Serializable | OLTP，PG 生态完整 |
| **TiDB** | MySQL 协议 | Snapshot Isolation | HTAP（OLTP + 分析） |
| **Vitess** | MySQL sharding | 分片内 | 超大规模 MySQL（YouTube/Slack 在用） |
| **Spanner / AlloyDB** | 自有 / PG | external consistency | GCP 托管，跨地域 |

通用规律：**Raft/Paxos 共识 + 分区式存储**。延迟会比单机 PG 高（多 RTT 共识），换来"水平扩展 + 不丢数据"。

---

## 第五章：CAP 定理

### 5.1 三个属性

- **Consistency**：所有节点同一时间看到同样数据
- **Availability**：每个请求都得到响应（不一定是最新）
- **Partition tolerance**：网络分区时系统继续工作

### 5.2 定理

**网络分区发生时**，只能在 C 和 A 之间选一个。

```
       P 发生：节点间断了
         ↓
        要么:
   - 拒绝服务（保 C，弃 A）
   - 接受可能不一致（保 A，弃 C）
```

### 5.3 现实

网络分区**必然**会发生（哪怕短暂）。所以实际是：

- **CP**：分区时拒服务，保证一致
- **AP**：分区时继续服务，允许不一致
- **CA**：理论上不存在分布式系统（因为 P 不可避免）

### 5.4 选型

| 系统 | 倾向 |
|---|---|
| PostgreSQL（单主）| CP——主挂了不接写 |
| MongoDB | CP（默认）/可调 |
| Cassandra | AP |
| DynamoDB | AP（可选 strong consistency） |
| Etcd / Zookeeper | CP |
| Redis（主从异步）| AP-ish |

### 5.5 PACELC 扩展

CAP 只讲分区时；**平时（else）** 还有 latency vs consistency trade-off：

- **PA/EL**：分区时 A，平时 L（低延迟）—— Cassandra
- **PC/EC**：分区时 C，平时 C（强一致）—— BigTable、HBase

---

## 第六章：一致性级别

### 6.1 强一致（linearizable）

操作看起来按某种"全局"顺序执行；读永远返回最新写。代价：synchronous + 高延迟。

### 6.2 顺序一致（sequential）

同一 client 看到操作的顺序一致；不同 client 看到的顺序可能不同。

### 6.3 因果一致（causal）

有因果关系的操作保持顺序；并发操作可乱序。

### 6.4 最终一致（eventual）

如果停止写入，最终所有副本会一致。但中间可能任意乱。AP 系统默认。

### 6.5 选择

- 账户余额、库存：强一致
- 用户资料、社交动态：最终一致
- 计数器、点赞：最终一致
- 抢购：强一致 + 锁

---

## 第七章：共识算法

### 7.1 Raft

Etcd、Consul、CockroachDB 使用。

- 选举：一个 leader，其他 follower
- log replication：leader 收到写，复制到 majority 后 commit
- failover：leader 失联 → 重新选举

3 节点容忍 1 节点挂；5 节点容忍 2 节点挂；**容忍数 = (N-1)/2**。

### 7.2 Paxos

更早提出的算法。复杂；现实多用 Raft（Raft 设计目标就是更易懂）。

### 7.3 共识的代价

每次写需要 majority 副本 ack：

```
N = 3, 跨可用区:
T_write ≈ max(latency to 2 of 3 AZs)
       ≈ 5-15 ms
```

跨大洲：50-300 ms。这是 CockroachDB、Spanner 等"全球数据库"写延迟较高的根本原因。

---

## 第八章：故障切换（failover）

### 8.1 步骤

1. **检测**：心跳超时
2. **选新主**：剩余副本中选 lag 最小的
3. **更新路由**：DNS / load balancer / 服务发现 指向新主
4. **客户端重连**

### 8.2 自动 vs 手动

- **自动**：减少 MTTR，但**脑裂**风险（两个主同时存在）
- **手动**：安全，慢

折中：自动 + 严格 quorum（多数派同意才提升）。

### 8.3 工具

- PostgreSQL：Patroni、Stolon、repmgr
- MySQL：MHA、Orchestrator、ClusterControl
- 云：AWS RDS Multi-AZ 自动切换

### 8.4 split-brain

两个节点都以为自己是 master → 双写、数据不一致。Patroni、Etcd 用 quorum 防止。

---

## 第九章：读副本陷阱

### 9.1 复制延迟

```
Write → Primary (t=0)
       Replica catch up (t=50ms)

Read from Replica at t=20ms → 看不到刚写的数据
```

用户写完立刻读 → "为啥没我刚发的内容？"

### 9.2 解决方案

**A. Read-your-writes 一致性**
```
写完后 30 秒内读走 primary
或者 sticky session 同一 user 始终某副本
```

**B. 客户端 LSN 跟踪**
```
写返回 WAL LSN → 客户端记
读时附 "至少要 LSN ≥ X"
副本未追上时拒绝或 wait
```

**C. 读 primary**
关键读路径绕过副本。代价：primary 压力。

### 9.3 副本越多越好？

不。每副本：
- 额外硬件 / 云成本
- primary 推送 WAL 的带宽
- failover 时多个候选

3-5 个副本通常够。

---

## 第十章：跨数据中心

### 10.1 模式

**单主跨 DC**：primary 在主 DC，副本在备 DC。备 DC 仅读（异地灾备）。

**多主**：每 DC 一个 leader，互复制。冲突解决：last-write-wins、向量时钟、CRDT。

**Spanner 风格**：全球分布 + TrueTime（原子钟）+ Paxos。写延迟高但全局强一致。

### 10.2 跨 DC 延迟

```
Same DC:           0.5 ms
Same region (AWS):  1-2 ms
Cross region (US):  20-80 ms
Global (US-EU):     80-150 ms
Antipodes (US-AU):  200-300 ms
```

跨 DC 同步复制 → 写延迟 = 跨 DC RTT。能否接受看业务（金融 vs 内容）。

---

## 第十一章：生产级最佳实践

1. **主从异步是默认**：99% 场景够。
2. **关键数据 semi-sync**：保证至少 1 副本有数据。
3. **跨可用区部署**：1 个 AZ 挂仍服务。
4. **自动 failover + quorum 验证**：防脑裂。
5. **监控复制延迟**：超过 1 秒告警。
6. **读副本绕过对延迟敏感的读**：写后立刻读走 primary。
7. **全球业务考虑 NewSQL**：CockroachDB / Spanner。
8. **逻辑复制做 OLTP → OLAP**：Debezium 是经典管道。
9. **演练 failover**：定期 chaos engineering。
10. **明确选 CP 还是 AP**：业务需要先想清楚。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：异步副本当强一致用
"读副本" + 写后立刻读 → 用户看不到自己刚提交的内容。

### ❌ 陷阱 2：脑裂没防
两个 primary 双写 → 数据冲突难修。

### ❌ 陷阱 3：跨大洲同步复制
写延迟几百 ms → 业务卡。

### ❌ 陷阱 4：CAP 误解
"我系统 CA"——不存在，分布式必有 P。

### ❌ 陷阱 5：副本太多
5+ 副本反而慢——primary 推送带宽爆。

### ❌ 陷阱 6：忘了演练 failover
出事才发现脚本不工作。chaos 测试。

### ❌ 陷阱 7：未限速副本追赶
副本掉队后狂追 → 把 primary 拖慢。限速重启。

---

## 第十三章：练习题

**练习 1**：解释为何 CockroachDB 跨 region 写比 PostgreSQL 单实例慢。

**练习 2**：用户提交评论后立刻刷新看不到——主从复制延迟 → 怎么解？

**练习 3**：CAP 中"CA"系统存在吗？

**练习 4**：3 节点 Raft 集群，2 节点宕机，会怎样？

**练习 5**：电商抢购：1 件商品 1000 人抢。用 AP 还是 CP？为何？

---

## 参考答案

**练习 1**：CockroachDB 写要 Raft majority 副本 ack。跨 region 部署 → 等 majority 涉及跨 region RTT（50-150ms）。PostgreSQL 单实例本地 fsync ~1ms。代价换的是分布式 ACID + 高可用。

**练习 2**：
- 短期：sticky read（写后 N 秒读 primary）
- 中期：客户端跟踪 LSN，副本未追上时 fallback
- 长期：评论数据库放 user 所在分片，写读都在本地

**练习 3**：理论上"分布式 CA 不存在"——P 必发生。但**单机** DB 是 CA（不分布所以没 P 概念）。CAP 是分布式系统的定理。

**练习 4**：仅 1 节点存活 → 不到 majority（2/3）→ 拒绝写。读可继续（但可能不是最新）。等待 1 个节点恢复才能写。

**练习 5**：CP。库存超卖比延迟更糟。用 SELECT FOR UPDATE 或原子 `UPDATE stock SET n=n-1 WHERE n > 0`。失败的 999 个请求返回 sold-out。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 拓扑 | 主从 / 多主 / leaderless |
| 同步 | sync / async / semi-sync |
| 实现 | WAL / binlog / 逻辑复制 |
| CAP | 分区时只能 CP 或 AP |
| 一致性 | strong / sequential / causal / eventual |
| 共识 | Raft majority；写延迟 = 跨副本 RTT |
| failover | 自动 + quorum 防脑裂 |
| 延迟 | 同 DC ms 级；跨大洲百 ms |

下一篇 **B13 — N+1 问题深度剖析** 将专题讲清各种 N+1 模式、检测、修复。

---

