# Lua 全场景课程 · Mermaid 可视化路线图

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
> 所有 Mermaid 图可在 GitHub、VS Code、Obsidian、Typora 等直接渲染
>
> **📅 内容基准：Lua 5.4.8 + LuaJIT 2.1**，覆盖 Lua 5.5.0（2025-12）新特性

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Lua 之旅]) --> M1[模块一: 语言基础]

    M1 --> L01[L01 运行模型]
    M1 --> L02[L02 类型与变量]
    M1 --> L03[L03 数值]
    M1 --> L04[L04 字符串与模式]
    M1 --> L05[L05 表底层]
    M1 --> L06[L06 元表]

    M1 --> M2[模块二: 函数与控制]
    M2 --> L07[L07 函数/闭包/upvalue]
    M2 --> L08[L08 协程]
    M2 --> L09[L09 错误处理]

    M2 --> M3[模块三: 组织与内存]
    M3 --> L10[L10 模块/require]
    M3 --> L11[L11 OOP]
    M3 --> L12[L12 GC/弱表]

    M3 --> M4[模块四: 性能与嵌入]
    M4 --> L13[L13 LuaJIT JIT]
    M4 --> L14[L14 FFI]
    M4 --> L15[L15 C API]
    M4 --> L16[L16 标准库]

    M4 --> M5[模块五: 应用场景]
    M5 --> L17[L17 OpenResty 架构]
    M5 --> L18[L18 cosocket]
    M5 --> L19[L19 共享内存/缓存]
    M5 --> L20[L20 网关实战]
    M5 --> L21[L21 Redis 脚本]
    M5 --> L22[L22 Neovim]
    M5 --> L23[L23 游戏脚本]
    M5 --> L24[L24 测试/调试/性能]
    M5 --> L25[L25 5.4-5.5 新特性]

    M5 --> End([Lua 全栈工程师])

    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef basic fill:#48bb78,stroke:#2f855a,color:#fff
    classDef ctrl fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef org fill:#ecc94b,stroke:#b7791f,color:#000
    classDef perf fill:#f56565,stroke:#c53030,color:#fff
    classDef app fill:#ed8936,stroke:#c05621,color:#fff

    class M1,M2,M3,M4,M5 module
    class L01,L02,L03,L04,L05,L06 basic
    class L07,L08,L09 ctrl
    class L10,L11,L12 org
    class L13,L14,L15,L16 perf
    class L17,L18,L19,L20,L21,L22,L23,L24,L25 app
```

---

## 🟢 模块一：语言基础（L01–L06）依赖图

```mermaid
graph LR
    L01[L01 运行模型] --> L02[L02 类型/变量]
    L02 --> L03[L03 数值]
    L02 --> L04[L04 字符串]
    L02 --> L05[L05 表]
    L05 --> L06[L06 元表]

    classDef basic fill:#48bb78,stroke:#2f855a,color:#fff
    class L01,L02,L03,L04,L05,L06 basic
```

**核心知识点速记**：
- L01：字节码 + 寄存器式 VM；`lua_State` 多实例；5.1–5.5 谱系
- L02：8 类型；只有 nil/false 假；全局即 `_G`/`_ENV`；and/or 返回操作数
- L03：整浮分离；`/`和`^`恒浮点；`//`整除；`1`与`1.0`同键
- L04：不可变/interning；**模式≠正则**；gsub 返回两值；concat 性能
- L05：数组部分+哈希部分；含洞 `#` 未定义；`next` 判空；pairs 无序
- L06：`__index`/`__newindex`；运算符重载；`__call`；rawset 防递归

---

## 🔵🟡 模块二/三：控制流与组织（L07–L12）

```mermaid
graph LR
    L07[L07 函数/闭包] --> L08[L08 协程]
    L07 --> L09[L09 错误]
    L07 --> L11[L11 OOP]
    L06[L06 元表] --> L11
    L02[L02 变量] --> L10[L10 模块]
    L10 --> L11
    L09 --> L12[L12 GC]
    L06 --> L12

    classDef ctrl fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef org fill:#ecc94b,stroke:#b7791f,color:#000
    classDef ref fill:#bee3f8,stroke:#2b6cb0,color:#000
    class L07,L08,L09 ctrl
    class L10,L11,L12 org
    class L02,L06 ref
```

**Lua 抽象机制心智模型**：
```mermaid
graph TB
    Table[表 table<br>唯一数据结构] --> Meta[元表 metatable]
    Meta --> OOP[OOP: __index 继承]
    Meta --> Op[运算符重载]
    Meta --> Resource[资源管理 __gc/__close]
    Closure[闭包 + upvalue] --> Private[私有状态]
    Closure --> Iter[迭代器]
    Coroutine[协程] --> Iter
    Coroutine --> Async[非阻塞 IO 基础]
    Iter --> App1[OpenResty/游戏逻辑]
    Async --> App1
    OOP --> App1

    style Table fill:#48bb78,color:#fff
    style Meta fill:#ecc94b,color:#000
    style Coroutine fill:#4299e1,color:#fff
```

---

## 🔴 模块四：性能与嵌入（L13–L16）

```mermaid
graph TD
    L13[L13 LuaJIT JIT] --> L14[L14 FFI]
    L13 -.对比.-> L15[L15 C API]
    L14 -.Lua调C.-> Cworld[C 世界]
    L15 -.C嵌Lua.-> Cworld
    L16[L16 标准库] -.string.pack/utf8.-> L13

    classDef perf fill:#f56565,stroke:#c53030,color:#fff
    class L13,L14,L15,L16 perf
    classDef c fill:#a0aec0,stroke:#4a5568,color:#000
    class Cworld c
```

**LuaJIT 性能决策流**：
```mermaid
flowchart TD
    Slow{代码慢?} --> Jp[luajit -jp 找热点]
    Jp --> Jv[luajit -jv 看 trace]
    Jv --> Abort{有 abort?}
    Abort -->|是 NYI| Fix1[换写法:<br>pairs→数字for<br>避免协程切换]
    Abort -->|否| Type{类型稳定?}
    Type -->|否| Fix2[统一变量类型]
    Type -->|是| Fix3[FFI 处理 C 数据<br>预分配 table.new]
    Fix1 --> Bench[重新 benchmark]
    Fix2 --> Bench
    Fix3 --> Bench

    style Abort fill:#f56565,color:#fff
```

---

## 🟠 模块五：应用场景（L17–L25）

```mermaid
graph TB
    LuaJIT[LuaJIT 引擎] --> OR[OpenResty]
    OR --> L17[L17 架构/11阶段]
    L17 --> L18[L18 cosocket/连接池]
    L18 --> L19[L19 共享内存/多级缓存]
    L19 --> L20[L20 网关:限流/鉴权/灰度]
    L20 -.全局限流/锁.-> L21[L21 Redis 脚本]

    LuaJIT --> L22[L22 Neovim]
    LuaJIT --> L23[L23 游戏/Love2D]
    CAPI[L15 C API] -.嵌入.-> L23

    L24[L24 测试/调试/性能] -.贯穿.-> L20
    L25[L25 5.4-5.5 新特性] -.语言演进.-> L17

    classDef app fill:#ed8936,stroke:#c05621,color:#fff
    classDef eng fill:#f56565,stroke:#c53030,color:#fff
    class L17,L18,L19,L20,L21,L22,L23,L24,L25 app
    class LuaJIT,CAPI eng
    class OR app
```

**OpenResty 请求生命周期（11 阶段）**：
```mermaid
graph LR
    Init[init_by_lua] --> IW[init_worker_by_lua]
    IW --> SSL[ssl_certificate]
    SSL --> RW[rewrite]
    RW --> AC["access<br>鉴权/限流"]
    AC --> BL["balancer<br>选后端/灰度"]
    BL --> CT["content<br>主响应"]
    CT --> HF[header_filter]
    HF --> BF[body_filter]
    BF --> LG["log<br>埋点"]

    style AC fill:#ecc94b,color:#000
    style CT fill:#48bb78,color:#fff
    style BL fill:#ed8936,color:#fff
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（推荐）

```mermaid
graph LR
    W1[第1-3周<br>L01-L06<br>语言基础] --> W2[第4-5周<br>L07-L12<br>控制/组织/内存]
    W2 --> W3[第6-7周<br>L13-L16<br>性能/嵌入]
    W3 --> W4[第8-11周<br>L17-L25<br>应用场景]

    style W1 fill:#48bb78
    style W2 fill:#4299e1
    style W3 fill:#f56565
    style W4 fill:#ed8936
```

### 路径 B：OpenResty 工程师

```mermaid
graph LR
    P1[L01-L08 语言] --> P2[L13-L14 LuaJIT/FFI]
    P2 --> P3[L17→L18→L19→L20 OpenResty]
    P3 --> P4[L21 Redis + L24 工程化]

    style P1 fill:#48bb78
    style P2 fill:#f56565
    style P3 fill:#ed8936
    style P4 fill:#ed8936
```

### 路径 C：游戏 / 嵌入

```mermaid
graph LR
    G1[L01-L08 语言] --> G2[L11 OOP + L12 GC]
    G2 --> G3[L15 C API + L13 LuaJIT]
    G3 --> G4[L23 游戏/嵌入/沙箱]

    style G1 fill:#48bb78
    style G2 fill:#ecc94b
    style G3 fill:#f56565
    style G4 fill:#ed8936
```

---

## 🧠 知识检索思维导图

```mermaid
mindmap
  root((Lua 全场景))
    语言基础
      运行模型 L01
        字节码/寄存器VM
        lua_State
        版本谱系
      类型变量 L02
        8种类型
        nil/false 唯二假
        _G/_ENV
      数值 L03
        整浮分离
        位运算
      字符串 L04
        不可变/interning
        模式≠正则
      表 L05
        数组+哈希部分
        序列与#
      元表 L06
        __index/__newindex
        运算符重载
    控制与组织
      函数闭包 L07
        多返回值
        upvalue共享
        尾调用
      协程 L08
        非对称
        迭代器
      错误 L09
        pcall/xpcall
        level
      模块 L10
      OOP L11
      GC L12
        增量/分代
        弱表
    性能嵌入
      LuaJIT L13
        trace JIT
        NYI
      FFI L14
        cdata 0-based
        64位整数
      C API L15
        虚拟栈
        userdata
      标准库 L16
        string.pack
        utf8
    应用场景
      OpenResty
        架构L17
        cosocketL18
        缓存L19
        网关L20
      Redis脚本 L21
      Neovim L22
      游戏 L23
        dt
        沙箱
      工程化 L24
      新特性 L25
        close/const
        5.5全局声明
```

---

## 📊 难度与重要性矩阵

> 难度：⭐ 简单 → ⭐⭐⭐⭐⭐ 极难　重要性：🔥🔥🔥🔥🔥 必学 → 🔥 选学

### 必学 + 难（高优先级，花时间啃）

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| L06 元表 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | Lua 抽象能力总开关 |
| L08 协程 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 迭代器 + OpenResty 非阻塞根基 |
| L05 表 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 唯一数据结构、`#` 陷阱 |
| L13 LuaJIT | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | NYI、性能模型 |
| L17 OpenResty | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 进程/阶段/全局陷阱 |

### 必学 + 性价比高

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| L02 类型变量 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | nil/false、全局陷阱 |
| L07 函数闭包 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | upvalue 共享、多返回值 |
| L09 错误处理 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | pcall/level/资源安全 |
| L03 数值 | ⭐⭐⭐ | 🔥🔥🔥🔥 | 整浮分离、LuaJIT 差异 |

### 按方向选学

| 课程 | 难度 | 何时学 |
|---|---|---|
| L14 FFI | ⭐⭐⭐⭐⭐ | LuaJIT/OpenResty/64位整数 |
| L15 C API | ⭐⭐⭐⭐⭐ | 嵌入宿主/游戏引擎 |
| L18–L20 OpenResty 进阶 | ⭐⭐⭐⭐ | 网关/Web 工程 |
| L21 Redis 脚本 | ⭐⭐⭐⭐ | 分布式限流/锁 |
| L22 Neovim | ⭐⭐⭐ | 编辑器配置 |
| L23 游戏 | ⭐⭐⭐⭐ | 游戏/嵌入 |
| L24 工程化 | ⭐⭐⭐⭐ | 生产项目 |
| L25 新特性 | ⭐⭐⭐⭐ | 跟进语言演进 |

---

## 🔗 跨章知识连接

某些主题贯穿多章——理解"为什么"比死记每章更重要。

```mermaid
graph TD
    Embed[可嵌入哲学 L01]
    Embed --> CAPI[C API L15]
    Embed --> Hosts[各宿主]
    Hosts --> OR2[OpenResty L17]
    Hosts --> Redis2[Redis L21]
    Hosts --> NV[Neovim L22]
    Hosts --> Game[游戏 L23]

    Which["你在用哪个 Lua?"]
    Which --> PUC[PUC 5.4/5.5:<br>整型/&lt;close&gt;]
    Which --> LJ[LuaJIT:<br>FFI/无整型/无&lt;close&gt;]
    LJ --> OR2
    LJ --> NV
    LJ --> Game
    PUC --> Redis5[Redis=5.1]

    Global[默认全局陷阱 L02]
    Global --> Leak[OpenResty 跨请求泄漏 L17]
    Global --> Strict[luacheck L24]
    Global --> Fix55[5.5 全局声明 L25]

    style Embed fill:#48bb78,color:#fff
    style Which fill:#f56565,color:#fff
    style Global fill:#ecc94b,color:#000
```

---

> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [QUIZ.md](./QUIZ.md) 测验题（125 题）
