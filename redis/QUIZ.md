# Redis 深度课程 · 自测题库

> 配合 [INDEX.md](./INDEX.md) 使用。每章 ~10 题，含答案与展开解析。
> 难度标记：⭐ 概念 ⭐⭐ 进阶 ⭐⭐⭐ 源码级 / 容易踩坑

> 答题建议：先盖住答案自答，再展开核对。能讲出"为什么"比答对结论更重要。

---

## R01 数据结构内部

### Q1.1 ⭐⭐ embstr 与 raw 的切换阈值是多少？为什么是这个数？

<details><summary>答案</summary>

**44 字节**。embstr 把 `redisObject` 头（16 字节）+ SDS 头（3 字节）+ 字符串内容 + 末尾 `\0` 一次性紧凑分配，配合 jemalloc 64 字节的内存块刚好。`16 + 3 + 44 + 1 = 64`。超过 44 就要单独 malloc SDS，变成 raw。

44 不是写死的逻辑，是为 jemalloc bin 大小量身定制的；改 allocator 时这个数会跟着变。

</details>

### Q1.2 ⭐⭐⭐ listpack 取代 ziplist 的根本原因是什么？

<details><summary>答案</summary>

**Cascading update（连锁更新）**。ziplist 每个 entry 头里记"前一个 entry 的长度"，当某个 entry 变大让长度字节数从 1 变到 5，会把后一个的"前长度"也撑大，可能再撑下一个，O(N²) 最坏复杂度。

listpack 把每个 entry 改成"自包含"：自己的总长度写在 entry 末尾，遍历仍可双向，但**没有任何字段引用前一个 entry 的长度**。一改只改自己，O(1)。

</details>

### Q1.3 ⭐⭐ Hash 类型在什么阈值下从 listpack 切换为 hashtable？

<details><summary>答案</summary>

两个阈值任一触发：

- `hash-max-listpack-entries 512`：字段数 > 512（Redis 7.x/8.x 默认；6.x 时代为 128）
- `hash-max-listpack-value 64`：任一字段名或值长度 > 64

切换是**单向**的：listpack → hashtable 后即使删字段也不退回。

</details>

### Q1.4 ⭐⭐⭐ Sorted Set 同时维护 skiplist 和 dict 的目的是什么？

<details><summary>答案</summary>

- skiplist：按 score 排序，支持范围查询 `ZRANGE` / `ZRANGEBYSCORE` O(log N + M)
- dict：按 member 查 score `ZSCORE` O(1)

两个结构的 member 字符串实际**指向同一份**（共享指针），不重复存。

为什么不用 B-tree？Redis 作者的论点：B-tree 的常数大、节点合并 / 分裂复杂；skiplist 实现 100 行内、范围查询同样 O(log N)、对 cache 不利但 Redis 是内存数据库本来就走内存。

</details>

### Q1.5 ⭐⭐ quicklist 是什么？

<details><summary>答案</summary>

List 类型的实现：**双向链表 + 每个节点是一个 listpack**。通过 `list-max-listpack-size` 控制每个节点 listpack 大小（默认 -2 即 8KB）。

避免了纯链表"每元素一个节点"的指针开销，又避免了纯 listpack 大量元素时插入要整体复制的开销。是工程妥协的典范。

</details>

### Q1.6 ⭐ intset 什么时候用？

<details><summary>答案</summary>

Set 类型当且仅当**所有元素都能解析为整数**且**元素数 ≤ `set-max-intset-entries`（默认 512）**时使用。一旦插入字符串就立刻升级为 listpack 或 hashtable。

</details>

### Q1.7 ⭐⭐⭐ Stream 用什么数据结构？

<details><summary>答案</summary>

**Radix tree**，每个叶子节点指向一个 listpack（小批 entry 的紧凑存储）。这样：

- ID 按时间顺序写入 → radix tree 按前缀压缩，索引高效
- 单 listpack 容纳多 entry → 减少树节点数与指针开销
- 范围扫描 / `XRANGE` 走 radix tree O(log N)

Consumer Group 状态独立用 listpack + radix tree 存 PEL（Pending Entry List）。

</details>

### Q1.8 ⭐⭐ HyperLogLog 占多少内存？为什么固定？

<details><summary>答案</summary>

**12 KB**。HLL 用 `m = 16384` 个 6-bit 桶估计基数，`16384 * 6 / 8 = 12288` 字节，加 header 约 12 KB。

容量上限约 `2^64`，标准误差约 0.81%。占用与基数大小无关——这是 HLL 的核心价值：**用固定 12 KB 估计任意大小的去重基数**。

</details>

### Q1.9 ⭐⭐ SDS 比 C 字符串好在哪？

<details><summary>答案</summary>

- O(1) 长度查询（不用扫到 `\0`）
- 二进制安全（中间可有 `\0`）
- 预分配 + 惰性释放，append 不每次 realloc
- 5 种 header 类型按串长度选最小（sdshdr5/8/16/32/64），节省内存
- 兼容 C：末尾仍补 `\0`，可传给 strcmp / printf

</details>

### Q1.10 ⭐⭐⭐ 为什么 Redis 7.0 把 ziplist 全面换成 listpack 之后，hash / set / zset 的 listpack-entries 默认值得以提高？

<details><summary>答案</summary>

ziplist 时代默认 64，因为 cascading update 在大 ziplist 上更危险。listpack 杜绝了 cascading update，可以安全用更大的紧凑结构 → 7.0 起 hash 默认调到 512（zset/set 仍为 128），**让更多 hash 留在 listpack 形态、内存更省**。

</details>

---

## R02 内存与过期

### Q2.1 ⭐⭐ Redis 8 种 maxmemory-policy 各自适用场景？

<details><summary>答案</summary>

| Policy | 行为 | 场景 |
|---|---|---|
| noeviction | 写满即报错 | 不能丢数据的纯 KV |
| allkeys-lru | 所有 key 中淘汰 LRU | 通用缓存 |
| allkeys-lfu | 所有 key 中淘汰 LFU（访问频次） | 热点明显的缓存 |
| allkeys-random | 全部随机淘汰 | 访问无规律 |
| volatile-lru | 仅有 TTL 的 key 淘汰 LRU | 缓存 + 持久化数据共存 |
| volatile-lfu | 同上但 LFU | 同上 + 热点明显 |
| volatile-random | 仅 TTL 的随机淘汰 | 同上 + 无规律 |
| volatile-ttl | TTL 最短的优先淘汰 | 有时序的缓存 |

LFU 在 Redis 4.0 引入，需要内存近似计数（Morris counter）。

</details>

### Q2.2 ⭐⭐⭐ Redis 的"近似 LRU"是什么意思？

<details><summary>答案</summary>

Redis 不维护全局 LRU 链表（代价太高）。每次淘汰时**随机采样 N 个 key**（`maxmemory-samples`，默认 5），从中淘汰最旧的。N 越大越接近真 LRU 但 CPU 越高。

LFU 同理：每次采样后比频次。

</details>

### Q2.3 ⭐⭐ Redis 的过期清理用了几种机制？

<details><summary>答案</summary>

两种叠加：

- **lazy expiration**：访问 key 时检查 TTL，过期则删
- **active expiration**：定时任务（每秒 hz=10 次）每次随机抽 20 个 TTL key，过期的删；如果过期比例 > 25% 就再抽一轮，直到 < 25% 或超时

两者结合保证既不漏（active）又不浪费 CPU（lazy 等访问时再删）。

</details>

### Q2.4 ⭐⭐ 设了 TTL 但 `INFO` 看到大量 expired key 没被清，是什么情况？

<details><summary>答案</summary>

正常。`expired_keys` 统计**已经被清掉的累计数**。如果你看 `INFO keyspace` 里 `expires=N`，那 N 是当前还有 TTL 的 key 数（可能已过期但未清）。

如果担心积压，可以：

- 调高 hz（active expire 频率）
- 业务侧主动 GET 触发 lazy expire
- 用 `SCAN` + `TTL` 巡检

</details>

### Q2.5 ⭐⭐ 怎么排查 Redis 内存碎片？

<details><summary>答案</summary>

```
INFO memory
# used_memory:1000000000        ← 实际数据
# used_memory_rss:1500000000    ← 进程 RSS
# mem_fragmentation_ratio:1.5   ← 碎片率
```

大于 1.5 算高。开 `activedefrag yes` 后台整理（CPU 代价 ~10%）。低于 1 反而是数据被 swap 到磁盘，更严重。

</details>

### Q2.6 ⭐⭐⭐ `EXPIRE k -1` 与 `PERSIST k` 的区别？

<details><summary>答案</summary>

- `EXPIRE k -1`（或任何 ≤ 0 的值）：**立即删除** key
- `PERSIST k`：移除 TTL，key 变永久

</details>

### Q2.7 ⭐⭐ jemalloc 比 glibc malloc 在 Redis 上好在哪？

<details><summary>答案</summary>

- 更小的碎片率（per-thread arena + size class）
- 提供 `je_mallctl` 等接口让 Redis 主动整理碎片（active defrag 依赖）
- 特定 size 命中固定 bin，分配更快

Redis 在 Linux 上自 2.4 起默认编译即用 jemalloc（非 Linux 平台默认 libc malloc）。

</details>

### Q2.8 ⭐⭐ `MEMORY USAGE key` 是怎么算的？

<details><summary>答案</summary>

递归遍历 key 和它的 value 数据结构，估算每个 entry 的内存占用 + 共享对象不重复算。`SAMPLES N` 控制抽样大集合时的精度；`SAMPLES 0` 全扫。

注意：**不算 dict bucket 等"开销"**，只算 entry 本身。所以加起来通常比 `used_memory` 少。

</details>

### Q2.9 ⭐⭐⭐ MEMORY DOCTOR 可能告诉你哪些事？

<details><summary>答案</summary>

- 高碎片率
- maxmemory 接近上限
- 大量 client output buffer（慢消费者）
- AOF 重写期内存峰值
- 主从复制 backlog 占用

是结合 `INFO memory` / `INFO clients` 等的简易诊断。

</details>

### Q2.10 ⭐⭐ `lazyfree-lazy-eviction yes` 干什么？

<details><summary>答案</summary>

evicition 时不在主线程释放大对象（O(N)），而是把对象交给 bio 线程异步释放。主线程立刻返回。

类似的有 `lazyfree-lazy-expire`（过期惰释）、`lazyfree-lazy-server-del`（DEL 大 key 自动转 UNLINK）、`lazyfree-lazy-user-flush`（FLUSHALL/DB 异步）。

生产建议：**全开**。

</details>

---

## R03 持久化

### Q3.1 ⭐⭐ RDB / AOF / 混合 三者怎么选？

<details><summary>答案</summary>

- **纯 RDB**：可以丢几分钟数据；快照小、恢复快；fork 子进程做 BGSAVE
- **纯 AOF**：不丢或秒级丢；文件大、恢复慢（要回放所有命令）
- **混合（推荐）**：`aof-use-rdb-preamble yes`，AOF rewrite 时先写 RDB 再追加增量 AOF。**恢复快 + 数据安全**

纯缓存且重启可接受丢数据：两者都关。

</details>

### Q3.2 ⭐⭐ AOF 三种 fsync 策略的差别？

<details><summary>答案</summary>

| 策略 | fsync 时机 | 性能 | 丢数据 |
|---|---|---|---|
| `always` | 每条命令 | 极慢（磁盘 IOPS 限） | 0 |
| `everysec`（默认） | 每秒一次（bio 线程） | 几乎无影响 | 最多 1s |
| `no` | 由 OS 决定 | 最快 | 系统页缓存刷盘前都丢 |

99% 场景用 `everysec`。

</details>

### Q3.3 ⭐⭐⭐ BGSAVE 是 fork 子进程，为什么不会复制全部内存？

<details><summary>答案</summary>

Linux **copy-on-write (COW)**：fork 后父子共享物理页，只有写时才复制对应页。Redis 主进程在 BGSAVE 期间继续接收写命令时，被改的页才独立。

所以 BGSAVE 的"内存峰值"取决于该期间的**写量**，而不是数据量。如果几乎没写，峰值 ≈ 原内存；写得多，最坏会 2x。

</details>

### Q3.4 ⭐⭐ AOF rewrite 是怎么压缩的？

<details><summary>答案</summary>

子进程遍历当前内存数据，把每个 key 用**最少命令**重新表达（例如不再有过期的 SET，重复的 INCR 合并为一个 SET 终值），写到新 AOF 文件。期间主进程的新写命令进 **AOF rewrite buffer**，rewrite 完后追加到新文件。

混合模式下子进程写的是 RDB 头 + 重写 buffer 的 AOF 增量。

</details>

### Q3.5 ⭐⭐ 同时开 RDB 和 AOF，重启时 Redis 加载哪个？

<details><summary>答案</summary>

**优先 AOF**（更新更频）。混合模式下 AOF 文件本身前半是 RDB，后半是增量 AOF，统一加载。

</details>

### Q3.6 ⭐⭐⭐ AOF 文件 truncated（机器突然断电）能恢复吗？

<details><summary>答案</summary>

`aof-load-truncated yes`（默认）会忽略最后不完整的命令直接启动。如果设 no 则启动失败需要 `redis-check-aof --fix` 修复。

</details>

### Q3.7 ⭐⭐ 持久化期间 latency 飙升一般是哪个阶段？

<details><summary>答案</summary>

- **fork**：内存大时 fork 本身要复制页表，大内存时几百毫秒到秒级 → `LATENCY HISTORY fork-event`
- **AOF fsync**：磁盘慢时 bio 线程 fsync 慢 → 写 buffer 排队，主线程被 backpressure
- **AOF rewrite**：子进程 + COW 内存 + 新文件写盘 IO 抢占

降低办法：用 SSD、关 THP（Transparent Huge Page）、调度 BGSAVE 时段。

</details>

### Q3.8 ⭐⭐ 关 THP 为什么重要？

<details><summary>答案</summary>

THP 让内核合并 4KB 页为 2MB 大页。fork 后 COW 是按页拷贝的——拷一个 2MB 比拷一个 4KB 慢 500x。Redis 启动会警告：

```
WARNING: you have Transparent Huge Pages (THP) support enabled
```

关法：

```
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

</details>

### Q3.9 ⭐⭐ `appendfsync everysec` 时 bio 线程在做什么？

<details><summary>答案</summary>

主线程 write 到 AOF buffer 后立即返回。bio 线程每秒一次 fsync 这个 buffer 落盘。如果 bio 线程 fsync 还没完，主线程下一秒又来 write 时**会先 wait**（避免 buffer 越堆越大）。这就是"everysec 在磁盘慢时仍可能阻塞主线程"的原理。

</details>

### Q3.10 ⭐⭐⭐ Redis 多机房部署，远程从库 AOF 用 always 行吗？

<details><summary>答案</summary>

不行。always 在跨机房磁盘上 RT 几十 ms，吞吐崩。要么本地 AOF + 异步复制到远程，要么用 Redis Enterprise / Active-Active 类商业方案。

</details>

---

## R04 复制与 Sentinel

### Q4.1 ⭐⭐⭐ PSYNC2 比 PSYNC1 强在哪？

<details><summary>答案</summary>

PSYNC1：从库重连后只能用 `runid + offset` 续传，且**主库换 runid 就只能全量**。
PSYNC2：

- **从库晋升为主库**后保留原 replication ID 作为 `replid2`，旧从库连过来时仍能 partial resync
- 多了 `secondary replication ID` 的概念，主从切换时不必全量

简单说：PSYNC2 让"failover 后旧从库不必全量重同步"成为可能。

</details>

### Q4.2 ⭐⭐ repl-backlog 是什么？多大合适？

<details><summary>答案</summary>

主库内存里的环形 buffer，记录最近一段时间的命令。从库断连重连后用 `offset` 在 backlog 里找位置续传。如果 backlog 已经被覆盖（断太久 / backlog 太小），就只能全量同步。

大小：`repl-backlog-size 1mb` 默认，**生产改 128MB-1GB**，按主库每秒写入字节数 × 预期最长断连秒数。

</details>

### Q4.3 ⭐⭐ Sentinel 至少要几个节点？

<details><summary>答案</summary>

**3 个**。Sentinel 选举主观下线 → 客观下线 → 故障转移要 quorum。3 个能容忍 1 个挂；2 个挂掉 1 个就剩 1 没法 quorum，等于无 HA。

5 个能容忍 2 个挂，更稳。

</details>

### Q4.4 ⭐⭐⭐ Sentinel 的 `quorum` 与 `majority` 是同一个概念吗？

<details><summary>答案</summary>

不是。

- **quorum**：`sentinel monitor mymaster <ip> <port> <quorum>` 中的数字，多少个 Sentinel 说"客观下线" 就触发 failover 选举
- **majority**：Sentinel 内部进行 leader 选举（哪个 Sentinel 真去做 failover）的多数派 = `floor(N/2)+1`

举例：5 个 Sentinel，quorum=2 → 2 个就能宣告主库下线，但 leader 选举仍需 3 个同意。可以 quorum < majority，但 failover 仍要多数派同意。

</details>

### Q4.5 ⭐⭐ split-brain 在 Sentinel 部署里怎么发生？

<details><summary>答案</summary>

主库被网络隔离（仍在跑），少数派 Sentinel 跟主库一起；多数派 Sentinel 选了新主。客户端有的连旧主，有的连新主，**两边都在写**。网络恢复后旧主降级为从，**它收到的写丢失**。

防御：`min-replicas-to-write N` + `min-replicas-max-lag M` —— 主库发现可写从库少于 N 或延迟过大就拒写。

</details>

### Q4.6 ⭐⭐ 主从复制延迟怎么看？

<details><summary>答案</summary>

```
INFO replication
# master_repl_offset:12345678
# slave0:offset=12345670, lag=0

# 从库执行
INFO replication
# master_link_status:up
# master_last_io_seconds_ago:0
# master_sync_in_progress:0
```

差值 `master_repl_offset - slaveN_offset` 就是延迟字节数。

</details>

### Q4.7 ⭐⭐⭐ `replica-read-only no` 配从库可写有什么用？

<details><summary>答案</summary>

允许在从库本地写**仅本地可见的临时数据**（不会被复制到主库或其他从库），偶尔用于本地缓存计算结果。**绝大多数场景不要开**——容易产生主从数据不一致 + 客户端误用。

</details>

### Q4.8 ⭐⭐ Sentinel 故障转移流程？

<details><summary>答案</summary>

1. 主观下线：单个 Sentinel ping 不到主库
2. 客观下线：quorum 数量的 Sentinel 都说不到 → 进入 failover 流程
3. Sentinel leader 选举（Raft-like）
4. leader 选 promotion 候选（健康度 + offset 最大 + 优先级最低）
5. 向候选发 `REPLICAOF NO ONE` → 升级
6. 让其他从库 `REPLICAOF <new-master>`
7. 通过 Pub/Sub 通知客户端切换

整个流程 10-30 秒。

</details>

### Q4.9 ⭐⭐ `replica-priority 0` 是什么？

<details><summary>答案</summary>

该从库**永不被 Sentinel 选为新主**。常用于"只读分担流量"的副本、或专用于做备份的副本。

</details>

### Q4.10 ⭐⭐⭐ 从库一启动就 `Loading the dataset in memory`，主库这边压力大吗？

<details><summary>答案</summary>

新从库连主库 → 主库 BGSAVE → 把 RDB 发给从库（占带宽）→ 从库 loading 期间不响应客户端。主库压力来自：

- BGSAVE fork → 内存峰值
- 网络带宽（RDB 大时几个 GB）
- 期间累积的写命令进 backlog（如果 backlog 满，从库 loading 完后又得全量）

**生产实践**：避免业务高峰期初始化新从库；在低峰期挂入；用更大的 backlog。

</details>

---

## R05 Cluster

### Q5.1 ⭐⭐ 为什么是 16384 slot？

<details><summary>答案</summary>

作者选定的常数。考量：

- gossip 协议每个节点要在心跳里发自己的 slot bitmap：16384 bit = 2KB，可接受；如果 65536 就 8KB，每秒几十次心跳带宽吃紧
- 16384 / 1000 节点 ≈ 16 slot/节点，仍有足够分片粒度
- Redis Cluster 设计上限就 1000 节点上下，更多 slot 没意义

</details>

### Q5.2 ⭐⭐⭐ MOVED 与 ASK 的本质区别？

<details><summary>答案</summary>

- **MOVED**：slot 已经永久迁到目标节点。客户端**更新本地 slot 表**，以后这个 slot 都直接发给新节点
- **ASK**：slot 正在迁移中，本次该 key 在目标节点。客户端**不更新本地表**，发到目标前先发 `ASKING` 取得"特例许可"

简单说：MOVED 是"搬完"，ASK 是"搬一半"。

</details>

### Q5.3 ⭐⭐ hash tag 怎么用？

<details><summary>答案</summary>

key 名包含 `{tag}` 子串时，slot 计算只用 `{}` 内的部分：

```
user:{1001}:profile   →  hash("1001") % 16384
user:{1001}:settings  →  同上 → 同 slot
order:{1001}          →  同上 → 同 slot
```

让相关数据落同 slot，可做 multi-key 操作（MGET / Lua / pipeline 优化）。

慎用：tag 设计不当会造成单 slot 过热（所有用户 1001 数据集中一节点）。

</details>

### Q5.4 ⭐⭐⭐ Cluster 怎么扩容？

<details><summary>答案</summary>

```
redis-cli --cluster add-node new:7006 existing:7000
redis-cli --cluster reshard existing:7000
# 交互：迁多少 slot、从谁迁、迁去哪个节点
```

底层每个 slot 迁移流程：

1. 目标节点 `CLUSTER SETSLOT slot IMPORTING source-id`
2. 源节点 `CLUSTER SETSLOT slot MIGRATING dest-id`
3. 源节点循环：`CLUSTER GETKEYSINSLOT slot N` → `MIGRATE dest-host port "" 0 timeout KEYS k1 k2 ...`
4. 全部迁完后所有节点 `CLUSTER SETSLOT slot NODE dest-id`

迁移期间该 slot 的 key 部分在源、部分在目标，靠 ASK 路由。

</details>

### Q5.5 ⭐⭐ Cluster 客户端拿不到拓扑会怎样？

<details><summary>答案</summary>

启动时连接 seed nodes，发 `CLUSTER SHARDS` / `CLUSTER SLOTS` 拿拓扑。如果 seed nodes 全挂客户端启动失败。运行中收到 MOVED 也会触发刷新。

**生产实践**：seed nodes 至少 3 个，分别在不同 master 上。

</details>

### Q5.6 ⭐⭐⭐ Cluster 没法保证强一致是为什么？

<details><summary>答案</summary>

主从异步复制 + failover 切换。可能场景：

- 主库已确认写但还没复制到从库 → 主库挂了 → 从库晋升 → 数据丢
- 网络分区 → 旧主仍接受写 → 恢复后旧主成从 → 写丢

Cluster 本质是 **AP**：可用性优先。

</details>

### Q5.7 ⭐⭐ `CLUSTER FAILOVER` 与 `CLUSTER FAILOVER FORCE` 区别？

<details><summary>答案</summary>

- `CLUSTER FAILOVER`：副本主动切换，**先与主库同步 offset 一致后才切**，零数据丢失（需要主库可达）
- `CLUSTER FAILOVER FORCE`：副本不等同步直接切（主库不可达时用）
- `CLUSTER FAILOVER TAKEOVER`：副本不通过其他主库投票直接 takeover（极端场景如多数派挂掉）

</details>

### Q5.8 ⭐⭐⭐ Cluster 中的 Lua / multi-key 命令限制？

<details><summary>答案</summary>

所有访问的 key 必须在**同一 slot**（要么同名 prefix 要么用 hash tag）。否则报 `CROSSSLOT Keys in request don't hash to the same slot`。

`MULTI / EXEC` 同样要求所有 key 同 slot。

</details>

### Q5.9 ⭐⭐ gossip 协议在 Cluster 里干什么？

<details><summary>答案</summary>

每秒每个节点向若干随机节点发心跳（PING），心跳里携带：

- 自己的 slot 分布
- 自己看到的其他节点状态（pfail / fail）
- runid + offset

接收方通过聚合多源信息更新本地视图。slot 表变化通过 gossip 自动扩散到全集群（秒级）。

</details>

### Q5.10 ⭐⭐⭐ "fail over"集群少数派分区，那一边的客户端会怎样？

<details><summary>答案</summary>

少数派分区里的主库继续接受写，但**因为 `cluster-require-full-coverage yes`（默认）+ 多数派失联，会变 cluster 状态 fail**，拒绝服务。少数派客户端的所有写都失败。等多数派恢复，少数派的主库会被多数派否决（如果它仍是 master），数据被丢弃。

如果配 `cluster-require-full-coverage no` + `cluster-allow-replica-migration no` 等可让局部继续工作，**但代价是数据一致性更难保证**。

</details>

---

## R06 事务、Pipeline、脚本

### Q6.1 ⭐⭐ MULTI/EXEC 中某条命令运行时报错，已执行的命令会回滚吗？

<details><summary>答案</summary>

**不会**。Redis 的 MULTI 没有回滚——只保证"中间不被其他客户端命令插入"。运行时错误（如 `INCR` 一个非数字 key）后续命令仍按顺序执行，已执行的不撤销。

语法错误（命令名拼错）会让整个事务在 EXEC 时拒绝执行。

</details>

### Q6.2 ⭐⭐⭐ WATCH 实现的"乐观锁"原理？

<details><summary>答案</summary>

`WATCH key` 后服务端记录该 key 的 version。`EXEC` 时如果该 key 在 WATCH 之后被任何客户端修改过，事务**整体放弃**返回 nil。客户端检测到 nil 后重试整个流程。

类似 CAS。适合"读 → 改 → 写"且竞争少的场景；竞争激烈时用 Lua 更高效。

</details>

### Q6.3 ⭐⭐ Pipeline 与 MULTI/EXEC 区别？

<details><summary>答案</summary>

- Pipeline：客户端层批量发送，**不保证原子**，命令之间可能被其他客户端插入
- MULTI/EXEC：服务端整体执行，**保证中间不插入**，但本身不保证错误回滚

要又快又原子：Lua 脚本（独占服务端单线程）。

</details>

### Q6.4 ⭐⭐ Lua 脚本执行期间会阻塞别的客户端吗？

<details><summary>答案</summary>

会。脚本期间 Redis 主线程独占。所以**脚本要短**，长脚本拆成多次调用 + 业务层重试。

`SCRIPT KILL` 可以杀长脚本（前提是脚本未做过任何写）。

</details>

### Q6.5 ⭐⭐⭐ EVAL 和 EVALSHA 的差别与组合用法？

<details><summary>答案</summary>

- EVAL 每次发脚本全文 → 浪费带宽
- EVALSHA 发 SHA1（40 字符）→ 服务端 SCRIPT cache 命中即执行；不命中报 `NOSCRIPT`

标准模式：

```python
sha = r.script_load(script)   # 一次注册
try:
    r.evalsha(sha, ...)
except NoScriptError:
    sha = r.script_load(script)
    r.evalsha(sha, ...)
```

主流 SDK 自动做这个回退。

</details>

### Q6.6 ⭐⭐⭐ Functions（7.0+）比 Lua 好在哪？

<details><summary>答案</summary>

- 命名 + 持久化（写到 RDB/AOF），重启不丢
- 通过 `FUNCTION LOAD` 一次部署多个相关函数 + library
- 复制时随数据走，副本自动有
- 版本管理 / 替换 / 删除有原子原语
- 后续可能支持其他语言（不只 Lua）

适合"应用 + Redis 紧密耦合的逻辑"，不适合临时一次性脚本。

</details>

### Q6.7 ⭐⭐ 事务里 WATCH 的 key 如果被同一连接的 MULTI 自己改了，会触发吗？

<details><summary>答案</summary>

不会。WATCH 只检测**其他连接**的修改。

</details>

### Q6.8 ⭐⭐ Cluster 下能用 MULTI/EXEC 跨 slot 吗？

<details><summary>答案</summary>

不能，所有 key 必须同 slot。Lua 同样。要跨节点原子需要应用层补偿（如 Saga）。

</details>

### Q6.9 ⭐⭐⭐ Lua 里能调 `redis.call` 一个会失败的命令吗？

<details><summary>答案</summary>

- `redis.call('INCR', 'non-int-key')` → 抛异常 → 整个脚本中止
- `redis.pcall(...)` → 失败返回 error table，脚本可继续

设计上要明确想要哪种行为。

</details>

### Q6.10 ⭐⭐⭐ `SCRIPT FLUSH` 影响范围？

<details><summary>答案</summary>

**清空当前节点的 SCRIPT cache**。所有客户端的 EVALSHA 都会 NOSCRIPT 一次（直到重新 SCRIPT LOAD）。Cluster 里要分别在每个 master 上 FLUSH。

ASYNC 选项（7.0+）可异步清，避免阻塞。

</details>

---

## R07 Streams 与 Pub/Sub

### Q7.1 ⭐⭐ Pub/Sub 与 Streams 关键区别？

<details><summary>答案</summary>

| | Pub/Sub | Streams |
|---|---|---|
| 持久化 | 无 | 有（RDB/AOF） |
| Consumer 不在线 | 消息丢 | 消息留下次读 |
| ack | 无 | XACK |
| 回溯 | 无 | XRANGE 任意 ID 起读 |
| 消费组 | 无（订阅同 channel 的全部都收到） | XREADGROUP 一条消息组内只有一个消费者收到 |
| 适用 | 即时通知、广播 | 消息队列、事件流 |

</details>

### Q7.2 ⭐⭐⭐ Stream 的 ID 格式是什么？

<details><summary>答案</summary>

`<毫秒时间戳>-<序号>`，例如 `1715600000000-0`。

- 默认 `XADD stream * ...` 的 `*` 让服务端自动生成 ID（保证递增）
- 同一毫秒多条 → 序号递增（0, 1, 2, ...）
- 客户端可指定 ID（必须比当前最大 ID 大），用于幂等

`MINID` / `MAXLEN` 修剪用 ID 或长度作上限。

</details>

### Q7.3 ⭐⭐ XREADGROUP 里 ID 用 `>` 与具体 ID 的区别？

<details><summary>答案</summary>

- `>`：只读"未分配给任何 consumer"的新消息
- 具体 ID（如 `0`）：读这个 consumer 自己的 PEL（pending 但未 ack 的）从该 ID 起

`>` 用于正常消费，具体 ID 用于"重启后 reclaim 自己之前没 ack 完的"。

</details>

### Q7.4 ⭐⭐ Consumer 挂了，它的 PEL 怎么办？

<details><summary>答案</summary>

`XPENDING stream group` 看所有 consumer 的 PEL；`XCLAIM stream group new-consumer min-idle-ms id1 id2 ...` 把消息转交给新 consumer。

```
XAUTOCLAIM stream group new-consumer 5000 0 COUNT 100
```

更方便：自动 claim 闲置 > 5000ms 的消息给 new-consumer。

</details>

### Q7.5 ⭐⭐⭐ Sharded Pub/Sub（7.0+）解决什么问题？

<details><summary>答案</summary>

普通 Pub/Sub 在 Cluster 里**全集群广播**——一条 PUBLISH 要发给所有节点检查订阅，节点多了浪费带宽。

`SPUBLISH / SSUBSCRIBE` 让消息按 channel 名 hash 到固定 slot/节点，**只在该 shard 里广播**。订阅者必须连那个 shard。

</details>

### Q7.6 ⭐⭐ Stream 的 MAXLEN 和 MINID 修剪？

<details><summary>答案</summary>

```
XADD stream MAXLEN ~ 1000 * field value   # 大约保留 1000 条（~ = 近似剪）
XADD stream MAXLEN 1000 * field value     # 严格 1000 条
XADD stream MINID ~ <id> * field value    # 删除 ID 小于该值的
```

`~` 让 Redis 在 listpack 边界处剪，效率高但实际可能多保几条。无 `~` 严格剪有性能代价。

</details>

### Q7.7 ⭐⭐ Pub/Sub 的客户端 output buffer 配错会怎样？

<details><summary>答案</summary>

`client-output-buffer-limit pubsub 32mb 8mb 60`：硬限 32MB / 软限 8MB 持续 60 秒 → 服务端断开慢消费者。

如果消费者处理慢、消息又多，连接被反复 kill → Pub/Sub 不稳。要么降低发布速率，要么换 Streams。

</details>

### Q7.8 ⭐⭐⭐ Streams 比 Kafka 强 / 弱在哪？

<details><summary>答案</summary>

**强**：

- 单进程，部署简单
- 内存级 latency
- 与 Redis 其他数据结构原子组合（Lua 里 XADD + INCR）

**弱**：

- 单 stream 单节点容量受限（Redis Cluster 不能跨 slot）
- 持久化耐用度不如 Kafka（强依赖 fsync 策略）
- 没 schema、没 partitioning（要分用多个 stream）
- 监控生态没 Kafka 成熟

适合中小规模（< 100MB/s 吞吐）+ 已有 Redis 的场景。

</details>

### Q7.9 ⭐⭐ 同一 group 同一 consumer 读两次 `>` 会怎样？

<details><summary>答案</summary>

第一次拿到一批消息进入 PEL，第二次再读 `>` 会拿到**新的一批**（不重复）。如果想重读已 PEL 的，用具体 ID。

</details>

### Q7.10 ⭐⭐⭐ Stream 怎么实现"恰好一次"语义？

<details><summary>答案</summary>

业务侧自己实现幂等。Stream 本身是"至少一次"——consumer 处理完没 ack 就崩，重启后 reclaim 仍会再来一次。

幂等手段：

- 消息体里带业务 ID，DB 唯一索引
- 消费方先 `SETNX dedupkey` 判重
- 用 message ID 作幂等 key

</details>

---

## R08 性能模型

### Q8.1 ⭐⭐ Redis 单线程为什么能 100k QPS？

<details><summary>答案</summary>

- 内存访问 ns 级，单条命令 1-2 微秒
- epoll 多路复用，无需为每客户端开线程
- 无锁串行处理，无上下文切换
- 数据结构选择极致务实

100k QPS = 单核 1 微秒/命令的极限附近。

</details>

### Q8.2 ⭐⭐⭐ Redis 6.0 的 I/O 多线程实际加速哪部分？

<details><summary>答案</summary>

只把 **read 解析 RESP** 与 **write 响应** offload 到多线程，**命令执行仍单线程**。

适用场景：单条命令小、连接多、网络成为瓶颈（如大量 GET/SET）。
**不**适用：CPU 受限于命令执行（大 hash / 复杂 Lua），加 IO 线程没用。

打开：

```
io-threads 4
io-threads-do-reads yes
```

通常 4-8 线程够。

</details>

### Q8.3 ⭐⭐ "单进程" ≠ "单线程"，后台还有什么线程？

<details><summary>答案</summary>

- bio 线程：fsync AOF / lazyfree / 关 fd
- I/O 多线程（如启用）
- 模块自己起的 worker
- BGSAVE / BGREWRITEAOF 时的子进程（不是线程）

</details>

### Q8.4 ⭐⭐⭐ slowlog 的实现？

<details><summary>答案</summary>

每次命令执行前后取时间戳，差值超过 `slowlog-log-slower-than`（默认 10000 微秒 = 10ms）就插入到环形 buffer（`slowlog-max-len` 默认 128）。

`SLOWLOG GET 10` 看最近 10 条。`SLOWLOG RESET` 清空。

注意：**只算执行时间，不含网络 / 排队**。

</details>

### Q8.5 ⭐⭐ latency monitor 与 slowlog 区别？

<details><summary>答案</summary>

- slowlog：单条命令耗时
- latency monitor：**事件级**延迟，包括 fork、aof-fsync、expire-cycle、evictioncycle 等

```
LATENCY DOCTOR              # 自动诊断
LATENCY HISTORY fork-event  # 查 fork 事件历史
LATENCY RESET
```

</details>

### Q8.6 ⭐⭐ KEYS 为什么是杀手命令？

<details><summary>答案</summary>

O(N) 扫整个 keyspace，单线程串行 → 卡死所有客户端。10w key ~ 几十 ms，1000w key ~ 几秒。生产严禁。用 SCAN 替代。

</details>

### Q8.7 ⭐⭐⭐ SCAN 怎么保证不漏不重？

<details><summary>答案</summary>

cursor 用反向二进制位递增（不是简单 +1），扫描期间 dict rehash 也能保证：

- **不漏**：rehash 期间同时遍历两个 table
- **不重**：cursor 设计让已访问位置不会再次访问

但**两次 SCAN 之间新增的 key 可能漏掉**（这是设计上接受的）。

</details>

### Q8.8 ⭐⭐ MONITOR 命令什么时候能用？

<details><summary>答案</summary>

只用于**调试**——它会让 Redis 把每条命令都发到这个连接的客户端，**性能掉 50%+**。生产严禁开。如果非要看流量用 `slowlog` / `LATENCY` / 客户端侧采样。

</details>

### Q8.9 ⭐⭐⭐ THP 关闭和 vm.overcommit_memory=1 是为什么？

<details><summary>答案</summary>

- THP：fork 时 COW 大页拷贝慢（详 Q3.8）
- `vm.overcommit_memory=1`：Linux 默认严格 overcommit，BGSAVE fork 可能因为"申请的虚拟内存 > 物理"被拒。设 1 让 fork 通过，COW 实际用到再分配

启动时 Redis 都会警告这两项没设。

</details>

### Q8.10 ⭐⭐ 一次 P99 飙升的诊断顺序？

<details><summary>答案</summary>

1. SLOWLOG GET 10 → 有没有慢命令
2. LATENCY DOCTOR → 有没有 fork / fsync / expire 事件
3. INFO clients → 连接数 / blocked_clients
4. INFO memory → maxmemory / 碎片率
5. INFO replication → 复制是否正常
6. dmesg / OS：CPU / 网卡 / 磁盘是否瓶颈

业务侧同步看：客户端池利用率、客户端 RT 直方图。

</details>

---

## R09 Redis 8 新数据类型

### Q9.1 ⭐⭐ Redis 8 集成进 core 的 8 种新类型？

<details><summary>答案</summary>

1. JSON
2. TimeSeries
3. Bloom Filter
4. Cuckoo Filter
5. Count-Min Sketch
6. Top-K
7. T-Digest
8. Vector Set

之前是 RedisStack 模块，Redis 8 起直接编译进核心。

</details>

### Q9.2 ⭐⭐ JSON 类型与 String 存 JSON 字符串区别？

<details><summary>答案</summary>

- String 存 JSON：每次改字段都要 GET 全文 → 改 → SET 全文，浪费带宽 + 非原子
- JSON 类型：服务端理解 JSON 结构，`JSON.SET` / `JSON.GET path` / `JSON.NUMINCRBY` 直接操作字段，部分更新

```
JSON.SET user:1 $ '{"name":"alice","age":30}'
JSON.GET user:1 $.name
JSON.NUMINCRBY user:1 $.age 1
```

</details>

### Q9.3 ⭐⭐ TimeSeries 用什么场景？

<details><summary>答案</summary>

- 监控指标存储
- IoT 时序数据
- 事件计数

特性：

- DOWNSAMPLE：自动降采样到长期保留
- LABELS：多维度查询
- 压缩：Gorilla 编码

```
TS.CREATE temperature LABELS sensor 1 location china
TS.ADD temperature * 22.5
TS.RANGE temperature - +
```

</details>

### Q9.4 ⭐⭐⭐ Bloom Filter 与 Cuckoo Filter 区别？

<details><summary>答案</summary>

- Bloom：插入 + 查；不能删
- Cuckoo：插入 + 查 + **删除**；空间利用率更高（~95% vs Bloom ~70%）；但插入可能因 cuckoo eviction 失败

需要删用 Cuckoo，不需要用 Bloom（实现更简单）。

</details>

### Q9.5 ⭐⭐ Count-Min Sketch 解决什么问题？

<details><summary>答案</summary>

近似计数：在固定内存里估计任意 key 的出现次数。永远 **overestimate（高估）**，不会低估。适合：

- 找 top-K 频次词
- 去重前的初筛
- 流量统计

```
CMS.INITBYDIM cms 2000 5
CMS.INCRBY cms item1 5
CMS.QUERY cms item1
```

</details>

### Q9.6 ⭐⭐⭐ T-Digest 比直方图好在哪？

<details><summary>答案</summary>

T-Digest 估计任意 quantile（如 P99 / P999）误差极小。直方图固定桶位，对极端 quantile 估计误差大。

```
TDIGEST.CREATE latency
TDIGEST.ADD latency 5.0 1 7.0 1 ...
TDIGEST.QUANTILE latency 0.99
```

适合 SLA 监控、长尾延迟分析。

</details>

### Q9.7 ⭐⭐ Vector Set 干什么？

<details><summary>答案</summary>

存浮点向量 + ANN（近似最近邻）查询。Redis 8 直接支持，做向量数据库 / RAG 的轻量替代。底层 HNSW 图，默认 int8 量化（Q8）。

```
# 添加：VADD key [REDUCE dim] (FP32 | VALUES num) vector element
VADD myindex VALUES 4 0.1 0.2 0.3 0.4 doc1
VADD myindex VALUES 4 0.5 0.6 0.7 0.8 doc2

# 搜索：VSIM key (ELE | FP32 | VALUES num) (vector | element)
VSIM myindex VALUES 4 0.1 0.2 0.3 0.4 COUNT 10 WITHSCORES
VSIM myindex ELE doc1 COUNT 10
```

不如 Milvus / Qdrant 专业，但已经够小规模 / 已有 Redis 的场景用。

</details>

### Q9.8 ⭐⭐ Top-K 与 Sorted Set 做 top-N 区别？

<details><summary>答案</summary>

- Sorted Set：精确 top-N，但存全部元素，内存随基数线性增长
- Top-K：固定内存，近似 top-K，适合千万 / 亿基数

```
TOPK.RESERVE topk 10 2000 7 0.925
TOPK.ADD topk item1 item2 ...
TOPK.LIST topk
```

</details>

### Q9.9 ⭐⭐⭐ 这些新类型在持久化里怎么存？

<details><summary>答案</summary>

集成到 RDB / AOF 的核心后，每个类型有自己的 RDB encoding 与 AOF 重写策略，**透明持久化**。这是从模块升级为 core type 的关键好处之一。

模块时代要分别加载模块、持久化兼容性脆弱。

</details>

### Q9.10 ⭐⭐ 用了新类型后能切回 Valkey 吗？

<details><summary>答案</summary>

**不能**直接切。Valkey 不内置这些类型（截至 8.x），要继续用对应模块（社区 fork 版）。如果数据用了新类型 → 迁移要导出 → 转格式 → 导入 Valkey 模块版，复杂。

如果有跨产品迁移可能：尽量只用 Redis 经典数据类型。

</details>

---

## R10 生产实践

### Q10.1 ⭐⭐ bigkey 阈值大概是多少？

<details><summary>答案</summary>

经验值（生产）：

- string > 10KB
- hash 字段 > 1000 或单字段 > 10KB
- list / set / zset 元素 > 5000

不是硬性，根据业务调。`redis-cli --bigkeys` 扫描发现。

</details>

### Q10.2 ⭐⭐⭐ 一个 50GB 的 hash bigkey 怎么不停服拆？

<details><summary>答案</summary>

1. 双写：业务侧改写代码，**同时写老 hash 和新分桶 hash**
2. 后台脚本：分批 HSCAN 老 hash → 写到新分桶
3. 切读：业务读改成"先读新桶，没有再读老 hash"
4. 全量切：等读没回退到老 hash → 删除老 hash（用 UNLINK 异步）

期间老 hash 仍服务，新写按双写规则同步。

</details>

### Q10.3 ⭐⭐ 缓存击穿怎么解决？

<details><summary>答案</summary>

热点 key 过期瞬间，大量请求穿透到 DB。

- **互斥锁**：第一个请求加分布式锁去 DB 加载，其他等
- **永不过期 + 后台异步刷新**
- **逻辑过期**：key 永不真过期，value 里带"逻辑过期时间"，到期后第一个发现的请求异步刷新

</details>

### Q10.4 ⭐⭐ 缓存穿透怎么解决？

<details><summary>答案</summary>

查询不存在的 key，每次都打 DB。

- **空值缓存**：DB 返回 null 也缓存（短 TTL，如 60s）
- **Bloom Filter**：先用 Bloom 判断"可能不存在"则直接拒
- **接口层校验**：明显不合法的 key（如负数 ID）直接拒

</details>

### Q10.5 ⭐⭐ 缓存雪崩怎么解决？

<details><summary>答案</summary>

大量 key 同时过期 / Redis 整体不可用。

- 过期时间加随机偏移（基础 TTL + jitter）
- 多级缓存（本地 + Redis + DB）
- 限流 + 降级（DB 保护）
- Redis 高可用（Sentinel / Cluster）

</details>

### Q10.6 ⭐⭐⭐ hotkey 怎么发现？怎么解决？

<details><summary>答案</summary>

发现：

- `redis-cli --hotkeys`（需要 LFU policy）
- 业务侧打点（每次访问 + key → 监控）
- proxy 层统计

解决：

- key 拆分 + hash tag 配多个副本（write 多写，read 随机选）
- 本地缓存 + tracking
- 业务层降级（限流热点、用近似数据）

</details>

### Q10.7 ⭐⭐ Redis 默认配置裸跑公网的最大风险？

<details><summary>答案</summary>

`bind 0.0.0.0` + 无密码 + `requirepass` 未设。

- 一秒内被扫描爆破
- 攻击者用 `CONFIG SET dir / dbfilename authorized_keys` + `SAVE` 写 SSH key 拿 root
- 或用 `MODULE LOAD` 加载恶意 .so

防御：

- `bind 127.0.0.1 私网IP`
- `protected-mode yes`
- `requirepass <强密码>` 或 ACL
- 重命名危险命令 `rename-command CONFIG ""`

</details>

### Q10.8 ⭐⭐ ACL 6.0 比 requirepass 强在哪？

<details><summary>答案</summary>

- 多用户 + 不同权限
- 命令白名单 / 黑名单（`+get -flushall`）
- key 模式限制（`~user:*`）
- channel 限制（`&events:*`）
- 6.2+ 选择性子命令权限
- 7.0 ACL v2：selectors（更细粒度）+ ACL CAT

```
ACL SETUSER alice on >password ~cache:* +get +set +del
```

</details>

### Q10.9 ⭐⭐⭐ 如何避免 active expire 卡顿？

<details><summary>答案</summary>

- `hz 10`（默认）：每秒 10 次过期扫描周期。调高让过期更及时但 CPU 占用上升
- `lazyfree-lazy-expire yes`：过期 key 释放交 bio 线程
- 过期 key 数量级很大时业务侧分散过期时间（不要 100w key 同一秒过期）

</details>

### Q10.10 ⭐⭐ 双写一致性怎么做？

<details><summary>答案</summary>

业务写流程的常见模式：

- **Cache-Aside**：写 DB → 删缓存（不是更新）。读时未命中再读 DB 写缓存
- **Write-Through**：写 DB + 写缓存原子（双写失败 rollback）
- **Write-Behind**：写缓存即返回，后台异步写 DB

最常用 Cache-Aside + 删除而非更新（避免并发覆盖）。仍有短暂不一致窗口（读到旧值），可加双删（先删 → 写 DB → 延迟再删一次）。

</details>

---

## R11 Redis vs Valkey

### Q11.1 ⭐⭐ Valkey fork 时间与基础版本？

<details><summary>答案</summary>

- 2024-03-20：Redis Inc. 改双许可证 RSALv2 + SSPLv1（不再 OSI 开源）
- 2024-03-28：Linux Foundation 宣布 Valkey fork
- 基础：Redis 7.2.4（最后一个 BSD 版本）
- 许可证：BSD-3

</details>

### Q11.2 ⭐⭐ Redis 8 的 tri-license 是哪三个？

<details><summary>答案</summary>

RSALv2、SSPLv1、AGPLv3。用户三选一。其中 AGPLv3 是 OSI 认可的开源协议，但有 network copyleft（卖 SaaS 要开源整个 stack）。

</details>

### Q11.3 ⭐⭐⭐ 为什么云厂商选 Valkey 不选 Redis 8 AGPLv3？

<details><summary>答案</summary>

AGPLv3 卖 SaaS 必须开源整个云平台代码 → 不可能。所以云厂商坚持 Valkey BSD-3。

</details>

### Q11.4 ⭐⭐ Redis 8 与 Valkey 8 哪个更快？

<details><summary>答案</summary>

差不多。Valkey 8 在多线程性能优化上做了较多工作（异步 IO 多线程优化），单实例 throughput 略好 10-20%（依工作负载）。Redis 8 集成了更多模块（JSON / TimeSeries 等）作为内置类型。命令层 100% 兼容。

</details>

### Q11.5 ⭐⭐ Redis 8 集成了哪些原本是模块的类型？Valkey 有吗？

<details><summary>答案</summary>

Redis 8 集成：JSON / TimeSeries / Bloom / Cuckoo / CMS / Top-K / T-Digest / Vector Set。
Valkey 8 不集成（保持轻量），用户要继续用社区模块。

</details>

### Q11.6 ⭐⭐⭐ 嵌入产品（出货 SDK / appliance）选哪个？

<details><summary>答案</summary>

**Valkey**。AGPLv3 嵌入产品要把整个产品开源，不可接受。BSD-3 无任何束缚。

</details>

### Q11.7 ⭐⭐ 已经在用 Redis 7.x，迁到哪？

<details><summary>答案</summary>

- Redis 7.x 仍是 BSD-3，可继续用
- 升 Redis 8.x 选 AGPLv3 协议（自部署内部用 OK）
- 切 Valkey 8.x：CLI / 协议兼容，迁移仅需切镜像 + 重启
- 不要继续升 RSALv2 / SSPLv1，许可证有麻烦

</details>

### Q11.8 ⭐⭐ AWS ElastiCache / GCP Memorystore 默认是哪个？

<details><summary>答案</summary>

2025 Q3-Q4 起默认 Valkey。原 Redis 实例可继续维护，但新建主推 Valkey。价格 Valkey 比 Redis 便宜 ~20%（AWS 数据）。

</details>

### Q11.9 ⭐⭐⭐ Valkey 治理结构与 Redis Inc. 的差别？

<details><summary>答案</summary>

- Valkey：Linux Foundation 中立基金会，多公司贡献者治理，代码贡献门槛低
- Redis Inc.：商业公司，CLA 必签，PR review 商业优先级

社区贡献者更倾向 Valkey。

</details>

### Q11.10 ⭐⭐ 短期内（2026）选型决策要点？

<details><summary>答案</summary>

| 你是 | 选 |
|---|---|
| 企业自部署内部用 | Redis 8（AGPLv3） 或 Valkey 任选 |
| 嵌入产品 | Valkey（必须 BSD） |
| 上云 SaaS 用 | 云厂商默认（多数 Valkey） |
| 想要 JSON / TS / Vector 内置 | Redis 8 |
| 想轻量 + 长期社区维护 | Valkey |

</details>

---

## R12 客户端

### Q12.1 ⭐⭐ RESP2 与 RESP3 关键差别？

<details><summary>答案</summary>

RESP3 增加 Map / Set / Boolean / Double / Null / Push / Attribute 等类型。Push 类型让服务端可在主连接主动推送（用于客户端缓存失效），Map 让 HGETALL 直接返回 map 不用客户端拼。

通过 `HELLO 3` 协商升级，默认仍 RESP2。

</details>

### Q12.2 ⭐⭐⭐ Jedis、Lettuce、Redisson 怎么选？

<details><summary>答案</summary>

- Jedis：简单同步场景；每线程一连接（用池）
- Lettuce：主流推荐；Netty 异步；单连接多路复用线程安全
- Redisson：要分布式锁 / 集合时；重量级，不适合纯 KV

Spring Boot 默认 Lettuce。

</details>

### Q12.3 ⭐⭐ 连接池大小怎么定？

<details><summary>答案</summary>

经验：

- Web / RPC：8-50
- 高并发：50-100
- 离线任务：5-10
- Lettuce 多路复用：1-4 即可

不是越大越好——服务端单线程串行处理，连接太多徒增 fd 与 epoll 开销。

</details>

### Q12.4 ⭐⭐⭐ NAT 超时导致的"连接突然 broken"怎么解决？

<details><summary>答案</summary>

云上 NAT/SLB 通常清理 idle > 350-900s 的连接。

- 客户端 keepalive 设小于 NAT 超时（如 300s）
- 服务端 `tcp-keepalive 60`
- 池配 testWhileIdle + minEvictableIdleTime < NAT 超时
- 应用层定期 PING

</details>

### Q12.5 ⭐⭐ MOVED 与 ASK 客户端怎么处理？

<details><summary>答案</summary>

- MOVED：更新本地 slot 表 + 重试
- ASK：**不**更新表，发请求前先发 `ASKING`，再发原命令

主流 SDK 自动处理，业务无感。

</details>

### Q12.6 ⭐⭐⭐ Cluster Pipeline 跟单节点有何不同？

<details><summary>答案</summary>

客户端要按 key 所在 slot 分组，分别发到不同节点，再按原始顺序拼回响应。主流 SDK 已内置实现。手撸要处理 MOVED 重定向 + 排序两件难事。

</details>

### Q12.7 ⭐⭐ 客户端缓存 Tracking 两种模式？

<details><summary>答案</summary>

- **default**：服务端记录每个 key 被哪些 client 读了，精确推送 invalidate；内存高
- **broadcasting**：按前缀订阅，前缀下任何 key 改都通知；内存低、但通知量大

```
CLIENT TRACKING ON BCAST PREFIX user: PREFIX product:
```

</details>

### Q12.8 ⭐⭐ 超时配置常见错误？

<details><summary>答案</summary>

- ReadTimeout 太短（< 100ms）：网络抖动就误报
- ReadTimeout 太长（> 业务总超时）：业务超时但 SDK 仍在等
- Pub/Sub / BLPOP 等长命令的 ReadTimeout 要单独大一些（必须 > 阻塞超时）
- 自动重试 + 非幂等命令：重复执行（INCR 翻倍）

</details>

### Q12.9 ⭐⭐⭐ EVAL 与 EVALSHA 的标准模式？

<details><summary>答案</summary>

```python
sha = r.script_load(script)
try:
    r.evalsha(sha, ...)
except NoScriptError:
    sha = r.script_load(script)
    r.evalsha(sha, ...)
```

主流 SDK（如 go-redis 的 `Script.Run`）自动做这个回退。

</details>

### Q12.10 ⭐⭐ `CLIENT LIST` 哪些字段排查最有用？

<details><summary>答案</summary>

- `addr`：客户端 IP，定位来源
- `age` / `idle`：长 idle 看泄漏
- `qbuf` / `qbuf-free`：输入缓冲区
- `obl` / `oll` / `omem`：输出缓冲区，omem 高 → 慢消费者
- `cmd`：上一条命令
- `flags`：是否阻塞 (b) / pub-sub (P)
- `lib-name` / `lib-ver`（7.2+）：定位 SDK 版本

</details>

---

## 综合实战题（架构师向）

### Q-A1 ⭐⭐⭐ 一个支持 1000 节点的 Redis Cluster 容量规划

设计要点：

- 实际不要超 500 节点（gossip 带宽快不够）
- 每节点 32GB-128GB，更大不利于 fork
- slot 均匀分配到 master，每 master 数十 slot
- 副本数 1（必要）-2（多机房）
- hash tag 让相关数据同 slot
- 跨机房用单独的 tunnel 加 backlog 大

### Q-A2 ⭐⭐⭐ 100GB 数据从单实例迁到 Cluster 不停服

方案：

1. Cluster 起空集群
2. `MIGRATE` 手动迁单 key 测试，OK 后批量
3. 写双写：业务同时写老单实例和新 Cluster
4. 后台脚本 SCAN 老实例 → 写 Cluster
5. 读切换：先少量灰度
6. 验证一致性
7. 全切 Cluster
8. 老实例下线

### Q-A3 ⭐⭐⭐ 设计一个分布式锁

- `SET lock:key uuid EX 30 NX` 加锁
- value 用 uuid 防误释
- 解锁用 Lua（GET + DEL 原子）
- watchdog 续约（持锁期间每 10s 续 30s）
- Redlock（5 节点 quorum）应对单机故障

不要用 SETNX + EXPIRE 两条（非原子）。

### Q-A4 ⭐⭐⭐ Redis 用作消息队列的可行方案

- List：LPUSH + BRPOP，可靠但无 ack（消费方崩消息丢）
- Streams + ConsumerGroup：有 ack + 回溯 + PEL，主流
- Pub/Sub：仅广播 / 通知，**不能当 MQ 用**

要持久化 + ack + 重试 → Streams。

### Q-A5 ⭐⭐⭐ 一个突发 P99 飙升 200ms 的诊断流程

```
SLOWLOG GET 10           → 有慢命令？KEYS / 大 hash 操作？
LATENCY DOCTOR           → fork / fsync / expire-cycle 事件？
INFO clients             → blocked_clients / 池满？
INFO memory              → 内存接近 maxmemory / 碎片高？
INFO replication         → 复制延迟 / repl-backlog full？
INFO stats               → instantaneous_ops_per_sec 突增？

OS 层：dmesg / iostat / netstat / mpstat
客户端：池利用率 / RT 直方图
```

按层定位根因，根因找到才止步。

---

> 🔁 反馈：每周抽 10 道做盲答，答不出来回去复习对应章节
