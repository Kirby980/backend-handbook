# 精通 Lua 测试、调试与性能剖析

> 课程编号：L24
> 路线图来源：Lua 全场景深度课程 — 工程化
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟
> 📅 内容基准：**Lua 5.4.8** / **LuaJIT 2.1** + **busted 2.x** / **luacheck** / **luacov**

---

## 引言：让 Lua 代码可信赖

```lua
-- 1) 这个 debug 函数能做什么？
local info = debug.getinfo(1, "nSl")
print(info.currentline, info.source)

-- 2) 怎么知道这个变量在哪被意外改成全局了？
-- 3) 怎么找出热点函数，而不是凭感觉优化？
```

Lua 灵活、动态——但灵活的另一面是易出错（忘 local 污染全局、`nil` 索引、类型混乱）。工程化工具链让 Lua 代码可信赖：**busted**（测试）、**luacheck**（静态检查）、**luacov**（覆盖率）、**debug 库 / 调试器**、**profiler**（性能）。这一章把它们串起来，是写生产级 Lua（OpenResty 服务、库、游戏）的最后一公里。

---

## 第一章：测试——busted

### 1.1 BDD 风格

**busted** 是最流行的 Lua 测试框架，BDD（行为驱动）风格：

```lua
-- spec/calculator_spec.lua
local calc = require("calculator")

describe("计算器", function()
    it("能做加法", function()
        assert.are.equal(5, calc.add(2, 3))
    end)

    it("能做除法", function()
        assert.are.equal(2, calc.div(6, 3))
    end)

    it("除零时报错", function()
        assert.has_error(function() calc.div(1, 0) end)
    end)

    describe("边界情况", function()
        it("处理负数", function()
            assert.is_true(calc.add(-1, 1) == 0)
        end)
    end)
end)
```

运行：

```bash
busted                    # 跑 spec/ 下所有测试
busted spec/calc_spec.lua # 指定文件
busted --coverage         # 配合 luacov 出覆盖率
```

### 1.2 断言库

```lua
assert.are.equal(expected, actual)       -- 相等（==）
assert.are.same({1,2}, {1,2})            -- 深度相等（表内容）
assert.is_true(x) / assert.is_false(x)
assert.is_nil(x) / assert.is_not_nil(x)
assert.has_error(fn[, expected_msg])     -- 期望抛错
assert.are_not.equal(a, b)
```

⚠️ `equal` 用 `==`（表比引用，L05）；比较表内容用 `same`（深度比较）。

### 1.3 setup/teardown

```lua
describe("数据库", function()
    local db
    before_each(function() db = create_test_db() end)   -- 每个 it 前
    after_each(function() db:close() end)                -- 每个 it 后

    it("能插入", function() db:insert(...) end)
end)
```

`setup`/`teardown`（整个 describe 一次）、`before_each`/`after_each`（每个 it）。

---

## 第二章：测试替身——mock 与 stub

依赖外部（网络、时间、随机）的代码要用替身隔离。busted 内置 `spy`/`stub`/`mock`：

```lua
-- stub：替换函数，控制返回值
it("处理 API 失败", function()
    local http = require("http")
    stub(http, "get", nil, "connection failed")   -- 让 http.get 返回失败
    local ok, err = my_module.fetch()
    assert.is_false(ok)
    http.get:revert()                               -- 还原！
end)

-- spy：记录调用但保留原行为
it("记录日志", function()
    local s = spy.on(logger, "info")
    do_work()
    assert.spy(s).was_called(1)
    assert.spy(s).was_called_with("done")
    s:revert()
end)
```

⚠️ stub/spy 用完**必须 `revert()`**（或在 after_each 还原），否则污染其它测试。

---

## 第三章：静态检查——luacheck

### 3.1 它能抓什么

**luacheck** 不运行代码就能发现一大类问题：

- **意外的全局变量**（忘写 local，L02/L17 的头号坑）
- 未使用的变量/参数
- 重复定义、遮蔽（shadowing）
- 访问未定义的字段

```bash
luacheck myfile.lua
luacheck src/             # 整个目录

# 输出示例：
# myfile.lua:5:5: setting non-standard global variable 'cont'   ← 忘了 local！
# myfile.lua:8:9: unused variable 'tmp'
```

### 3.2 配置 `.luacheckrc`

```lua
-- .luacheckrc
std = "lua54"                    -- 或 "luajit" / "ngx_lua"（OpenResty）
globals = { "ngx", "vim" }       -- 声明允许的全局（如 OpenResty 的 ngx）
ignore = { "212" }               -- 忽略特定警告码（如未用参数）
max_line_length = 120
```

针对场景设 `std`：OpenResty 用 `ngx_lua`（认识 `ngx.*`）、Neovim 用 `globals = {"vim"}`。把 luacheck 接入 CI，是防全局污染的标准防线。

---

## 第四章：覆盖率——luacov

```bash
# 配合 busted
busted --coverage
luacov                    # 生成 luacov.report.out

# 或手动
lua -lluacov myscript.lua
luacov
```

报告显示每行执行次数、未覆盖行、总覆盖率。`.luacov` 配置包含/排除文件。覆盖率帮你找到**没被测试触及的分支**——但记住覆盖率高 ≠ 测试好，关键路径和边界才是重点。

---

## 第五章：调试

### 5.1 `debug` 库

```lua
debug.getinfo(level, what)       -- 获取调用栈某层信息（函数名、行号、源）
debug.traceback([msg, level])    -- 调用栈字符串（L09 用过）
debug.getlocal(level, i)         -- 取某层第 i 个局部变量
debug.setlocal(level, i, v)
debug.getupvalue(fn, i)          -- 取闭包的 upvalue（L07 用过）
debug.sethook(fn, mask, count)   -- 设钩子（行/调用/计数，沙箱超时用过 L23）
```

```lua
-- 打印当前位置
local function where()
    local info = debug.getinfo(2, "nSl")   -- level 2 = 调用者
    return ("%s:%d"):format(info.short_src, info.currentline)
end

-- 简易追踪钩子：打印每次函数调用
debug.sethook(function(event)
    local info = debug.getinfo(2, "n")
    print(event, info.name or "?")
end, "c")    -- "c" = call 事件
```

⚠️ `debug` 库开销大、能破坏封装（读写别人的局部/upvalue），**生产代码别依赖它做正常逻辑**，仅用于调试/工具。`debug.sethook` 尤其影响性能。

### 5.2 交互式调试器

- **ZeroBrane Studio**：带图形调试器的 Lua IDE，支持断点、单步、变量查看。
- **mobdebug**：远程调试库（ZeroBrane 用它），能调试 OpenResty/游戏里的 Lua。
- **打印调试**：最朴素但有效——`print`/`ngx.log`（OpenResty）/`vim.notify`（Neovim）。

```lua
-- mobdebug 远程调试（在目标代码里）
require("mobdebug").start()   -- 连接到 ZeroBrane 调试器
```

---

## 第六章：性能剖析

### 6.1 LuaJIT profiler（`-jp`）

LuaJIT 自带采样 profiler，找热点最方便：

```bash
luajit -jp script.lua            # 默认：按函数采样
luajit -jp=v script.lua          # 详细（含 VM 状态）
luajit -jp=f script.lua          # 按函数 + 行
luajit -jp=10,profile.txt script.lua   # 每 10ms 采样，输出到文件
```

输出按耗时排序的热点函数。配合 L13 的 `-jv`（看 trace abort），能定位"哪里慢 + 为什么没 JIT"。

### 6.2 火焰图

LuaJIT profiler 输出可转成**火焰图**（FlameGraph），直观看调用栈耗时分布。OpenResty 生态有 `stapxx`、`lua-resty-* ` 的火焰图工具链（基于 SystemTap/eBPF），是线上 OpenResty 性能分析的利器。

### 6.3 PUC-Lua 的剖析

PUC-Lua 没内置 profiler，用 `debug.sethook` 自己采样，或用第三方库（`luatrace`、`pl.test` 的计时）。简单计时用 `os.clock`（L16）：

```lua
local t0 = os.clock()
heavy_work()
print(("%.3f s"):format(os.clock() - t0))
```

### 6.4 microbenchmark 注意

```lua
-- 多跑几次取均值，避免 JIT 预热/GC 干扰
local function bench(fn, n)
    collectgarbage()           -- 先清 GC（L12）
    local t0 = os.clock()
    for _ = 1, n do fn() end
    return (os.clock() - t0) / n
end
```

⚠️ LuaJIT 下要注意：**死代码消除**会优化掉"结果没被使用"的循环（测了个寂寞）——要用结果（如累加返回）防止被优化掉。

---

## 第七章：常见陷阱清单

### ❌ 陷阱 1：用 `equal` 比较表内容

```lua
assert.are.equal({1, 2}, {1, 2})   -- 失败！比引用（L05）
```
深度比较用 `assert.are.same`。

### ❌ 陷阱 2：stub 不 revert 污染测试

```lua
stub(m, "f")   -- 没 revert → 影响后续测试
```
用完 `m.f:revert()` 或在 after_each 还原。

### ❌ 陷阱 3：luacheck 不配 std 误报

```lua
-- OpenResty 代码不配 std=ngx_lua → 把 ngx 当未定义全局
```
按场景设 `std`/`globals`。

### ❌ 陷阱 4：生产依赖 debug 库

```lua
local x = debug.getlocal(...)   -- 慢且破坏封装，别用于正常逻辑
```
debug 仅用于调试/工具。

### ❌ 陷阱 5：benchmark 被死代码消除

```lua
for i = 1, 1e8 do compute(i) end   -- LuaJIT 可能整个优化掉
```
用结果（累加/返回）防优化。

### ❌ 陷阱 6：覆盖率当质量指标

```lua
-- 100% 覆盖但没测边界/错误路径，仍可能有 bug
```
关注关键路径和边界，而非数字。

---

## 第八章：练习题

**练习 1**：`assert.are.equal` 和 `assert.are.same` 的区别？

**练习 2**：写一个测试，验证某函数除零时抛错。

**练习 3**：luacheck 最常帮你抓到的 Lua 高频错误是什么？

**练习 4**：怎么找出一段 LuaJIT 代码的性能热点？

**练习 5**：找 bug：
```lua
it("test", function()
    stub(io, "open", nil)
    do_work()
    assert.is_false(check())
end)
```

---

## 参考答案与解析

**练习 1**：`equal` 用 `==`（表比引用，两个不同表即使内容相同也不等）；`same` 做**深度比较**（递归比表内容）。比较表内容用 `same`。

**练习 2**：
```lua
it("除零报错", function()
    assert.has_error(function() divide(1, 0) end, "除零")  -- 可选第二参匹配错误信息
end)
```

**练习 3**：**意外的全局变量**（忘写 `local`）——这是 Lua 默认全局（L02）+ OpenResty 长驻泄漏（L17）的头号来源，luacheck 一眼揪出。

**练习 4**：`luajit -jp script.lua`（采样 profiler）找耗时热点函数；再 `luajit -jv` 看有没有 trace abort（NYI，L13）导致热路径没 JIT；必要时转火焰图看调用栈分布。

**练习 5**：`stub(io, "open", nil)` 没 `revert`——会污染后续用到 `io.open` 的测试。修正：测试末尾 `io.open:revert()`，或放 after_each。

---

## 小结

| 工具 | 用途 |
|---|---|
| busted | BDD 测试；`equal`(引用)/`same`(深度)；setup/teardown |
| stub/spy/mock | 隔离依赖；**用完 revert** |
| luacheck | 静态检查，**抓全局污染**；`.luacheckrc` 设 std/globals |
| luacov | 覆盖率；高覆盖 ≠ 高质量 |
| debug 库 | getinfo/traceback/sethook；**仅调试，别用于逻辑** |
| 调试器 | ZeroBrane + mobdebug；OpenResty/游戏可远程调 |
| profiler | LuaJIT **`-jp`** 找热点 + `-jv` 看 abort；火焰图 |
| benchmark | 多跑取均值、先清 GC、**用结果防死代码消除** |

---

## 📅 2026 现状/更新

- **busted + luacheck + luacov** 仍是 2026 Lua 工程化的标准三件套，普遍接入 CI。
- OpenResty 性能分析靠 LuaJIT profiler + SystemTap/eBPF 火焰图（`stapxx` 等）；线上热点定位成熟。
- luacheck 的 `ngx_lua`/`luajit`/`lua54` 标准库定义让它能精准服务不同宿主（OpenResty/Neovim/通用），是防全局污染的第一道关。

---

> 🔁 下一篇 **L25 — 精通 Lua 5.4 新特性与 2026 现状**：全系列收官。深入 to-be-closed 变量 `<close>`、`<const>` 常量、整型与数值 for 的语义、分代 GC，并总览 Lua/LuaJIT/Luau 生态在 2026 的现状。
>
> 反馈：把 luacheck 接进你的项目 CI——它能在代码合并前挡掉一整类低级错误。
