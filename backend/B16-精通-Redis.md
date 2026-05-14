# 精通 Redis：数据结构、持久化与集群

> 课程编号：B16
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Redis
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：不止是缓存

```redis
SET user:42 "alice"            # 缓存
INCR counter:page_views        # 计数
ZADD leaderboard 100 alice     # 排行榜
LPUSH queue:tasks "task-1"     # 队列
PUBLISH events "user_created"  # pub/sub
XADD stream * msg "hello"      # 持久消息流
```

同一 Redis 集群能做缓存、限流、排行榜、消息队列、分布式锁、计数器、session。这一章讲清各数据结构、持久化选项、cluster、生产部署。

---

## 第一章：基础

### 1.1 单线程 + 内存

Redis 单线程处理命令（6.0+ 网络 I/O 多线程，逻辑仍单线程）。命令原子。命令直接操作内存——所有数据 RAM。

### 1.2 性能

- 简单命令：~100K QPS / 单实例
- pipeline / batch：百万 QPS 可能
- 延迟：本地 < 1ms

### 1.3 Key

- 字符串
- 通常 `entity:id:field` 命名
- 短而表意：`u:42:profile` 比 `user_profile_for_42` 节省

### 1.4 重要：Redis vs Valkey 之争（2026 选型必读）

**2024-03**：Redis Inc. 把 Redis 7.4 起的许可证从 BSD 改成 dual RSALv2 / SSPLv1（限制云厂商二次商业化）。**2024-03 末**：Linux Foundation + AWS / Google / Oracle / Ericsson 等 fork Redis 7.2.4 → **Valkey**，沿用 BSD-3。

**2025-05**：Redis Inc. 让步——**Redis 8.0 改回开源**，但用 **AGPLv3**（tri-license：RSALv2 / SSPLv1 / AGPLv3 任选）。

**到了 2026 年实际状况**：

| 维度 | Redis 8.x | Valkey 8.x |
|---|---|---|
| **协议兼容** | Redis 命令集（含 8.0 新增） | Redis 7.2.4 起 fork，命令兼容 |
| **License** | 三选一（含 AGPLv3）；商业用 AGPL 要小心网络 copyleft | BSD-3-Clause，无负担 |
| **治理** | Redis Inc. 单厂商主导 | Linux Foundation 多厂商 |
| **核心模块** | 集成 JSON / TimeSeries / Bloom / Top-K / Vector Set | 仅核心 + 部分模块 fork |
| **AI / Vector** | 主推方向（vector set beta） | 较保守，强化 cluster / 内存效率 |
| **云厂商默认** | Redis Cloud 自家 | AWS ElastiCache 默认、GCP Memorystore for Valkey |

**选型建议**：

- **新自部署服务**：选 **Valkey**——许可证清爽、无单厂商风险、性能持平甚至更优。
- **已用 AWS ElastiCache / GCP Memorystore**：跟着托管服务走（多数已默认 Valkey）。
- **需要原生 vector / JSON / 时序模块**：用 Redis 8（评估 AGPLv3 是否影响你的产品）。
- **绝大多数纯 cache + queue 场景**：两者**完全可以无感切换**——客户端、命令、行为基本一致。

> 本课程后续示例同时适用 Redis 8.x 和 Valkey 8.x，除非显式标注为 Redis 8 独有模块特性。

---

## 第二章：数据结构

### 2.1 String

```redis
SET key "value"
GET key
INCR counter
EXPIRE key 60
SET key "v" EX 60 NX     # 仅不存在时设置 + 60s 过期
MGET k1 k2 k3
```

最常见。可存 JSON、序列化对象、计数器、token。

最大 512MB / value——但单 key 不应超 1MB（网络 + 阻塞）。

### 2.2 List

```redis
LPUSH list "a"
RPUSH list "b"
LPOP list
LRANGE list 0 -1
LLEN list
```

双端链表。适合：
- 队列（LPUSH + RPOP）
- 最近访问历史（capped）
- 时间线

LIMIT：`LTRIM list 0 99` 仅保留前 100 个。

### 2.3 Hash

```redis
HSET user:42 name "alice" age 30 email "a@x.com"
HGET user:42 name
HGETALL user:42
HINCRBY user:42 age 1
```

object 风格。比单独多个 string key 更省内存。

### 2.4 Set

```redis
SADD tags "go" "redis" "backend"
SMEMBERS tags
SISMEMBER tags "go"
SINTER set1 set2          # 交集
SUNION set1 set2
SDIFF set1 set2
SRANDMEMBER set 3
```

无序、唯一。适合：
- 标签
- 唯一访客
- 关注关系

### 2.5 Sorted Set (ZSet)

```redis
ZADD lb 100 "alice" 80 "bob" 120 "charlie"
ZRANGE lb 0 9              # top 10 升序
ZREVRANGE lb 0 9           # top 10 降序
ZRANGEBYSCORE lb 80 100
ZINCRBY lb 5 "alice"
ZRANK lb "alice"
```

按 score 排序的 set。**Redis 最强大的结构之一**。

经典用法：
- 排行榜
- 延迟队列（score = 触发时间戳）
- 优先级队列
- 时间窗口限流（score = timestamp）

### 2.6 HyperLogLog

近似 unique counter。极少内存（12KB）记 10 亿唯一值（误差 0.81%）。

```redis
PFADD visitors "user1" "user2" "user3"
PFCOUNT visitors          # 唯一计数
PFMERGE total v1 v2 v3    # 合并多个 hll
```

UV 统计经典。

### 2.7 Bitmap

```redis
SETBIT user:42:days_active 100 1   # 第 100 天 active
GETBIT user:42:days_active 100
BITCOUNT user:42:days_active        # 总活跃天数
BITOP AND result u1 u2              # 多 bitmap 操作
```

适合"用户 X 行为日历"等 N 选 1 场景。

### 2.8 Geo

```redis
GEOADD places 13.361 38.115 "Palermo"
GEORADIUS places 15 37 200 km
GEODIST places "Palermo" "Catania"
```

经纬度索引。底层是 sorted set + geohash。

### 2.9 Stream（5.0+）

```redis
XADD events * type "click" user 42
XREAD COUNT 10 STREAMS events 0
XGROUP CREATE events grp1 0
XREADGROUP GROUP grp1 consumer1 COUNT 10 STREAMS events >
```

像 Kafka 的轻量版。持久消息、消费组、ACK。比 list 队列功能强很多。

---

## 第三章：过期与驱逐

### 3.1 TTL

```redis
EXPIRE key 60               # 60s 后过期
PEXPIRE key 60000           # 毫秒
EXPIREAT key 1234567890     # 绝对 unix 时间
TTL key                     # 剩余秒
PERSIST key                 # 移除 TTL
```

过期实现：lazy expiration（访问时检查）+ 主动扫描（每秒少量随机 key）。

### 3.2 Eviction policy

maxmemory 满后选择策略：

| Policy | 含义 |
|---|---|
| noeviction | 拒绝写（OOM 错误） |
| allkeys-lru | LRU 全 key |
| allkeys-lfu | LFU 全 key |
| volatile-lru | LRU 仅带 TTL 的 |
| volatile-lfu | LFU 仅带 TTL |
| volatile-ttl | 删最快过期 |
| allkeys-random | 随机 |
| volatile-random | 随机带 TTL 的 |

cache 场景：`allkeys-lru`。主存场景：`noeviction`。

### 3.3 内存监控

```redis
INFO memory
MEMORY USAGE key            # 单 key 估算大小
```

`used_memory_rss` 是 OS 看到的；`used_memory` 是 Redis 分配的。

---

## 第四章：持久化

### 4.1 RDB（快照）

定期把整个数据集 dump 成二进制文件。

```
save 900 1     # 900s 内 ≥ 1 个变更 → 触发
save 300 10
save 60 10000
```

优点：紧凑、恢复快、备份友好。
缺点：两次 snapshot 间数据可能丢；fork 进程瞬时内存翻倍。

### 4.2 AOF（追加日志）

每条写命令追加到日志。

```
appendonly yes
appendfsync everysec    # always / everysec / no
```

- `always`：每命令 fsync——0 丢失，但慢
- `everysec`：每秒 fsync——最多 1 秒丢失（默认推荐）
- `no`：OS 决定 fsync——快但可能丢更多

文件越来越大 → rewrite（重写为最小命令集）。

### 4.3 混合

Redis 4.0+ 支持 AOF + RDB 混合：AOF 文件开头是 RDB snapshot，后面是增量命令。**生产推荐**。

```
aof-use-rdb-preamble yes
```

### 4.4 cache 还是主存

- 仅 cache：可关持久化（`save ""`），性能最佳
- 主存或半重要：AOF everysec 默认
- 极致安全：AOF always（性能代价大）

---

## 第五章：复制与高可用

### 5.1 主从

```
redis-server --replicaof master.host 6379
```

异步复制。备库可读不可写（除非 slave-read-only no）。

### 5.2 Sentinel

Sentinel 监控主，主挂自动选 replica 升级，并通知 client。

```
sentinel monitor mymaster 192.168.1.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

3 个 sentinel 节点 + 1 主 + N replica = 高可用 + 自动 failover。

### 5.3 Cluster

水平分片 + 自动 failover。16384 个 slot 分到 N 个 master（每 master 可有自己的 replica）。

```
CLUSTER MEET ...
CLUSTER ADDSLOTS 0..5460
```

特性：
- 多 key 命令仅当 key 在同一 slot 时有效
- 用 `{tag}` 强制同 slot：`user:{42}:name`、`user:{42}:age` 同 slot
- 自动 reshard
- 集群裂脑容忍：majority 才提升

### 5.4 哪个？

- **单实例**：开发、低规模
- **主从 + Sentinel**：中等规模 + HA
- **Cluster**：数据超单机内存 或 极高 QPS

---

## 第六章：典型用法

### 6.1 缓存

```python
val = r.get(f"user:{id}")
if not val:
    val = db.query(id)
    r.setex(f"user:{id}", 3600, val)
```

### 6.2 分布式锁

```python
lock_key = f"lock:order:{id}"
if r.set(lock_key, my_id, nx=True, ex=10):
    try:
        # critical section
    finally:
        # 用 lua 脚本保证"id 匹配才删"，防误删他人锁
        r.eval(release_script, 1, lock_key, my_id)
```

注意：单实例 lock 不可靠（主挂时复制延迟可能让两人拿锁）。Redlock 算法是多节点版（也有争议，Martin Kleppmann 写过批评文）。

### 6.3 限流（token bucket）

```python
# fixed window
key = f"rate:{user_id}:{minute}"
count = r.incr(key)
if count == 1: r.expire(key, 60)
if count > 100: raise RateLimited

# sliding window（用 sorted set）
key = f"rate:{user_id}"
now = time.time()
r.zremrangebyscore(key, 0, now - 60)
r.zadd(key, {req_id: now})
r.expire(key, 60)
if r.zcard(key) > 100: raise RateLimited
```

### 6.4 排行榜

```python
r.zadd("leaderboard", {user_id: score})
top10 = r.zrevrange("leaderboard", 0, 9, withscores=True)
my_rank = r.zrevrank("leaderboard", user_id)
```

### 6.5 延迟队列

```python
# 入队
r.zadd("delayed", {task_id: trigger_at_timestamp})

# 消费 worker
while True:
    now = time.time()
    tasks = r.zrangebyscore("delayed", 0, now, start=0, num=10)
    for t in tasks:
        r.zrem("delayed", t)
        process(t)
    time.sleep(0.1)
```

### 6.6 计数器

```python
r.incr(f"views:{post_id}")
r.hincrby(f"stats:{post_id}", "likes", 1)
```

原子，高并发安全。

### 6.7 Session

```python
r.setex(f"session:{token}", 3600, json.dumps(session_data))
```

TTL 自动过期。

---

## 第七章：Lua 脚本

### 7.1 用途

把多命令打包原子执行。

```lua
-- check-and-decrement
local current = redis.call("GET", KEYS[1])
if not current or tonumber(current) < tonumber(ARGV[1]) then
    return -1
end
return redis.call("DECRBY", KEYS[1], ARGV[1])
```

```python
script = r.register_script(lua_code)
result = script(keys=["stock:42"], args=[1])
```

### 7.2 注意

- 阻塞单线程——脚本运行期间其他命令排队
- 应短小（< 50ms）
- 不要 KEYS/SCAN 全表

### 7.3 替代：Redis Functions（7.0+）

类似 Lua 但更结构化，可注册函数库。

---

## 第八章：性能与优化

### 8.1 Pipeline

```python
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f"k{i}", "v")
pipe.execute()
```

1000 个命令一个 RTT。提升 10-100x。

### 8.2 批量命令

`MGET k1 k2 k3` 比 3 次 GET 快。

### 8.3 避免大 key

```redis
LRANGE huge_list 0 -1   # 几十万元素 → 阻塞
```

`SCAN`、`HSCAN`、`SSCAN` 分页。

### 8.4 keys vs scan

```redis
KEYS user:*       # 阻塞整库扫描！
SCAN 0 MATCH user:* COUNT 100   # 游标式
```

生产**永远不要 KEYS**。

### 8.5 慢日志

```redis
SLOWLOG GET 10
CONFIG SET slowlog-log-slower-than 10000   # 10ms
```

---

## 第九章：生产级最佳实践

1. **maxmemory + 合适 eviction policy**：防 OOM。
2. **AOF everysec + RDB**：混合持久化。
3. **monitor 关键 metric**：内存、QPS、慢日志、连接数。
4. **生产用 Sentinel 或 Cluster**：单点不可接受。
5. **value 不要大**：单 key < 1MB；list / hash / set 控制元素数。
6. **过期 TTL**：避免内存爆。
7. **pipeline 批量化**：减 RTT。
8. **scan 替代 keys**。
9. **lock 用 Lua 释放**：防误删。
10. **不同业务隔离**：DB 号或单独实例。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：KEYS *
生产阻塞。

### ❌ 陷阱 2：长 list / hash
LPUSH 几亿 → LRANGE 慢；分多个 key。

### ❌ 陷阱 3：忘 TTL
内存爆满 → OOM 或 eviction 删错。

### ❌ 陷阱 4：分布式锁不释放
异常路径漏掉。defer 释放 + lock 自带 TTL。

### ❌ 陷阱 5：脚本运行太久
阻塞整服务。脚本短小。

### ❌ 陷阱 6：单实例当 HA
主挂全部歇菜。Sentinel / Cluster。

### ❌ 陷阱 7：cache 当主存
持久化保证不如关系 DB。重要数据回到 RDB。

---

## 第十一章：练习题

**练习 1**：实现一个"用户最近浏览 10 个商品"列表。

**练习 2**：实现一个分布式锁，要求自动释放、防误删。

**练习 3**：1000 万用户，每天活跃 100 万——如何用 100MB 内存近似统计每天活跃数？

**练习 4**：为什么 Redis 单线程仍能做到 100K QPS？

**练习 5**：Redis Cluster 中 `MSET user:1 a user:2 b` 可能失败，为什么？怎么修？

---

## 参考答案

**练习 1**：
```python
key = f"recent:{user_id}"
r.lpush(key, product_id)
r.ltrim(key, 0, 9)        # 仅保留前 10
r.expire(key, 86400 * 30)
```

**练习 2**：
```python
def acquire(key, ttl=10):
    token = uuid.uuid4().hex
    if r.set(key, token, nx=True, ex=ttl):
        return token
    return None

release_script = """
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
"""
def release(key, token):
    r.eval(release_script, 1, key, token)
```

**练习 3**：HyperLogLog。`PFADD daily:2026-05-12 user_id`，`PFCOUNT daily:2026-05-12` 给近似数。100MB 够数百天的统计。

**练习 4**：
- 内存操作纳秒级
- 简单数据结构（list/hash/set 都是 O(1) 或 O(log N) 的常用 op）
- I/O 多路复用（epoll）
- 6.0+ 网络多线程
- 单线程避免锁开销 + cache 友好

**练习 5**：MSET 多 key 必须同 slot。`user:1` 和 `user:2` 用默认哈希到不同 slot → CROSSSLOT 错误。修：`MSET {user}:1 a {user}:2 b` 或拆成单独 SET。

---

## 小结

| 数据结构 | 用途 |
|---|---|
| String | 缓存、计数 |
| Hash | 对象字段 |
| List | 队列、最近 |
| Set | 标签、唯一 |
| ZSet | 排行榜、延迟队列、限流 |
| HLL | 近似 unique 计数 |
| Bitmap | 用户日历 |
| Geo | 经纬度 |
| Stream | 持久消息队列 |

| 持久化 | 选择 |
|---|---|
| 仅 cache | 关 |
| 一般 | AOF everysec + RDB |
| 极致安全 | AOF always |

| 部署 | 何时 |
|---|---|
| 单实例 | dev |
| Sentinel | HA |
| Cluster | 数据量 + 极高 QPS |

下一篇 **B17 — 消息队列：Kafka vs RabbitMQ vs NATS** 会拆开主流 MQ 的设计、何时选哪个。

---

> 📁 本课程位于 `/data/workspace/dp4/backend/B16-精通-Redis.md`
