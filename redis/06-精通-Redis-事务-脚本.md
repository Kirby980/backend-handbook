# 精通 Redis 事务、Pipeline 与脚本：MULTI/EXEC、Lua、Functions

> 关联章节：[07 Cluster](./07-精通-Redis-Cluster.md)、[09 Streams](./09-精通-Redis-Streams.md)、[10 性能模型](./10-精通-Redis-性能模型.md)

---

## 引言：Redis 上的"原子性"有多种

很多开发者第一次写 Redis 的多步操作时会问："Redis 有事务吗？" 答案是：**有，但不是你以为的那种**。Redis 上有四种"原子级"操作模式：

1. **单条命令**：天然原子（单线程）
2. **MULTI / EXEC 事务**：命令打包顺序执行；**没有回滚**；只保证"中间不会插入别人的命令"
3. **Lua 脚本**：服务器端执行的脚本；**整个脚本期间独占主线程**，比 MULTI 更强的原子性
4. **Functions（Redis 7.0+）**：Lua 脚本的"持久化命名版本"，可热部署管理

加上"Pipeline"（客户端层批量发送，**不保证原子**）总共 5 个相关概念，很容易混。

读完之后你应该能回答：

- 为什么 MULTI/EXEC 中的某条命令失败不会回滚已执行的？
- WATCH 是怎么实现"乐观锁"的？什么场景能用、什么场景不行？
- 一段复杂业务（扣库存 + 写日志 + 更新计数）是 Lua 写还是事务写？
- Pipeline 跟事务的关键差别在哪？
- Functions 比 Lua 优在何处？

---

## 第一章：单条命令的原子性

```
INCR counter        # 原子的递增
HSET h f1 v1 f2 v2  # 一条命令同时改两个字段，原子
LPUSH list a b c d  # 同时入队 4 个元素，原子
ZADD z 1 a 2 b      # 同时加 2 个元素，原子
SET k v EX 60 NX    # set + expire + nx 三个动作合一，原子
```

**Redis 单线程**：任何一条命令在执行期间没有其他命令插入。所以"单条复杂命令"已经天然解决了大量并发场景。

实际上 Redis 提供了很多"组合命令"专门取代客户端的"读-改-写"：

| 反模式 | 推荐 |
|---|---|
| `GET k → 改值 → SET k` | `INCR / DECR / INCRBY` 或 `SET k newvalue EX X NX` |
| `EXISTS k → SET k v` | `SET k v NX` |
| `GET k → DEL k` | `GETDEL k` (Redis 6.2+) |
| `GET k → SET k v_new` | `SET k v_new GET` 或 `GETSET`（已 deprecated） |
| `HGET h f → HSET h f v_new` | `HSETNX` / `HINCRBY` |

**写 Redis 代码第一原则：单命令能做的事别用事务**。

---

## 第二章：MULTI / EXEC —— 命令打包

### 2.1 基本用法

```
MULTI               ← 开启事务
SET k1 v1
INCR counter
LPUSH list item
EXEC                ← 一次性把队列里的命令顺序执行

返回:
1) OK
2) (integer) 42
3) (integer) 3
```

MULTI 之后的命令**入服务端队列但不立刻执行**，EXEC 时一次性按顺序执行。期间**这个 client 的连接是独占的**——其他 client 的命令不会插入。

### 2.2 没有回滚

```
MULTI
SET k v
INCRBY k 10     ← 错误：k 是字符串 "v"，不能 INCR
SET k2 v2
EXEC

返回:
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
3) OK
```

**第二条报错，第一条和第三条仍然执行**。Redis 不回滚。

理由：作者认为"语法错误"或"类型错误"基本都是 bug，应当在开发期就解决；运行时回滚成本高且不必要。这跟 SQL 数据库哲学不同。

**实际工程影响**：如果你写的事务里第二条失败可能导致一致性问题，需要：

1. 用 Lua 脚本代替（详 §4）
2. 业务层判断 EXEC 返回值并自行补偿

### 2.3 语法错误 vs 运行时错误

```
# 语法错误：整批不执行
MULTI
SET                    ← 语法错误：缺参数
INCR counter
EXEC
→ (error) EXECABORT Transaction discarded because of previous errors.

# 运行时错误：仍执行，单条失败
MULTI
SET k 5
INCR k                 ← 假设上一条成功了，这条 OK
INCR not_a_number      ← 假设 not_a_number 是字符串，这条运行时失败
EXEC
→ ... 一部分成功一部分失败 ...
```

`MULTI` 之后的命令客户端会先在本地校验语法 → 服务端 EXEC 之前已确定能"队列入栈"。`EXECABORT` 是命令进队列时就发现的（如参数数量错）；运行时错误是真执行才发现的。

### 2.4 DISCARD

```
MULTI
SET k v
DISCARD           # 取消事务，所有排队命令丢弃
```

### 2.5 客户端 SDK 包装

```python
# redis-py
pipe = r.pipeline(transaction=True)   # 默认就是 MULTI/EXEC
pipe.set("k1", "v1")
pipe.incr("counter")
results = pipe.execute()
```

```java
// Lettuce
RedisCommands<String, String> sync = connection.sync();
sync.multi();
sync.set("k1", "v1");
sync.incr("counter");
TransactionResult result = sync.exec();
```

---

## 第三章：WATCH —— 乐观锁

### 3.1 场景

经典"读改写"竞争：

```
client A:  GET counter → 5    
client B:  GET counter → 5    （并发）
client A:  SET counter 6
client B:  SET counter 6      （A 的更新被覆盖）
```

INCR 命令本身原子可解决这个简单场景。但**复杂业务**（如"判断 counter < limit 后才 +1"）需要"读+条件判断+写"。

### 3.2 WATCH 实现

```
WATCH counter           ← 监视 counter
val = GET counter
if val < 100:
    MULTI
    INCR counter
    EXEC                ← 如果 counter 在 WATCH 后被改过，EXEC 返回 nil，全部不执行
else:
    UNWATCH
```

**核心机制**：

- WATCH 后，Redis 标记这些 key 为"被某 client 监视"
- 任何对这些 key 的修改（来自其他 client）都会标记该监视失效
- 该 client 后续的 EXEC 检查到失效就**整体回滚为 nil**

效果：**乐观锁 + CAS**——发现冲突就重试。

### 3.3 完整代码

```python
def safe_incr_to_limit(r, key, limit):
    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(key)
                val = int(pipe.get(key) or 0)
                if val >= limit:
                    pipe.unwatch()
                    return False
                pipe.multi()
                pipe.incr(key)
                pipe.execute()
                return True
            except redis.WatchError:
                continue   # 重试
```

**注意点**：

- WATCH 后调用 multi() 之前可以正常读，但**任何修改命令在 multi() 之前会直接执行**，不会入队
- EXEC 失败（返回 nil）会自动 UNWATCH
- 主动 UNWATCH 取消所有监视
- DISCARD 会取消所有监视

### 3.4 WATCH 的限制

- **集群模式**：被 WATCH 的所有 key 必须同 slot（hash tag）
- **不适合高竞争场景**：高并发下重试可能多次失败，CPU 浪费
- **不能 WATCH 永不存在的 key**：监视一个不存在的 key 是合法的——如果该 key 被创建出来就视为修改

### 3.5 WATCH vs Lua

通常 **Lua 优于 WATCH**：

| 维度 | WATCH | Lua |
|---|---|---|
| 原子性 | 乐观锁，可能失败重试 | 强原子，单次成功 |
| 客户端复杂度 | 高（手动重试循环） | 低（一行 EVAL） |
| 网络 RTT | 每次重试 1 个 RTT | 1 个 RTT |
| 主线程压力 | 每个 client 独立排队 | 长脚本可能阻塞主线程 |
| 适用 | 极少冲突 + 简单读改写 | 中等冲突 + 复杂逻辑 |

---

## 第四章：Lua 脚本

### 4.1 为什么用 Lua

```
EVAL "return redis.call('INCR', KEYS[1])" 1 counter
```

服务端执行 Lua 脚本：

- **整个脚本期间独占主线程**（其他 client 命令阻塞等待）
- **完全原子**——脚本里若干 redis.call 之间不可能插入任何其他命令
- **不需要客户端往返**——比"客户端读 + 客户端判断 + 客户端写"省 N 个 RTT
- **可在脚本里做条件逻辑**

### 4.2 一个真实业务：原子扣库存

```lua
-- key: stock:item_id
-- argv: quantity
local stock = tonumber(redis.call('GET', KEYS[1]))
local qty   = tonumber(ARGV[1])
if not stock or stock < qty then
    return 0   -- 库存不足
end
redis.call('DECRBY', KEYS[1], qty)
return 1       -- 成功
```

执行：

```
EVAL "..." 1 stock:item_001 5
```

业务侧只有两种返回：成功 / 失败。**整个过程在 Redis 主线程内串行**，绝不会出现"两个 client 都看到 stock=10 后各扣 8"的超卖。

### 4.3 EVAL vs EVALSHA

EVAL 每次发送脚本字符串；EVALSHA 只发 SHA1：

```
SCRIPT LOAD "return redis.call('INCR', KEYS[1])"
→ "e0e1f9fabfc9d4800c877a703b823ac0578ff831"   ← 脚本的 SHA1

EVALSHA e0e1f9fabfc9d4800c877a703b823ac0578ff831 1 counter
→ (integer) 1
```

服务端缓存了脚本，**避免重复传输脚本本身**——高频脚本省网络。Redis client SDK 一般自动用：

1. 先 EVALSHA，如果 NOSCRIPT 错误（脚本未缓存）
2. 自动 SCRIPT LOAD + 重试 EVALSHA

### 4.4 KEYS 和 ARGV 的区分

**为什么把 key 单独列在 KEYS 数组里**？

- Cluster 模式：Redis 需要知道脚本要访问哪些 key 才能路由
- 安全性：Lua 不能动态算 key 后再访问（避免脚本绕过 cluster 路由）

```
-- 错误：在 Lua 里硬编码 key
EVAL "return redis.call('GET', 'mykey')" 0
-- 在 cluster 下：Redis 不知道访问 mykey，可能路由到错节点

-- 正确：通过 KEYS 传入
EVAL "return redis.call('GET', KEYS[1])" 1 mykey
```

**Cluster 下，KEYS 中所有 key 必须同 slot**——这是 cluster 强制要求。

### 4.5 Lua 与确定性

**默认情况下 Lua 脚本必须确定性**——同样的输入产生同样的输出。原因：脚本会写入 AOF / 复制给 replica，replica 要重放产生同样状态。

不确定的操作：

- `redis.call('TIME')`
- `redis.call('RANDOMKEY')`
- 调用 `math.random()`
- 排序：`redis.call('SMEMBERS', ...)` 返回顺序随机

要在脚本里用 `TIME` 或 `RANDOMKEY`：

```
redis.replicate_commands()   -- 显式声明：本脚本以"命令复制"而非"脚本复制"
-- 此后允许非确定性操作
```

Redis 5.0+ 默认就是 replicate_commands 模式——这条声明已经不必要，但老代码里仍能看到。

### 4.6 Lua 脚本的代价

**长时间脚本阻塞主线程**：

```lua
for i = 1, 1000000 do
    redis.call('SET', 'k:' .. i, 'v')
end
```

跑这种脚本期间整个 Redis 卡住——业务延迟暴增。

防御：

```
lua-time-limit 5000          # 脚本最长 5 秒，超过会被记录但仍继续
```

5 秒后 Redis 不会强行中止脚本（数据完整性优先），但会接受 `SCRIPT KILL` / `SHUTDOWN NOSAVE` 来终止。

**生产铁律**：单脚本应当在毫秒级完成。复杂业务拆 / 用 Streams 异步处理。

---

## 第五章：Functions（Redis 7.0+）

### 5.1 为什么需要 Functions

Lua 的问题：

- 脚本只在 SCRIPT LOAD 时缓存，重启 / failover 后丢——客户端必须每次启动重新 LOAD
- 没有"命名 + 版本"机制——脚本管理混乱
- 没有库的概念——不能复用工具函数

Functions 引入：

- **持久化**：函数定义写入 RDB / AOF，重启不丢
- **复制**：自动复制到所有 replica
- **命名空间**：library + function 名
- **可热部署**：FUNCTION LOAD REPLACE 实时替换

### 5.2 注册和调用

```
FUNCTION LOAD REPLACE "#!lua name=mylib
redis.register_function('addone',
    function(keys, args)
        return redis.call('INCR', keys[1])
    end
)
redis.register_function('addn',
    function(keys, args)
        return redis.call('INCRBY', keys[1], args[1])
    end
)
"

FCALL addone 1 counter          ← 调用 addone，参数：1 个 key
FCALL addn 1 counter 5
```

### 5.3 管理

```
FUNCTION LIST                    # 列出所有 lib
FUNCTION LIST WITHCODE           # 含源码
FUNCTION DUMP                    # 序列化所有 lib（备份用）
FUNCTION RESTORE <payload>       # 从 dump 恢复
FUNCTION DELETE mylib            # 删除 lib
FUNCTION FLUSH                   # 删除所有 lib
FUNCTION STATS                   # 当前运行的 function 状态
```

### 5.4 何时选 Functions 而非 Lua

| 场景 | 推荐 |
|---|---|
| 一次性临时脚本 | Lua（EVAL） |
| 业务核心常驻脚本 | Functions |
| 需要 library 复用 | Functions |
| 团队多人共享脚本 | Functions（版本管理） |

新项目尽量用 Functions。老项目 Lua 仍正常工作，无需强迁。

---

## 第六章：Pipeline

### 6.1 不是事务

Pipeline 是**客户端层**优化：

```python
pipe = r.pipeline(transaction=False)    # 注意：transaction=False
pipe.set("k1", "v1")
pipe.set("k2", "v2")
pipe.incr("counter")
pipe.execute()
```

行为：

- 客户端把 3 条命令一次性发到 server（一个 TCP 包）
- server 顺序执行
- 客户端一次性接收所有响应

**关键差别**：

- **没有 MULTI/EXEC 包裹** → server 端可能在执行第 1 条和第 2 条之间穿插其他 client 命令
- **不保证原子性** → 别的 client 看得到中间状态
- 但**网络往返从 N 次降到 1 次**——性能提升非常大

### 6.2 性能对比

```
单条命令，单机 RTT 1ms：
  100 条独立 GET：100 ms
  100 条 Pipeline：~3 ms（1 RTT + 服务端处理）
```

50-200 命令打包是甜区。打包太多（如 10000+）单次 buffer 占内存大、对单 client 阻塞过久。

### 6.3 Pipeline + MULTI/EXEC

最常见用法：用 Pipeline 包裹一个 MULTI/EXEC：

```python
pipe = r.pipeline()           # 默认 transaction=True
pipe.multi()                   # 显式
pipe.set("k1", "v1")
pipe.incr("counter")
pipe.execute()
```

- 客户端把 `MULTI / SET / INCR / EXEC` 4 条命令在一个 TCP 包发出
- 服务端按 MULTI/EXEC 语义原子执行
- 1 个 RTT + 原子保证

这是日常多操作的标准模式。

### 6.4 Cluster 模式下的 Pipeline

key 跨 slot → 不能一次 pipeline 发——SDK 自动按 slot 拆成多个 pipeline 并发：

```python
# 假设 k1 在 slot A，k2 在 slot B
pipe.set("k1", "v1")
pipe.set("k2", "v2")
pipe.execute()
```

SDK 内部实际是：

- pipeline A → 节点 A：SET k1 v1
- pipeline B → 节点 B：SET k2 v2
- 并发执行 + 收集结果

注意：**单个 pipeline 内已经不是原子的**——k1 写成功不代表 k2 也成功。

---

## 第七章：典型业务模式选择

### 7.1 简单计数 / 累加

```
INCR counter
HINCRBY h field 1
ZINCRBY z 1 member
```

单条命令原子。

### 7.2 条件写

```
SET k v NX           ← 不存在才设
SET k v XX           ← 存在才设
SET k v EX 60 NX     ← 不存在才设 + 60 秒过期
```

不要写 EXISTS + SET 的两步。

### 7.3 限流（令牌桶）

```lua
-- key: ratelimit:user_42
-- argv: max_tokens, refill_rate, request_tokens
local key = KEYS[1]
local max = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local req = tonumber(ARGV[3])
local now = redis.call('TIME')
local now_ms = now[1] * 1000 + now[2] / 1000

local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(bucket[1]) or max
local last = tonumber(bucket[2]) or now_ms

-- 按时间间隔填充令牌
tokens = math.min(max, tokens + (now_ms - last) / 1000 * rate)

if tokens < req then
    return 0
end

tokens = tokens - req
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now_ms)
redis.call('EXPIRE', key, 3600)
return 1
```

### 7.4 排行榜：原子取 top N + 写日志

```lua
local key = KEYS[1]
local member = ARGV[1]
local score = tonumber(ARGV[2])

redis.call('ZADD', key, score, member)
local rank = redis.call('ZREVRANK', key, member)
redis.call('LPUSH', 'changelog', member .. ':' .. score .. ':' .. rank)
return rank
```

### 7.5 秒杀扣库存（最经典）

```lua
local stock_key = KEYS[1]
local order_key = KEYS[2]
local user_id = ARGV[1]

if redis.call('SISMEMBER', order_key, user_id) == 1 then
    return -1   -- 已下单
end
local stock = tonumber(redis.call('GET', stock_key) or 0)
if stock <= 0 then
    return 0    -- 售罄
end
redis.call('DECR', stock_key)
redis.call('SADD', order_key, user_id)
return 1
```

整个流程原子，**单 Redis 实例可承载 10万+ TPS**。

---

## 第八章：生产级最佳实践

1. **能用单命令就别用事务**——Redis 内置组合命令很丰富
2. **多步操作首选 Lua / Functions**——比 MULTI/EXEC 强、比 WATCH 简单
3. **Lua 脚本毫秒级完成**——超时阻塞主线程
4. **Cluster 下 WATCH/Lua 注意同 slot**——用 hash tag
5. **Pipeline + MULTI/EXEC 是日常默认**——1 RTT + 原子
6. **新项目用 Functions 替代 EVAL**——持久化 + 命名空间
7. **EVAL 调用必发 KEYS 列表**——不要在脚本里硬编码 key
8. **业务热点脚本预先 SCRIPT LOAD**——客户端启动时一次加载
9. **lua-time-limit 5000 监控**——慢脚本告警
10. **不要在 Lua 里 fanout 大量命令**——单脚本 < 100 redis.call 是底线

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：以为 MULTI/EXEC 会回滚
不会。每条命令的成败独立。业务要做幂等 + 补偿。

### ❌ 陷阱 2：WATCH 用错对象
WATCH 监视的是 key，不是字段。WATCH hashkey 之后改 hashkey 任一字段都算修改。要细粒度只能 WATCH 多个独立 key。

### ❌ 陷阱 3：Lua 脚本里跑长循环
卡主线程整集群。

### ❌ 陷阱 4：Pipeline 当事务用
默认 redis-py / Jedis 的 pipeline 是 transactional，但 transaction=False 模式下并非原子。

### ❌ 陷阱 5：SCRIPT LOAD 后 failover
新 master 没有这个脚本缓存 → client EVALSHA 收到 NOSCRIPT 错误。SDK 一般自动 fallback 到 EVAL，但 Functions（持久化）更彻底。

### ❌ 陷阱 6：Lua 改了大量 key 但单条 EVAL 走 AOF
脚本即使运行 1ms，AOF 会记录脚本本身——重放时再跑一次。如果脚本影响很大，AOF 体积大涨。

### ❌ 陷阱 7：把同步 client 和 pipeline 混用
某些 SDK 在 pipeline 期间不能复用同一连接做其他操作——会乱序。

### ❌ 陷阱 8：Cluster + 跨 slot 的 EVAL
报 CROSSSLOT。所有访问的 key 用 hash tag 归一 slot。

### ❌ 陷阱 9：EXEC 后没看返回值
`pipe.execute()` 返回的是每条命令的返回值列表。其中某条可能是异常对象——不检查就当成功，业务出错。

### ❌ 陷阱 10：依赖 Lua 里 math.random() 做"幂等"
math.random 在 Lua 里非线程安全 / 不可重现。用业务层生成的 uuid 作为参数传入。

---

## 第十章：练习题

**练习 1**：用一条 Redis 命令实现"如果 key 不存在则设值且 60s 过期"。

**练习 2**：用 Lua 实现"原子下单"：扣库存 + 写入用户订单集合 + 维护今日下单总数。涉及 3 个 key，给出脚本和 EVAL 调用。

**练习 3**：MULTI/EXEC 中一条命令失败，业务层应如何处理（不能用 Lua 重写场景）。

**练习 4**：在 Cluster 模式下，要写一个"原子在两个不同用户间转账"的 Lua 脚本。设计 key 名 + hash tag 让它可行。

**练习 5**：Pipeline 1000 个 SET 命令耗时 5ms；用 100 个 pipeline 各 10 个命令耗时 50ms。解释为什么 batch size 太小性能反而下降。

---

## 参考答案

**练习 1**：

```
SET mykey myvalue EX 60 NX
```

`NX` = not exists 时才设；`EX 60` = 60 秒过期。返回 OK 表示设置成功，nil 表示 key 已存在未设。

**练习 2**：

```lua
-- KEYS[1] = stock:{item_id}
-- KEYS[2] = orders:{item_id}
-- KEYS[3] = orders_count:{today}
-- ARGV[1] = user_id
local stock = tonumber(redis.call('GET', KEYS[1]) or 0)
if stock <= 0 then return -1 end                    -- 售罄
if redis.call('SISMEMBER', KEYS[2], ARGV[1]) == 1 then
    return -2                                       -- 已下过单
end
redis.call('DECR', KEYS[1])
redis.call('SADD', KEYS[2], ARGV[1])
redis.call('INCR', KEYS[3])
return 1
```

调用（key 用 hash tag 同 slot）：

```
EVAL "..." 3 stock:{item_001} orders:{item_001} orders_count:{item_001} user_42
```

实际 orders_count 通常按日维度全局，不能跟 stock 同 slot——这种情况要拆成两次操作或换设计（counter 分散到各分片）。

**练习 3**：

```python
pipe = r.pipeline()
pipe.multi()
pipe.set("k1", "v1")
pipe.incr("k2_should_be_number")  # 假设 k2 是字符串，运行时错
pipe.set("k3", "v3")
results = pipe.execute()

# 检查
for i, r in enumerate(results):
    if isinstance(r, Exception):
        print(f"Command {i} failed: {r}")
        # 决策：补偿（回滚已写）/ 重试 / 抛错
```

业务层要：

1. 遍历 results 检查每个返回值
2. 对失败的命令决定是否回滚之前的（手动调用 DEL k1）
3. 复杂场景重写为 Lua（用 if 判断后再决定是否执行）

**练习 4**：转账涉及两个 user 的余额，必须同 slot：

```
key: balance:{shard0}:user_A
key: balance:{shard0}:user_B
```

用同样的 `{shard0}` 把两个 user 强制到同 slot。但**这样会让某些"碰巧 hash 到同 slot"的 user 永远在同一节点——分片不均匀**。

更合理设计：所有用户余额按 user_id 自然分片，**转账走两阶段提交**（业务层）：

1. EVAL 减 A 的余额（带 transaction log 记录）
2. EVAL 增 B 的余额（带 transaction log）
3. 异步对账修复失败的

或者使用业务上的"账户系统"——余额数据放传统数据库，Redis 仅做读缓存。**金融转账场景，Redis 不是合适的强一致存储**。

**练习 5**：100 个 pipeline 每个 10 命令 → 100 × (1 RTT + 10 命令处理) = 100 × (1ms + 0.1ms) = 110ms 网络 + 10ms 服务端 = 120ms（实际 ~50ms 看 batch 重叠）。

1 个 pipeline 1000 命令 → 1 RTT + 1000 命令处理 = 1ms + ~4ms = ~5ms。

差异核心：**RTT 重复成本**。每个 pipeline = 1 RTT；100 个 pipeline = 100 RTT。即使每个 pipeline 命令少，仍要付 RTT 代价。

合理 batch size：**网络 RTT > 100 × 单命令处理时间** 时合并越多越好。但极大 batch（10000+）会让单 client 长时间占主线程 + buffer 占内存 → 影响其他 client。50-500 是甜区。

---

## 小结

| 机制 | 原子性 | 网络 RTT | 业务复杂度 | 适用 |
|---|---|---|---|---|
| 单条命令 | ✓ | 1 | 低 | 80% 场景 |
| MULTI/EXEC | 部分（无回滚） | 1（Pipeline + MULTI） | 中 | 多个简单写 |
| WATCH + MULTI | 乐观锁，可能重试 | 多次 | 高 | 罕见冲突 + 简单逻辑 |
| Lua / Functions | ✓ 强 | 1 | 中 | 复杂业务原子 |
| Pipeline（无 MULTI） | ✗ | 1 | 低 | 批量独立操作 |

四条铁律：

1. **单命令优先**——Redis 已经把大量原子组合做进去了
2. **MULTI/EXEC 无回滚**——不是真事务，是命令打包
3. **Lua 是真原子**——但脚本必须快
4. **Cluster + 多 key 必同 slot**——hash tag 是工具

下一篇 **R07 — 精通 Redis Streams 与发布订阅**：XADD/XREAD、Consumer Group、Sharded Pub/Sub。
