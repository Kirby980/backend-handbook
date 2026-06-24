# Lua 全场景课程 · 知识检测测验

> 每章 5 道题，共 125 题。答案在最末尾"参考答案"章节。
> 题型：单选（含真假）、概念辨析、代码输出。
> 建议：先盖住答案独立完成；> 80% 正确视为掌握该章；< 50% 建议重读。
> 📅 基准：Lua 5.4.8 / LuaJIT 2.1，含 5.5.0 新特性。

---

## L01 — 概览与运行模型

**1.1 ⭐⭐** Lua 的虚拟机是哪种类型？
- A. 栈式（stack-based）
- B. 寄存器式（register-based）
- C. 直接解释 AST
- D. 编译成本地机器码

**1.2 ⭐** 以下哪行会执行 `print`？
- A. `if 0 then print(1) end`
- B. `if nil then print(2) end`
- C. `if false then print(3) end`
- D. 都不执行

**1.3 ⭐⭐** 一个 C 进程里能同时存在几个独立的 Lua 解释器？
- A. 只能 1 个（类似 Python GIL）
- B. 最多 CPU 核数个
- C. 任意多个（每个一个 `lua_State`）
- D. 取决于内存

**1.4 ⭐⭐⭐** 一个 `.lua` 文件被 `load` 后，本质是什么？
- A. 一个表
- B. 一个匿名函数（其参数是 `...`）
- C. 一段无法调用的字节码
- D. 一个协程

**1.5 ⭐⭐** LuaJIT 基于哪个 Lua 版本？
- A. 5.4　B. 5.3　C. 5.1（+部分扩展）　D. 5.5

---

## L02 — 值、类型与变量

**2.1 ⭐** Lua 有几种基本类型？
- A. 6　B. 7　C. 8　D. 10

**2.2 ⭐⭐** `print(nil or "x", false and 1, 0 and 2)` 输出？
- A. `x  false  2`
- B. `x  nil  0`
- C. `nil  false  2`
- D. `x  false  0`

**2.3 ⭐⭐** 全局变量 `x`（5.2+）等价于对什么的访问？
- A. `local.x`　B. `_ENV.x`　C. `self.x`　D. `package.x`

**2.4 ⭐** 以下哪些值为"假"？
- A. `0` 和 `nil`
- B. `""` 和 `false`
- C. `nil` 和 `false`
- D. `0`、`""`、`nil`、`false`

**2.5 ⭐⭐** `local a, b, c = 1, 2` 之后 `c` 是？
- A. `0`　B. `nil`　C. 报错　D. `2`

---

## L03 — 数值：整浮分离与运算

**3.1 ⭐⭐** `math.type(4 / 2)` 返回？
- A. `"integer"`　B. `"float"`　C. `"number"`　D. `nil`

**3.2 ⭐⭐** `2 ^ 10` 的结果类型是？
- A. integer 1024
- B. float 1024.0
- C. string "1024"
- D. 报错

**3.3 ⭐⭐⭐** 执行后 `t` 有几个键？
```lua
local t = {}; t[1] = "a"; t[1.0] = "b"; t[1.5] = "c"
```
- A. 1　B. 2　C. 3　D. 4

**3.4 ⭐⭐** `math.maxinteger + 1` 等于？
- A. 报错　B. 变成 float　C. `math.mininteger`（回绕）　D. `math.huge`

**3.5 ⭐⭐⭐** LuaJIT 下 `math.type(3)` 返回？
- A. `"integer"`　B. `"float"`　C. `nil`（LuaJIT 无此函数）　D. `"number"`

---

## L04 — 字符串与模式匹配

**4.1 ⭐⭐** Lua 模式中匹配"一个数字"用？
- A. `\d`　B. `%d`　C. `[0-9]+`（仅此）　D. `\\d`

**4.2 ⭐⭐⭐** `("a1b2"):gsub("%a", "X")` 返回几个值，分别是？
- A. 1 个：`"X1X2"`
- B. 2 个：`"X1X2"`, `2`
- C. 2 个：`"X1X2"`, `4`
- D. 1 个：`"XXXX"`

**4.3 ⭐⭐** `#"中文"`（UTF-8）等于？
- A. 2　B. 4　C. 6　D. 报错

**4.4 ⭐⭐⭐** 去除字符串首尾空白的惯用模式是？
- A. `"%s*(.*)%s*"`
- B. `"^%s*(.-)%s*$"`
- C. `"trim(%s)"`
- D. `"^(.+)$"`

**4.5 ⭐⭐** Lua 模式**不**支持以下哪个？
- A. 捕获 `()`　B. 量词 `*`　C. 交替 `|`　D. 锚点 `^`

---

## L05 — 表(table)底层结构

**5.1 ⭐⭐** Lua 表内部由哪两部分组成？
- A. 键区 + 值区
- B. 数组部分 + 哈希部分
- C. 栈 + 堆
- D. 静态 + 动态

**5.2 ⭐⭐⭐** `#{1, 2, nil, 4}` 的结果？
- A. 一定是 4
- B. 一定是 2
- C. 未定义（可能 2 或 4）
- D. 报错

**5.3 ⭐⭐** 判断字典型表是否为空，正确做法？
- A. `#t == 0`
- B. `t == {}`
- C. `next(t) == nil`
- D. `t == nil`

**5.4 ⭐⭐** `{1} == {1}` 的结果？
- A. `true`　B. `false`（比较引用）　C. 报错　D. `nil`

**5.5 ⭐⭐⭐** `pairs` 遍历表的顺序？
- A. 插入顺序
- B. 键排序
- C. 不保证任何顺序
- D. 总是从 1 开始

---

## L06 — 元表与元方法

**6.1 ⭐⭐** `__index` 元方法在什么时候触发？
- A. 写入任何键
- B. 读取表中**不存在**的键
- C. 删除键
- D. 遍历表

**6.2 ⭐⭐⭐** `__newindex` 对"表中**已存在**的键"赋值时？
- A. 触发 `__newindex`
- B. **不触发**（已存在键直接赋值）
- C. 报错
- D. 触发 `__index`

**6.3 ⭐⭐⭐** 在 `__newindex` 里要真正写入表，应该用？
- A. `t[k] = v`
- B. `rawset(t, k, v)`
- C. `setmetatable`
- D. `t.k = v`

**6.4 ⭐⭐** `__eq` 元方法什么时候触发？
- A. 任意两值比较
- B. 两个操作数同为表/userdata 且原始不等时
- C. 数字与表比较时
- D. 字符串比较时

**6.5 ⭐⭐** `rawget(t, k)` 的作用？
- A. 删除键
- B. 绕过 `__index` 直接取表自身的值
- C. 触发 `__index`
- D. 设置元表

---

## L07 — 函数、闭包与 upvalue

**7.1 ⭐⭐⭐** `print(f(), f())`，若 `f` 返回 `1,2`，输出？
- A. `1 2 1 2`
- B. `1 1 2`
- C. `1 2`
- D. `2 2`

**7.2 ⭐⭐** `(f())`（括号包裹多返回值函数）的作用？
- A. 全部展开
- B. 强制截断为 1 个值
- C. 报错
- D. 返回表

**7.3 ⭐⭐⭐** 捕获**同一个**局部变量的多个闭包之间？
- A. 各自拥有独立副本
- B. 共享那一个变量
- C. 无法访问该变量
- D. 只有第一个能访问

**7.4 ⭐⭐** 准确统计变长参数个数（含 nil）用？
- A. `#{...}`
- B. `select("#", ...)`
- C. `#arg`
- D. `count(...)`

**7.5 ⭐⭐⭐** 以下哪个是正确的尾调用？
- A. `return f(x) + 1`
- B. `return (f(x))`
- C. `return f(x)`
- D. `local y = f(x); return y`

---

## L08 — 协程 coroutine

**8.1 ⭐⭐** Lua 协程是？
- A. 操作系统线程
- B. 单线程协作式可暂停函数
- C. 多进程
- D. 真并行的轻量线程

**8.2 ⭐⭐** `coroutine.create(fn)` 之后，`fn` 立即执行吗？
- A. 立即执行
- B. 不执行，要 `resume` 才执行
- C. 执行一半
- D. 报错

**8.3 ⭐⭐⭐** `coroutine.wrap` 与 `coroutine.create`+`resume` 的关键区别？
- A. wrap 更慢
- B. wrap 返回函数、出错**直接抛出**（不返回状态布尔）
- C. wrap 不能传参
- D. 没有区别

**8.4 ⭐⭐** resume 一个已 `dead` 的协程？
- A. 重新开始
- B. 返回 `false, "cannot resume dead coroutine"`
- C. 报致命错误
- D. 返回 nil

**8.5 ⭐⭐⭐** 两个协程能同时运行在两个 CPU 核上吗？
- A. 能
- B. 不能（单线程协作式）
- C. 取决于核数
- D. 取决于 GOMAXPROCS

---

## L09 — 错误处理

**9.1 ⭐⭐** `pcall(f)` 成功时返回？
- A. `f` 的返回值
- B. `true` + `f` 的返回值
- C. `nil`
- D. `false`

**9.2 ⭐⭐⭐** 校验函数里 `error("bad", 2)` 的 level 2 让错误指向？
- A. error 所在行
- B. 调用该校验函数的那一行
- C. 程序第一行
- D. 不显示位置

**9.3 ⭐⭐** `error` 可以抛出什么类型的值？
- A. 只能字符串
- B. 任意类型（包括表）
- C. 只能数字
- D. 只能错误对象

**9.4 ⭐⭐⭐** 要在出错时拿到完整调用栈，应该用？
- A. `pcall`
- B. `xpcall(f, debug.traceback)`
- C. `assert`
- D. `error`

**9.5 ⭐⭐** `pcall` 能捕获语法错误吗？
- A. 能
- B. 不能（语法错误在编译期，pcall 在运行期）
- C. 只能捕获部分
- D. 取决于版本

---

## L10 — 模块、require 与 LuaRocks

**10.1 ⭐⭐** `require("mod")` 调用两次，模块代码执行几次？
- A. 2 次
- B. 1 次（有缓存 `package.loaded`）
- C. 0 次
- D. 取决于模块

**10.2 ⭐⭐** 现代 Lua 模块的标准写法是？
- A. `module("name", package.seeall)`
- B. 定义 `local M = {}` ... `return M`
- C. 全部用全局函数
- D. 用 `class`

**10.3 ⭐⭐** `require("a.b")` 中的 `.` 表示？
- A. 字段访问
- B. 目录层级（`a/b.lua`）
- C. 字符串拼接
- D. 方法调用

**10.4 ⭐⭐⭐** 热重载一个已加载模块，正确做法？
- A. 直接再 `require`
- B. `package.loaded["mod"] = nil` 后再 `require`
- C. 重启进程
- D. `reload("mod")`（内置）

**10.5 ⭐⭐** C 模块 `mylib.so` 的入口函数名是？
- A. `main`
- B. `luaopen_mylib`
- C. `init_mylib`
- D. `mylib_open`

---

## L11 — 基于元表的 OOP

**11.1 ⭐⭐⭐** 实现"类"的枢纽语句是？
- A. `setmetatable(o, {})`
- B. `Class.__index = Class`
- C. `Class.new = function() end`
- D. `require("class")`

**11.2 ⭐⭐** `obj:method()` 等价于？
- A. `obj.method()`
- B. `obj.method(obj)`
- C. `method(obj)`
- D. `Class.method()`

**11.3 ⭐⭐** 调用父类方法 `super` 在 Lua 里怎么写？
- A. `super.method(self)`
- B. `Parent.method(self)`
- C. `self.super.method()`
- D. Lua 有 `super` 关键字

**11.4 ⭐⭐⭐** 把可变默认值 `items = {}` 放在**类表**上的后果？
- A. 每个实例独立
- B. 所有实例**共享同一个** items 表
- C. 报错
- D. items 为 nil

**11.5 ⭐⭐** 漏写 `Class.__index = Class` 会导致？
- A. 构造失败
- B. 实例调用方法时 `attempt to call a nil value`
- C. 内存泄漏
- D. 正常工作

---

## L12 — 垃圾回收与弱表

**12.1 ⭐⭐** `collectgarbage("count")` 返回的单位是？
- A. 字节
- B. KB
- C. MB
- D. 对象数

**12.2 ⭐⭐⭐** Lua 5.4 新增了哪种 GC 模式？
- A. 引用计数
- B. 分代（generational）
- C. 标记整理
- D. 停止-复制

**12.3 ⭐⭐** 弱表 `__mode = "v"` 表示？
- A. 键弱引用
- B. 值弱引用
- C. 键值都弱
- D. 只读

**12.4 ⭐⭐⭐** `__gc` 终结器的调用时机？
- A. 对象不可达后立即
- B. 由 GC 决定，**非确定性**
- C. 程序退出时
- D. 手动调用

**12.5 ⭐⭐** 给对象挂"侧表"元数据又不阻止其回收，用？
- A. 普通表
- B. 弱键表（`__mode = "k"`）
- C. 全局表
- D. 注册表

---

## L13 — LuaJIT：JIT 与性能模型

**13.1 ⭐⭐⭐** LuaJIT 的 JIT 是哪种？
- A. 方法级 JIT（编译整个函数）
- B. trace JIT（编译一条执行路径）
- C. AOT 编译
- D. 解释器

**13.2 ⭐⭐⭐** 以下哪个操作是著名的 NYI（无法 JIT）？
- A. 数字 `for` 循环
- B. `ipairs`
- C. `pairs`
- D. 算术运算

**13.3 ⭐⭐** 查看 LuaJIT trace abort 用？
- A. `luajit -jv`
- B. `luajit -O3`
- C. `lua -d`
- D. `luajit --debug`

**13.4 ⭐⭐** LuaJIT 有原生 64 位整型吗？
- A. 有
- B. 没有（数字是 double，大整数靠 FFI）
- C. 只在 64 位平台有
- D. 5.3 起有

**13.5 ⭐⭐⭐** 热路径遍历数组，性能最好的写法？
- A. `for k,v in pairs(arr)`
- B. `for i=1,#arr do ... arr[i]`
- C. 递归
- D. `while next(arr)`

---

## L14 — LuaJIT FFI 与 C 数据

**14.1 ⭐⭐⭐** FFI 创建的 C 数组 `ffi.new("int[3]")` 索引从几开始？
- A. 1
- B. 0（真 C 数组）
- C. -1
- D. 取决于平台

**14.2 ⭐⭐** FFI 是哪个实现独有的？
- A. PUC-Lua 5.4
- B. LuaJIT
- C. 所有 Lua
- D. Lua 5.5

**14.3 ⭐⭐⭐** LuaJIT 处理精确 64 位整数的正确方式？
- A. 普通 number
- B. FFI `int64_t`（`123LL` 后缀）
- C. 字符串拼接
- D. `math.maxinteger`

**14.4 ⭐⭐** FFI cdata 数组越界访问 `a[100]`（容量 3）会？
- A. 返回 nil
- B. 未定义行为（可能崩溃/读垃圾）
- C. 自动扩容
- D. 报 Lua 错误

**14.5 ⭐⭐⭐** 沙箱里运行不可信代码，关于 FFI 要？
- A. 保留它方便调用
- B. **必须禁用**（可访问任意内存）
- C. 无所谓
- D. 只读模式

---

## L15 — Lua C API：嵌入宿主

**15.1 ⭐⭐** C 与 Lua 之间通过什么交换数据？
- A. 全局变量
- B. 虚拟栈
- C. 共享内存
- D. 管道

**15.2 ⭐⭐** C API 中索引 `-1` 指？
- A. 栈底
- B. 栈顶
- C. 注册表
- D. 错误

**15.3 ⭐⭐⭐** 一个 C 函数 `return 2;` 表示？
- A. 返回整数 2
- B. 向 Lua 返回 2 个值（栈顶 2 个）
- C. 出错码 2
- D. 压入 2

**15.4 ⭐⭐⭐** 嵌入时为什么用 `lua_pcall` 而非 `lua_call`？
- A. 更快
- B. `lua_call` 出错会 longjmp 可能崩宿主；pcall 保护
- C. 没区别
- D. pcall 支持更多参数

**15.5 ⭐⭐** 要把 C 对象（带 `__gc`）暴露给 Lua，用？
- A. light userdata
- B. full userdata
- C. light + 字符串
- D. 全局表

---

## L16 — 标准库全景

**16.1 ⭐⭐** 测量代码 CPU 耗时用？
- A. `os.time()`
- B. `os.clock()`
- C. `os.date()`
- D. `ngx.now()`

**16.2 ⭐⭐** UTF-8 字符串的"字符数"用？
- A. `#s`
- B. `utf8.len(s)`
- C. `string.len(s)`
- D. `s:byte()`

**16.3 ⭐⭐⭐** 处理二进制协议打包用（5.3+）？
- A. `string.format`
- B. `string.pack` / `string.unpack`
- C. `table.concat`
- D. `tostring`

**16.4 ⭐⭐** `math.random` 适合密码学用途吗？
- A. 适合
- B. 不适合（非 CSPRNG）
- C. 5.4 起适合
- D. 取决于种子

**16.5 ⭐⭐** `io.lines("file")` 迭代结束后？
- A. 需手动 close
- B. 自动关闭文件
- C. 文件保持打开
- D. 报错

---

## L17 — OpenResty 架构与生命周期

**17.1 ⭐⭐** OpenResty 每个 worker 有几个 LuaJIT VM？
- A. 共享 1 个
- B. 每 worker 一个独立 VM
- C. 每请求一个
- D. 每核一个

**17.2 ⭐⭐** 每个请求对应一个？
- A. 进程
- B. OS 线程
- C. 轻量协程
- D. VM

**17.3 ⭐⭐⭐** `os.execute("sleep 1")` 在 OpenResty 里的后果？
- A. 正常等待
- B. 阻塞卡死整个 worker（连同所有请求）
- C. 只影响当前请求
- D. 报错

**17.4 ⭐⭐** 跨阶段传递请求私有数据放？
- A. 全局变量
- B. 模块表
- C. `ngx.ctx`
- D. shared dict

**17.5 ⭐⭐⭐** 鉴权/限流应该放在哪个阶段？
- A. `content_by_lua`
- B. `access_by_lua`
- C. `log_by_lua`
- D. `init_by_lua`

---

## L18 — cosocket 与非阻塞 IO

**18.1 ⭐⭐⭐** cosocket 在等 IO 时，worker 在做什么？
- A. 空等
- B. 协程 yield 让出，处理其它请求
- C. 阻塞
- D. 新建线程

**18.2 ⭐⭐** 高频访问 Redis，用完连接应该？
- A. `close()`
- B. `setkeepalive()` 放回连接池
- C. 不处理
- D. 销毁

**18.3 ⭐⭐** cosocket 连域名前必须配置？
- A. `worker_processes`
- B. `resolver` 指令
- C. `lua_code_cache`
- D. SSL

**18.4 ⭐⭐⭐** 并发访问 3 个后端用？
- A. 串行 connect
- B. `ngx.thread.spawn` + `wait`
- C. 多个 worker
- D. 子进程

**18.5 ⭐⭐** 连接放回池前必须保证？
- A. 速度快
- B. 连接状态干净（响应读完、无残留）
- C. 加密
- D. 无要求

---

## L19 — 共享内存与 lua-resty 生态

**19.1 ⭐⭐** `lua_shared_dict` 能直接存 table 吗？
- A. 能
- B. 不能（只存标量，要序列化）
- C. 只能存小表
- D. 5.4 起能

**19.2 ⭐⭐⭐** `lua-resty-lrucache` 的作用范围？
- A. 跨 worker 共享
- B. 单 worker 内（可存任意 Lua 值）
- C. 跨机器
- D. 全局

**19.3 ⭐⭐⭐** 缓存击穿（热 key 失效瞬间大量回源）的防护？
- A. 加大缓存
- B. `lua-resty-lock` 让一个请求回源、其它等待
- C. 禁用缓存
- D. 多 worker

**19.4 ⭐⭐** shared dict 原子自增用？
- A. `get` + `set`
- B. `incr`
- C. `add`
- D. `replace`

**19.5 ⭐⭐** 多级缓存的推荐封装库？
- A. lua-cjson
- B. lua-resty-mlcache
- C. lua-resty-http
- D. lua-resty-jwt

---

## L20 — 网关实战：限流·鉴权·灰度

**20.1 ⭐⭐⭐** 多网关实例的**全局**限流应该用？
- A. 各自的 shared dict
- B. Redis 原子脚本
- C. 本地变量
- D. nginx limit_req

**20.2 ⭐⭐** JWT 的优势是？
- A. 加密强
- B. 无状态、网关本地验签不用查库
- C. 体积小
- D. 不会过期

**20.3 ⭐⭐⭐** 灰度发布分流应该用？
- A. `math.random`
- B. 一致性哈希（用户 ID），保证同一用户稳定路由
- C. 轮询
- D. 随机时间

**20.4 ⭐⭐** WAF 规则匹配应该用？
- A. Lua 模式 `string.match`
- B. `ngx.re`（PCRE 正则）
- C. `string.find`
- D. `==`

**20.5 ⭐⭐** APISIX 插件本质是？
- A. 独立进程
- B. 在 OpenResty 某阶段注册的 Lua 逻辑
- C. C 模块
- D. 配置文件

---

## L21 — Redis Lua 脚本

**21.1 ⭐⭐⭐** Redis 脚本中访问的 key 必须？
- A. 硬编码在脚本里
- B. 通过 `KEYS` 传入（Cluster 路由）
- C. 用全局变量
- D. 从 ARGV 拼接

**21.2 ⭐⭐** Redis 脚本执行期间？
- A. 可以并发其它命令
- B. 原子独占 Redis（不处理其它命令）
- C. 可以 sleep
- D. 可以阻塞

**21.3 ⭐⭐⭐** 脚本 `return 3.99` 客户端收到？
- A. `3.99`
- B. 整数 `3`（小数截断）
- C. 字符串 "3.99"
- D. 报错

**21.4 ⭐⭐⭐** 安全释放分布式锁为什么要用脚本？
- A. 更快
- B. 原子地"校验持有者==自己 → 才 DEL"，避免误删
- C. 省带宽
- D. Redis 要求

**21.5 ⭐⭐** Redis 脚本里 `x = 10`（无 local）？
- A. 创建全局变量
- B. 报错（禁止全局变量）
- C. 静默忽略
- D. 创建局部

---

## L22 — Neovim Lua 配置与插件

**22.1 ⭐⭐** Neovim 内嵌的是哪个 Lua？
- A. PUC-Lua 5.4
- B. LuaJIT
- C. Lua 5.1（非 JIT）
- D. Luau

**22.2 ⭐⭐** `vim.keymap.set` 的 rhs 可以是？
- A. 只能字符串
- B. 字符串或 Lua 函数
- C. 只能 Vimscript
- D. 只能命令

**22.3 ⭐⭐⭐** 2026 主流的 Neovim 插件管理器？
- A. vim-plug
- B. packer
- C. lazy.nvim
- D. Vundle

**22.4 ⭐⭐⭐** 在 libuv（`vim.uv`）回调里调 `vim.api` 要？
- A. 直接调
- B. 用 `vim.schedule` 切回主循环
- C. 用 pcall
- D. 不可能

**22.5 ⭐⭐** `vim.g.mapleader` 应该在何时设置？
- A. 任意位置
- B. 加载插件/定义映射**之前**（最前面）
- C. 最后
- D. 不需要设

---

## L23 — 游戏脚本：Love2D 与嵌入

**23.1 ⭐⭐** Love2D 中移动物体为什么 `x = x + speed * dt`？
- A. 美观
- B. 让移动**帧率无关**
- C. 防止溢出
- D. 必须的语法

**23.2 ⭐⭐** 游戏热重载保留运行时状态的关键？
- A. 不可能
- B. 状态外置、逻辑模块纯函数化
- C. 用全局变量
- D. 重启

**23.3 ⭐⭐⭐** 运行不可信脚本的沙箱，`load(code, name, mode, env)` 的 mode 应该？
- A. `"bt"`（允许字节码）
- B. `"t"`（只允许文本，防字节码注入）
- C. `"b"`
- D. 不设

**23.4 ⭐⭐⭐** 沙箱里**不应**保留哪些？
- A. `math`、`string`
- B. `os`、`io`、`load`、`ffi`
- C. `pairs`、`ipairs`
- D. `print`

**23.5 ⭐⭐** Roblox 用的 Lua 方言是？
- A. 标准 Lua 5.4
- B. Luau（渐进类型）
- C. LuaJIT
- D. MoonScript

---

## L24 — 测试、调试与性能剖析

**24.1 ⭐⭐⭐** busted 比较两个表的**内容**用？
- A. `assert.are.equal`（用 ==）
- B. `assert.are.same`（深度比较）
- C. `assert.is_true`
- D. `==`

**24.2 ⭐⭐** luacheck 最常帮你抓到的 Lua 错误是？
- A. 语法错误
- B. 意外的全局变量（忘 local）
- C. 类型错误
- D. 内存泄漏

**24.3 ⭐⭐** LuaJIT 找性能热点用？
- A. `luajit -jp`
- B. `luajit -O3`
- C. `time`
- D. `print`

**24.4 ⭐⭐** `debug` 库应该？
- A. 用于正常业务逻辑
- B. 仅用于调试/工具（开销大、破坏封装）
- C. 生产高频使用
- D. 从不使用

**24.5 ⭐⭐⭐** stub/spy 用完必须？
- A. 删除测试
- B. `revert()` 还原
- C. 重启
- D. 无需处理

---

## L25 — 5.4/5.5 新特性与 2026 现状

**25.1 ⭐⭐⭐** 5.4 的 `<close>` 变量何时释放？
- A. 由 GC 决定（非确定）
- B. 离开作用域时确定性调用 `__close`（含出错退出）
- C. 程序退出
- D. 手动调用

**25.2 ⭐⭐** `local MAX <const> = 100` 后 `MAX = 200`？
- A. 正常修改
- B. 编译错误
- C. 运行时警告
- D. 静默忽略

**25.3 ⭐⭐⭐** Lua 5.5（2025-12）的头条特性是？
- A. 移除协程
- B. 可选的全局变量声明（修正"默认全局"）
- C. 内置 JSON
- D. 内置正则

**25.4 ⭐⭐⭐** Lua 5.5 中 `for i = 1, 10 do i = i + 1 end`？
- A. 正常
- B. 编译错误（for 变量只读 const）
- C. 死循环
- D. 警告

**25.5 ⭐⭐** OpenResty（LuaJIT）能用 `<close>` 吗？
- A. 能
- B. 不能（LuaJIT ≈5.1，无此特性）
- C. 5.4 起能
- D. 配置后能

---

# 参考答案

## L01
1.1 **B**（寄存器式）｜1.2 **A**（0 为真）｜1.3 **C**（任意多 lua_State）｜1.4 **B**（chunk 是匿名函数）｜1.5 **C**（5.1+扩展）

## L02
2.1 **C**（8 种）｜2.2 **A**（`x`/`false`/`2`：or 返真值、and 返假操作数、`0 and 2` 返 2）｜2.3 **B**（`_ENV.x`）｜2.4 **C**（nil 和 false）｜2.5 **B**（不足补 nil）

## L03
3.1 **B**（`/` 恒 float）｜3.2 **B**（`^` 恒 float 1024.0）｜3.3 **B**（1 与 1.0 同键，加 1.5 共 2 键）｜3.4 **C**（回绕）｜3.5 **C**（LuaJIT 无 math.type）

## L04
4.1 **B**（`%d`）｜4.2 **B**（结果串 + 替换次数 2）｜4.3 **C**（6 字节）｜4.4 **B**（`^%s*(.-)%s*$`）｜4.5 **C**（无交替 `|`）

## L05
5.1 **B**（数组部分+哈希部分）｜5.2 **C**（含洞未定义）｜5.3 **C**（`next(t)==nil`）｜5.4 **B**（比引用）｜5.5 **C**（不保证顺序）

## L06
6.1 **B**（读缺失键）｜6.2 **B**（已存在键不触发）｜6.3 **B**（rawset 防递归）｜6.4 **B**（同为表/userdata）｜6.5 **B**（绕过 __index）

## L07
7.1 **B**（`1 1 2`：非末尾截断、末尾展开）｜7.2 **B**（括号截断）｜7.3 **B**（共享）｜7.4 **B**（select("#")）｜7.5 **C**（`return f(x)`）

## L08
8.1 **B**（单线程协作式）｜8.2 **B**（resume 才执行）｜8.3 **B**（wrap 直接抛错）｜8.4 **B**｜8.5 **B**（不能并行）

## L09
9.1 **B**（true + 返回值）｜9.2 **B**（指向调用者）｜9.3 **B**（任意类型）｜9.4 **B**（xpcall+traceback）｜9.5 **B**（语法错误在编译期）

## L10
10.1 **B**（有缓存，1 次）｜10.2 **B**（return M）｜10.3 **B**（目录层级）｜10.4 **B**（清 package.loaded）｜10.5 **B**（luaopen_mylib）

## L11
11.1 **B**（`Class.__index=Class`）｜11.2 **B**（`obj.method(obj)`）｜11.3 **B**（`Parent.method(self)`）｜11.4 **B**（所有实例共享）｜11.5 **B**（方法找不到）

## L12
12.1 **B**（KB）｜12.2 **B**（分代）｜12.3 **B**（值弱）｜12.4 **B**（非确定）｜12.5 **B**（弱键表）

## L13
13.1 **B**（trace JIT）｜13.2 **C**（pairs NYI）｜13.3 **A**（-jv）｜13.4 **B**（无原生整型）｜13.5 **B**（数字 for + #）

## L14
14.1 **B**（0-based）｜14.2 **B**（LuaJIT 独有）｜14.3 **B**（int64_t / LL）｜14.4 **B**（未定义行为）｜14.5 **B**（必须禁用 FFI）

## L15
15.1 **B**（虚拟栈）｜15.2 **B**（栈顶）｜15.3 **B**（返回 2 个值）｜15.4 **B**（pcall 保护防崩）｜15.5 **B**（full userdata）

## L16
16.1 **B**（os.clock）｜16.2 **B**（utf8.len）｜16.3 **B**（string.pack）｜16.4 **B**（非 CSPRNG）｜16.5 **B**（自动关闭）

## L17
17.1 **B**（每 worker 一 VM）｜17.2 **C**（轻量协程）｜17.3 **B**（卡死 worker）｜17.4 **C**（ngx.ctx）｜17.5 **B**（access 阶段）

## L18
18.1 **B**（yield 让出）｜18.2 **B**（setkeepalive）｜18.3 **B**（resolver）｜18.4 **B**（ngx.thread.spawn）｜18.5 **B**（状态干净）

## L19
19.1 **B**（只存标量）｜19.2 **B**（单 worker 内）｜19.3 **B**（lua-resty-lock）｜19.4 **B**（incr）｜19.5 **B**（mlcache）

## L20
20.1 **B**（Redis 脚本）｜20.2 **B**（无状态验签）｜20.3 **B**（一致性哈希）｜20.4 **B**（ngx.re PCRE）｜20.5 **B**（阶段注册 Lua）

## L21
21.1 **B**（经 KEYS）｜21.2 **B**（原子独占）｜21.3 **B**（整数 3，截断）｜21.4 **B**（原子校验+删）｜21.5 **B**（禁全局变量）

## L22
22.1 **B**（LuaJIT）｜22.2 **B**（字符串或函数）｜22.3 **C**（lazy.nvim）｜22.4 **B**（vim.schedule）｜22.5 **B**（最前面）

## L23
23.1 **B**（帧率无关）｜23.2 **B**（状态外置）｜23.3 **B**（"t" 防字节码）｜23.4 **B**（os/io/load/ffi）｜23.5 **B**（Luau）

## L24
24.1 **B**（same 深度比较）｜24.2 **B**（意外全局）｜24.3 **A**（-jp）｜24.4 **B**（仅调试）｜24.5 **B**（revert）

## L25
25.1 **B**（确定性 __close）｜25.2 **B**（编译错误）｜25.3 **B**（全局变量声明）｜25.4 **B**（for 变量只读）｜25.5 **B**（LuaJIT 不支持）

---

> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [ROADMAP.md](./ROADMAP.md) 可视化路线图
> 错题对应章节回去重读——尤其陷阱清单部分。
