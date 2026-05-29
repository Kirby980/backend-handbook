# Kafka 深度课程 · 自测题库

> 配合 [INDEX.md](./INDEX.md) 使用。每章 ~10 题，含答案与展开解析。
> 难度标记：⭐ 概念 ⭐⭐ 进阶 ⭐⭐⭐ 源码级 / 容易踩坑

> 答题建议：先盖住答案自答，再展开核对。能讲出"为什么"比答对结论更重要。

---

## K01 Topic / Partition / Offset 模型

### Q1.1 ⭐ commit log 与传统 MQ 模型的核心区别？

<details><summary>答案</summary>

传统 MQ：消息消费后删除（destructive read）。

Kafka commit log：消息**持久保留** + consumer 维护自己的 offset（cursor）。多个 consumer 互不影响，可以"回放"过去的消息。

核心差异：

- 消费 ≠ 删除
- 多订阅者独立 offset
- 可重放 / 重处理
- broker 不维护"已读"状态

</details>

### Q1.2 ⭐⭐ partition 数选错会发生什么？

<details><summary>答案</summary>

- 太少 → 并行度低，单 partition 写入热点，consumer 不能扩
- 太多 → 元数据膨胀、broker 文件句柄爆、复制开销增加、controller 切换慢

经验：

- 单 partition 大小 < 50GB
- 单 broker < 4000 partition
- topic 起步 6-12 partition

partition **只能加不能减**，起步保守。

</details>

### Q1.3 ⭐⭐⭐ partition 数与有序性的关系？

<details><summary>答案</summary>

**单 partition 内**消息有序（按写入顺序）。**跨 partition 无序**。

有序保证：

- 同 key 总进同 partition → 按 key 有序
- 全局有序 = 1 个 partition（牺牲并行）

加 partition 破坏 key 的路由（`hash(key) % new_count`），同 key 可能换 partition → 顺序破。

</details>

### Q1.4 ⭐ consumer group 的两个核心规则？

<details><summary>答案</summary>

1. **组内一个 partition 只被一个 consumer 消费**（保证不重复）
2. **不同组互相独立**（pub-sub 语义，offset 独立）

组内 consumer 数 > partition 数 → 多余 consumer 空闲。

</details>

### Q1.5 ⭐⭐ offset 与 timestamp 的关系？

<details><summary>答案</summary>

- offset：单 partition 内单调递增的整数 ID
- timestamp：消息创建时间或写入 broker 时间

两者独立：

- 消费按 offset 推进
- 用 `offsetsForTimes(timestamp)` 反查给定时间的 offset（走 .timeindex）

注意：跨 partition 同一时间的 offset 不同。

</details>

### Q1.6 ⭐⭐ key 为 null 时怎么路由？

<details><summary>答案</summary>

- Kafka < 2.4：RoundRobin 轮询
- Kafka 2.4-3.2：sticky 分区（KIP-480，类 UniformStickyPartitioner/DefaultPartitioner，粘到一个直到 batch 满）
- Kafka 3.3+：KIP-794 内置默认分区器（按 batch.size 字节切换 + 自适应负载，慢 broker 少给数据）；UniformStickyPartitioner 类自 3.3 起 deprecated

为什么不用 RoundRobin？— 让 batch 失效（每 partition 都只有 1 条记录）。Sticky 让连续记录粘同 partition 实现真正 batching。

</details>

### Q1.7 ⭐ committed offset 与 log end offset

<details><summary>答案</summary>

- **committed offset**：consumer group 已确认处理到的位置
- **log end offset (LEO)**：partition 当前最新位置
- **HW**：所有 ISR 副本的 LEO 最小值，是 consumer 可读位置

`lag = log_end_offset - committed_offset` 衡量消费滞后。

</details>

### Q1.8 ⭐⭐ partition 与并行度的关系？

<details><summary>答案</summary>

partition 是 Kafka 的**唯一并行单元**：

- producer 可以并发写多个 partition
- consumer 一个 partition 一个 thread
- replication 一个 partition 一个 follower fetcher

consumer 并发上限 = partition 数。

</details>

### Q1.9 ⭐⭐⭐ partition leader 与 leader epoch

<details><summary>答案</summary>

每个 partition 有一个 **leader** 副本（处理读写）。leader 切换时 **epoch 递增**（单调）。

epoch 用于：

- 区分新老 leader 写的数据
- follower 截断不一致的日志（log truncation）
- 防止"已下线 leader 复苏后继续写"造成分歧

</details>

### Q1.10 ⭐⭐ consumer poll 模型 vs push 模型

<details><summary>答案</summary>

Kafka 是 pull 模型。好处：

- consumer 控制速率（背压自然）
- broker 不用维护 consumer 状态
- 容易实现批量消费

代价：

- 空 poll 浪费 RPC（fetch.min.bytes + fetch.max.wait.ms 缓解）
- 实时性靠 poll 频率

</details>

---

## K02 KRaft

### Q2.1 ⭐ KRaft 比 ZK 启动快的 3 个原因？

<details><summary>答案</summary>

1. **增量 fetch metadata log**：broker 不全量 reload，只 fetch 上次断点之后
2. **没 ZK watch 风暴**：通过 metadata log 顺序推送，不是回调
3. **controller 切换不全量 reload**：新 controller 也只从 metadata log 续

实测大集群冷启动从 10+ min 降到 1 min 内。

</details>

### Q2.2 ⭐⭐ 3 controller 集群挂 2 个能做什么？

<details><summary>答案</summary>

- 数据面正常：producer / consumer 普通读写不受影响（broker 用本地缓存的元数据）
- 控制面冻结：
  - 不能创建 / 删除 topic
  - 不能改 ISR / leader
  - ACL 不能改
  - rebalance 受影响（依赖元数据变更）

**控制面与数据面隔离**是 KRaft 设计的核心好处。

</details>

### Q2.3 ⭐ process.roles=broker,controller 适用什么？

<details><summary>答案</summary>

适用：小集群（< 10 broker）或开发 / 测试。

不适用：

- 大集群（broker 角色繁忙影响 controller）
- 跨机房部署（controller 需要稳定低延迟，broker 可能跨域）

生产推荐分离部署：3 专门 controller + N broker。

</details>

### Q2.4 ⭐⭐ metadata log 与普通 topic 的相同 / 不同？

<details><summary>答案</summary>

**相同**：

- 同样的 segment / index / log 文件格式
- 同样的 fetch 协议
- 同样的 snapshot 机制

**不同**：

- 单 partition（保证全序）
- 副本 = controller quorum 数
- 记录类型不是用户消息（PartitionChangeRecord、ConfigRecord 等）
- 客户端不能直接写

</details>

### Q2.5 ⭐⭐⭐ fencing 触发条件？

<details><summary>答案</summary>

active controller 检查 broker 心跳：

- 心跳过期（超过 `broker.session.timeout.ms`）→ 标记 fence
- broker metadata offset 落后 HW 过多 → fence

Fenced broker：

- 暂停作为任何 partition 的 leader（其他副本接管）
- 仍可服务读（如果还是 ISR 副本）
- 心跳恢复 + offset 追上 → 解除 fencing

</details>

### Q2.6 ⭐⭐ 1000 broker 集群 controller quorum 怎么设？

<details><summary>答案</summary>

3 或 5 个 controller，**与 broker 数无关**。

- 3 controller 容忍 1 挂
- 5 controller 容忍 2 挂

分布在 3 个机房 / AZ（每机房至少 1 个 controller）：

- 单机房挂仍能选主
- 跨机房 RTT 低（< 10ms）以避免选主抖动

controller 节点不需要数据节点的大磁盘 / 网卡，4C/8GB 中等配置即可。

</details>

### Q2.7 ⭐⭐⭐ ZK → KRaft 迁移可回滚阶段？

<details><summary>答案</summary>

**可回滚**：bridge 模式期间（双写 ZK + KRaft，broker 还有部分跑 ZK 模式）。

**不可回滚**：

- 所有 broker 全切 KRaft 后
- 关闭 ZK 集群后

实操：

- bridge 期间留 1 周观察
- 全切前业务 / 监控完全验证
- 关 ZK 前留快照备份

</details>

### Q2.8 ⭐⭐ current-leader / high-watermark / log-end-offset

<details><summary>答案</summary>

- `current-leader`：raft 当前已知的 leader id（-1 表示无 leader / 选举中）
- `high-watermark`：已 commit 的 offset（quorum 同意的）
- `log-end-offset`：本地 log 末尾（可能 > HW，未提交的）

监控点：

- `current-leader` 频繁变 → 选主抖动
- `log-end-offset - HW` 大 → quorum 落后

</details>

### Q2.9 ⭐⭐ snapshot 过密 / 过稀的问题？

<details><summary>答案</summary>

过密：snapshot 文件频繁生成 → 磁盘 IO 浪费 + 元数据 churn。

过稀：snapshot 大、不频繁 → broker 重启 / 加入 quorum 时要 replay 很长 log。

经验 `metadata.log.max.snapshot.interval.ms=3600000`（1h）是平衡点。元数据 churn 大（大量 topic 变更）的集群可调小。

</details>

### Q2.10 ⭐⭐⭐ 跨 region 部署 controller quorum

<details><summary>答案</summary>

**不推荐**单集群跨 region。原因：

- Raft 选举依赖低延迟心跳（RTT < 10ms）
- 跨 region RTT 50-100ms 让选举不稳定
- 元数据写入延迟高

正确做法：

- 每 region 一个独立集群
- 跨 region 用 MirrorMaker / CCR 异步复制
- 业务侧处理跨 region 切换

</details>

---

## K03 Producer

### Q3.1 ⭐ acks=0/1/all 容忍故障？

<details><summary>答案</summary>

| acks | 容忍 | 丢失场景 |
|---|---|---|
| 0 | 啥都不等 | 网络丢包 / broker crash |
| 1 | leader 写入即返 | leader ack 后立刻挂，follower 没复制到 |
| all | ISR 全 ack | ISR < min.insync 时阻塞写入 |

**配 acks=all + min.insync.replicas=2 + RF=3** 才是真正持久。

</details>

### Q3.2 ⭐⭐ linger.ms=0 vs linger.ms=10 影响

<details><summary>答案</summary>

linger.ms=0：

- 延迟低（不等 batch）
- 吞吐低（小 batch、单条 RPC 多）
- 压缩效果差

linger.ms=10：

- 延迟略增（最多 10ms 攒批）
- 吞吐显著提高（big batch）
- 压缩比好

经验：业务消息 5-20 ms，日志 50ms+。

</details>

### Q3.3 ⭐⭐ sticky 分区演进：KIP-480 vs KIP-794

<details><summary>答案</summary>

KIP-480（Kafka 2.4，类 UniformStickyPartitioner/DefaultPartitioner）：随机粘一个 partition 到 batch 满 / linger 到，再换。

KIP-794（Kafka 3.3+，内置默认分区器逻辑，移除 partitioner.class + 设 partitioner.ignore.keys=true 启用；UniformStickyPartitioner 类自 3.3 起 deprecated）：

- 同样粘性，但按 batch.size 字节切换
- 选 partition 时**优先选 broker 队列短的**（自适应负载，慢 broker 少给数据）
- 避免老版本下出现"运气差粘到慢 broker"

效果：连续记录仍达成 batching，同时避免热点 broker。

</details>

### Q3.4 ⭐⭐⭐ idempotent producer 保证什么不保证什么

<details><summary>答案</summary>

**保证**：

- 单 partition 内**不重复**（broker 用 PID + sequence 去重）
- 单 partition 内**按发送顺序**（即使 max.in.flight=5）

**不保证**：

- 跨 partition / 跨 topic 原子（要 transactional）
- producer 重启后旧消息不重复（新 PID）
- 应用层副作用幂等（外部系统调用）

</details>

### Q3.5 ⭐⭐ max.in.flight 对顺序保证的影响

<details><summary>答案</summary>

非 idempotent + max.in.flight > 1：

- req1 失败重试时 req2 已经写入 → 顺序乱

idempotent + max.in.flight ≤ 5：

- 重试时按 sequence 重新排，broker 拒绝乱序 → 应用层重发
- 顺序保证

非 idempotent 要严格顺序 → max.in.flight=1（性能差）。

</details>

### Q3.6 ⭐⭐⭐ idempotent 强制 max.in.flight ≤ 5 的原因

<details><summary>答案</summary>

broker 端为每个 PID-partition 保存 **最近 5 个 sequence number** 的窗口。

- > 5：broker 无法判断收到的 sequence 是重复还是乱序
- ≤ 5：broker 总能正确处理重试 / 乱序

调大窗口理论可行，但 5 已经能跑满网络。

</details>

### Q3.7 ⭐⭐ 压缩为什么靠 batch 才有效？

<details><summary>答案</summary>

压缩本质是**找重复模式**。

单条消息内部重复模式少 → 压缩比近 1。

多条同类消息（JSON 字段名重复）合一个 batch → 压缩 3-10×。

所以开压缩 + 增大 batch / linger 是组合拳。

</details>

### Q3.8 ⭐⭐ RecordTooLargeException 怎么排查

<details><summary>答案</summary>

错误：单条消息超过 broker 的 `message.max.bytes`（默认 1MB）。

排查：

1. 检查消息实际大小
2. 检查 producer `max.request.size`（默认 1MB）
3. 检查 broker / topic 的 `max.message.bytes`

修复：

- 业务侧拆消息（大文件外部存）
- 或调大三处配置一致

</details>

### Q3.9 ⭐⭐⭐ transactional.id 必须唯一的原因 + 多实例怎么配

<details><summary>答案</summary>

transactional.id 跨重启**保持同一 producer 身份**（zombie fencing 关键）。

多实例错用同 ID → 互相 fence → 全部不可用。

正确：

```
transactional.id = "tx-" + stable_instance_id
```

K8s 用 StatefulSet（hostname 稳定）。

Kafka Streams 自动管理 transactional.id（per task）。

</details>

### Q3.10 ⭐ send 异步 vs 同步

<details><summary>答案</summary>

| 方式 | 吞吐 | 延迟 | 用途 |
|---|---|---|---|
| 异步 + callback | 数十万 QPS | 低 | 主流 |
| 同步 (.get()) | 几千 QPS | 等 RTT | 调试 / 个别强一致 |

99% 场景用异步。同步只在"必须等结果才能继续"时用。

</details>

---

## K04 Consumer / KIP-848

### Q4.1 ⭐ session.timeout vs max.poll.interval

<details><summary>答案</summary>

- `session.timeout.ms`：多久没收到心跳算成员死了。**后台心跳线程**发，应用卡住不影响。
- `max.poll.interval.ms`：两次 poll() 之间最大间隔。应用处理慢就会触发。

典型踩坑：处理一批要 10min 但 max.poll.interval=5min → 被踢出。

修复：

- 缩 max.poll.records（每批少）
- 调大 max.poll.interval
- 业务异步化

</details>

### Q4.2 ⭐⭐ 默认自动 commit 为什么不是真正 at-least-once

<details><summary>答案</summary>

`enable.auto.commit=true` + `auto.commit.interval.ms=5000`：

- 每次 poll 检查"距上次 commit > 5s"则 commit 当前最新 offset
- 不等"处理完"
- 可能 commit 了还没处理完的 offset → crash 后丢

实际是 **at-most-once**。要真 at-least-once：手动 commit 处理完再 commit。

</details>

### Q4.3 ⭐⭐ CooperativeStickyAssignor vs StickyAssignor

<details><summary>答案</summary>

- StickyAssignor：尽量保持上次分配不变（最小迁移），但 rebalance 仍是 stop-the-world
- CooperativeStickyAssignor：sticky + **增量 rebalance**，分两步 revoke / assign，不停整组

CooperativeSticky 是 KIP-848 前最佳。

</details>

### Q4.4 ⭐⭐⭐ KIP-848 消除 stop-the-world 关键技术

<details><summary>答案</summary>

1. **broker 主导**：分配方案在 broker 算，不依赖 consumer leader
2. **心跳异步上报 / 下发**：consumer 心跳带"现状"，broker 心跳响应带"目标"
3. **增量 reconcile**：consumer 主动 revoke 即将失去的 partition + 主动 fetch 新分配的，整组无需同步停止
4. **统一协议**：JoinGroup/SyncGroup/Heartbeat 三种合并成 ConsumerGroupHeartbeat 一种

</details>

### Q4.5 ⭐⭐ offset out of range 处理

<details><summary>答案</summary>

原因：consumer 上次 commit 的 offset 早已被 retention 删除。

修复：

```bash
kafka-consumer-groups.sh --reset-offsets --to-earliest --topic foo --group g --execute
```

预防：

- 监控 consumer 离线时长
- retention 足够长（覆盖最长 downtime）
- 配 `auto.offset.reset=earliest` 自动重置

</details>

### Q4.6 ⭐⭐ 大消息 fetch 现象

<details><summary>答案</summary>

某条消息 > `max.partition.fetch.bytes`（默认 1MB）：

- broker 不会拆消息
- fetch 永远返回 0 字节
- consumer poll 阻塞 / lag 涨

修复：

- 调大 `max.partition.fetch.bytes`（如 10MB）
- 业务侧拆消息
- 大文件外部存（S3 + Kafka 发指针）

</details>

### Q4.7 ⭐⭐ KafkaConsumer 多线程使用

<details><summary>答案</summary>

`KafkaConsumer` **不是线程安全**。一个实例只能被一个线程 poll。

**唯一安全的跨线程调用**：`wakeup()`，用于优雅关闭。

多线程消费方案：

- 多 consumer 实例（每实例一线程）
- 单 consumer + 工作线程池（要注意 commit 时机保证 at-least-once）

</details>

### Q4.8 ⭐⭐ auto.offset.reset 何时生效

<details><summary>答案</summary>

**只对没 committed offset 的新 group 生效**（或 committed offset 不在 retention 范围）。

已 committed 的 group 永远从 committed 处续，不会用 reset 策略。

要"忘记之前的 offset 从头读"：手动 `--reset-offsets --to-earliest`。

</details>

### Q4.9 ⭐⭐⭐ Consumer 处理慢但不能丢，怎么架构

<details><summary>答案</summary>

几种方案，递进：

1. **poll 后异步处理**：把记录丢线程池，等所有处理完再 commitSync
2. **per-partition 处理**：每 partition 独立线程，各自 commit per-partition offset
3. **拆 topic + 业务分流**：慢逻辑独立 topic，快通道不阻塞
4. **流处理框架**：Streams / Flink 自动管理

注意：纯多线程 + 自动 commit 是 at-most-once 陷阱。

</details>

### Q4.10 ⭐⭐ read_committed 代价

<details><summary>答案</summary>

read_committed consumer 只能读到 **LSO**（Last Stable Offset），即第一个未完成事务之前。

代价：

- 长事务卡住所有下游 consumer（即使消息不在该事务内）
- LSO 与 HW 之间的数据延迟

缓解：

- transaction.timeout.ms 调小（让卡住事务超时 abort）
- 业务侧短事务

</details>

---

## K05 复制 ISR

### Q5.1 ⭐ LEO 与 HW 区别

<details><summary>答案</summary>

- **LEO**（Log End Offset）：单副本本地日志末尾，下一条要写的 offset
- **HW**（High Watermark）：所有 ISR 副本的 LEO 最小值，是 commit / consumer 可见边界

HW ≤ leader LEO，且 HW 在 follower 上滞后于 leader 一次 fetch。

</details>

### Q5.2 ⭐⭐ HW 在 follower 滞后的原因

<details><summary>答案</summary>

HW 在 follower 上**通过下次 fetch response 才更新**：

```
T0: leader LEO=5, follower LEO=4, HW=4
T1: follower fetch → leader 给 m4，response 里 HW=4
    follower append → LEO=5
T2: follower 下次 fetch → leader 知道 follower LEO=5
    更新 HW=5，response 里带 HW=5
    follower 更新 HW=5
```

→ follower 看到 HW 比 leader 实际 HW 慢一拍。这是为什么 leader truncate 后 follower 可能出现 log divergence（leader epoch 解决）。

</details>

### Q5.3 ⭐⭐ ISR 缩张触发 + replica.lag.time 调小风险

<details><summary>答案</summary>

触发：follower 超过 `replica.lag.time.max.ms`（默认 30s）没追上 leader LEO。

调小（如 10s）：

- 网络抖动就触发缩 ISR
- ISR 抖动频繁 → 元数据变更频繁 → controller / metadata log 压力

调大（如 5min）：

- 副本"假装在 ISR"实际没追上
- leader 切换时风险大

30s 是平衡。

</details>

### Q5.4 ⭐⭐ acks=1 vs acks=all + min.insync=1

<details><summary>答案</summary>

| 配置 | 行为 |
|---|---|
| acks=1 | leader 写入即 ack |
| acks=all + min.insync=1 | ISR 至少 1（含 leader）写入即 ack |

实际**等价**——都是 leader 写入即 ack。

差异在 ISR 缩到 0 时：

- acks=1：还是接受写（但持久化堪忧）
- acks=all + min.insync=1：ISR 含 leader 至少 1，等价

通常 min.insync >= 2 才有意义。

</details>

### Q5.5 ⭐⭐⭐ unclean leader election 何时该开

<details><summary>答案</summary>

**几乎从不开**（默认 false 是正确选择）。

唯一可能开的场景：

- 业务对**可用性**远比一致性敏感（如指标 / 日志）
- 接受最近几秒消息丢失换"任何时候都能写"

代价：ISR 缩到 0 + leader 挂时，从 OSR 选主，最后写的消息丢。

更好方案：提高 RF（如 RF=5 + min.insync=3），让 ISR 永远不缩到 1。

</details>

### Q5.6 ⭐⭐ Isr=1 但 Replicas=1,2,3 说明什么

<details><summary>答案</summary>

只有副本 1 在 ISR，副本 2 和 3 已经 OSR（落后超过 30s）。

危险：

- 副本 1 挂 → ISR 空 → partition 不可用（除非开 unclean）
- 不容忍单节点故障

要查：为什么 2 和 3 跟不上？网络？磁盘？broker 卡？

</details>

### Q5.7 ⭐⭐ preferred leader election 解决什么

<details><summary>答案</summary>

每个 partition 有 **preferred leader**（assignment 列表第一个副本）。

集群 leader 分布不均 → 部分 broker 热点。

preferred leader election：把 leader 切回 preferred 副本（assignment 列表第一个），让 leader 在 broker 间均匀分布。

```bash
kafka-leader-election.sh --election-type preferred ...
```

或开 `auto.leader.rebalance.enable=true` 自动。

</details>

### Q5.8 ⭐⭐ broker.rack 配置好处

<details><summary>答案</summary>

`node.attr.rack=us-east-1a`（每 broker 标 rack）+ `replica.selector.class=RackAwareReplicaSelector`：

1. **副本分散**：自动让 partition 副本分布在不同 rack（单 rack 挂不丢主副本）
2. **consumer rack-aware fetch**：优先读同 rack 的副本，省跨 rack 带宽

适合跨 AZ 部署。

</details>

### Q5.9 ⭐⭐ reassignment 期间 producer 超时

<details><summary>答案</summary>

reassignment 复制流量打满网卡。

修复：

```bash
kafka-reassign-partitions.sh ... --execute --throttle 52428800  # 50 MB/s
```

throttle 限制复制速率，保留业务带宽。

迁完后清理 throttle：

```bash
kafka-reassign-partitions.sh ... --verify
```

</details>

### Q5.10 ⭐⭐ min.insync=replication.factor 为什么不好

<details><summary>答案</summary>

例 RF=3 + min.insync=3：任何副本挂 → ISR=2 < 3 → 写入失败。

集群完全不能容忍单节点故障 → 可用性极差。

正确：min.insync = RF - 1（如 RF=3 + min.insync=2），容忍 1 节点挂。

</details>

---

## K06 存储与 Segment

### Q6.1 ⭐ 一个 segment 由哪几个文件组成

<details><summary>答案</summary>

- `.log`：实际消息记录
- `.index`：offset → 物理位置稀疏索引
- `.timeindex`：timestamp → offset 索引

Kafka 还可能有 `.snapshot`（producer state）和 `leader-epoch-checkpoint`（leader epoch）。

</details>

### Q6.2 ⭐⭐ .index 是稀疏的原因

<details><summary>答案</summary>

每 `log.index.interval.bytes`（默认 4KB）记一项，不是每条消息一项。

好处：

- 索引文件小（几 MB 而不是几 GB）
- 整索引能放内存
- 二分查找 O(log N) + 扫一小段 log

代价：找具体消息要扫最多 4KB log。

</details>

### Q6.3 ⭐⭐ log.segment.bytes=1MB 问题

<details><summary>答案</summary>

每 1MB 一个 segment → 几千个 segment 文件：

- 文件句柄爆（Too many open files）
- 索引文件多 → 内存映射开销
- retention 删除频繁

正常 1GB，特殊场景可调到 512MB（更准 retention）。

</details>

### Q6.4 ⭐⭐ retention 1 天但仍有 3 天前数据

<details><summary>答案</summary>

retention 按 **segment** 删，不按消息：

- 单 segment 1GB，写入慢可能很久才 roll
- 老 segment 内最新一条 timestamp 决定是否能删
- 即使 segment 内大部分消息超期，最新一条没超期就保留

修复：

- 调小 segment.bytes 让 roll 更勤
- 或加 log.roll.hours / log.roll.ms

</details>

### Q6.5 ⭐ log compaction 适合什么数据

<details><summary>答案</summary>

适合：

- 实体状态（用户、配置、价格）
- CDC 流（last write wins）
- 计算中间结果

不适合：

- 事件流（每条独立事件）
- 日志数据
- 时间序列指标

判断：**key 表示"实体 / 状态"** 就适合。

</details>

### Q6.6 ⭐⭐ tombstone 与 delete.retention.ms

<details><summary>答案</summary>

tombstone = `value=null`，表示 key 被删除。

compact 后保留 tombstone `delete.retention.ms`（默认 1 天），让所有 consumer 看到删除事件，然后才物理删除。

太短 → 慢 consumer 看不到删除，状态不一致。
太长 → 占空间。

</details>

### Q6.7 ⭐⭐⭐ tiered storage 本地与远程 retention 关系

<details><summary>答案</summary>

```properties
local.retention.ms=86400000     # 本地 1 天
retention.ms=7776000000          # 远程总 90 天
```

行为：

- 1 天内 segment 在本地
- 1-90 天 segment 在远程对象存储
- 90 天后整删

local.retention < retention，本地是远程的"热缓存"。

</details>

### Q6.8 ⭐⭐ partition 同时 segment roll 危害

<details><summary>答案</summary>

所有 partition 同时 roll → 同时关闭老 segment + 开新 segment + 索引重建：

- 磁盘 IO 飙高
- 文件句柄申请 / 释放高峰
- 监控误报"系统抖动"

修复：`log.roll.jitter.ms` 加随机抖动，分散 roll 时点。

</details>

### Q6.9 ⭐ offsetsForTimes 走什么索引

<details><summary>答案</summary>

`.timeindex`：timestamp → offset 的稀疏索引。

实现：

1. 二分 .timeindex 找最大 ≤ target_timestamp 的项 → 得到 offset
2. consumer.seek(offset) + poll

适合：时间穿越消费、按时间回溯。

</details>

### Q6.10 ⭐⭐ active segment 内旧 key 会 compact 吗

<details><summary>答案</summary>

**不会**。compact 只在已封闭 segment 上做。active segment 永远不被 compact。

所以快速变化的 key 在 active segment 期间会有多个版本，等 roll 后才合并。

调 `segment.ms` 让 roll 更勤可加快 compact。

</details>

---

## K07 精确一次

### Q7.1 ⭐ idempotent vs transactional 保证

<details><summary>答案</summary>

idempotent：

- 单 partition 内不重复
- 单 partition 内按发送顺序
- 单 producer 进程内

transactional：

- 上面所有 +
- 跨 partition / 跨 topic 原子（要么全 commit 要么全 abort）
- 跨 producer 重启（用 transactional.id 续身份）

</details>

### Q7.2 ⭐⭐ PID / epoch / sequence 作用

<details><summary>答案</summary>

- **PID**（Producer ID）：producer 全局唯一标识，由 coordinator 分配
- **epoch**：同 transactional.id 重启时递增，防 zombie
- **sequence**：单 partition 内单调递增，broker 用来去重

broker 端记 `(PID, partition) → last_sequence`，sequence 重复或乱序时处理。

</details>

### Q7.3 ⭐⭐⭐ transactional.id 必须稳定的原因

<details><summary>答案</summary>

`initTransactions()` 时：

- 同 transactional.id → 同 PID 但 epoch++
- 老 epoch 的请求被 fence

如果重启拿新 transactional.id：

- 老事务挂在 coordinator 永远不解决（除非超时 abort）
- 多实例各拿不同 ID → 没有 zombie 保护

实操：transactional.id = "prefix-" + stable_instance_id（K8s StatefulSet hostname）。

</details>

### Q7.4 ⭐⭐ read_committed 的 LSO 卡顿

<details><summary>答案</summary>

LSO = Last Stable Offset = 第一个未完成事务起点。

如果一个事务长时间不 commit：

- LSO 不前进
- read_committed consumer 看不到 LSO 之后的消息（即使这些消息不在该事务内）
- 整 partition 下游处理 stall

缓解：

- `transaction.timeout.ms` 调小，让卡住事务 timeout abort
- 业务侧短事务

</details>

### Q7.5 ⭐⭐ ProducerFencedException 根因

<details><summary>答案</summary>

`initTransactions` 时 coordinator 看到同 transactional.id 已有更高 epoch → 当前 producer 是 zombie → fence。

场景：

- 多实例错用同 transactional.id
- 长 GC 让监控误认挂掉，启动新实例（合法 fencing）

处理：fenced 立刻 close + 退出进程（不要重试）。

</details>

### Q7.6 ⭐⭐⭐ read-process-write 不能用 c.commitSync 的原因

<details><summary>答案</summary>

直接 commitSync 不在事务里：

```
事务 commit 成功 → output 写入有效
然后 c.commitSync() 失败 → offset 没记
重启后 → 重新处理 → output 重复 ❌
```

正确：`p.sendOffsetsToTransaction(offsets, c.groupMetadata())` 把 offset commit 进事务。

</details>

### Q7.7 ⭐⭐ EOS v1 vs v2

<details><summary>答案</summary>

v1（Kafka 2.5 前）：每 task 一个 transactional.id，但全 consumer 共享一个，rebalance 时复杂。

v2（KIP-447，Kafka 2.6 引入；要求 broker >= 2.5）：

- transactional.id 跟 consumer group + partition 绑定
- rebalance 时自动重新 init
- 网络开销少一半（事务 commit 与 offset commit 共享路径）

切 v1→v2 不能滚动（协议不兼容），要全停切。

</details>

### Q7.8 ⭐⭐ 长事务（10 min）影响

<details><summary>答案</summary>

- LSO 卡 10 分钟，下游 read_committed consumer 全 stall
- transaction.timeout.ms 默认 60s（producer 端），10min 事务早就会被触发 abort（除非显式调大）；broker 端上限 transaction.max.timeout.ms 默认 15min
- broker 端事务状态保留时间延长
- 失败重做的代价大（10 分钟工作丢）

实操：事务尽量短（< 1 分钟）。

</details>

### Q7.9 ⭐⭐⭐ Kafka EOS 能保证发邮件恰好一次吗

<details><summary>答案</summary>

**不能**。Kafka 事务只管 Kafka 内部 partition 写入 + offset commit。

发邮件是外部副作用：

- 事务 commit 后崩溃 → 邮件没发 → 处理重新跑 → 重复发
- 邮件发后崩溃前事务未 commit → 重处理 → 又发一次

要"Kafka + 外部副作用"原子：

- transactional outbox 模式（消息写到 DB outbox 表 + 业务表事务）
- CDC 把 outbox 推到 Kafka
- 这样 Kafka 端只需 idempotent

</details>

### Q7.10 ⭐⭐ transactional max.in.flight 上限 5

<details><summary>答案</summary>

transactional 隐含开启 idempotent，所以继承 idempotent 的 5 上限：

- broker 端 PID-partition 维护最近 5 个 sequence 的窗口
- 重试 / 乱序判定基于这窗口

idempotent / transactional producer 上限都是 5。

</details>

---

## K08 性能调优

### Q8.1 ⭐⭐ sendfile 比传统 read+write 快的原因

<details><summary>答案</summary>

传统 read+write：

- 4 次内存复制（含 DMA）
- 2 次系统调用
- 2 次上下文切换

sendfile：

- 1 次系统调用
- 0 次用户态复制
- DMA 直接把 page cache 推到网卡

效果：consumer 拉数据时 broker CPU 几乎为 0。

</details>

### Q8.2 ⭐⭐⭐ Kafka 不主动 fsync 也敢说可靠

<details><summary>答案</summary>

依赖**多副本**而非 fsync：

- 写 page cache → 异步 flush
- 同时复制到 N 个副本的 page cache
- 即使单 broker 异常崩（page cache 丢）→ 其他副本有数据 → leader 切换不丢

主动 fsync 会让吞吐降 5-10×，但安全性提升微小（多副本已经覆盖单点）。

</details>

### Q8.3 ⭐⭐ linger.ms vs batch.size 先到先发

<details><summary>答案</summary>

任一满足触发发送：

- batch.size 攒满 → 立刻发
- linger.ms 到 → 即使没满也发

两者协同：

- 高吞吐：大 batch + 长 linger
- 低延迟：小 batch + linger=0

</details>

### Q8.4 ⭐⭐ lz4 / zstd / gzip 选择

<details><summary>答案</summary>

| 算法 | CPU | 压缩比 | 用途 |
|---|---|---|---|
| lz4 | 极低 | 中 | **推荐**，通用 |
| snappy | 低 | 中 | 老项目 |
| zstd | 中 | 高 | 跨网带宽贵 |
| gzip | 高 | 高 | 几乎不用（zstd 替代） |

lz4 是甜区。zstd 适合存储 / 带宽更贵的场景。

</details>

### Q8.5 ⭐⭐⭐ JBOD vs RAID0

<details><summary>答案</summary>

JBOD（多独立磁盘）：

- Kafka 自己跨 broker 做副本
- 单盘坏只丢该盘上的 partition（其他磁盘 partition 不影响）
- Kafka 2.6+ 单盘失败可继续运行其他盘

RAID0：

- 单盘坏整 broker 挂
- 没有性能优势（Kafka 是顺序 IO）

**JBOD > RAID0** 在 Kafka 场景。

</details>

### Q8.6 ⭐⭐⭐ 单 partition 热点诊断 / 修复

<details><summary>答案</summary>

诊断：

```bash
# 看每 partition 写入速率
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=*,partition=*
```

修复：

- key 加 random 后缀
- 自定义 partitioner 散热点
- 业务拆分大客户
- 长期：业务侧重新设计 key

</details>

### Q8.7 ⭐⭐ THP 要关的原因

<details><summary>答案</summary>

Transparent Huge Pages 在 Kafka / 高 IO 场景下：

- 有过几次 stall 报告（kernel 做 huge page 合并时卡 IO）
- Kafka 的小记录 + 大量 mmap 不需要 huge page

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

</details>

### Q8.8 ⭐⭐ swappiness=1 而不是 0

<details><summary>答案</summary>

- swappiness=0：极端情况下内核仍可 swap（kernel 4.x+ 行为）
- swappiness=1：永远倾向不 swap，但保留极端情况下的逃生通道
- swappiness=10（默认）：会主动 swap，Kafka 不喜欢

设 1 是社区共识"几乎不 swap 但比 0 安全"。

</details>

### Q8.9 ⭐⭐ consumer lag 大但 CPU 不忙

<details><summary>答案</summary>

可能原因：

- broker 端 quota 限速（`consumer_byte_rate`）
- consumer 处理是 IO bound（写外部慢 sink）
- partition 分配不均（某 consumer 拿了大 partition）
- fetch.min.bytes 太大 + fetch.max.wait 长（请求间隔大）
- 网络 RTT 高

诊断：看 `fetch-throttle-time-avg`、`records-consumed-rate`、partition 分布。

</details>

### Q8.10 ⭐⭐⭐ broker P99 突然飙高，5 个排查方向

<details><summary>答案</summary>

1. **GC 暂停**：JVM GC log
2. **磁盘满 / IO 饱和**：iostat
3. **网卡饱和**：sar -n DEV
4. **ISR 重建 / replication 拥塞**：UnderReplicatedPartitions、recovery metrics
5. **请求队列堆积**：RequestQueueSize、ResponseQueueSize
6. **热 partition**：单 partition 写 / 读极高
7. **客户端突发**：某客户端瞬时写入翻倍

</details>

---

## K09 Kafka Streams

### Q9.1 ⭐ KStream / KTable / GlobalKTable 适合什么

<details><summary>答案</summary>

- **KStream**：事件流（订单、点击、操作日志）
- **KTable**：实体状态（用户档案、当前价格）
- **GlobalKTable**：小字典数据（汇率表、国家代码）—— 每实例全量

</details>

### Q9.2 ⭐⭐ State store changelog cleanup policy

<details><summary>答案</summary>

`cleanup.policy=compact`：

- state store 是"key → 状态"
- 只需保留每 key 最新值
- compact 把磁盘占用控制在合理范围

故 KTable / 聚合的 changelog 自动配 compact。

</details>

### Q9.3 ⭐⭐⭐ EOS v1 vs v2 in Streams

<details><summary>答案</summary>

`processing.guarantee`：

- `exactly_once`（v1）：每 task 独立 transactional producer
- `exactly_once_v2`（v2）：thread 级 producer 共享，事务 commit 路径优化

v2 网络开销少 50%，吞吐高 1.5-2×。KIP-447（Kafka 2.6 引入；要求 broker >= 2.5）推荐 v2。

</details>

### Q9.4 ⭐⭐ grace period 与迟到数据

<details><summary>答案</summary>

窗口关闭后 grace period 内仍接受迟到消息：

```java
TimeWindows.ofSizeAndGrace(Duration.ofMinutes(5), Duration.ofMinutes(1))
```

5min 窗口 + 1min grace：窗口 [10:00, 10:05) 在 10:06 真正关闭，期间仍接受 timestamp ∈ [10:00, 10:05) 的迟到消息。

grace 长 → 容忍乱序但等待久；grace 短 → 迟到消息丢失。

</details>

### Q9.5 ⭐⭐ KStream-KTable join vs KTable-KTable join

<details><summary>答案</summary>

- **KStream-KTable**：lookup 语义。每条 stream 来时查当前 KTable 状态。KTable 更新不触发 join。
- **KTable-KTable**：双向。任一 KTable 更新都触发 join，发出新 KTable 记录。

实战：

- 富化事件 → KStream-KTable
- 多状态 join（user × order_count → 实时仪表盘）→ KTable-KTable

</details>

### Q9.6 ⭐⭐⭐ Interactive Query 跨实例查 key

<details><summary>答案</summary>

```java
KeyQueryMetadata metadata = streams.queryMetadataForKey("store-name", "key", serializer);
if (metadata.activeHost().equals(localHost)) {
    // 本地查
} else {
    // 转发到 metadata.activeHost()
    httpClient.get("http://" + metadata.activeHost() + ":8080/query/" + key);
}
```

应用必须自己实现 HTTP 转发。每个实例都开 HTTP server。

</details>

### Q9.7 ⭐⭐ num.standby.replicas=1 好处和代价

<details><summary>答案</summary>

好处：

- failover 几秒（不用从 changelog 重建）
- 计划重启更快

代价：

- 内存 / 磁盘双倍（每 task 一个 standby）
- 多一份 changelog 跟随流量

适合：state 大、failover 时间敏感的场景。

</details>

### Q9.8 ⭐⭐⭐ RocksDB 用堆外内存对 K8s 影响

<details><summary>答案</summary>

RocksDB 用 native 内存（不在 JVM heap）：

- K8s container limit 必须包含 heap + RocksDB off-heap + JVM overhead
- 监控只看 heap 看不到真实占用，要看 container RSS
- OOM kill 可能因 RocksDB 不受控

解决：

- container memory >> heap（如 heap 4G、container 16G）
- 限制 RocksDB cache 大小（block_cache、write_buffer）
- 用 jemalloc / tcmalloc 替代默认 malloc 降碎片

</details>

### Q9.9 ⭐⭐ session window vs tumbling window

<details><summary>答案</summary>

| 窗口 | 行为 |
|---|---|
| tumbling | 固定时长，不重叠（每 5 分钟一个） |
| hopping | 固定时长 + advance（可重叠） |
| session | 同 key 间隔 < gap 算同 session |

session 适合：

- 用户访问 session 分析
- 一次性活动（如下单流程）

tumbling 适合：

- 周期性聚合（每分钟订单数）
- 时间序列指标

</details>

### Q9.10 ⭐⭐⭐ 何时用 Flink 而非 Streams

<details><summary>答案</summary>

选 Flink：

- 复杂 event-time + watermark 语义
- 跨多源数据（DB + Kafka + 文件）
- 严格 EOS（Flink 的 2PC 比 Streams 更彻底）
- 大数据量 + 复杂状态（Flink RocksDB + 增量 checkpoint 更强）
- 团队能撑独立集群

选 Streams：

- 团队 Java 栈
- 拓扑简单
- 不想搭独立集群
- 数据全在 Kafka

</details>

---

## K10 Connect 与 Schema Registry

### Q10.1 ⭐ source vs sink connector

<details><summary>答案</summary>

- **source**：外部 → Kafka。如 Debezium 从 MySQL binlog → Kafka topic。
- **sink**：Kafka → 外部。如 ES Sink 从 topic 写到 Elasticsearch。

source 内部是 producer，sink 内部是 consumer。

</details>

### Q10.2 ⭐⭐ distributed 模式三个内部 topic

<details><summary>答案</summary>

- `connect-configs`：connector 配置（compact）
- `connect-offsets`：source connector 进度（compact）
- `connect-status`：connector / task 状态（compact）

存集群元数据，重启 / failover 时恢复。

</details>

### Q10.3 ⭐⭐⭐ Schema Registry payload 格式

<details><summary>答案</summary>

```
[magic_byte=0][schema_id 4 bytes][serialized data]
```

- magic_byte：标识格式版本
- schema_id：到 registry 查 schema
- 数据：按 schema 序列化

consumer 拿到字节 → 读前 5 字节 → 查 schema → 反序列化。

</details>

### Q10.4 ⭐⭐ BACKWARD vs FORWARD 兼容

<details><summary>答案</summary>

- **BACKWARD**：新 schema 能读老数据。允许加可选字段 / 删字段。**默认**。适合 consumer 升级早于 producer。
- **FORWARD**：老 schema 能读新数据。允许删字段 / 加可选字段。适合 producer 升级早于 consumer。

业务上：

- 单团队控制 producer + consumer 升级顺序 → BACKWARD
- 多团队独立 → FULL 兼容（两者都要）

</details>

### Q10.5 ⭐⭐ Avro / Protobuf / JSON Schema

<details><summary>答案</summary>

| 格式 | 优势 | 劣势 |
|---|---|---|
| Avro | Confluent 强、schema evolution 规则严 | 跨语言工具链不均匀 |
| Protobuf | 跨语言、Google 标准 | schema evolution 要小心 |
| JSON Schema | 人类可读、灵活 | 性能差、type 弱 |

实战：Avro（Confluent 生态）或 Protobuf（新项目趋势）。

</details>

### Q10.6 ⭐⭐ SMT 适合什么

<details><summary>答案</summary>

适合：

- 简单变换（字段重命名、提取嵌套、脱敏、类型转换）
- 一对一映射

不适合：

- 复杂业务逻辑
- 跨记录聚合 / join
- 调外部服务

复杂逻辑用 Kafka Streams / Flink，不要塞 SMT。

</details>

### Q10.7 ⭐ DLQ 的作用 + 监控

<details><summary>答案</summary>

DLQ（Dead Letter Queue）：

- 处理失败的消息不阻塞主流
- 后续人工 / 工具分析

监控必要：

- DLQ topic 大小持续增长 → 系统性错误（不是偶然）
- 没人看 = 数据黑洞

配 `errors.deadletterqueue.topic.name` + 告警。

</details>

### Q10.8 ⭐⭐ Debezium + Avro 契合点

<details><summary>答案</summary>

- 表结构 → Avro schema 自然映射
- ALTER TABLE → 新 schema 自动注册到 Schema Registry
- BACKWARD 兼容刚好对应"加列 with default"
- consumer 拿到带 schema_id 的消息，能解析新老版本数据

CDC + Avro 是事实标准组合。

</details>

### Q10.9 ⭐⭐ sink 写入失败处理

<details><summary>答案</summary>

两个策略：

1. **errors.tolerance=none**：失败 → connector failed，停止消费。适合不容许任何错误。
2. **errors.tolerance=all + DLQ**：失败 → 写 DLQ，继续消费。适合大多数场景。

外加：sink 端要**幂等**（重试不重复写下游）。

</details>

### Q10.10 ⭐⭐⭐ source RUNNING 但 offset 不进

<details><summary>答案</summary>

可能原因：

1. **外部源没新数据**：正常等待
2. **轮询周期太长**：调 poll.interval
3. **filter 全过滤**：检查 transforms 配置
4. **凭证 / 权限问题**：能连但读不到数据
5. **某 partition 卡住**：单 task 阻塞影响其他

诊断顺序：

- 看 task status detailed
- 看 worker log
- 直接连外部源验证数据
- DEBUG 模式看 source.poll() 返回什么

</details>

---

## K11 Kafka 安全

### Q11.1 ⭐ SASL_PLAIN vs SASL_SSL

<details><summary>答案</summary>

- SASL_PLAINTEXT：SASL 认证 + 明文传输（密码可被窃听）
- SASL_SSL：SASL 认证 + TLS 加密

**SASL_PLAINTEXT 几乎不能在生产用**——除非 100% 内网且互信。

</details>

### Q11.2 ⭐⭐ mTLS vs SASL/SCRAM 适用

<details><summary>答案</summary>

- **mTLS**：服务对服务（不变身份），PKI 体系完善
- **SASL/SCRAM**：人 / 客户端，密码易轮换

混合：broker 间 mTLS，业务客户端 SASL/SCRAM。

</details>

### Q11.3 ⭐ allow.everyone.if.no.acl.found

<details><summary>答案</summary>

- **true**：没 ACL 时允许所有（迁移期友好，但不安全）
- **false**：没 ACL 时拒绝（默认 + 安全）

生产必须 false。迁移期可临时 true，但有严格期限。

</details>

### Q11.4 ⭐⭐⭐ transactional producer 需要的 ACL

<details><summary>答案</summary>

```bash
# 写 topic
--allow-principal User:alice --operation WRITE --topic orders
# 描述 topic
--allow-principal User:alice --operation DESCRIBE --topic orders
# 事务 ID
--allow-principal User:alice --operation WRITE --transactional-id tx-1
--allow-principal User:alice --operation DESCRIBE --transactional-id tx-1
# idempotent
--allow-principal User:alice --operation IDEMPOTENT_WRITE --cluster
```

主要：topic 写 + transactional-id 写 + cluster IdempotentWrite。

</details>

### Q11.5 ⭐⭐ quota 超过时 broker 行为

<details><summary>答案</summary>

**延迟响应**（throttle），不是拒绝。

broker 在响应中带 `throttle-time-ms`，客户端理论上应延迟下次请求。

```
kafka.server:type=Produce,name=throttle-time-ms
```

throttle-time > 0 → 客户端被限速，但请求最终成功。

</details>

### Q11.6 ⭐⭐ inter.broker 用 PLAINTEXT 安全吗

<details><summary>答案</summary>

**如果内网严格隔离，安全**。

- broker 间通信不经公网
- VLAN / 安全组限制只允许 broker 互访
- 性能最好（无 TLS 开销）

如果有任何"内网不可信"风险（如多租户云、共享网络），就要 inter.broker SSL。

</details>

### Q11.7 ⭐⭐ 证书过期后果

<details><summary>答案</summary>

- broker 证书过期 → 客户端连接失败（TLS handshake error）
- 业务全停
- mTLS 模式下，broker 也连不上彼此（集群挂）

预防：

- cert-manager 自动 renew
- 监控证书 expiry < 60 天告警
- 每年至少一次"模拟过期"演练

</details>

### Q11.8 ⭐⭐ KRaft 时代 ACL 存哪 改善

<details><summary>答案</summary>

- ZK 时代：`/kafka-acl` znode + watch 通知
- KRaft 时代：metadata log（AccessControlEntryRecord）+ broker fetch

改善：

- 变更延迟更低（broker fetch 增量）
- 没 watch 风暴
- 与其他元数据一致性更好（同一 raft log）

</details>

### Q11.9 ⭐⭐ SCRAM-SHA-256 vs SHA-512

<details><summary>答案</summary>

- SHA-256：快、安全够用
- SHA-512：慢、更安全（对暴力破解）

SCRAM 用 iteration（默认 4096-8192）放大 hash 计算，单次认证不一定慢。

生产推荐 SHA-512 + iterations 4096+。

</details>

### Q11.10 ⭐⭐⭐ OAUTHBEARER vs SCRAM

<details><summary>答案</summary>

| 维度 | SCRAM | OAUTHBEARER |
|---|---|---|
| 凭证 | 用户名密码 | JWT token |
| 集成 | Kafka 自管 | 与 IDP（Okta / Auth0 / Keycloak）集成 |
| 轮换 | 改密码 | token 自动短期 + refresh |
| 团队规模 | 小到中 | 大企业 SSO 体系 |
| Claim | 无 | 可带 role / department 等做授权 |

</details>

---

## K12 运维 + 替代品

### Q12.1 ⭐ 日常巡检 5 个核心指标

<details><summary>答案</summary>

1. **集群健康**：ActiveControllerCount=1
2. **UnderReplicatedPartitions=0**
3. **broker 全在线**
4. **consumer lag** 没超阈值
5. **磁盘使用率** < 80%

5 分钟巡检每天做。

</details>

### Q12.2 ⭐⭐ reassignment throttle 数值

<details><summary>答案</summary>

公式：

```
throttle ≈ 网卡带宽 × 0.3
```

例：10 Gbps 网卡 → throttle 350 MB/s（留 70% 给业务）。

太小：迁移慢
太大：业务被影响

观察业务延迟 + 监控调整。

</details>

### Q12.3 ⭐⭐⭐ partition 加倍对业务影响

<details><summary>答案</summary>

- **路由破坏**：`hash(key) % new_count` 让同 key 可能换 partition
- **顺序破**：相同 key 历史在 partition A，新数据可能在 partition B
- **已有数据不重分布**：老 partition 数据不动

如果业务依赖 key 顺序（如订单状态机），加 partition 是 breaking change。

正确：新建 topic（更多 partition）+ 双写 + consumer 切换 + 下线老 topic。

</details>

### Q12.4 ⭐⭐ 主备 vs 双活

<details><summary>答案</summary>

| 模式 | 复杂度 | RPO | RTO |
|---|---|---|---|
| 主备 | 低 | 几秒（复制延迟） | 几分钟（人工切换）|
| 双活 | 高 | 接近 0 | 接近 0 |

双活复杂在：

- 解决 echo（A→B→A 循环）
- 应用层标识来源集群
- 数据冲突解决

99% 场景主备够用。

</details>

### Q12.5 ⭐⭐ MirrorMaker 2 改 Identity 名

<details><summary>答案</summary>

默认：源 topic `orders` → 目标 `primary.orders`（加前缀防止 echo）。

要保持原名（双活场景）：

```properties
replication.policy.class=org.apache.kafka.connect.mirror.IdentityReplicationPolicy
```

但要解决 echo：

- 用 record header 标识来源（应用层）
- 或 MirrorMaker filter

</details>

### Q12.6 ⭐⭐⭐ Redpanda vs Kafka 核心差异

<details><summary>答案</summary>

- **语言**：Redpanda C++（无 JVM）vs Kafka Java
- **共识协议**：Redpanda per-partition Raft vs Kafka ISR
- **架构**：Redpanda thread-per-core vs Kafka 多线程
- **元数据**：都用 Raft（Kafka 4.0 KRaft）

性能：Redpanda 同样硬件吞吐 2-6×、延迟更稳。

生态：Kafka 远胜（Connect、Streams、社区）。

</details>

### Q12.7 ⭐⭐ Pulsar broker / storage 分离好处

<details><summary>答案</summary>

- broker 无状态 → 扩容只是加节点（不要数据迁移）
- 存储（BookKeeper）独立扩缩
- 多租户隔离更彻底

代价：组件复杂（broker + bookies + ZK），运维难。

</details>

### Q12.8 ⭐⭐⭐ WarpStream 延迟高的原因

<details><summary>答案</summary>

数据写 S3：

- S3 PUT 延迟 50-200ms 起步（vs SSD < 1ms）
- consumer 拉数据从 S3：GET 延迟
- 没 broker 本地缓存（全 S3）

写入路径完全没法绕过 S3 RTT。

适合：

- 日志 / 归档
- 容忍秒级延迟
- 数据量极大（S3 比 SSD 便宜）

</details>

### Q12.9 ⭐⭐ inter.broker.protocol.version 升级注意

<details><summary>答案</summary>

不能"升级二进制同时切协议版本"。流程：

1. 全 broker 升级二进制（still 宣称老协议）
2. 一个个 broker 改 inter.broker.protocol.version 重启
3. 完成

原因：升级中途，老 broker 收到新协议 message 会解析失败。先统一二进制再切协议。

</details>

### Q12.10 ⭐ RPO vs RTO 含义

<details><summary>答案</summary>

- **RPO**（Recovery Point Objective）：故障时**可容忍丢失多少数据**。RPO=0 意味着不能丢。
- **RTO**（Recovery Time Objective）：故障后**多久必须恢复**。RTO=10min 意味着 10 分钟内业务恢复。

Kafka MirrorMaker 异步复制：RPO 几秒，RTO 几分钟（人工切换）。

银行级要 RPO=0 + RTO < 1min：需要同步复制（成本高）。

</details>

---

## 综合大题

### Q.综合.1 ⭐⭐⭐ 设计一个百万 QPS 日志收集系统

<details><summary>答案</summary>

容量估算：
- 1M QPS × 500 字节 = 500 MB/s
- 7 天保留 × 3 副本 = 900 TB

集群：
- **12 broker**（64GB RAM / 32 core / NVMe 4TB×4 / 10Gbps）
- **3 KRaft controller**（专用）
- partition：120（10/broker）

Topic：`logs.app1`、`logs.app2` 等按业务分

Producer：
```properties
acks=1   # 日志可容忍极少丢失
linger.ms=20
batch.size=262144
compression.type=lz4
```

Broker：
```properties
num.network.threads=16
num.io.threads=32
num.replica.fetchers=8
log.retention.hours=168
log.cleanup.policy=delete
log.segment.bytes=1073741824
default.replication.factor=2   # 日志数据 2 副本足够
```

Tiered Storage：本地 1 天，S3 6 天。

Consumer：用 Kafka Streams 实时聚合关键指标 → Prometheus；写 S3 sink 长期归档。

监控：Burrow 看 lag、Cruise Control 自动 balance。

</details>

### Q.综合.2 ⭐⭐⭐ 实现 read-process-write EOS

<details><summary>答案</summary>

input topic：`orders`
output topic：`enriched-orders`
事务隔离：read_committed

```java
Properties cp = new Properties();
cp.put("group.id", "enricher");
cp.put("group.protocol", "consumer");     // KIP-848
cp.put("isolation.level", "read_committed");
cp.put("enable.auto.commit", "false");
cp.put("auto.offset.reset", "earliest");
KafkaConsumer<String, Order> c = new KafkaConsumer<>(cp);

Properties pp = new Properties();
pp.put("transactional.id", "enricher-" + System.getenv("INSTANCE_ID"));
pp.put("enable.idempotence", "true");
pp.put("acks", "all");
KafkaProducer<String, EnrichedOrder> p = new KafkaProducer<>(pp);
p.initTransactions();

c.subscribe(Collections.singleton("orders"));

while (running) {
    ConsumerRecords<String, Order> records = c.poll(Duration.ofMillis(1000));
    if (records.isEmpty()) continue;

    p.beginTransaction();
    try {
        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        for (ConsumerRecord<String, Order> r : records) {
            EnrichedOrder enriched = enrich(r.value());
            p.send(new ProducerRecord<>("enriched-orders", r.key(), enriched));
            offsets.put(
                new TopicPartition(r.topic(), r.partition()),
                new OffsetAndMetadata(r.offset() + 1)
            );
        }
        p.sendOffsetsToTransaction(offsets, c.groupMetadata());
        p.commitTransaction();
    } catch (ProducerFencedException e) {
        // zombie，立刻退出
        System.exit(1);
    } catch (Exception e) {
        p.abortTransaction();
    }
}
```

K8s StatefulSet 保证 INSTANCE_ID 跨重启稳定 → transactional.id 稳定 → fencing 工作。

</details>

### Q.综合.3 ⭐⭐⭐ 集群突然 leader 选举抖动，排查

<details><summary>答案</summary>

诊断：

```bash
# 1. 看 controller 是否稳定
kafka-metadata-quorum.sh ... describe --status
# CurrentVoters 是否在抖动

# 2. 看 ISR 变化
grep "ISR" server.log | tail -100

# 3. 看节点心跳
GET _nodes/stats（broker 端类似的）

# 4. 看 GC log
grep "Pause" gc.log
```

可能根因：

1. **网络抖动**：broker 间心跳超时
2. **GC 暂停**：broker JVM 长 GC
3. **磁盘 IO 卡**：broker append translog 慢
4. **controller 故障**：active controller 失联
5. **配置错**：session.timeout 设太小

修复：

- 网络：检查交换机、ping RTT
- GC：调 G1 参数 / 加 heap
- IO：换 SSD / 减负载
- controller：稳定 quorum
- 配置：session.timeout >= replica.fetch.wait

</details>

---

## 答题统计

| 章节 | 题数 | 难度分布 |
|---|---|---|
| K01 模型 | 10 | ⭐ 3 / ⭐⭐ 5 / ⭐⭐⭐ 2 |
| K02 KRaft | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| K03 Producer | 10 | ⭐ 1 / ⭐⭐ 5 / ⭐⭐⭐ 4 |
| K04 Consumer | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| K05 ISR | 10 | ⭐ 1 / ⭐⭐ 6 / ⭐⭐⭐ 3 |
| K06 存储 | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| K07 EOS | 10 | ⭐ 1 / ⭐⭐ 5 / ⭐⭐⭐ 4 |
| K08 性能 | 10 | ⭐ 0 / ⭐⭐ 6 / ⭐⭐⭐ 4 |
| K09 Streams | 10 | ⭐ 1 / ⭐⭐ 5 / ⭐⭐⭐ 4 |
| K10 Connect | 10 | ⭐ 2 / ⭐⭐ 6 / ⭐⭐⭐ 2 |
| K11 安全 | 10 | ⭐ 2 / ⭐⭐ 5 / ⭐⭐⭐ 3 |
| K12 运维 | 10 | ⭐ 2 / ⭐⭐ 5 / ⭐⭐⭐ 3 |
| 综合 | 3 | ⭐⭐⭐ 3 |
| **合计** | **123** | — |

---

## 自评标准

- 答对 > 100 题：能独立设计 / 运维生产 Kafka 集群
- 80-100：核心能力扎实，部分边角细节缺
- 60-80：还在中级，建议把不会的逐条研究
- < 60：从 K01-K02 重新读，把概念和实操结合

---

> 🔁 反馈：把不会的题写到笔记里，3 个月后再答一遍
