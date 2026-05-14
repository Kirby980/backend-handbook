# 精通 Redis Streams 与发布订阅：消息流、Consumer Group、Sharded Pub/Sub

> 关联章节：[08 事务与脚本](./08-精通-Redis-事务-脚本.md)、[07 Cluster](./07-精通-Redis-Cluster.md)、[12 生产实践](./12-精通-Redis-生产实践.md)

---

## 引言：Redis 不止是缓存

Redis 在 5.0 (2018) 引入了 **Streams**——一个时间序列日志结构 + Consumer Group。这让 Redis 可以承担一些**轻量级消息队列 / 事件流**场景，挑战了 Kafka / RabbitMQ 的"必须独立部署"的工程惯性。

加上更早就有的 **Pub/Sub**（5.0 之前唯一的消息机制）+ **Sharded Pub/Sub**（Redis 7.0 引入，解决 Cluster 下扇出过大问题），Redis 的"消息能力栈"已经足以应对：

- 日志事件流（用户行为、订单变更）
- 任务队列（处理失败重试、超时认领）
- 实时推送（聊天室、通知）
- 跨服务事件解耦

但**它不能替代 Kafka**——本章会讲清两者的能力边界。

读完之后你应该能回答：

- Streams 的 ID `1715587200-0` 是怎么生成的？保证唯一吗
- Consumer Group 的 Pending Entries List 是什么，怎么用它保证"消息至少一次"
- Sharded Pub/Sub 解决了原始 Pub/Sub 的什么致命问题
- 一个 Redis 实例能扛多少 Stream 写入？什么时候要换 Kafka

---

## 第一章：Pub/Sub —— 原始的扇出机制

### 1.1 基本用法

```
# Terminal A: 订阅
SUBSCRIBE chan1 chan2

# Terminal B: 发布
PUBLISH chan1 "hello"
→ (integer) 1   ← 返回收到该消息的订阅者数

# Pattern 订阅（glob 通配）
PSUBSCRIBE news.*
```

### 1.2 工作原理

```
+--------+    PUBLISH chan1 "msg"    +--------+
| Client |--------------------------> | Master |
+--------+                            +--------+
                                          |
                                          | (查 channel → subscribers list)
                                          v
                       +-----------------+----------------+
                       |                 |                |
                       v                 v                v
                  +--------+        +--------+       +--------+
                  | Sub A  |        | Sub B  |       | Sub C  |
                  +--------+        +--------+       +--------+
```

Redis 服务端维护一个 `pubsub_channels` dict：channel name → subscriber 客户端列表。PUBLISH 命令遍历列表逐个把消息推送给所有 subscriber。

### 1.3 致命缺陷：消息无持久化

```
Subscriber 上线之前发布的消息：丢失
Subscriber 断连重连期间的消息：丢失
某个 subscriber 处理慢导致 buffer 满：被强断（client-output-buffer-limit）+ 重连丢失
```

Pub/Sub 是**纯 fire-and-forget**——没有 ACK、没有重试、没有持久化。任何丢失都不可追溯。

### 1.4 Cluster 下的另一个大问题

Redis Cluster 下，PUBLISH 命令会**广播到集群所有节点**（每个节点维护各自的 subscriber 列表）。1 个发布消息要打全集群 N 个节点 → 网络放大 N 倍。

100 节点 Cluster 上发 1000 msg/s → 实际跨节点流量 100,000 msg/s。**这是为什么大集群下 Pub/Sub 是带宽杀手**。

### 1.5 适用场景

只有这些场景用原始 Pub/Sub：

- **消息丢失可接受**：实时游戏中的"在线状态广播"，丢一条不影响
- **订阅者长在线**：服务间 leader 选举、配置广播
- **集群规模小**：单实例或 ≤ 3 节点 Cluster

否则用 **Streams 或 Sharded Pub/Sub**。

---

## 第二章：Sharded Pub/Sub —— Redis 7.0 救场

### 2.1 解决"扇出全集群"问题

Sharded Pub/Sub 用 **slot 隔离**：

- 消息 channel 由 `slot = CRC16(channel) % 16384` 映射到一个 slot
- 该 slot 所在的 master 节点处理 SPUBLISH / SSUBSCRIBE
- **不广播到其他节点**

```
SSUBSCRIBE chan1            # 订阅（订阅 chan1 的 client 必须连到 chan1 对应 slot 的节点）
SPUBLISH chan1 "msg"        # 发布（发布者也必须连到该节点）
```

### 2.2 行为对比

| 维度 | Pub/Sub | Sharded Pub/Sub |
|---|---|---|
| 命令前缀 | (P)SUBSCRIBE / PUBLISH | S(P)SUBSCRIBE / SPUBLISH |
| Cluster 下流量 | 广播全集群 | 仅目标 slot 节点 |
| 持久化 | 无 | 无 |
| Pattern 订阅 | PSUBSCRIBE 通配 | SSUBSCRIBE 不支持 pattern |
| 适用规模 | 小集群 | 大集群 |

**新项目无脑选 Sharded Pub/Sub**——除非真要 pattern 订阅。

---

## 第三章：Streams —— 持久化的时间日志

### 3.1 数据模型

Stream 就是 R01 提到的 **Radix Tree + listpack** 物理实现。逻辑上：

```
mystream:
  +---------+-------+------+-------+
  | 1715587200000-0 | field1 = val1, field2 = val2 |  ← entry
  +---------+-------+------+-------+
  | 1715587200001-0 | event = login, user = alice  |
  +---------+-------+------+-------+
  | 1715587200001-1 | event = login, user = bob    |  ← 同毫秒内的第 2 条
  +---------+-------+------+-------+
```

每条 entry 有：

- **ID**：`<ms>-<seq>` 格式，单调递增。`<ms>` 是发布时的 Unix 毫秒时间戳；`<seq>` 是同毫秒内的序号
- **fields**：任意 K-V 对（类似 hash）

### 3.2 写入

```
XADD mystream * event login user alice
→ "1715587200000-0"

XADD mystream MAXLEN 10000 * event login user bob    # 写入时限制流长度
→ "1715587200001-0"

XADD mystream MAXLEN ~ 10000 * ...                    # 近似修剪（更快）
```

- `*` = 让服务端生成 ID（默认）
- `MAXLEN N` = 写入时把流截到最近 N 条
- `MAXLEN ~ N` = 近似截到 N 条（按 listpack 节点边界截，比精确快 10x）
- `MINID id` = 截到所有 ID > 给定值的（适合按时间清理）

### 3.3 读取

```
# 按 ID 范围读
XRANGE mystream - +                         # 全部，从早到晚
XRANGE mystream 1715587200000 +             # 从指定 ID 开始
XRANGE mystream - + COUNT 100               # 前 100 条
XREVRANGE mystream + -                      # 倒序

# 阻塞式读（等待新消息）
XREAD COUNT 10 BLOCK 5000 STREAMS mystream $    # 等 5 秒，从最后位置之后开始
XREAD COUNT 10 STREAMS mystream 0               # 从开头读
```

`$` = 当前流尾部（"只接新的"）；`0` = 从头开始。

### 3.4 长度与修剪

```
XLEN mystream            # 当前条数
XTRIM mystream MAXLEN 10000      # 主动修剪
XTRIM mystream MINID 1715000000000   # 删除时间戳 < 指定值的
```

---

## 第四章：Consumer Group —— 工作分发 + 至少一次

### 4.1 没有 Group 时

`XREAD` 适合**单消费者**或**多消费者各自读全量**（订阅模式）。要"多消费者分担一份消息"必须用 Consumer Group。

### 4.2 创建 Group

```
XGROUP CREATE mystream mygroup $          # 从当前尾部开始
XGROUP CREATE mystream mygroup 0          # 从开头开始
XGROUP CREATE mystream mygroup $ MKSTREAM # 流不存在时自动建
```

Group 在 stream 上是**轻量元数据**：记录"该 group 当前读到哪个 ID" + "每个 consumer 的 PEL（待 ACK 列表）"。

### 4.3 Consumer 读

```
XREADGROUP GROUP mygroup consumer-1 COUNT 10 BLOCK 5000 STREAMS mystream >
```

- `>` 特殊 ID = "从未投递给本 group 的新消息"
- `0` 或具体 ID = "重新读取我未 ACK 的消息"
- consumer 名字（如 `consumer-1`）自动注册（首次使用即注册）

Redis 服务端记录"这些消息已投递给 consumer-1"——**进入 PEL**（Pending Entries List）。

### 4.4 ACK

```
XACK mystream mygroup 1715587200000-0 1715587200001-0
→ (integer) 2   # 成功 ACK 2 条
```

消息一旦 ACK，从 PEL 移除——视为消费完成。

**关键**：Redis Streams 是"至少一次"——消息一定送达，可能多次。业务必须幂等。

### 4.5 PEL 与故障恢复

```
XPENDING mystream mygroup
→ 1) (integer) 5                ← PEL 总数
   2) "1715587200000-0"         ← 最早未 ACK
   3) "1715587200004-0"         ← 最晚未 ACK
   4) 1) 1) "consumer-1"
        2) "3"
      2) 1) "consumer-2"
        2) "2"

XPENDING mystream mygroup - + 10 consumer-1
→ 1) 1) "1715587200000-0"
      2) "consumer-1"
      3) (integer) 600000        ← 闲置了 10 分钟
      4) (integer) 3             ← 投递次数

```

每条未 ACK 消息记录：

- 投递给谁
- 闲置多久（idle time）
- 投递次数（delivery count）

### 4.6 XAUTOCLAIM —— 自动重新分配死消息

```
XAUTOCLAIM mystream mygroup consumer-2 60000 0 COUNT 10
```

把 consumer-1 闲置 > 60 秒的 PEL 消息**转给 consumer-2**。典型用法：

```python
# consumer 启动时定期跑这条 → 接管挂掉/慢的同事的消息
```

这是 Streams "故障自愈" 的核心——挂掉的 consumer 上的待处理消息不会被永远卡住。

### 4.7 重投递次数控制

XPENDING 返回的 `delivery_count` 可以用来"超过 N 次失败就放弃"：

```python
pending = r.xpending_range("mystream", "mygroup", "-", "+", 100)
for entry in pending:
    if entry["times_delivered"] > 3:
        # 进入死信队列
        r.xadd("dead_letter", {"original_id": entry["message_id"], ...})
        r.xack("mystream", "mygroup", entry["message_id"])
```

---

## 第五章：完整工作模式

### 5.1 单消费者：日志/事件订阅

```python
# consumer
last_id = "0"   # 从头开始
while True:
    res = r.xread({"events": last_id}, block=1000, count=100)
    if not res:
        continue
    for stream, messages in res:
        for msg_id, fields in messages:
            process(fields)
            last_id = msg_id
```

适用：监听变更通知、单一处理者。

### 5.2 多消费者负载均衡：Worker Pool

```python
# 一次性创建 group
try:
    r.xgroup_create("tasks", "workers", id="0", mkstream=True)
except:
    pass  # 已存在

# 每个 worker
def worker(name):
    while True:
        # 1) 先把自己 PEL 里之前没 ACK 的处理掉
        pending = r.xreadgroup("workers", name, {"tasks": "0"}, count=10)
        process_and_ack(pending)

        # 2) 再拿新消息
        new = r.xreadgroup("workers", name, {"tasks": ">"}, count=10, block=5000)
        process_and_ack(new)

        # 3) 定期接管慢同事
        r.xautoclaim("tasks", "workers", name, min_idle_time=60000, start_id="0", count=10)
```

适用：异步任务、订单处理、邮件发送。

### 5.3 多消费者各自读全量：Pub/Sub 替代

```python
# 每个订阅者一个独立 group（持久版的"广播"）
group = f"audit-{instance_id}"
r.xgroup_create("events", group, id="$", mkstream=True)

while True:
    msgs = r.xreadgroup(group, instance_id, {"events": ">"}, block=1000)
    for stream, items in msgs:
        for msg_id, fields in items:
            handle(fields)
            r.xack("events", group, msg_id)
```

每个 group 独立 offset → 多个订阅者各自接收全量。**这是替代 Pub/Sub 的持久化方案**。

---

## 第六章：性能与容量

### 6.1 写入吞吐

- 单 Redis 实例 Streams XADD：**~100k/s 单核**（小消息）
- 配合 pipeline 批量 XADD：**~500k/s**
- 配合 `MAXLEN ~`：吞吐降 < 5%

### 6.2 内存占用

每条 entry：

- Radix Tree 节点 overhead（共享前缀，分摊很小）
- listpack 中：约 30-60 字节 + 各 field/value 长度

实测 100 字节 payload 的 entry 约占 150-200 字节内存。1 亿 entry ≈ 15-20 GB。

### 6.3 与 Kafka 对比

| 维度 | Redis Streams | Kafka |
|---|---|---|
| 单 stream 吞吐 | 100k/s 单核 | 1M/s 单 partition |
| 持久化 | RDB / AOF（同进程） | 独立 broker 磁盘日志 |
| 横向扩展 | 靠 Cluster slot 切分 | 原生 partition |
| Consumer 协议 | XREAD/XREADGROUP + XACK | poll + offset commit |
| 重试 / 死信 | 业务自实现（XAUTOCLAIM） | Connect / Streams 内建 |
| 消息保留 | MAXLEN / MINID 修剪 | retention.ms / .bytes |
| 适用 | 单 DC、~MB/s、低延迟 | 跨 DC、~GB/s、批处理 |
| 部署复杂度 | 已有 Redis 直接用 | 独立集群 + ZK/KRaft |

**经验值**：单 stream 吞吐 < 100 MB/s + 单 DC + 业务量级中等 → Streams 足够。**超过**或要跨 DC 复制 → Kafka。

---

## 第七章：Cluster 下的 Streams

### 7.1 单 stream = 单 slot

`XADD mystream *` 的 `mystream` 是 key → 落到某 slot → 单节点处理。**Stream 不内置分片**——它是单节点资源。

要扩容：**业务侧分片**。

```python
def write(event):
    shard = hash(event["user_id"]) % 16
    r.xadd(f"events:{{shard{shard}}}", event)
```

用 hash tag `{shard0}` ... `{shard15}` 让 16 个 stream 落到不同 slot。读取时各 consumer 处理各自 shard。

### 7.2 Consumer Group 与 Cluster

Group 元数据跟 stream 在同一节点。consumer 只能连那个节点。

如果业务侧 16 个 shard stream：每个 stream 一个 group，consumer 各自连各自节点。**复杂度自管理**。

---

## 第八章：观察与故障排查

### 8.1 Stream 状态

```
XLEN mystream
XINFO STREAM mystream FULL
XINFO GROUPS mystream
XINFO CONSUMERS mystream mygroup
```

输出：

```
XINFO STREAM mystream
1) "length" / (integer) 12345
2) "radix-tree-keys" / (integer) 25       ← Radix Tree 节点数
3) "radix-tree-nodes" / ...
4) "groups" / (integer) 2
5) "last-generated-id" / "1715587200000-5"
6) "first-entry" / ...
7) "last-entry" / ...
```

### 8.2 健康指标

- `XLEN mystream`：流长度——如果只增不减，要看是否漏了 XTRIM
- `XPENDING mygroup` 总数：持续高 = consumer 跟不上或大量未 ACK
- `XINFO CONSUMERS` 各 consumer idle time：长时间 idle 的 consumer 可能挂了
- 各 consumer 的 PEL 数：分布不均 = 工作分配偏斜

---

## 第九章：生产级最佳实践

1. **写入必带 MAXLEN ~ N**——否则 stream 无限增长。生产典型 N=百万-千万
2. **Consumer Group 是分发模型，独立 group 才能广播**——别误用
3. **XAUTOCLAIM 定期跑**——别让挂掉 consumer 的消息卡死
4. **业务必须幂等**——至少一次保证，可能重复
5. **PEL 超过阈值告警**——consumer 跟不上的早期信号
6. **大流量 stream 业务侧分片**——单 stream 单节点瓶颈在 100k/s
7. **不要用 Pub/Sub 替代消息队列**——除非真的 fire-and-forget
8. **大 Cluster 用 Sharded Pub/Sub**——节省扇出带宽
9. **死信队列分开 stream 存**——避免长 PEL 影响 group 性能
10. **持久化必开**——Streams 没有 AOF/RDB 等于内存 fire-and-forget

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：Pub/Sub 当消息队列用
任何离线时间的消息都丢。生产系统几乎必出问题。

### ❌ 陷阱 2：Stream 不修剪
内存疯长。XADD 写入时必带 MAXLEN ~。

### ❌ 陷阱 3：consumer 拿了消息后崩溃
PEL 里那条卡死。需要其他 consumer 用 XAUTOCLAIM 或 XCLAIM 接管。

### ❌ 陷阱 4：业务非幂等
重投递时数据双写。INCR / SADD 等天然幂等的还好，写库的要带唯一约束。

### ❌ 陷阱 5：单 stream 当全公司事件总线
吞吐瓶颈。按业务域 / 用户 hash 分多个 stream。

### ❌ 陷阱 6：Sharded Pub/Sub 与 Cluster 没用 hash tag
订阅者和发布者都要连同一个 slot 的节点。channel 用 `{tag}:event` 形式确保 slot 一致。

### ❌ 陷阱 7：XREADGROUP 用 `0` 而非 `>` 拿新消息
`0` 是重读自己 PEL；`>` 才是拿新消息。

### ❌ 陷阱 8：MAXLEN 写得很大但内存压力其实在别处
真正吃内存的可能是 PEL（每条未 ACK 都占元数据）。

### ❌ 陷阱 9：把 Streams 当数据库做大量 XRANGE 查询
XRANGE 是 O(N)。不适合"按时间范围检索"的高频读——上 ES 或时序库。

### ❌ 陷阱 10：以为 XGROUP CREATE 是幂等的
重复创建会报 BUSYGROUP 错误。要 try/except 或先 XINFO 检查。

---

## 第十一章：练习题

**练习 1**：设计一个用 Streams 实现的"用户行为日志"管道：5 个生产者写入，3 个消费者处理，要求消费失败自动重试，重试 5 次仍失败转死信。

**练习 2**：1 亿条历史事件存在 stream 里，每条 200 字节 payload。估算内存 + 修剪策略。

**练习 3**：在 Redis Cluster 100 节点下，Pub/Sub 发 1000 msg/s 实际打到网络的流量是多少？换成 Sharded Pub/Sub 呢？

**练习 4**：consumer 处理消息时网络抖动 30 秒。PEL 里的消息能被其他 consumer 接管吗？需要哪些前提？

**练习 5**：写一段 Python 脚本，每分钟扫一次 PEL，把 delivery_count > 3 的消息转到死信 stream 然后 ACK。

---

## 参考答案

**练习 1**：

```python
# 生产者
def producer(event):
    r.xadd("user_events", event, maxlen=10_000_000, approximate=True)

# 消费者
def consumer(name):
    try:
        r.xgroup_create("user_events", "processors", id="$", mkstream=True)
    except: pass

    while True:
        # 自愈：接管慢同事
        r.xautoclaim("user_events", "processors", name,
                     min_idle_time=60000, start_id="0", count=10)

        # 拿新消息
        msgs = r.xreadgroup("processors", name,
                            {"user_events": ">"}, count=10, block=5000)
        for stream, items in msgs:
            for msg_id, fields in items:
                # 看投递次数
                info = r.xpending_range("user_events", "processors",
                                        msg_id, msg_id, 1)
                if info and info[0]["times_delivered"] >= 5:
                    r.xadd("dead_letter", {**fields,
                                            "_original_id": msg_id,
                                            "_failures": info[0]["times_delivered"]})
                    r.xack("user_events", "processors", msg_id)
                    continue
                try:
                    process(fields)
                    r.xack("user_events", "processors", msg_id)
                except Exception:
                    # 不 ACK，自然进入下次重投递
                    pass
```

**练习 2**：每条 ~200B + ~50B overhead = ~250B/entry → 1 亿 ~ 25 GB。
修剪：业务上保留多久？保 30 天事件用 `MINID = (now - 30天)`；保最新 1000 万条用 `MAXLEN ~ 10000000`。
**注意 PEL**：未 ACK 消息也占内存。监控 XPENDING 总数。

**练习 3**：
- 原始 Pub/Sub：100 节点广播，1000 msg → 100 节点都收到 → 实际跨节点 + 内部 dispatch 流量是 100 倍 = 100k msg/s 节点间
- Sharded Pub/Sub：仅 channel 对应的 slot 节点处理 → 1000 msg/s 单节点，无跨节点广播

100 倍带宽差异。大集群必须 Sharded。

**练习 4**：能。前提：
1. Group 已创建
2. 其他活跃 consumer 调用 `XAUTOCLAIM` 或 `XCLAIM`，参数 `min_idle_time` < 30 秒
3. 抖动 consumer 重连后会发现自己 PEL 被清空（消息已被认领）

`XAUTOCLAIM mystream mygroup new-consumer 30000 0 COUNT 10`：把 idle > 30s 的消息转给 new-consumer。

**练习 5**：

```python
import redis
import time

r = redis.Redis()
STREAM = "mystream"
GROUP = "mygroup"
DLQ = "mystream:dlq"

while True:
    pending = r.xpending_range(STREAM, GROUP, "-", "+", 1000)
    for entry in pending:
        msg_id = entry["message_id"]
        delivered = entry["times_delivered"]
        if delivered > 3:
            # 取消息内容
            msgs = r.xrange(STREAM, msg_id, msg_id, count=1)
            if msgs:
                _, fields = msgs[0]
                fields[b"_original_id"] = msg_id
                fields[b"_delivered"] = str(delivered).encode()
                r.xadd(DLQ, fields, maxlen=100000, approximate=True)
            r.xack(STREAM, GROUP, msg_id)
    time.sleep(60)
```

---

## 小结

| 机制 | 持久化 | Cluster 友好 | 投递保证 | 适用 |
|---|---|---|---|---|
| Pub/Sub | ✗ | ✗（扇出全集群） | At-most-once | 仅小集群 / 可丢 |
| Sharded Pub/Sub | ✗ | ✓ | At-most-once | 大集群替代 Pub/Sub |
| Streams + XREAD | ✓ | 单 stream 单 slot | 客户端管理 offset | 单消费 / 全量订阅 |
| Streams + Group | ✓ | 单 stream 单 slot | At-least-once + PEL | 工作队列 / 异步任务 |

四条铁律：

1. **Pub/Sub 不是消息队列**——丢消息是常态
2. **Streams 至少一次，幂等是底线**
3. **MAXLEN 必带**——别让 stream 无限增长
4. **GB/s 流量找 Kafka**——Streams 是"轻量级 + 集成"的甜区

下一篇 **R08 — 精通 Redis 性能与单线程模型**：单线程为什么不慢、I/O 多线程怎么帮忙、latency monitor 与 slowlog 实战。
