# 计算机基础四大件 · 总目录

> 对标 **计算机考研 408**（数据结构 / 计算机组成原理 / 操作系统 / 计算机网络）的完整知识点覆盖
> 每篇含概念定义、原理推导、图示、代码实现、408 真题考点、陷阱清单与练习题
> 数据结构每章附 **力扣（LeetCode）实战题单**，把考点直接接到编码练习上
>
> **📅 内容基准：2026 年 7 月**。经典理论为主，涉及现代实现处标注真实工程现状。

---

## 📚 四科总览

| 科目 | 篇数 | 目录 | 408 分值 | 重点 |
|---|---|---|---|---|
| 🔷 **数据结构** | 8 | [data-structure/](./data-structure/INDEX.md) | 45 分 | 线性表 / 树 / 图 / 查找 / 排序 |
| 🔶 **计算机组成原理** | 8 | [computer-organization/](./computer-organization/INDEX.md) | 45 分 | 数据表示 / 存储 / 指令 / CPU / 总线 / I/O |
| 🟩 **操作系统** | 9 | [operating-system/](./operating-system/INDEX.md) | 35 分 | 进程 / 同步 / 死锁 / 内存 / 文件 / I/O |
| 🟦 **计算机网络** | 6 | [computer-network/](./computer-network/INDEX.md) | 25 分 | 分层 / 链路 / 网络 / 传输 / 应用 |

合计 **31 篇，已全部完成**。408 总分 150（含 30 分选择题跨科），本系列按考纲章节组织，不按分值裁剪。

每科子目录下配 `INDEX.md`（该科总目录与模块划分）、`ROADMAP.md`（Mermaid 路线图）、`QUIZ.md`（每章 5 题 + 参考答案）。

---

## 🔷 数据结构（DS01-DS08）

> 每章末尾附 **力扣实战题单**，按"必刷 / 进阶 / 挑战"三档分级。

| # | 课程 | 408 考纲章节 | 难度 |
|---|---|---|---|
| DS01 | [数据结构绪论与复杂度分析](./data-structure/DS01-精通-绪论与复杂度分析.md) | 第 1 章 | ⭐⭐ |
| DS02 | [线性表：顺序表与链表](./data-structure/DS02-精通-线性表.md) | 第 2 章 | ⭐⭐⭐ |
| DS03 | [栈、队列与数组](./data-structure/DS03-精通-栈队列与数组.md) | 第 3 章 | ⭐⭐⭐ |
| DS04 | [串与 KMP 模式匹配](./data-structure/DS04-精通-串与KMP.md) | 第 4 章 | ⭐⭐⭐⭐ |
| DS05 | [树与二叉树](./data-structure/DS05-精通-树与二叉树.md) | 第 5 章 | ⭐⭐⭐⭐ |
| DS06 | [图](./data-structure/DS06-精通-图.md) | 第 6 章 | ⭐⭐⭐⭐⭐ |
| DS07 | [查找：BST / AVL / B树 / 散列](./data-structure/DS07-精通-查找.md) | 第 7 章 | ⭐⭐⭐⭐⭐ |
| DS08 | [排序：内部排序与外部排序](./data-structure/DS08-精通-排序.md) | 第 8 章 | ⭐⭐⭐⭐ |

---

## 🔶 计算机组成原理（CO01-CO08）

> 计算题占比最高的一科：进制转换、IEEE 754、Cache 命中率、流水线加速比、总线带宽。详见 [computer-organization/INDEX.md](./computer-organization/INDEX.md)。

| # | 课程 | 408 考纲章节 | 难度 |
|---|---|---|---|
| CO01 | [计算机系统概述与性能指标](./computer-organization/CO01-精通-计算机系统概述与性能指标.md) | 第 1 章 | ⭐⭐ |
| CO02 | [数据的表示：定点、浮点与 IEEE 754](./computer-organization/CO02-精通-数据的表示与-IEEE-754.md) | 第 2 章 | ⭐⭐⭐⭐ |
| CO03 | [运算方法与运算器（ALU / 加法器 / 乘除法）](./computer-organization/CO03-精通-运算方法与运算器.md) | 第 2 章 | ⭐⭐⭐⭐ |
| CO04 | [存储系统：层次结构、SRAM/DRAM 与主存](./computer-organization/CO04-精通-存储系统.md) | 第 3 章 | ⭐⭐⭐ |
| CO05 | [Cache 与虚拟存储器](./computer-organization/CO05-精通-Cache-与虚拟存储器.md) | 第 3 章 | ⭐⭐⭐⭐⭐ |
| CO06 | [指令系统：格式、寻址方式与 CISC/RISC](./computer-organization/CO06-精通-指令系统.md) | 第 4 章 | ⭐⭐⭐⭐ |
| CO07 | [中央处理器：数据通路、控制器与流水线](./computer-organization/CO07-精通-中央处理器.md) | 第 5 章 | ⭐⭐⭐⭐⭐ |
| CO08 | [总线与输入输出系统（中断 / DMA）](./computer-organization/CO08-精通-总线与输入输出系统.md) | 第 6-7 章 | ⭐⭐⭐⭐ |

---

## 🟩 操作系统（OS01-OS09）

> 概念密集、大题集中在调度计算、银行家算法、地址变换、页面置换、磁盘调度。详见 [operating-system/INDEX.md](./operating-system/INDEX.md)。

| # | 课程 | 408 考纲章节 | 难度 |
|---|---|---|---|
| OS01 | [操作系统概述与运行环境（内核态 / 中断 / 系统调用）](./operating-system/OS01-精通-操作系统概述与运行环境.md) | 第 1 章 | ⭐⭐⭐ |
| OS02 | [进程与线程](./operating-system/OS02-精通-进程与线程.md) | 第 2 章 | ⭐⭐⭐⭐ |
| OS03 | [CPU 调度算法](./operating-system/OS03-精通-CPU-调度.md) | 第 2 章 | ⭐⭐⭐ |
| OS04 | [同步与互斥：信号量、管程与经典同步问题](./operating-system/OS04-精通-同步与互斥.md) | 第 2 章 | ⭐⭐⭐⭐⭐ |
| OS05 | [死锁：预防、避免、检测与解除](./operating-system/OS05-精通-死锁.md) | 第 2 章 | ⭐⭐⭐⭐ |
| OS06 | [内存管理：连续分配、分页、分段与段页式](./operating-system/OS06-精通-内存管理.md) | 第 3 章 | ⭐⭐⭐⭐ |
| OS07 | [虚拟内存：请求分页与页面置换算法](./operating-system/OS07-精通-虚拟内存.md) | 第 3 章 | ⭐⭐⭐⭐⭐ |
| OS08 | [文件管理](./operating-system/OS08-精通-文件管理.md) | 第 4 章 | ⭐⭐⭐ |
| OS09 | [磁盘与 I/O 管理](./operating-system/OS09-精通-磁盘与-IO-管理.md) | 第 4-5 章 | ⭐⭐⭐⭐ |

---

## 🟦 计算机网络（CN01-CN06）

> 分值最低但最好拿分：性能指标计算、CRC / 海明码、子网划分、TCP 拥塞窗口是四大计算题。详见 [computer-network/INDEX.md](./computer-network/INDEX.md)。

| # | 课程 | 408 考纲章节 | 难度 |
|---|---|---|---|
| CN01 | [网络体系结构与性能指标](./computer-network/CN01-精通-网络体系结构与性能指标.md) | 第 1 章 | ⭐⭐⭐ |
| CN02 | [物理层与数据链路层](./computer-network/CN02-精通-物理层与数据链路层.md) | 第 2-3 章 | ⭐⭐⭐⭐ |
| CN03 | [局域网与介质访问控制（CSMA/CD、以太网、VLAN）](./computer-network/CN03-精通-局域网与介质访问控制.md) | 第 3 章 | ⭐⭐⭐⭐ |
| CN04 | [网络层：IP、子网划分、路由算法](./computer-network/CN04-精通-网络层.md) | 第 4 章 | ⭐⭐⭐⭐⭐ |
| CN05 | [传输层：UDP、TCP 与拥塞控制](./computer-network/CN05-精通-传输层.md) | 第 5 章 | ⭐⭐⭐⭐⭐ |
| CN06 | [应用层：DNS、FTP、Email、HTTP](./computer-network/CN06-精通-应用层.md) | 第 6 章 | ⭐⭐⭐ |

---

## 🔗 与仓库其他专题的关系

这四科是**理论基座**，仓库里已有的专题是**工程落地**。刻意做了区分，不重复：

| 本系列（理论 / 408 视角） | 已有专题（工程视角） |
|---|---|
| DS05 树、DS06 图、DS08 排序 | [algorithm/](../algorithm/INDEX.md) 20 篇 —— **刷题技巧**视角 |
| OS02 进程线程、OS03 调度、OS07 虚拟内存 | [linux/](../linux/INDEX.md) 20 篇 —— **Linux 内核实现**视角 |
| CN04 IP 与路由、CN05 TCP 拥塞控制 | [backend/B01](../backend/B01-精通互联网工作原理.md) 应用层视角、[linux/L11-L13](../linux/INDEX.md) 协议栈实现 |
| CN06 HTTP | [backend/B02](../backend/B02-精通-HTTP-语义.md) HTTP 语义、[backend/B26](../backend/B26-精通-Nginx.md) Nginx |
| CO05 Cache 与内存屏障 | [golang/G20](../golang/G20-精通-Go-内存管理.md) GC、[golang/G05](../golang/G05-精通-Go-Struct-内存布局与嵌入.md) 内存对齐 |
| CO07 流水线与分支预测 | [golang/G21](../golang/G21-精通-Go-逃逸分析.md) 逃逸分析、[golang/G22](../golang/G22-精通-Go-pprof-性能剖析.md) 性能剖析 |

**读法建议**：

- **考研 408** → 只读本系列，按 DS → CO → OS → CN 顺序
- **面试补基础** → 本系列的"高频考点"小节 + [algorithm/](../algorithm/INDEX.md) 刷题
- **工程师补理论** → 先读本系列对应章，再回到 `linux/`、`backend/` 看真实实现

---

## 🎯 学习路径

### 路径 A：408 完整备考（4-6 个月）

```
DS01-DS08（数据结构，45 分，最耗时）  ~8 周
   ↓ 边学边刷力扣题单
CO01-CO08（组成原理，45 分，计算题多）~6 周
   ↓
OS01-OS09（操作系统，35 分）          ~5 周
   ↓
CN01-CN06（计算机网络，25 分，最好拿分）~3 周
   ↓
真题 + 错题复盘                        ~4 周
```

### 路径 B：后端工程师补基础（1-2 个月）

只挑对工程有直接回报的：

- **CO05 Cache** —— 理解 cache line、伪共享，直接影响并发代码性能
- **OS04 同步与互斥** —— 信号量/管程是所有并发原语的祖先
- **OS07 页面置换** —— LRU/Clock 就是缓存淘汰策略的源头
- **CN05 TCP 拥塞控制** —— 排查线上网络问题的必备
- **DS07 B树/B+树** —— 数据库索引的底层结构
- **DS08 外部排序** —— 大数据 sort/merge 的理论原型

### 路径 C：只补数据结构（1 个月）

DS01 → DS02 → DS03 → DS05 → DS07 → DS08 → DS06 → DS04

每章配套力扣题单**必刷档全做完**，进阶档挑一半。

---

## 📋 配套资源

> ✅ **当前进度**：31 篇正文全部完成，四科的 `INDEX / ROADMAP / QUIZ` 已配齐。

| 科目 | 总目录 | 路线图 | 测验 |
|---|---|---|---|
| 数据结构 | [INDEX](./data-structure/INDEX.md) | [ROADMAP](./data-structure/ROADMAP.md) | [QUIZ](./data-structure/QUIZ.md)（40 题） |
| 计算机组成原理 | [INDEX](./computer-organization/INDEX.md) | [ROADMAP](./computer-organization/ROADMAP.md) | [QUIZ](./computer-organization/QUIZ.md)（40 题） |
| 操作系统 | [INDEX](./operating-system/INDEX.md) | [ROADMAP](./operating-system/ROADMAP.md) | [QUIZ](./operating-system/QUIZ.md)（45 题） |
| 计算机网络 | [INDEX](./computer-network/INDEX.md) | [ROADMAP](./computer-network/ROADMAP.md) | [QUIZ](./computer-network/QUIZ.md)（30 题） |

外部资源：

- **408 考纲**：[中国研究生招生信息网](https://yz.chsi.com.cn/)（每年 9 月发布）
- **力扣题库**：[leetcode.cn](https://leetcode.cn/problemset/)
- **数据结构可视化**：[VisuAlgo](https://visualgo.net/zh)、[USFCA 可视化](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)
- **CPU 与缓存**：[What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)

---

> 🔁 反馈：发现错误或建议改进
