# 精通 Redis Cluster：16384 slot、gossip、MOVED/ASK、resharding

> 关联章节：[06 复制与 Sentinel](./04-精通-Redis-复制与-Sentinel.md)、[10 性能模型](./08-精通-Redis-性能模型.md)、[14 客户端](./12-精通-Redis-客户端.md)

---

## 引言：从主从到水平扩展

主从复制（R04）解决了高可用问题，但**单个 master 仍然是容量与吞吐的天花板**：

- 单节点内存通常 < 25-50GB（fork 时间、运维代价）
- 单节点 QPS < 100k（单线程瓶颈）

业务再大就需要**水平分片**。Redis Cluster 是官方的分片方案——把数据按 key 散到多个分片（每分片仍是主从），客户端能感知集群拓扑直接路由。

这一章把 Cluster 拆开看：

- 16384 slot 是怎么映射 key 到节点的
- gossip 协议怎么让所有节点对集群状态达成一致
- MOVED 和 ASK 重定向的区别
- 在线 resharding（扩缩容）期间数据怎么搬而不停服
- 跨 slot 命令为什么不允许，hash tag 怎么绕开
- failover 在 Cluster 里和 Sentinel 模式有什么不同

---

## 第一章：16384 slot 模型

### 1.1 为什么是 16384

```
slot = CRC16(key) % 16384
```

每个 key 根据 CRC16 哈希分到 0-16383 中的一个 slot。每个 slot 归属一个 master 节点。

```
Cluster: 3 master + 3 replica
  Master A: slots 0-5460       (5461 slots)
  Master B: slots 5461-10922   (5462 slots)
  Master C: slots 10923-16383  (5461 slots)

  Replica A → 跟 Master A
  Replica B → 跟 Master B
  Replica C → 跟 Master C
```

为什么不是 64K 或 1024？历史决定：

- 16384 = 2^14。每个节点维护一份 slot map → 2KB bitmap（16384 bit）
- 在 gossip 消息里要带这个 bitmap → 2KB / 包，可接受
- 实际集群最大 ~1000 节点，16384 slot 足够细粒度

### 1.2 节点视角的 slot 表

每个节点都维护**完整的集群状态**：

```c
// src/cluster.h（简化）
typedef struct clusterState {
    clusterNode *myself;
    uint64_t currentEpoch;
    int state;          // OK / FAIL
    int size;           // 主节点数

    dict *nodes;        // nodeId → clusterNode
    clusterNode *slots[CLUSTER_SLOTS];  // slot 16384 → 归属节点
    ...
} clusterState;
```

任一节点接到 client 请求时，自己能立刻判断"这个 key 该路由到谁"——不需要 query 中心元数据。

### 1.3 CLUSTER INFO / NODES / SLOTS

```
$ redis-cli -p 7000 CLUSTER INFO
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3

$ redis-cli -p 7000 CLUSTER NODES
<nodeid> 10.0.0.1:7000@17000 myself,master - 0 1715587200000 1 connected 0-5460
<nodeid> 10.0.0.2:7000@17000 master - 0 1715587200000 2 connected 5461-10922
<nodeid> 10.0.0.3:7000@17000 master - 0 1715587200000 3 connected 10923-16383
<nodeid> 10.0.0.4:7000@17000 slave <master_a_id> 0 1715587200000 4 connected
...

$ redis-cli -p 7000 CLUSTER SLOTS
1) 1) (integer) 0
   2) (integer) 5460
   3) 1) "10.0.0.1"          ← master
      2) (integer) 7000
      3) "abc123..."
   4) 1) "10.0.0.4"          ← replica
      2) (integer) 7000
      3) "def456..."
2) ...
```

`CLUSTER NODES` 是运维最常看的——一眼判断集群健康。

---

## 第二章：Gossip 协议

### 2.1 节点怎么互相发现 + 同步状态

Cluster 内每个节点都有一个 **bus port**（默认数据端口 + 10000，如 7000 数据 / 17000 bus）。节点间通过 bus port 发 gossip 消息：

```
每 100ms：
  随机选 1 个节点（优先选很久没通讯的）
  发 PING 包，内含：
    - 我的状态（master/replica、连接 master 是谁）
    - 我对 N 个随机节点的"间接观测"（gossip 字段：每个节点的 ip/port、最近 ping/pong 时间、状态标记）
    - 我的 slot 配置（哪些 slot 在我手里）
对方回 PONG，内容同上
```

效果：**所有节点最终都能感知所有节点的状态**（最终一致性），无需中心点。

### 2.2 标记节点为 PFAIL / FAIL

```
1) 节点 A 给节点 B 发 PING，cluster-node-timeout（默认 15s）内没收到 PONG
   → A 把 B 标记为 PFAIL（probable failure，主观下线）
2) gossip 把"A 认为 B 是 PFAIL"传给其他节点
3) 半数以上 master 都标记 B 为 PFAIL
   → 升级为 FAIL（客观下线）
4) B 的 replica 触发 failover 流程（见 §6）
```

这套机制和 Sentinel 的 SDOWN/ODOWN 类似，但**没有独立的"sentinel"角色**——所有节点都参与。

### 2.3 关键配置

```
cluster-node-timeout 15000           # 多久无响应判 PFAIL（毫秒）。生产 5-15s
cluster-announce-ip 10.0.0.1         # NAT / 容器场景显式声明对外 IP
cluster-announce-port 7000
cluster-announce-bus-port 17000
cluster-replica-validity-factor 10   # replica 数据落后 (timeout × factor) 才有资格 failover
cluster-migration-barrier 1          # master 至少保留多少 replica 才允许迁移
cluster-require-full-coverage yes    # 任一 slot 失联整个集群拒写（防数据不一致）
```

---

## 第三章：MOVED 与 ASK 重定向

### 3.1 client 算 slot 但可能算错

理想：client 知道集群拓扑 → 算 slot → 直连对应节点。但拓扑会变（扩缩容），client 缓存可能过期。

Redis Cluster 的设计：**任何节点收到不属于自己的 key 请求，回 MOVED / ASK 让 client 重定向**。

### 3.2 MOVED

```
client → nodeA: SET foo bar
nodeA: slot 1234 在 nodeB
nodeA → client: -MOVED 1234 10.0.0.2:7000
client：更新本地 slot map → 重发到 nodeB
```

MOVED = **永久性重定向**：这个 slot 现在归 nodeB，以后都找它。client 应当：

1. 更新本地 slot 缓存
2. 重新发命令到目标节点

### 3.3 ASK

发生在 **slot 正在迁移中**：

```
slot 1234 正从 nodeA 迁到 nodeB（migrating 状态）
client → nodeA: GET foo
  - 如果 foo 还在 nodeA：正常返回
  - 如果 foo 已经迁走：返回 -ASK 1234 10.0.0.2:7000
client → nodeB: ASKING + GET foo
  - 必须先发 ASKING 命令告诉 nodeB "我知道这个 slot 在迁移，临时给我服务"
  - 否则 nodeB（importing 状态）会拒绝（slot 还没正式归它）
```

ASK = **临时性重定向**：仅这一次请求转去 nodeB，**不更新本地 slot 缓存**（因为迁移完成后还是 nodeB 归属，但缓存现在更新会污染其他 key）。

### 3.4 client 实现要点

| 行为 | MOVED | ASK |
|---|---|---|
| 更新本地 slot 表 | ✓ | ✗ |
| 下次同 slot 的 key 直接路由到新节点 | ✓ | ✗ |
| 重发请求时需要 ASKING | ✗ | ✓ |

主流 SDK（Lettuce、redis-py、StackExchange.Redis、go-redis 等）都内置这套逻辑。**自己写 client 90% 的坑在这里**。

---

## 第四章：在线 Resharding

### 4.1 工具：redis-cli --cluster

```
# 扩容：加新节点
redis-cli --cluster add-node 10.0.0.5:7000 10.0.0.1:7000
redis-cli --cluster add-node 10.0.0.6:7000 10.0.0.1:7000 --cluster-slave \
          --cluster-master-id <master-d-id>

# 把 slot 从老节点迁到新节点
redis-cli --cluster reshard 10.0.0.1:7000
  → How many slots? 4096
  → Receiving node ID? <master-d-id>
  → Source node IDs? all（或具体 ID）

# 平衡
redis-cli --cluster rebalance 10.0.0.1:7000

# 缩容：迁出 + 移除
redis-cli --cluster reshard ...  # 把 master-d 的 slot 迁回其他节点
redis-cli --cluster del-node 10.0.0.1:7000 <node-d-id>
```

### 4.2 一个 slot 迁移的内部步骤

```
1) source 标记 slot=N 为 MIGRATING → target
2) target 标记 slot=N 为 IMPORTING ← source
3) 循环：
   - 从 source 取 100 个 key（属于 slot=N）：CLUSTER GETKEYSINSLOT N 100
   - 对每个 key：MIGRATE target ... AUTH ... REPLACE
4) 当 source 中 slot=N 为空：
   - CLUSTER SETSLOT N NODE <target_id>（先到 source）
   - 广播 gossip
   - 其他节点收到后更新 slot map
5) target 上 slot=N 正式归它
```

迁移期间该 slot 的命令：

- 在 source 上：找到 key → 正常服务；找不到 key → 返回 -ASK 重定向 client 去 target
- 在 target 上：未带 ASKING → -MOVED 回 source；带 ASKING → 服务这次请求

**业务零感知**——SDK 自动处理 MOVED/ASK。

### 4.3 MIGRATE 命令的代价

```
MIGRATE target_ip target_port "" 0 timeout KEYS k1 k2 k3 ...
```

行为：

- source 把 key 的值序列化（DUMP 内部格式）
- 通过 socket 发给 target
- target 反序列化 + 存入 dataset
- target 回 OK 后 source 删除该 key

**MIGRATE 是同步阻塞**——source / target 主线程都被卡住直到完成。大 key（如 1GB hash）的 MIGRATE 可能卡几秒——业务延迟暴增。

**修法**：

- 用 `redis-cli --cluster reshard --cluster-pipeline 10`（pipeline 一批 key 减少 RTT）
- 大 key 提前拆分（详见 R10 生产实践）

### 4.4 业务流量切换

resharding 完成后客户端会逐步通过 MOVED 更新缓存，**老 client 仍可能访问错节点**——只是多一次重定向，**不会出错**。所以 resharding 在业务侧无需协调。

---

## 第五章：跨 slot 命令与 hash tag

### 5.1 默认不允许多 key 跨 slot

```
> MSET k1 v1 k2 v2 k3 v3        # k1/k2/k3 可能不同 slot
(error) CROSSSLOT Keys in request don't hash to the same slot
```

Cluster 强制：**单个命令涉及的所有 key 必须在同一 slot**。

原因：

1. 同一命令可能要原子操作多个 key（如 MSET、SUNIONSTORE）
2. 跨节点原子是分布式事务问题（PAXOS / 2PC），Redis 拒绝引入这种复杂度
3. 让"原子语义"的边界 = "slot 边界" = "单节点边界"

### 5.2 Hash tag —— 强制路由到同 slot

```
key = "user:{42}:cart"     # {} 内的是 hash tag
key = "user:{42}:profile"

CRC16("42") % 16384   ← 仅对花括号内的字符做 CRC16
所以 user:{42}:cart 和 user:{42}:profile 一定同 slot
```

**用法**：把同一业务对象的所有相关 key 用同一 tag → 强制路由到同分片 → 单节点上可以原子操作 / 用事务 / Lua / MGET。

```
MSET user:{42}:name alice user:{42}:age 30          # OK，同 slot
MULTI
HSET user:{42}:cart item1 1
HINCRBY user:{42}:counters cart_size 1
EXEC                                                 # OK，单节点事务
```

### 5.3 Hash tag 的反模式

```
key = "tag:hot"   ← 用一个固定 tag 让所有 key 都到同一分片
```

会让该分片成为热点，违背分片的初衷。**hash tag 只该用于"业务上必然一起操作的对象"**，不该当成"绕过 cluster 限制的工具"。

---

## 第六章：Cluster 中的 Failover

### 6.1 自动 failover 与 Sentinel 模式的差别

Sentinel 模式：Sentinel 集群独立监督 master，集中决策切换。
Cluster 模式：**所有 master 节点共同监督**，replica 自己竞选当 master。

### 6.2 触发条件

```
1) Master B 死掉
2) 其他 master 通过 gossip 检测 → 标记 PFAIL → 多数确认 → 标记 FAIL
3) B 的 replicas 检测到 B 已 FAIL
4) 满足条件的 replica 发起选举：
   - replica 上次同步时间 < (cluster-node-timeout × cluster-replica-validity-factor)
   - 数据 offset 最大者优先
5) 候选 replica 向 master 节点发"投我一票"请求（current_epoch）
6) 拿到 majority master 投票（不是 replica 投票！）→ 当选
7) 新 master 广播 PONG（带新 slot 配置）
8) 其他节点更新拓扑
```

### 6.3 几个关键细节

- **投票方是 master 节点**（不像 Sentinel 投票方是 sentinel）
- **每个 master 在一个 epoch 只投一票**（防多个 replica 同时当选）
- **replica 数据落后太多没资格竞选**（`cluster-replica-validity-factor` 默认 10）

### 6.4 没有 replica 怎么办

某 master 死了但没有 replica：

- 该 slot 范围**完全不可用**
- 默认 `cluster-require-full-coverage yes` → **整个集群拒写**（防止部分写入导致一致性问题）
- 改 `no` → 其他 slot 仍可用，但失联 slot 的请求返回 CLUSTERDOWN

**生产铁律：每个 master 至少 1 replica**。

### 6.5 手动 failover

```
# 在某个 replica 上：
CLUSTER FAILOVER         # 普通模式：先确保数据同步完成再切
CLUSTER FAILOVER FORCE   # 强制：不等同步
CLUSTER FAILOVER TAKEOVER # 紧急：连 master 投票都不要（脑裂风险）
```

适用：滚动升级、机器迁移、master 计划性维护。

---

## 第七章：客户端的 Cluster-aware 实现

### 7.1 标准流程

```
1) 启动：根据配置连接任一 cluster 节点
2) 执行 CLUSTER SLOTS / CLUSTER NODES → 拿到完整 slot → 节点映射
3) 命令执行前：
   - 计算 slot = CRC16(key) % 16384
   - 查本地 slot 表 → 直连目标节点
4) 收到 -MOVED：
   - 更新本地 slot 表（增量 / 全量 CLUSTER SLOTS）
   - 重发命令到目标
5) 收到 -ASK：
   - 临时去新节点 + 加 ASKING 前缀
   - 不更新本地 slot 表
6) 周期性刷新 slot 表（如每 60s）防止变化
```

### 7.2 连接池策略

每节点维护一个连接池：

```
go-redis 默认：
  PoolSize: 10 × CPU 数（per-node）
  MinIdleConns: 0
  MaxConnAge: 0（永不老化）
```

**避免**：每节点一个池，但全集群 100 节点 × 100 conn = 1 万连接。生产实际 `PoolSize` 控制在 10-30/节点。

### 7.3 pipeline / 事务在 Cluster 下的限制

- **Pipeline**：多个命令的 key 在同一 slot 才能在一个 pipeline 里。SDK 通常会"按 slot 自动拆分"——一组 key 同 slot 走一个 pipeline，其他 key 走别的 pipeline。**透明但性能差异需感知**
- **事务（MULTI/EXEC）**：必须单 slot
- **Lua 脚本**：脚本内访问的所有 key 必须同 slot

---

## 第八章：监控关键指标

```
INFO cluster                          # cluster_enabled / cluster_state
CLUSTER INFO                          # 集群状态 + slot 覆盖
CLUSTER COUNT-FAILURE-REPORTS <id>    # 多少节点报告该节点 PFAIL
CLUSTER COUNTKEYSINSLOT N             # 某 slot 有多少 key
```

Prometheus alert：

- `cluster_state != ok`
- `cluster_slots_ok < 16384`（有 slot 失联）
- `cluster_known_nodes` 突变（节点失联或新增未授权）
- 各分片 used_memory 差异 > 30%（不均衡，需 rebalance）
- 各 master 各自 lag / fragmentation 同 Sentinel 监控

---

## 第九章：生产级最佳实践

1. **至少 3 master + 3 replica**——任一节点挂仍能 failover
2. **节点跨机架 / AZ**——master 和它的 replica 分散
3. **`cluster-node-timeout 5000-15000`**——过短易误判，过长 failover 慢
4. **`cluster-require-full-coverage yes`**——除非业务能接受局部不可用
5. **hash tag 谨慎使用**——只用于真正需要原子的对象
6. **大 key 不要落到 cluster**——MIGRATE 期间会卡 source/target；先拆
7. **新加节点先放空跑几天**——保留余量；满了再 reshard 风险高
8. **client 用最新 SDK**——MOVED/ASK 逻辑稳定，连接管理好
9. **不要混用 cluster 和 sentinel SDK**——两套协议
10. **每次 reshard 在业务低峰**——MIGRATE 是同步阻塞，量大时延迟可见

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：跨 slot 命令报错
`MGET k1 k2 k3` 在 cluster 上可能 CROSSSLOT。SDK 通常会自动按 slot 拆分多次 MGET 但失去原子。

### ❌ 陷阱 2：用 KEYS / SCAN
`KEYS *` 只扫单节点。`SCAN` 也只在单节点内迭代——要遍历整个 cluster 必须 `--cluster call ... SCAN ...` 在所有节点跑。

### ❌ 陷阱 3：客户端缓存的 slot 表过期
故障转移后客户端不知道，仍连老 master IP → 报错。SDK 必须订阅 +SLOT 更新或周期刷新。

### ❌ 陷阱 4：MIGRATE 大 key 卡主线程
50MB hash 的 MIGRATE 可能让两边主线程都阻塞几秒。先拆 key 再 reshard。

### ❌ 陷阱 5：cluster-require-full-coverage no 但应用没准备好
slot 局部失联时部分 key 返回 CLUSTERDOWN，应用如果没处理这种错误就报 5xx。

### ❌ 陷阱 6：hash tag 用错
`user:42:cart` 不会自动同 slot；要 `user:{42}:cart`。注意花括号位置。

### ❌ 陷阱 7：bus port 没开
默认 dataport + 10000。若机器只开了 7000 没开 17000 → gossip 失败 → 节点互相看不到。

### ❌ 陷阱 8：NAT / 容器环境忘配 cluster-announce
节点 PING 时报告自己的 IP，如果是容器内部 IP，对端无法回连。必须 `cluster-announce-ip` 设外部可达地址。

### ❌ 陷阱 9：用 Cluster 但只 3 个节点（无 replica）
任一 master 挂 → 1/3 数据不可用。最少 3master + 3replica。

### ❌ 陷阱 10：Cluster 当 Sentinel 用
有人误以为 Cluster 不分片也能用——但 Cluster **强制要求 slot 覆盖 16384**。单纯 HA 用 Sentinel；要分片用 Cluster。

---

## 第十一章：练习题

**练习 1**：业务对象"用户购物车"包含 `user:{id}:cart`、`user:{id}:cart_meta`、`user:{id}:counters`。说明 hash tag 怎么让它们能用 MULTI/EXEC。

**练习 2**：3 master + 3 replica，cluster-node-timeout 15s，cluster-replica-validity-factor 10。master A 宕机后，多久内不能 failover？

**练习 3**：从 3 master → 6 master 扩容。给出步骤、影响、监控点。

**练习 4**：业务 key 设计为 `order:12345`（订单 ID 全局唯一）。某个分片 QPS 比其他高 3 倍。诊断 + 修复。

**练习 5**：client 报"CROSSSLOT" 错误。给出可能的 5 种原因和对应解法。

---

## 参考答案

**练习 1**：所有 key 用同一 tag `{42}`（其中 42 是 user_id）：

```
SET    user:{42}:cart "..."
HSET   user:{42}:cart_meta updated_at 1715587200
INCR   user:{42}:counters:cart_size
```

slot 由 `42` 决定 → 都在同一节点。然后：

```
MULTI
HSET user:{42}:cart_meta last_op "add_item"
LPUSH user:{42}:cart item_5
INCR user:{42}:counters:cart_size
EXEC
```

整个事务在单节点原子执行。

**练习 2**：master A 死了 → 其他 master 等 15s 才能标 PFAIL → 多数确认升 FAIL → A 的 replica 看到 A FAIL 后才发起选举。replica 数据 lag 必须 < 15 × 10 = 150s 才有资格。**最快 ~15-20 秒**（detection + 选举 + slot 配置广播）。

**练习 3**：
1. 加 3 个新 master + 3 个新 replica（CLUSTER MEET）
2. 用 `redis-cli --cluster reshard` 把 16384 slot 重分：从 3 老 master 各分约 2730 slot 出来给新节点
3. 影响：
   - 业务零停机（SDK 自动处理 MOVED/ASK）
   - MIGRATE 大 key 期间延迟尖刺（业务侧观察 latency）
   - reshard 完成总时间 ~按 dataset 大小 + 网络速度；100 GB / 1Gbps 大约 30-60 分钟
4. 监控点：
   - `cluster_slots_ok` 仍 16384
   - 各分片 used_memory 趋于均衡
   - 业务 P99 latency 仍正常

**练习 4**：诊断：`order:12345` 没用 hash tag，但具体某分片 QPS 高可能是：

- "热点订单"：某些订单 ID 被高频访问 → key 设计问题，需热数据缓存到本地或 multi-tier
- slot 分布不均 → `--cluster rebalance` 重平衡
- 业务访问偏斜：所有"查询最近 10 个订单" 总是访问同一个时间段的 ID 段，碰巧落到同一 slot
- 单一统计 key 如 `total_orders` 不分片：换 `total_orders:{shard_n}` 多 key 分散

修复优先看 `CLUSTER COUNTKEYSINSLOT` 找异常 slot，再看具体 key 访问模式。

**练习 5**：可能原因：
1. 多 key 命令 + 不同 slot：用 hash tag 或拆成多个命令
2. Pipeline 里混 slot 但 SDK 没自动拆：换支持 cluster pipeline 的 SDK
3. MULTI/EXEC 多 key 跨 slot：必须同 slot
4. Lua 脚本里 redis.call 访问其他 slot 的 key：把所有 key 列在 KEYS 数组里并保证同 slot
5. SUNIONSTORE / ZUNIONSTORE 等聚合命令的 source 与 destination 跨 slot：手动 SUNION 取出再 SADD 到目标

---

## 小结

| 概念 | 含义 |
|---|---|
| Slot | 0-16383，CRC16(key) % 16384 |
| Hash Tag | `{...}` 内字符决定 slot |
| MOVED | 永久重定向，client 更新缓存 |
| ASK | 临时重定向（迁移中），不更新缓存 |
| Gossip | 节点间状态同步 |
| PFAIL / FAIL | 主观/客观下线 |
| Bus Port | data port + 10000 |

四条铁律：

1. **slot 是 cluster 一切的物理单位**——key→slot→node
2. **跨 slot 不允许多 key 命令**——hash tag 是唯一逃生口
3. **每 master 至少 1 replica**——否则单点失联整集群拒写
4. **MIGRATE 大 key = 集群杀手**——上 cluster 前先拆

下一篇 **R06 — 精通 Redis 事务、Pipeline 与脚本**：MULTI/EXEC/WATCH 真伪事务、Pipeline 与原子性、Lua 与 Functions 的本质区别。
