# 现代 C++ 全栈深度课程 · 总目录

> 32 篇中文深度课程，覆盖 C++ 语言核心 → 对象模型 → 模板/泛型 → STL → 并发 → 现代特性与工程化
> 每篇含底层原理、可运行代码、mermaid 图、陷阱清单、练习题
> **📅 内容基准：C++23**（ISO/IEC 14882:2024）+ GCC 14 / Clang 18 / MSVC 19.4x；覆盖 C++26 进展（见 C32）

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| C01 | [编译模型与构建](./C01-精通-C++-编译模型与构建.md) | ⭐⭐⭐⭐ | TU / ODR / 链接 / name mangling / CMake |
| C02 | [类型系统与 auto](./C02-精通-C++-类型系统与-auto.md) | ⭐⭐⭐⭐ | auto/decltype 推导 / const / CTAD / 四种 cast |
| C03 | [值类别与移动语义](./C03-精通-C++-值类别与移动语义.md) | ⭐⭐⭐⭐⭐ | lvalue/rvalue / 移动构造 / std::move / noexcept |
| C04 | [完美转发与引用折叠](./C04-精通-C++-完美转发与引用折叠.md) | ⭐⭐⭐⭐⭐ | 转发引用 / 引用折叠 / std::forward |
| C05 | [RAII 与智能指针](./C05-精通-C++-RAII-与智能指针.md) | ⭐⭐⭐⭐ | unique_ptr / shared_ptr / weak_ptr / 零法则 |
| C06 | [拷贝控制与五法则](./C06-精通-C++-拷贝控制与五法则.md) | ⭐⭐⭐⭐ | 特殊成员函数 / 三五零法则 / =default/delete |
| C07 | [类与对象布局](./C07-精通-C++-类与对象布局.md) | ⭐⭐⭐⭐ | 内存布局 / 对齐 / EBO / trivial/POD |
| C08 | [虚函数与多态](./C08-精通-C++-虚函数与多态.md) | ⭐⭐⭐⭐⭐ | vtable/vptr / 虚析构 / override/final / RTTI |
| C09 | [继承与 CRTP](./C09-精通-C++-继承与-CRTP.md) | ⭐⭐⭐⭐ | 继承方式 / 菱形 / 虚继承 / CRTP 静态多态 |
| C10 | [运算符重载与三路比较](./C10-精通-C++-运算符重载与三路比较.md) | ⭐⭐⭐⭐ | operator / explicit / `<=>` / 字面量 |
| C11 | [模板基础](./C11-精通-C++-模板基础.md) | ⭐⭐⭐⭐ | 函数/类模板 / 特化 / 偏特化 / 实例化 |
| C12 | [可变参数模板](./C12-精通-C++-可变参数模板.md) | ⭐⭐⭐⭐⭐ | parameter pack / 折叠表达式 / 递归展开 |
| C13 | [模板元编程与 SFINAE](./C13-精通-C++-模板元编程与-SFINAE.md) | ⭐⭐⭐⭐⭐ | SFINAE / type_traits / if constexpr / 编译期 |
| C14 | [Concepts](./C14-精通-C++-Concepts.md) | ⭐⭐⭐⭐ | concept / requires / 约束 / 替代 SFINAE |
| C15 | [模板进阶](./C15-精通-C++-模板进阶.md) | ⭐⭐⭐⭐⭐ | 依赖名/typename / ADL / CRTP / 策略 |
| C16 | [STL 容器内部](./C16-精通-C++-STL-容器内部.md) | ⭐⭐⭐⭐ | vector/map/unordered_map 内部 / 迭代器失效 |
| C17 | [迭代器与范围](./C17-精通-C++-迭代器与范围.md) | ⭐⭐⭐⭐ | 迭代器类别 / sentinel / 失效规则 |
| C18 | [STL 算法](./C18-精通-C++-STL-算法.md) | ⭐⭐⭐ | 算法库 / 谓词 / 执行策略 / erase-remove |
| C19 | [Ranges](./C19-精通-C++-Ranges.md) | ⭐⭐⭐⭐ | views / 惰性 / 管道 / ranges 算法 |
| C20 | [lambda 与函数对象](./C20-精通-C++-lambda-与函数对象.md) | ⭐⭐⭐⭐ | 闭包 / 捕获 / std::function / 泛型 lambda |
| C21 | [线程与 std::thread](./C21-精通-C++-线程与-thread.md) | ⭐⭐⭐⭐ | thread/jthread / join/detach / thread_local |
| C22 | [互斥与同步](./C22-精通-C++-互斥与同步.md) | ⭐⭐⭐⭐⭐ | mutex / lock_guard / condition_variable / 死锁 |
| C23 | [原子与内存模型](./C23-精通-C++-原子与内存模型.md) | ⭐⭐⭐⭐⭐ | atomic / memory_order / happens-before / 无锁 |
| C24 | [future / async / promise](./C24-精通-C++-future-async-promise.md) | ⭐⭐⭐⭐ | future / async / promise / packaged_task |
| C25 | [协程](./C25-精通-C++-协程.md) | ⭐⭐⭐⭐⭐ | co_await/yield/return / promise_type / 生成器 |
| C26 | [异常与错误处理](./C26-精通-C++-异常与错误处理.md) | ⭐⭐⭐⭐ | 异常安全 / noexcept / 栈展开 / std::expected |
| C27 | [constexpr 编译期编程](./C27-精通-C++-constexpr-编译期编程.md) | ⭐⭐⭐⭐⭐ | constexpr/consteval/constinit / 编译期容器 |
| C28 | [Modules](./C28-精通-C++-Modules.md) | ⭐⭐⭐⭐ | 模块 vs 头文件 / import/export / 编译加速 |
| C29 | [字符串与 format](./C29-精通-C++-字符串与-format.md) | ⭐⭐⭐ | string/string_view / std::format/print / Unicode |
| C30 | [内存管理进阶](./C30-精通-C++-内存管理进阶.md) | ⭐⭐⭐⭐⭐ | 分配器 / placement new / 对齐 / pmr / 内存池 |
| C31 | [性能优化](./C31-精通-C++-性能优化.md) | ⭐⭐⭐⭐⭐ | 缓存友好 / 拷贝消除/RVO / 内联 / UB / profiling |
| C32 | [C++23/26 新特性与现状](./C32-精通-C++-C++23-26-新特性与现状.md) | ⭐⭐⭐⭐ | expected/print/deducing this / 反射/契约 / 生态 |

---

## 🗺️ 按模块组织

### 🟢 语言基础与构建（C01–C06）
编译模型/ODR → 类型系统/auto → 移动语义 → 完美转发 → RAII/智能指针 → 拷贝控制/五法则。**地基，必学。**

### 🔵 对象模型与 OOP（C07–C10）
对象内存布局 → 虚函数/vtable → 继承/CRTP → 运算符重载/`<=>`。理解 C++ 多态的底层。

### 🟡 模板与泛型（C11–C15）
模板基础 → 可变参数 → 元编程/SFINAE → Concepts → 模板进阶。C++ 泛型威力的核心。

### 🔴 STL（C16–C20）
容器内部 → 迭代器 → 算法 → Ranges → lambda。标准库的正确高效使用。

### 🟣 并发（C21–C25）
线程 → 互斥同步 → 原子/内存模型 → future/async → 协程。多核时代的必修。

### 🟠 现代特性与工程化（C26–C32）
异常 → constexpr → Modules → 字符串/format → 内存管理 → 性能 → C++23/26 现状。

---

## 🎯 学习路径

- **路径 A · 从零到精通（4–6 月）**：按 C01→C32 顺序，每周 2–3 篇。
- **路径 B · 面试突击（4 周）**：C03 移动 / C05 智能指针 / C06 五法则 / C08 虚函数 / C11 模板 / C16 容器 / C22 锁 / C23 原子 / C26 异常 / C31 性能。
- **路径 C · 性能特化（6 周）**：C03/C04 移动转发 → C07 布局 → C16 容器 → C23 原子 → C27 constexpr → C30 内存 → C31 性能。
- **路径 D · 现代 C++ 升级（3 周）**：C14 Concepts / C19 Ranges / C25 协程 / C28 Modules / C29 format / C32 新特性。

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 编译 C++23 | `g++ -std=c++23 -Wall -Wextra -O2 a.cpp` |
| 看汇编 | `g++ -S -O2 a.cpp` 或 [godbolt.org](https://godbolt.org) |
| 反 mangle | `nm a.out \| c++filt` |
| AddressSanitizer | `g++ -fsanitize=address,undefined a.cpp` |
| 构建 | `cmake -B build -G Ninja && cmake --build build` |
| 静态分析 | `clang-tidy a.cpp` |
| 格式化 | `clang-format -i a.cpp` |
| 性能剖析 | `perf record ./a.out` / `valgrind --tool=callgrind` |

---

## ✅ 完读检查清单

- [ ] 解释 ODR、为什么 inline/template 能放头文件
- [ ] 说清 lvalue/rvalue，std::move 只是 cast
- [ ] 区分转发引用与右值引用、引用折叠规则
- [ ] 用 unique_ptr/shared_ptr/weak_ptr 管理资源（零法则）
- [ ] 写出正确的五法则（含 noexcept 移动）
- [ ] 解释 vtable/vptr、为什么基类析构要 virtual
- [ ] 用 Concepts 约束模板、用 SFINAE/if constexpr 做编译期分支
- [ ] 说出 vector/unordered_map 的迭代器失效规则
- [ ] 用 memory_order 写一个正确的无锁计数器
- [ ] 用协程写一个生成器
- [ ] 区分 constexpr/consteval/constinit
- [ ] 列举 C++23 关键新特性（expected/print/deducing this）

---

## 🆕 版本特性索引

| 版本 | 关键特性 | 出现在 |
|---|---|---|
| **C++11** | 移动语义/auto/lambda/智能指针/线程/右值引用 | C03/C05/C20/C21 |
| **C++14** | 泛型 lambda / 变量模板 / `make_unique` | C20/C05 |
| **C++17** | CTAD / 结构化绑定 / `if constexpr` / `string_view` / `optional`/`variant` / 并行算法 | C02/C13/C18/C29 |
| **C++20** | **Concepts / Ranges / 协程 / Modules** / `<=>` / `std::format` / `constinit`/`consteval` | C14/C19/C25/C28/C10/C29/C27 |
| **C++23** | `std::expected` / `std::print` / deducing this / `mdspan` / `flat_map` / ranges 增强 | C26/C29/C32 |
| **C++26**（进展） | 反射 / 契约 / `std::execution` / 模式匹配 | C32 |

权威来源：[cppreference](https://en.cppreference.com/) · [ISO C++](https://isocpp.org/) · [godbolt 在线编译](https://godbolt.org/)

---

## 📋 配套资源

- **可视化路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：[QUIZ.md](./QUIZ.md)（每章 5 题，共 160 题）
- **生态库选型地图**：[Cpp-生态库选型地图.md](./libraries/Cpp-生态库选型地图.md)（2026 实战精选：构建/包管理、测试、JSON、网络、协程、分配器等场景的有主见选型）
- **必读**：《Effective Modern C++》（Scott Meyers）、《C++ Concurrency in Action》（Anthony Williams）、[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/)

---

> 🔁 反馈：C++ 的深度在"机制背后的为什么"。每段代码都丢进 [godbolt](https://godbolt.org) 看汇编，理解会深一个量级。
