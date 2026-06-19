# Java 后端深度课程 · 总目录

> 面向后端 Java 工程师面试与生产的系统进阶课程，共 **32 篇**深度长文。
> 每篇约 10000-15000 字，含底层原理、源码剖析、代码示例、生产实践、陷阱清单与练习题。
>
> **📅 内容基准：2026 年 6 月** —— Java 21 LTS（虚拟线程 / 记录类 / 密封类 / 模式匹配 / 分代 ZGC）主流、Java 17 LTS 仍广泛在用、Spring Boot 3.x（Spring 6 / Jakarta EE 9+ / GraalVM 原生镜像 / Observability）、Java 25 LTS（2025-09 发布）新特性跟进；JMM、G1/ZGC、AQS 等核心原理稳定。
>
> 本专题与 [Go 专题](../golang/INDEX.md) 互为镜像——同样的并发、内存、工程化主题，讲 Java 的实现与生态。是 OfferPilot 题库 Java 赛道（KR2.1）的内容底座。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| J01 | [精通集合框架与 List 源码](./J01-精通-集合框架与List源码.md) | ⭐⭐⭐ | ArrayList / LinkedList / 扩容 / fail-fast / Iterator |
| J02 | [精通 HashMap 与 ConcurrentHashMap](./J02-精通-HashMap与ConcurrentHashMap.md) | ⭐⭐⭐⭐⭐ | 扰动 / 扩容 / 红黑树 / 分段锁 / CAS+synchronized |
| J03 | [精通 String 与不可变设计](./J03-精通-String与不可变设计.md) | ⭐⭐⭐ | 不可变 / 常量池 / intern / StringBuilder |
| J04 | [精通泛型与类型擦除](./J04-精通-泛型与类型擦除.md) | ⭐⭐⭐⭐ | 类型擦除 / 通配符 / PECS / 桥方法 |
| J05 | [精通异常处理与最佳实践](./J05-精通-异常处理与最佳实践.md) | ⭐⭐⭐ | 受检/非受检 / try-with-resources / 异常链 |
| J06 | [精通反射、注解与动态代理](./J06-精通-反射注解与动态代理.md) | ⭐⭐⭐⭐ | Class / 动态代理 / JDK vs CGLIB / 注解处理 |
| J07 | [精通 JMM 与 happens-before](./J07-精通-JMM与happens-before.md) | ⭐⭐⭐⭐⭐ | 内存模型 / 可见性 / 有序性 / 重排序 |
| J08 | [精通 synchronized 与 volatile](./J08-精通-synchronized与volatile.md) | ⭐⭐⭐⭐⭐ | 对象头 / 锁升级 / 偏向锁 / 内存屏障 |
| J09 | [精通 AQS 原理](./J09-精通-AQS原理.md) | ⭐⭐⭐⭐⭐ | state / CLH 队列 / 独占共享 / Condition |
| J10 | [精通显式锁与锁优化](./J10-精通-显式锁与锁优化.md) | ⭐⭐⭐⭐ | ReentrantLock / 读写锁 / 公平性 / 死锁 |
| J11 | [精通线程池 ThreadPoolExecutor](./J11-精通-线程池.md) | ⭐⭐⭐⭐⭐ | 核心参数 / 拒绝策略 / 队列 / 调优 |
| J12 | [精通并发容器](./J12-精通-并发容器.md) | ⭐⭐⭐⭐ | CopyOnWrite / BlockingQueue / ConcurrentHashMap |
| J13 | [精通 CAS 与原子类](./J13-精通-CAS与原子类.md) | ⭐⭐⭐⭐ | CAS / ABA / LongAdder / Unsafe |
| J14 | [精通 ThreadLocal](./J14-精通-ThreadLocal.md) | ⭐⭐⭐⭐ | ThreadLocalMap / 弱引用 / 内存泄漏 / 传递 |
| J15 | [精通 JVM 运行时内存结构](./J15-精通-JVM运行时内存结构.md) | ⭐⭐⭐⭐ | 堆 / 栈 / 方法区 / 元空间 / 直接内存 |
| J16 | [精通类加载与双亲委派](./J16-精通-类加载与双亲委派.md) | ⭐⭐⭐⭐ | 加载过程 / 双亲委派 / 打破 / SPI |
| J17 | [精通垃圾回收算法与可达性](./J17-精通-垃圾回收算法.md) | ⭐⭐⭐⭐ | 可达性分析 / 三色标记 / 分代 / 引用 |
| J18 | [精通垃圾收集器 G1 与 ZGC](./J18-精通-垃圾收集器G1与ZGC.md) | ⭐⭐⭐⭐⭐ | G1 / ZGC / CMS / Region / 停顿 |
| J19 | [精通 JVM 调优与故障排查](./J19-精通-JVM调优与故障排查.md) | ⭐⭐⭐⭐⭐ | 参数 / OOM / jstack / Arthas / MAT |
| J20 | [精通对象内存布局与逃逸分析](./J20-精通-对象内存布局与逃逸分析.md) | ⭐⭐⭐⭐ | 对象头 / 压缩指针 / 逃逸分析 / 栈上分配 |
| J21 | [精通字节码与执行引擎](./J21-精通-字节码与执行引擎.md) | ⭐⭐⭐⭐ | 字节码 / 操作数栈 / JIT / 解释执行 |
| J22 | [精通 Spring IOC 容器](./J22-精通-Spring-IOC容器.md) | ⭐⭐⭐⭐⭐ | Bean 生命周期 / 依赖注入 / BeanFactory |
| J23 | [精通 Spring AOP](./J23-精通-Spring-AOP.md) | ⭐⭐⭐⭐ | 动态代理 / 切面 / 通知 / 失效场景 |
| J24 | [精通 Spring 事务与循环依赖](./J24-精通-Spring事务与循环依赖.md) | ⭐⭐⭐⭐⭐ | 传播级别 / 失效 / 三级缓存 |
| J25 | [精通 Spring MVC](./J25-精通-Spring-MVC.md) | ⭐⭐⭐⭐ | DispatcherServlet / 请求流程 / 参数绑定 / 拦截器 / REST |
| J26 | [精通 Spring Boot 自动配置](./J26-精通-SpringBoot自动配置.md) | ⭐⭐⭐⭐ | 自动装配 / starter / 条件注解 / SPI |
| J27 | [精通 MyBatis 原理](./J27-精通-MyBatis原理.md) | ⭐⭐⭐⭐ | SqlSession / 动态代理 / 一二级缓存 / 插件 |
| J28 | [精通 Java IO 与 NIO](./J28-精通-Java-IO与NIO.md) | ⭐⭐⭐⭐ | BIO/NIO/AIO / Channel / Selector / 零拷贝 |
| J29 | [精通 Java 版本特性演进](./J29-精通-Java版本特性演进.md) | ⭐⭐⭐⭐ | Lambda/Stream / Record / 虚拟线程 / 密封类 |
| J30 | [精通虚拟线程与结构化并发](./J30-精通-虚拟线程与结构化并发.md) | ⭐⭐⭐⭐⭐ | 虚拟线程 / 载体线程 / mount-unmount / pinning / ScopedValue / 结构化并发 |
| J31 | [精通函数式编程与 Stream](./J31-精通-函数式编程与Stream.md) | ⭐⭐⭐⭐ | Lambda/invokedynamic / 函数式接口 / 惰性与短路 / Collector / 并行流 / Optional |
| J32 | [精通 Java 测试](./J32-精通-Java测试.md) | ⭐⭐⭐⭐ | JUnit5 / Mockito / 参数化 / Testcontainers / 测试切片 / 覆盖率 |

---

## 🗺️ 按模块组织

### 🟢 模块一：语言核心与集合（J01-J06）

> Java 面试的"地基"——集合源码、字符串、泛型、反射，几乎每场必问。

- **J01 集合框架**：ArrayList/LinkedList 源码、扩容、fail-fast、Iterator
- **J02 HashMap**：扰动函数、扩容与树化、线程不安全表现、ConcurrentHashMap（JDK8 CAS+synchronized）⭐ 面试最高频
- **J03 String**：不可变设计、字符串常量池、intern、StringBuilder/StringBuffer
- **J04 泛型**：类型擦除、通配符与 PECS、桥方法
- **J05 异常**：受检 vs 非受检、try-with-resources、异常链与最佳实践
- **J06 反射注解**：Class、JDK 动态代理 vs CGLIB、注解处理（连 Spring）

### 🔵 模块二：并发编程（J07-J14）

> Java 后端面试的"重头戏"，也是和 Go 差异最大的地方。

- **J07 JMM**：内存模型、可见性/有序性/原子性、happens-before、重排序
- **J08 synchronized/volatile**：对象头、锁升级（偏向→轻量→重量）、内存屏障
- **J09 AQS**：state + CLH 队列、独占/共享、Condition——并发包的基石
- **J10 显式锁**：ReentrantLock、读写锁、StampedLock、公平性、死锁
- **J11 线程池**：七大参数、执行流程、拒绝策略、队列选择、调优
- **J12 并发容器**：ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue
- **J13 CAS**：CAS 原理、ABA、Unsafe、LongAdder vs AtomicLong
- **J14 ThreadLocal**：ThreadLocalMap、弱引用、内存泄漏、父子线程传递

### 🟡 模块三：JVM（J15-J21）

> 理解 JVM 是高级 Java 工程师的分水岭，调优和排查全靠它。

- **J15 内存结构**：堆/虚拟机栈/本地方法栈/程序计数器/方法区（元空间）/直接内存
- **J16 类加载**：加载-验证-准备-解析-初始化、双亲委派、打破双亲（Tomcat/SPI）
- **J17 GC 算法**：可达性分析、引用类型、标记-清除/复制/整理、三色标记、分代
- **J18 收集器**：CMS、G1（Region/SATB）、ZGC（染色指针/读屏障）、选型与停顿 ⭐
- **J19 调优排查**：核心参数、OOM 类型、jstack/jmap/jstat/Arthas/MAT 实战
- **J20 对象布局**：对象头/Mark Word、压缩指针、逃逸分析、栈上分配/标量替换
- **J21 字节码**：class 文件、操作数栈、解释执行、JIT（C1/C2）、内联

### 🟣 模块四：框架与生态（J22-J27）

> Spring 全家桶是 Java 后端的事实标准，面试必考其原理。

- **J22 Spring IOC**：Bean 生命周期、依赖注入、BeanFactory vs ApplicationContext、扩展点
- **J23 Spring AOP**：动态代理实现、切面/通知/切点、AOP 失效场景
- **J24 Spring 事务**：传播级别、隔离级别、事务失效的 N 种原因、循环依赖与三级缓存 ⭐
- **J25 Spring MVC**：DispatcherServlet 请求处理全流程、`@Controller`/`@RequestMapping`、参数绑定与返回值处理、拦截器、RESTful、全局异常处理 ⭐
- **J26 Spring Boot**：自动配置原理、starter、条件注解、`spring.factories`/SPI
- **J27 MyBatis**：SqlSession、Mapper 动态代理、一级/二级缓存、插件机制

### 🟠 模块五：IO 与语言演进（J28-J29）

- **J28 IO/NIO**：BIO/NIO/AIO、Channel/Buffer/Selector、多路复用、零拷贝、Netty 关联
- **J29 版本演进**：Java 8（Lambda/Stream/Optional）→ 11 → 17（Record/Sealed/Pattern）→ 21/25（虚拟线程/结构化并发）

### 🟤 模块六：现代并发、函数式与工程化（J30-J32）

> 三篇进阶补充，分别从「现代并发」「语言深度」「工程化」三个维度补齐前五个模块——与 Go 专题对照最强的部分。

- **J30 虚拟线程与结构化并发**：M:N 用户态线程、载体线程与 mount/unmount、pinning、ScopedValue、`StructuredTaskScope`、vs 线程池/响应式选型 ⭐ 2026 头号考点（深化 [J11](./J11-精通-线程池.md)/[J28](./J28-精通-Java-IO与NIO.md)）
- **J31 函数式编程与 Stream**：函数式接口体系、Lambda 的 `invokedynamic` 实现、Stream 惰性与短路、Collector 四件套与自定义、并行流陷阱、Optional 范式（深化 [J29](./J29-精通-Java版本特性演进.md)）
- **J32 Java 测试**：JUnit 5 架构与扩展模型、Mockito 的 stub/verify、参数化、Spring Boot 测试切片、Testcontainers、覆盖率与变异测试（对照 Go [G18](../golang/G18-精通-Go-测试.md)）

---

## 🎯 学习路径建议

### 路径 A：Java 后端面试突击（3-4 周）

```
J02 HashMap + J01 集合（必背源码）
   ↓
J07-J11 并发核心（JMM/锁/AQS/线程池）+ J30 虚拟线程（2026 头号考点）
   ↓
J15-J19 JVM（内存/类加载/GC/调优）
   ↓
J22-J24 Spring（IOC/AOP/事务循环依赖）+ J31 函数式/Stream
```

### 路径 B：并发专精（2 周）

```
J07 JMM → J08 synchronized/volatile → J13 CAS → J09 AQS
   ↓
J10 锁 → J11 线程池 → J12 并发容器 → J14 ThreadLocal
```

### 路径 C：JVM 专精（2-3 周）

```
J15 内存结构 → J20 对象布局 → J16 类加载
   ↓
J17 GC 算法 → J18 收集器 → J21 字节码
   ↓
J19 调优与故障排查（实战收尾）
```

### 路径 D：Spring 原理（2 周）

```
J06 反射/动态代理（前置）→ J22 IOC → J23 AOP
   ↓
J24 事务/循环依赖 → J25 Spring MVC → J26 Spring Boot → J27 MyBatis
```

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **综合测验**：见 [QUIZ.md](./QUIZ.md)（开放题，配套 Day 17.5 AI 补答案）
- **Go 镜像专题**：[golang/](../golang/INDEX.md)——对照学习两种语言的并发与内存模型
- **通用后端**：[backend/](../backend/INDEX.md)——语言无关的设计与协议

---

## 🔁 与 Go 专题的对照

Java 和 Go 在并发与内存模型上思路迥异，对照学习收获最大：

| 主题 | Java | Go |
|---|---|---|
| 并发模型 | 线程 + 锁 + 线程池 | goroutine + channel + GMP |
| 内存模型 | JMM / happens-before | Go MM / happens-before |
| 同步原语 | synchronized / AQS / Lock | sync.Mutex / channel |
| 轻量并发 | 虚拟线程 + 结构化并发（[J30](./J30-精通-虚拟线程与结构化并发.md)） | goroutine（[G11](../golang/G11-精通-Goroutines-与-GMP-调度.md)） |
| GC | 分代 / G1 / ZGC | 三色标记并发 GC（[G20](../golang/G20-精通-Go-内存管理.md)） |
| 泛型 | 类型擦除（J04） | 类型参数（[G09](../golang/G09-精通-Go-泛型-类型参数与约束.md)） |

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **J30 虚拟线程** | **虚拟线程（Virtual Threads）** Java 21 正式 GA，"一请求一线程"重新可行；配套**结构化并发 + ScopedValue**（Java 25 趋于转正）颠覆传统线程池/响应式思路。Java 24（JEP 491）起 `synchronized` 不再 pin |
| **J18 GC** | **分代 ZGC** Java 21 默认可用，亚毫秒停顿；G1 仍是默认收集器 |
| **J22-J27 Spring** | **Spring Boot 3.x / Spring 6**：基线 Java 17、Jakarta EE（`javax`→`jakarta`）、GraalVM 原生镜像、内置 Observability（Micrometer + OTel） |
| **J28 语言** | **Java 25 LTS**（2025-09）：跟进 records/sealed/pattern matching/虚拟线程/结构化并发的稳定化 |
| **J15 内存** | 元空间（Metaspace）早已取代永久代（PermGen，Java 8 移除），仍是高频考点 |
| **J06/J25 原生镜像** | GraalVM 原生镜像对反射/动态代理有限制，需配置元数据——影响 Spring 用法 |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 讲清 HashMap 扩容、树化、为什么线程不安全，以及 ConcurrentHashMap 如何保证并发安全
- [ ] 解释 JMM 的可见性/有序性问题，volatile 和 synchronized 各解决什么
- [ ] 手画 AQS 的 state + CLH 队列，说清 ReentrantLock 加锁流程
- [ ] 说出线程池七大参数、执行流程和拒绝策略，并能为场景配参数
- [ ] 画出 JVM 运行时内存结构，区分堆/栈/元空间
- [ ] 解释可达性分析、三色标记，对比 G1 和 ZGC 的停顿控制
- [ ] 用 jstack/jmap/Arthas 排查一次 CPU 飙高 / OOM / 死锁
- [ ] 讲清 Spring Bean 生命周期、AOP 实现、事务失效场景、循环依赖三级缓存
- [ ] 说清 Spring MVC 中 DispatcherServlet 处理一个请求的完整流程
- [ ] 说清虚拟线程相比平台线程的优势与适用场景
- [ ] 解释虚拟线程的载体线程、mount/unmount 与 pinning，以及结构化并发/ScopedValue 解决了什么
- [ ] 说清 Stream 的惰性求值与短路、Collector 四件套，以及并行流的适用与陷阱
- [ ] 用 JUnit 5 + Mockito + Testcontainers 写出分层（单元/切片/集成）测试

---

> 🔁 反馈：发现错误或建议改进，直接 PR 改 markdown。
