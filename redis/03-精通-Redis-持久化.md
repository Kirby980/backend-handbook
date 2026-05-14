# 精通 Redis 持久化机制：RDB、AOF、混合与崩溃恢复

> 关联章节：[03 数据结构](./03-精通-Redis-数据结构内部.md)、[04 内存与过期](./04-精通-Redis-内存与过期.md)、[06 复制与 Sentinel](./06-精通-Redis-复制与-Sentinel.md)

---

## 引言：内存型数据库怎么"不丢"

Redis 是内存型数据库——一旦进程退出，内存里的数据就消失。但绝大多数生产使用场景都不能容忍数据丢失。Redis 提供两种持久化机制：**RDB**（snapshot）和 **AOF**（append-only log），以及它们的**混合模式**。

这一章把这三种机制讲深：

- 物理上是怎么把内存写到磁盘？fork + COW 的代价
- `appendfsync` 三档（always / everysec / no）的可靠性 vs 性能权衡
- 崩溃后启动时怎么恢复？加载顺序、损坏文件的处理
- "everysec 是怎么做到对延迟无感"的——背后 bio_aof_fsync 后台线程的精妙
- Redis 8 的多 RDB / 多 AOF 重大改进（multi-part AOF）

读完之后你应该能回答：

- "断电时 Redis 最多丢多少数据"——按当前配置精确算出来
- 一次 BGSAVE 真的"零阻塞"吗
- 写入 50 GB 数据集时，AOF rewrite 期间内存为什么会涨 2 倍
- 为什么 master 经常配 "no persistence"，只让 replica 持久化

---

## 第一章：RDB —— 内存快照

### 1.1 触发方式

```
# 配置：自动触发
save 3600 1       # 1 小时内有 1 个 key 变化
save 300 100      # 5 分钟内有 100 个 key 变化
save 60 10000     # 1 分钟内有 1 万个 key 变化

save ""           # 关闭自动 RDB（纯 AOF 或纯内存场景）

# 手动触发
SAVE              # 主线程阻塞执行，生产严禁
BGSAVE            # 后台执行，主线程不阻塞
DEBUG RELOAD      # 内部测试用，BGSAVE + 重启加载
```

### 1.2 BGSAVE 的真相：fork + COW

`BGSAVE` 不是"开个线程写盘"——它是 **fork 出一个子进程**，子进程拿着内存"快照"写 RDB 文件。

```
1) 主进程调用 fork()
   - 操作系统创建子进程，与父进程共享所有内存页（COW，Copy-On-Write）
   - 此刻子进程看到的内存就是 fork 那一刻的快照
2) 子进程遍历整个 dataset，按 RDB 格式写到磁盘临时文件
3) 写完后 rename 成正式 dump.rdb
4) 子进程退出

期间父进程继续接业务命令；任何写操作会触发对应内存页的 COW（操作系统克隆一份给父进程修改）。
```

### 1.3 fork 的代价

- **fork 本身**：复制父进程的页表。**不是复制所有内存数据**——只是页表 metadata。但对于一个用了 50GB 内存的 Redis，页表本身也有几百 MB。**fork 操作本身可能阻塞主线程 50-500 毫秒**——这是 BGSAVE 唯一一次"小阻塞"。
- **COW 期间的内存放大**：父进程对每个被修改的内存页都要克隆一份。极端情况下（fork 期间所有页都被修改），父进程内存翻倍。**典型业务**：fork 期间 RSS 涨 5-20%。
- **CPU 与 I/O**：子进程写盘期间消耗带宽和 CPU。

### 1.4 INFO 看 BGSAVE 状态

```
INFO persistence

rdb_bgsave_in_progress:0
rdb_last_save_time:1715587200
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:25            ← 上次 BGSAVE 耗时
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:524288000            ← fork 期间 COW 复制的内存量
```

`rdb_last_cow_size` 反映"fork 期间数据集修改强度"。这个数字 = fork 完成时复制的内存页总和。**长期看这个数值大致 = 你能容忍多少内存余量**。

### 1.5 RDB 文件格式

```
+--------+-----------+--------------+-----+---------+----+--------+
| MAGIC  | 版本号    | metadata     | DB  | ENTRY * | ... | CRC64 |
| REDIS  | 5 字节    | (aux fields) | 0/1 |         |     |        |
+--------+-----------+--------------+-----+---------+----+--------+
```

- MAGIC = `REDIS` 5 字节
- 版本号 = `0011` 之类（Redis 内部 RDB 版本，不同 Redis 大版本可能改）
- aux fields：版本号、UUID、used-mem、repl-stream-db、redis-bits（32/64）
- DB selector：进入第几个 DB
- 每条 ENTRY：type + key + value（按类型序列化）
- 末尾 8 字节 CRC64 校验

文件天然紧凑——比 dataset 占内存还小（因为只存数据本身，无 hashtable bucket、redisObject 头等开销）。**冷备份首选 RDB**。

### 1.6 RDB 优劣

**优**：

- 紧凑，文件小（典型 = dataset 60-80%）
- 加载快（启动加载比 AOF 重放快 5-10x）
- fork 后子进程异步写盘，主线程几乎不感知
- 适合**远程冷备份 / 灾难恢复**——一个 RDB 文件即一个时间点的完整快照

**劣**：

- 两次 RDB 之间断电会丢这段数据
- fork 在大 dataset 下昂贵（页表复制 + COW 内存涨）
- 不适合"分钟级 RPO" 要求的业务

---

## 第二章：AOF —— 操作日志

### 2.1 工作原理

AOF 把**每一条改写命令**追加到磁盘上的日志文件。崩溃后启动时按顺序重放整个日志重建状态。

```
client → SET foo bar
       ↓
   server 执行 → 写入 AOF buffer
                ↓
              （定期 fsync 到磁盘）
              ↓
         appendonly.aof
```

### 2.2 三档 fsync 策略

```
appendfsync always       # 每条命令都 fsync。最安全，性能差 10x+
appendfsync everysec     # 每秒 fsync 一次。最多丢 1 秒数据（默认 + 推荐）
appendfsync no           # 不主动 fsync，让 OS 自己决定（30 秒一次左右）
```

**always**：每次写都得等磁盘真的写入完成才返回。HDD 上 IOPS 上限约 100 → 100 写/秒。SSD 也只能到几千。**不推荐**——除非业务真的不能丢一条。

**everysec**：默认。后台线程每秒 fsync 一次。最多丢 1 秒。主线程零阻塞——把 fsync 工作交给 `bio_aof_fsync` 后台线程做。

**no**：性能最高但 fsync 时机由 OS 决定，断电可能丢几十秒。

### 2.3 everysec 怎么做到零阻塞

主线程做的事：

1. 命令执行后写到 `server.aof_buf` 缓冲区
2. 定时任务（每个事件循环）检测：上次 fsync 是否超过 1 秒？是 → 提醒 bio_aof_fsync 线程做 fsync

bio_aof_fsync 后台线程做的事：

1. 调用 `fsync(aof_fd)` 把已 write 的数据持久化
2. 完成

**主线程不等 fsync 完成**——异步触发即可。这是"everysec 零阻塞" 的关键。

唯一例外：如果上次 fsync 还没完成（业务高峰磁盘慢），主线程下次 write 时**会等 fsync 完成**——避免 AOF buffer 无限增长。这种"主线程意外被 fsync 阻塞"会在 `latency monitor` 显示为 `aof-fsync-always` 事件。

### 2.4 AOF rewrite —— 不可缺少的瘦身

AOF 是追加写——同一个 key 被改 1000 次，AOF 里就有 1000 条命令。文件会无限膨胀。

**AOF rewrite**：扫一遍当前 dataset，写出一份"等价的最小命令集"到新 AOF 文件，然后 rename 替换。

```
config set auto-aof-rewrite-percentage 100   # 文件大小翻倍就 rewrite
config set auto-aof-rewrite-min-size 64mb    # 文件至少 64MB 才考虑 rewrite
```

或手动 `BGREWRITEAOF`。

rewrite 也是 fork 子进程做（与 BGSAVE 同样的 COW 机制）。期间产生的新写入命令同时**追加到 AOF buffer + AOF rewrite buffer**，子进程写完后父进程把 rewrite buffer 里的命令追加到新文件末尾，原子 rename。

### 2.5 multi-part AOF —— Redis 7.0 的革新

Redis 6.x 及之前，AOF 是单一文件。问题：

- rewrite 期间额外占内存（rewrite buffer）
- rewrite 中途崩溃会有损坏的"半 rewrite 文件"
- 不支持"先 RDB 再 AOF 重放"的混合 + 多文件并存

Redis 7.0 改成 **multi-part AOF**：

```
appendonlydir/
├── manifest.aof                           ← 清单：列出当前 AOF 由哪些文件组成
├── appendonly.aof.1.base.rdb              ← BASE 文件：rewrite 完成时的"基线"（RDB 格式）
├── appendonly.aof.1.incr.aof              ← INCR 文件：BASE 之后的增量命令
└── appendonly.aof.2.incr.aof              ← 下一次 rewrite 产生的新 INCR（或没有）
```

- BASE 文件 = rewrite 产物，**默认是 RDB 格式**（混合 AOF）
- INCR 文件 = BASE 之后追加的命令日志
- manifest 文件 = 顺序索引

启动时按 manifest 顺序加载 BASE + 各个 INCR。**rewrite 期间不再需要 rewrite buffer**——父进程把新命令写到下一个 INCR 文件即可，子进程独立写 BASE 文件，互不干扰。

这是 Redis 7+ 的重大改进——内存占用降、操作原子性强、可分阶段恢复。

---

## 第三章：混合持久化（RDB-in-AOF）

### 3.1 默认开启（Redis 4.0+）

```
aof-use-rdb-preamble yes     # 默认 yes
```

AOF rewrite 时，新文件的 BASE 部分用 **RDB 格式**（不是命令日志格式）。后续 INCR 才是普通命令日志。

为什么这样设计？

- RDB 加载快 5-10x（二进制紧凑 + 直接构造内存结构）
- AOF 命令格式适合"增量"——少量命令重放快
- 两者结合：**rewrite 后的 BASE 走 RDB 速度，INCR 走 AOF 精度**

启动时：

1. 加载 BASE（RDB 格式，一次性加载到内存）
2. 顺序重放各 INCR 文件（命令格式，逐条重放）

### 3.2 一个崩溃恢复实例

时刻 T：dataset 50 GB
T+5min：AOF rewrite 完成，BASE.rdb = 50GB
T+10min：业务写入产生 INCR.aof = 200MB
T+10min:30s：断电

重启加载：

1. 加载 50GB BASE（~30 秒，RDB 解析快）
2. 重放 200MB INCR（~5 秒，命令重放慢但量小）

总恢复时间 ~35 秒。**如果纯 AOF**：50GB 全是命令日志，重放可能需要 5-10 分钟。

---

## 第四章：启动加载顺序

```
1) 配置 aof-use-rdb-preamble + AOF 开启 + appendonly.aof.* 存在
   → 从 appendonlydir/ 加载 (BASE.rdb + INCR.aof)
2) 仅开启 AOF
   → 从 appendonly.aof 加载（老版本格式，无 manifest）
3) 仅开启 RDB（appendonly no）
   → 从 dump.rdb 加载
4) 都开启但都不存在
   → 启动空 Redis
```

**关键原则**：AOF 优先于 RDB。如果 AOF 文件存在，永远不读 RDB。

### 4.1 文件损坏怎么办

```
redis-check-rdb dump.rdb              # 检查 RDB 是否损坏
redis-check-aof --fix appendonly.aof  # 检查 + 修复 AOF（截断末尾损坏部分）
```

AOF 文件末尾 N 字节损坏（断电常见场景）：

- 默认配置 `aof-load-truncated yes`：启动时打印警告，截断损坏部分继续加载
- `aof-load-truncated no`：拒绝启动，要求人工修复

**生产推荐 yes**——损坏 1 KB 通常只是丢最后几条命令，比拒绝启动有用得多。

---

## 第五章：性能权衡矩阵

| 配置 | RPO（最多丢） | 写入吞吐 | 启动恢复时间 | 适用场景 |
|---|---|---|---|---|
| 纯 RDB save 60 10000 | 60 秒数据 | 极高 | 快（30s/50GB） | 缓存 + 可丢 |
| 纯 RDB save 3600 1 | 1 小时数据 | 极高 | 快 | 完全可丢 |
| 纯 AOF everysec | 1 秒 | 高（5% 损失） | 慢（5min/50GB） | 大多数生产 |
| 纯 AOF always | 0 | 低（10x 慢） | 慢 | 金融级 |
| 混合 (默认) | 1 秒 + AOF 设置 | 高 | 中等（30s+5s） | **推荐** |
| 关闭（save '' + appendonly no） | 全部 | 最高 | 即开即空 | 纯缓存 + 可重建 |

### 5.1 业务选型

**电商订单系统**：必须 AOF everysec + 混合。哪怕断电丢 1 秒可能也要追溯订单——但行业普遍可接受。

**消息队列后端 / Streams**：AOF everysec。

**纯缓存 / Session 短 TTL**：关闭持久化即可。重启=丢数据，但 5 分钟全量预热回来。

**金融交易/支付**：用 Redis 不要持久化，直接走传统数据库主链路。Redis 仅做读缓存。

**配置中心 / 元数据**：RDB save 3600 1 即可。变更不频繁，1 小时备份足够。

---

## 第六章：fork 在大内存下的代价

### 6.1 现象

```
INFO memory:
used_memory: 100gb
used_memory_rss: 105gb       (碎片率 1.05，健康)

执行 BGSAVE 期间：
INFO stats: latest_fork_usec: 850000   (850ms！主线程阻塞了 0.85 秒)
INFO memory: used_memory_rss: 130gb    (COW 让 RSS 暂时涨 25%)
```

100GB Redis fork 一次：

- **fork 系统调用**：复制页表 ~500ms - 1500ms（依 OS、CPU、页表大小）
- **COW 内存膨胀**：业务写入引发 page 复制；典型 +5-20%

### 6.2 缓解

**1. 关闭 THP（Transparent Huge Pages）**

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

THP 把 4KB 页合成 2MB 大页。fork 期间任何 2MB 范围内任一字节变化都会 COW 整个 2MB——**放大 COW 512 倍**。Redis 官方明确建议关闭。

**2. 用 replica 持久化，master 不持久化**

```
master.conf:
  save ""
  appendonly no

replica.conf:
  save 3600 1
  appendonly yes
```

主节点专心服务业务请求，复制流推到从节点；从节点负责 fork + 持久化。**master 完全没有 fork 抖动**。生产大集群常用方案。

**3. 缩小 dataset**

如果一个实例 100GB 是因为存了过多冷数据——拆 Cluster 或归档到对象存储。Redis 单实例**不应超过 25-50GB**（fork 时间能控制在 100-300ms 范围）。

**4. 用支持的虚拟化**：

避免在 Xen 等老 Hypervisor 上跑大内存 Redis——fork 系统调用慢。KVM、Linux 容器、裸机均可。

---

## 第七章：observability 与诊断

### 7.1 三组关键日志

```
# 持久化健康检查
INFO persistence

# 看上次 fork 多慢
INFO stats | grep -E "latest_fork_usec|total_forks"

# 看 AOF 状态
INFO persistence | grep -E "aof_|loading"
```

`latest_fork_usec` > 100000（100ms）就要警觉；> 500000 必查。

### 7.2 LATENCY 框架

```
CONFIG SET latency-monitor-threshold 100   # 阈值 100ms

# 跑业务一段时间后
LATENCY HISTORY fork
LATENCY HISTORY aof-write
LATENCY HISTORY aof-fsync-always
LATENCY DOCTOR     # 人话报告
```

`LATENCY DOCTOR` 输出例：

```
Dave, I have observed latency spikes in this Redis instance.
You can use it to fix specific issues.

1) High AOF fsync latency: 130 ms. The OS is unable to fsync your AOF
   in less than 130 ms. This is likely due to slow disk I/O.
   Suggestions: ...
```

### 7.3 监控 dashboard 必备项

- `redis_persistence_loading` —— 启动加载耗时
- `redis_rdb_last_save_time_seconds` —— 上次 BGSAVE 完成时间
- `redis_rdb_last_bgsave_duration_sec` —— 上次 BGSAVE 耗时
- `redis_rdb_last_cow_size_bytes` —— 上次 fork COW 内存
- `redis_aof_last_rewrite_duration_sec` —— 上次 AOF rewrite 耗时
- `redis_aof_pending_fsync` —— pending fsync 字节数
- `latest_fork_usec` —— 单点告警 > 500ms

---

## 第八章：生产级最佳实践

1. **AOF + 混合 = 大多数业务的默认**：`appendonly yes` + `aof-use-rdb-preamble yes` + `appendfsync everysec`。
2. **`save ""` 关闭自动 RDB**：避免和 AOF rewrite 同时 fork 双重压力。需要远程备份就 cron 定时 `BGSAVE` 后拷文件。
3. **大实例 master 不持久化**：让 replica 干这个活；用 `MIN-REPLICAS-TO-WRITE 1` 保证至少有 replica 正常。
4. **`THP never` 是必须**：装机第一件事，写进 Ansible / 启动脚本。
5. **磁盘选 SSD**：HDD 的 fsync 延迟 5-20ms，业务 P99 永远爬不下去。SSD 是 < 1ms。
6. **`stop-writes-on-bgsave-error yes`**：默认就是。BGSAVE 失败时 Redis 拒绝写入（避免静默丢数据）。
7. **AOF rewrite 时间窗错峰**：默认 `auto-aof-rewrite-percentage 100` 容易在业务高峰触发。改用 `auto-aof-rewrite-percentage 50 + auto-aof-rewrite-min-size 1gb` + cron 在业务低峰 `BGREWRITEAOF`。
8. **磁盘空间预留 2x dataset**：RDB / AOF rewrite 都要写新文件，旧文件未删除前并存。
9. **不要 RDB 和 AOF 都关**：除非真的可重建（如纯缓存）。否则进程重启就是全军覆没。
10. **定期 redis-check-rdb / redis-check-aof**：备份脚本里加校验，防止备份本身损坏没察觉。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：以为 BGSAVE 真"零阻塞"
fork 系统调用本身阻塞主线程几十到几百毫秒。在大内存 Redis 上是显著的。

### ❌ 陷阱 2：THP 开着 + 大内存
COW 单位从 4KB 变 2MB，COW 内存放大 500 倍。fork 期间 RSS 可能直接翻倍 OOM。

### ❌ 陷阱 3：master + replica 都开持久化
同时 fork 会导致内存峰值翻倍 + 双倍磁盘 I/O。让 replica 单独做。

### ❌ 陷阱 4：appendfsync always 跑高频写
等同把 Redis 退化到 HDD 100 写/秒的水平。

### ❌ 陷阱 5：以为关掉 AOF 就一定快
AOF 关闭后还有 RDB save，自动触发同样 fork。要真的"无持久化"两个都关：`save '' + appendonly no`。

### ❌ 陷阱 6：磁盘满 = Redis 拒写
`stop-writes-on-bgsave-error yes` 时 BGSAVE 一次失败，Redis 此后所有写命令返回 `MISCONF Redis is configured to save RDB...`。立刻清磁盘或临时 `CONFIG SET stop-writes-on-bgsave-error no`。

### ❌ 陷阱 7：AOF 文件被外部 dd 复制时正在 rewrite
得到一个损坏的 AOF。备份要么 `BGSAVE` 后拷 RDB，要么用 LVM snapshot 整目录。

### ❌ 陷阱 8：升级 Redis 6 → 7 不知道 AOF 改成 multi-part
Redis 7 启动时检测到老 `appendonly.aof` 会自动转换为 multi-part 格式，但**这次转换没有备份**——升级前手动备份。

### ❌ 陷阱 9：误把 RDB 文件名改成 .aof
启动时按文件名识别格式。改错名直接报错或乱加载。

### ❌ 陷阱 10：rdbcompression no + 大量字符串
默认 `rdbcompression yes` 用 LZF。关掉后 RDB 文件涨 3-5x，加载更慢、备份更贵。除非 CPU 极紧张否则别关。

---

## 第十章：练习题

**练习 1**：业务高峰期间用户报"偶发 200ms 延迟尖刺"。`INFO stats` 显示 `latest_fork_usec: 280000`、`total_forks: 24`。给出诊断 + 修复方案。

**练习 2**：`appendfsync everysec`，每秒写 10000 命令，每条命令产生 100 字节 AOF 日志。算出 AOF 文件每天增长多少、rewrite 频率、rewrite 期间内存增量。

**练习 3**：设计一套备份策略，要求 RPO < 5 分钟、RTO < 2 分钟、备份不影响业务、远程异地保留 7 天。

**练习 4**：解释 `aof-load-truncated yes` 在断电场景下"丢最后几条命令" 而非"拒绝启动"的工程合理性。

**练习 5**：dataset 100 GB 的 Redis 想从 `save 3600 1` + AOF 切换到"纯 AOF everysec"。给出零停机切换步骤。

---

## 参考答案

**练习 1**：fork 引起。280ms 是单次 fork 时间，在 ~50GB+ dataset 下符合预期。修复优先级：

1. **关 THP**：`echo never > .../enabled`，确认 `INFO server: thp` 报告 never
2. **看 save 配置**：若有 `save 60 10000` 这种激进配置，改成只在低峰手动 BGSAVE
3. **架构层**：让 replica 做持久化，master 关 save + AOF

**练习 2**：
- AOF 日增 = 10000 × 100 × 86400 = 86 GB/天
- 假设 dataset 实际 50 GB 不增，AOF rewrite 阈值默认 `100% + 64MB`：第一次 rewrite 后 AOF ~50GB，再翻倍 = 100GB 触发下一次。生产实际 1-2 次/天 rewrite
- rewrite 期间：fork + COW ~5-15% 内存膨胀 = +2.5-7.5 GB；rewrite buffer / INCR 期间临时 buffer ~几 MB-几百 MB；rewrite 完成后老 AOF 删除前并存 → 磁盘临时增 100+ GB

**练习 3**：
- master：`appendonly yes + everysec + aof-use-rdb-preamble yes`，AOF rewrite cron 凌晨 3 点；`save ""` 关闭 RDB
- replica：跟 master 异步复制（默认）；replica 上加 `save 300 100` —— 每 5 分钟 BGSAVE 一次，达到 RPO < 5 分钟目标
- 备份：每 BGSAVE 完成后 cron 脚本检测 `rdb_last_save_time`，新文件 rsync 到对象存储；保留 7 天滚动
- RTO：恢复 = 从对象存储拉 RDB → 新实例启动加载（100GB ~ 60 秒）+ 应用层切换 DNS → 2 分钟内

**练习 4**：断电时 OS page cache 里的 AOF 数据没 fsync 到磁盘 → 重启后文件末尾几 KB-几 MB 是"半条命令"或"无效字节"。`aof-load-truncated yes` 自动截断从最后一条**完整**命令之后的部分，丢失的就是 fsync 间隔（最多 1 秒，everysec 模式）。**工程合理性**：业务多数能容忍 1 秒丢失；"拒绝启动"反而导致更长时间不可用，恢复成本高。

**练习 5**：

```bash
# 1. 当前是 RDB + AOF off
redis-cli CONFIG GET appendonly        # "no"

# 2. 在线启用 AOF（不停服）
redis-cli CONFIG SET appendonly yes
# 此时 Redis 会自动做一次 BGREWRITEAOF 创建 base.rdb，产生第一份 AOF
# INFO persistence 看 aof_rewrite_in_progress=1，等其完成（100GB 大约 30-90 秒）

# 3. 等 aof_rewrite_in_progress=0 且 aof_enabled=1 后，关闭 RDB
redis-cli CONFIG SET save ""

# 4. 写持久化配置文件（避免重启丢配置）
redis-cli CONFIG REWRITE

# 5. 验证
redis-cli CONFIG GET appendonly      # yes
redis-cli CONFIG GET save            # ""
ls -la /var/lib/redis/appendonlydir/ # 看到 base.rdb + manifest
```

整个过程在线、不停服、不丢数据。

---

## 小结

| 维度 | RDB | AOF | 混合 |
|---|---|---|---|
| RPO | 取决于 save 配置 | 0-1 秒 | 0-1 秒 |
| 写吞吐影响 | 仅 fork 期间 | 持续小幅 | 持续小幅 |
| 启动恢复 | 快（30s/50GB） | 慢（5min/50GB） | 中（35s/50GB） |
| 文件大小 | 紧凑 | 大 | base.rdb 紧凑 + incr 小 |
| 冷备份适用 | ★★★★★ | ★★ | ★★★★ |
| 增量精度 | 差（save 间隔） | 极好 | 极好 |

四条铁律：

1. **混合 + everysec 是 90% 业务的默认**
2. **THP 永远关闭**
3. **大 master 不持久化，让 replica 干**
4. **磁盘永远是 SSD**

下一篇 **R04 — 精通 Redis 主从复制与 Sentinel**：PSYNC2、部分重同步、Sentinel quorum / split-brain、故障转移流程。
