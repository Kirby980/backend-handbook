# 精通 Redis Lua 脚本

> 课程编号：L21
> 路线图来源：Lua 全场景深度课程 — Redis 脚本宿主
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟
> 📅 内容基准：**Redis 7.x**（Functions 自 7.0）+ 内嵌 **Lua 5.1**

---

## 引言：在数据库里跑 Lua

```lua
-- Redis 内嵌了一个 Lua 解释器。这段脚本在 Redis 服务端原子执行：
-- EVAL "return redis.call('INCR', KEYS[1])" 1 counter

-- 几个必须先想清楚的问题：
-- 1) 为什么参数要分成 KEYS 和 ARGV 两组？
-- 2) "原子执行"意味着脚本里能 sleep 或调用阻塞命令吗？
-- 3) 脚本里用 math.random 或 os.time 会有什么隐患？
```

Redis 把 Lua **嵌入服务端**（用 C API，L15），让你把"多个命令 + 逻辑"打包成一个**原子操作**在服务端执行——避免多次往返、保证原子性。这是实现分布式锁、原子限流、CAS 的利器。本章讲清 Redis 脚本的执行模型、`KEYS`/`ARGV`、原子性约束、以及 7.0 的 Functions。与 `../redis/06-精通-Redis-事务-脚本.md` 互补（那里偏 Redis 视角，这里偏 Lua 视角）。

> 注意：Redis 内嵌的是 **Lua 5.1**（接近 LuaJIT 的语言版本），所以没有原生整型、`<close>` 等 5.3+ 特性。

---

## 第一章：EVAL 基础

### 1.1 `EVAL` 命令

```
EVAL script numkeys key1 key2 ... arg1 arg2 ...
```

- `script`：Lua 源码字符串。
- `numkeys`：后面有几个是"键名"。
- 接下来 `numkeys` 个是 **KEYS**，其余是 **ARGV**。

```bash
# 给 KEYS[1] 这个 key 自增，增量来自 ARGV[1]
EVAL "return redis.call('INCRBY', KEYS[1], ARGV[1])" 1 mycounter 5
# → (integer) 5
```

在脚本里：
- `KEYS[1]`, `KEYS[2]`... 是键名（1-based，Lua 数组，L05）。
- `ARGV[1]`, `ARGV[2]`... 是其它参数。

### 1.2 为什么分 KEYS 和 ARGV

⚠️ 这是 Redis 脚本最重要的规则：**所有要访问的 key 必须通过 KEYS 传入，不能在脚本里硬编码或拼接**。原因：

- **Redis Cluster** 需要知道脚本碰哪些 key，才能判断它们是否在同一个 slot（跨 slot 的脚本会被拒绝）。
- 引擎据此做正确性检查和路由。

```lua
-- ❌ 错误：硬编码 key 名，Cluster 无法路由
return redis.call('GET', 'user:1')

-- ✅ 正确：key 通过 KEYS 传入
return redis.call('GET', KEYS[1])
```

### 1.3 `EVALSHA` —— 避免重复传脚本

每次 `EVAL` 都传整段脚本浪费带宽。`SCRIPT LOAD` 预加载脚本得到 SHA1，之后用 `EVALSHA` 只传哈希：

```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → "a1b2c3..."（SHA1）
EVALSHA a1b2c3... 1 mykey
# 若脚本未缓存（NOSCRIPT 错误），客户端回退到 EVAL 重新加载
```

客户端库（如 `lua-resty-redis`，L19）通常封装"先 EVALSHA，NOSCRIPT 则 EVAL"的逻辑。

---

## 第二章：原子性——最大的卖点也是最大的约束

### 2.1 脚本是原子的

Redis 是单线程执行命令的。脚本执行期间，**Redis 不处理任何其它命令**——整个脚本是一个不可分割的原子单元。这让"check-then-act"类操作天然安全（无需事务/锁）：

```lua
-- 原子地：如果库存足够就扣减（不会有并发超卖）
local stock = tonumber(redis.call('GET', KEYS[1]))
if stock and stock >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1   -- 成功
end
return 0       -- 库存不足
```

这段在普通客户端要"GET → 判断 → DECRBY"三次往返且有竞态；脚本里一次原子完成。

### 2.2 约束：不能阻塞、要快

正因为脚本独占 Redis，**脚本必须快速完成**，且：

- **不能调用阻塞命令**（`BLPOP`、`WAIT` 等会被禁止或报错）。
- **不能 `sleep`**（会冻结整个 Redis）。
- **不能做重 CPU 计算**（拖慢所有客户端）。
- 默认有执行时间限制（`busy-reply-threshold`/`lua-time-limit`，超时其它客户端收到 BUSY，可 `SCRIPT KILL`）。

**铁律**：脚本要短小、确定、快速。复杂逻辑别全塞脚本里。

---

## 第三章：`redis.call` 与类型转换

### 3.1 `redis.call` vs `redis.pcall`

```lua
redis.call('SET', KEYS[1], ARGV[1])    -- 出错直接中止脚本、向客户端报错
local ok, err = pcall(function()       -- 或用 redis.pcall 捕获
    return redis.call('INCR', KEYS[1]) -- 对非整数 key INCR 会报错
end)

-- redis.pcall：捕获错误为表，不中止脚本
local res = redis.pcall('INCR', KEYS[1])
if res.err then
    -- 处理错误，脚本继续
end
```

`redis.call` 出错即中止；`redis.pcall` 捕获错误返回带 `.err` 的表（类似 Lua pcall，L09），让脚本能处理后继续。

### 3.2 Lua ↔ Redis(RESP) 类型转换

脚本返回值和 `redis.call` 结果在 Lua 与 Redis 协议间转换，规则需牢记：

| Lua | → Redis 回复 |
|---|---|
| number | **整数**（小数部分被截断！） |
| string | bulk string |
| table（数组部分） | 数组（遇到第一个 nil 截断！） |
| `table { ok = "OK" }` | 状态回复 |
| `table { err = "msg" }` | 错误回复 |
| `false` / nil | nil（空回复） |
| `true` | 整数 1 |

```lua
return 3.99        -- 客户端收到整数 3（小数丢失！）
return "hello"     -- bulk string
return { 1, 2, 3 } -- 数组
return { 1, nil, 3 }  -- 只返回 {1}（nil 截断数组！L05）
return redis.status_reply("OK")    -- 状态回复
return redis.error_reply("失败")    -- 错误回复
```

⚠️ 两个高频坑：**number 返回会截断小数**（要精确返回字符串）；**数组里有 nil 会截断**（L05 的 nil 洞问题）。

---

## 第四章：实战脚本

### 4.1 原子限流（令牌桶 / 固定窗口）

L20 提到全局限流要靠 Redis。一个固定窗口计数限流脚本：

```lua
-- KEYS[1]=限流key, ARGV[1]=limit, ARGV[2]=window秒
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[2])   -- 第一次设过期
end
if current > tonumber(ARGV[1]) then
    return 0   -- 超限
end
return 1       -- 放行
```

原子地"自增 + 首次设过期 + 判断"，多网关共享这个 Redis key 即实现**全局限流**（L20 的分布式版本）。

### 4.2 分布式锁——安全释放

获取锁用 `SET key val NX PX ttl`（一条命令原子）。但**释放锁**必须校验持有者再删（否则可能删掉别人的锁），这需要脚本原子化：

```lua
-- 安全释放锁：KEYS[1]=锁key, ARGV[1]=我的唯一标识
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])   -- 是我的锁才删
else
    return 0                             -- 不是我的，不删
end
```

为什么要脚本：`GET` 判断和 `DEL` 之间若非原子，锁可能在间隙过期并被他人获取，导致误删。脚本保证"检查 + 删除"原子。这是 Redlock/分布式锁的标准释放方式（详见 `../redis` 与 `../microservices` 分布式锁篇）。

### 4.3 CAS（compare-and-swap）

```lua
-- 仅当当前值等于期望值时才更新：KEYS[1]=key, ARGV[1]=期望旧值, ARGV[2]=新值
if redis.call('GET', KEYS[1]) == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
end
return 0
```

---

## 第五章：Redis 7.0 Functions

### 5.1 EVAL 的问题

`EVAL`/`EVALSHA` 的脚本是"临时"的——不持久、要客户端管理 SHA、难复用。**Redis 7.0 引入 Functions**：把脚本作为**持久化的、命名的函数库**注册到 Redis。

### 5.2 注册与调用

```lua
#!lua name=mylib

-- 注册一个函数
redis.register_function('my_incr', function(keys, args)
    return redis.call('INCR', keys[1])
end)

redis.register_function('rate_limit', function(keys, args)
    local current = redis.call('INCR', keys[1])
    if current == 1 then redis.call('EXPIRE', keys[1], args[2]) end
    return current <= tonumber(args[1]) and 1 or 0
end)
```

```bash
FUNCTION LOAD "..."              # 加载函数库（持久化，随 RDB/AOF）
FCALL rate_limit 1 mykey 100 60  # 调用，类似 EVAL 但用函数名
FUNCTION LIST                    # 列出已注册函数
```

### 5.3 Functions vs EVAL

| | EVAL/EVALSHA | Functions（7.0+） |
|---|---|---|
| 持久 | 否（重启/SCRIPT FLUSH 丢失） | **是**（随持久化、复制传播） |
| 标识 | SHA1（客户端管理） | **命名函数** |
| 组织 | 单脚本 | 库（多函数 + 共享逻辑） |
| 复用 | 差 | 好 |

新项目优先用 Functions；存量大量用 EVAL。`keys`/`args` 参数对应 `KEYS`/`ARGV`。

---

## 第六章：确定性与复制——隐藏的坑

### 6.1 脚本必须确定性

Redis 通过复制把写操作传给从库/AOF。**脚本必须确定性**（同样输入产生同样副作用），否则主从不一致：

```lua
-- ❌ 危险：随机/时间使脚本非确定，主从可能不一致
local r = math.random()                  -- 主从随机数不同！
redis.call('SET', KEYS[1], r)

local t = redis.call('TIME')             -- 各节点时间可能不同
```

现代 Redis（5.0+）采用**effects replication**（复制脚本产生的实际写命令，而非脚本本身），缓解了部分确定性问题。但仍应避免：

- 在写入中依赖 `math.random`（要随机用 `redis.call` 拿 Redis 的随机，或从 ARGV 传入）。
- 依赖 `TIME`/系统状态做写决策（用 ARGV 传入时间戳）。

### 6.2 不要污染全局

Redis 脚本环境**禁止创建全局变量**（保护沙箱，L23 思想）：

```lua
x = 10        -- ❌ 报错：Script attempted to create global variable 'x'
local x = 10  -- ✅ 必须 local
```

这强制了 L02 的"务必 local"铁律——Redis 直接用沙箱拦截。

---

## 第七章：常见陷阱清单

### ❌ 陷阱 1：硬编码 key 不走 KEYS

```lua
redis.call('GET', 'user:1')   -- ❌ Cluster 无法路由
```
用 `KEYS[1]`。

### ❌ 陷阱 2：脚本里阻塞/慢操作

```lua
redis.call('BLPOP', KEYS[1], 0)   -- ❌ 阻塞命令，冻结 Redis
```
脚本要快、非阻塞。

### ❌ 陷阱 3：number 返回截断小数

```lua
return 3.99    -- 客户端收到整数 3！
```
要精确用字符串：`return tostring(3.99)`。

### ❌ 陷阱 4：返回数组含 nil 被截断

```lua
return { "a", nil, "c" }   -- 只返回 {"a"}
```
避免 nil 洞（L05）。

### ❌ 陷阱 5：脚本依赖随机/时间

```lua
redis.call('SET', KEYS[1], math.random())   -- 主从不一致风险
```
随机/时间从 ARGV 传入。

### ❌ 陷阱 6：忘了 local 创建全局

```lua
counter = 0   -- ❌ Redis 拦截：禁止全局变量
```
必须 `local`。

---

## 第八章：练习题

**练习 1**：为什么 Redis 脚本要求 key 通过 KEYS 传入？

**练习 2**：写一个原子的"库存扣减"脚本（不超卖）。

**练习 3**：为什么释放分布式锁要用脚本而不是直接 DEL？

**练习 4**：脚本 `return 2.7` 客户端收到什么？怎么精确返回？

**练习 5**：EVAL 和 Functions（7.0）的核心区别？

---

## 参考答案与解析

**练习 1**：让 Redis（尤其 Cluster）知道脚本访问哪些 key——据此判断 key 是否在同一 slot、做路由和正确性检查。硬编码 key 会让 Cluster 无法判断，脚本被拒。

**练习 2**：
```lua
local stock = tonumber(redis.call('GET', KEYS[1])) or 0
local need = tonumber(ARGV[1])
if stock >= need then
    redis.call('DECRBY', KEYS[1], need)
    return 1
end
return 0
```

**练习 3**：直接 `DEL` 可能删掉别人的锁——如果你的锁已过期，别人获取了同名锁，你的 DEL 会误删他的。脚本原子地"GET 校验持有者 == 自己 → 才 DEL"，避免误删。

**练习 4**：收到**整数 2**（小数被截断）。精确返回用字符串：`return tostring(2.7)`，或返回 `"2.7"`。

**练习 5**：EVAL 脚本是临时的（不持久、靠 SHA1、难复用）；Functions（7.0）把脚本作为**持久化的命名函数库**注册，随复制/持久化传播，可组织多函数、易复用。新项目用 Functions。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| EVAL | `EVAL script numkeys KEYS... ARGV...`；EVALSHA 省带宽 |
| KEYS/ARGV | **所有 key 必经 KEYS**（Cluster 路由）；ARGV 是其它参数 |
| 原子性 | 脚本独占 Redis，天然原子；**禁阻塞/sleep/重计算，要快** |
| call/pcall | call 出错中止；pcall 捕获 `.err` 继续 |
| 类型转换 | number **截断小数**；数组遇 nil **截断**；status/error_reply |
| 实战 | 原子限流、安全释放锁（GET 校验+DEL）、CAS |
| Functions | 7.0+ 持久化命名函数库；优于 EVAL |
| 确定性 | 避免随机/时间做写决策；**禁全局变量**（沙箱） |

---

## 📅 2026 现状/更新

- **Redis Functions（7.0+）** 是 2026 推荐的脚本组织方式，逐步取代裸 EVAL；effects replication 缓解了确定性约束。
- Redis 嵌入 **Lua 5.1**（无 5.3+ 整型/`<close>`），处理大整数/精确数值要用字符串。
- Redis 脚本 + OpenResty（L20）是分布式限流/锁的黄金组合：网关本地 shared dict 快路径 + Redis 脚本全局原子。详见 `../redis/06-精通-Redis-事务-脚本.md` 与 `../microservices` 分布式锁篇。

---

> 🔁 下一篇 **L22 — 精通 Neovim Lua 配置与插件**：又一个 Lua 宿主——编辑器。`init.lua`、`vim.api`/`vim.opt`、用 lazy.nvim 管理插件、内置 LSP，以及用 Lua 写 Neovim 插件。
>
> 反馈：把"原子限流"和"安全释放锁"两个脚本背下来，它们是分布式系统的高频面试题与实战工具。
