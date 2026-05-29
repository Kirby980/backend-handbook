# 精通 Event Sourcing 与 CQRS：事件溯源、物化视图与读写分离

> 关联章节：[精通分布式事务](./08-精通-分布式事务.md)、[精通异步消息与事件驱动](./07-精通-异步消息与事件驱动.md)、[精通服务拆分与 DDD](./02-精通-服务拆分与-DDD.md)

---

## 引言：从"当下状态"到"全部历史"的思维迁移

在传统 CRUD 系统里，数据库存的是"现在"——一行记录 `accounts(id=1, balance=500)`，今天是 500，明天是 800，记录被覆盖，过去就消失了。

如果有一天 CEO 在凌晨 3 点问你："3 天前 14:23 那一刻的余额是多少？"——CRUD 系统会瑟瑟发抖：可能查不到，可能日志已被归档，可能要扫几 TB 慢查询。

而 **Event Sourcing**（事件溯源）给出的回答是：**所有的改变都是事件，我们存的是事件流，状态可以从事件流"重放"出来**。问 3 天前的余额？把 3 天前之前的所有事件 replay 一遍就行。

学完这一章你应能回答：

- Event Sourcing 究竟在解决什么问题？为什么不是"日志表"那么简单？
- 它跟传统 CRUD 在数据模型、查询、审计、复杂度上的本质区别是什么？
- 事件存储（Event Store）有哪些选型？EventStoreDB、Kafka、Postgres append-only 怎么选？
- 全量 replay 太慢怎么办？快照（Snapshot）的设计原则是什么？
- 业务演化导致老事件格式过期怎么办？Upcaster 是什么？
- 什么是 CQRS？它和 ES 是不是一回事？
- 哪些业务**应该**用 ES？哪些业务用了反而是灾难？
- 如何把一个 CRUD 账户服务改造为 ES + CQRS？

ES 不是"看起来很酷"的技术——它是**一种数据建模哲学**，用对了所向披靡，用错了会让整个团队陷入认知地狱。本章会用 12 章的篇幅把这件事讲透。

---

## 第一章：ES 核心思想 —— 事件序列是唯一真实来源

### 1.1 一句话定义

**Event Sourcing**：不存储"当前状态"，而是存储一系列**不可变事件**（events），当前状态由事件按顺序 replay 计算得出。

```
传统 CRUD：
    state(t) = 直接读取数据库

Event Sourcing：
    state(t) = fold(events, initial_state) 其中 events.time <= t
```

### 1.2 一个银行账户的对比

**CRUD 版本**：

```sql
CREATE TABLE accounts (
    id      BIGINT PRIMARY KEY,
    balance NUMERIC(18, 2) NOT NULL,
    updated_at TIMESTAMPTZ
);

UPDATE accounts SET balance = balance + 100 WHERE id = 1;   -- 存款
UPDATE accounts SET balance = balance - 30 WHERE id = 1;    -- 取款
```

数据库里只有一行：`(id=1, balance=70, updated_at=2026-05-25)`。**过去的所有变化都被覆盖了**。

**ES 版本**：

```sql
CREATE TABLE account_events (
    id           BIGSERIAL PRIMARY KEY,
    account_id   BIGINT NOT NULL,
    event_type   VARCHAR(64) NOT NULL,
    payload      JSONB NOT NULL,
    version      BIGINT NOT NULL,
    occurred_at  TIMESTAMPTZ NOT NULL,
    UNIQUE (account_id, version)
);
```

数据：

| id | account_id | event_type | payload | version | occurred_at |
|---|---|---|---|---|---|
| 1 | 1 | AccountOpened | `{"initial":0}` | 1 | 2026-05-20 10:00 |
| 2 | 1 | MoneyDeposited | `{"amount":100}` | 2 | 2026-05-22 12:00 |
| 3 | 1 | MoneyWithdrawn | `{"amount":30}` | 3 | 2026-05-25 14:00 |

要查"当前余额"，要把所有事件 replay 一遍：

```go
func ReplayBalance(events []Event) float64 {
    var balance float64
    for _, e := range events {
        switch e.Type {
        case "AccountOpened":
            balance = e.Payload["initial"].(float64)
        case "MoneyDeposited":
            balance += e.Payload["amount"].(float64)
        case "MoneyWithdrawn":
            balance -= e.Payload["amount"].(float64)
        }
    }
    return balance
}
```

### 1.3 三个关键属性

1. **Append-Only**：事件**只追加不修改**，写错了也只能再追加一个"修正事件"
2. **不可变**：事件代表"已经发生的事实"，事实不会变
3. **有序**：每个聚合内的事件**严格有序**（用 version 或 sequence 编号）

### 1.4 为什么这么折腾？

| 能力 | CRUD | Event Sourcing |
|---|---|---|
| 当前状态 | 直接读 | 需要 replay |
| 历史变化 | 看不到（除非加日志表） | 天然有 |
| 时间旅行 | 不可能 | 简单：截止某 version 之前 replay |
| 审计 | 需要额外审计表 | 事件本身就是审计 |
| 业务追溯 | "为什么 balance=70？"——查不到 | 看事件序列即可 |
| 修复 bug 后重算 | 几乎不可能 | replay 即可 |
| 复杂查询 | SQL 即可 | 需要物化视图（CQRS） |
| 写复杂度 | 低 | 中-高 |
| 读复杂度 | 低 | 高（如果不做投影） |
| 团队认知成本 | 低 | 高 |

ES 的核心承诺：**用写时的复杂度，换读时的灵活、审计的完整、和未来需求的可演化**。

---

## 第二章：与传统 CRUD 对比

### 2.1 详细对比

| 维度 | CRUD | Event Sourcing |
|---|---|---|
| **存储模型** | 当前状态行 | 事件流 |
| **写入** | UPDATE / INSERT / DELETE | 仅 APPEND |
| **读取** | SELECT 当前行 | replay 或读物化视图 |
| **审计** | 单独审计表 | 事件即审计 |
| **变更追溯** | 困难 | 天然 |
| **撤销操作** | UPDATE 回滚 | 追加"反向事件" |
| **数据演化** | 改 schema | 加新事件 + Upcaster |
| **存储成本** | 低 | 高（保留全历史） |
| **查询性能** | 直接 SQL | 需要投影表 |
| **并发控制** | 行锁 / MVCC | 版本号 / 乐观锁 |
| **复杂度** | 入门简单 | 入门陡峭 |
| **团队适应** | 易 | 需培训 |

### 2.2 一个真实场景：库存超卖事故复盘

某电商 CRUD 系统发生超卖。复盘时发现 inventory.stock 从 5 直接变成 -3，但没人知道是怎么变的：

- 是不是有 8 个并发请求？
- 是不是有人手动改了数据库？
- 是不是某个定时任务出 bug？

CRUD 表没有历史，只有日志（不全），DBA 通宵也没复盘清楚。

**如果是 ES**：

```
v=23  StockReserved   amount=1   user=A
v=24  StockReserved   amount=1   user=B
v=25  StockReserved   amount=1   user=C
v=26  StockReserved   amount=1   user=D     ← stock=1
v=27  StockReserved   amount=1   user=E     ← 超卖发生
v=28  StockReserved   amount=1   user=F
```

事件序列一看就知道——业务逻辑没在写入时校验 `stock >= 0`。**问题定位时间从数小时降到数分钟**。

### 2.3 何时 ES 优于 CRUD（业务证据）

满足下列任一条，ES 的边际价值就开始显现：

- 业务**需要审计 / 合规**（金融、医疗、政府）
- 业务**经常需要时间旅行查询**（"昨天此刻库存？"）
- 业务**复杂度高、迭代频繁**（需要保留历史以便修复 bug 后 replay）
- 业务**强依赖事件**（事件就是业务本身，例如行情、订单流）

---

## 第三章：事件存储 Event Store 选型

事件存储是 ES 系统的核心基础设施。三种主流选型：

### 3.1 选型 A：EventStoreDB（专用）

[EventStoreDB](https://www.eventstore.com/) 是一个专门为 ES 设计的数据库，原生支持：

- 事件流（Stream）
- 订阅（Catch-up Subscription、Persistent Subscription）
- Projection（投影引擎）
- 快照
- HTTP / gRPC / TCP 客户端

```go
// 写入
client.AppendToStream(ctx, "account-1", esdb.AppendToStreamOptions{
    ExpectedRevision: esdb.Revision(2),
}, esdb.EventData{
    EventType:   "MoneyDeposited",
    ContentType: esdb.ContentTypeJson,
    Data:        []byte(`{"amount": 100}`),
})

// 读取
stream, _ := client.ReadStream(ctx, "account-1",
    esdb.ReadStreamOptions{Direction: esdb.Forwards}, 100)
```

| 优点 | 缺点 |
|---|---|
| 原生支持 ES 语义 | 引入新组件 |
| 性能优秀（写入 50k+/s） | 团队学习成本 |
| 投影、订阅开箱即用 | 社区比 Kafka 小 |
| 内置 ACL、cluster | 商业版才有高级功能 |

适合：**Greenfield 项目，团队愿意拥抱专用方案**。

### 3.2 选型 B：Kafka + 事件主题

Kafka 天然就是"append-only 事件日志"，常被当作 Event Store 使用：

```
Topic: account-events
  Partition 0: [evt-1] [evt-2] [evt-3] ...
```

```go
// 写入
producer.Send(&kafka.Message{
    Topic: "account-events",
    Key:   []byte("account-1"),    // 同 account 同 partition，保证有序
    Value: eventPayload,
})

// 消费 / replay
consumer.Subscribe("account-events")
for msg := consumer.Poll() {
    apply(msg)
}
```

| 优点 | 缺点 |
|---|---|
| 团队通常已有 Kafka | 不是真正的"数据库" |
| 极高吞吐 | 没有原生的 stream-per-aggregate（要靠 key 分区） |
| 与下游消费天然集成 | 长期保留要配置（默认 7 天） |
| 工具链丰富 | 不支持随机查询（要顺序消费） |

适合：**事件量大、需要与流式管道集成的场景**。

### 3.3 选型 C：PostgreSQL Append-Only 表

最简单的方案：用普通的关系数据库，建一张 append-only 表：

```sql
CREATE TABLE events (
    id            BIGSERIAL PRIMARY KEY,
    aggregate_id  VARCHAR(64) NOT NULL,
    aggregate_type VARCHAR(64) NOT NULL,
    event_type    VARCHAR(64) NOT NULL,
    payload       JSONB NOT NULL,
    metadata      JSONB,
    version       BIGINT NOT NULL,
    occurred_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (aggregate_id, version)
);

CREATE INDEX ON events (aggregate_id, version);
CREATE INDEX ON events (event_type, occurred_at);
```

写入（带版本检查）：

```go
_, err := db.ExecContext(ctx, `
    INSERT INTO events(aggregate_id, aggregate_type, event_type, payload, version)
    VALUES ($1, $2, $3, $4, $5)`,
    accountID, "Account", "MoneyDeposited", payload, expectedVersion+1)
if err != nil {
    if isUniqueViolation(err) {
        return ErrConcurrentModification    // 乐观锁失败
    }
    return err
}
```

读取 / replay：

```sql
SELECT event_type, payload, version
FROM events
WHERE aggregate_id = $1
ORDER BY version ASC;
```

| 优点 | 缺点 |
|---|---|
| 团队完全熟悉 | 单点压力（高 TPS 需要分库） |
| 与业务库可以同事务 | 索引膨胀 |
| 任意 SQL 查询 | replay 需要 N 次 IO |
| Outbox 模式直接复用 | 没有原生订阅（要 LISTEN/NOTIFY 或 CDC） |

适合：**中小型 ES 项目，TPS < 5k**。这也是大多数生产 ES 系统的起点。

### 3.4 三种选型对比

| 维度 | EventStoreDB | Kafka | Postgres |
|---|---|---|---|
| 性能 | 高 | 极高 | 中 |
| 易用 | 中 | 中 | 高 |
| 团队认知 | 新 | 已有 | 已有 |
| 查询能力 | 中 | 弱 | 强 |
| 订阅能力 | 强 | 强 | 弱（需 CDC） |
| 长期存储 | 强 | 需配置 | 强 |
| 成本 | 商业 / 自建 | 已有集群 | 已有集群 |

**经验法则**：

- TPS < 5k：Postgres
- TPS 5k-50k 且和流式管道紧密：Kafka
- 业务 = 事件本身（如 IoT 设备数据 / 金融行情 / 审计场景）：EventStoreDB

---

## 第四章：Replay 与 Snapshot

### 4.1 全量 Replay 的痛

随着事件累积，每次加载聚合都要 replay 所有事件：

```
Account 1 events: 10,000,000 条
Load: SELECT ... ORDER BY version → 10M 行
Replay: for-loop apply → 几十秒甚至几分钟
```

显然不可接受。

### 4.2 Snapshot 设计

**Snapshot**：定期把"replay 到某 version 时的状态"序列化保存：

```sql
CREATE TABLE snapshots (
    aggregate_id   VARCHAR(64) NOT NULL,
    aggregate_type VARCHAR(64) NOT NULL,
    version        BIGINT NOT NULL,
    state          JSONB NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_id, version)
);
```

加载流程：

```go
func LoadAggregate(ctx context.Context, id string) (*Account, error) {
    // 1. 读最近 snapshot
    snap, _ := loadLatestSnapshot(ctx, id)
    var account *Account
    var fromVersion int64
    if snap != nil {
        account = deserializeSnapshot(snap.State)
        fromVersion = snap.Version + 1
    } else {
        account = NewAccount(id)
        fromVersion = 1
    }

    // 2. 加载 snapshot 之后的事件
    events, _ := loadEvents(ctx, id, fromVersion)

    // 3. 在 snapshot 基础上 apply
    for _, e := range events {
        account.Apply(e)
    }
    return account, nil
}
```

### 4.3 Snapshot 频率

| 策略 | 描述 | 适用 |
|---|---|---|
| 每 N 事件 | 例如每 100 个事件做一次 | 通用 |
| 每 N 秒 | 例如每 60s 检查一次活跃聚合 | 高频写入 |
| 异步生成 | 后台任务批量制作 | 写入热点 |
| 显式触发 | 业务上有"日终结算" | 金融、对账 |

**经验法则：snapshot 频率让 replay 控制在 100ms 以内**。

### 4.4 Snapshot 不是"真相"

关键认知：**snapshot 只是 replay 的加速缓存，不是真相**。如果业务规则改了，你应该能从 0 开始 replay 重新生成 snapshot。

```
Truth     = Event Stream
Cache     = Snapshot
Truth ⇒ Cache (可重新生成)
Cache ⇏ Truth (snapshot 永远不能改写历史)
```

### 4.5 Snapshot 存储选型

- 同库的 `snapshots` 表：最简单
- Redis：极快，但要 backup
- S3 / 对象存储：海量聚合时分散热点
- EventStoreDB 自带 snapshot 流

### 4.6 Snapshot 与 Schema 演化

业务规则变了，旧 snapshot 失效怎么办？

策略一：版本化 snapshot

```json
{
  "schema_version": 2,
  "state": { ... }
}
```

加载时校验 version，不匹配则丢弃，重新 replay。

策略二：永远从 event 流重建（牺牲性能换正确性）。

---

## 第五章：Schema Evolution —— 事件版本升级

事件是不可变的，但**事件的 schema 会演化**。这是 ES 工程上最棘手的问题。

### 5.1 演化三种类型

| 类型 | 描述 | 难度 |
|---|---|---|
| **加字段** | 新加可选字段 | 低 |
| **删字段** | 废弃字段（但保留兼容） | 中 |
| **改语义** | 同名字段含义变了 / 字段类型变了 | 高 |

### 5.2 三种应对策略

#### 策略 A：Versioned Events

为每种事件加版本号：

```json
{ "event_type": "MoneyDeposited.v1", "amount": 100 }
{ "event_type": "MoneyDeposited.v2", "amount": 100, "currency": "USD" }
```

apply 时按版本分发：

```go
switch e.Type {
case "MoneyDeposited.v1":
    account.Balance += e.Payload.Amount  // 默认 USD
case "MoneyDeposited.v2":
    if e.Payload.Currency == "USD" {
        account.Balance += e.Payload.Amount
    } else {
        account.Balance += convertToUSD(e.Payload.Amount, e.Payload.Currency)
    }
}
```

#### 策略 B：Upcaster 模式

**Upcaster**：把老版本事件"升级"为新版本的转换函数。读取时透明升级：

```go
func Upcast(e Event) Event {
    switch e.Type {
    case "MoneyDeposited.v1":
        return Event{
            Type: "MoneyDeposited.v2",
            Payload: map[string]any{
                "amount":   e.Payload["amount"],
                "currency": "USD",  // 默认值
            },
        }
    }
    return e
}

// 加载时
for _, e := range events {
    e = Upcast(e)
    account.Apply(e)
}
```

Upcaster 的优点：业务代码**只处理最新版本**，老事件由 upcaster 自动升级。

#### 策略 C：Weak Schema（JSON）

事件存储用 JSON / Avro / Protobuf，本身支持 forward / backward 兼容：

- 新加字段：旧代码忽略
- 老字段废弃：保留但不读

需要遵守严格的**schema 演化规范**（如 Avro 的 backward compatibility）。

### 5.3 反模式：直接改老事件

```sql
-- 灾难！
UPDATE events SET payload = ... WHERE event_type = 'OldType';
```

事件**不可改**——这是 ES 的核心契约。一旦你改了老事件：

- 历史 replay 结果变了
- 已生成的物化视图与新 replay 不一致
- 审计链断裂

**永远只能"加新事件"或"加 Upcaster"**。

### 5.4 真实演化示例

业务变迁：从"USD 单一货币"到"多币种"。

```
v1 事件: MoneyDeposited { amount: 100 }                   // 隐含 USD
v2 事件: MoneyDeposited { amount: 100, currency: "EUR" }
```

Upcaster：

```go
func upcastMoneyDeposited(e Event) Event {
    if e.Type == "MoneyDeposited.v1" {
        return Event{
            Type:    "MoneyDeposited.v2",
            Payload: map[string]any{
                "amount":   e.Payload["amount"],
                "currency": "USD",
            },
        }
    }
    return e
}
```

旧事件物理保存为 v1，但加载时被 upcaster 升级为 v2，业务代码只处理 v2。

---

## 第六章：物化视图与 Projection

### 6.1 为什么需要物化视图

ES 写入是 append-only 事件流，但查询场景千变万化：

- "按用户查最近 10 笔交易" → 需要"事务表"
- "按日期统计 GMV" → 需要"日 GMV 聚合表"
- "查所有余额 < 0 的账户" → 需要"账户当前状态表"

如果每次查询都 replay 所有事件，性能不可接受。**物化视图（Materialized View / Read Model）** 就是事件流的"针对性投影"。

### 6.2 Projection 工作原理

```
事件流（写入）
   │
   ├─→ Projection 1 ─→ accounts_current（账户当前状态表）
   ├─→ Projection 2 ─→ transactions_view（交易明细表）
   ├─→ Projection 3 ─→ daily_gmv（日 GMV 表）
   └─→ Projection 4 ─→ ElasticSearch 索引（全文搜索）
```

每个 projection 是一个**事件消费器**，订阅事件流，根据事件更新自己的视图表。

### 6.3 一个 Projection 示例

```go
type AccountCurrentProjection struct {
    db *sql.DB
}

func (p *AccountCurrentProjection) Handle(ctx context.Context, e Event) error {
    switch e.Type {
    case "AccountOpened":
        _, err := p.db.ExecContext(ctx, `
            INSERT INTO accounts_current(id, balance, opened_at)
            VALUES ($1, 0, $2)
            ON CONFLICT (id) DO NOTHING`,
            e.AggregateID, e.OccurredAt)
        return err

    case "MoneyDeposited":
        _, err := p.db.ExecContext(ctx, `
            UPDATE accounts_current SET balance = balance + $1, updated_at = $2
            WHERE id = $3`,
            e.Payload.Amount, e.OccurredAt, e.AggregateID)
        return err

    case "MoneyWithdrawn":
        _, err := p.db.ExecContext(ctx, `
            UPDATE accounts_current SET balance = balance - $1, updated_at = $2
            WHERE id = $3`,
            e.Payload.Amount, e.OccurredAt, e.AggregateID)
        return err
    }
    return nil
}
```

### 6.4 Projection 的特性

| 特性 | 说明 |
|---|---|
| **可重建** | 删掉物化视图，从 event 流 replay 即可 |
| **可独立演化** | 业务需要新视图，加个 projection 就行 |
| **可异步** | 物化视图是最终一致 |
| **可分布式** | 不同 projection 可以放不同存储 |

### 6.5 Projection 的最终一致性窗口

事件写入到 projection 表之间有延迟：

```
T0: event 写入 event store
T1: projection 消费 event
T2: projection 写入 view table
```

`T2 - T0` 通常 ms ~ 秒级。这意味着**用户写入后立即查询，可能看不到自己的修改**——这是 CQRS 系统典型的 UX 问题。

应对：

- 写入后强制读 event store 而不是 view（read-your-writes）
- 写入响应中返回新 version，前端轮询直到 view catch up
- UI 层乐观更新（先在前端 patch，再异步对账）

### 6.6 Projection 失败 & 回放

Projection 消费失败时：

- 记录 `last_processed_version`
- 失败时重试
- 实在不行：删表 + 重置 offset + 重新 replay

```sql
CREATE TABLE projection_offsets (
    projection_name VARCHAR(64) PRIMARY KEY,
    last_version    BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ
);
```

---

## 第七章：CQRS —— 命令与查询的责任分离

### 7.1 一句话定义

**CQRS**（Command Query Responsibility Segregation）：**写模型和读模型分离**。

- 写模型（Command）：处理状态变更，强一致，复杂业务规则
- 读模型（Query）：处理查询，可以是非范式化的，针对查询优化

```
        命令路径                          查询路径
   ┌──────────────┐                  ┌──────────────┐
   │   Command    │                  │    Query     │
   │   Handler    │                  │   Handler    │
   └──────┬───────┘                  └──────┬───────┘
          │                                  │
          ▼                                  ▼
   ┌──────────────┐                  ┌──────────────┐
   │ Write Model  │ ── events ──→    │  Read Model  │
   │   (Domain)   │                  │ (Projection) │
   └──────────────┘                  └──────────────┘
```

### 7.2 简单 CQRS（没有 ES）

CQRS 不一定需要 ES：

```
                                  ┌─→ Read DB（只读副本 / 反范式）
Command ─→ Write DB  ─(CDC)─→ ───┤
                                  └─→ ElasticSearch
                                  └─→ Cache
```

例如：电商订单写入 MySQL，CDC 同步到 ElasticSearch（用于搜索）+ Redis（用于按用户查最近订单）。这就是不带 ES 的 CQRS。

### 7.3 命令与查询的不同需求

| 维度 | 命令 | 查询 |
|---|---|---|
| 一致性 | 强一致 | 最终一致 |
| 范式化 | 高度范式化（domain model） | 反范式（针对查询优化） |
| 数据库 | OLTP（PG / MySQL） | OLAP / 搜索引擎 / KV |
| 业务规则 | 复杂（业务核心） | 简单（只是读） |
| QPS | 中-低 | 高 |
| 模型 | 一个聚合 | 多个视图 |

### 7.4 何时 CQRS 是必要的

- 读写比悬殊（读 >> 写）
- 读和写有完全不同的数据模型需求
- 需要多种异构查询（SQL + 全文 + 图）
- 需要独立横向扩展读

### 7.5 CQRS 反模式

- 读写库共用同一张表（"伪 CQRS"）
- 命令端没有领域模型，纯 CRUD（CQRS 失去意义）
- 读模型也保留所有写时校验逻辑（重复劳动）

---

## 第八章：ES 与 CQRS 是独立概念，但常配合

### 8.1 四象限矩阵

| | 用 CQRS | 不用 CQRS |
|---|---|---|
| **用 ES** | ★★★★★ 最常见 | 罕见（ES 几乎一定需要 projection） |
| **不用 ES** | ★★★ 简化 CQRS（CRUD + 投影） | ★ 普通 CRUD |

### 8.2 为什么 ES 几乎一定要配 CQRS

ES 的写入是 append-only 事件流，**没有"当前状态表"可直接 SELECT**。如果不做 projection，每次查询都要 replay，根本扛不住。

所以 ES + CQRS 几乎是绑定的：ES 提供写时审计 / 时间旅行，CQRS 提供读时性能。

### 8.3 为什么 CQRS 不一定要 ES

如果业务只是"读多写少 + 读模型复杂"，用 CRUD + CDC 同步到搜索引擎 / cache 就够了，不需要 ES 的额外复杂度。

### 8.4 决策树

```
是否需要审计 / 时间旅行 / 复杂业务追溯？
   ├─ 是 → 用 ES
   │       │
   │       └─→ 读查询是否复杂？
   │            ├─ 是 → ES + CQRS
   │            └─ 否 → ES（自带 replay 即可）
   │
   └─ 否 → 不用 ES
           │
           └─→ 读写需求是否差异巨大？
                ├─ 是 → CRUD + CQRS
                └─ 否 → 普通 CRUD
```

---

## 第九章：与 Saga 协同

### 9.1 Saga 也是事件驱动

[Saga 模式](./08-精通-分布式事务.md)：长事务用一系列**本地事务 + 补偿事务**组成。

Saga 的"本地事务"完成后通常**发出事件**——这与 ES 天然契合。

### 9.2 Saga + ES 的协同

```
OrderCreated  ──→ Order Saga ──→ ReserveStock
                                    │
                              StockReserved
                                    │
                                    ▼
                                 ChargePayment
                                    │
                              PaymentCharged
                                    │
                                    ▼
                                 CompleteOrder
                                    │
                              OrderCompleted
```

每个事件**既是业务事实，又是 Saga 的下一步触发器**。

实现要点：

- Saga 协调器订阅事件流
- 每个事件到来时决定下一步命令
- 失败事件触发补偿命令

### 9.3 编排式（Orchestration） vs 协作式（Choreography）

| | 编排式（Orchestration） | 协作式（Choreography） |
|---|---|---|
| 协调者 | 中心 Saga Orchestrator | 各服务独立监听事件 |
| 事件 | 命令风格（DoX） | 事实风格（XHappened） |
| ES 配合 | Orchestrator 也用 ES 记录 Saga 状态 | 各聚合自带 ES |

详见 [精通分布式事务 - Saga 章节](./08-精通-分布式事务.md)。

---

## 第十章：何时该用 ES，何时不要用

### 10.1 该用 ES 的信号

| 信号 | 解释 |
|---|---|
| 业务**强审计需求** | 金融、医疗、政府、合规 |
| 业务**强时间旅行需求** | "某日某时的状态？" |
| 业务**变化频繁** | 频繁需要"修复 bug 后 replay" |
| 业务**本身就是事件流** | IoT、行情、行车记录 |
| 团队**有 DDD 沉淀** | 已习惯领域模型与聚合根 |
| 需要**写后多种异构读** | 同时要 SQL、全文、图、cache |

### 10.2 不要用 ES 的信号

| 信号 | 解释 |
|---|---|
| 业务**纯 CRUD**（如 CMS、博客） | 加 ES 是过度设计 |
| 团队**没有 DDD / ES 经验** | 学习曲线 6 个月起 |
| 业务**未来变化少** | ES 的灵活性用不上 |
| 业务**强查询为主**（如报表系统） | 直接 OLAP 更简单 |
| 强一致跨聚合需求多 | ES 不擅长跨聚合一致性 |
| 业务**还没有边界** | 还在 PMF 验证期，模型会大改 |

### 10.3 反模式：把 CRUD 强行套 ES

最常见的失败案例：

```
事件设计：
  AccountUpdated { id: 1, balance: 70 }    ← 这不是事件！
  UserUpdated { id: 1, name: "Alice" }     ← 这也不是事件！
```

这是 "CRUD-flavored ES"——事件名只是 `XUpdated`，把整个新状态塞进事件。**这等于没用 ES**，因为：

- 失去了"为什么变化"的业务语义
- 失去了"哪些字段变了"的精度
- replay 时无法重现业务逻辑

正确的事件：

```
MoneyDeposited { account_id: 1, amount: 100, reason: "salary" }
EmailChanged { user_id: 1, old: "a@x.com", new: "b@x.com", reason: "user_request" }
```

**事件应该反映业务意图（domain intent），而非数据库变更（data mutation）**。

### 10.4 决策清单

回答这 7 个问题，给自己打分（每个"是"+1 分）：

1. 业务**必须**审计每一次状态变化吗？
2. 用户经常问"过去某时刻的状态"吗？
3. 业务规则**经常变化**且需要重算历史吗？
4. 业务本身的核心是"事件"（订单、交易、设备数据）吗？
5. 团队有 DDD / 事件驱动设计经验吗？
6. 同一份数据需要服务于多种异构查询吗？
7. 强一致性可以局限在聚合内（不需要跨聚合）吗？

- 0-2 分：**别用 ES**，CRUD 即可
- 3-4 分：**慎重**，可以局部试点（一两个核心聚合）
- 5-7 分：**全面采用 ES + CQRS**

---

## 第十一章：实战案例 —— 账户余额服务从 CRUD 改造为 ES

### 11.1 业务背景

一家支付公司的账户系统最初是简单 CRUD：

```sql
CREATE TABLE accounts (
    id        BIGINT PRIMARY KEY,
    user_id   BIGINT,
    balance   NUMERIC(18, 2),
    updated_at TIMESTAMPTZ
);
```

问题：

- 合规要求所有变动留痕（CRUD 难做到完整）
- 客服经常被问"我昨天 14:23 的余额"（CRUD 查不出）
- 修复 bug 后无法 replay 重算
- 跨币种、跨账户 BU 调拨场景越来越复杂

决定改造为 ES + CQRS。

### 11.2 事件设计

```
AccountOpened    { account_id, user_id, currency, occurred_at }
MoneyDeposited   { account_id, amount, currency, source, reason, ref_id, occurred_at }
MoneyWithdrawn   { account_id, amount, currency, target, reason, ref_id, occurred_at }
AccountFrozen    { account_id, reason, by_admin_id, occurred_at }
AccountUnfrozen  { account_id, by_admin_id, occurred_at }
TransferOut      { account_id, to_account_id, amount, currency, transfer_id, occurred_at }
TransferIn       { account_id, from_account_id, amount, currency, transfer_id, occurred_at }
```

注意：

- 事件名是**业务意图**（不是 `BalanceUpdated`）
- 每个事件携带充分的上下文（reason、ref_id）
- 跨账户转账拆成两个事件（TransferOut / TransferIn）

### 11.3 事件存储

```sql
CREATE TABLE account_events (
    id           BIGSERIAL PRIMARY KEY,
    account_id   BIGINT NOT NULL,
    event_type   VARCHAR(64) NOT NULL,
    event_version INT NOT NULL DEFAULT 1,
    payload      JSONB NOT NULL,
    metadata     JSONB,
    version      BIGINT NOT NULL,
    occurred_at  TIMESTAMPTZ NOT NULL,
    UNIQUE (account_id, version)
);

CREATE INDEX ON account_events (account_id);
CREATE INDEX ON account_events (event_type, occurred_at);
CREATE INDEX ON account_events ((payload->>'ref_id'));
```

### 11.4 聚合（Account）实现

```go
type Account struct {
    ID       int64
    UserID   int64
    Currency string
    Balance  decimal.Decimal
    Frozen   bool
    Version  int64

    // 待持久化的新事件
    pendingEvents []Event
}

func (a *Account) Deposit(amount decimal.Decimal, source, reason, refID string) error {
    if a.Frozen {
        return ErrAccountFrozen
    }
    if amount.LessThanOrEqual(decimal.Zero) {
        return ErrInvalidAmount
    }
    a.raise(Event{
        Type: "MoneyDeposited",
        Payload: map[string]any{
            "amount":   amount.String(),
            "currency": a.Currency,
            "source":   source,
            "reason":   reason,
            "ref_id":   refID,
        },
    })
    return nil
}

func (a *Account) Withdraw(amount decimal.Decimal, target, reason, refID string) error {
    if a.Frozen {
        return ErrAccountFrozen
    }
    if a.Balance.LessThan(amount) {
        return ErrInsufficientFunds
    }
    a.raise(Event{
        Type: "MoneyWithdrawn",
        Payload: map[string]any{
            "amount":   amount.String(),
            "currency": a.Currency,
            "target":   target,
            "reason":   reason,
            "ref_id":   refID,
        },
    })
    return nil
}

func (a *Account) raise(e Event) {
    a.Apply(e)
    a.pendingEvents = append(a.pendingEvents, e)
}

func (a *Account) Apply(e Event) {
    switch e.Type {
    case "AccountOpened":
        a.UserID = e.Payload["user_id"].(int64)
        a.Currency = e.Payload["currency"].(string)
        a.Balance = decimal.Zero
    case "MoneyDeposited":
        amt, _ := decimal.NewFromString(e.Payload["amount"].(string))
        a.Balance = a.Balance.Add(amt)
    case "MoneyWithdrawn":
        amt, _ := decimal.NewFromString(e.Payload["amount"].(string))
        a.Balance = a.Balance.Sub(amt)
    case "AccountFrozen":
        a.Frozen = true
    case "AccountUnfrozen":
        a.Frozen = false
    }
    a.Version++
}
```

### 11.5 仓储（Repository）

```go
type AccountRepository struct {
    db *sql.DB
}

func (r *AccountRepository) Load(ctx context.Context, id int64) (*Account, error) {
    // 1. 尝试加载最近 snapshot
    var snap struct {
        Version int64
        State   []byte
    }
    err := r.db.QueryRowContext(ctx,
        `SELECT version, state FROM account_snapshots
         WHERE account_id=$1 ORDER BY version DESC LIMIT 1`, id).
        Scan(&snap.Version, &snap.State)

    var account *Account
    var fromVersion int64
    if err == nil {
        account = deserializeAccount(snap.State)
        fromVersion = snap.Version + 1
    } else {
        account = &Account{ID: id}
        fromVersion = 1
    }

    // 2. 加载新事件
    rows, _ := r.db.QueryContext(ctx,
        `SELECT event_type, payload, version FROM account_events
         WHERE account_id=$1 AND version >= $2 ORDER BY version`, id, fromVersion)
    defer rows.Close()

    for rows.Next() {
        var e Event
        rows.Scan(&e.Type, &e.Payload, &e.Version)
        account.Apply(e)
    }
    return account, nil
}

func (r *AccountRepository) Save(ctx context.Context, a *Account) error {
    tx, _ := r.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    expectedVersion := a.Version - int64(len(a.pendingEvents))
    for _, e := range a.pendingEvents {
        expectedVersion++
        _, err := tx.ExecContext(ctx, `
            INSERT INTO account_events(account_id, event_type, payload, version, occurred_at)
            VALUES ($1, $2, $3, $4, now())`,
            a.ID, e.Type, e.Payload, expectedVersion)
        if err != nil {
            if isUniqueViolation(err) {
                return ErrConcurrentModification    // 乐观锁
            }
            return err
        }
    }
    return tx.Commit()
}
```

### 11.6 命令处理

```go
type DepositCommand struct {
    AccountID int64
    Amount    decimal.Decimal
    Source    string
    Reason    string
    RefID     string    // 幂等用
}

func (h *AccountHandler) HandleDeposit(ctx context.Context, cmd DepositCommand) error {
    // 1. 幂等检查（详见上一章）
    if existing := h.idempotency.Get(cmd.RefID); existing != nil {
        return nil
    }

    // 2. 加载聚合
    account, err := h.repo.Load(ctx, cmd.AccountID)
    if err != nil {
        return err
    }

    // 3. 调用聚合方法（业务规则在此校验）
    if err := account.Deposit(cmd.Amount, cmd.Source, cmd.Reason, cmd.RefID); err != nil {
        return err
    }

    // 4. 保存事件
    if err := h.repo.Save(ctx, account); err != nil {
        return err
    }

    // 5. 标记幂等
    h.idempotency.Set(cmd.RefID, account.Version)
    return nil
}
```

### 11.7 物化视图（账户当前余额表）

```sql
CREATE TABLE accounts_view (
    id             BIGINT PRIMARY KEY,
    user_id        BIGINT NOT NULL,
    currency       VARCHAR(8) NOT NULL,
    balance        NUMERIC(18, 2) NOT NULL,
    frozen         BOOLEAN NOT NULL DEFAULT false,
    last_version   BIGINT NOT NULL,
    last_event_at  TIMESTAMPTZ NOT NULL
);
```

Projection 消费事件：

```go
func (p *AccountViewProjection) Handle(ctx context.Context, e Event) error {
    switch e.Type {
    case "AccountOpened":
        _, err := p.db.ExecContext(ctx, `
            INSERT INTO accounts_view(id, user_id, currency, balance, last_version, last_event_at)
            VALUES ($1, $2, $3, 0, $4, $5)
            ON CONFLICT (id) DO NOTHING`,
            e.AggregateID, e.Payload["user_id"], e.Payload["currency"],
            e.Version, e.OccurredAt)
        return err
    case "MoneyDeposited":
        _, err := p.db.ExecContext(ctx, `
            UPDATE accounts_view
            SET balance = balance + $1, last_version = $2, last_event_at = $3
            WHERE id = $4 AND last_version < $2`,
            e.Payload["amount"], e.Version, e.OccurredAt, e.AggregateID)
        return err
    case "MoneyWithdrawn":
        _, err := p.db.ExecContext(ctx, `
            UPDATE accounts_view
            SET balance = balance - $1, last_version = $2, last_event_at = $3
            WHERE id = $4 AND last_version < $2`,
            e.Payload["amount"], e.Version, e.OccurredAt, e.AggregateID)
        return err
    }
    return nil
}
```

注意 `WHERE last_version < $2`——**保证 projection 幂等**（即使消息被重投）。

### 11.8 时间旅行查询

```sql
-- 查询账户在 2026-05-25 14:00 时刻的余额
WITH events AS (
    SELECT event_type, payload, version
    FROM account_events
    WHERE account_id = 1
      AND occurred_at <= '2026-05-25 14:00:00'
    ORDER BY version
)
SELECT * FROM events;
```

在应用层 replay 即可：

```go
func (r *Repo) StateAt(ctx context.Context, id int64, t time.Time) (*Account, error) {
    rows, _ := r.db.QueryContext(ctx,
        `SELECT event_type, payload, version FROM account_events
         WHERE account_id=$1 AND occurred_at<=$2 ORDER BY version`, id, t)
    defer rows.Close()

    account := &Account{ID: id}
    for rows.Next() {
        var e Event
        rows.Scan(&e.Type, &e.Payload, &e.Version)
        account.Apply(e)
    }
    return account, nil
}
```

### 11.9 改造收益总结

| 项 | 之前（CRUD） | 之后（ES + CQRS） |
|---|---|---|
| 审计 | 单独表，不完整 | 事件即审计，100% 完整 |
| 时间旅行 | 不可能 | 简单 |
| 修复 bug 后重算 | 不可能 | replay 即可 |
| 客服自助查询历史 | 需 DBA | 业务系统直接查 |
| 跨币种 / 跨账户调拨 | 复杂逻辑 | 拆事件清晰 |
| 写入复杂度 | 低 | 中（学习曲线） |
| 团队认知成本 | 低 | 高（培训 3 个月） |
| 整体维护 | 中 | 中-低（结构更清晰） |

---

## 第十二章：反模式

### 12.1 反模式 1：把 CRUD 强行套 ES

```
事件：UserUpdated { full_user_object }
```

事件没有业务意图，本质上还是 CRUD。**症状**：所有事件都是 `XUpdated / XCreated / XDeleted`。

**修复**：事件应该回答"用户想做什么"——`EmailChanged`、`UserPromoted`、`AccountClosed`。

### 12.2 反模式 2：跨聚合的强一致事件

```
事件：OrderPlacedAndStockReserved { ... }
```

把两个聚合的变化塞到一个事件，违反了"一个事件属于一个聚合"的原则。

**修复**：拆成两个事件 + Saga 协同：

```
OrderPlaced  →  StockReservationRequested  →  StockReserved  →  OrderConfirmed
```

### 12.3 反模式 3：滥用 CQRS

读模型有 50 张表，对应 50 种查询。维护成本爆炸。

**修复**：

- 只为**高频**或**模型差异大**的查询单独建 read model
- 简单 SQL 直接查写库即可（少量 OLTP 查询）

### 12.4 反模式 4：事件设计成"通知"而非"事实"

```
事件：UserRegistrationCompleted   ← "完成了" 是过去式
事件：UserShouldReceiveEmail      ← "应该" 是命令，不是事实
```

ES 中事件**必须是过去式 + 业务事实**。"Should" 之类的命令应该走命令通道，不是事件。

### 12.5 反模式 5：projection 表当成"另一个事实来源"

业务代码同时写事件 + 写 projection 表（双写）。这会导致两者不一致。

**修复**：

- projection 表**只能从事件流生成**
- 任何对 projection 表的"修复"都要追溯到事件

### 12.6 反模式 6：用事件做"消息通信"

跨聚合调用直接 emit 事件并消费，没有 Saga 协调。

**修复**：

- 跨聚合协作用 **Command + Saga**
- 事件**只反映本聚合内的事实**，不用于跨聚合调度

### 12.7 反模式 7：snapshot 当"修正历史"的工具

bug 修复后，直接在 snapshot 里改个数字，让查询变正确。

**修复**：snapshot 是缓存，永远不要"绕过事件流"修改。要么追加修正事件，要么 replay 重建 snapshot。

---

## 第十三章：生产实践

### 13.1 一定要做的

- 事件设计先于代码（事件风暴 Event Storming 工作坊）
- 所有事件**带 version + occurred_at**
- 事件存储用**append-only 表 + 唯一约束**
- 写入聚合时用**乐观锁**
- 每个聚合**最大事件数控制在 10k 以内**（超过分析是否聚合粒度过大）
- snapshot 频率让 replay < 100ms
- 物化视图保留**last_version** 字段保证幂等
- 事件 schema 用 Avro / Protobuf / JSON Schema 强约束

### 13.2 一定要监控的

| 指标 | 含义 |
|---|---|
| `events_per_aggregate` | 单聚合事件数（增长警报） |
| `replay_latency` | 加载聚合耗时 |
| `projection_lag` | projection 落后 event store 的时间 |
| `concurrent_modification_rate` | 乐观锁冲突频率 |
| `snapshot_hit_rate` | snapshot 命中率 |
| `event_store_size_growth` | 事件存储增速 |

### 13.3 一定要测试的

- 单元测试：聚合的业务规则
- 事件 replay 测试：同一组事件多次 replay 结果相同
- Projection 幂等测试：重复消费同一事件结果一致
- Schema 演化测试：老事件 + Upcaster + 新代码能正确 replay
- 压测：高并发下乐观锁冲突的退避策略

### 13.4 团队建设

- 全员培训 ES 心智（至少 1 周）
- 准备"反模式清单"作为 code review checklist
- 引入 DDD / 事件风暴工作坊
- 关键聚合代码 review 由 2 人共同把关

---

## 第十四章：陷阱清单

| 序号 | 陷阱 | 修复 |
|---|---|---|
| 1 | 事件用 `XUpdated` 风格 | 改为业务意图事件 |
| 2 | 一个事件跨多个聚合 | 拆成多个事件 + Saga |
| 3 | 直接修改老事件 | 永远只能追加 + Upcaster |
| 4 | snapshot 当真相来源 | snapshot 仅缓存，可重建 |
| 5 | projection 双写 | 只从事件流写 projection |
| 6 | 没有版本号导致并发覆盖 | 加 version + 乐观锁 |
| 7 | replay 慢但不做 snapshot | 定期 snapshot |
| 8 | 单聚合事件数百万 | 重新审视聚合粒度 |
| 9 | 事件无 occurred_at | 时间旅行不可能 |
| 10 | projection 无 last_version | 重复消费导致错误 |
| 11 | 强行用 ES 做 CRUD 业务 | 退回 CRUD |
| 12 | 跨聚合事件传递业务逻辑 | 用 Saga |
| 13 | schema 演化没做兼容 | 引入 Upcaster |
| 14 | 没有事件 schema 校验 | 加 schema registry |
| 15 | 测试只测"当前状态" | 加 replay 测试 |

---

## 第十五章：2026 现状

### 15.1 工具生态

- **EventStoreDB**：23.10 GA 起支持 Multi-stream Transaction
- **Kafka**：3.7 引入 KIP-848（消费者组协议升级）
- **Axon Framework**（Java）：6.0 集成 Spring Boot 3、原生 GraalVM
- **EventStore Go SDK**：2.5 起支持 OpenTelemetry
- **Marten**（.NET，基于 Postgres）：7.0 默认支持异步 projection
- **Eventuous**（.NET）：流行的轻量 ES 框架
- **Equinox**（F#）：金融场景广泛使用

### 15.2 数据库支持

- **PostgreSQL 17**：分区表 + JSONB GIN 索引让基于 PG 的 event store 在 10TB 规模下依然可行
- **CockroachDB / TiDB**：分布式事务支持让"事件 + outbox"同事务变得简单
- **ClickHouse**：作为 projection 目标越来越流行（高基数聚合）

### 15.3 行业实践

- **金融 / 支付**：ES + CQRS 几乎是标配（合规驱动）
- **电商 / SaaS**：选择性应用（核心订单 / 库存用 ES，其它 CRUD）
- **IoT / 物联网**：天然 ES（设备事件流）
- **游戏行业**：玩家行为流、战斗回放，ES 优势明显
- **AI Agent 平台**：Anthropic / OpenAI 等 Agent 编排平台广泛采用 ES 记录会话 / Agent 步骤（2025 后兴起）

### 15.4 与 AI 的结合

2026 年的新趋势：用 LLM 分析事件流：

- 异常检测：LLM 读事件流找出"反常模式"
- 客服自助：用户问"为什么我的余额是 X"，LLM 直接读事件流回答
- 业务洞察：LLM 在事件流上做"自然语言查询"

这让 ES 的"完整历史"价值进一步放大——历史越完整，AI 能挖掘的越多。

---

## 第十六章：练习题

### 题 1（基础）：判断 ES 适用性

下面哪些场景适合 ES？

```
A. 一个博客评论系统
B. 银行转账系统
C. CMS 内容管理
D. 智能电表的能耗采集
E. 简单的 Todo 应用
F. 期货交易撮合
G. 游戏战斗回放
```

**答案要点**：B / D / F / G 适合（强审计 / 事件本质 / 时间旅行）；A / C / E 不适合（普通 CRUD）。

### 题 2（设计）：事件命名

下面哪些是好的事件名？为什么？

```
A. UserUpdated
B. EmailChanged
C. OrderProcessed
D. OrderPaid
E. AccountStateChangedToActive
F. AccountActivated
G. StockDecreased
H. StockReservedForOrder
```

**答案要点**：B / D / F / H 是好的（业务意图明确）；A / C / E / G 是坏的（CRUD 风格 / 太通用）。

### 题 3（设计）：账户冻结

设计"管理员冻结账户"的事件与状态机。

**答案要点**：

- 事件：`AccountFrozen { by_admin_id, reason, occurred_at }`
- 聚合内 `Frozen` 字段
- 后续 `Deposit / Withdraw` 命令检查 `Frozen` 状态
- 解冻：`AccountUnfrozen { by_admin_id, occurred_at }`

### 题 4（演化）：Schema 演化

事件 v1：`MoneyDeposited { amount }`（默认 USD）；现在要支持多币种。如何演化？

**答案要点**：

- 引入 v2：`MoneyDeposited.v2 { amount, currency }`
- 加 Upcaster：v1 → v2，currency = "USD"
- 老事件物理不变
- 业务代码只处理 v2

### 题 5（性能）：聚合事件数膨胀

某聚合事件已超过 100 万条，replay 5 秒。如何优化？

**答案要点**：

- 加 snapshot（每 1000 事件一个）
- 检查聚合粒度是否过粗（要不要按时间分聚合？）
- 不常用的聚合用懒加载
- replay 引入并行（不同聚合并行）

### 题 6（实战）：Projection 失败重建

某 projection 表数据损坏，要重建。流程是什么？

**答案要点**：

- 停止 projection consumer
- TRUNCATE projection 表
- 重置 offset 为 0
- 启动 consumer 从头 replay 全部事件
- 校验完成后切流量

### 题 7（架构）：CRUD vs ES vs CQRS

请画出三种系统的数据流，标出读路径和写路径。

**答案要点**：

- 纯 CRUD：单库，读写同一张表
- 纯 CQRS（无 ES）：写主库，CDC 同步到读副本 / 搜索引擎
- ES + CQRS：写事件流，projection 消费事件生成读模型

### 题 8（陷阱）：跨聚合强一致

业务要求"下单时同步扣库存"，能不能用一个事件 `OrderPlacedAndStockReserved` 解决？

**答案要点**：

- 不能。事件属于一个聚合
- 应该：`OrderPlaced` + Saga + `StockReservationRequested` + `StockReserved` + `OrderConfirmed`
- 跨聚合的强一致只能在业务上接受最终一致
- 或者把 Order + Stock 合并为一个聚合（但要考虑性能）

---

## 第十七章：延伸阅读

- **书**：Greg Young《Versioning in an Event Sourced System》（电子书，免费）
- **书**：Vaughn Vernon《Implementing Domain-Driven Design》
- **书**：Alberto Brandolini《Introducing EventStorming》
- **书**：Martin Kleppmann《Designing Data-Intensive Applications》第 11 章
- **网站**：[eventstore.com/blog](https://www.eventstore.com/blog) 大量 ES 实战
- **视频**：Greg Young 在 GOTO Conference 上历年的 ES talk
- **博客**：Martin Fowler《Event Sourcing》《CQRS》两篇经典文
- **代码库**：`github.com/EventStore/EventStore-Client-Go`、`github.com/looplab/eventhorizon`
- **关联章节**：[精通分布式事务](./08-精通-分布式事务.md)、[精通异步消息与事件驱动](./07-精通-异步消息与事件驱动.md)、[精通服务拆分与 DDD](./02-精通-服务拆分与-DDD.md)、[精通幂等与去重](./09-精通-幂等与去重.md)

---

> **本章金句**：**"状态会撒谎，事件不会"**。CRUD 让你只看到"现在"，ES 让你拥有"全部"。但 ES 也不是银弹——选错战场，它就会成为团队认知地狱。**Don't use ES for the sake of ES. Use ES when history matters.**
