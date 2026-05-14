# 精通 Redis 客户端与连接管理：RESP3、连接池、Cluster-aware、客户端缓存

> 关联章节：[07 Cluster](./05-精通-Redis-Cluster.md)、[08 性能模型](./08-精通-Redis-性能模型.md)、[10 生产实践](./10-精通-Redis-生产实践.md)

---

## 引言：服务端写得再好，客户端用错也白搭

工程师调研 Redis 通常停在服务端：数据结构、持久化、Cluster。**但生产事故里有大概一半根因在客户端**：

- 没用连接池，每次 new 连接，TCP 三次握手 + AUTH 把 Redis 当成 HTTP 用
- 用了连接池但 size 配置错——8 条连接撑 10k QPS，全在排队
- Cluster 客户端 slot 缓存没刷新，MOVED 重定向风暴
- 命令超时配 200ms，结果业务逻辑里串了 5 个 Redis 调用，整体超时 1s
- TLS 握手没复用 session，每次连接 100ms+ 开销
- Lua 脚本每次都发全文不用 EVALSHA，把带宽吃满
- Pub/Sub 长连接断了不重连
- RESP3 升级了但没开 push 通知，丢失客户端缓存失效

这一章把客户端的所有关键点过一遍。读完之后你应该能：

- 解释 RESP2 与 RESP3 的差别，知道何时升级
- 选对 Java / Go / Python / Node 的客户端
- 配出"刚好"的连接池大小（不是越大越好）
- 理解 Cluster 客户端的 slot map 缓存 + MOVED/ASK 处理
- 用 Client Tracking 做服务端推送式客户端缓存
- 排查"连接突然全部 broken"类故障

---

## 第一章：RESP 协议

### 1.1 RESP2：80% 客户端今天还在用的协议

RESP（REdis Serialization Protocol）是 Redis 自定义的二进制安全文本协议，1.2 引入，2.0 起稳定，叫 **RESP2**。

5 种类型，每行 `\r\n` 结尾：

```
+OK\r\n                       Simple String（成功响应）
-ERR wrong number...\r\n      Error
:1000\r\n                     Integer
$5\r\nhello\r\n               Bulk String（带长度的字符串，可含二进制）
*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n  Array（数组）
```

客户端发送命令统一用 **Array of Bulk String**：

```
SET name redis
↓ 编码为
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nredis\r\n
```

抓包看 RESP 很直观：

```bash
redis-cli -h 127.0.0.1 -p 6379 --no-raw
> SET foo bar
"OK"
# tcpdump 抓出来：*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
```

### 1.2 RESP2 的痛点

| 痛点 | 表现 |
|---|---|
| 类型穷困 | 只有 5 种，bool / float / map / set 全用字符串或数组凑 |
| 客户端要"猜" | `HGETALL` 返回 `["k1","v1","k2","v2"]` 数组，客户端自己拼 map |
| 错误信息无结构 | 全是 `-ERR ...` 字符串，没有错误码 |
| 不能在请求/响应外推消息 | Pub/Sub 与命令响应共用一个连接时只能靠类型 hack |

### 1.3 RESP3（Redis 6.0 引入，默认仍 RESP2）

RESP3 增加了多种类型：

```
,3.14\r\n                Double
#t\r\n / #f\r\n          Boolean
_\r\n                    Null（区别于空 Bulk String）
=15\r\ntxt:Some text\r\n Verbatim String（带子类型）
(1234567890\r\n          Big Number
%2\r\n$3\r\nk1\r\n$3\r\nv1\r\n$3\r\nk2\r\n$3\r\nv2\r\n  Map（HGETALL 直接返回 map）
~3\r\n$1\r\na\r\n$1\r\nb\r\n$1\r\nc\r\n   Set
|2\r\n...                Attribute（响应附带元数据，如缓存版本）
>4\r\n...                Push（服务端主动推送）
```

**RESP3 的两个杀手特性**：

1. **`>` Push 类型**：服务端主动推送，跟正常响应区分开 → 让"客户端缓存失效通知"在同一个连接上推送，不需要单独 Pub/Sub 连接
2. **`%` Map 类型**：`HGETALL` / `CONFIG GET` 等返回 Map，客户端不用拼

### 1.4 协议握手 HELLO

```
HELLO 3 [AUTH user pass] [SETNAME conn-1]
```

- 服务端返回 server 信息（version、modules、role 等）
- 协商协议版本（2 或 3）
- 顺便完成认证 + setname，比 `AUTH` + `CLIENT SETNAME` 省一个 RTT

老客户端连旧服务器：直接 `AUTH` + 自动走 RESP2。
新客户端连新服务器：`HELLO 3` → 走 RESP3。

```bash
redis-cli -3                  # 强制 RESP3
> CLIENT INFO
id=10 addr=... resp=3 ...     # 看到 resp=3
```

### 1.5 升级到 RESP3 要小心什么

- **响应类型变了**：`HGETALL` 在 RESP2 返回 array，在 RESP3 返回 map。客户端代码升级要兼容（多数 SDK 自动做了）
- **Pub/Sub 不再独占连接**：RESP3 用 push 类型推到主连接，但要客户端支持 push handler
- **服务端 / 客户端版本要够新**：Redis 6+ 服务端、客户端 SDK 要明确支持 RESP3
- **某些工具仅支持 RESP2**：MONITOR / 一些老 proxy

---

## 第二章：主流客户端选型

### 2.1 Java 三选一

| 客户端 | 适用 | 特性 |
|---|---|---|
| **Jedis** | 简单同步场景 | 阻塞 API，**每个线程一条连接**（用连接池），轻量 |
| **Lettuce** | 主流推荐 | 基于 Netty 异步、**线程安全的共享连接**、原生 reactive |
| **Redisson** | 需要分布式锁 / 集合 | 把 Redis 包成 Java 数据结构（RMap / RLock / RSemaphore），重量级 |

**Jedis 致命点**：单连接非线程安全，多线程必须用连接池（`JedisPool`）。一个 Web 应用 200 并发请求 → 至少 200 条连接。

**Lettuce 优势**：单连接可被多线程共享（**Pipeline / 多路复用** —— 多个线程的命令交错写入一条 socket，由响应顺序匹配）。Spring Boot 默认。

**Redisson 注意**：每个分布式对象（RLock 等）都隐含订阅频道、心跳、看门狗线程，复杂度高，**不适合做轻量 KV**。

### 2.2 Go：`go-redis/redis`（一统江湖）

```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{
    Addr:         "localhost:6379",
    PoolSize:     20,
    MinIdleConns: 5,
    DialTimeout:  3 * time.Second,
    ReadTimeout:  500 * time.Millisecond,
    WriteTimeout: 500 * time.Millisecond,
})
```

历史上还有 `redigo`（更底层、手写命令），但 `go-redis` 已是事实标准：

- Cluster / Sentinel / 单节点统一 API
- 自动 RESP3 协商
- 连接池内置
- pipeline / tx / pub-sub 一应俱全

### 2.3 Python：`redis-py`（官方维护）

```python
import redis

r = redis.Redis(
    host='localhost', port=6379,
    decode_responses=True,
    socket_timeout=0.5,
    socket_connect_timeout=2.0,
    max_connections=20,
)

# 异步版本
import redis.asyncio as aioredis
r = aioredis.Redis(...)
```

`aioredis` 老仓库已合并进 `redis-py`，用 `redis.asyncio` 即可。

### 2.4 Node.js：`ioredis` vs `node-redis`

| | ioredis | node-redis |
|---|---|---|
| 维护 | Luin 个人 + 社区 | 官方 Redis Inc. |
| Cluster 支持 | 老牌可靠 | 4.0 起 |
| Pipeline | 流畅 | 流畅 |
| 推荐 | 现存项目 | 新项目 |

两者都成熟，新项目偏向官方维护的 `node-redis`。

### 2.5 Rust / C# / Ruby

- Rust：`redis-rs`（同步） / `fred` / `deadpool-redis`
- C#：`StackExchange.Redis`（事实标准，跟 Lettuce 类似的多路复用）
- Ruby：`redis-rb` 官方

---

## 第三章：连接池

### 3.1 为什么需要连接池

每次 `connect()` 包含：

1. TCP 三次握手（同机房 0.1ms，跨机房 1-10ms）
2. TLS 握手（如启用）3-4 个 RTT
3. `AUTH` 或 `HELLO`（1 RTT + 服务端校验）
4. `SELECT db`（如非 0 库）

**没有池**：每次请求 5-20ms 启动开销。
**有池**：复用连接，启动开销摊到首次，之后只剩 1 RTT（命令本身）。

### 3.2 池大小的科学

常见误区："连接越多越好"。实际：

- 服务端 Redis **单线程串行处理**——你给它 1000 连接，它仍一条条处理
- 连接太多的问题：
  - 服务端 `maxclients`（默认 10000）逼近上限
  - 每条连接占主线程一份 fd + buffer，过多会增加 epoll 开销
  - 客户端 GC / 上下文切换增加

**合理池大小估算公式**：

```
pool_size = (并发请求数 × 单请求 Redis 调用次数 × 单调用耗时) / 单连接每秒可承担次数
```

更工程的简化版：

```
pool_size = min(并发请求数, CPU 核数 × 4 ~ 8)
```

经验值：

| 场景 | 池大小 |
|---|---|
| Web 应用，单 pod | 8-20 |
| 高并发 RPC | 30-50 |
| 离线任务 | 5-10 |

**配错的代价**：池满 → 后续请求阻塞等连接 → 业务超时雪崩。**Lettuce 用单连接多路复用，池大小 1-4 即可**。

### 3.3 连接生命周期

```
[idle pool] ──borrow→ [in use] ──return→ [idle pool]
                          ↓
                       [broken]
                          ↓
                       [discard]
```

- **borrow**：业务从池里拿一条
- **return**：用完归还，**不是关闭**
- **broken**：检测到连接异常（read/write 失败、超时） → 关闭并移出池

### 3.4 健康检查：心跳与 min idle

```
testOnBorrow / testWhileIdle / minIdleConnections
```

- **testOnBorrow**：借连接前 `PING` 一次。**生产慎开**：每次借都加 1 RTT，业务响应变慢
- **testWhileIdle**：后台线程定期对 idle 连接 PING。**推荐**：开销摊到后台
- **minIdleConnections**：维持最低空闲数，避免低峰回收完，高峰再建

Jedis 配置例：

```java
JedisPoolConfig cfg = new JedisPoolConfig();
cfg.setMaxTotal(20);
cfg.setMaxIdle(10);
cfg.setMinIdle(5);
cfg.setTestWhileIdle(true);
cfg.setTimeBetweenEvictionRunsMillis(30_000); // 30s 巡检一次
cfg.setMinEvictableIdleTimeMillis(60_000);
cfg.setBlockWhenExhausted(true);              // 池满时阻塞而非报错
cfg.setMaxWaitMillis(2_000);                  // 最多等 2s
```

### 3.5 池满怎么办

```
JedisException: Could not get a resource from the pool
```

3 个根因：

1. **业务 QPS 真高**：扩池 + 排查慢命令
2. **某个慢命令卡住所有连接**：`SLOWLOG` 看
3. **连接泄漏**：借了不归还（return 漏在异常路径）

第 3 个最常见。**Java try-with-resources**：

```java
try (Jedis j = pool.getResource()) {
    j.set("k", "v");
}  // 自动归还
```

Go 的 `go-redis` 不需要手动 borrow/return，框架代办。Python 用 `with r.pipeline() as p:` 上下文管理。

### 3.6 长连接陷阱：NAT 超时

云上常见：客户端经过 NAT/SLB 到 Redis，NAT 设备会**清理空闲 > N 分钟的连接**（AWS NLB 默认 350 秒，阿里云 SLB 默认 900 秒）。

症状：

- 业务低峰期后第一波请求集体 `connection reset`
- 客户端日志看到 `RST` 或 `broken pipe`

解决：

1. 客户端 keepalive：TCP_KEEPALIVE 设小于 NAT 超时（如 300s）
2. 服务端 `tcp-keepalive 60`（默认 300）
3. 客户端定期 PING（应用层心跳）
4. 池配置 testWhileIdle + minEvictableIdleTime < NAT 超时

```
# redis.conf
tcp-keepalive 60
```

---

## 第四章：Cluster 客户端

### 4.1 客户端为什么必须 Cluster-aware

非 Cluster：客户端不需要知道分片，连一个 endpoint 就行。
Cluster：每个 key 哈希到 0-16383 中一个 slot，slot 分布在不同节点。如果客户端不知道 slot 分布：

```
client → node A: SET key1 v   ← key1 的 slot 在 node B
node A: -MOVED 5474 node-B-ip:6379
client → node B: SET key1 v   ← 重试
```

**每次都 MOVED 重定向 → 多 1 RTT**。所以客户端要缓存 slot 表。

### 4.2 Slot 表的获取与刷新

启动时：

```
CLUSTER SHARDS         # Redis 7+ 返回 shard 拓扑（master + replicas）
CLUSTER SLOTS          # 老命令，返回 slot 范围 → 节点
CLUSTER NODES          # 详细但需要解析
```

客户端解析后建本地 map：`slot → master node`。

主动刷新场景：

1. 收到 MOVED：可以只更新该 slot，也可全表刷新
2. 收到 ASK（迁移中）：临时单次重定向，**不缓存**
3. 拓扑变化（节点上下线）：通过 gossip 后所有节点都知道；客户端可定期或被动刷新

**陷阱**：刷新频率过高 → 给所有 master 加 RPS；过低 → MOVED 风暴。go-redis / Lettuce 默认在 MOVED 时触发刷新，间隔有最小限制。

### 4.3 MOVED vs ASK 的处理差异

| 错误 | 含义 | 客户端行为 |
|---|---|---|
| `-MOVED 5474 1.2.3.4:6379` | slot 5474 已经在 1.2.3.4，**永久**改了 | 更新本地 slot 表 + 重试 |
| `-ASK 5474 1.2.3.4:6379` | slot 5474 正在迁移，本次去 1.2.3.4，**临时** | **不**更新本地表；连过去先发 `ASKING`，再发原命令 |

`ASKING` 是单次许可，告诉目标节点"这个命令属于正在导入的 slot，请处理"。

错误处理示例（go-redis 自动做）：

```go
err := rdb.Set(ctx, "key", "v", 0).Err()
// 框架收到 MOVED/ASK 后自动重定向 + 重试，业务无感
```

### 4.4 跨 slot 操作的限制

```
MGET k1 k2 k3       # 如果 k1 k2 k3 不在同 slot → CROSSSLOT 错误
```

解决：

1. **hash tag**：用 `{tag}` 强制同 slot
   ```
   user:{1001}:profile, user:{1001}:settings   ← 都在 hash("1001") % 16384
   ```
2. **客户端拆分**：客户端按 slot 分组，分别发到不同节点
3. **改用 pipeline / 多次调用**：放弃原子性

Cluster 下的 Lua 脚本同样要求脚本里所有 key 在同一 slot。

### 4.5 读副本

默认 Cluster 客户端只读写 master。副本读：

```
READONLY              # 在副本连接上发，告诉副本"允许我读副本"
GET key               # 现在可以从副本读
```

主流客户端有 `readonly` / `readReplicas` 配置：

```go
rdb := redis.NewClusterClient(&redis.ClusterOptions{
    Addrs:      []string{":7000", ":7001", ":7002"},
    RouteRandomly: true,    // 命令随机路由到 master 或 replica（仅读命令）
    ReadOnly:   true,       // 自动 READONLY
})
```

注意：副本是**异步复制**，可能读到旧数据。强一致读必须走 master。

### 4.6 Cluster 客户端的 Pipeline

Cluster Pipeline 比单节点复杂——客户端要按 key 分组发到不同节点：

```
client.Pipeline:
  GET k1   ← slot 1234 → node A
  GET k2   ← slot 5678 → node B
  GET k3   ← slot 1234 → node A
```

客户端逻辑：

1. 按 slot 拆请求 → 3 个分组到 A，1 个到 B
2. 并行发往两个节点
3. 把响应按原始顺序拼回

主流 SDK 都已实现，**不要自己手撸**。

---

## 第五章：Pipeline 与 Tx 在客户端的实现

### 5.1 Pipeline = 批量发送 + 批量接收

```go
pipe := rdb.Pipeline()
pipe.Set(ctx, "k1", "v1", 0)
pipe.Set(ctx, "k2", "v2", 0)
pipe.Incr(ctx, "counter")
cmders, err := pipe.Exec(ctx)
```

底层：

```
TCP send: *3\r\n$3\r\nSET\r\n$2\r\nk1\r\n$2\r\nv1\r\n
          *3\r\n$3\r\nSET\r\n$2\r\nk2\r\n$2\r\nv2\r\n
          *2\r\n$4\r\nINCR\r\n$7\r\ncounter\r\n   ← 一次性发完
TCP recv: +OK\r\n+OK\r\n:1\r\n                    ← 一次性收完
```

**3 个命令 1 个 RTT**，比串行 3 RTT 快 3 倍。

### 5.2 Pipeline ≠ 事务

Pipeline 是客户端层的优化，**不保证原子性**：

- 命令在 Redis 服务端可能被别的客户端命令穿插
- 一个命令报错后续仍执行

要原子性用 `MULTI / EXEC`（详见 R06）。

### 5.3 客户端层 Pipeline 实现

**同步客户端**：客户端先把命令缓到本地 buffer，`Exec()` 时一次性 flush。

**异步客户端（Lettuce / Node ioredis）**：**自动 pipeline** —— 多个并发请求被自动合并到一次 socket flush 里，无需手动 `pipeline()`。这是它们性能极高的原因。

### 5.4 Pipeline 大小有上限

```go
// 一次性 10 万命令
pipe := rdb.Pipeline()
for i := 0; i < 100_000; i++ {
    pipe.Set(ctx, fmt.Sprintf("k%d", i), "v", 0)
}
pipe.Exec(ctx)
```

风险：

- 客户端内存堆积响应等待
- 服务端 output buffer 撑爆（`client-output-buffer-limit`）
- 单次响应到几十 MB → 网卡瞬时打满

**经验**：单次 pipeline 控制在 500-2000 命令，超大批量分多次。

---

## 第六章：客户端缓存（Tracking）

### 6.1 问题：每次都查 Redis 也慢

业务 → Redis：1ms（局域网）。要把这 1ms 压到 0.01ms，**应用本地内存缓存** Redis 结果是常见做法。但本地缓存怎么知道 Redis 里的值变了？

传统三种方案：

1. **TTL**：本地缓存 5 秒过期 → 5 秒延迟可接受？
2. **业务双写**：改值时同时清本地缓存 → 多实例怎么同步？
3. **MQ 广播**：改值 → 发 Kafka → 各应用清缓存 → 重型

### 6.2 Redis 6.0 的 Tracking 模式

服务端记录"哪个 client 读了哪个 key"。当 key 改变，主动 push 通知给这些 client。

```
CLIENT TRACKING ON                      # 开启
GET user:1001                            # 服务端记下：connA 读了 user:1001
                                         # ... 别人 SET user:1001 newval
                                         # 服务端 push: invalidate user:1001
```

两种模式：

| 模式 | 服务端开销 | 网络开销 | 适用 |
|---|---|---|---|
| **default**（精确） | 维护"key → client 列表"，**内存高** | 准确通知 | 缓存 key 数少 |
| **broadcasting**（前缀广播） | 维护"前缀 → client 列表"，**内存低** | 该前缀下任何 key 改都通知该 client | 缓存 key 多但分前缀 |

```
CLIENT TRACKING ON BCAST PREFIX user: PREFIX product:
```

### 6.3 推送通道：RESP3 push vs RESP2 redirect

**RESP3** 下推送直接走主连接的 push 帧（`>`）：

```
> CLIENT TRACKING ON
< +OK
... 其他客户端改了 key ...
< >2\r\n$11\r\ninvalidate\r\n*1\r\n$8\r\nuser:1001\r\n   ← 主动推送
```

**RESP2** 下不能在主连接 push，所以 Redis 让客户端**额外开一条专门接通知的 Pub/Sub 连接**：

```
CLIENT TRACKING ON REDIRECT <client-id-of-notification-connection>
```

通知会发到 `__redis__:invalidate` 频道。

### 6.4 SDK 支持现状

- **Lettuce**：原生支持 RESP3 + tracking → 客户端缓存（`ClientSideCachingOptions`）
- **node-redis** 4+：支持
- **redis-py**：实验性
- **go-redis**：手动实现（`PubSub` + tracking 通知）

### 6.5 何时用客户端缓存

- 读远大于写
- key 集合可枚举（白名单适合 BCAST 前缀）
- 一致性要求秒级，不能更高
- 应用进程内存够（典型 100MB-1GB）

不适合：

- 写极频繁 → invalidate 风暴
- 一致性要求毫秒 → 仍可能短暂不一致（push 本身有传播延迟）

---

## 第七章：TLS / mTLS

### 7.1 Redis 6.0+ 内置 TLS

```
# redis.conf
tls-port 6380
port 0                          # 关闭明文端口
tls-cert-file /etc/redis/tls/redis.crt
tls-key-file /etc/redis/tls/redis.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients yes            # 启用 mTLS
```

客户端：

```
redis-cli --tls --cert client.crt --key client.key --cacert ca.crt -h ... -p 6380
```

### 7.2 性能影响

- 握手 4 RTT（无 session resumption） / 1 RTT（有）
- 加密 / 解密 CPU 开销：~10-15% 吞吐损失（单节点 100k → ~85k）
- TLS 1.3 显著优于 1.2

### 7.3 客户端配置要点

| SDK | 配置 |
|---|---|
| Jedis | `JedisClientConfig.builder().ssl(true).hostnameVerifier(...).build()` |
| Lettuce | `RedisURI.Builder.sentinel(...).withSsl(true).withVerifyPeer(true)` |
| go-redis | `Options.TLSConfig = &tls.Config{...}` |
| redis-py | `redis.Redis(ssl=True, ssl_ca_certs=..., ssl_cert_reqs='required')` |

**生产必查**：

1. `verifyPeer = true`（默认 false 在某些 SDK），否则中间人 → 失效
2. SNI 启用（域名场景）
3. session resumption / 0-RTT 启用

---

## 第八章：超时配置

### 8.1 几个不同层的超时

```
[业务 RPC 总超时]
  └─[客户端连接获取]
      └─[Redis 调用超时]
          ├─[连接建立 DialTimeout]
          ├─[读 ReadTimeout]
          └─[写 WriteTimeout]
```

任何一层配置不当都会传染整个链路。

### 8.2 推荐值（同机房）

| 超时 | 推荐 | 注释 |
|---|---|---|
| DialTimeout | 1-3s | 包括 TCP + TLS 握手 |
| ReadTimeout | 100-500ms | 单命令；BLPOP 之类要长 |
| WriteTimeout | 100-500ms | 一般等于 ReadTimeout |
| 连接池等待 | 1-2s | maxWait |
| 业务 RPC 总超时 | 1-3s | 给业务流程留余地 |

### 8.3 长操作的超时单独配

`BLPOP / BRPOP / XREAD BLOCK`：默认阻塞，要单独大超时

```go
// go-redis 例子
rdb.BLPop(ctx, 10*time.Second, "queue").Result()
// ReadTimeout 必须 > 10s，否则命令还没返回客户端先超时
```

### 8.4 超时 + 重试的危险组合

```
client → SET k v   ← ReadTimeout 500ms 后客户端报错
                   ← 但实际 Redis 已经处理完，只是返回包 600ms 才到
client 重试 SET k v ← 二次写入
```

**很多 SDK 默认会自动重试**。对幂等命令（GET/SET 同值）无害，对**非幂等**（INCR / SADD）就重复了。**生产建议：禁用 SDK 自动重试，由业务层判断**。

```go
rdb := redis.NewClient(&redis.Options{
    MaxRetries: 0,   // 关闭重试
})
```

---

## 第九章：Pub/Sub 与 Streams 客户端

### 9.1 Pub/Sub 长连接管理

订阅模式下客户端这条连接专用于接收消息，不能再发普通命令（**RESP2**；RESP3 可以共用，因为有 push 类型）。

```python
p = r.pubsub()
p.subscribe('channel1')
for msg in p.listen():
    print(msg)
```

陷阱：

1. **断线重连后要重新 SUBSCRIBE**：Pub/Sub 是服务端 in-memory 状态，断连即丢
2. **没有 ack / persistence**：消息丢就丢了
3. **慢消费者会被 kill**：`client-output-buffer-limit pubsub 32mb 8mb 60` 触发即断连

### 9.2 Sharded Pub/Sub（7.0+）

`SPUBLISH` / `SSUBSCRIBE` 让 Pub/Sub 走 Cluster 分片，消息按 channel 名 hash 到固定节点，**不再全集群广播**。

```
SSUBSCRIBE channel1   ← 客户端连到 channel1 所在的 shard
SPUBLISH channel1 msg ← 也要发到那个 shard
```

客户端必须 Cluster-aware 才能用，类似 key 路由。

### 9.3 Streams Consumer Group 客户端

```python
# 创建 group
r.xgroup_create('mystream', 'mygroup', id='0', mkstream=True)

# 消费
while True:
    msgs = r.xreadgroup('mygroup', 'consumer-1', {'mystream': '>'}, count=10, block=5000)
    for stream, msg_list in msgs:
        for msg_id, fields in msg_list:
            process(fields)
            r.xack('mystream', 'mygroup', msg_id)
```

关键点：

- `>` 表示"只取没分配过的"
- 处理完必须 `XACK`，否则进 PEL（pending list）
- 用 `XPENDING` / `XCLAIM` 处理超时未 ack 的消息（消费者挂掉的场景）

详见 [R07 Streams](./07-精通-Redis-Streams.md)。

---

## 第十章：Lua 与 EVALSHA

### 10.1 EVAL vs EVALSHA

```
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
EVALSHA <sha1> 1 mykey
```

EVAL 每次发全文，EVALSHA 只发 40 字符 SHA1 → **省带宽**。

### 10.2 客户端缓存 SHA 的标准模式

```python
script = "return redis.call('GET', KEYS[1])"
sha = r.script_load(script)   # 一次性加载到服务端 SCRIPT cache

# 业务代码
try:
    r.evalsha(sha, 1, key)
except redis.exceptions.NoScriptError:
    # 服务端重启 / FLUSH 后 SCRIPT cache 没了
    sha = r.script_load(script)
    r.evalsha(sha, 1, key)
```

主流 SDK 自动做这个回退（如 go-redis 的 `Script.Run`）。

### 10.3 Cluster 下的 Lua

所有 KEYS 必须同 slot，详见 R06。**Functions（7.0+）** 是更好的替代品。

---

## 第十一章：诊断与排查

### 11.1 看客户端连接

```
CLIENT LIST                # 列出所有连接
CLIENT INFO                # 当前连接
CLIENT KILL ADDR ip:port   # 踢一条
CLIENT NO-EVICT ON         # 这条连接保护：内存满时不踢
CLIENT NO-TOUCH ON         # 此连接的读不刷 LRU/LFU 计数
CLIENT PAUSE 5000          # 暂停所有 client 5 秒（用于热备切换）
```

`CLIENT LIST` 输出字段：

```
id=10 addr=10.0.0.5:53294 fd=12 name=worker-1 age=3600 idle=0 flags=N
  db=0 sub=0 psub=0 ssub=0 multi=-1 qbuf=0 qbuf-free=20474 obl=0 oll=0
  omem=0 events=r cmd=get user=default resp=3 lib-name=jedis lib-ver=5.0.0
```

关键字段：

| 字段 | 含义 | 排查 |
|---|---|---|
| `age` / `idle` | 总活时 / 闲置秒 | idle 太久看是不是泄漏 |
| `qbuf` / `qbuf-free` | 输入缓冲区已用 / 剩余 | qbuf 高 → 客户端发大 pipeline 没收 |
| `obl` / `oll` / `omem` | 输出缓冲区当前 / 队列 / 总内存 | omem 高 → 慢消费者 |
| `cmd` | 上一条命令 | 看 client 都在跑啥 |
| `flags` | N=normal, M=master, S=slave, O=monitor, P=pubsub, b=blocked | 查阻塞客户端 |
| `lib-name` / `lib-ver` | 客户端 SDK 信息（7.2+，需客户端发 `CLIENT SETINFO`） | 排查特定 SDK 行为 |

### 11.2 连接数突增排查

```
INFO clients
# connected_clients:5234              ← 突然涨到 5k
# blocked_clients:200                 ← BLPOP 等等待中
# tracking_clients:50                 ← tracking 开了的
# clients_in_timeout_table:0
```

突增几个根因：

1. 业务实例上下线频繁，没复用连接
2. 池配置错（无上限）
3. 连接泄漏
4. DDoS / 扫描

抓现场：

```
CLIENT LIST | awk '{print $2}' | cut -d= -f2 | cut -d: -f1 | sort | uniq -c | sort -nr
# 看哪个 IP 连接最多
```

### 11.3 客户端缓存命中率

```
INFO stats
# total_connections_received:12345
# instantaneous_ops_per_sec:8200
# tracking_total_keys:..             ← tracking 在追的 key 数
# tracking_total_items:..            ← 全部 (key, client) 项数
# tracking_total_prefixes:..         ← BCAST 前缀数
```

业务侧自己埋点统计命中率，跟 Redis 这边比对。

### 11.4 全链路 RTT 与排队

慢命令排查：

```
SLOWLOG GET 10
LATENCY DOCTOR
LATENCY HISTORY event-name
```

客户端侧记录耗时：

```go
start := time.Now()
err := rdb.Get(ctx, key).Err()
elapsed := time.Since(start)
if elapsed > 50*time.Millisecond {
    log.Warn("slow redis call: %v", elapsed)
}
```

P99 飙升时关键问题：**是 Redis 慢，还是客户端排队？**

- Redis 慢 → 服务端 SLOWLOG 会有
- 客户端排队（池满 / Lettuce 多路复用满负载）→ Redis 这边看不到，只看客户端指标

---

## 第十二章：上线 checklist

### 12.1 部署前

- [ ] 连接池 max size 计算过，写进配置
- [ ] 连接池 testWhileIdle 开启，evictionRun 30s
- [ ] DialTimeout 3s / ReadTimeout 500ms（按业务调）
- [ ] tcp-keepalive 客户端 + 服务端都配（< NAT 超时）
- [ ] SDK 自动重试**显式**配置（默认 1 或 0）
- [ ] TLS verifyPeer = true
- [ ] Cluster 客户端用 cluster 模式，不要单实例模式连 cluster endpoint
- [ ] Lua 脚本 EVALSHA + NoScript 回退
- [ ] RESP3（如启用）测试 HGETALL 等响应类型变化
- [ ] Pub/Sub / BLPOP 等长命令 ReadTimeout 单独大一些
- [ ] 客户端 SetName（`CLIENT SETNAME`）便于排查
- [ ] CLIENT SETINFO LIB-NAME / LIB-VER（7.2+）

### 12.2 运行中

- [ ] `CLIENT LIST` 每天检查连接数和 idle 分布
- [ ] 监控 `connected_clients` / `blocked_clients` / 客户端池使用率
- [ ] `SLOWLOG` 接到告警平台
- [ ] 客户端侧 RT 直方图（P50 / P99 / P99.9）
- [ ] 连接借用失败率告警
- [ ] 报错日志按 SDK 错误类别（NoScript / MOVED / CROSSSLOT / Timeout）分类统计

### 12.3 故障演练

- [ ] kill 一条连接（`CLIENT KILL`）→ SDK 是否自动重连 + 重新 HELLO
- [ ] CLUSTER FAILOVER 主备切换 → SDK 是否平滑（< 5s 切流）
- [ ] 网络抖动注入（tc qdisc loss）→ 超时与重试行为符合预期
- [ ] 慢命令注入（`DEBUG SLEEP 1`）→ 业务降级路径生效

---

## 第十三章：常见反模式

### 13.1 每次请求 new 一个客户端

```go
func handler() {
    rdb := redis.NewClient(...)   // ❌ 每次新建
    defer rdb.Close()
    rdb.Get(ctx, "k")
}
```

正确：全局单例 / 注入。

### 13.2 用同步阻塞 SDK 在 reactive 应用里

Spring Reactive 用 Jedis（阻塞）→ 反应式线程被阻塞 → 性能崩。
正确：用 Lettuce 的 reactive API。

### 13.3 Cluster 客户端配 Master 列表只填一个

```
ClusterClient{ Addrs: []string{":7000"} }   ← 只填一个 master
```

如果这个 master 宕了，客户端启动期就拿不到拓扑。
正确：填全部 master 或 seed nodes（3 个起步）。

### 13.4 用 KEYS 而不是 SCAN

详见 R08。客户端代码里如果出现 `KEYS *`，code review 直接 reject。

### 13.5 Lua 脚本依赖时间

```lua
-- ❌ 不可重入：脚本里调 redis.call('TIME') 拿时间用作 key
local now = redis.call('TIME')
redis.call('SET', KEYS[1] .. ':' .. now[1], ARGV[1])
```

复制到副本时副本的 `TIME` 不同 → 副本数据与 master 不一致。
正确：时间从客户端传 ARGV 进去。

### 13.6 Pub/Sub 当 MQ 用

详见 R07。Pub/Sub 没持久化 / 没 ack / 没回溯，宕机消息全丢。要 MQ 语义用 Streams。

### 13.7 客户端缓存不设上限

Tracking 开了之后客户端无限缓存 → OOM。
正确：本地缓存用 LRU + 大小上限 + 过期。

---

## 总结

客户端是服务端性能的"放大器"——服务端再快，客户端用错性能就是 1/10。本章核心：

1. **协议**：RESP3 是趋势，升级要注意响应类型变化；RESP2 兼容性好
2. **客户端选型**：Java 主流 Lettuce，Go 用 go-redis，Python 用 redis-py，Node 用官方 node-redis
3. **连接池**：大小宁少勿多，开 testWhileIdle，开 tcp-keepalive 抗 NAT
4. **Cluster aware**：让 SDK 处理 MOVED/ASK，自己别造轮子
5. **Pipeline / Tx**：能 Pipeline 就 Pipeline，要原子用 MULTI 或 Lua
6. **客户端缓存**：Tracking + RESP3 push 是减少 RTT 的杀器
7. **超时**：DialTimeout / ReadTimeout / 业务总超时分层配
8. **诊断**：`CLIENT LIST` / `INFO clients` / 客户端 RT 直方图 一个不能少

---

## 练习题

1. 写代码：一个 Go 程序，连 Cluster（3 主 3 从），用连接池，配合理超时，跑 1 万 QPS 持续 1 分钟。监控池利用率与每条命令 RT。
2. 用 `redis-cli -3` + `MONITOR` 抓 `HELLO 3` 握手的报文，画出 RESP3 与 RESP2 的差别。
3. 在测试集群上手动 `CLUSTER FAILOVER`，观察 go-redis 切流耗时与错误数。
4. 配一个 Lettuce 客户端开 Client Tracking，写一个 demo：A 客户端读 key，B 客户端改 key，A 是否收到 invalidate 推送？延迟多少？
5. 写一段会触发"连接泄漏"的伪代码，用 `CLIENT LIST` 在 30 分钟内看连接数变化。然后改成 try-with-resources / context manager，对比。
6. 用 tc 给 redis 链路加 100ms 延迟，看你的客户端超时配置会不会触发误报。
7. 用 `CLIENT PAUSE 5000` 模拟主线程卡顿 5 秒，看你的应用降级是否生效。
8. 写一个 Lua 脚本 + EVALSHA 调用，故意在调用前 `SCRIPT FLUSH`，看你的 SDK 是否正确回退到 SCRIPT LOAD。
9. RESP3 下 `HGETALL` 返回 map。把 SDK 升级到 RESP3 后，原本拼数组的代码会怎么炸？写一个 demo 重现并修复。
10. 容量规划题：业务 QPS 1 万，平均 1 个请求要打 5 次 Redis（GET），单次 RT 1ms。求最少需要多少连接池容量？

---

> 🔁 反馈：客户端的"坑"基本都在线上才暴露。强烈建议做一次混沌演练（kill 连接 / 主备切换 / 网络抖动）压一遍 SDK 行为
