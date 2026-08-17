# 精通 Redis 数据结构内部：SDS、listpack、quicklist、skiplist 与 dict

> 路线图来源：[Redis 官方数据类型介绍](https://redis.io/docs/latest/develop/data-types/) + Redis 8.x 源码
> 关联章节：[R02 内存与过期](./02-精通-Redis-内存与过期.md)、[R08 性能模型](./08-精通-Redis-性能模型.md)

---

## 引言：Redis 为什么"快"的另一个答案

人们谈起 Redis 快，第一反应是"全内存 + 单线程不锁"。这只对了一半。

真正的秘密在更深一层：**Redis 给每个对外的数据类型（String / List / Hash / Set / Sorted Set / Stream / Bitmap / HyperLogLog）都准备了多种内部表示，按数据规模与访问模式动态切换**。一个 `Hash` 在元素少时是紧凑的连续内存块（listpack），元素多了自动膨胀成哈希表。一个 `Sorted Set` 既是哈希表又是跳表，两个结构共享同一份数据，前者支持 O(1) 取分值，后者支持 O(log N) 范围扫描。这套设计让 Redis 在**小数据零开销、大数据可扩展**之间无缝切换——这才是它在 1KB 和 1MB 两种 key 上都能稳定 100K QPS 的原因。

本章把每一种结构拆开看：内存布局、操作复杂度、何时切换、踩过的坑（CVE 级别）。读完之后你应该能在 `redis-cli` 里用 `OBJECT ENCODING` 一眼判断每个 key 长什么样，预估它的内存占用，并知道修改哪个 config 能让它更紧凑或更可扩展。

---

## 第一章：redisObject —— 万物的外壳

每一个对外暴露的 Redis 值（不论 String、List 还是 ZSet）都由一个 `redisObject` 包装：

```c
// src/server.h (简化)
typedef struct redisObject {
    unsigned type:4;       // OBJ_STRING / OBJ_LIST / OBJ_HASH / OBJ_SET / OBJ_ZSET / OBJ_STREAM
    unsigned encoding:4;   // 实际底层编码（见下表）
    unsigned lru:LRU_BITS; // LFU 或 LRU 元数据，24 bit
    int refcount;          // 引用计数（共享对象用）
    void *ptr;             // 指向真正的底层结构
} robj;
```

固定 16 字节头部。`type` 是对外可见类型，`encoding` 是当前的内部实现。**同一个对外类型可以有多种 encoding**，Redis 按规模阈值在它们之间切换。`ptr` 才是指向真正的数据。

可观测：

```
127.0.0.1:6379> SET foo bar
OK
127.0.0.1:6379> OBJECT ENCODING foo
"embstr"
127.0.0.1:6379> SET num 1234
OK
127.0.0.1:6379> OBJECT ENCODING num
"int"
127.0.0.1:6379> LPUSH mylist a b c
(integer) 3
127.0.0.1:6379> OBJECT ENCODING mylist
"listpack"
```

完整对照表（Redis 8 默认）：

| 对外类型 | 可能的 encoding | 切换关键阈值 |
|---|---|---|
| String | `int` / `embstr` / `raw` | 数字 `int`；≤44 字节 `embstr`；其余 `raw` |
| List | `listpack` / `quicklist` | 节点超 128 项或单项 > 64 字节 → quicklist |
| Hash | `listpack` / `hashtable` | 元素 > 128 或单个 entry > 64 字节 |
| Set | `intset` / `listpack` / `hashtable` | 全整数且 ≤ 512 → intset；少量字符串 → listpack；其余 hashtable |
| ZSet | `listpack` / `skiplist` | 元素 > 128 或单 member > 64 字节 → skiplist |
| Stream | `stream`（Radix Tree + 节点 listpack） | 无切换 |
| Bitmap | 复用 String | — |
| HyperLogLog | 复用 String（固定 12 KB 稠密 / 稀疏） | — |

所有阈值都通过 `CONFIG SET` 可调；变小利于内存，变大利于"小集合一直紧凑"。

---

## 第二章：String —— int / embstr / raw 三态

### 2.1 三种 encoding 的物理布局

```
1) int（值是 long 范围内的整数）
   redisObject{ type=STRING, encoding=int, ptr = (void*)(long)1234 }
   ptr 字段直接放整数值，**无额外分配**。

2) embstr（短字符串）
   redisObject 头部 + sds 头部 + 字符串数据 + '\0'
   全部一次 malloc，**保证 redisObject 和 sds 在同一个 cache line**。
   边界：sds 数据 ≤ 44 字节（Redis 3.2 之后从 39 改到 44，3.2 引入紧凑 SDS 头使 64 字节 jemalloc class 内能多放 5 字节）。

3) raw（长字符串）
   两次 malloc：redisObject 一次，sds 另一次。ptr 指过去。
```

为什么 44？源码里写死了，根因是 jemalloc 在 64 字节 size class 内可以放下 `redisObject(16) + sdshdr8(3) + content(44) + '\0'(1) = 64`，再大一字节就跨到 80 字节 class，浪费就来了。

### 2.2 SDS（Simple Dynamic String）

```c
// src/sds.h (简化)
struct sdshdr8 {
    uint8_t len;        // 已用长度
    uint8_t alloc;      // 不含头/'\0' 的总分配
    unsigned char flags;// 头类型 bit 标记 sdshdr8/16/32/64
    char buf[];         // 实际数据 + 终止 '\0'
};
```

5 个变种（sdshdr5 / 8 / 16 / 32 / 64）按字符串长度选最省头部的版本。`sdshdr5` 头只 1 字节，专用于小到 < 32 字节的字符串。

`len` 字段让 `STRLEN` 是 O(1)；二进制安全（不靠 `'\0'` 判长度）；预分配策略让 `APPEND` 平均 O(1)：

```
1) 字符串 < 1MB：扩容到 2 * new_len
2) 字符串 ≥ 1MB：每次只多分 1MB
```

这种"前期翻倍后期线性"的策略避免了 GB 级 key 上一次扩出几个 GB。

### 2.3 共享 integer 对象

Redis 启动时预创建了 0–9999 的共享 String integer 对象，所有 SET 一个小整数时直接复用，**refcount++ 而非新分配**。源码中由常量 `OBJ_SHARED_INTEGERS = 10000` 决定（编译期固定，无法通过 CONFIG 修改）。

注意：开启 `maxmemory` 且 eviction 模式为 LRU/LFU 时，共享对象**不再共享**——因为 LFU/LRU 元数据是 per-object 的，共享会导致计数错乱。这是个容易踩的优化误区。

---

## 第三章：listpack —— ziplist 的接班人

### 3.1 历史背景：ziplist 为什么被抛弃

Redis 长期用 `ziplist` 作为小集合的紧凑表示。它是一段连续内存，每个 entry 头部存"前一项长度 prevlen"用于反向遍历。问题：**修改任一 entry 可能引起 prevlen 字段从 1 字节升到 5 字节，进而触发后续所有 entry 的级联更新（cascading update）**——最坏 O(N²)。在大 ziplist 上是性能杀手，最终也成了 [CVE-2021-32628](https://github.com/redis/redis/security/advisories/GHSA-vw22-qm3h-49pr)（并列 CVE-2021-32627）等几个安全问题的根源。

Redis 7.0 引入 **listpack** 取代 ziplist，并在 7.x–8.x 内逐步把 hash / set / zset / list / stream 的内部小型表示全部迁移过去。Redis 8 里 ziplist 已经彻底退役。

### 3.2 listpack 结构

```
+--------+-------+ ... +---+
| Header | Entry | ... |END|
+--------+-------+ ... +---+

Header: 4 字节 total bytes + 2 字节 num elements
Entry:  encoding + content + backlen
END:    1 字节 0xFF
```

**关键差别**：每个 entry 末尾存 `backlen`（自身长度），**反向遍历靠"从后往前累减 backlen"完成**。修改一个 entry 不影响前后任何 entry——彻底消除级联更新。

`backlen` 是变长整数（1–5 字节），所以小元素仍然紧凑。空间利用率比 ziplist 略低（多一个 backlen 字段）但绝对安全。

### 3.3 在哪用

- Hash 元素数 ≤ 128 且单 entry ≤ 64 字节
- Set 含字符串且元素数 ≤ 128 且单 entry ≤ 64 字节
- ZSet 元素数 ≤ 128 且单 member ≤ 64 字节
- List 每个 quicklist 节点内部
- Stream 每个 Radix Tree 叶子的 entry batch

阈值由四个 config 控制：
```
hash-max-listpack-entries 128
hash-max-listpack-value   64
set-max-listpack-entries  128
zset-max-listpack-entries 128
zset-max-listpack-value   64
list-max-listpack-size   -2     # -2 = 每节点最大 8KB；正数 = 每节点元素数
```

**调优经验**：把阈值调大（如 hash 改 512 / 256）能让更多场景保持紧凑表示，但每次写都是 O(N) memmove——读多写少的场景才划算。

---

## 第四章：dict —— Redis 自己实现的哈希表

### 4.1 dict 的二维结构

```c
// src/dict.h (简化)
typedef struct dict {
    dictEntry **ht_table[2];  // 两个桶数组（用于渐进式 rehash）
    unsigned long ht_used[2];
    unsigned long ht_size_exp[2]; // log2 大小
    long rehashidx;               // -1 = 没在 rehash；否则正在搬第几个桶
    ...
} dict;

typedef struct dictEntry {
    void *key;
    union { void *val; uint64_t u64; int64_t s64; double d; } v;
    struct dictEntry *next;       // 拉链
} dictEntry;
```

两个 `ht_table` 是为了实现**渐进式 rehash**。

### 4.2 负载因子与 rehash 触发

- 默认 `dict_force_resize_ratio = 4`
- 正常（`DICT_RESIZE_ENABLE`）时，`used / size >= 1` 且没有 BGSAVE/BGREWRITEAOF 进行时，触发 rehash 扩容
- 有子进程（`DICT_RESIZE_AVOID`）时，需 `used >= 4 * size`（即 `dict_force_resize_ratio = 4`）才强制扩容，避免在 fork 写时复制期间触发大量内存复制

扩容目标：第一个 ≥ `used * 2` 的 2^N。

### 4.3 渐进式 rehash 的关键

每次 dict 的**增删改查**操作，顺手搬一个老桶到新桶（最多 100 个 entry / 1ms）。后台定时任务每 100ms 也补搬一次。**永不阻塞主线程**。

代价：rehash 期间所有读写都要查两次表（先看新表再看老表）。所以 Redis 文档建议**不要在主从切换、AOF rewrite 期间做大量插入**——会同时触发 rehash + COW，性能掉一截。

### 4.4 为什么 Redis 自己实现 dict 而不用 std

- **支持 SCAN 游标稳定遍历**：即使中途 rehash，cursor 也能保证不重不漏（用反向二进制位 Knuth 算法）
- **可植入 LFU/LRU 元数据**：每个 dictEntry 关联一个 redisObject，包含 lru 字段
- **可中断**：渐进式 rehash 是其他通用 hashmap 没有的

---

## 第五章：quicklist —— List 的双层结构

### 5.1 结构

```
+-----------+    +-----------+    +-----------+
| quicklist | -> | quicklist | -> | quicklist | (双向链表)
|  node     |    |  node     |    |  node     |
+-----------+    +-----------+    +-----------+
     |                |                |
  listpack         listpack         listpack
 [a, b, c]      [d, e, f, g, h]    [i, j]
```

每个 quicklist node 内部是一个 listpack。"双层链表"：外层链表保证大 List 也能 O(1) push/pop 两端；内层 listpack 让小段密集存储。

### 5.2 大小控制

```
list-max-listpack-size -2   # 默认每个节点最多 8KB
list-compress-depth 0       # 默认 0，可改 1/2/3
```

`list-compress-depth=1` 意思："两端各保留 1 个未压缩节点，中间节点全部 LZF 压缩"。**适合很长但只在两端高频访问的 List**（如消息队列）。压缩节点在被访问时透明解压。

### 5.3 复杂度速查

| 操作 | 复杂度 |
|---|---|
| `LPUSH` / `RPUSH` / `LPOP` / `RPOP` | O(1)（除非节点满要新建） |
| `LRANGE 0 n` | O(n) |
| `LINDEX i` | O(N)（要遍历节点链） |
| `LINSERT BEFORE/AFTER pivot value` | O(N) |
| `LREM count value` | O(N) |

> **结论**：List 不适合**中间随机访问**——这种场景就别用 List，改用 Sorted Set（按 score 索引）或干脆 Hash。

---

## 第六章：Sorted Set —— skiplist + dict 的双结构

### 6.1 为什么用两份结构

Sorted Set 的语义是"按 score 排序的 member 集合"，需要同时支持：

- O(1) 取 member 的 score：`ZSCORE`
- O(log N) 按 score 范围扫描：`ZRANGEBYSCORE`
- O(log N) 按排名扫描：`ZRANGE`
- O(log N) 插入 / 删除：`ZADD` / `ZREM`

**单一数据结构难以两者兼得**：哈希表能 O(1) 查找但不支持排序；B-tree 能排序但 O(log N) 查找；跳表能排序且查找 O(log N) 但不到 O(1)。

Redis 的设计是**两份结构共享 member 实例**：

```c
typedef struct zset {
    dict *dict;        // member -> score
    zskiplist *zsl;    // skiplist 按 score 排序
} zset;
```

`dict` 提供 O(1) 的 member→score 查询；`zsl` 提供 O(log N) 的范围扫描。两者**共用 member 字符串**（一个 sds 被两个结构都指着），所以内存额外开销远不到 2 倍。

### 6.2 跳表的物理结构

```
Level 4:  HEAD ------------------------------> NIL
Level 3:  HEAD ------> [m=4] ----------------> NIL
Level 2:  HEAD ------> [m=4] -----> [m=8] ---> NIL
Level 1:  HEAD -> [m=2] -> [m=4] -> [m=8] ---> NIL
Level 0:  HEAD -> [m=2] -> [m=4] -> [m=6] -> [m=8] -> NIL

每个节点的层数随机生成：每升一层的概率为 1/4（ZSKIPLIST_P=0.25）。
预期高度 O(log N)，最大 ZSKIPLIST_MAXLEVEL = 32（源码常量，长期未变）。
```

```c
typedef struct zskiplistNode {
    sds ele;               // member
    double score;
    struct zskiplistNode *backward;  // 前向指针（O(log N) 反向）
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;          // 到下个节点跨过多少节点（用于按 rank 查找）
    } level[];                       // 柔性数组
} zskiplistNode;
```

`span` 是 Redis 对跳表的关键定制：让 **`ZRANGE BYRANK` 也是 O(log N)**——遍历时累加 span 就是 rank。

### 6.3 为什么不用红黑树 / B-tree

源码作者 antirez 给出三点（已成经典 Q&A）：

1. **实现简单**：跳表 100 行，红黑树要 500+ 行还容易写错
2. **范围扫描快**：跳表底层是有序链表，连续 next 即可；红黑树要回溯节点栈
3. **更省内存（在 Redis 场景下）**：每节点平均 1.33 指针（p=0.25 几何分布），低于红黑树固定的两个孩子指针

工程考量超过纯算法 BigO 比较。

### 6.4 listpack 与 skiplist 的切换

```
默认：
zset-max-listpack-entries 128
zset-max-listpack-value   64

超过任一阈值，从 listpack 切到 skiplist+dict。
切换是一次性的 O(N) 重建。
```

实际效果：**小 ZSet 几乎零额外开销**（紧凑连续内存 + cache 友好），**大 ZSet 才付 skiplist 的指针开销**。

---

## 第七章：Set —— intset / listpack / hashtable 三态

### 7.1 intset：纯整数集合

```c
typedef struct intset {
    uint32_t encoding;  // INTSET_ENC_INT16 / 32 / 64
    uint32_t length;
    int8_t   contents[];// 实际是 int16/32/64 数组，按值升序
} intset;
```

按值升序存储，**二分查找 O(log N)**。插入用 memmove。

升级：插入更大值时整个数组重写到下一档编码。**不会降级**——一旦升到 int64，即使后续删除大值也保持 int64。

阈值：`set-max-intset-entries 512`。超出转 hashtable。

### 7.2 listpack：少量字符串集合

Redis 7.2 开始 Set 也支持 listpack 表示（之前只有 intset / hashtable）。当 Set 含字符串但元素少且短，自动用 listpack。这是 Redis 7.2 的内存优化之一，老版本里小字符串 Set 是 hashtable，浪费多。

### 7.3 何时是 hashtable

```
set-max-listpack-entries 128
set-max-listpack-value   64
set-max-intset-entries   512
```

任一阈值超出 → 转 hashtable。

### 7.4 SINTERSTORE 的性能秘密

`SINTER s1 s2` 取交集时，**先按大小排序所有输入 set**，从最小的开始遍历，依次去其余 set 里查存在性。复杂度 O(N * M)，N 是最小 set 的大小，M 是 set 数量。**不是简单的 O(N1*N2)**。

实际：往大 set 里塞小数据没问题，**避免 N 个等大的大 set 做交集**，瓶颈会暴露。

---

## 第八章：Hash —— listpack ↔ hashtable

### 8.1 切换阈值

```
hash-max-listpack-entries 128
hash-max-listpack-value   64
```

超出阈值 → 转 hashtable。

`HGET` 在 listpack 表示下是 O(N)，但 N ≤ 128 且数据 cache 友好，实测比 hashtable 还快。这是为什么阈值定 128 而非更小——再大就线性扫描太久。

### 8.2 经典反模式：用 String 模拟 Hash

```
# 反模式
SET user:1:name "alice"
SET user:1:age  "30"
SET user:1:city "NYC"

# 正确
HSET user:1 name alice age 30 city NYC
```

每个 String key 都自带一个 dictEntry（约 32 字节头）+ key 字符串 + redisObject(16) + 实际值——**字段越多浪费越大**。用 Hash + listpack 表示能省 5-10 倍内存。

### 8.3 字段级过期（Redis 7.4+）

老版本 Redis Hash 不支持单个字段过期，只能 key 整体 TTL。**Redis 7.4 加入字段级 TTL**：

```
HEXPIRE myhash 60 FIELDS 1 session_id
HTTL    myhash      FIELDS 1 session_id
HPERSIST myhash     FIELDS 1 session_id
```

底层：每个字段额外 8 字节存到期时间戳（绝对毫秒）。`HEXPIRE` 后该 Hash **不再用 listpack 表示**（listpack 没空间存额外的 TTL 元数据），自动转 hashtable。

---

## 第九章：Stream —— 时间序列日志的真容

### 9.1 物理：Radix Tree + 节点 listpack

```
                       Radix Tree（按 ID 范围切片）
                      /          |          \
              listpack         listpack          listpack
        [1234567-0, 1234568-0,..][1234571-0,...][1234580-0,...]
```

Stream 的 ID 是单调递增的 `毫秒-序列号`，按 ID 前缀切到不同 Radix Tree 叶子。每个叶子是一个 listpack 装多条 entry。这种设计**让按时间范围扫描和按 ID 精确定位都很快**。

### 9.2 Consumer Group 的"未确认列表"PEL

每个 consumer group 维护一个 `Pending Entries List`（PEL）——记录"读了但未 ACK"的 ID。这个 PEL 也是 Radix Tree。

故障恢复：consumer 重启后能 `XPENDING` 看到自己的待 ACK 列表；超时未 ACK 可被 `XAUTOCLAIM` 转给其他 consumer。这套机制是 Stream 比 Pub/Sub 更可靠的根本原因。

### 9.3 与 Kafka 的差异

| 维度 | Redis Stream | Kafka |
|---|---|---|
| 持久化 | RDB+AOF（同进程） | 磁盘日志（独立 broker） |
| 单 Stream 吞吐 | 单核 ~100K/s | 单 partition ~1M/s |
| 水平扩展 | 靠 Cluster slot 分流 | 原生 partition |
| 消费者协议 | XREAD/XREADGROUP + 显式 ACK | 长连接 + offset commit |
| 消息保留 | MAXLEN / MINID 修剪 | retention.ms / .bytes |
| 适用 | 单 DC、~10MB/s 量级、低延迟 | 跨 DC、~GB/s 量级、批处理 |

---

## 第十章：观察与调试

### 10.1 看一个 key 的内部表示

```
OBJECT ENCODING <key>           # 当前 encoding
OBJECT REFCOUNT <key>           # 引用计数（共享对象 > 1）
OBJECT IDLETIME <key>           # 多久没访问（LRU 信息）
OBJECT FREQ <key>               # LFU 频次（仅 LFU 模式）
DEBUG OBJECT <key>              # 一切细节（含 serializedlength、type）
MEMORY USAGE <key> SAMPLES 0    # 完整测量内存占用（含 sds 头 + redisObject + 内部结构）
```

### 10.2 主动测试切换阈值

```
127.0.0.1:6379> DEL h
127.0.0.1:6379> HSET h f1 v1
127.0.0.1:6379> OBJECT ENCODING h
"listpack"
127.0.0.1:6379> CONFIG SET hash-max-listpack-entries 1
OK
127.0.0.1:6379> HSET h f2 v2
127.0.0.1:6379> OBJECT ENCODING h
"hashtable"
```

调小阈值能复现切换。**注意**：阈值是"创建/修改时"判定的，**已经在 hashtable 状态的 hash 即使元素减少也不会回到 listpack**——单向升级。

### 10.3 内存估算

| 类型 | encoding | 单元素近似开销（64 位机） |
|---|---|---|
| String int | int | 16 字节（redisObject 头） |
| String embstr | embstr | 16 + 3 + len + 1，one alloc |
| String raw | raw | 16 + 16 + 3 + len + 1，two alloc |
| Hash listpack | listpack | (field + value) + ~5 字节 overhead per entry |
| Hash hashtable | hashtable | dictEntry 32 + sds(field) + sds(value) + redisObject 16 |
| ZSet listpack | listpack | (member + score(8)) + ~5 字节 |
| ZSet skiplist | skiplist+dict | dictEntry 32 + skiplistNode 32 + sds(member) + 8 |

大致结论：**同样数据，listpack 比 hashtable 省 3-5 倍内存**。

---

## 第十一章：生产级最佳实践

1. **永远观察 encoding**：定期采样 `OBJECT ENCODING` 验证 key 是否在预期表示。一旦"应该是 listpack 的 hash 变成了 hashtable"，要么是数据量超预期，要么是阈值被改小。
2. **bigkey 用 hash + 拆分**：用户画像这种字段多的，**不要每个属性单独 SET**——`HSET user:1 ...`，listpack 紧凑省内存且支持 HEXPIRE 字段级 TTL（7.4+）。
3. **不要在大 ZSet 上 `ZRANGE 0 -1`**：那是 O(N) 全表，几百万 member 直接卡主线程几百 ms。要分页就 `ZRANGEBYSCORE` + LIMIT。
4. **List 不要做随机访问**：`LINDEX i` 在大 List 上是 O(N)，远不如 ZSet O(log N)。
5. **慎用 `HGETALL` / `LRANGE 0 -1`**：返回大集合时**会一次性序列化全部数据到 client 缓冲区**，触发 `client-output-buffer-limit` 强断连。改用 `HSCAN` / `LRANGE 0 99` 分页。
6. **共享 integer 与 maxmemory 互斥**：开 maxmemory 后小整数不再共享，约 5% 内存开销。生产配 maxmemory 时记住这一点。
7. **Stream 修剪要趁早**：默认 `XADD ... MAXLEN ~ N` 是近似修剪（避免每次 XADD 都精确切），比 `MAXLEN N` 快得多。`~` 误差 < 节点大小。
8. **RESP3 客户端值得切**：拿到 Hash 时直接是 client 端 dict 而非 array（少一次格式转换）；服务端也减少了一些序列化判断。
9. **小心 EXPIRE 大 key**：`UNLINK` 在 listpack/intset 上和 `DEL` 一样快（释放一段连续内存），但 hashtable / skiplist 上 `DEL` 是 O(N) 同步释放——上 GB key 用 `UNLINK` 走异步。
10. **调阈值是双刃剑**：把 `hash-max-listpack-entries` 调到 512 能让更多 hash 保持紧凑，但每次 `HSET` 都是 O(N) memmove；**读多写少**才划算。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：以为 OBJECT ENCODING 会自动 downgrade
hash 一旦升级到 hashtable，即使删到只剩一个字段也不会回 listpack。要"重新紧凑"得整体 `COPY` 到新 key。

### ❌ 陷阱 2：用 List 做时间窗口去重
`LPUSH + LREM value` 是 O(N) 扫描整个 List。改用 ZSet（score=时间戳）+ `ZRANGEBYSCORE`。

### ❌ 陷阱 3：把超长字符串塞 hash 字段
`hash-max-listpack-value 64` 是单 entry 上限。`HSET h field large_value`（>64B）会立刻把整个 hash 升级到 hashtable，可能损失 5x 内存。

### ❌ 陷阱 4：ZADD CH 与 ZADD INCR 混淆
`ZADD CH` 改"返回值含义"为"变更的元素数"；`ZADD INCR` 把 score 改为"增加 delta"。前者用于幂等去重写法，后者用于计数器。

### ❌ 陷阱 5：用 SORT 排大 Set
`SORT myset` 在大 Set 上是 O(N log N) + 内存复制。改 ZSet 一开始就排好。

### ❌ 陷阱 6：HyperLogLog 用作精确去重
HLL 误差 ~0.81%。如果业务要精确（如 distinct user id），改 Set 或 Bitmap。

### ❌ 陷阱 7：Bitmap 上 BITCOUNT 全表
大 Bitmap 全表 BITCOUNT 是 O(N) 主线程操作。要么分段 BITCOUNT，要么改 HLL。

### ❌ 陷阱 8：被 SCAN 的"游标重复"吓到
SCAN 不保证不重复——可能在 rehash 期间返回同一 key 两次。**应用层要去重**。这是为 O(1) 游标稳定性付出的代价。

### ❌ 陷阱 9：Stream 不修剪
没有 `MAXLEN` 的 Stream 会无限增长。生产必须配 `XADD ... MAXLEN ~ N` 或定时 `XTRIM`。

### ❌ 陷阱 10：以为 EXPIRE 是精确的
Redis 的 active expire 周期默认每 100ms 一次，每轮随机采样约 20 个带 TTL 的 key、删除其中已过期者；若过期比例 >25% 则立即重复一轮，否则结束。**过期 key 可能在到期后几秒才真正被回收**。延迟敏感场景要主动 GET 触发 lazy expire。

---

## 第十三章：练习题

**练习 1**：以下 4 个 key 的 encoding 分别是什么？

```
SET k1 "hello"
SET k2 "this is a string longer than forty four bytes _________"
SET k3 12345
ZADD k4 1 a 2 b
```

**练习 2**：一个有 100 万 member 的 ZSet 占用约多少内存？（假设每个 member 平均 16 字节，score 是 double）写出估算式。

**练习 3**：以下命令的复杂度是多少？

```
LRANGE mylist 100 200
HGETALL big_hash         # 含 10000 字段
SINTER s1 s2 s3          # |s1|=10, |s2|=100, |s3|=10000
SRANDMEMBER myset 5
```

**练习 4**：业务场景：每个用户有一个购物车（最多 50 件商品，每件 ID + 数量 + 加入时间）。设计 Redis 数据模型，要求能：
- O(1) 加商品 / 减商品
- O(N) 列出所有商品（N ≤ 50）
- O(log N) 按加入时间排序
说出选哪种结构、为什么、阈值怎么配。

**练习 5**：复现 dict 渐进式 rehash：写一段 Python 代码连续往一个 hash 加 5000 个字段，每加一个就 `OBJECT ENCODING` 看一次——能看到哪个时间点切到 hashtable？

---

## 参考答案

**练习 1**：
- k1 = embstr（5 字节 ≤ 44）
- k2 = raw（54 字节 > 44）
- k3 = int（共享对象，refcount > 1）
- k4 = listpack（元素数 2 远 < 128，member 长度 1 字节 < 64）

**练习 2**：1M member × (32 字节 dictEntry + 32 字节 skiplistNode + 平均 1.33 个 forward 指针 × 8 + 8 字节 score + sds 头 3 字节 + 16 字节 member sds 内容) ≈ 100 字节 × 1M = **~100 MB**。

**练习 3**：
- `LRANGE mylist 100 200`：O(min(N, len-N))，从近端开始遍历，~100 步
- `HGETALL big_hash`：O(N) = O(10000)，**且单次返回 1 万字段到 client buffer，注意 client output buffer 限制**
- `SINTER`：按最小 set 排序后逐个查存在性，O(10 × 3) = O(30)，**不是 O(10 × 100 × 10000)**
- `SRANDMEMBER myset 5`：O(5)，不需要遍历全表

**练习 4**：用 Hash 存购物车，field=商品ID，value=`数量:加入时间戳`。
- O(1) 加 / 减：`HSET` / `HDEL`
- 列出所有：`HGETALL`，元素 ≤ 50 < 128 → listpack 紧凑
- 按加入时间排序：客户端排（50 个项排序很便宜），或为热点用户**额外**维护一个 ZSet 镜像
配置：保持默认 `hash-max-listpack-entries 128`、`hash-max-listpack-value 64`，确保购物车永远是 listpack 表示，单用户购物车内存占用 < 1KB。

**练习 5**：默认 `hash-max-listpack-entries 128`、`hash-max-listpack-value 64`。加到第 129 个字段时（或某个 value 超过 64 字节）会切到 hashtable。脚本示意：

```python
import redis
r = redis.Redis()
r.delete("h")
for i in range(200):
    r.hset("h", f"f{i}", "v")
    print(i, r.object("encoding", "h"))
```

会看到 128 之前是 `b'listpack'`，第 129 个 hset 后变成 `b'hashtable'`。

---

## 小结

| 类型 | encoding 候选 | 关键阈值 / 切换 | 适用场景 |
|---|---|---|---|
| String | int / embstr / raw | ≤44B embstr；其余 raw | 缓存值、计数器 |
| List | listpack（小） / quicklist | 节点 8KB / 128 项 | 队列、最新 N 项 |
| Hash | listpack / hashtable | 128 entries / 64B value | 对象的字段 |
| Set | intset / listpack / hashtable | 512 整数；128/64B | 标签、去重 |
| ZSet | listpack / skiplist+dict | 128 entries / 64B member | 排行榜、范围查询 |
| Stream | Radix Tree + listpack | 无切换 | 消息流、事件日志 |
| Bitmap | 复用 String | — | 在线状态、统计 |
| HyperLogLog | 复用 String 12KB | 稀疏→稠密 | 大基数去重 |

记住 4 个原则：
1. **小集合天然紧凑**——别担心 small Redis 内存浪费
2. **阈值过线则 O(N) 一次性升级**——批量插入大集合时一次到位
3. **没有 downgrade**——大集合"瘦身"得手动 COPY
4. **观察永远是第一步**——`OBJECT ENCODING` + `MEMORY USAGE` 比猜测靠谱

下一篇 **R02 — Redis 内存管理与过期回收** 将拆开 `maxmemory` 八种淘汰策略、lazy + active 过期、active defrag——把"为什么我的 Redis 内存涨上去就不下来"讲到底。
