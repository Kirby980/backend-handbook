# 精通 Topic / Partition / Offset：Kafka 的 commit log 抽象

> 关联章节：[K02 KRaft 元数据](./02-精通-KRaft.md)、[K03 Producer](./03-精通-Producer.md)、[K04 Consumer + KIP-848](./04-精通-Consumer-KIP-848.md)、[K06 存储](./06-精通-存储与-Segment.md)

---

## 引言：把"消息队列"忘掉

Kafka 第一次出来时被归类为"消息队列"，跟 RabbitMQ、ActiveMQ 并列。这个标签害了一代人——很多人按 RabbitMQ 的"queue → consumer 消费即删"心智模型来用 Kafka，结果踩了所有"为什么 offset 重置"" 为什么消息没消失"的坑。

**Kafka 的核心抽象是 commit log，不是 queue**。一个 partition 就是一个**追加写入、永远保留（直到 retention 到期）、按顺序读出**的日志文件。生产者只能往尾部追加；消费者各自维护一个"读到哪里"的指针（offset），互不影响。

这一章把这个抽象拆开。读完之后你应该能解释：

- Topic / Partition / Offset 三者各自的物理含义
- 为什么 partition 是并行单元，topic 不是
- 为什么"删除某条消息"在 Kafka 里几乎不可能
- consumer group 是怎么把同一份消息**同时分发给多个独立业务**而不互相干扰
- key 路由的物理后果——为什么一旦定了 partition 数就很难加

后续章节都建立在这些基础概念上。

---

## 第一章：Commit Log —— 唯一的核心抽象

### 1.1 一个 partition 就是一个 commit log

```
Partition: orders-0
+----------------------------------------------------+
| msg_0  | msg_1 | msg_2 | msg_3 | msg_4 | msg_5 | ... |
+----------------------------------------------------+
  offset=0  offset=1 ...                  ↑ next write
```

属性：

1. **追加写入**：只能在尾部加新消息，**不能改不能删某一条**（除了 retention / compaction，详见 K06）
2. **顺序读**：消费者按 offset 升序读
3. **持久化**：写入磁盘 + 副本
4. **保留期**：默认 7 天后自动按时间或大小删除最老 segment

### 1.2 为什么不像传统 Queue？

| 维度 | 传统 Queue（RabbitMQ） | Kafka Commit Log |
|---|---|---|
| 消费完是否删 | 是（ACK 即删） | 否（offset 推进，消息仍在） |
| 重新消费 | 难（消息没了） | 简单（offset 重置即可） |
| 多消费者独立读 | 难（共享 queue 或 fanout 复制） | 原生支持（每个 group 独立 offset） |
| 写吞吐 | 万级 / 秒 | 百万级 / 秒（单 broker） |
| 消息删除时间复杂度 | O(1)（出队即删） | N/A（不删，靠 retention） |
| 严格顺序 | 一般 queue 内有序 | 一个 partition 内有序 |

**核心差异**：Kafka 把"读"和"写"完全分离——写是单线程顺序追加（极快），读是各自跑各自的 offset（互不干扰）。这种分离是 100x 性能差的根因。

### 1.3 一个 commit log 怎么落到磁盘

```
/var/lib/kafka/orders-0/
├── 00000000000000000000.log         ← segment 文件，offset 从 0 开始
├── 00000000000000000000.index       ← 稀疏索引：offset → file position
├── 00000000000000000000.timeindex   ← 时间索引：timestamp → offset
├── 00000000000000123456.log         ← 下一个 segment（>1GB 或超 segment.ms 后切）
├── 00000000000000123456.index
└── leader-epoch-checkpoint          ← leader 任期信息（K05 详解）
```

文件名 = 该 segment 第一条消息的 offset，对齐到 20 位。这种命名让"查 offset = X 在哪个 segment"变成简单的字典序 + 二分。

详见 K06。

---

## 第二章：Topic —— 业务层的逻辑分组

### 2.1 Topic 不是物理实体

Topic 是**一组 partition 的命名集合**——本身没有数据，只是元数据。

```bash
kafka-topics.sh --create --topic orders --partitions 6 --replication-factor 3
```

这条命令做的事：

1. 在元数据 log（KRaft）/ ZooKeeper（< 4.0）里登记 topic `orders` 有 6 个 partition
2. controller 决定每个 partition 的 3 副本（含 leader）分布在哪些 broker
3. 各 broker 创建 `orders-0` / `orders-1` / ... 的本地目录

**Topic 没有"中心存储"——数据全在 partition 里**。

### 2.2 命名约定

Kafka 没有内置命名空间，业界规范：

```
prefix.environment.bounded-context.entity[.event-type]

例:
prod.payment.order.created
stg.search.query
internal.kafka-streams-app.changelog.user-state
```

**避免**：

- 同时用 `.` 和 `_`（Kafka 内部 metric 名混淆）
- 名字含 `__`（双下划线开头是 Kafka 内部 topic：`__consumer_offsets`、`__transaction_state`）
- 长度 > 200（影响 metric 标签）

### 2.3 内部 topic

```bash
$ kafka-topics.sh --list --bootstrap-server :9092
__consumer_offsets          ← 50 分区，存所有 consumer group offset
__transaction_state         ← 50 分区，存事务状态
_schemas                    ← Schema Registry 用（如有）
my_topic
my_topic-changelog          ← Kafka Streams app 自动建的
```

**`__consumer_offsets` 是 Kafka 关键的"系统 topic"**——它本身也是一个 Kafka topic，存着所有 consumer group 的 offset。compaction 模式，只保留每个 (group, topic, partition) 的最新 offset。

---

## 第三章：Partition —— 真正的并行单元

### 3.1 Partition 数决定一切并行度

```
Topic orders, 6 partitions
+-----------+-----------+-----------+-----------+-----------+-----------+
| orders-0  | orders-1  | orders-2  | orders-3  | orders-4  | orders-5  |
+-----------+-----------+-----------+-----------+-----------+-----------+

每个 partition 是一个独立的 commit log，可分布在不同 broker 上：
B1:  [orders-0 leader] [orders-3 follower]
B2:  [orders-1 leader] [orders-4 follower]
B3:  [orders-2 leader] [orders-5 follower]
```

**为什么 partition 是并行单元**：

- 一个 consumer group 内：**一个 partition 同时只能被一个 consumer 消费**（保证 partition 内顺序）
- 想增加消费并行度 → 加 partition + 加 consumer
- **consumer 数 > partition 数 = 多余的 consumer idle**

**这是为什么"开多线程跑同一个 consumer" 没意义**——并行度卡死在 partition 数。

### 3.2 顺序保证仅在 partition 内

```
Producer A 发: A1, A2, A3
Producer B 发: B1, B2

如果 A1, A2, A3 都进 partition-0，B1, B2 进 partition-1：
  partition-0:  A1 → A2 → A3   (这三条之间有序)
  partition-1:  B1 → B2        (这两条之间有序)
  但 A1 / B1 谁先到 broker 是不确定的
```

要"全局有序"？只能用**单 partition topic**——但这等于把并行度限制为 1，吞吐天花板就是单 broker 单线程。

要"按 key 局部有序"（如同一用户的事件按顺序）→ producer 用 `key=user_id`，Kafka 默认 partitioner 把同 key 路由到同 partition。

### 3.3 partition 数怎么选

**经验公式**：

```
partition 数 ≈ max(目标吞吐 / 单 partition 吞吐, 目标 consumer 并行度)

单 partition 吞吐：约 10-50 MB/s 写入、20-100 MB/s 读（看消息大小、压缩、副本数）
单集群 partition 总数上限：Kafka 4.0 KRaft 单集群支持百万级
```

**反模式**：

- **过少**：吞吐 / 消费并行度上不去
- **过多**：每个 partition 是一组文件 + 一份元数据，broker 重启加载慢，控制面压力大；空闲 partition 也占内存（page cache + 索引）

**实务**：日常 topic 6-12 partition；高吞吐核心业务 24-100；超高频或要细粒度并行 100-1000。

### 3.4 加 partition 的代价

```bash
kafka-topics.sh --alter --topic orders --partitions 12   # 6 → 12
```

**可以加，但不可减**。加 partition 后果：

1. **新 partition 立刻可用**——key 路由的散列空间变了
2. **同 key 的旧消息留在老 partition；新消息可能进新 partition**——破坏 per-key 顺序
3. **Kafka Streams 中表的语义被破坏**——changelog 重建时会乱掉

**生产实务**：宁可一开始多设几个 partition，也不要后期临时扩。如果非加不可，业务侧要做好"短期内 per-key 顺序可能错"的预案；Streams 应用要 reset state stores。

---

## 第四章：Offset —— 消息的身份证

### 4.1 Offset 是 partition 内的单调递增整数

```
Partition orders-0:
+--------+--------+--------+--------+
| msg_0  | msg_1  | msg_2  | msg_3  |  ← producer 视角的写入顺序
+--------+--------+--------+--------+
  offset=0  1       2       3

下一个写入的位置 = 4 (LEO, Log End Offset)
```

offset 是 broker 端在消息追加时**分配的**，producer 无法指定。

64 位整数，理论上 9.2e18 上限——实际上 partition 数据保留几个月就远低于这个。

### 4.2 高水位（HW）：消费者能读到哪里

```
Partition: leader B1, followers B2 B3
LEO (Log End Offset, leader 的最新写入位置) = 100
B1 已写入 0..99
B2 已复制 0..98
B3 已复制 0..97

HW (High Watermark) = min(LEO of all ISR) = 97
↓
消费者只能读 offset <= 97 的消息
```

**HW 之后的消息对消费者不可见**——保证消费者读到的消息都已经被多数副本持久化，避免 leader 崩溃后丢"假读到"的消息。

详细见 K05 ISR 章节。

### 4.3 offset 几个特殊位置

| 名字 | 含义 |
|---|---|
| **earliest** | partition 当前最老消息的 offset（受 retention 影响，老的被删了，earliest 推进） |
| **latest** | LEO，下一条要写入的位置 |
| **HW** | High Watermark，消费者能读到的最新可见 offset |
| **LSO** | Last Stable Offset，事务模式下 read_committed consumer 能读到的位置（小于 HW） |
| **Committed offset** | consumer group 已 commit 的位置（存在 `__consumer_offsets`） |

### 4.4 offset reset 策略

consumer 第一次启动（无 committed offset）或 offset 已被 retention 删除时：

```
auto.offset.reset = earliest   ← 从最老开始读，可能很慢但不丢
auto.offset.reset = latest     ← 只读新来的（默认，但要小心：第一次启动会丢之前所有）
auto.offset.reset = none       ← 报错，让应用决定
```

**生产推荐 `earliest` + 应用层处理 dedupe**——比 `latest` 安全。

### 4.5 offset 的提交

```java
consumer.subscribe(List.of("orders"));
while (running) {
    var records = consumer.poll(Duration.ofMillis(100));
    process(records);
    consumer.commitSync();        // 同步提交（慢但确定）
    // 或 consumer.commitAsync(); // 异步提交（快但可能失败）
    // 或 enable.auto.commit=true 自动提交（最简但有重复消费风险）
}
```

**at-least-once 与 at-most-once 的区分**：

- `process → commit`：失败重试时可能重复消费 → at-least-once（业务必须幂等）
- `commit → process`：commit 后 process 失败 → 消息丢失 → at-most-once（很少用）

**exactly-once** 需要 Kafka transactional API + 端到端配合 → K07。

### 4.6 commit 到哪里去了

```
特殊的内部 topic: __consumer_offsets
  - 50 分区（默认）
  - replication-factor = 3
  - compact 模式（只保留每个 key 的最新值）
  - key = "group.id|topic|partition" → value = offset + metadata
```

任何 consumer commit 都是往这个 topic 写一条 record。集群重启后再启 consumer，从该 topic 读出 last commit 即可。

---

## 第五章：Consumer Group —— 消息的"广播-分组"机制

### 5.1 同一 topic 服务多个独立业务

```
Topic: user.activity (6 partitions)

Consumer Group A (analytics): 3 consumers
  consumer-a1: orders-0, orders-1
  consumer-a2: orders-2, orders-3
  consumer-a3: orders-4, orders-5

Consumer Group B (notification): 2 consumers
  consumer-b1: orders-0..2
  consumer-b2: orders-3..5

Consumer Group C (audit): 1 consumer
  consumer-c1: orders-0..5
```

**每个 group 独立维护 offset，互相完全不干扰**。这是 Kafka 替代传统"消息总线"的关键能力——同一份数据可以同时给 N 个下游消费，每个下游按自己的节奏。

### 5.2 group 内的 partition 分配

默认策略 **RangeAssignor**（按 partition 范围）或 **StickyAssignor**（粘性，rebalance 时尽量不变）。

KIP-848 协议（Kafka 4.0 默认）改由 **broker 端**统一计算分配——客户端只接收结果。这消除了"客户端版本不一致导致分配冲突"的问题。详 K04。

### 5.3 rebalance 触发时机

- 新 consumer 加入 group
- 某 consumer 退出 / 崩溃 / heartbeat 超时
- topic 加了 partition
- group 订阅的 topic 列表变化

**老协议**：所有 consumer 短暂"停摆"重新协商。**KIP-848 新协议**：增量重分配，不停摆。

### 5.4 何时不用 consumer group

如果你想要**广播**（每条消息每个实例都处理），不要用同一个 group——给每个实例一个独立 group.id。

```
instance-1: group.id = "notif-svc-1"
instance-2: group.id = "notif-svc-2"
```

每个 group 独立读完整数据。这种用法常见于本地缓存预热 / 多副本配置广播。

---

## 第六章：Key 与路由

### 6.1 默认 partitioner

```java
ProducerRecord<String, String> rec = new ProducerRecord<>("orders", "user_42", payload);
producer.send(rec);
```

```
partition = murmur2(key) % partition_count    // 默认行为
```

效果：**同 key 总是进同 partition**——保证 per-key 顺序。

如果 key 为 null：

- Kafka < 2.4：round-robin
- Kafka 2.4+：**sticky partitioner**——一段时间内固定到一个 partition（让 batch 更紧凑），定期切换。是 producer 吞吐显著提升的关键优化。

### 6.2 加 partition 后 key 路由改变

```
原: 6 partitions, key="user_42" → murmur2(.) % 6 = 3
加成 8 partitions, key="user_42" → murmur2(.) % 8 = 5
```

**老消息在 partition 3，新消息在 partition 5** —— per-key 顺序破裂！

这是为什么"加 partition" 在生产是高风险操作。彻底解法：**一开始就用一致性 hash 或预 sharded partition** —— 但 Kafka 默认 partitioner 不支持。需要自定义 `Partitioner` 接口。

### 6.3 自定义 partitioner

```java
public class GeoPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        // 按业务规则：北区 → 前一半 partition，南区 → 后一半
        return decideRegion((User) value) == "north"
               ? Math.abs(key.hashCode()) % (cluster.partitionCountForTopic(topic) / 2)
               : cluster.partitionCountForTopic(topic) / 2
                 + Math.abs(key.hashCode()) % (cluster.partitionCountForTopic(topic) / 2);
    }
}
```

配 `producer.partitioner.class=com.example.GeoPartitioner`。

---

## 第七章：观察与调试

### 7.1 topic 概况

```bash
$ kafka-topics.sh --bootstrap-server :9092 --describe --topic orders
Topic: orders   PartitionCount: 6  ReplicationFactor: 3
  Topic: orders   Partition: 0   Leader: 1   Replicas: 1,2,3   Isr: 1,2,3
  Topic: orders   Partition: 1   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
  ...
```

- **Leader**：当前 partition 主副本
- **Replicas**：分配的所有副本（包含 leader）
- **ISR (In-Sync Replicas)**：当前同步的副本子集——只有 ISR 内的副本能被选 leader

**Isr 列表少于 Replicas 是健康警告**——某副本 lag 过大被踢出 ISR。

### 7.2 group lag

```bash
$ kafka-consumer-groups.sh --bootstrap-server :9092 --describe --group analytics
GROUP    TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID
analytics orders   0          150000          150100          100   consumer-1-...
analytics orders   1          150100          150100          0     consumer-1-...
...
```

`LAG = LOG-END-OFFSET - CURRENT-OFFSET` = 还差多少未消费。**lag 持续增长 = 消费速度跟不上生产**。

### 7.3 dump 一个 segment 看消息

```bash
$ kafka-dump-log.sh --files /var/lib/kafka/orders-0/00000000000000000000.log \
    --print-data-log

Starting offset: 0
baseOffset: 0 lastOffset: 99 count: 100 baseSequence: -1 lastSequence: -1
producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 0 isTransactional: false
isControl: false position: 0 CreateTime: 1715587200000 size: 12345
magic: 2 compresscodec: NONE crc: 1234567890 isvalid: true

| offset: 0 CreateTime: 1715587200000 keySize: 7 valueSize: 100
  key: user_42 payload: {"order_id":...}
| offset: 1 ...
```

**生产故障排查神器**——能直接看到磁盘上具体每条消息的内容。

---

## 第八章：生产级最佳实践

1. **partition 一开始就多设**——12 / 24 / 48 比"以后扩"省心得多。
2. **使用 key 路由保 per-key 顺序**——不要靠 partition 顺序写入。
3. **__consumer_offsets 不要乱删**——任何 delete 都会让 group 失去所有 offset。
4. **lag 加监控告警**——Burrow / Kafka Exporter + Prometheus。
5. **生产 topic 命名标准化**——`environment.context.entity.[event]` 一目了然。
6. **永远不删 partition / 不减 partition**——Kafka 不支持，会出大故障。
7. **replication.factor >= 3 + min.insync.replicas = 2**——核心 topic 起步。
8. **生产开 idempotent producer**——`enable.idempotence=true` 几乎零代价就避免了重复（详 K03）。
9. **commit 频率适中**——每条 commit 太频繁会打爆 `__consumer_offsets`；几秒一次或几百条一次最合理。
10. **不要在同一应用里用同一 group 跑多种业务**——一个 group 对应一种业务用途；多用途用多个 group。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：以为 topic 是数据存储位置
topic 是元数据集合，**真正的数据在每个 partition**。"备份 topic"就是备份所有 partition 的 segment 文件。

### ❌ 陷阱 2：consumer 数 > partition 数
多余的 consumer **永远 idle**。partition=6 时跑 10 个 consumer = 4 个白吃 CPU。

### ❌ 陷阱 3：随便加 partition
打乱 per-key 顺序。详见 §6.2。

### ❌ 陷阱 4：用 partition 实现优先级队列
Kafka 不支持优先级。模拟方案是开多个 topic（high-priority / low-priority），消费侧自己控制读取顺序——但这跟"真正的优先级"语义有差距。

### ❌ 陷阱 5：把 offset commit 当成"已经处理完"
默认 `enable.auto.commit=true` 是 **poll 之后 5 秒自动 commit**——即使你的业务逻辑还没跑完。生产应用建议关掉，处理完业务再 `commitSync`。

### ❌ 陷阱 6：以为 retention 是按消息数
默认 `retention.ms=168h`（7 天）和 `retention.bytes=-1`（不限制）。**只按时间删除老 segment**，不会按消息数删。

### ❌ 陷阱 7：把 `__consumer_offsets` 当普通 topic 配 retention
默认是 compact 模式（不删，只压缩同 key 旧版本）。如果误改成 delete，会大批丢失 offset → 全部 group 失去消费位点。

### ❌ 陷阱 8：误以为 partition 内严格 FIFO 就是 producer 发送顺序
producer 异步 send 时，如果某条消息**重试**，可能比后发的更晚到 broker——破坏 producer 视角的顺序。修法：`max.in.flight.requests.per.connection=1`（牺牲并发）或 `enable.idempotence=true`（保留并发且去重）。

### ❌ 陷阱 9：单个 producer 实例发往太多 topic / partition
producer 会为每个目标 broker 维护连接 + 缓冲区。10 个 topic × 100 partition × 50 broker = 大量内存。**生产建议每应用几个 producer 实例分担**。

### ❌ 陷阱 10：把 Kafka 当数据库长期查
Kafka 不是 KV 存储——按 key 找一条消息要扫整个 partition（O(N)）。要查询 → 索引到 ES / 数据库 / KV 里。

---

## 第十章：练习题

**练习 1**：解释 partition 数对吞吐、可靠性、运维复杂度的影响。给 6 / 24 / 240 partition 三种规模的适用场景。

**练习 2**：你的 topic 有 12 partition，consumer group 有 8 个 consumer。算出每个 consumer 平均分到几个 partition？某个 consumer 挂了后是什么状况？

**练习 3**：业务要把订单按 `user_id` 分组顺序处理。设计 topic + partition + key 的方案。如果一开始定 24 partition，半年后业务量翻倍想加到 48，你的应对策略？

**练习 4**：写一个 Java consumer 实现"处理完业务再提交 offset"的可靠消费模式（含失败处理）。

**练习 5**：解释为什么 Kafka 单 partition 顺序写入能达到 100MB/s+ 吞吐，而传统 MQ 单 queue 一般只有几 MB/s。

---

## 参考答案

**练习 1**：
- **partition 数 ↑ 吞吐 ↑**：更多并行写入和消费
- **partition 数 ↑ 可靠性 ↓ (轻微)**：每个 partition 是独立故障域，更多 partition = 更多 leader 切换可能
- **partition 数 ↑ 运维复杂度 ↑**：broker 重启加载慢，控制面状态大
- 适用：6 = 中小应用日均 GB；24 = 主流业务日均 TB；240 = 超大吞吐 + 细粒度消费分组（如点击流）

**练习 2**：12 / 8 = 1.5，所以分配是 4 个 consumer 2 个 partition，4 个 consumer 1 个 partition（具体看 Assignor，但总和 12）。某 consumer 挂了 → rebalance → 12 partition 重新分到剩下 7 个 consumer，每个 ~1.7 个。处理期间老协议会"停摆"，KIP-848 是增量过渡。

**练习 3**：
- topic `orders` 24 partition + producer 用 `key=user_id`（默认 partitioner，自动 murmur2 % 24 路由）
- 单一 user_id 永远进同一 partition，保 per-key 顺序
- 想加 partition：**A 方案**（推荐）：建新 topic `orders-v2` 48 partition，做 producer/consumer 双写双读切换；**B 方案**：硬加 partition + 业务侧接受"过渡期 per-key 乱序" + 同步处理 Streams 状态重建。**绝大多数生产选 A**。

**练习 4**：

```java
props.put("enable.auto.commit", "false");          // 关自动 commit
props.put("isolation.level", "read_committed");    // 事务隔离

try (var consumer = new KafkaConsumer<String, String>(props)) {
    consumer.subscribe(List.of("orders"));
    while (running) {
        var records = consumer.poll(Duration.ofMillis(500));
        for (var rec : records) {
            try {
                process(rec);   // 业务处理：幂等！
            } catch (Exception e) {
                log.error("process failed for offset={}, will retry", rec.offset(), e);
                // 不 commit；下次 poll 会再拿到（at-least-once）
                throw e;
            }
        }
        try {
            consumer.commitSync();   // 整批处理成功后 commit
        } catch (CommitFailedException e) {
            // rebalance 期间 commit 失败，下次 poll 再尝试
            log.warn("commit failed, will retry on next poll");
        }
    }
}
```

**练习 5**：
- 单 partition 是**顺序磁盘写**，机械盘 ~100 MB/s、SSD ~500 MB/s，跟随机写差 100x
- Kafka 直接用 **page cache** + sendfile 实现零拷贝（消费者读时数据从 page cache → socket，不进 user space）
- 单 partition 内无锁、无事务、无索引维护（写时不更新索引；index 是稀疏离线生成）
- 传统 MQ 要维护 ACK 状态、可见性时间窗、消息删除——每条消息都有元数据 + 索引更新

---

## 小结

| 概念 | 物理含义 | 关键约束 |
|---|---|---|
| Topic | partition 集合的命名 | 元数据存在 KRaft / ZK |
| Partition | 一个 commit log | 顺序追加，并行单元 |
| Offset | partition 内单调整数 | broker 分配，不能跳号 |
| Replica | partition 的副本 | 一个 leader + N follower |
| ISR | 当前同步的副本子集 | 选 leader 的候选 |
| HW | 消费者可见上限 | min(LEO of ISR) |
| Consumer Group | 协作消费的 consumer 集合 | partition 在 group 内独占分配 |
| Key | 路由依据 | murmur2(key) % partitions |

四条铁律：

1. **partition 是并行单元**——一切吞吐、消费并行度都靠它
2. **顺序只在 partition 内有效**——要全局顺序就只能单 partition，吞吐天花板降到单线程
3. **offset 是消费者的事**——broker 不知道谁消费到哪，每个 group 独立
4. **Kafka 是 commit log 不是 queue**——读不删，重读简单，多消费者天然支持

下一篇 **K02 — 精通 KRaft 元数据管理** 将拆开 Kafka 4.0 默认的元数据架构：controller quorum、metadata log、snapshot、从 ZK 模式的迁移路径。
