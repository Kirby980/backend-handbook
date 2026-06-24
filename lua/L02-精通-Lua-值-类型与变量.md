# 精通 Lua 值、类型与变量

> 课程编号：L02
> 路线图来源：Lua 全场景深度课程 — 类型系统与变量
> 难度：⭐⭐⭐
> 预计阅读时间：50 分钟
> 📅 内容基准：**Lua 5.4.8** + **LuaJIT 2.1**

---

## 引言：四个会让人栽跟头的小问题

```lua
-- 1) 输出？
print(type(nil), type(true), type(10), type("x"), type(print), type({}))

-- 2) 输出？
local a, b, c = 1, 2
print(a, b, c)

-- 3) 输出？
local x = nil or "默认值"
local y = false and "A" or "B"
print(x, y)

-- 4) 输出？
local t = {}
t.self = t
print(t == t.self, t == {})
```

答案分别是：① `nil boolean number string function table`；② `1  2  nil`（右值不够，多出的变量补 `nil`）；③ `默认值  B`（`or` 返回第一个真值，`and`/`or` 返回的是**操作数本身**而非布尔）；④ `true  false`（表是**引用类型**，`t == t.self` 同一引用为真，`t == {}` 是两个不同表为假）。

这一章看似是"类型表 + 变量声明"的入门内容，实则藏着 Lua 类型系统最核心的几条规则——它们是元表、闭包、模块、沙箱的共同地基。

---

## 第一章：八种基本类型

Lua 是**动态类型**语言：**变量没有类型，值才有类型**。一共只有 **8 种**基本类型：

| 类型 | `type()` 返回 | 说明 |
|---|---|---|
| nil | `"nil"` | 唯一值 `nil`，表示"无/未定义" |
| boolean | `"boolean"` | `true` / `false` |
| number | `"number"` | 数字（5.3+ 内部再分整型/浮点，见 L03） |
| string | `"string"` | 不可变字节串（见 L04） |
| function | `"function"` | 一等函数（Lua 函数或 C 函数） |
| table | `"table"` | 唯一的复合结构（见 L05） |
| userdata | `"userdata"` | 宿主 C 数据（见 L15） |
| thread | `"thread"` | 协程（见 L08） |

记忆口诀：**nil / 布尔 / 数字 / 字符串 / 函数 / 表 / userdata / 线程**。其中 `function`、`table`、`userdata`、`thread` 是**引用类型**（按引用传递、比较），其余是**值类型**。

```lua
print(type(type))        -- function（type 本身是个函数）
print(type(type(1)))     -- string（type 的返回值是字符串！）
```

注意第二行：`type(x)` 返回的是**字符串**，所以判断类型要写 `if type(x) == "table"`，别误写成 `if type(x) == table`。

---

## 第二章：nil ——"无"的哲学

### 2.1 nil 表示"不存在"

```lua
local x              -- 声明但未赋值 → x 是 nil
print(x)             -- nil
print(undefined)     -- nil（访问未定义的全局变量，不报错，得 nil）

local t = {}
print(t.missing)     -- nil（表里不存在的键，得 nil）
```

Lua 中"不存在"和"值为 nil"是**同一回事**：

```lua
local t = {a = 1, b = 2}
t.a = nil            -- 等价于"从表里删除 a"
print(t.a)           -- nil
for k in pairs(t) do print(k) end   -- 只剩 b
```

这条规则有个重要推论：**你不能把 nil 当作表里的"占位值"存储**——存 nil 等于删除。需要"空占位"时常用一个哨兵值（如 `local NULL = {}`）。

### 2.2 全局变量删除

```lua
GLOBAL = 42
GLOBAL = nil         -- 把全局变量"还给"垃圾回收
```

把全局设为 nil 是清理全局污染的标准手段。

---

## 第三章：boolean ——只有 nil 和 false 是假

这是 Lua **最重要、也最容易被其他语言背景的人搞错**的一条规则：

> **在条件判断中，只有 `nil` 和 `false` 为"假"；其它一切（包括 `0`、`""`、`{}`、`0.0`）都为"真"。**

```lua
if 0 then print("0 为真") end           -- 打印
if "" then print("空串为真") end         -- 打印
if {} then print("空表为真") end         -- 打印
if 0.0 then print("0.0 为真") end        -- 打印
-- 只有这两个不会进：
if nil then end
if false then end
```

从 C/Python/JS 转来的人几乎都在这里栽过——在那些语言里 `0` / `""` 是假。判断"数值为零"或"字符串为空"必须**显式比较**：

```lua
if n ~= 0 then ... end
if s ~= "" then ... end
if next(t) ~= nil then ... end   -- 判断表非空（见 L05）
```

### 3.1 `and` / `or` 返回操作数本身

Lua 的 `and`、`or` **不返回布尔**，而是返回**决定结果的那个操作数**，并且**短路求值**：

- `a and b`：`a` 为假就返回 `a`，否则返回 `b`。
- `a or b`：`a` 为真就返回 `a`，否则返回 `b`。

```lua
print(1 and 2)        -- 2  （1 真 → 返回 b）
print(nil and 2)      -- nil（a 假 → 返回 a，短路，不求值 2）
print(nil or "默认")   -- 默认（a 假 → 返回 b）
print(false or nil)   -- nil
```

由此诞生两个 Lua 习语：

```lua
-- 1) 默认值
local name = arg_name or "anonymous"

-- 2) 三元表达式模拟
local sign = (x >= 0) and "+" or "-"
```

⚠️ 三元习语有一个**致命陷阱**：当"真分支"的值本身可能为 `false`/`nil` 时会失灵：

```lua
local v = true and false or "X"   -- 期望 false，实际得 "X"！
-- true and false → false（假）→ or "X" → "X"
```

所以 `cond and A or B` 只在 **A 永远为真值**时安全。否则老老实实用 `if`。

---

## 第四章：变量——局部、全局与作用域

### 4.1 默认全局，`local` 才是局部

Lua 的一个"危险默认"：**不加 `local` 的变量都是全局的**。

```lua
function f()
    x = 10          -- 没写 local → 创建/修改全局变量 x ！
    local y = 20    -- 局部，函数结束即销毁
end
f()
print(x)            -- 10（全局污染了）
print(y)            -- nil
```

这是 Lua 最大的"脚啰枪"之一——一个手滑忘写 `local`，就往全局环境里塞了个变量，可能与别处冲突，还可能在 OpenResty 这类长驻进程里造成**跨请求数据泄漏**（见 L17）。

**铁律**：**默认一律写 `local`**，只有确实需要全局时才不写。配合 `luacheck` 静态检查全局污染（见 L24）。

### 4.2 全局变量 = `_G` 的字段

第 L01 章已埋下伏笔：全局变量本质是**全局环境表 `_G`** 的字段。

```lua
x = 10
print(_G.x)          -- 10
print(_G["print"])   -- function: ...（连 print 都在 _G 里）

_G.y = 99
print(y)             -- 99
```

所以"读全局变量 `foo`"在底层就是"`_G.foo`"，"调用 `print`"就是"`_G.print`"。理解这一点，沙箱（替换 `_G`）、模块、`strict` 模式都迎刃而解。

### 4.3 `_ENV`——5.2+ 的环境机制

5.2 起，全局访问被重新定义为对一个叫 **`_ENV`** 的 upvalue 的索引：编译器把 `x`（全局）翻译成 `_ENV.x`。`_G` 只是 `_ENV` 默认指向的那张表。

```lua
-- 这两段等价（5.2+）
print(x)
print(_ENV.x)

-- 换掉 _ENV 就等于换了一套"全局环境"——沙箱的核心机制
local function sandboxed()
    local _ENV = { print = print }   -- 这个函数只能看到 print
    print("hi")
    -- os.time()  -- 这里会报错：os 在新 _ENV 里不存在
end
```

这是 L23（游戏脚本沙箱）的理论基础。LuaJIT（≈5.1）没有 `_ENV`，用的是旧的 `setfenv`/`getfenv`——又一个"你在用哪个 Lua"的差异点。

### 4.4 词法作用域与块

`local` 的作用域从**声明处**开始，到**所在块结束**：

```lua
local x = 1
do
    local x = 2      -- 新的 x，遮蔽外层
    print(x)         -- 2
end
print(x)             -- 1

-- 作用域从声明处开始：
local y = y          -- 右边的 y 是"外层/全局的 y"，左边是新 local
```

`if`/`while`/`for`/`do...end` 都引入新块。`for` 的循环变量是**每次迭代独立的局部变量**（这点和 Go 1.22 后类似，但 Lua 一直如此）：

```lua
local fns = {}
for i = 1, 3 do
    fns[i] = function() return i end
end
print(fns[1](), fns[2](), fns[3]())   -- 1  2  3（每个 i 独立，闭包各捕获各的）
```

这与 L07（闭包/upvalue）紧密相关。

---

## 第五章：多重赋值

Lua 支持一行多赋值，**右侧先全部求值，再整体赋给左侧**：

```lua
local a, b = 1, 2
a, b = b, a          -- 优雅交换，无需临时变量
print(a, b)          -- 2  1
```

变量与值**个数不匹配**的规则：

```lua
local a, b, c = 1, 2          -- 不够：c 补 nil  → 1 2 nil
local d, e = 1, 2, 3          -- 多余：3 被丢弃 → 1 2
```

函数多返回值在这里展开（详见 L07）：

```lua
local function minmax() return 1, 9 end
local lo, hi = minmax()       -- 1  9
local x = minmax()            -- 1（只取第一个，其余丢弃）
local y, z = minmax(), 100    -- ⚠️ y=1, z=100：函数不在末尾时只取 1 个值！
```

最后一行是经典陷阱：**多返回值只有出现在列表最后才会全部展开**，否则被截断为 1 个。

---

## 第六章：类型转换与判断

### 6.1 显式转换

```lua
tonumber("42")        -- 42
tonumber("0x1A")      -- 26（支持十六进制）
tonumber("ff", 16)    -- 255（指定进制）
tonumber("abc")       -- nil（转换失败返回 nil）
tostring(42)          -- "42"
tostring(nil)         -- "nil"
tostring(true)        -- "true"
```

### 6.2 算术上下文的自动强制（coercion）

Lua 在算术运算里会尝试把字符串转数字（这是历史包袱，5.4 收紧了规则）：

```lua
print("10" + 5)       -- 15（字符串被转成数字）
print(10 .. 20)       -- "1020"（数字被转成字符串，.. 是连接）
```

⚠️ 这种隐式转换可读性差、易藏 bug，**生产代码应显式 `tonumber`/`tostring`**。5.4 起字符串转数字的规则更严格（如不再接受某些边角格式）。

### 6.3 安全的类型判断

```lua
local function is_callable(v)
    return type(v) == "function"
       or (type(v) == "table" and type(getmetatable(v) and getmetatable(v).__call) == "function")
end
```

`type()` 是判断的基石；对"像函数一样可调用"的对象还要看元表 `__call`（见 L06）。

---

## 第七章：全局污染防御——strict 模式

长驻进程（OpenResty、游戏服务器）里，意外的全局变量是隐患。经典做法是给 `_G` 装一个元表，拦截"读/写未声明的全局"：

```lua
-- strict.lua（思路演示，生产可用 etc/strict.lua 或 luacheck）
setmetatable(_G, {
    __newindex = function(_, name)
        error("尝试创建全局变量 '" .. name .. "'（忘了 local?）", 2)
    end,
    __index = function(_, name)
        error("访问未定义全局变量 '" .. name .. "'", 2)
    end,
})

-- 此后：
foo = 1    -- ❌ 直接报错，逼你写 local 或显式 rawset
```

这是元表 `__index`/`__newindex` 的第一个实战应用——L06 会系统展开。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：用 `0` / `""` 当假

```lua
local n = 0
if n then print("总会进来") end   -- 0 为真
```
修正：`if n ~= 0 then`。

### ❌ 陷阱 2：忘写 `local` 污染全局

```lua
for i = 1, 3 do total = (total or 0) + i end   -- total 是全局！
```
修正：`local total = 0` 放循环外。

### ❌ 陷阱 3：`x and A or B` 中 A 可能为假

```lua
local enabled = config.enabled and true or false   -- 想规范化布尔，OK
local v = found and result or fallback              -- 若 result 可能是 false/nil 就出错
```
修正：值可能为假时用 `if`。

### ❌ 陷阱 4：多返回值非末尾被截断

```lua
local t = { f(), g() }   -- f() 只取 1 个值，g() 全展开
```
理解"只有最后一个表达式展开"。

### ❌ 陷阱 5：把 `type` 的返回当类型对象

```lua
if type(x) == table then ... end   -- ❌ 永远不成立（table 是值，type 返回字符串 "table"）
```
修正：`== "table"`。

### ❌ 陷阱 6：在表里用 nil 占位

```lua
local cache = {}
cache[key] = nil   -- 这不是"标记为空"，是"删除"
```
需要空占位用哨兵：`local NULL = {}; cache[key] = NULL`。

---

## 第九章：练习题

**练习 1**：输出？
```lua
print(nil and 1, nil or 1, false or nil, 1 and nil)
```

**练习 2**：下面哪些分支执行？
```lua
for _, v in ipairs({0, "", false, nil, {}}) do
    if v then io.write("T") else io.write("F") end
end
```
（提示：`ipairs` 遇到 `nil` 会停，见 L05）

**练习 3**：找 bug：
```lua
function counter()
    count = count + 1   -- 想做计数器
    return count
end
```

**练习 4**：输出？为什么？
```lua
local a, b, c = (function() return 1, 2, 3 end)()
local x, y, z = (function() return 1, 2, 3 end)(), 10
print(a, b, c)
print(x, y, z)
```

**练习 5**：用 `_ENV` 写一个只能访问 `print` 和 `math` 的受限函数。

---

## 参考答案与解析

**练习 1**：`nil  1  nil  nil`。逐个：`nil and 1`→nil（短路）；`nil or 1`→1；`false or nil`→nil；`1 and nil`→nil（1 真，返回 b=nil）。

**练习 2**：`ipairs` 从索引 1 连续遍历到第一个 nil 停止。数组是 `{0, "", false}`（第 4 个 nil 终止，`{}` 在第 5 位取不到）。所以遍历 `0, "", false`：`0`→T、`""`→T、`false`→F。输出 `TTF`。

**练习 3**：`count` 是全局变量且初值为 nil，`nil + 1` 直接报错 `attempt to perform arithmetic on a nil value`。修正：用闭包或显式初始化（见 L07）：
```lua
local function make_counter()
    local count = 0
    return function() count = count + 1; return count end
end
```

**练习 4**：
```
1  2  3
1  10  nil
```
第一行：函数调用是整个列表的最后（也是唯一）表达式，全展开 → 1,2,3。第二行：函数调用**不在末尾**（后面还有 10），被截断为 1 个值 → x=1, y=10, z=nil。

**练习 5**：
```lua
local function restricted()
    local _ENV = { print = print, math = math }
    print(math.sqrt(16))   -- 4
    -- os.time()  -- 报错：os 不在受限环境里
end
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 类型 | 8 种；变量无类型、值有类型；`type()` 返回**字符串** |
| nil | "不存在"="值为 nil"；存 nil = 删除 |
| 真假 | **只有 nil 和 false 假**；0、""、{} 皆真 |
| and/or | 返回**操作数本身**且短路；`x or 默认`、`c and A or B`（A 须真） |
| 变量 | 默认全局，**务必写 local**；全局即 `_G`/`_ENV` 字段 |
| 作用域 | 词法块作用域；`for` 循环变量每轮独立 |
| 多重赋值 | 右先求值；不足补 nil、多余丢弃；多返回值仅末尾展开 |
| 防御 | strict（`_G` 元表）+ luacheck 防全局污染 |

---

## 📅 2026 现状/更新

- **5.4** 起字符串↔数字的隐式强制规则收紧；仍建议显式 `tonumber`/`tostring`。
- `_ENV` 机制自 **5.2** 稳定，是现代沙箱（L23）与模块隔离的基础；**LuaJIT 用旧的 `setfenv`/`getfenv`**，迁移时注意。
- OpenResty 场景中"误用全局变量导致跨请求污染"仍是头号低级事故，`lua-resty-core` 与 `luacheck` 是标准防线（见 L17/L24）。

---

> 🔁 下一篇 **L03 — 精通 Lua 数值：整浮分离与运算**：5.3 引入的整型/浮点子类型、64 位整数、`//` 整除与 `^` 恒为浮点、位运算符、以及 `3//2` 与 `1==1.0` 作为表键时的微妙差异。
>
> 反馈：每个"输出？"问题都先口算再验证，错了的地方就是你认知的盲区。
