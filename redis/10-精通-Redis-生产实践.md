# 精通 Redis 生产实践与陷阱：bigkey、hotkey、三大缓存问题、ACL

> 关联章节：[04 内存与过期](./04-精通-Redis-内存与过期.md)、[07 Cluster](./07-精通-Redis-Cluster.md)、[10 性能模型](./10-精通-Redis-性能模型.md)

---

## 引言：生产事故的常客

把前面 R03-R11 的机制都搞懂之后，你已经能避免 70% 的生产坑。剩下 30% 来自**业务和 Redis 的耦合方式**——而不是 Redis 本身。

这一章把这 30% 列清楚：

- **bigkey**：单 key 超大，主线程操作卡顿，迁移困难
- **hotkey**：某 key QPS 占据单分片大半，分散不开
- **缓存击穿**：热点 key 失效瞬间，大量请求打到 DB
- **缓存穿透**：查询不存在的 key，每次都打到 DB
- **缓存雪崩**：大量 key 同时过期或 Redis 整体故障
- **ACL 与安全**：默认 0.0.0.0 监听 + 无密码 = 黑产入侵首选
- **客户端连接异常**：连接泄漏、不正确的连接池、超时不当

读完之后你应该能在 code review 时一眼识破常见问题，在故障复盘时知道挖到哪一层为止。

---

## 第一章：bigkey

### 1.1 定义与判断

| 类型 | bigkey 阈值（生产经验） |
|---|---|
| string | 单值 > 10 KB |
| hash | 字段数 > 1000 或单字段值 > 10 KB |
| list | 元素数 > 5000 或单元素 > 10 KB |
| set | 元素数 > 5000 |
| zset | 元素数 > 5000 |
| stream | 元素数 > 1 万（多由业务长期累积） |

### 1.2 危害

1. **主线程操作 O(N)**：`HGETALL` / `LRANGE 0 -1` / `SMEMBERS` 在大 key 上几十毫秒到秒级——卡所有 client
2. **网络阻塞**：单次返回 GB 级响应，把 NIC 占满
3. **DEL 卡主线程**：UNLINK 缓解 unlink 步但**实际释放仍占主线程一小段**
4. **MIGRATE 失败**：Cluster 扩容时一个大 key 卡死 reshard
5. **过期清理慢**：active expire 删大 key 时主线程长卡
6. **AOF 重写代价**：单个大 key 在 AOF 里展开为大量命令
7. **内存增长不均衡**：单 key 100GB 导致单节点失衡

### 1.3 发现

```bash
# redis-cli 自带工具
redis-cli --bigkeys                    # 扫一遍 top N 大 key（按类型分类）
redis-cli --memkeys                    # 按内存占用排
redis-cli --hotkeys                    # 按 LFU 频次排（需 maxmemory-policy=allkeys-lfu）

# 手动测量
MEMORY USAGE mykey SAMPLES 0           # 精确测量该 key 占用
DEBUG OBJECT mykey                     # serializedlength 看序列化大小
OBJECT ENCODING mykey                  # 当前 encoding（hashtable / listpack）
```

`--bigkeys` 是 SCAN 实现，不卡主线程，但**只是扫描后取最大的几个**，不一定是真正最大的（统计意义）。

### 1.4 优化：拆 key

#### 大 Hash 拆 buckets

```python
# 原：100 万字段的 hash
HSET huser:1234 field_0 v field_1 v ... field_999999 v

# 拆：100 个 hash 每个 ~ 1 万字段
def bucket(field):
    return hash(field) % 100

bucket_id = bucket(field)
r.hset(f"huser:1234:bucket:{bucket_id}", field, value)
r.hget(f"huser:1234:bucket:{bucket_id}", field)
```

实测 1 个 1 万字段 hash 在 listpack 阈值之上变成 hashtable；如果调大阈值（hash-max-listpack-entries=10000）反而不行——listpack 上 O(N) 查找在 1 万级别就慢了。所以**调小 bucket 大小 + 多桶**更优。

#### 大 List 拆段

```python
# 原：消息队列单 list 千万元素
LPUSH msgq:user msg1 msg2 ...

# 拆：按时间 / 范围分多个 list
day = time.strftime("%Y%m%d")
r.lpush(f"msgq:user:{day}", msg)

# 消费时按日期顺序处理
for day in get_days_to_consume():
    while True:
        msg = r.rpop(f"msgq:user:{day}")
        if not msg: break
```

#### Stream 用 MAXLEN ~ N

```python
r.xadd("events", {"k": "v"}, maxlen=1_000_000, approximate=True)
```

主动控制流长度。`~` 是近似截，比精确快 10x。

### 1.5 删除大 key

```
# 危险
DEL huser:1234         # 10MB hash 可能卡主线程几百 ms

# 安全
UNLINK huser:1234      # 异步释放
```

但 UNLINK 的 unlink 步本身仍是主线程操作（从 dict 摘除）；真正长卡的是大 hash 的内部元素释放——这部分异步。

更彻底的"渐进式删除"：

```python
# 大 hash 渐进删除
cursor = "0"
while True:
    cursor, fields = r.hscan("huser:1234", cursor=cursor, count=100)
    if fields:
        r.hdel("huser:1234", *fields.keys())
    if cursor == "0":
        break
r.delete("huser:1234")
```

每次 HDEL 100 个字段 ~ < 1ms，业务完全不感知。**生产清理大 key 的标准做法**。

### 1.6 监控

每天定时跑 `--bigkeys` 把 top 10 输出到日志：

```bash
redis-cli --bigkeys | grep -E "(Biggest|Sampled)" >> bigkey.log
```

或者用 `MEMORY USAGE` 对热点 key 自动告警。

---

## 第二章：hotkey

### 2.1 现象

业务上某些 key 的访问频率远高于其他：

- 商品详情：爆款商品 QPS 比普通商品高 1000 倍
- 配置 key：所有请求都读
- 排行榜：每次刷新都拉

集中到一个 Cluster 节点 → 该节点 CPU 100% 而其他节点闲着。

### 2.2 发现

```
# Redis 自带（要 maxmemory-policy=allkeys-lfu）
redis-cli --hotkeys

# 或 INFO commandstats 看哪类命令最频繁
INFO commandstats

# 客户端侧采样：业务层加 tracing
```

### 2.3 解法

#### 多副本读

Sentinel 模式：业务 client 读走 replica（写仍 master）。**replica 多 → 读吞吐可以横向扩展**。但写吞吐仍是 master 单点。

#### 客户端本地缓存

```python
# 读路径
val = local_cache.get(key)
if val is None:
    val = r.get(key)
    local_cache.set(key, val, ttl=10)   # 本地缓存 10 秒
return val
```

热点 key 在本地缓存命中 → 完全不打 Redis。**配合 Redis 6 RESP3 client tracking** 可以主动 invalidate。

#### Cluster 中拆分 key

```python
# 原：单 key 排行榜，QPS 全打到一个分片
ZADD leaderboard 100 alice 90 bob ...

# 拆：按 hash 分 16 个 leaderboard，业务层聚合
shard = hash(user) % 16
r.zadd(f"leaderboard:{{{shard}}}", {user: score})

# 查 top 100 → 各分片各取 top 100，合并
all_top = []
for s in range(16):
    all_top.extend(r.zrevrange(f"leaderboard:{{{s}}}", 0, 99, withscores=True))
top_100 = sorted(all_top, key=lambda x: -x[1])[:100]
```

代价：合并逻辑业务自实现。

#### 异步预热 + 削峰

发现某 key 即将变热（业务上预测）→ 提前预热到多个节点的本地缓存 + Redis 副本。秒杀场景常用。

---

## 第三章：缓存三大问题

### 3.1 缓存击穿（hot key expire）

**现象**：某个热点 key 突然过期，瞬间几万请求并发查 DB，DB 直接被打挂。

**解法**：

1. **物理永不过期 + 异步刷新**：业务侧定期主动刷新缓存内容；过期靠业务层判断"逻辑过期"
2. **互斥锁**：缓存 miss 后，第一个查 DB 的 client 拿到分布式锁，其他等结果

```python
def get(key):
    val = r.get(key)
    if val:
        return val
    # 缓存 miss，加锁
    lock = r.set(f"lock:{key}", "1", nx=True, ex=10)
    if lock:
        try:
            val = db.query(key)
            r.set(key, val, ex=3600)
            return val
        finally:
            r.delete(f"lock:{key}")
    else:
        time.sleep(0.05)
        return get(key)   # 递归重试
```

3. **TTL 加随机化**（R02 已讲）：避免同一时刻多个 key 集中过期

### 3.2 缓存穿透（query non-existent key）

**现象**：恶意或 bug，查询大量不存在的 key（如 user_id = -1）。缓存永不命中 → 每次打 DB。

**解法**：

1. **Bloom Filter 前置**（R09）：
```python
if not r.execute_command("BF.EXISTS", "users_bf", user_id):
    return None    # 一定不存在
```

2. **缓存空对象**：DB 查不到也写入缓存（值为特殊标记），TTL 短

```python
val = r.get(key)
if val == "__NULL__":
    return None
if val:
    return parse(val)
val = db.query(key)
if val is None:
    r.set(key, "__NULL__", ex=60)   # 缓存 1 分钟
    return None
r.set(key, serialize(val), ex=3600)
return val
```

3. **参数校验**：业务侧拒绝明显非法的 key（如 -1、字符串过长）

### 3.3 缓存雪崩（mass expire / Redis down）

**现象**：

- 大量 key 在同一时刻过期（如所有商品在每天 0 点重新加载）→ 全打 DB
- Redis 整体故障 → 全打 DB

**解法**：

1. **TTL 随机化**：`r.set(k, v, ex=3600 + random.randint(-300, 300))`
2. **多级缓存**：本地缓存 + Redis + DB；Redis 挂掉时本地缓存兜底
3. **熔断 + 降级**：DB QPS 超阈值时自动降级（返回静态 / 拒绝）
4. **Redis 高可用**：Sentinel / Cluster，避免单点故障

---

## 第四章：分布式锁

### 4.1 简单实现

```python
# 加锁
lock = r.set("lock:resource", str(uuid.uuid4()), nx=True, ex=10)
if not lock:
    raise LockFailed()

# 业务
do_work()

# 解锁：必须校验 value 是自己的
unlock_script = """
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
"""
r.eval(unlock_script, 1, "lock:resource", lock_value)
```

要点：

- `NX + EX` 一条命令原子加锁 + 过期
- value 用 UUID 标识所有者，防止"我超时了别人接手，我 timer 没停又 DEL 了别人的锁"
- 解锁用 Lua 校验 value + 删除原子

### 4.2 锁超时与续期

业务执行超时 → 锁自动过期 → 别人拿到 → **同时两个 client 持锁**。

解法：**心跳续期**——业务执行期间另起线程定期 `EXPIRE` 延长。

```python
import threading, time

class RedisLock:
    def __init__(self, r, key, ttl=10):
        self.r = r
        self.key = key
        self.value = str(uuid.uuid4())
        self.ttl = ttl
        self.stop = False

    def acquire(self):
        return self.r.set(self.key, self.value, nx=True, ex=self.ttl)

    def heartbeat(self):
        while not self.stop:
            time.sleep(self.ttl / 3)
            self.r.expire(self.key, self.ttl)

    def __enter__(self):
        self.acquire()
        t = threading.Thread(target=self.heartbeat, daemon=True)
        t.start()
        return self

    def __exit__(self, *a):
        self.stop = True
        # ... unlock 同上
```

### 4.3 Redlock

Redis 作者提的"在多个独立 Redis 实例间获取多数派锁"算法。**安全性有学术争议**（Martin Kleppmann 等批评）。**生产建议**：

- 单实例 Redis 锁 + 业务可接受偶发问题 → 已经够用
- 强一致 + 不容忍任何一次失败 → 别用 Redis 做锁，用 etcd / ZK / Consul

---

## 第五章：安全与 ACL

### 5.1 默认监听 0.0.0.0 是高危

Redis 6 之前默认 `bind 127.0.0.1`（安全）。但**很多镜像 / 教程把 bind 改成 0.0.0.0 + 不设密码** → 公网扫描器秒抓 → 挖矿 / 数据库泄漏。

**生产铁律**：

```
bind 10.0.0.1 -::1                # 只绑特定内网 IP
protected-mode yes                # 默认 yes，但很多人误关
requirepass <random-32-bytes>     # 必设
masterauth <password>             # replica 同步用
```

### 5.2 ACL（Redis 6+）

老版本只有"全局 password"——所有 client 同样权限。Redis 6 引入 ACL，按用户分权：

```
ACL SETUSER readonly_user on >mypassword ~* +@read -@write
ACL SETUSER admin on >adminpass ~* +@all
ACL LIST
ACL WHOAMI
ACL DELUSER readonly_user
```

语法：

- `on` / `off`：启用 / 禁用
- `>pass`：密码（多个就是多个候选）
- `~pattern`：可访问的 key pattern
- `+@category` / `-@category`：命令类别（read / write / admin / connection 等）
- `+command` / `-command`：单个命令

### 5.3 命令类别

```
ACL CAT                 # 列所有类别
ACL CAT read            # read 类别下的命令
ACL CAT dangerous       # 危险命令：CONFIG / FLUSHALL / DEBUG / KEYS / MONITOR...
```

生产建议：

- 业务用户：`+@read +@write -@dangerous -@admin`
- 运维用户：`+@all` + IP 白名单（如果 Redis 6.2+ 支持 `&channel-pattern`）
- 监控用户：`+@read +ping +info`

### 5.4 默认 user 怎么处理

默认有个 `default` 用户。**生产把它禁掉**：

```
ACL SETUSER default off
ACL SETUSER app on >password ~app:* +@read +@write
```

或者保留 default 但去掉所有权限。

### 5.5 TLS（Redis 6+）

```
port 0                       # 关明文
tls-port 6379
tls-cert-file /path/cert.crt
tls-key-file /path/cert.key
tls-ca-cert-file /path/ca.crt
tls-auth-clients yes         # 客户端也要证书（mTLS）
```

零额外性能损失（< 5%）。**生产 Redis 强烈建议开 TLS**——尤其跨网络。

---

## 第六章：连接池配置

### 6.1 不要每次请求新连接

每次连接都要 TCP 握手 + AUTH，~1ms 起步。

```python
# 错误
def get_user(id):
    r = redis.Redis(host="...", port=6379)
    return r.get(f"user:{id}")    # 每次都新连接

# 正确
pool = redis.ConnectionPool(host="...", port=6379, max_connections=50)
r = redis.Redis(connection_pool=pool)
def get_user(id):
    return r.get(f"user:{id}")
```

### 6.2 连接池大小

经验值：

- 每应用实例 50-100 连接（中等 QPS）
- 高 QPS：100-500
- 微服务规模：所有服务实例 × 单实例连接数 < Redis maxclients（默认 10000）

**反模式**：每个微服务实例配 1000 连接 + 200 个实例 = 20 万 → Redis 直接拒绝新连接。

### 6.3 超时

```python
redis.Redis(
    socket_connect_timeout=2,    # 连接建立超时 2 秒
    socket_timeout=1,            # 单命令超时 1 秒
    health_check_interval=30,    # 30 秒一次 PING 检查
    retry_on_timeout=True,
)
```

**永远要设超时**——默认无限等是埋雷。

### 6.4 long-lived 连接陷阱

- Linux NAT 网关 30 分钟 idle 断 TCP → 客户端拿到无效连接
- 防火墙规则更新断老连接
- LB 后端切换

**解法**：

- `health_check_interval` 让 SDK 主动 PING
- `tcp-keepalive 60`（Redis server 侧）+ OS 层 SO_KEEPALIVE

---

## 第七章：监控告警 checklist

| 指标 | 阈值 | 含义 |
|---|---|---|
| `used_memory_rss / maxmemory` | > 0.9 | 内存即将打满 |
| `mem_fragmentation_ratio` | > 1.5 | 碎片严重 |
| `rejected_connections` | > 0 | maxclients 满 |
| `connected_clients` | 监控趋势 | 连接泄漏 |
| `instantaneous_ops_per_sec` | 跟容量规划比 | QPS 异常 |
| `latest_fork_usec` | > 500ms | fork 慢，可能 OOM 风险 |
| `replica_lag` | > 30 秒 | 复制跟不上 |
| `master_link_status` | != up | 复制断 |
| `aof_pending_fsync` | > 0 持续 | 磁盘慢 |
| `cluster_state` | != ok | 集群异常 |
| `cluster_slots_ok` | < 16384 | slot 缺失 |
| `evicted_keys` / `expired_keys` 速率 | 突增 | 内存压力 / 集中过期 |
| `slowlog_len` 或新增 | > 0 | 慢命令 |

工具栈：`redis_exporter` + Prometheus + Grafana + AlertManager。

---

## 第八章：典型生产事故复盘

### 8.1 案例 1：bigkey 删除导致服务卡顿

**现象**：周三凌晨 3 点业务报警，Redis 主线程 CPU 飙到 100%，所有 client P99 飙到 10 秒。

**调查**：

- `SLOWLOG GET 10` → 显示 `DEL big_log_2023_03`，耗时 8 秒
- `INFO keyspace` → 该 key 已不存在（已被 DEL 完）

**根因**：某个清理脚本删了一个 5GB 的 list（包含历史日志）。

**修复**：

- 立刻：等 Redis 恢复（已自动）
- 长期：所有清理脚本改 UNLINK；建立定期 `--bigkeys` 扫描 + 告警

### 8.2 案例 2：缓存击穿引发 DB 雪崩

**现象**：电商首页加载缓慢，DB 连接池打满。

**调查**：

- DB 慢查询日志显示同一条 `SELECT * FROM banner WHERE id=1` 几千次/秒
- Redis 上 `banner:1` 不存在（被某次 deploy 误清）
- 业务代码 cache miss 直接打 DB，无锁无防护

**修复**：

- 立刻：手动 SET 该 banner
- 长期：所有热点 key 加分布式锁 + 缓存空值 + TTL 随机化

### 8.3 案例 3：Cluster 加节点后业务持续报 MOVED

**现象**：扩容加了 3 个新 master 后，业务侧 5% 请求收到 MOVED 报错（client log）。

**调查**：客户端用的 SDK 不支持 cluster MOVED 重定向，自己实现的简易 redis-cli wrapper。

**修复**：

- 升级到 go-redis / Jedis cluster mode；MOVED 自动处理

### 8.4 案例 4：fork 引起 master 抖动

**现象**：Redis dataset 80 GB，每天凌晨自动 BGSAVE 时业务 P99 涨到 1 秒+。

**调查**：

- `LATENCY DOCTOR` → "Big 'fork' event: 850 ms"
- THP 状态：`echo` `[always]` —— 开启状态

**修复**：

- `echo never > /sys/kernel/mm/transparent_hugepage/enabled`
- 进一步：把 BGSAVE 转到 replica 做，master `save ""` + `appendonly no`

---

## 第九章：生产级最佳实践

1. **每天扫 bigkey + hotkey**——纳入自动化巡检
2. **缓存击穿用分布式锁 + 异步刷新**
3. **缓存穿透用 Bloom Filter 或缓存空值**
4. **缓存雪崩 TTL 随机化 + 多级缓存**
5. **ACL 强制开启 + default user 禁用**
6. **TLS 强制开启**（哪怕内网）
7. **连接池大小预设上限**——按 maxclients 倒推
8. **所有命令带超时**——`socket_timeout` 必设
9. **监控 13 项黄金指标**——见 §7
10. **每月做一次故障演练**——Sentinel failover、Cluster 节点宕机、Redis OOM 重启

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：DEL 大 key
卡主线程。改 UNLINK，或渐进式 HDEL/SREM。

### ❌ 陷阱 2：每次请求新 Redis 连接
握手 + AUTH 各 1ms 起步。用连接池。

### ❌ 陷阱 3：没设超时
默认无限等。Redis 慢 → 业务 hang。

### ❌ 陷阱 4：在缓存 miss 后没锁直接打 DB
雪崩前奏。加分布式锁或单飞模式。

### ❌ 陷阱 5：default user 还有所有权限
密码泄漏 = 数据库泄漏。`ACL SETUSER default off`。

### ❌ 陷阱 6：bind 0.0.0.0
公网扫描秒抓。`bind <内网 IP>`。

### ❌ 陷阱 7：把 Redis 当持久数据库
RDB / AOF 有窗口期；强一致用 SQL。

### ❌ 陷阱 8：所有 Redis 实例共享一个 password
一处泄漏全军覆没。ACL 分用户分权 + 不同环境不同 password。

### ❌ 陷阱 9：监控只盯内存 / QPS
等延迟报警时已晚。必看 slowlog / latency / replica lag。

### ❌ 陷阱 10：故障演练只在文档里
没真做过的预案出事时全是 bug。每季度演练。

---

## 第十一章：练习题

**练习 1**：业务一个 Hash key 已经 100 万字段，业务侧需要每个字段独立 TTL。给出 Redis 7.4+ 的方案。

**练习 2**：电商详情页接口 QPS 5k，缓存命中 99%。设计一个三层缓存（本地、Redis、DB）+ 击穿穿透雪崩防护。

**练习 3**：生产 Redis bind 0.0.0.0 + 无密码 + 公网 IP，外部团队需要访问。给出最小化暴露的安全改造步骤。

**练习 4**：业务订单写入 Redis 5 分钟过期，凌晨发现 Redis 内存爆。可能原因？

**练习 5**：分布式锁场景：业务 A 拿锁后处理 30 秒，期间锁 TTL 10 秒。如何避免锁过期被别人抢到？

---

## 参考答案

**练习 1**：Redis 7.4+ 的字段级 TTL：

```
HEXPIRE huser:1234 60 FIELDS 1 session_token
HTTL huser:1234 FIELDS 1 session_token
HPERSIST huser:1234 FIELDS 1 session_token
```

底层每字段 8 字节 TTL → 100 万字段 ~8MB 额外开销。一旦 HEXPIRE 该 hash **不再用 listpack**——转 hashtable。要保留紧凑就别给所有字段都加 TTL，只给业务真需要的字段加。

**练习 2**：

```python
import threading
from cachetools import TTLCache

local = TTLCache(maxsize=10000, ttl=10)

def get_product(pid):
    # 1. 本地
    if pid in local:
        return local[pid]

    # 2. Redis（带分布式锁防击穿）
    val = r.get(f"product:{pid}")
    if val:
        local[pid] = json.loads(val)
        return local[pid]

    # 3. Cache miss → 加锁查 DB
    lock_key = f"lock:product:{pid}"
    if r.set(lock_key, "1", nx=True, ex=10):
        try:
            prod = db.query("SELECT ... WHERE id=?", pid)
            if prod is None:
                # 防穿透：缓存空值
                r.set(f"product:{pid}", "__NULL__", ex=60)
                return None
            ttl = 3600 + random.randint(-300, 300)   # 防雪崩
            r.set(f"product:{pid}", json.dumps(prod), ex=ttl)
            local[pid] = prod
            return prod
        finally:
            r.delete(lock_key)
    else:
        # 别人在查，等一下重试
        time.sleep(0.05)
        return get_product(pid)
```

**练习 3**：

1. 立刻：
   - `CONFIG SET requirepass <random-strong-password>` + 通知所有 client 更新
   - 防火墙 / 安全组：只允许已知 IP 段访问 6379
2. 短期：
   - 改配置文件 `bind <内网 IP>` + `protected-mode yes`
   - 重启或 reload
3. 中期：
   - 开 TLS：`tls-port 6379` + 客户端 mTLS
   - 启用 ACL，给每个外部团队单独 user，最小权限（`~team_a:* +@read`）
   - 禁用 default user
4. 长期：
   - Redis 不暴露公网，通过 VPN / 私有网络访问

**练习 4**：

- TTL 都是 5 分钟但创建瞬间高度集中（业务下单峰值 100k/s）→ 5 分钟内累计 3000 万 key
- active expire 跟不上 → key 累积
- 内存可能在 5 分钟后才开始下降（要等 active 扫到）
- 解决：TTL 随机化（5 ± 1 分钟）；调大 `hz`；监控 `expired_keys` 速率

**练习 5**：

```python
class RedisLock:
    def __init__(self, r, key, ttl=10):
        self.r = r
        self.key = key
        self.val = str(uuid.uuid4())
        self.ttl = ttl
        self.running = False

    def acquire(self):
        ok = self.r.set(self.key, self.val, nx=True, ex=self.ttl)
        if ok:
            self.running = True
            threading.Thread(target=self._renew, daemon=True).start()
        return ok

    def _renew(self):
        # 心跳：每 TTL/3 续期一次
        while self.running:
            time.sleep(self.ttl / 3)
            self.r.eval(
                "if redis.call('GET', KEYS[1])==ARGV[1] then return redis.call('EXPIRE',KEYS[1],ARGV[2]) end",
                1, self.key, self.val, self.ttl
            )

    def release(self):
        self.running = False
        self.r.eval(
            "if redis.call('GET', KEYS[1])==ARGV[1] then return redis.call('DEL',KEYS[1]) end",
            1, self.key, self.val
        )

# 用法
lock = RedisLock(r, "lock:order_123", ttl=10)
if lock.acquire():
    try:
        do_30_second_work()    # 期间锁会被自动续期
    finally:
        lock.release()
```

进阶：用 Redisson（Java）/ go-redsync / redis-py-lock 等现成库，已实现 watchdog。

---

## 小结

| 问题 | 主要解法 |
|---|---|
| bigkey | 拆 + UNLINK + 渐进式删 |
| hotkey | 本地缓存 + 副本读 + 拆 key |
| 缓存击穿 | 分布式锁 + 异步刷新 |
| 缓存穿透 | Bloom Filter + 缓存空值 |
| 缓存雪崩 | TTL 随机化 + 多级缓存 + 熔断 |
| 分布式锁 | NX + 心跳续期 |
| 安全 | ACL + TLS + bind 内网 |
| 连接 | 连接池 + 超时 + 健康检查 |

四条铁律：

1. **bigkey 是延迟杀手 + 迁移杀手**——一开始就避免
2. **缓存三大问题的本质**：cache miss 时 DB 的承压能力
3. **安全默认关键三件**：bind / 密码 / ACL
4. **没监控就没真生产**——13 项指标看齐

下一篇 **R11 — Redis vs Valkey**：fork 背景、许可证细节、技术演进、选型决策。
