# 精通 Redis 内存管理与过期回收：maxmemory、eviction、active expire

> 关联章节：[R01 数据结构](./03-精通-Redis-数据结构内部.md)、[R03 持久化](./05-精通-Redis-持久化.md)、[R08 性能模型](./10-精通-Redis-性能模型.md)

---

## 引言："Redis 内存涨上去就不下来" 是怎么回事

很多团队第一次大规模用 Redis 时都会遇到同一个问题：

> 业务高峰过去了，理论上应该回落到几 GB 的内存，但 `INFO memory` 显示仍然 50GB+，曲线就是平的。重启一遍内存确实降下来了，但治标不治本。

要回答这个问题，需要弄清三件事：

1. **Redis 进程占的内存 ≠ 数据本身的内存**——还有元数据、复制 buffer、客户端 buffer、内存碎片
2. **删除一个 key 不一定把内存还给操作系统**——jemalloc 把空闲块挂回内部 freelist，进程 RSS 不减
3. **过期 key 不是到点就消失**——主线程的"主动 + 被动"双策略合力清理，存在延迟

这一章把 maxmemory 控制面、八种 eviction 策略、lazy + active expire、active defrag、内存碎片这一整套机制讲清。

---

## 第一章：Redis 进程的内存到底花在哪

`INFO memory` 输出几十个字段，最关键的几个：

```
used_memory:53687091200             ← Redis 视角"用了多少"（含数据 + 元数据 + lib 开销）
used_memory_rss:64424509440         ← 操作系统视角进程驻留内存
used_memory_peak:67108864000        ← 历史峰值
used_memory_dataset:50000000000     ← 纯数据（去除 overhead）
mem_fragmentation_ratio:1.20        ← rss / used_memory
allocator_allocated:53000000000     ← jemalloc 视角实际分配
allocator_active:55000000000        ← jemalloc 占用的内存页
allocator_resident:60000000000      ← 已映射但部分页可释放
```

四个层次的内存：

```
应用层 (Redis)          used_memory_dataset
   ↓
分配器视角 (jemalloc)   used_memory          (含 overhead)
   ↓
进程视角 (RSS)          used_memory_rss      (含碎片)
   ↓
系统视角 (Cgroup / VM)  RSS + 系统 page cache
```

**`mem_fragmentation_ratio > 1.5` = 内存碎片严重**。常见原因：

- 大量删除 + 重新写入，但新对象大小不一致 → jemalloc 内部很多空闲块拼不起来
- jemalloc 不主动还给操作系统（除非 `MADV_DONTNEED` 触发）
- 大对象（如 1MB string）释放后，那一整页不一定能立即归还

`< 1.0` 则更糟糕——通常说明 OS 在 swap，Redis 性能会断崖式下降。

### 1.1 各类 overhead 的占比

| 来源 | 典型占比 | 说明 |
|---|---|---|
| 数据本身 | 60-80% | 主要部分 |
| dict bucket | 5-10% | `dict_size = 2^N` 桶数组 |
| redisObject 头 | 5-10% | 每个 key + value 一个 redisObject = 16B |
| 客户端 buffer | 1-5% | 每个 client 对应一个 query buffer + output buffer |
| 复制 backlog | 0-5% | `repl-backlog-size`（默认 1MB） |
| AOF buffer | 0-2% | 重写期间额外占用 |

如果你的 Redis 跑 10M key、每 key 16B value，那"数据"是 160 MB，但 `used_memory` 可能 600 MB 起步——剩下都是 overhead。**小 key 的内存放大效应远比你想象的大**。

---

## 第二章：maxmemory 与八种 eviction 策略

### 2.1 maxmemory 是软限制

```
CONFIG SET maxmemory 8gb
CONFIG SET maxmemory-policy allkeys-lru
```

行为：

- 每次写入前检查 `used_memory > maxmemory`？是 → 按 policy 淘汰若干 key → 再写入
- 不是按 `RSS` 判断——所以**实际进程 RSS 可能因为碎片显著超过 maxmemory**
- **没设 maxmemory ≠ 无限**——OS / Cgroup OOM 会直接 kill 进程

### 2.2 八种淘汰策略

| Policy | 候选范围 | 选择标准 |
|---|---|---|
| **noeviction**（默认） | — | 不淘汰，写满了直接返回 OOM 错误 |
| **allkeys-lru** | 所有 key | 最久未访问 |
| **allkeys-lfu** | 所有 key | 访问频次最低 |
| **allkeys-random** | 所有 key | 随机 |
| **volatile-lru** | 设了 TTL 的 key | 最久未访问 |
| **volatile-lfu** | 设了 TTL 的 key | 访问频次最低 |
| **volatile-random** | 设了 TTL 的 key | 随机 |
| **volatile-ttl** | 设了 TTL 的 key | 距过期最近 |

### 2.3 LRU 实现：近似 LRU，不是真 LRU

真 LRU 需要双向链表 + 哈希表，每次访问都把节点移到链表头——**内存与 CPU 代价都不可接受**。

Redis 的做法：

- `redisObject.lru` 字段（24 bit）记录访问时钟（秒级）
- 淘汰时**随机采样 `maxmemory-samples` 个候选**（默认 5），从中选最久未访问的删
- Redis 4.0+ 增强：维护一个**淘汰池**（默认 16），新采样的更老候选会进入池子，每次从池子里删最老的

**采样数越大越精确但 CPU 越高**。生产实务：缓存场景 `maxmemory-samples 10` 是甜区。

### 2.4 LFU 实现：Morris 计数器近似频次

```c
// 每个 redisObject 的 lru 字段 24 bit 在 LFU 模式下拆成：
//   - 高 16 bit: 最后访问时间（分钟，会衰减）
//   - 低  8 bit: counter（对数计数器，0-255）
```

counter 用 [Morris 概率计数器](https://en.wikipedia.org/wiki/Approximate_counting_algorithm)：每次访问按 `1 / (counter * factor + 1)` 概率 +1。counter 达到 255 表示 access > 10M 次（默认配置）。

衰减：每 `lfu-decay-time` 分钟（默认 1）counter -1，防止历史热点永不被淘汰。

LFU 适合**长时间观察热点 key**的缓存场景（如商品详情页）。LRU 在**访问模式高度时间局部性**（如最近活跃用户）的场景更好。

### 2.5 怎么选

| 业务类型 | 推荐 policy |
|---|---|
| 纯缓存（可丢） | `allkeys-lru` 或 `allkeys-lfu` |
| 缓存 + 持久数据混存 | `volatile-lru`（只淘汰带 TTL 的） |
| Session 存储 | `volatile-lru`（短 TTL 自然回收） |
| 持久化数据库（不该丢） | `noeviction`（写满即报错，监控告警） |
| 计数器 / 限流 | `noeviction` + 应用层主动清理 |

**严禁混用**：有些 key 是缓存（应丢）、有些是业务真实数据（不能丢），但 policy 一刀切。解法：**按 key 前缀分到不同 Redis 实例**。

---

## 第三章：过期机制——lazy + active 双策略

### 3.1 TTL 怎么存

`EXPIRE` / `SET key value EX 60` 不立即触发任何事——只是把 (key, expire_at_ms) 写到一个 **`expires` 字典**里。这个字典和主 dict 平行。

Key 真正"消失"的时机：

### 3.2 Lazy expiration（被动过期）

任何**访问**该 key 时（GET / HGET / SISMEMBER / 任何操作），先看 expires 字典——已过期则**立刻删 + 返回 nil**。

优点：CPU 0 浪费（不访问就不检查）。
缺点：**过期 key 不被访问就永远占内存**。

### 3.3 Active expiration（主动过期）

后台周期性扫描 expires 字典找过期 key 删掉。算法：

```
每 100ms 一次 (由 hz 控制，默认 10)：
  循环:
    随机取 20 个带 TTL 的 key
    删除已过期的
    如果删除比例 > 25%，立即重复一轮 (高密度过期)
    否则结束
  每轮最多消耗 25% CPU 时间，超出停止
```

**关键参数**：

```
hz 10               # 频率：每秒 10 次（默认）。延迟敏感场景调 100
maxmemory-policy ...
```

`hz` 越高，过期 key 越快被清理，但主线程开销越大。**生产 hz=10 一般够；TTL 大量集中过期时调到 100**。

### 3.4 验证：手动观察 expired 速度

```
# 写 100 万带 1 秒 TTL 的 key
for i in $(seq 1 1000000); do redis-cli SET k:$i v EX 1; done

# 等 1 秒后查
sleep 2
redis-cli DBSIZE                # 可能仍显示 90 万（active 还没扫完）
redis-cli INFO stats | grep expired_keys
```

`expired_keys` 增长曲线能看到 active 清理速度。**写入"集中过期"是 Redis 主线程毛刺的常见来源**——active 一轮扫到 50% 都过期会连续多轮扫描，挤占业务命令。

### 3.5 解决"集中过期"

**反模式**：业务每天凌晨写一大批 `EX 86400` 的 key → 第二天凌晨同一时刻全部过期 → active expire 一直扫，业务延迟飙升。

**修法**：

```python
# 让 TTL 随机化，把过期时间均匀打散
ttl = base_ttl + random.randint(-300, 300)  # ± 5 分钟
r.set(key, value, ex=ttl)
```

把过期时间散开几分钟即可消除毛刺。

---

## 第四章：UNLINK 与 lazyfree

### 4.1 DEL 的代价

`DEL` 是同步删除：在主线程中递归释放该对象的所有内存。

- string / int：O(1)，几微秒
- 小 hash / list / set / zset（listpack 表示）：O(1)，仍然几微秒（一次性释放整段连续内存）
- **大 hash / list / set / zset（hashtable / skiplist）：O(N)**——10M 元素的 hash `DEL` 可能阻塞主线程 1-5 秒

这是为什么"删大 key" 会出问题。

### 4.2 UNLINK：异步删除

```
UNLINK mykey
```

行为：

- 同步：从主 dict 中 unlink（O(1)）
- 异步：把对象引用扔进一个 lazyfree 队列，由后台线程释放

主线程零阻塞。代价：删除完成有几毫秒到几秒的延迟（取决于对象大小和 lazyfree 线程负载）。

### 4.3 其他 lazyfree 触发点

```
lazyfree-lazy-eviction yes       # 内存淘汰时异步释放
lazyfree-lazy-expire yes         # 主动过期时异步释放
lazyfree-lazy-server-del yes     # 服务端命令导致的删除（如 RENAME、SET 覆盖大 key）
lazyfree-lazy-user-del yes       # 用户的 DEL 命令也走异步
replica-lazy-flush yes           # 从节点全量同步时清空异步
```

**Redis 7.x 默认全部 yes**——这是社区认可的最佳实践。除非有特殊一致性要求，否则照默认走。

### 4.4 后台线程

```
INFO threads
```

Redis 默认有 3 个 bio 后台线程：

- bio_close_file：异步关文件描述符（AOF rewrite 后老文件）
- bio_aof_fsync：AOF fsync 不阻塞主线程
- bio_lazy_free：lazyfree 释放

此外还有一套可选的 I/O 多线程（`io-threads`），仅并行网络读写、默认关闭（`io-threads 1`），与上述 bio 线程是两套独立机制。

详见 R08。

---

## 第五章：内存碎片与 active defrag

### 5.1 碎片怎么产生

```
时刻 1: 写 100 个 1KB 的 key → jemalloc 分配 100 个 1KB 块
时刻 2: 删掉其中 80 个 → jemalloc 把这些 1KB 块挂回 freelist
时刻 3: 写 50 个 2KB 的 key → 1KB freelist 帮不上忙，新申请 OS 内存

结果：进程 RSS 占用 = 时刻 1 的 100KB + 时刻 3 的 100KB = 200KB
      实际数据：(100-80)*1KB + 50*2KB = 120KB
      碎片率：200/120 = 1.67
```

碎片在两种场景特别严重：

1. **写删交替 + 对象大小变化大**——电商秒杀 / 排行榜变更
2. **大 list / hash 元素删除**——quicklist 中间节点删除导致 listpack 缩水

### 5.2 active defrag —— Redis 4.0+ 的主动反碎片

```
CONFIG SET activedefrag yes
CONFIG SET active-defrag-ignore-bytes 100mb        # 总碎片低于此值不启动
CONFIG SET active-defrag-threshold-lower 10        # 碎片率超 10% 启动
CONFIG SET active-defrag-threshold-upper 100       # 碎片率超 100% 全力
CONFIG SET active-defrag-cycle-min 1               # 最小 CPU 占用 %
CONFIG SET active-defrag-cycle-max 25              # 最大 CPU 占用 %
```

机制：主线程周期性把碎片高的内存块"挪到"紧凑位置（用 jemalloc 的 `mallctl` 接口找到内部布局信息）。

**生产实战**：默认关闭。开了之后内存碎片率能持续保持 < 1.2。代价：1-25% CPU 用于碎片整理（自适应）。**对延迟敏感的服务有微小影响**，但绝大多数业务可接受。

### 5.3 手动触发

```
MEMORY PURGE       # 通知 jemalloc 试图把 free 内存还给 OS
```

不解决碎片，但能让 RSS 下来一些。日常监控可定期执行。

### 5.4 终极手段：重启 + replica failover

如果碎片率长期 > 2.0，最干净的方法：

1. 让一个 replica 升级为新 master（数据 fresh，碎片低）
2. 把老 master 重启
3. （新老互换 / 平滑过渡）

这是无停机重启 Redis 的标准做法。

---

## 第六章：Redis 内存监控的关键指标

| 指标 | 阈值 | 含义与处理 |
|---|---|---|
| `used_memory / maxmemory` | < 80% | 健康；>90% 调容量或上 eviction |
| `mem_fragmentation_ratio` | 1.0 - 1.5 | < 1.0 说明 swap，> 1.5 说明碎片严重 |
| `evicted_keys` 增速 | 业务允许范围 | 持续高 = maxmemory 不够 |
| `expired_keys` 增速 | 与业务 TTL 一致 | 突增可能 active expire 在赶工 |
| `keyspace_misses / hits` | 业务相关 | miss 突增 = 缓存命中下降 |
| `connected_clients` | < 10000 | 客户端连接泄漏 |
| `mem_clients_normal` | < 100MB | 单 client output buffer 异常 |
| `mem_clients_slaves` | < repl-backlog-size | 异常时副本复制堆积 |

监控通过 Prometheus + `redis_exporter`，关键 alert：

```
redis_memory_used_bytes / redis_memory_max_bytes > 0.9
redis_memory_fragmentation_ratio > 1.5 for 5m
rate(redis_evicted_keys_total[5m]) > 100
rate(redis_expired_keys_total[5m]) sudden spike > 10x normal
```

---

## 第七章：典型场景调优

### 7.1 纯缓存（命中率优先）

```
maxmemory 8gb
maxmemory-policy allkeys-lru
maxmemory-samples 10
activedefrag yes
hz 10
```

不存 TTL（让 LRU 自然管理）。监控 hit rate；命中率 < 95% 就该扩容或加预热。

### 7.2 Session 存储

```
maxmemory 4gb
maxmemory-policy volatile-lru
maxmemory-samples 5

# 应用代码
SET session:abc data EX 1800     # 30 分钟
```

依赖 TTL 自然过期；维护 volatile-lru 作为兜底。

### 7.3 持久化数据 + 缓存混存

**强烈建议拆 Redis 实例**——不同业务不同 policy。如果非要混存：

```
maxmemory 16gb
maxmemory-policy volatile-lru     # 只淘汰带 TTL 的
```

代码侧约定：

- 缓存 key：必带 TTL
- 业务 key：永不带 TTL

`volatile-lru` 在内存满时只看带 TTL 的 → 业务 key 永远不被误删。

### 7.4 大 key 主动拆分

如果某 hash 元素达 10 万级别：

```
原: huser:1234 = { f1, f2, ..., f100000 }
拆: huser:1234:bucket:0 = { f1..f1000 }
    huser:1234:bucket:1 = { f1001..f2000 }
    ...
```

按 `bucket = hash(field) % 100` 路由读写。带来的好处：

- 单次 HGETALL 不会爆（每 bucket 1000 元素仍是 listpack 表示）
- 单次 UNLINK 释放一个 bucket，主线程零阻塞

详 R10 生产实践。

---

## 第八章：生产级最佳实践

1. **`maxmemory` 必设**——不设等于把 OOM Kill 让给操作系统。
2. **`maxmemory-policy` 按业务选**——纯缓存 LRU；session volatile-lru；强一致业务 noeviction + 容量告警。
3. **`hz=10` 默认够**——TTL 集中过期时调 50-100；调高代价是主线程 CPU 增加。
4. **lazyfree 默认 yes**——别关。
5. **`activedefrag yes`**——99% 业务受益，1% 极延迟敏感的关。
6. **TTL 加随机化**——`±` 5-10% 防集中过期。
7. **`memory usage` + `--bigkeys` 定期巡检**——大 key 是延迟的常见来源。
8. **client output buffer 设上限**——`client-output-buffer-limit normal 0 0 0 / replica 256mb 64mb 60 / pubsub 32mb 8mb 60`，防一个慢客户端把内存打爆。
9. **不要混用业务**——分实例比共享更省心。
10. **碎片率 > 1.5 报警**——长期不下来考虑 failover 重启。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：以为没 OOM 就是健康
碎片率 1.8 时 RSS 实际 = 1.8x 数据。监控看 `used_memory_rss` 而不是 `used_memory`。

### ❌ 陷阱 2：高频写 + 大对象 → 内存不下降
对象大小不一致 → jemalloc free 块挪不齐。开 activedefrag 或定期 failover。

### ❌ 陷阱 3：突然某天 latency 飙升找不到原因
往往是 active expire 撞上"集中过期"。看 `expired_keys` 速率与 latency 曲线对照。修法：TTL 加随机化。

### ❌ 陷阱 4：DEL bigkey
10MB+ 的 key DEL 卡主线程几秒。改 UNLINK 或开 lazyfree-lazy-user-del。

### ❌ 陷阱 5：用 noeviction 但没监控
写满了 Redis 返回 OOM 错误，业务收 exception。除非主动监控，不然就是"一个静默的炸弹"。

### ❌ 陷阱 6：`KEYS *` 在生产
单线程阻塞，几百万 key 时数秒级。改用 `SCAN`。

### ❌ 陷阱 7：以为 maxmemory 限制 RSS
maxmemory 限的是 `used_memory`，RSS = used_memory + 碎片。`maxmemory=8gb`，RSS 涨到 12 GB 完全可能。

### ❌ 陷阱 8：volatile-* 但没 key 有 TTL
策略起不到作用，内存满时 Redis 直接 OOM 错误（行为同 noeviction）。

### ❌ 陷阱 9：client output buffer 不设上限
一个慢 PSUBSCRIBE client 会让该 client 的 buffer 涨到 GB 级。

### ❌ 陷阱 10：persistence + memory 共用同一磁盘
fork 时 COW 触发，加上 BGSAVE 写盘 → IOPS / 内存双爆。生产 dataset 大于内存一半就要小心。

---

## 第十章：练习题

**练习 1**：业务每天 0 点定时写 1000 万条 `EX 86400` 的 key，第二天 0 点又一波写入。怎么避免 active expire 引发的 P99 抖动？

**练习 2**：`mem_fragmentation_ratio = 1.8`、`used_memory = 20GB`、`maxmemory = 30GB`。问：进程实际 RSS 约多少？这种情况下能再写多少业务数据？

**练习 3**：业务 A（缓存，可丢，命中率优先）和业务 B（计数器，绝不能丢）共用同一个 Redis。给一个 maxmemory-policy 配置方案让业务 B 安全，业务 A 充分利用空间。

**练习 4**：测一个 5000 万元素的 hash `DEL` 和 `UNLINK` 的延迟差。写出实验方法。

**练习 5**：解释为什么把 hz 从 10 调到 100 能加速过期清理但增加主线程 CPU 开销，以及对延迟的影响。

---

## 参考答案

**练习 1**：TTL 加随机化：`EX 86400 + random(-3600, 3600)` 把过期点散在 ±1 小时。配合 `hz=20` 加速扫描。监控 expired_keys 速率，保证没有"扎堆"。

**练习 2**：RSS ≈ 20 × 1.8 = **36 GB**。但 maxmemory 限制是 used_memory=30GB 上限，所以再写 10GB 业务数据时 used_memory 到 30 GB，但 RSS 可能到 54 GB——**远超物理内存**。要么扩内存，要么开 activedefrag 把碎片率拉到 1.2 左右。

**练习 3**：

```
maxmemory 16gb
maxmemory-policy volatile-lru
```

业务约定：

- 业务 A 写入 `cache:*` 必带 TTL（`EX 600`）
- 业务 B 写入 `counter:*` 不带 TTL

volatile-lru 只淘汰带 TTL 的 → 业务 B 永远不被误删。监控 `evicted_keys` 全是 cache:*。

**练习 4**：

```python
import redis, time
r = redis.Redis()

# 准备
r.delete("h_del", "h_unlink")
mapping = {f"f{i}": "v" for i in range(5_000_000)}
r.hset("h_del", mapping=mapping)
r.hset("h_unlink", mapping=mapping)

# 测 DEL
t = time.perf_counter()
r.delete("h_del")
print("DEL:", time.perf_counter() - t)

# 测 UNLINK
t = time.perf_counter()
r.execute_command("UNLINK", "h_unlink")
print("UNLINK:", time.perf_counter() - t)
```

典型结果：DEL ~3-8 秒（主线程阻塞），UNLINK <10 毫秒（异步释放）。

**练习 5**：hz=100 表示后台定时任务每秒 100 次（10ms 一次），每次扫 20 个 TTL key。每秒最多扫 2000 个候选，每个轮次 ≤25% CPU。**清理速度**：1000 万过期 key 在 hz=10 下需要 ~500 秒；hz=100 下 ~50 秒。**主线程 CPU**：hz=10 平均 0.5%，hz=100 平均 5%。对单核 CPU 来说，5% 是显著开销，业务命令的 P99 会有 ~1ms 增量。**结论**：常驻 hz=10；TTL 集中过期窗口期临时 `CONFIG SET hz 100` 加速。

---

## 小结

| 控制面 | 关键配置 | 默认 |
|---|---|---|
| 上限 | maxmemory | 0（无） |
| 淘汰策略 | maxmemory-policy | noeviction |
| 采样精度 | maxmemory-samples | 5 |
| 频率 | hz | 10 |
| 过期延迟 | （由 hz + 业务访问驱动） | — |
| 大 key 删除 | lazyfree-* | yes |
| 碎片整理 | activedefrag | no |
| 客户端缓冲限制 | client-output-buffer-limit | 见配置 |

四条铁律：

1. **`used_memory_rss` 才是真实占用**——监控这一个，而非 used_memory
2. **TTL 不是删除时间，是"可被回收的时间"**——清理由 hz + 访问触发
3. **大 key 是延迟杀手**——拆 + UNLINK
4. **碎片率 > 1.5 = 重启信号**——别等业务挂

下一篇 **R03 — 精通 Redis 持久化机制**：RDB / AOF / 混合，fsync 策略与崩溃恢复，bgsave fork 的代价。
