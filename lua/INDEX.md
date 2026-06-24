# Lua 全场景深度课程 · 总目录

> 25 篇中文深度课程，覆盖 Lua 语言核心 → LuaJIT/C 嵌入 → OpenResty/Redis/Neovim/游戏全场景
> 每篇约 1 万–1.5 万字，含底层原理、可运行代码、mermaid 图、陷阱清单、练习题
> 适合从中级到高级的 Lua / OpenResty / 游戏 / 嵌入式工程师系统进阶
>
> **📅 内容基准：Lua 5.4.8**（2025-06，5.4 末版）+ **LuaJIT 2.1**（rolling）
> 并覆盖 **Lua 5.5.0**（2025-12-22 发布，五年来首个大版本）的新特性，详见 [L25](./L25-精通-Lua-5.4-新特性与-2026-现状.md)
> 应用篇另涉 OpenResty 1.27.x / Redis 7.x / Neovim 0.10+ / LÖVE 11.x

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| L01 | [精通 Lua 概览与运行模型](./L01-精通-Lua-概览与运行模型.md) | ⭐⭐⭐ | lua_State / 寄存器式字节码VM / 版本史 / 嵌入式定位 |
| L02 | [精通 Lua 值、类型与变量](./L02-精通-Lua-值-类型与变量.md) | ⭐⭐⭐ | 8 种类型 / nil 与 false / `_G`/`_ENV` / and-or 短路 |
| L03 | [精通 Lua 数值：整浮分离与运算](./L03-精通-Lua-数值-整浮分离与运算.md) | ⭐⭐⭐ | 5.3 整型 / 64-bit / `//` / 位运算 / 表键归一 |
| L04 | [精通 Lua 字符串与模式匹配](./L04-精通-Lua-字符串与模式匹配.md) | ⭐⭐⭐⭐ | 不可变/interning / Lua patterns(非正则) / gsub/gmatch |
| L05 | [精通 Lua 表(table)底层结构](./L05-精通-Lua-表-table-底层结构.md) | ⭐⭐⭐⭐ | 数组部分+哈希部分 / rehash / `#`与序列 / 引用语义 |
| L06 | [精通 Lua 元表与元方法](./L06-精通-Lua-元表与元方法.md) | ⭐⭐⭐⭐⭐ | `__index`/`__newindex` / 运算符重载 / `__call`/raw |
| L07 | [精通 Lua 函数、闭包与 upvalue](./L07-精通-Lua-函数-闭包与-upvalue.md) | ⭐⭐⭐⭐ | 多返回值展开/截断 / 变长参数 / upvalue 共享 / 尾调用 |
| L08 | [精通 Lua 协程 coroutine](./L08-精通-Lua-协程-coroutine.md) | ⭐⭐⭐⭐⭐ | 非对称协程 / yield-resume / 迭代器 / 生产者消费者 |
| L09 | [精通 Lua 错误处理](./L09-精通-Lua-错误处理.md) | ⭐⭐⭐⭐ | error/pcall/xpcall / 错误对象 / traceback / level |
| L10 | [精通 Lua 模块、require 与 LuaRocks](./L10-精通-Lua-模块-require-与-LuaRocks.md) | ⭐⭐⭐ | require 缓存/搜索器 / package.path / 模块写法 / LuaRocks |
| L11 | [精通 Lua 基于元表的 OOP](./L11-精通-Lua-基于元表的-OOP.md) | ⭐⭐⭐⭐ | `Class.__index=Class` / self / 继承 / 共享字段陷阱 |
| L12 | [精通 Lua 垃圾回收与弱表](./L12-精通-Lua-垃圾回收与弱表.md) | ⭐⭐⭐⭐⭐ | 增量/分代 GC / collectgarbage / `__gc` / 弱表 |
| L13 | [精通 LuaJIT：JIT 与性能模型](./L13-精通-LuaJIT-JIT-与性能模型.md) | ⭐⭐⭐⭐⭐ | trace JIT / NYI(`pairs`!) / 性能模型 / `jit.*` |
| L14 | [精通 LuaJIT FFI 与 C 数据](./L14-精通-LuaJIT-FFI-与-C-数据.md) | ⭐⭐⭐⭐⭐ | ffi.cdef / cdata(0-based!) / 零开销 / 64位整数 |
| L15 | [精通 Lua C API：嵌入宿主](./L15-精通-Lua-C-API-嵌入宿主.md) | ⭐⭐⭐⭐⭐ | 虚拟栈 / lua_push/to / userdata / lua_pcall |
| L16 | [精通 Lua 标准库全景](./L16-精通-Lua-标准库全景.md) | ⭐⭐⭐ | string/table/math/os/io / string.pack / utf8 |
| L17 | [精通 OpenResty 架构与 ngx_lua 生命周期](./L17-精通-OpenResty-架构与-ngx_lua-生命周期.md) | ⭐⭐⭐⭐ | 多worker/每请求一协程 / 11阶段 / 全局泄漏陷阱 |
| L18 | [精通 OpenResty cosocket 与非阻塞 IO](./L18-精通-OpenResty-cosocket-与非阻塞-IO.md) | ⭐⭐⭐⭐⭐ | cosocket / 连接池 setkeepalive / ngx.thread 并发 |
| L19 | [精通 OpenResty 共享内存与 lua-resty 生态](./L19-精通-OpenResty-共享内存与-lua-resty-生态.md) | ⭐⭐⭐⭐ | shared dict / lrucache / mlcache / lock 防击穿 |
| L20 | [精通 Lua 网关实战：限流·鉴权·灰度](./L20-精通-Lua-网关实战-限流鉴权灰度.md) | ⭐⭐⭐⭐ | 漏桶/令牌桶 / JWT/HMAC / 一致性哈希灰度 / APISIX |
| L21 | [精通 Redis Lua 脚本](./L21-精通-Redis-Lua-脚本.md) | ⭐⭐⭐⭐ | EVAL/KEYS/ARGV / 原子性 / Functions / 限流·锁脚本 |
| L22 | [精通 Neovim Lua 配置与插件](./L22-精通-Neovim-Lua-配置与插件.md) | ⭐⭐⭐ | init.lua / vim.api/opt / lazy.nvim / LSP / vim.uv |
| L23 | [精通 Lua 游戏脚本：Love2D 与嵌入](./L23-精通-Lua-游戏脚本-Love2D-与嵌入.md) | ⭐⭐⭐⭐ | 游戏循环/dt / 引擎嵌入 / 热重载 / 沙箱(禁FFI) |
| L24 | [精通 Lua 测试、调试与性能剖析](./L24-精通-Lua-测试-调试与性能剖析.md) | ⭐⭐⭐⭐ | busted / luacheck / luacov / debug / `-jp` profiler |
| L25 | [精通 Lua 5.4 新特性与 2026 现状](./L25-精通-Lua-5.4-新特性与-2026-现状.md) | ⭐⭐⭐⭐ | `<close>`/`<const>` / **5.5 全局声明/for只读** / 生态 |

---

## 🗺️ 按模块组织

### 🟢 模块一：语言基础（L01–L06）

> 从运行模型到元表，覆盖每个 Lua 程序员必须吃透的核心。

- **L01 运行模型**：源码→字节码→寄存器式 VM；`lua_State` 多实例；版本谱系
- **L02 类型与变量**：8 种类型；只有 nil/false 假；全局即 `_G`/`_ENV`；and/or 短路
- **L03 数值**：整浮分离、`//`、位运算；表键归一；LuaJIT 无整型
- **L04 字符串**：不可变/interning；**Lua 模式≠正则**；gsub/gmatch；concat 性能
- **L05 表**：数组部分+哈希部分、rehash；`#`与序列；引用语义；pairs 无序
- **L06 元表**：`__index`/`__newindex`、运算符重载、`__call`/`__gc`/`__close`、raw

### 🔵 模块二：函数与控制（L07–L09）

> 一等函数、协程、错误——Lua 控制流的精华。

- **L07 函数/闭包**：多返回值展开规则；变长参数；**upvalue 共享**；尾调用
- **L08 协程**：单线程协作式；yield-resume 双向传值；**写递归得迭代器**；流水线
- **L09 错误**：error/pcall/xpcall；level 参数；错误对象；traceback；warn

### 🟡 模块三：组织与内存（L10–L12）

> 把代码组织成模块、对象，并管好内存。

- **L10 模块**：require 缓存/搜索器；package.path；返回 table 写法；LuaRocks
- **L11 OOP**：`Class.__index=Class` 枢纽；self/继承；共享字段陷阱；middleclass
- **L12 GC**：增量/分代 GC；collectgarbage；`__gc` 终结器；弱表缓存

### 🔴 模块四：性能与嵌入（L13–L16）

> 把 Lua 用快、用进 C 的世界。

- **L13 LuaJIT**：trace JIT；**NYI（`pairs`!）**；性能模型；`-jv`/`-jp` 诊断
- **L14 FFI**：ffi.cdef；cdata（**0-based、无边界检查**）；零开销；64 位整数
- **L15 C API**：虚拟栈；push/to；C 函数；userdata；嵌入宿主；`lua_pcall`
- **L16 标准库**：string/table/math/os/io；**string.pack** 二进制；utf8

### 🟠 模块五：应用场景（L17–L25）

> Lua 作为"嵌入式胶水语言"的真实战场。

- **L17–L20 OpenResty**：架构与 11 阶段 / cosocket 与连接池 / 共享内存与多级缓存 / 网关实战
- **L21 Redis 脚本**：EVAL 原子性、KEYS/ARGV、Functions、限流/锁脚本
- **L22 Neovim**：init.lua、vim.api、lazy.nvim、内置 LSP、vim.uv 异步
- **L23 游戏**：游戏循环/dt、引擎嵌入、热重载、沙箱（禁 FFI）、Luau
- **L24 工程化**：busted/luacheck/luacov、debug、profiler
- **L25 新特性**：5.4 `<close>`/`<const>`/分代 GC + **5.5 全局声明/for 只读/紧凑数组**

---

## 🎯 学习路径建议

### 路径 A：从零到精通（3–6 个月）

按编号顺序通读，每周 2–3 篇。每篇：① 脑中过引言钩子代码、预测输出 → ② 通读、关键点动手敲 → ③ 做练习题 → ④ 看小结自检。

### 路径 B：OpenResty / 网关工程师（1–2 个月）

- 语言基础速过：**L01–L08**（重点 L05 表、L06 元表、L08 协程）
- LuaJIT：**L13**（NYI！）、**L14**（FFI）
- OpenResty 全家桶：**L17 → L18 → L19 → L20**
- 配合 **L21**（Redis 脚本，分布式限流/锁）、**L24**（luacheck 防全局泄漏）

### 路径 C：游戏脚本开发（1 个月）

- 核心：**L01–L08**、**L11**（OOP）、**L12**（GC 控帧率）
- 嵌入：**L15**（C API）、**L13**（LuaJIT 性能）
- 实战：**L23**（游戏循环/dt/沙箱），按需 Luau

### 路径 D：嵌入式 / 给 C 程序加脚本（3 周）

- **L01**（运行模型/lua_State）、**L02–L07**（语言核心）
- **L15**（C API 嵌入）、**L14**（LuaJIT FFI 对比）
- **L23**（嵌入引擎模式）、**L12**（内存控制）

### 路径 E：Neovim 配置进阶（2 周）

- **L01–L07** 语言基础、**L10** 模块
- **L22**（init.lua/vim.api/lazy.nvim/LSP/vim.uv）

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 运行脚本 | `lua script.lua` / `luajit script.lua` |
| 交互式 + 跑脚本 | `lua -i script.lua` |
| 看字节码 | `luac -l script.lua` |
| 预编译 | `luac -o out.luc in.lua` |
| LuaJIT trace 信息 | `luajit -jv script.lua` |
| LuaJIT 详细 dump | `luajit -jdump script.lua` |
| LuaJIT profiler | `luajit -jp script.lua` |
| 静态检查 | `luacheck src/` |
| 测试 | `busted` / `busted --coverage` |
| 覆盖率 | `luacov` |
| 包管理 | `luarocks install <pkg>` / `opm get <pkg>` |
| 格式化 | `stylua .` |
| 内存监控（脚本内） | `collectgarbage("count")` |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 解释为什么 `0` 和 `""` 在 Lua 里为真，只有 nil/false 为假
- [ ] 说清表的"数组部分 vs 哈希部分"，以及为什么含洞表的 `#` 未定义
- [ ] 用元表实现只读表、默认值表、运算符重载
- [ ] 解释多个闭包如何共享同一个 upvalue
- [ ] 用协程把一个递归遍历变成惰性迭代器
- [ ] 说出 LuaJIT 为什么 `pairs` 慢（NYI），如何用 `-jv` 找 trace abort
- [ ] 用 FFI 在 LuaJIT 里精确处理 64 位整数（并知道 cdata 0-based）
- [ ] 描述 OpenResty"每 worker 一 VM、每请求一协程、禁阻塞调用"
- [ ] 设计 lrucache + shared dict + lock 的多级缓存
- [ ] 写一个原子的 Redis 限流脚本和安全释放锁脚本
- [ ] 用 `<close>` 写错误安全的资源管理，并知道 LuaJIT 不支持
- [ ] 说出 Lua 5.5 相对 5.4 的关键变化（全局声明、for 只读）

---

## 🆕 版本特性快速索引

| 版本 | 关键特性 | 出现在 |
|---|---|---|
| **5.1** (2006) | require/协程标准化、`#` | L01/L08/L10；**LuaJIT 停在此** |
| **5.2** (2011) | `_ENV`、`goto`、废弃 `module()` | L02/L10 |
| **5.3** (2015) | **整型/浮点分离**、位运算符、`utf8`、`string.pack` | L03/L16 |
| **5.4** (2020) | **`<close>`**、**`<const>`**、分代 GC、`warn`、`coroutine.close` | L25/L12/L08/L09 |
| 5.4.8 (2025-06) | 5.4 系列末版补丁 | — |
| **5.5** (2025-12) | **全局变量声明**、**for 变量只读**、紧凑数组(-60%内存)、major GC 增量化、`table.create` 标准化 | **L25** |

权威来源：[Lua 版本历史](https://www.lua.org/versions.html) · [Lua 5.5 readme](https://www.lua.org/manual/5.5/readme.html) · [Lua 5.4 manual](https://www.lua.org/manual/5.4/)

---

## 📋 配套资源

- **可视化路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)（每章 5 题，共 125 题）
- **生态库选型地图**：见 [libraries/Lua-生态库选型地图.md](./libraries/Lua-生态库选型地图.md)（按 PUC-Lua / LuaJIT / OpenResty / Neovim / Love2D 三套运行时组织，2026 现状）
- **官方手册**：[Lua 5.4](https://www.lua.org/manual/5.4/) · [Lua 5.5](https://www.lua.org/manual/5.5/)
- **LuaJIT**：[luajit.org](https://luajit.org/) · **OpenResty**：[openresty.org](https://openresty.org/)
- **Programming in Lua**（PiL，作者 Roberto Ierusalimschy）：语言圣经

---

> 🔁 反馈：Lua 的精髓是"机制而非策略"——少数正交机制（表/元表/协程/lua_State）组合出无穷可能。把机制吃透，胜过背任何 API。写完每篇都建议跑一遍代码验证。
