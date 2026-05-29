# 精通 Redis 性能与单线程模型：单线程不慢的真相、I/O 多线程、latency 诊断

> 关联章节：[03 数据结构](./03-精通-Redis-数据结构内部.md)、[04 内存与过期](./04-精通-Redis-内存与过期.md)、[12 生产实践](./12-精通-Redis-生产实践.md)

---

## 引言：100K QPS 的工程师误解

"Redis 单线程怎么可能快？"——这是初学者的第一个困惑。

答案不是"Redis 跑得快是因为内存"——内存只是必要条件。**真正的快秘诀是设计上避开了所有传统数据库的慢源**：

1. **无磁盘 I/O 在主路径**（持久化用 fork 子进程）
2. **无锁**（单线程串行处理，天然原子）
3. **数据结构选择极度务实**（R01 讲过的 listpack / skiplist / dict）
4. **协议极简**（RESP，几字节开销）
5. **事件循环 + IO 多路复用**（epoll / kqueue）

结果：**单核 100k QPS**。再快不靠多核（仍是单线程逻辑），靠 6.0+ 的 **I/O 多线程**——把 socket 读写 offload 给多线程，逻辑仍单线程。

这一章把性能模型从根讲透。读完之后你应该能：

- 解释 epoll 事件循环的一次 tick
- 区分"单线程"和"单进程"——后台线程在干什么
- 知道何时 I/O 多线程能帮你（很少）何时不能（多数）
- 用 `latency monitor` + `slowlog` 定位一次 P99 飙升
- 给"Redis 慢"做一份诊断 checklist

---

## 第一章：单线程模型

### 1.1 主线程在干什么

```c
// src/server.c 主循环（极简化）
while (running) {
    aeProcessEvents();   // 处理事件
}

aeProcessEvents() 内部：
  1. 处理所有"到时间的定时任务" (TimeEvent)
     - active expire 扫一批 TTL key
     - 心跳 / 客户端超时检查
     - 后台监控更新
  2. 调用 epoll_wait 等待 I/O 事件
  3. 对每个 ready fd：
     - 读：解析 RESP 命令 → 执行 → 写返回到 client output buffer
     - 写：把 output buffer 发到 socket
```

**所有命令处理都在这个循环里串行**——没有锁，没有并发冲突，没有上下文切换开销。

### 1.2 为什么单线程能 100k QPS

| 单条命令耗时 | 单核每秒能处理 |
|---|---|
| GET / SET（小 value） | ~1-2 微秒 | 500k-1M QPS 上限 |
| HGET / HSET | ~2-3 微秒 | 300k-500k QPS |
| ZRANGE 10 个元素 | ~5-10 微秒 | 100k-200k QPS |
| EVAL 简单 Lua | ~10-20 微秒 | 50k-100k QPS |
| KEYS *（10w key） | ~50 ms | **6 QPS**（卡死所有 client） |

只要每条命令足够快，单线程顺序处理就够。**问题不在并发能力，在于"长命令会卡住所有 client"**。

### 1.3 单线程的代价：长命令是杀手

```
client A: KEYS *               ← 50ms 卡住
client B: GET k1               ← 等
client C: SET k2 v             ← 等
...
clients D-Z 都得等
```

50ms 不长，但**所有 client 都得排队**——P99 看起来就是 50ms+。这是单线程模型的核心代价。

**长命令清单（必须避免）**：

- `KEYS *` → 用 `SCAN`
- `SMEMBERS` / `HGETALL` / `LRANGE 0 -1` 大集合 → 用 `SSCAN` / `HSCAN` / 分页
- `SORT` 大 set → 改 ZSet
- `FLUSHALL` / `FLUSHDB` → 用 `FLUSHALL ASYNC`
- `DEBUG SLEEP`（测试用）→ 生产严禁
- `DEBUG OBJECT` 在 GB 级 key 上 → 仍然 O(N)
- 大 key 的 `DEL` → 用 `UNLINK`
- 长 Lua 脚本 → 拆 / 异步化

---

## 第二章：后台线程

### 2.1 单进程 ≠ 单线程

Redis **进程**单线程是指**主事件循环单线程**。但服务端实际有多个**后台线程**：

```
INFO threads
```

输出（Redis 7+）：

```
io_threads_active:0          ← I/O 多线程是否启用
bio_thread_aof:1             ← AOF fsync 线程
bio_thread_lazy_free:1       ← lazyfree 释放线程
bio_thread_file_close:1      ← 关文件描述符线程
```

加上：

- BGSAVE / BGREWRITEAOF 时 fork 出的**子进程**
- 部分场景的辅助线程（如 module 的 worker）

### 2.2 bio_aof_fsync 的精妙

R03 提到 `appendfsync everysec` 怎么做到对主线程零阻塞：

```
主线程：write() 到 AOF buffer → 立刻返回 client
                  ↓
        定时任务每秒触发：通知 bio_aof_fsync 线程
                  ↓
              bio_aof_fsync 线程：fsync(aof_fd)
                  ↓
                  fsync 慢 → 不阻塞主线程
```

**fsync 慢但业务不感知**——核心机制。

### 2.3 bio_lazy_free

R02 提到大 key 异步释放：

- 主线程从 dict 中 unlink（O(1) 修改指针）
- 异步：bio_lazy_free 线程做真正的内存释放（遍历 hash/list 元素逐个 free）

主线程零阻塞。代价：内存"延迟"几毫秒到几秒才真正归还 jemalloc。

### 2.4 fork 子进程

BGSAVE / BGREWRITEAOF：

- fork 系统调用本身阻塞主线程几十-几百 ms（页表复制）
- 子进程独立做磁盘 I/O，主线程继续服务
- **业务期间业务命令对内存的修改触发 COW**

详 R03。

---

## 第三章：I/O 多线程（Redis 6.0+）

### 3.1 解决什么问题

主事件循环单线程做：

1. **socket I/O**：读 client 命令 + 写返回到 client
2. **命令解析**（RESP 协议）
3. **命令执行**（数据结构操作）
4. **命令回写**

实测：**对小命令而言，1+2+4（I/O + 协议）占 50-70% 时间，3（execute）只占 30-50%**。

I/O 多线程把 **1+4 offload** 给多个 worker 线程：

```
主线程：从 epoll 拿到 ready fd 集合
       → 把 fd 分给多个 IO 线程
                     → 各 IO 线程 read socket 到 buffer
       ← 各 IO 线程返回（all done barrier）
       主线程串行解析 + 执行所有命令（这部分仍单线程）
       → 把回复 buffer 分给多个 IO 线程
                     → 各 IO 线程 write socket
       ← all done barrier
```

**执行仍单线程**——所以"无锁、无并发问题"的核心保持不变。

### 3.2 配置

```
io-threads 4                  # 启用 4 个 IO 线程（含主线程，所以 worker=3）
io-threads-do-reads yes       # 读也用多线程（默认 no，只多线程写）
```

`io-threads-do-reads` 开了才真正能加速读密集场景。

### 3.3 实测收益

| 场景 | 单线程 QPS | io-threads=4 QPS | 增益 |
|---|---|---|---|
| 小 SET / GET 高并发 | 100k | 200-300k | 2-3x |
| 大 value（10KB） | 30k | 50-80k | 1.7-2.7x |
| Pipeline（10 cmd / batch） | 500k | 600-700k | 1.2-1.4x |
| 单 client 顺序 | 80k | 80k | 0 |

**适合**：客户端连接数多 + 命令简单 + 网络成为瓶颈。**不适合**：少量连接、Pipeline 已经用得很好、CPU 成为瓶颈（命令本身复杂）。

### 3.4 io-threads 数怎么选

- 4 核机器 → `io-threads 2-3`
- 8 核机器 → `io-threads 4`
- 多于 6 → 增益递减（同步开销变大）

不是越多越好。

---

## 第四章：网络协议 RESP

### 4.1 RESP2 与 RESP3

RESP (REdis Serialization Protocol) 是 Redis 的应用层协议。

```
Client → Server:
*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n

Server → Client:
+OK\r\n
```

`*3` = 数组长度 3；`$3` = 字符串长度 3；`+OK` = 简单字符串成功。

**RESP2**（默认，Redis 6 之前的唯一版本）：
- 5 种类型：simple string、error、integer、bulk string、array
- 类型贫乏，map / set 等高级类型靠客户端再解析

**RESP3**（Redis 6+ 引入）：
- 15 种类型（= 5 个 RESP2 类型 + 10 个 RESP3 新增类型）：map、set、bool、double、null、verbatim string、attribute（带元数据的响应）
- Push 类型支持 invalidation 通知

### 4.2 RESP3 的客户端开关

```
HELLO 3 [AUTH user pass]
→ % 服务端切到 RESP3 与该 client 通信
```

RESP3 优点：

- **Hash 直接是 map**：`HGETALL` 返回 client-端 dict，无需再解析
- **客户端缓存**：`CLIENT TRACKING ON` + RESP3 让服务端推送 invalidation 给 client
- **Streams 数据结构化**：XRANGE 返回带类型的嵌套结构

不是所有 client 都支持 RESP3。Lettuce、redis-py 6+、go-redis v9 等都支持。Jedis 旧版本仅 RESP2。

### 4.3 客户端缓存

```
# Client A
CLIENT TRACKING ON              # 开启
GET foo                         # 拿到值，本地缓存

# Client B（另一连接）
SET foo new_value
→ Server 主动给 Client A push 一条 "foo invalidated" 消息

# Client A 收到 push → 删除本地缓存
```

**LRU 之上又加一层应用本地缓存**——延迟可降到亚微秒（不必再发 RTT）。适合**少量热点 key + 高读频次**。

---

## 第五章：连接与 client buffer

### 5.1 每个 client 占多少内存

- TCP 缓冲区（kernel 管理，~几 KB - 几十 KB）
- query buffer：当前 client 待解析的命令缓冲（< 1MB 通常）
- output buffer：服务端待发给 client 的响应（可大可小）

`INFO clients`：

```
connected_clients:1500
client_recent_max_input_buffer:8192     ← 最大单 client query buffer
client_recent_max_output_buffer:65536   ← 最大单 client output buffer
```

### 5.2 client-output-buffer-limit

```
client-output-buffer-limit normal   0    0    0        # 普通 client 无上限
client-output-buffer-limit replica  256mb 64mb 60      # replica：硬上限 256MB；软上限 64MB 持续 60 秒
client-output-buffer-limit pubsub   32mb  8mb  60      # pubsub 订阅者
```

某个慢 client 来不及消费 → output buffer 涨 → 触发硬 / 软上限 → 服务端**强断这个连接**。

**关键场景**：

- **慢 replica**：复制流写入太快 replica 处理慢 → 触发 limit → 复制断 → 全量重同步 → 雪崩。生产 replica buffer 应当**调大**或保持默认（256MB 已不小）
- **PSUBSCRIBE 大流量 channel**：订阅者处理慢 → 断连。**别把高频事件 pub 到不处理它的 subscriber**

### 5.3 max-clients

```
maxclients 10000     # 默认 10000
```

注意：每个 client 在 OS 是一个 fd → `ulimit -n` 也要相应调大（默认 1024 显然不够）。生产 `ulimit -n 65535`。

---

## 第六章：latency monitor

### 6.1 框架

```
CONFIG SET latency-monitor-threshold 100    # 阈值 100ms
```

主线程每次执行某些"可能慢"的操作前后会记录时间。超过阈值就记录一条 latency event。

事件类型（部分）：

- `command` / `fast-command`：某条命令超时
- `event-loop`：单轮事件循环超时
- `aof-write` / `aof-fsync-always` / `aof-fstat`
- `rdb-unlink-temp-file`
- `expire-cycle`：active expire 一轮
- `fork`：fork 系统调用
- `unlink`：UNLINK 异步释放慢

### 6.2 查看

```
LATENCY LATEST          # 各事件最近一次发生时间和值
LATENCY HISTORY event   # 该事件历史
LATENCY DOCTOR          # 人话报告
LATENCY RESET           # 清零
```

`LATENCY DOCTOR` 输出例：

```
Dave, I have observed latency spikes in this Redis instance.

1) High AOF fsync latency: 130 ms. The OS is unable to fsync your AOF
   in less than 130 ms. Suggestions: faster disk; appendfsync everysec or no.

2) Big "fork" event: 250 ms. fork is slow on large memory data sets.
   Tips: disable transparent huge pages; reduce dataset size; use a 
   replica for persistence.
```

直接给修法建议——**首次排查必跑**。

---

## 第七章：slowlog

### 7.1 配置

```
slowlog-log-slower-than 10000      # 10ms（微秒，默认）
slowlog-max-len 128                # 保留最近 128 条
```

任何执行时间超过阈值的命令都被记录到内存中的 slowlog。

### 7.2 查看

```
SLOWLOG GET 10        # 最近 10 条慢命令

1) 1) (integer) 14                       ← 唯一 ID
   2) (integer) 1715587200               ← 时间戳
   3) (integer) 52000                    ← 耗时（微秒）= 52ms
   4) 1) "KEYS"
      2) "*"
   5) "127.0.0.1:54321"                  ← client addr
   6) "myapp-worker-3"                   ← client name（如设了）

SLOWLOG LEN
SLOWLOG RESET
```

### 7.3 实战

slowlog + latency monitor 是诊断 P99 飙升的标准两件套：

1. `SLOWLOG GET 10` 看是否有具体慢命令
2. `LATENCY DOCTOR` 看是否有系统级事件（fork、fsync 等）
3. 两者都正常但延迟还高 → 看网络 / 客户端

---

## 第八章：典型性能瓶颈

### 8.1 主线程 CPU 100%

排查：

```
INFO cpu
used_cpu_sys: ...
used_cpu_user: ...
```

可能原因：

- QPS 超过单核能力 → 扩容到 Cluster
- 命令复杂（大 ZRANGE、大 SORT）→ 优化数据结构
- Lua 脚本长 → 拆 / 异步化
- active expire 频繁（hz=100）→ 调回 10

### 8.2 内存碎片高

`mem_fragmentation_ratio > 1.5`：

- 开 `activedefrag yes`
- 重启 / failover 切换

### 8.3 fork 慢

`latest_fork_usec > 500000`：

- 关 THP
- 让 replica 做持久化，master 关 save + AOF
- 缩小 dataset

### 8.4 网络瓶颈

```
INFO stats | grep -E "net_|total_net_"

total_net_input_bytes
total_net_output_bytes
instantaneous_input_kbps
instantaneous_output_kbps
```

带宽接近 NIC 上限（1Gbps ≈ 100 MB/s，10Gbps ≈ 1 GB/s）→ 扩容到多 NIC / Cluster。

### 8.5 持久化 I/O 抖动

- AOF rewrite 抖动 → 错峰到业务低峰
- 磁盘共享导致 fsync 慢 → 独享磁盘 / SSD

---

## 第九章：基准测试

### 9.1 redis-benchmark

```
redis-benchmark -h host -p 6379 -t set,get -n 1000000 -c 50 -d 100

-t set,get          # 测哪些命令
-n 1000000          # 总请求数
-c 50               # 并发 client 数
-d 100              # value 大小 100 字节
-P 16               # pipeline batch 16
```

典型输出：

```
SET: 250000.00 requests per second
GET: 280000.00 requests per second
```

注意：`redis-benchmark` 默认所有 client 用同 key——**没法真实反映你的业务模式**。建议加 `-r 1000000` 让 key 随机化。

### 9.2 业务级压测

更可靠的是用 [`memtier_benchmark`](https://github.com/RedisLabs/memtier_benchmark) 或自己写脚本模拟真实业务。

---

## 第十章：生产级最佳实践

1. **永远不要在生产 KEYS \***——用 SCAN
2. **大 key 拆 + UNLINK**——避免主线程 O(N) 操作
3. **`latency-monitor-threshold 100`**——常驻开启，几乎零开销
4. **`slowlog-log-slower-than 10000`**——10ms 阈值
5. **`io-threads 4`** + **`io-threads-do-reads yes`**——多客户端业务下显著加速
6. **`maxclients 10000`** + **`ulimit -n 65535`**——配套调整
7. **client-output-buffer-limit 别误调小**——尤其 replica
8. **关 THP**——fork 性能必备
9. **持久化让 replica 做**——大 master 零 fork 抖动
10. **加 Prometheus + redis_exporter**——CPU / 内存 / latency / slowlog 一并监控

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：单线程理解错
单线程指主事件循环；后台线程（bio_*）和 fork 子进程都是独立的。

### ❌ 陷阱 2：开 io-threads 但单 client
单 client 顺序请求场景 io-threads 完全无帮助。增益要看并发 client 数。

### ❌ 陷阱 3：MONITOR 在生产
`MONITOR` 让 Redis 把每条命令推送给 monitor client → 性能下降 50%+。**生产严禁，调试用**。

### ❌ 陷阱 4：DEBUG SLEEP 测试残留
测试代码忘删除，生产 `DEBUG SLEEP 5` 把所有 client 卡 5 秒。

### ❌ 陷阱 5：高频 INFO 命令
`INFO` 不便宜（要遍历 dict 统计）。监控系统每秒 INFO 一次会占主线程 1-5% CPU。改 5-10 秒间隔。

### ❌ 陷阱 6：CLIENT LIST 大量 client
1 万 client 时 `CLIENT LIST` 自身就要几十 ms。运维查看时主线程被卡。

### ❌ 陷阱 7：KEYS pattern 比 KEYS * 还多
`KEYS user:*` 仍要扫所有 key。SCAN 才是正解。

### ❌ 陷阱 8：在 Lua 里写循环 + redis.call
每次 redis.call 都同步等待执行。100 次 INCR 在 Lua 里跟独立 100 次 INCR 时间相近——但卡住主线程。

### ❌ 陷阱 9：连接池 size 设太大
1000 连接 × 100 个微服务 = 10 万。Redis 单实例 maxclients=10000 直接连不上。统一连接池规模或拆 Redis。

### ❌ 陷阱 10：用 PSUBSCRIBE pattern 实际匹配全集
`PSUBSCRIBE *` 会让每条 PUBLISH 都推给该 subscriber。流量翻倍。

---

## 第十二章：练习题

**练习 1**：业务 P99 持续 200ms，QPS 50k，主线程 CPU 70%。给出诊断步骤。

**练习 2**：解释为什么 `io-threads 4` 在"单 client 用 Pipeline 100 命令"场景下几乎无加速。

**练习 3**：客户端 reports `LOADING Redis is loading the dataset in memory`。可能原因 + 处理。

**练习 4**：用 `LATENCY DOCTOR` 定位以下故障各一次：fork 慢、AOF fsync 慢、大 key UNLINK。写出场景。

**练习 5**：连接数从 1000 涨到 10000 后 Redis QPS 反而下降。可能原因？

---

## 参考答案

**练习 1**：

1. `SLOWLOG GET 20` 看是否有慢命令——如有 → 优化具体命令
2. `LATENCY DOCTOR` 看系统事件——fork / fsync / expire-cycle
3. `INFO commandstats` 看哪类命令占 CPU 多——可能 EVAL / 大 ZRANGE
4. `CLIENT LIST` 看是否有异常大 client（query/output buffer 大）
5. 看是否在 AOF rewrite 期间（INFO persistence: aof_rewrite_in_progress=1）
6. 看是否网络瓶颈（带宽 / NIC 包数饱和）
7. 70% CPU @ 50k QPS = 单核 70%。空间还有 → 没到上限；考虑 io-threads 提升吞吐

**练习 2**：Pipeline 100 命令通常 1-2 个 RTT 即完成，**socket I/O 工作量小**。io-threads 优化的是"多客户端并发 socket"——单 client 单连接的 I/O 工作交给 1 个 io-thread 没意义，反而引入了线程同步开销。

**练习 3**：Redis 启动 / failover 后正在加载 RDB+AOF 到内存。期间拒绝业务命令。处理：
- 看 `INFO persistence: loading_total_bytes / loading_loaded_bytes` 看进度
- 等加载完成（大 dataset 可能几分钟）
- 应用层 retry + 健康检查
- 频繁重启的话考虑用 replica failover 替代重启（fresh master）

**练习 4**：
- **fork 慢**：场景：50GB dataset + THP 开 + 频繁 BGSAVE。LATENCY DOCTOR 报 `fork: 1200ms`。修法：关 THP、缩小 dataset、让 replica 做持久化
- **AOF fsync 慢**：场景：HDD 共享盘 + `appendfsync always`。报 `aof-fsync-always: 200ms`。修法：换 SSD / `everysec`
- **大 key UNLINK**：场景：删 10GB hash。报 `unlink: 500ms`。修法：UNLINK 仍走主线程 unlink 步（虽然实际释放异步）；改成业务侧拆 key 后再 UNLINK

**练习 5**：

- **每 client 占主线程时间**：连接管理 / 协议解析有固定开销。10k 连接互相切换主线程的 I/O 处理增加
- **output buffer 累积**：慢 client 多了，主线程花更多时间检查 buffer 限制
- **TCP 缓冲争用**：内核管理 1 万个 socket 比 1000 个慢
- **maxclients 接近上限**：每接受新连接需要扫描 client 列表
- **客户端没用连接池**：每次请求重新 TCP 握手 + auth → 主线程花更多时间处理握手
- 解法：限制连接数（每应用统一连接池）、开启 io-threads、拆 Cluster 分散连接

---

## 小结

| 优化层 | 工具 / 配置 | 适用 |
|---|---|---|
| 单条命令快 | 选对数据结构、避免 O(N) | 90% 优化点 |
| 网络 I/O | io-threads、Pipeline、RESP3 | 高并发场景 |
| 持久化 | replica 做 / 关 THP / SSD | 大 dataset |
| 内存 | activedefrag、UNLINK、lazyfree | 长跑系统 |
| 诊断 | latency monitor、slowlog、INFO | 必备 |
| 扩展 | Cluster 分片 | 单核到顶后 |

四条铁律：

1. **单线程指主事件循环**——不要因为它"慢"
2. **长命令是延迟杀手**——清单要熟记
3. **fsync / fork 是抖动来源**——隔离到 replica
4. **诊断从 LATENCY DOCTOR + SLOWLOG 开始**——别瞎猜

下一篇 **R09 — 精通 Redis 8 新数据类型**：JSON、TimeSeries、Bloom 系列、Vector Set。
