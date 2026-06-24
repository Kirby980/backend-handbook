# 精通 Lua 函数、闭包与 upvalue

> 课程编号：L07
> 路线图来源：Lua 全场景深度课程 — 函数与闭包
> 难度：⭐⭐⭐⭐
> 预计阅读时间：55 分钟
> 📅 内容基准：**Lua 5.4.8** + **LuaJIT 2.1**

---

## 引言：函数比你以为的更"活"

```lua
-- 1) 输出？
local function f() return 1, 2, 3 end
print(f())
print((f()))           -- 多了一层括号
local t = {f(), f()}
print(#t)

-- 2) 这两个闭包共享变量吗？
local function make()
    local n = 0
    local function inc() n = n + 1; return n end
    local function get() return n end
    return inc, get
end
local inc, get = make()
inc(); inc()
print(get())

-- 3) 输出？
local function sum(...)
    local s = 0
    for _, v in ipairs({...}) do s = s + v end
    return s
end
print(sum(1, 2, 3, 4))
```

答案：① `1 2 3`，然后 `(f())` 用括号**截断为 1 个值** → `1`，`#t` 是 `4`（第一个 `f()` 截断为 1，第二个在末尾展开为 3，共 4 个）；② **共享**——`inc` 和 `get` 捕获**同一个 upvalue `n`**，所以 `get()` 返回 `2`；③ `10`。

函数是 Lua 的"一等公民"，闭包和 upvalue 是它最精妙的部分——它们是 OOP（L11）、迭代器（L08）、回调、模块私有状态的共同基础。这一章把"多返回值的展开规则"和"upvalue 的共享语义"两个高频踩坑点彻底讲清。

---

## 第一章：一等函数

### 1.1 函数是值

函数可以赋值、传参、返回、存进表——和数字、字符串一样：

```lua
local f = function(x) return x * 2 end   -- 匿名函数赋给变量
local list = { f, print, math.sin }       -- 函数存进表
list[1](21)                               -- 调用 → 42

local function apply(fn, x) return fn(x) end
print(apply(f, 10))                       -- 20（函数作参数）
```

### 1.2 声明语法都是语法糖

```lua
-- 这三种等价：
local function f(x) return x end
local f = function(x) return x end       -- 但前者支持递归引用自身

-- 全局
function g(x) end          -- 等价 g = function(x) end，即 _G.g = ...

-- 表字段
function t.method(x) end   -- 等价 t.method = function(x) end
function t:method(x) end   -- 等价 t.method = function(self, x) end —— 注意隐式 self！
```

`t:method()` 的**冒号**自动加 `self` 参数——这是 OOP 的语法基础（L11）。`local function f` 与 `local f = function` 的细微区别：前者先声明 `f` 再赋值，**函数体内可引用 `f` 自身**（递归）；后者赋值时 `f` 还是 nil。

```lua
local function fact(n) return n <= 1 and 1 or n * fact(n-1) end   -- ✅ 能递归
local fact2 = function(n) return n <= 1 and 1 or n * fact2(n-1) end  -- fact2 在体内是 upvalue，但定义时还没赋值
```

---

## 第二章：多返回值——展开与截断

Lua 函数可以返回**任意多个值**，这是它区别于多数语言的特性。但"什么时候全展开、什么时候截断为 1 个"有精确规则。

### 2.1 返回多值

```lua
local function minmax(t)
    return math.min(table.unpack(t)), math.max(table.unpack(t))
end
local lo, hi = minmax({3, 1, 4, 1, 5})
print(lo, hi)    -- 1  5
```

### 2.2 展开规则

多返回值（以及 `...`、`table.unpack`）**只有出现在"表达式列表的最后一个位置"时才全部展开**，否则截断为 1 个值。出现"最后位置"的场景：

- 函数调用的最后一个参数
- 表构造器的最后一个元素
- `return` 的最后一个表达式
- 多重赋值的最后一个右值

```lua
local function f() return 1, 2, 3 end

print(f())             -- 1 2 3（f() 是 print 的最后/唯一参数 → 展开）
print(f(), 10)         -- 1 10（f() 不在末尾 → 截断为 1）
print(10, f())         -- 10 1 2 3（f() 在末尾 → 展开）

local t = {f()}        -- {1, 2, 3}
local u = {f(), 10}    -- {1, 10}（f() 被截断）
local v = {10, f()}    -- {10, 1, 2, 3}

local a, b = f()       -- a=1, b=2（取前两个）
local c = f()          -- c=1（只要一个）
```

### 2.3 用括号强制截断

**给多值表达式加一层括号 `( )`，强制只取第 1 个值**：

```lua
print((f()))           -- 1（括号截断）
local a, b = (f())     -- a=1, b=nil
```

这是一个常用技巧：当你只想要第一个返回值时，`(expr)` 明确截断。

```mermaid
graph TD
    E["多值表达式 f() / ... / unpack"] --> P{在表达式列表末尾?}
    P -->|是| Expand[全部展开]
    P -->|否，后面还有元素| Trunc[截断为 1 个值]
    E --> Paren{被括号包裹?}
    Paren -->|(expr)| Trunc
    style Expand fill:#48bb78,color:#fff
    style Trunc fill:#f56565,color:#fff
```

---

## 第三章：变长参数 `...`

### 3.1 接收与转发

```lua
local function log(level, ...)
    print(level, ...)          -- ... 在末尾，全部转发
end
log("INFO", "user", "login")   -- INFO  user  login
```

### 3.2 收集与计数

```lua
local function f(...)
    local args = {...}         -- 收进表（但含 nil 时长度不可靠！）
    print(#args)
    print(select("#", ...))    -- select("#") 才是准确的参数个数（含 nil）
    print(select(2, ...))      -- 从第 2 个开始的所有参数
end
f("a", nil, "c")
-- #args 可能是 1 或 3（nil 洞），select("#") 准确返回 3
```

⚠️ `{...}` 在含 nil 参数时长度不可靠（见 L05 的 `#` 问题）。**准确计数用 `select("#", ...)`，准确收集用 `table.pack(...)`（有 `.n`）**：

```lua
local function f(...)
    local args = table.pack(...)
    for i = 1, args.n do print(i, args[i]) end   -- 正确处理 nil
end
```

### 3.3 `...` 只在 vararg 函数体内可用

```lua
local function g(...)
    local function inner()
        -- print(...)   -- ❌ 错误：inner 不是 vararg 函数
    end
end
```

`...` 不能跨函数边界，要转发得先收集。

---

## 第四章：闭包与 upvalue —— 核心中的核心

### 4.1 什么是闭包

**闭包（closure）= 函数 + 它捕获的外部局部变量（upvalue）**。当内层函数引用了外层的局部变量，这个变量就成为内层函数的 **upvalue**，即使外层函数已返回，upvalue 依然存活。

```lua
local function counter()
    local n = 0                          -- n 是局部变量
    return function() n = n + 1; return n end   -- 内层捕获 n → n 成为 upvalue
end
local c = counter()
print(c(), c(), c())   -- 1  2  3（n 在 counter 返回后依然存活，被闭包持有）
```

`n` 不在栈上销毁，而是被闭包"关闭（close over）"——这就是"闭包"得名的由来。

### 4.2 多个闭包共享同一 upvalue

关键语义：**捕获同一个局部变量的多个闭包，共享那一个变量**（不是各拷一份）。

```lua
local function make_account(balance)
    local function deposit(n) balance = balance + n end
    local function withdraw(n) balance = balance - n end
    local function get() return balance end
    return deposit, withdraw, get
end
local dep, wd, get = make_account(100)
dep(50); wd(30)
print(get())   -- 120（三个闭包操作同一个 balance）
```

`deposit`、`withdraw`、`get` 共享同一个 `balance` upvalue——这是用闭包实现"对象私有状态"的基础（一种不需要元表的 OOP，见 L11）。

### 4.3 循环中的闭包：每轮独立

Lua 的 `for` 循环变量**每次迭代是新的局部变量**，所以循环里创建的闭包各自捕获独立的值：

```lua
local fns = {}
for i = 1, 3 do
    fns[i] = function() return i end
end
print(fns[1](), fns[2](), fns[3]())   -- 1  2  3（各捕获各的 i）
```

⚠️ 但如果在循环外声明变量，则所有闭包共享它：

```lua
local fns = {}
local i                              -- 循环外声明
for j = 1, 3 do
    i = j
    fns[j] = function() return i end  -- 都捕获同一个外层 i
end
print(fns[1](), fns[2](), fns[3]())   -- 3  3  3（共享 i，循环结束 i=3）
```

这正是 L01/L02 提过的差异——Lua 因"循环变量每轮独立"而天然没有 JS 那个 `var` 闭包坑。

### 4.4 用 `debug` 观察 upvalue

```lua
local function f()
    local x = 10
    return function() return x end
end
local g = f()
print(debug.getupvalue(g, 1))   -- x  10（upvalue 名和值）
```

---

## 第五章：尾调用（Tail Call）

### 5.1 什么是尾调用

当函数的**最后一个动作是 `return another_function(...)`**，Lua 做**正确的尾调用（proper tail call）**——复用当前栈帧而非新建，所以**不增加调用栈深度**。

```lua
local function loop(n)
    if n == 0 then return "done" end
    return loop(n - 1)        -- 尾调用：复用栈帧
end
print(loop(1000000))          -- "done"，不会栈溢出！
```

普通递归 100 万层会爆栈；尾递归不会，因为它复用栈帧（类似循环）。

### 5.2 什么"不是"尾调用

```lua
return f(x) + 1        -- ❌ 不是：返回后还要 +1
return f(x), g(x)      -- ❌ 不是尾调用（多个返回表达式）
return (f(x))          -- ❌ 不是：括号截断算一次操作
local y = f(x); return y  -- ❌ 不是：return 的是变量不是调用
```

只有 `return f(...)` **精确这一形式**才是尾调用。

### 5.3 用途

尾调用让你写"状态机"式的相互递归而不担心爆栈：

```lua
local function odd(n)  if n == 0 then return false end return even(n-1) end
function even(n) if n == 0 then return true  end return odd(n-1)  end
print(even(100000))    -- true，无栈溢出
```

---

## 第六章：常见陷阱清单

### ❌ 陷阱 1：多返回值非末尾被截断

```lua
local t = { f(), g() }   -- f() 截断为 1，只有 g() 展开
local x = f() or 0       -- f() 被 or 截断为 1 个值
```

### ❌ 陷阱 2：`{...}` 含 nil 时计数错误

```lua
local function f(...) return #{...} end
print(f(1, nil, 3))      -- 可能 1 或 3
```
修正：`select("#", ...)` 或 `table.pack(...).n`。

### ❌ 陷阱 3：以为循环外变量被每轮独立捕获

```lua
local handlers = {}
local name                       -- 外层！
for _, n in ipairs(names) do
    name = n
    handlers[n] = function() use(name) end   -- 全捕获同一个 name → bug
end
```
修正：把 `name` 移进循环体内 `local name = n`。

### ❌ 陷阱 4：误以为 `return (f())` 是尾调用

```lua
return (f())   -- 括号截断，不是尾调用，会增加栈帧
```

### ❌ 陷阱 5：`local function` vs `local f = function` 的递归差异

```lua
local fib = function(n) return n < 2 and n or fib(n-1) + fib(n-2) end  -- fib 在体内可能是 nil/全局
```
递归函数用 `local function fib(...)`。

### ❌ 陷阱 6：`...` 在嵌套函数里不可见

```lua
local function f(...)
    return (function() return ... end)()   -- ❌ 内层不是 vararg
end
```
修正：先 `local args = table.pack(...)` 再用。

---

## 第七章：练习题

**练习 1**：输出？
```lua
local function f() return 1, 2 end
print(f(), f(), f())
```

**练习 2**：实现一个生成"唯一自增 ID"的函数，每次调用 +1，状态私有。

**练习 3**：这两个闭包的输出？
```lua
local function pair()
    local x = 0
    return function() x = x + 1; return x end,
           function() x = x + 10; return x end
end
local a, b = pair()
print(a(), b(), a())
```

**练习 4**：判断哪些是尾调用：
```lua
-- (1) return f(x)
-- (2) return f(x) + 0
-- (3) return x and f(x)
-- (4) return f(g(x))
```

**练习 5**：用闭包实现一个 `once(f)`，让 `f` 只在第一次调用时执行，后续返回首次结果。

---

## 参考答案与解析

**练习 1**：`1 1 1 2`。前两个 `f()` 不在末尾各截断为 1，最后一个 `f()` 在末尾展开为 1,2。所以 `1 1 1 2`。

**练习 2**：
```lua
local function make_id()
    local id = 0
    return function() id = id + 1; return id end
end
local next_id = make_id()
print(next_id(), next_id())   -- 1  2
```

**练习 3**：`1  11  12`。共享 `x`：`a()` → x=1 返回 1；`b()` → x=11 返回 11；`a()` → x=12 返回 12。

**练习 4**：尾调用是 **(1)** 和 **(3)**。(1) `return f(x)` 标准尾调用；(3) `return x and f(x)` 当 x 真时整体是 `f(x)`，是尾调用。(2) 返回后还要 `+0` 不是；(4) `f(g(x))`：`f(...)` 是尾调用，但 `g(x)` 不是（它是 f 的参数，普通调用）。

**练习 5**：
```lua
local function once(f)
    local called, result = false, nil
    return function(...)
        if not called then called = true; result = f(...) end
        return result
    end
end
local init = once(function() print("初始化"); return 42 end)
print(init(), init())   -- 打印一次"初始化"，返回 42  42
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 一等函数 | 函数是值；`t:m()` 隐式加 `self`；递归用 `local function` |
| 多返回值 | 仅在**末尾位置**展开，否则截断为 1；`(expr)` 强制截断 |
| 变长参数 | `select("#",...)` 准确计数；`table.pack().n` 准确收集；`...` 不跨函数 |
| 闭包 | 函数 + 捕获的 upvalue；外层返回后 upvalue 仍存活 |
| upvalue 共享 | 多闭包捕获同一变量则**共享**；私有状态/无元表 OOP 的基础 |
| 循环闭包 | `for` 变量每轮独立；循环外变量则共享 |
| 尾调用 | 仅 `return f(...)` 形式；复用栈帧，深递归不爆栈 |

---

## 📅 2026 现状/更新

- 闭包/upvalue/尾调用语义自始稳定，跨 PUC-Lua 与 LuaJIT 一致。
- `table.pack`/`table.unpack`（5.2+ 标准化）是处理含 nil 变长参数的正解。
- 尾调用在 OpenResty 的回调链、状态机式协议解析中很实用（避免深栈）；闭包则是 `lua-resty-*` 库封装私有连接状态的常用手段（L19）。

---

> 🔁 下一篇 **L08 — 精通 Lua 协程 coroutine**：非对称协程的 `create`/`resume`/`yield`、与线程的本质区别、双向传值、以及用协程实现生成器、迭代器、生产者-消费者流水线。
>
> 反馈：闭包共享 upvalue 这一点，自己写个"银行账户"对象（deposit/withdraw/balance）巩固。
