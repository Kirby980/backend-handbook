# Redis 深度课程 · 总目录

> 12 篇中文深度课程，覆盖 Redis 从底层数据结构到生产实践
> 每篇约 10000-15000 字，含源码级原理、命令深度、生产陷阱、练习题
> 适合从中级到高级后端 / SRE 工程师
>
> **📅 内容基准：Redis 8.0**（2025-05 发布，AGPLv3 重新开源）+ **Valkey 8.x**（2024 LF fork，BSD-3）
> 命令与协议层 100% 兼容，原理章节同时适用两者；差异点在 R11 专门拆解

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| R01 | [精通 Redis 数据结构内部](./01-精通-Redis-数据结构内部.md) | ⭐⭐⭐⭐⭐ | SDS / listpack / quicklist / hashtable / skiplist / intset / Stream Radix |
| R02 | [精通 Redis 内存管理与过期回收](./02-精通-Redis-内存与过期.md) | ⭐⭐⭐⭐ | maxmemory / 8 种 eviction / lazy + active expire / active defrag |
| R03 | [精通 Redis 持久化机制](./03-精通-Redis-持久化.md) | ⭐⭐⭐⭐ | RDB / AOF / 混合 / fsync 策略 / 数据安全等级 |
| R04 | [精通 Redis 主从复制与 Sentinel](./04-精通-Redis-复制与-Sentinel.md) | ⭐⭐⭐⭐ | PSYNC2 / 部分重同步 / 故障转移 / split-brain |
| R05 | [精通 Redis Cluster 集群](./05-精通-Redis-Cluster.md) | ⭐⭐⭐⭐⭐ | 16384 slot / gossip / MOVED/ASK / resharding |
| R06 | [精通 Redis 事务、Pipeline 与脚本](./06-精通-Redis-事务-脚本.md) | ⭐⭐⭐⭐ | MULTI/EXEC/WATCH / Pipeline / Lua / Functions |
| R07 | [精通 Redis Streams 与发布订阅](./07-精通-Redis-Streams.md) | ⭐⭐⭐⭐ | XADD/XREAD / Consumer Group / Sharded Pub/Sub |
| R08 | [精通 Redis 性能与单线程模型](./08-精通-Redis-性能模型.md) | ⭐⭐⭐⭐⭐ | 单线程 / I/O 多线程 / latency monitor / slowlog |
| R09 | [精通 Redis 8 新数据类型](./09-精通-Redis-8-新类型.md) | ⭐⭐⭐⭐ | JSON / TimeSeries / Bloom 系列 / Vector Set |
| R10 | [精通 Redis 生产实践与陷阱](./10-精通-Redis-生产实践.md) | ⭐⭐⭐⭐ | bigkey / hotkey / 缓存击穿穿透雪崩 / ACL |
| R11 | [Redis vs Valkey：fork 后的两条路](./11-精通-Redis-vs-Valkey.md) | ⭐⭐⭐ | 许可证 / 治理 / 迁移决策 |
| R12 | [精通 Redis 客户端与连接管理](./12-精通-Redis-客户端.md) | ⭐⭐⭐⭐ | RESP3 / 连接池 / Cluster aware / 长连接 |

---

## 🗺️ 按模块组织

### 🟢 模块一：底层内核（R01-R03）
> 数据怎么存、内存怎么管、断电不丢——Redis 性能与可靠性的物理基础。

### 🔵 模块二：分布式（R04-R05）
> 单点 → 主从 → 哨兵 → 集群。可用性与扩展性怎么取舍。

### 🟡 模块三：编程模型（R06-R07）
> 事务、Pipeline、Lua/Functions、Streams、Pub/Sub——把 Redis 当应用框架用。

### 🔴 模块四：性能与新特性（R08-R09）
> 单线程为什么不慢，I/O 多线程怎么用；Redis 8 八个新数据类型怎么落地。

### 🟠 模块五：生态与生产（R10-R12）
> 真正上线后要应对的：bigkey、hotkey、迁移、客户端选型。

---

## 🎯 学习路径

### 路径 A：从零到生产（4 周）
按编号顺序通读。每篇配套：跑一遍命令、画一遍数据结构图、写一段会触发该机制的代码。

### 路径 B：面试速成（1 周）
**R01 数据结构** + **R03 持久化** + **R05 Cluster** + **R08 性能模型** + **R10 生产陷阱**——5 篇覆盖 80% 面试点。

### 路径 C：架构师特化（2 周）
**R02 内存** + **R04 复制** + **R05 Cluster** + **R08 性能** + **R10 生产**——选型 / 容量规划 / 故障预案的核心。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[redis.io/docs](https://redis.io/docs/latest/)
- **源码**：[github.com/redis/redis](https://github.com/redis/redis)（master 分支 = Redis 8.x，关注 `src/t_string.c` / `src/dict.c` / `src/networking.c`）
- **Valkey 源码**：[github.com/valkey-io/valkey](https://github.com/valkey-io/valkey)

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 启动单节点 | `redis-server --daemonize yes` |
| 启动 Sentinel | `redis-sentinel /etc/redis/sentinel.conf` |
| 启动 Cluster（6 节点）| `redis-cli --cluster create 127.0.0.1:7000 ... --cluster-replicas 1` |
| 命令行连接 | `redis-cli -h host -p 6379 -a password --tls --cacert ca.pem` |
| 监控慢命令 | `SLOWLOG GET 10` |
| 实时命令流 | `MONITOR`（生产慎用，性能极大下降） |
| latency 诊断 | `LATENCY DOCTOR` / `LATENCY HISTORY event` |
| 大 key 扫描 | `redis-cli --bigkeys` 或 `MEMORY USAGE key` |
| 内存分析 | `MEMORY STATS` / `INFO memory` |
| Cluster 状态 | `CLUSTER NODES` / `CLUSTER INFO` / `CLUSTER SLOTS` |
| RDB / AOF 切换 | `CONFIG SET save ""` / `CONFIG SET appendonly yes` |
| 在线 rewrite AOF | `BGREWRITEAOF` |
| 在线快照 | `BGSAVE` |
| ACL 列表 | `ACL LIST` / `ACL WHOAMI` |
| 模块加载 | `MODULE LOAD /path/to/module.so` |

---

## ✅ 完读检查清单

- [ ] 解释 listpack 取代 ziplist 的根因（CVE-2020-1.x cascading update）
- [ ] 画出 hash 类型的两种内部表示（listpack ↔ hashtable）切换阈值
- [ ] 说明 sorted set 为什么用 skiplist 而不是 B-tree
- [ ] 设计一个不会触发 cluster split-brain 的部署
- [ ] 写出一个支持原子"扣库存 + 写日志"的 Lua 脚本
- [ ] 解释 PSYNC2 partial resync 与 PSYNC1 的差别
- [ ] 给一个真实业务设计 Cluster slot 分布与 hash tag 用法
- [ ] 用 latency monitor + slowlog 定位一次 P99 飙升
- [ ] 把一个 50GB 的 bigkey hash 拆掉而不停服
- [ ] 选择 Redis 8 还是 Valkey 8 并说出理由

---

> 📁 本目录位于 `/data/workspace/dp4/redis/INDEX.md`
> 🔁 反馈：每篇都建议起一个本地 Redis 跑一遍命令再下结论
