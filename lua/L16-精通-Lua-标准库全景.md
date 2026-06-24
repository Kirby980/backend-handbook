# 精通 Lua 标准库全景

> 课程编号：L16
> 路线图来源：Lua 全场景深度课程 — 标准库
> 难度：⭐⭐⭐
> 预计阅读时间：50 分钟
> 📅 内容基准：**Lua 5.4.8**（`utf8` 自 5.3、`string.pack` 自 5.3）+ **LuaJIT 2.1**

---

## 引言：小而精的标准库

```lua
-- 1) os.time 与 os.clock 的区别？
print(os.time())      -- ?
print(os.clock())     -- ?

-- 2) 输出？
print(math.huge, -math.huge, math.huge == 1/0)

-- 3) 这个 UTF-8 字符串有几个"字符"？几个字节？
local s = "héllo"
print(#s, utf8.len(s))

-- 4) 把整数打包成 4 字节大端再解出来？
local packed = string.pack(">I4", 1000)
print(#packed, string.unpack(">I4", packed))
```

答案：① `os.time()` 返回 Unix 时间戳（秒，整数），`os.clock()` 返回**程序 CPU 时间**（秒，浮点，用于测耗时）；② `inf  -inf  true`（`math.huge` 就是 `1/0`）；③ `#s` 是 6（é 占 2 字节），`utf8.len(s)` 是 5（字符数）；④ `4  1000  5`（打包成 4 字节，解出 1000，第二返回值 5 是下一个读取位置）。

Lua 标准库刻意保持精简——没有正则、没有内置 JSON、没有网络（这些靠库）。但它提供的 `string`/`table`/`math`/`os`/`io`/`utf8` 等都很实用。这一章系统梳理（前面分散讲过的回顾要点，未讲的补全），并重点讲 `string.pack`（二进制）和 `utf8`。

---

## 第一章：基础全局函数

这些不属于任何库，直接可用：

```lua
print(...)              -- 输出到 stdout，制表符分隔，换行
type(v)                 -- 类型字符串（L02）
tostring(v) / tonumber(s[, base])   -- 转换（L02/L03）
pairs(t) / ipairs(t)    -- 遍历（L05）
next(t[, k])            -- 底层迭代：返回下一对键值；判空 next(t)==nil
select(n, ...)          -- 取变长参数（L07）；select("#",...) 算个数
assert(v[, msg])        -- 断言（L09）
error(msg[, level])     -- 抛错（L09）
pcall / xpcall          -- 保护调用（L09）
rawget/rawset/rawequal/rawlen   -- 绕过元方法（L06）
setmetatable/getmetatable       -- 元表（L06）
collectgarbage(opt)     -- GC 控制（L12）
load(chunk[, name, mode, env])  -- 编译代码（L01）
require(modname)        -- 加载模块（L10）
```

### 1.1 `load`——动态编译

```lua
local f = load("return 1 + 2")    -- 编译字符串为函数
print(f())                         -- 3

-- 带环境（沙箱，L23）
local env = { x = 10 }
local g = load("return x * 2", "chunk", "t", env)
print(g())                         -- 20

-- load 可接函数作为"分块读取器"
```

`load` 失败返回 `nil, errmsg`（编译期错误）。第三参 `mode`：`"t"` 只允许文本、`"b"` 二进制、`"bt"` 都行（防注入预编译字节码）。

---

## 第二章：`string` 库（回顾 + 补全）

L04 详讲了模式匹配。补充要点：

```lua
string.format(fmt, ...)    -- printf 风格（L04）
string.rep(s, n[, sep])    -- 重复
string.byte(s[, i[, j]])   -- 字节值（可范围）
string.char(...)           -- 字节 → 字符串
string.pack/unpack/packsize -- 二进制打包（见第六章）
```

### 2.1 `string.format` 进阶

```lua
string.format("%5.2f", 3.14159)   -- " 3.14"（宽 5，2 位小数）
string.format("%-10s|", "left")   -- "left      |"（左对齐）
string.format("%q", 'a\nb')       -- 可反读的转义字符串
string.format("%g", 0.0001)       -- 自动选择 %e/%f
string.format("%c", 65)           -- "A"（字符）
```

---

## 第三章：`table` 库（回顾）

L05 详讲。速记：

```lua
table.insert(t, [pos,] v)
table.remove(t[, pos])
table.concat(t[, sep[, i[, j]]])   -- 拼接（性能关键，L04）
table.sort(t[, cmp])               -- 原地、不稳定、严格弱序
table.unpack(t[, i[, j]])          -- 表→多值（5.1 是全局 unpack）
table.pack(...)                    -- 多值→表（含 .n）
table.move(a1, f, e, t[, a2])      -- 区间搬移（5.3+）
```

`table.move` 可高效实现数组复制、插入、循环缓冲：

```lua
local t = {1, 2, 3, 4, 5}
table.move(t, 2, 4, 1)             -- 把 [2,4] 移到从 1 开始 → {2,3,4,4,5}
```

---

## 第四章：`math` 库

```lua
math.pi                  -- 3.1415...
math.huge                -- 正无穷（= 1/0）
math.maxinteger/mininteger  -- 5.3+ 整型边界（L03）
math.abs/ceil/floor/sqrt/sin/cos/exp/log
math.fmod(a, b)          -- C 风格取模（向零截断，区别于 % 的 floor）
math.modf(x)             -- 分离整数和小数部分
math.max(...)/min(...)
math.tointeger(x)        -- 5.3+：能无损转整型则转，否则 nil
math.type(x)             -- 5.3+："integer"/"float"/nil（L03）
math.ult(a, b)           -- 无符号比较
```

### 4.1 随机数

```lua
math.randomseed(os.time())   -- 播种（5.4 不播种也有较好默认种子）
math.random()                -- [0, 1) 浮点
math.random(n)               -- [1, n] 整数
math.random(m, n)            -- [m, n] 整数
```

⚠️ 5.4 改进了随机数生成器（用 xoshiro256**），质量更好；旧版本依赖 C 的 `rand()` 质量差。**密码学场景别用 `math.random`**（不安全），用专门的 CSPRNG。

### 4.2 `fmod` vs `%`

```lua
print(-5 % 3)         --  1（Lua %：floor 语义，符号随除数）
print(math.fmod(-5, 3))  -- -2（C fmod：向零截断，符号随被除数）
```

两者对负数结果不同，按需选择。

---

## 第五章：`os` 与 `io` 库

### 5.1 `os` 库

```lua
os.time([table])         -- Unix 时间戳；可从 {year,month,day,...} 构造
os.date([format[, time]]) -- 格式化时间
os.clock()               -- 程序 CPU 时间（秒，浮点）——测耗时用
os.difftime(t2, t1)      -- 时间差
os.getenv("PATH")        -- 环境变量
os.execute(cmd)          -- 执行 shell 命令（返回成功标志 + 状态）
os.remove/os.rename      -- 文件操作
os.tmpname()             -- 临时文件名
os.exit([code])          -- 退出
```

```lua
-- 测耗时
local t0 = os.clock()
do_work()
print(("耗时 %.3f 秒"):format(os.clock() - t0))

-- 格式化日期
print(os.date("%Y-%m-%d %H:%M:%S"))   -- 2026-05-29 14:30:00
print(os.date("!%Y-%m-%dT%H:%M:%SZ")) -- ! 表示 UTC

-- 从字段构造时间戳
local t = os.time({ year = 2026, month = 5, day = 29, hour = 12 })
```

⚠️ `os.time` 返回**秒级**精度。要更高精度（毫秒/纳秒）标准库没有，要靠 `socket.gettime`（LuaSocket）、`ngx.now`（OpenResty）或 FFI 调 `gettimeofday`（L14）。

### 5.2 `io` 库

两套模型：**简单模型**（隐式当前文件）和**完整模型**（显式文件句柄）。

```lua
-- 简单模型
io.write("no newline")          -- 写 stdout（不加换行，不调 tostring）
local line = io.read()           -- 读 stdin 一行

-- 完整模型（推荐）
local f = assert(io.open("data.txt", "r"))   -- 打开（失败 assert 抛错）
local content = f:read("a")                   -- 读全部（5.3 用 "a"，旧版 "*a"）
f:close()

-- 逐行迭代
for line in io.lines("data.txt") do
    print(line)
end

-- 写文件
local out = assert(io.open("out.txt", "w"))
out:write("line1\n", "line2\n")
out:close()
```

`read` 的格式：`"a"`（全部）、`"l"`（一行不含换行）、`"L"`（含换行）、`"n"`（一个数）、数字 n（n 字节）。5.3 起去掉了 `*` 前缀（`"*a"` → `"a"`，但兼容旧写法）。

⚠️ 文件句柄要 `close`，或用 5.4 的 `<close>`（L25）确保。`io.lines` 迭代完会自动关闭。

---

## 第六章：`string.pack` —— 二进制处理（5.3+）

处理网络协议、二进制文件格式时，`string.pack`/`unpack` 把 Lua 值打包成字节串：

```lua
-- 打包：格式串 + 值
local data = string.pack("I4 I2 s1", 1000, 50, "hi")
-- I4: 4字节无符号整数, I2: 2字节, s1: 带1字节长度前缀的字符串

-- 解包：返回值们 + 下一个读取位置
local a, b, str, pos = string.unpack("I4 I2 s1", data)
print(a, b, str)        -- 1000  50  hi

-- 字节序
string.pack(">I4", 1)   -- 大端（网络字节序）
string.pack("<I4", 1)   -- 小端
string.pack("=I4", 1)   -- 本机字节序

print(string.packsize("I4 I2"))   -- 6（计算固定格式的字节数）
```

格式字符：`i/I`（有符号/无符号整数 + 字节数）、`h/H`（short）、`l/L`（long）、`f/d`（float/double）、`s`（带长度前缀字符串）、`z`（零结尾字符串）、`x`（填充字节）、`<`/`>`/`=`（字节序）。

这是不用 FFI 也能在 PUC-Lua 处理二进制协议的利器（Redis RESP、MQTT、自定义帧）。

---

## 第七章：`utf8` 库（5.3+）

Lua 字符串是字节串（L04），`utf8` 库提供 UTF-8 感知操作：

```lua
local s = "héllo世界"
print(#s)                  -- 字节数（é=2, 中文=3 each）
print(utf8.len(s))         -- 字符数 7
utf8.char(72, 233)         -- 码点 → UTF-8 字符串
utf8.codepoint(s, i)       -- 位置 i 的码点
utf8.offset(s, n)          -- 第 n 个字符的字节偏移（用于安全切片）

-- 遍历字符
for pos, code in utf8.codes(s) do
    print(pos, code)        -- 字节位置, 码点
end

-- 安全截取前 3 个字符
local function utf8_sub(s, i, j)
    local start = utf8.offset(s, i)
    local stop = utf8.offset(s, j + 1)
    return s:sub(start, stop and stop - 1 or -1)
end
```

⚠️ `utf8` 库只处理编码层面，**不做规范化、不懂字形/书写方向**（那要专门的库）。LuaJIT 默认无 `utf8` 库（5.3 特性）。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：`os.time` 当高精度计时

```lua
local t0 = os.time()   -- 秒级！测不出毫秒级耗时
```
测耗时用 `os.clock()`（CPU 时间）或 `ngx.now`/LuaSocket（墙钟）。

### ❌ 陷阱 2：`#s` 当 UTF-8 字符数

```lua
print(#"中文")   -- 6（字节），不是 2
```
用 `utf8.len`。

### ❌ 陷阱 3：忘记 close 文件

```lua
local f = io.open("x")   -- 不 close → 句柄泄漏
```
`f:close()` 或 5.4 `<close>`。

### ❌ 陷阱 4：`math.random` 用于安全

```lua
local token = math.random(1e9)   -- ❌ 可预测，不安全
```
密码学用 CSPRNG（如 `/dev/urandom`、`resty.random`）。

### ❌ 陷阱 5：`math.fmod` 与 `%` 混淆

```lua
math.fmod(-5, 3)   -- -2（向零）
-5 % 3             --  1（floor）
```

### ❌ 陷阱 6：LuaJIT 缺 5.3 库

```lua
utf8.len(s)              -- LuaJIT 默认无 utf8 库
string.pack(...)         -- LuaJIT 部分支持/需注意
```
确认运行时能力。

---

## 第九章：练习题

**练习 1**：测量一段代码的 CPU 耗时。

**练习 2**：把 `{year=2026, month=1, day=1}` 转成时间戳并格式化成 `YYYY-MM-DD`。

**练习 3**：安全获取 UTF-8 字符串的前 N 个字符。

**练习 4**：用 `string.pack` 把 3 个整数（大端 4 字节）打包，再解出来。

**练习 5**：判断真假——"`io.lines` 迭代结束后需要手动 close 文件。"

---

## 参考答案与解析

**练习 1**：
```lua
local t0 = os.clock()
-- work
print(("%.4f s"):format(os.clock() - t0))
```

**练习 2**：
```lua
local ts = os.time({ year = 2026, month = 1, day = 1 })
print(os.date("%Y-%m-%d", ts))   -- 2026-01-01
```

**练习 3**：
```lua
local function head(s, n)
    local stop = utf8.offset(s, n + 1)
    return stop and s:sub(1, stop - 1) or s
end
print(head("héllo世界", 3))   -- hél
```

**练习 4**：
```lua
local d = string.pack(">I4 >I4 >I4", 1, 256, 65536)
print(string.unpack(">I4 >I4 >I4", d))   -- 1  256  65536  13
```

**练习 5**：**假**。`io.lines("file")` 在迭代到文件末尾时**自动关闭**文件。但 `f:lines()`（已打开的句柄）不会自动关。

---

## 小结

| 库 | 关键点 |
|---|---|
| 基础 | print/type/pairs/select/load/require；`load` 带 mode 防字节码注入 |
| string | 模式匹配（L04）+ format/pack；`%q` 可反读 |
| table | concat 性能 / sort 不稳定 / pack 的 .n / move 搬移 |
| math | 5.3+ 整型函数；`math.random` 5.4 质量提升但**非密码学**；fmod≠% |
| os | `time` 秒级 / `clock` CPU 时间测耗时 / `date` 格式化 |
| io | 简单 vs 完整模型；`read` 格式 a/l/L/n；记得 close |
| string.pack | 5.3+ 二进制打包；字节序 `<`/`>`/`=`；处理协议 |
| utf8 | 5.3+ 字符级操作；`len`/`offset`/`codes`；LuaJIT 默认无 |

---

## 📅 2026 现状/更新

- `utf8`、`string.pack`、5.4 的高质量 `math.random` 是相对新的能力；**LuaJIT 对 5.3+ 标准库支持不完整**，OpenResty 里这些常由 `lua-resty-*` 或 FFI 补足。
- 高精度时间、网络、JSON、加密等不在标准库，靠生态：OpenResty 用 `ngx.*`、`cjson`、`resty.*`（L17–L21）；通用 Lua 用 LuaSocket、lua-cjson、luaossl 等。
- `string.pack` 让 PUC-Lua 不依赖 FFI 也能处理二进制协议，是 Redis 脚本、文件格式解析的实用工具。

---

> 🔁 下一篇 **L17 — 精通 OpenResty 架构与 ngx_lua 生命周期**：进入应用篇。OpenResty 如何把 Nginx 与 LuaJIT 结合、11 个处理阶段的 `*_by_lua` 指令、每请求一协程的非阻塞模型，以及 `ngx.*` 核心 API。
>
> 反馈：标准库要"知道有什么"，用时查手册；把 `string.pack` 和 `utf8` 这两个新能力记牢，它们最容易被忽略。
