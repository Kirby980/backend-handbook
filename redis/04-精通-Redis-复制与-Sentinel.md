# 精通 Redis 主从复制与 Sentinel：PSYNC2、故障转移、split-brain

> 关联章节：[05 持久化](./05-精通-Redis-持久化.md)、[07 Cluster](./07-精通-Redis-Cluster.md)、[10 性能模型](./10-精通-Redis-性能模型.md)

---

## 引言：从单点到高可用

单实例 Redis 在挂掉那一刻丢失所有内存数据 + 业务不可用——这是绝大多数生产场景不能接受的。Redis 提供两层"对抗单点"的能力：

1. **主从复制（replication）**：异步把主节点的写入复制到一个或多个从节点；从节点可以承担读请求。**仍然需要人手动做故障转移**。
2. **Sentinel**：在主从之上加一个监控集群，自动检测主节点故障并提升某个从节点为新主。

> 注意：Sentinel 不解决"水平扩展"——所有节点存全量数据。要分片去看 [07 Cluster](./07-精通-Redis-Cluster.md)。

读完之后你应该能：

- 解释 PSYNC2 协议怎么避免"主从断连一会就全量同步"
- 说出 Sentinel quorum 和 majority 各自的意义
- 设计一个不会触发 split-brain 的部署
- 给一次主从切换的全流程画出时序图

---

## 第一章：复制的基本机制

### 1.1 拓扑

```
                  +--------+
                  | Master |
                  +---+----+
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
  +----------+ +-----------+ +-----------+
  | Replica1 | | Replica2  | | Replica3  |
  +----------+ +-----------+ +-----------+
```

或链式（chain replication，节省 master 出口带宽）：

```
Master → Replica1 → Replica2 → Replica3
```

**配置**：

```
# replica 启动
replicaof <master-ip> <master-port>     # 老版本叫 slaveof
masterauth <password>                    # master 设了 requirepass 必填
replica-read-only yes                    # 从节点默认只读
```

### 1.2 复制是异步的

Master 写入 → 立即返回 client → 异步推送到 replica。

后果：

- master 崩溃 → 已 ACK 给 client 但未推到 replica 的写入**会丢**
- replica 在异步追赶 master，存在 lag

**没有原生同步复制**。`WAIT N timeout_ms` 命令可让 client 主动等"至少 N 个 replica 已收到"，但仍不是事务级保证（replica 收到 ≠ 持久化）。

### 1.3 全量同步流程

新 replica 第一次连接 master：

```
1) Replica → Master: PSYNC ? -1            (我没历史，请全量)
2) Master 触发 BGSAVE + 从此刻起把所有写命令缓存到 client-output-buffer
3) BGSAVE 完成 → 发送 RDB 文件给 Replica
4) Replica 接收 RDB + 加载到内存
5) Master 把缓存的命令发给 Replica
6) 进入"在线复制" 状态：master 写一条就发一条
```

**步骤 2-4** 期间 master 要：

- fork 子进程（COW 代价见 R03）
- 缓存命令（占用 client-output-buffer，可能很大）
- 发 RDB（网络带宽）

如果同一 master 同时被多个新 replica 连接，仅做**一次 BGSAVE** + 把 RDB 分发给所有等待的 replica（**redis 优化点**）。

---

## 第二章：PSYNC2 —— 部分重同步的关键

### 2.1 痛点

老协议（Redis 2.6 及之前）：replica 一旦断连重连，**强制全量同步**。即使只断了 1 秒。

100 GB dataset 全量同步要花几分钟到几十分钟——业务高峰这是灾难。

### 2.2 PSYNC（Redis 2.8 引入）

加了一个 **replication backlog**（master 的环形缓冲区，默认 1MB）：

```
Master 把每条复制命令同时写到三处：
  1. 各 replica 的 output buffer（即时推送）
  2. replication backlog（环形 buffer，存最近 N 字节）
  3. AOF（如开启）

Replica 维护一个 offset：当前接收到 master 流的位置（字节）
```

断连重连时：

```
Replica → Master: PSYNC <replication-id> <offset>
Master 检查：offset 是否还在 backlog 范围内？
  在 → 只发缺失的部分（partial resync）
  不在 → 全量同步
```

### 2.3 PSYNC2（Redis 4.0）增强

旧 PSYNC 在两种场景仍退化为全量：

1. **Replica 提升为新 master**：原 replication-id 变了 → 其他 replica 重连时 ID 不匹配 → 全量
2. **Master 重启**：内存 replication-id 丢了 → 全量

PSYNC2 引入：

- **replication-id1 + replication-id2 双 ID**：promote 时记录上一个 ID，让兼容 offset 仍可用
- **持久化 replication-id 到 RDB 头部**：master 重启后能恢复
- **Replica 也写 replication-id**：升级为 master 后能"继承"

效果：90% 的"短暂网络抖动 + master 重启"场景都能 partial resync 完成。

### 2.4 关键配置

```
repl-backlog-size 1mb        # 默认；高写入场景调到 256mb-1gb
repl-backlog-ttl 3600        # backlog 空闲多久后释放（无 replica 连接时）
repl-timeout 60              # master/replica 心跳超时
repl-ping-replica-period 10  # 心跳间隔
repl-diskless-sync yes       # 全量同步直接通过 socket 发 RDB（不落盘）
repl-diskless-sync-delay 5   # 等多少秒让多个 replica 并入同一次全量
```

### 2.5 backlog 大小怎么算

```
backlog_size = max_disconnect_seconds × peak_write_throughput_bytes_per_sec
```

如果业务峰值写 10MB/s，预期最长断连 30 秒：backlog ≥ 300MB。

**反模式**：保留默认 1MB → 网络稍微抖动 1 秒就触发全量同步，引发雪崩。**生产建议 256MB 起步**。

### 2.6 diskless replication（无盘复制）

老版全量同步：master fork → 子进程写 RDB 到磁盘 → 主线程 send 给 replica。两次磁盘 I/O。

`repl-diskless-sync yes`：fork 后子进程**直接把 RDB 流写到 socket**，不落盘。

适用：

- master 磁盘 I/O 慢（HDD / 共享存储）
- replica 多，多次 RDB 文件读写不划算
- master 磁盘空间紧张

不适用：磁盘很快 + replica 数量极多（多 replica 并发 send 容易消耗 CPU）。

---

## 第三章：复制中的几个微妙问题

### 3.1 Replica 写命令

`replica-read-only yes`（默认）保证 client 不能直接写 replica。但**有些命令** master 没有，replica 可能局部执行（如 `DEBUG SLEEP`）。

**陷阱**：`CONFIG SET replica-read-only no` 后向 replica 写入 → master 不知道 → master/replica 数据不一致。这种"局部写"是开发期调试方便，**生产严禁**。

### 3.2 Replica 过期问题

老 Redis：master 上 key 过期时由 master 发 `DEL` 命令给 replica。但**replica 上 lazy expire 不会触发 DEL**——如果业务在 replica 上读到一个"已过期但 master 还没扫到"的 key，会返回值（不一致）。

Redis 3.2+：replica 读取已逻辑过期但未收到主节点 DEL 的 key 时返回 nil（key 仍在内存中，直到主节点 DEL 传来）。Redis 6.0 改进的是主动过期（active expire cycle）算法，新增 `active-expire-effort` 等，并非从节点读返回 nil 的逻辑。

### 3.3 复制延迟监控

```
INFO replication

# master 视角
role:master
connected_slaves:2
slave0:ip=10.0.0.2,port=6379,state=online,offset=12345678,lag=0
slave1:ip=10.0.0.3,port=6379,state=online,offset=12340000,lag=1   ← 落后 1 秒
master_repl_offset:12345678
repl_backlog_size:1048576
repl_backlog_first_byte_offset:11297102
repl_backlog_histlen:1048576

# replica 视角
role:slave
master_host:10.0.0.1
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_repl_offset:12345678
```

**告警项**：

- `master_link_status != up` → 复制断了
- `master_last_io_seconds_ago > 30` → 主从通讯停滞
- `master_repl_offset - slave_repl_offset > 阈值` → lag 过大

### 3.4 min-replicas-to-write（半同步替代）

```
min-replicas-to-write 1
min-replicas-max-lag 10
```

Master 在每次写入前检查：有 ≥1 个 replica 的 lag ≤ 10 秒？没有 → 拒绝写入，返回 `NOREPLICAS` 错误。

**这是 Redis "近似半同步"的手段**——不是真同步（仍异步推送 + ACK），但写入前会确认 replica 存活。能避免"master 唯一可用时仍接受写，崩溃后这些写没复制" 的丢数据场景。

代价：所有 replica 都挂时 master 不可写。

---

## 第四章：Sentinel —— 自动故障转移

### 4.1 拓扑

```
            +-----------+
            | Sentinel1 |
            +-----+-----+
                  |  监控 + 互相通信
       +----------+----------+
       v                     v
+-----------+         +-----------+
| Sentinel2 |---------| Sentinel3 |
+-----------+         +-----------+
       \                    /
        \   监控所有        /
         v                 v
   +--------+         +-----------+
   | Master | <-----  | Replica   |
   +--------+         +-----------+
                      | Replica   |
                      +-----------+
```

**Sentinel 数量奇数（3 或 5）**——quorum 投票需要。

### 4.2 配置

```conf
# sentinel.conf
port 26379                                    # Sentinel 端口
sentinel monitor mymaster 10.0.0.1 6379 2     # 监控的 master：name / IP / port / quorum
sentinel down-after-milliseconds mymaster 5000   # 5 秒无响应判定下线
sentinel parallel-syncs mymaster 1               # 故障转移时多少 replica 并行同步新 master
sentinel failover-timeout mymaster 60000         # 失败转移超时
sentinel auth-pass mymaster <password>           # Redis 密码
```

**quorum**：判定 master 主观下线（SDOWN）后，需要多少 Sentinel 投票确认才转为客观下线（ODOWN）。3 Sentinel 配 quorum=2 即可。

### 4.3 故障检测：SDOWN → ODOWN

```
1) Sentinel 每秒 PING master
2) down-after-milliseconds 内无响应 → 标记 master 为 SDOWN（主观下线，本节点视角）
3) 通过 pub/sub 询问其他 Sentinel："你们也认为它下线吗？"
4) 收集到 ≥ quorum 个 SDOWN 投票 → 标记为 ODOWN（客观下线）
5) 开始故障转移流程
```

### 4.4 选举 Leader Sentinel

要做故障转移，需要选一个 leader 主持。基于 **Raft-like 选举**：

```
1) 检测到 ODOWN 的 Sentinel 发起 "我要当 leader" 请求
2) 其他 Sentinel 在当前 epoch 内只投一票（先到先得）
3) 收到 majority（N/2 + 1）票 → 成为 leader
4) Leader 开始挑选新 master 候选
```

**majority** = `(num_sentinels / 2) + 1`。

> 注意：判定 ODOWN 用 **quorum**（你配的值，可能小于 majority），但 leader 选举必须 **majority**（防 split-brain）。

### 4.5 挑选新 master

Leader 在所有 replica 中选最优的：

```
1) 排除 disconnected / lag > 10*down-after-milliseconds 的
2) 按 replica-priority（数字，越小越优先；0 表示永不被选）
3) 同 priority 中比较 replication offset（最新的）
4) 同 offset 比较 runid（字典序最小的）
```

业务可通过 `CONFIG SET replica-priority 0` 把某个 replica 设为"永不当 master"（如跨机房备份节点）。

### 4.6 故障转移流程

```
1) Leader Sentinel 选出新 master 候选 X
2) 给 X 发 SLAVEOF NO ONE → X 变成 master
3) 给其他 replica 发 SLAVEOF X:port → 它们连到新 master 重新同步
4) parallel-syncs 控制并行度（默认 1 → 一个一个同步，慢但稳）
5) 老 master 上线后会被自动配置为新 master 的 replica
6) Leader Sentinel 把切换信息广播给所有 Sentinel + 通过 pub/sub 通知 client
```

整个流程典型 5-30 秒。

### 4.7 客户端感知故障转移

老 client 缓存 master 地址 → 失效后报错。现代 client（Lettuce、Jedis、redis-py 等）支持 Sentinel 模式：

```python
import redis
r = redis.Sentinel([("sentinel1", 26379), ("sentinel2", 26379), ("sentinel3", 26379)])
master = r.master_for("mymaster", socket_timeout=0.5)
replica = r.slave_for("mymaster")
```

客户端启动时连 Sentinel 拿 master 地址，订阅 `+switch-master` 频道。故障转移时 Sentinel 发布消息 → 客户端切换。

---

## 第五章：Split-Brain（脑裂）—— 部署的核心要求

### 5.1 经典场景

```
Region A:  master + sentinel1 + sentinel2
Region B:  replica + sentinel3

网络分区！A B 互相看不到。

Region A 视角：
  master 正常，2/3 Sentinel 可见 → 一切正常
  写入继续 → 数据写到 master

Region B 视角：
  3 个 Sentinel 中只看到 1 个（自己），未达 quorum，不能故障转移
  → 等待

网络恢复后：
  原 master 收到 Region B 的 replica + sentinel3 信息
  发现自己确实是 master → 一切恢复
  
后果：Region B 收到的写请求只能"等"，不会引发数据冲突。安全。
```

### 5.2 真正的脑裂

```
Region A:  master + sentinel1
Region B:  replica1 + sentinel2 + sentinel3

网络分区！

Region A: master 仍接受写（quorum 不足，无法转移，但本机服务）
Region B: 3 个 Sentinel 中 2 个可见 → 达到 quorum → 选 replica1 为新 master
         → replica1 接受写

现在两个 master 各自接受写！网络恢复后老 master 被降级为 replica，
  其在分区期间收到的写入会被新 master 全量覆盖 → 数据丢失。
```

### 5.3 防御：min-replicas-to-write

```
min-replicas-to-write 1
min-replicas-max-lag 10
```

让 master 在"没有任何 replica 同步"时拒绝写入。分区时老 master 看不到任何 replica → 直接拒写 → 无新增脏数据。

### 5.4 部署原则

1. **Sentinel 数量奇数 ≥ 3**：3 容忍 1 节点故障，5 容忍 2
2. **Sentinel 跨可用区**：不要全在一个机房；分布在多个 AZ 形成 majority
3. **master + 至少 1 replica + 多数 Sentinel 同一区域**：故障转移成功率最高
4. **min-replicas-to-write 1 是必备**：哪怕牺牲一些可用性

---

## 第六章：典型生产架构

### 6.1 单机房 3 Sentinel + 1 Master + 2 Replica

```
Node A: Master
Node B: Replica1 + Sentinel1
Node C: Replica2 + Sentinel2 + Sentinel3 (or one more dedicated)
```

- 任一节点挂掉 → 至少 2 Sentinel 在线 → quorum + majority 都满足 → 自动转移
- 适合中小集群

### 6.2 跨机房灾备：3 机房均衡

```
DC1: Master  + Sentinel1
DC2: Replica + Sentinel2
DC3: Replica + Sentinel3 (priority=0, 永不当 master)
```

DC3 是"见证人"——单纯参与 Sentinel quorum，节点 priority=0 防止"跨机房网络抖动"误将远端节点选为 master。**跨机房做 master 会让写延迟拉长**。

### 6.3 Sentinel 与 Cluster 选择

| 维度 | Sentinel | Cluster |
|---|---|---|
| 数据分片 | 否（全量复制） | 是（16384 slot） |
| 自动故障转移 | 是 | 是 |
| 可水平扩展 | 否（master 单点容量上限） | 是 |
| Client 复杂度 | 中（Sentinel-aware） | 高（Cluster-aware） |
| 跨 slot 事务 | 支持 | 不支持（除非 hash tag） |
| 适合 | 中小规模 + 简单 | 大规模 + 高吞吐 |

**经验值**：单实例 < 25 GB / 单实例 QPS < 80k → Sentinel 足够；超过 → Cluster。

---

## 第七章：观察工具

### 7.1 Sentinel 状态

```
redis-cli -p 26379

SENTINEL master mymaster              # 当前 master 信息
SENTINEL replicas mymaster            # 当前 replica 列表
SENTINEL sentinels mymaster           # 其他 Sentinel
SENTINEL ckquorum mymaster            # 检查是否有足够 quorum 故障转移
SENTINEL failover mymaster            # 手动触发一次故障转移（测试用）
SENTINEL reset mymaster               # 重置该 master 的所有状态（紧急用）
```

### 7.2 关键日志

Sentinel 日志里会看到：

```
+sdown master mymaster 10.0.0.1 6379       ← 主观下线
+odown master mymaster 10.0.0.1 6379 #quorum 2/2  ← 客观下线
+new-epoch 7
+vote-for-leader abc123 7
+config-update-from sentinel ...
+switch-master mymaster 10.0.0.1 6379 10.0.0.2 6379   ← 切换完成
+slave-reconf-sent slave ...
+slave-reconf-done slave ...
```

### 7.3 客户端日志

健康的客户端在 failover 期间应该：

1. 一段时间（5-30 秒）报 ConnectionError
2. 自动重连 + 拿新 master 地址 + 业务恢复

如果错误持续 >1 分钟没恢复——多半是客户端没正确订阅 `+switch-master`，要升 SDK 版本。

---

## 第八章：生产级最佳实践

1. **Sentinel 奇数 ≥ 3，跨 AZ 部署**——至少有 majority 在大多数故障下仍能开会
2. **`min-replicas-to-write 1 + min-replicas-max-lag 10`**——防裂脑误写
3. **`repl-backlog-size` 至少 256MB**——避免短断引发全量同步
4. **`repl-diskless-sync yes`**——多 replica + SSD 充足下推荐
5. **`replica-priority` 跨 AZ 节点配 0**——防误选远端 master
6. **客户端必须 Sentinel-aware**——直接连 master IP 是埋雷
7. **`down-after-milliseconds` 别调太小**——5000ms 是合理起点，太短易误判
8. **客户端 reconnect 间隔 + retry**——故障转移期间几十秒不可写，业务要降级或排队
9. **观察 lag 与 master_link_status**——上 Prometheus alert
10. **不要 master / Sentinel 共机**——Sentinel 是监督者，跟被监督对象同生死违背设计意图

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：2 个 Sentinel
quorum 2 + majority 2 = 任一 Sentinel 挂掉就死锁。**至少 3 个**。

### ❌ 陷阱 2：把 Sentinel 装在 master 上
master 故障时 Sentinel 也挂——观察者自己消失。

### ❌ 陷阱 3：repl-backlog-size 默认 1MB
高写入业务网络抖动 1 秒就触发全量同步。改 256MB+。

### ❌ 陷阱 4：忘了 masterauth
master 设了 requirepass 但 replica 没设 masterauth → replica 连不上 master → 静默丢复制。

### ❌ 陷阱 5：min-replicas-to-write 1 但只有 1 个 replica
该 replica 一挂 master 拒写。生产至少 2 replica + min-replicas-to-write=1 才有冗余。

### ❌ 陷阱 6：客户端直连 master IP
故障转移后 master IP 变了，客户端报错连不上。改用 Sentinel SDK。

### ❌ 陷阱 7：客户端在 failover 期间不重试
导致瞬间大量业务失败。SDK 配 retry-on-timeout + 指数退避。

### ❌ 陷阱 8：parallel-syncs 设得过大
多 replica 同时全量同步 → master 网络饱和 / fork 风暴。默认 1 是稳妥的。

### ❌ 陷阱 9：sentinel.conf 不能容忍手动修改
Sentinel 运行时会**主动改写**自己的配置文件（记录 known sentinel / replica）。手动改完忘了 SENTINEL FLUSHCONFIG。

### ❌ 陷阱 10：把 Sentinel 当数据库代理
Sentinel 只是控制面，不代理数据。客户端拿到 master 地址后**直连 master**，不经过 Sentinel。

---

## 第十章：练习题

**练习 1**：3 Sentinel + 1 Master + 2 Replica 的拓扑里，最多能容忍几个节点宕机仍能自动 failover？分别列出。

**练习 2**：业务峰值写入 50 MB/s，最长可能网络抖动 60 秒。算出 repl-backlog-size 应至少多大。

**练习 3**：Master 突然进程死掉，Sentinel 检测 → ODOWN → 选 leader → 选新 master → 切换。整个流程预期耗时多少？给出每一步的典型耗时。

**练习 4**：跨 3 机房部署 Redis 高可用。给出 Sentinel 数量与分布、master/replica 分布、priority 配置。

**练习 5**：客户端用 redis-py，业务 P99 是 10ms。故障转移期间业务表现是什么？怎么让"用户感知最小"？

---

## 参考答案

**练习 1**：
- 容忍 1 Sentinel 宕机：剩 2 Sentinel 仍有 majority(2/3) → ✓
- 容忍 1 Sentinel + 1 Replica：剩 2 Sentinel + master + 1 replica → ✓
- 容忍 Master：触发 failover，2 Sentinel 投票完成，promote 一个 replica → ✓
- 容忍 Master + 1 Sentinel：剩 2 Sentinel 仍可投票（majority 2/3）→ ✓
- 不能容忍 2 Sentinel + Master 同时挂：剩 1 Sentinel 无 majority → 无法 failover

**练习 2**：50 MB/s × 60s = 3000 MB = **3 GB**。生产建议 4-8 GB 留余量。

**练习 3**：典型耗时
- `down-after-milliseconds` 等待：5 秒（配置值）
- SDOWN → ODOWN（pub/sub 询问）：100ms
- Leader Sentinel 选举：100-500ms
- 选新 master + SLAVEOF NO ONE：< 100ms
- 客户端订阅 +switch-master 收到通知：< 1 秒
- 客户端重连 + 业务恢复：100-500ms
- **总计 6-8 秒**（典型）；最长 ~30 秒

**练习 4**：

```
DC1: Master + Sentinel1
DC2: Replica1 (priority=100) + Sentinel2
DC3: Replica2 (priority=0, 永不当 master) + Sentinel3
```

理由：
- Sentinel 跨 3 机房，任一机房挂仍有 majority(2/3)
- master 在 DC1，replica1 在 DC2 候补；replica2 在 DC3 仅做异地容灾，priority=0 避免远端跨机房当 master
- 配 `min-replicas-to-write 1 min-replicas-max-lag 10` 防 split-brain

**练习 5**：故障转移期间
- 头 5 秒：客户端命令 timeout（master 死了 + Sentinel 还没判定）→ 业务报 ConnectionError
- 5-30 秒：Sentinel 选举 + 切换；客户端持续 retry 仍失败
- 30 秒后：客户端拿到新 master 地址 → 业务恢复

让用户感知最小：
1. **客户端层 retry + 指数退避**（200ms, 400ms, 800ms... 上限 2 秒）
2. **应用层降级**：临时切到本地缓存 / 拒绝非核心请求 / 排队
3. **`min-replicas-to-write 1 + 多 replica`**：减少切换时间的"窗口"（保证候选 replica 数据是新的）
4. **客户端 socket_timeout 不要太长**：500-1000ms 即可，让重试快速触发
5. **告警**：超 30 秒未恢复立即报警人工介入

---

## 小结

| 概念 | 含义 | 关键参数 |
|---|---|---|
| 全量同步 | RDB + buffer 命令 | repl-diskless-sync |
| 部分重同步 | 用 backlog 补缺失 | repl-backlog-size |
| PSYNC2 | 主备切换 / master 重启仍能部分同步 | 双 replication-id |
| min-replicas-to-write | 没足够 replica 时拒写 | 防裂脑 |
| Sentinel quorum | 判 ODOWN 的票数 | 通常 = (N/2)+1 |
| Sentinel majority | 选 leader 的票数 | (N/2)+1，硬性 |

四条铁律：

1. **复制是异步的**——已 ACK 但未复制的写在 master 挂时丢
2. **Sentinel 跨 AZ + 奇数**——majority 永远成立
3. **backlog ≥ 数百 MB**——别让短抖动触发全量
4. **min-replicas-to-write 是裂脑防御**——必配

下一篇 **R05 — 精通 Redis Cluster**：16384 slot、gossip、MOVED/ASK 重定向、resharding 的工程实践。
