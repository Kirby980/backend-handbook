# Python 后端深度课程 · 总目录

> 面向后端 / AI 工程师面试与生产的 Python 系统进阶课程，共 **30 篇**深度长文。
> 每篇约 10000-15000 字，含底层原理、CPython 源码视角、代码示例、生产实践、陷阱清单与练习题。
>
> **📅 内容基准：2026 年 6 月** —— Python 3.13（free-threading / 实验 JIT）主流可用、3.12 仍广泛在用、**3.14（2025-10）free-threading 转为官方支持**；Faster CPython 持续提速；类型注解（PEP 695 新语法）、asyncio 生态（FastAPI）、uv 工具链成为主流。
>
> 本专题与 [Go 专题](../golang/INDEX.md)、[Java 专题](../java/INDEX.md) 互为镜像——同样的并发、内存、类型、工程化主题，讲 Python 的实现与生态，三语言对照学习收获最大。是 OfferPilot 题库 Python 赛道的内容底座。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| P01 | [精通 Python 数据模型与对象](./P01-精通-Python-数据模型与对象.md) | ⭐⭐⭐ | 一切皆对象 / dunder / is vs == / 可变不可变 |
| P02 | [精通 Python 内置容器底层](./P02-精通-Python-内置容器底层.md) | ⭐⭐⭐⭐ | list / dict 紧凑哈希 / set / tuple / 复杂度 |
| P03 | [精通 Python 字符串与编码](./P03-精通-Python-字符串与编码.md) | ⭐⭐⭐ | str/bytes / Unicode / 格式化 / intern |
| P04 | [精通 Python 函数、参数与作用域](./P04-精通-Python-函数参数与作用域.md) | ⭐⭐⭐⭐ | LEGB / 闭包 / 默认可变参数坑 / *args/**kwargs |
| P05 | [精通 Python 迭代器与生成器](./P05-精通-Python-迭代器与生成器.md) | ⭐⭐⭐⭐ | iterable/iterator / yield / 生成器 / itertools |
| P06 | [精通 Python 装饰器与上下文管理器](./P06-精通-Python-装饰器与上下文管理器.md) | ⭐⭐⭐⭐ | 装饰器 / functools.wraps / with / contextlib |
| P07 | [精通 Python 推导式与函数式](./P07-精通-Python-推导式与函数式.md) | ⭐⭐⭐ | comprehension / 生成器表达式 / map-filter / lambda |
| P08 | [精通 Python 异常处理与最佳实践](./P08-精通-Python-异常处理.md) | ⭐⭐⭐ | 异常体系 / EAFP / 异常组 / 上下文管理 |
| P09 | [精通 Python 类与属性查找](./P09-精通-Python-类与属性查找.md) | ⭐⭐⭐⭐ | 实例/类属性 / `__dict__` / `__slots__` / 属性协议 |
| P10 | [精通 Python 继承与 MRO](./P10-精通-Python-继承与MRO.md) | ⭐⭐⭐⭐ | C3 线性化 / super / 多继承 / mixin |
| P11 | [精通 Python 描述符与 property](./P11-精通-Python-描述符与property.md) | ⭐⭐⭐⭐⭐ | 描述符协议 / property / 方法的本质 |
| P12 | [精通 Python 元类与类创建](./P12-精通-Python-元类与类创建.md) | ⭐⭐⭐⭐⭐ | type / metaclass / `__init_subclass__` |
| P13 | [精通 Python 鸭子类型、ABC 与 Protocol](./P13-精通-Python-鸭子类型与协议.md) | ⭐⭐⭐⭐ | duck typing / 抽象基类 / 结构化子类型 |
| P14 | [精通 Python 类型注解与 typing](./P14-精通-Python-类型注解与typing.md) | ⭐⭐⭐⭐ | type hints / 泛型 / PEP 695 / mypy / Pydantic |
| P15 | [精通 CPython 执行模型与字节码](./P15-精通-Python-执行模型与字节码.md) | ⭐⭐⭐⭐ | 源码→字节码 / 解释器循环 / PyObject / dis |
| P16 | [精通 Python 内存管理与垃圾回收](./P16-精通-Python-内存管理与GC.md) | ⭐⭐⭐⭐⭐ | 引用计数 / 分代 GC / 循环引用 / 弱引用 |
| P17 | [精通 Python GIL 原理](./P17-精通-Python-GIL原理.md) | ⭐⭐⭐⭐⭐ | 全局解释器锁 / 为何存在 / 影响 / free-threading |
| P18 | [精通 Python 对象内存优化](./P18-精通-Python-对象内存优化.md) | ⭐⭐⭐⭐ | `__slots__` / intern / 小整数缓存 / getsizeof |
| P19 | [精通 Python 性能剖析与优化](./P19-精通-Python-性能剖析与优化.md) | ⭐⭐⭐⭐ | cProfile / timeit / 热点 / 向量化 / C 扩展 |
| P20 | [精通 Python 多线程 threading](./P20-精通-Python-多线程threading.md) | ⭐⭐⭐⭐ | GIL 下线程 / 同步原语 / IO 密集 |
| P21 | [精通 Python 多进程 multiprocessing](./P21-精通-Python-多进程multiprocessing.md) | ⭐⭐⭐⭐ | 绕过 GIL / IPC / 进程池 / 共享内存 |
| P22 | [精通 Python asyncio 与协程原理](./P22-精通-Python-asyncio与协程.md) | ⭐⭐⭐⭐⭐ | 事件循环 / async-await / coroutine / Task |
| P23 | [精通 Python 异步实战与陷阱](./P23-精通-Python-异步实战与陷阱.md) | ⭐⭐⭐⭐ | 并发原语 / 阻塞调用陷阱 / 异步生态 |
| P24 | [精通 Python 并发选型与 free-threading](./P24-精通-Python-并发选型与free-threading.md) | ⭐⭐⭐⭐⭐ | 线程/进程/异步 / concurrent.futures / no-GIL |
| P25 | [精通 Python 包管理与虚拟环境](./P25-精通-Python-包管理与虚拟环境.md) | ⭐⭐⭐ | pip / venv / poetry / uv / 打包 / 依赖锁 |
| P26 | [精通 Python 测试](./P26-精通-Python-测试.md) | ⭐⭐⭐⭐ | pytest / fixture / 参数化 / mock / 覆盖率 |
| P27 | [精通 Python 日志与可观测](./P27-精通-Python-日志与可观测.md) | ⭐⭐⭐ | logging 体系 / 结构化日志 / OTel |
| P28 | [精通 Python Web 框架与 WSGI/ASGI](./P28-精通-Python-Web框架与WSGI-ASGI.md) | ⭐⭐⭐⭐ | Flask/Django/FastAPI / 同步异步 / Pydantic |
| P29 | [精通 Python 数据库访问与 ORM](./P29-精通-Python-数据库访问与ORM.md) | ⭐⭐⭐⭐ | DB-API / SQLAlchemy / 连接池 / N+1 / 异步驱动 |
| P30 | [精通 Python 版本特性演进](./P30-精通-Python-版本特性演进.md) | ⭐⭐⭐⭐ | 3.11 提速 / 3.12 typing / 3.13 free-threading+JIT / 3.14 |

---

## 🗺️ 按模块组织

### 🟢 模块一：语言核心（P01-P08）

> Python 的"地基"——对象模型、内置容器、函数与作用域、迭代/生成、装饰器。面试与日常的高频区。

- **P01 数据模型**：一切皆对象、`__dunder__` 协议、`is` vs `==`、可变 vs 不可变、`id()`
- **P02 内置容器**：list 动态数组、dict 紧凑哈希表（3.7+ 有序）、set、tuple，及各操作复杂度
- **P03 字符串与编码**：str（Unicode 码点）vs bytes、编解码、f-string、intern
- **P04 函数与作用域**：LEGB 规则、闭包与 `nonlocal`、**默认可变参数陷阱**、`*args/**kwargs`
- **P05 迭代器与生成器**：iterable/iterator 协议、`yield`、生成器、`itertools`、惰性
- **P06 装饰器与上下文管理器**：装饰器本质（高阶函数 + 闭包）、`functools.wraps`、`with`/`contextlib`
- **P07 推导式与函数式**：列表/字典/集合推导、生成器表达式、`map/filter`、`lambda`
- **P08 异常处理**：异常体系、EAFP vs LBYL、`try/except/else/finally`、异常组（3.11）

### 🔵 模块二：面向对象与类型系统（P09-P14）

> Python OOP 的深水区——属性查找机制、MRO、描述符、元类，以及渐进式类型系统。

- **P09 类与属性查找**：实例/类属性、`__dict__`、`__slots__`、属性查找顺序
- **P10 继承与 MRO**：C3 线性化、`super()` 协作式继承、多继承、mixin
- **P11 描述符与 property**：描述符协议（`__get__/__set__`）、`@property`、**方法/函数本质是描述符** ⭐
- **P12 元类**：`type` 是类的类、`metaclass`、`__new__` vs `__init__`、`__init_subclass__`
- **P13 鸭子类型/ABC/Protocol**：duck typing、抽象基类（`abc`）、结构化子类型（`Protocol`）
- **P14 类型注解**：type hints、泛型、PEP 695 新语法 `class C[T]`、`mypy`、Pydantic 关联

### 🟡 模块三：CPython 内幕与内存（P15-P19）

> 理解 CPython 是 Python 高级工程师的分水岭，性能与并发问题全靠它。

- **P15 执行模型**：源码→AST→字节码→解释器循环（ceval）、`PyObject`、`dis` 反汇编
- **P16 内存管理与 GC**：引用计数（主）+ 分代回收（处理循环引用）、`gc` 模块、弱引用 ⭐
- **P17 GIL**：全局解释器锁是什么、为何存在、对多线程的影响、**free-threading（no-GIL）** ⭐
- **P18 内存优化**：`__slots__` 省内存、字符串 intern、小整数缓存（-5~256）、`sys.getsizeof`
- **P19 性能剖析**：`cProfile`/`timeit`/`py-spy`、热点优化、向量化（NumPy）、C 扩展/Cython 思路

### 🟣 模块四：并发与异步（P20-P24）

> Python 并发的"重头戏"，也是和 Go/Java 差异最大的地方（GIL 的存在塑造了一切）。

- **P20 多线程**：GIL 下线程只适合 **IO 密集**、`threading` 同步原语、线程安全
- **P21 多进程**：`multiprocessing` 绕过 GIL 做 **CPU 并行**、IPC、进程池、共享内存
- **P22 asyncio 协程**：事件循环、`async/await`、coroutine 本质、`Task`、单线程高并发 IO ⭐
- **P23 异步实战**：`TaskGroup`/并发原语、**阻塞调用毁灭事件循环**的陷阱、异步生态
- **P24 并发选型**：线程 vs 进程 vs 异步决策树、`concurrent.futures`、**free-threading 的冲击** ⭐

### 🟠 模块五：工程化（P25-P27）

> 让 Python 代码可信赖、可交付：依赖、测试、日志。

- **P25 包管理**：`pip`/`venv`/`poetry`/**`uv`**、依赖解析与锁、`pyproject.toml`、打包发布
- **P26 测试**：`pytest`、fixture、参数化、`unittest.mock`、覆盖率、Testcontainers 关联
- **P27 日志与可观测**：`logging`（Logger/Handler/Formatter）、结构化日志、OpenTelemetry

### 🔴 模块六：Web、数据与演进（P28-P30）

- **P28 Web 框架**：WSGI vs ASGI、Flask/Django（同步）、FastAPI（异步 + Pydantic）、选型
- **P29 数据库访问**：DB-API、SQLAlchemy（Core/ORM）、连接池、N+1、异步驱动（asyncpg）
- **P30 版本演进**：3.11 提速（PEP 659）→ 3.12（PEP 695 类型/per-interpreter GIL）→ 3.13（free-threading/JIT）→ 3.14

---

## 🎯 学习路径建议

### 路径 A：Python 后端面试突击（3-4 周）

```
P01 数据模型 + P02 容器 + P04 作用域（必背基础）
   ↓
P05 生成器 + P06 装饰器 + P11 描述符（高频原理）
   ↓
P16 内存/GC + P17 GIL（分水岭考点）
   ↓
P20-P22 并发三件套（线程/进程/asyncio）+ P24 选型
```

### 路径 B：CPython 内幕专精（2 周）

```
P15 执行模型/字节码 → P16 内存与 GC → P17 GIL
   ↓
P18 内存优化 → P19 性能剖析
```

### 路径 C：异步与并发专精（2 周）

```
P17 GIL（前置）→ P20 线程 → P21 进程
   ↓
P22 asyncio 原理 → P23 异步实战 → P24 并发选型 + free-threading
```

### 路径 D：AI / 数据后端（结合 [ai-backend](../ai-backend/INDEX.md)）

```
P01-P08 语言核心 → P14 类型注解（Pydantic）
   ↓
P22-P24 异步并发 → P28 FastAPI → P29 数据库 → ai-backend RAG/Agent
```

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **生态库选型地图**：见 [Python-生态库选型地图.md](./libraries/Python-生态库选型地图.md)（2026 实战精选：uv/ruff/Pydantic v2/FastAPI/polars… 按场景选型）
- **综合测验**：见 [QUIZ.md](./QUIZ.md)（开放题，配套 AI 补答案）
- **镜像专题**：[Go](../golang/INDEX.md) / [Java](../java/INDEX.md)——三语言对照学习并发与内存模型
- **AI 后端**：[ai-backend/](../ai-backend/INDEX.md)——Python 是 LLM 应用的主力语言

---

## 🔁 与 Go / Java 专题的对照

三语言在并发与类型系统上思路迥异，对照学习收获最大：

| 主题 | Python | Go | Java |
|---|---|---|---|
| 执行方式 | 解释（CPython 字节码）+ 实验 JIT | 编译为机器码 | 字节码 + JVM JIT |
| 并发模型 | 线程(GIL)/进程/asyncio | goroutine + channel | 线程 + 池 / 虚拟线程 |
| 真并行 | 多进程 / free-threading（[P24](./P24-精通-Python-并发选型与free-threading.md)） | 原生 goroutine | 原生线程 |
| 内存管理 | 引用计数 + 分代 GC（[P16](./P16-精通-Python-内存管理与GC.md)） | 三色标记并发 GC | 分代 / G1 / ZGC |
| 类型系统 | 动态 + 渐进类型注解（[P14](./P14-精通-Python-类型注解与typing.md)） | 静态 + 泛型 | 静态 + 泛型（擦除） |
| 多态 | 鸭子类型 / Protocol（[P13](./P13-精通-Python-鸭子类型与协议.md)） | 隐式接口 | 显式 interface |
| 包管理 | pip / poetry / uv（[P25](./P25-精通-Python-包管理与虚拟环境.md)） | go modules | Maven / Gradle |

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **P17/P24 GIL** | **free-threading（no-GIL，PEP 703）**：3.13 实验性、**3.14（2025-10）转为官方支持**（仍是可选构建）——Python 真多核并行成为可能，颠覆"Python 用不了多核"的认知 |
| **P15/P19 性能** | **Faster CPython**：3.11 提速 10-60%（PEP 659 自适应专门化解释器）；3.13 引入**实验 JIT**（PEP 744，copy-and-patch） |
| **P14 类型** | **PEP 695（3.12）**新泛型语法 `class C[T]`、`type X = ...` 语句；类型注解 + mypy/Pyright 成主流；Pydantic v2（Rust 核心）大幅提速 |
| **P25 工具链** | **`uv`（Rust 实现）** 成为极速包/项目管理器，在很多场景取代 pip/poetry/pyenv |
| **P22/P28 异步** | asyncio 生态成熟：`TaskGroup`（3.11 结构化并发）、FastAPI/ASGI 主流 |
| **P16 内存** | per-interpreter GIL（PEP 684，3.12）为子解释器并行铺路（`InterpreterPoolExecutor`，3.14） |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 讲清"一切皆对象"、`is` 与 `==`、可变与不可变、默认可变参数为什么是坑
- [ ] 说清 dict 紧凑哈希表结构与有序性、各容器操作复杂度
- [ ] 解释迭代器/生成器协议、`yield` 的惰性，写出生成器流水线
- [ ] 讲清装饰器本质、`functools.wraps` 为什么必要、上下文管理器协议
- [ ] 手画属性查找顺序、C3 MRO，解释描述符为什么是"方法的本质"
- [ ] 说清引用计数 + 分代 GC 如何协作、循环引用怎么处理
- [ ] 讲透 GIL 是什么、为何存在、对多线程/多进程/asyncio 的影响
- [ ] 给场景正确选型：线程 / 进程 / asyncio / free-threading
- [ ] 解释 asyncio 事件循环、为什么阻塞调用会毁掉并发
- [ ] 用 pytest + fixture + mock 写分层测试，用 cProfile 定位热点

---

> 🔁 反馈：发现错误或建议改进，直接 PR 改 markdown。
