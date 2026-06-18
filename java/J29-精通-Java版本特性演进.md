# 精通 Java 版本特性演进

> Java 从 8 到 21/25 演进巨大：Lambda/Stream 带来函数式、Record/Sealed 简化建模、虚拟线程颠覆并发。面试常问"你用过哪些新特性""虚拟线程是什么"。本篇梳理各 LTS 版本的关键特性，串起整个 Java 专题。
>
> **📅 基准：2026 年 6 月。LTS：Java 8 / 11 / 17 / 21 / 25（2025-09）。**

---

## 一、Java 版本与 LTS

Java 自 9 起改为**每 6 个月发布一个版本**，每隔几年出一个 **LTS（长期支持）**版本：

| 版本 | 年份 | 地位 | 标志特性 |
|---|---|---|---|
| **Java 8** | 2014 | 经典 LTS，存量巨大 | Lambda、Stream、新日期 API |
| **Java 11** | 2018 | LTS | 模块化稳定、var、HttpClient |
| **Java 17** | 2021 | LTS，当前主流 | Record、Sealed、Pattern Matching |
| **Java 21** | 2023 | LTS | **虚拟线程**、结构化并发、分代 ZGC |
| **Java 25** | 2025 | 最新 LTS | 虚拟线程/模式匹配/结构化并发的稳定与增强 |

生产主流是 **Java 17 / 21**，老系统大量还在 **Java 8**。选 LTS（稳定 + 长期支持）而非中间版本。

---

## 二、Java 8 革命

Java 8 是里程碑，带来函数式编程：

- **Lambda 表达式**：`(a, b) -> a + b`，函数作为参数，简化匿名内部类。
- **Stream API**：声明式集合处理（filter/map/reduce）。
- **函数式接口**：`Function`、`Predicate`、`Consumer`、`Supplier`（只有一个抽象方法的接口，配 `@FunctionalInterface`）。
- **Optional**：优雅处理 null，减少 NPE（[J05](./J05-精通-异常处理与最佳实践.md)）。
- **接口默认方法**：`default` 方法，让接口能加方法而不破坏实现类。
- **新日期时间 API（java.time）**：`LocalDate`/`LocalDateTime`/`DateTimeFormatter`——**不可变、线程安全**，取代有线程安全问题的 `Date`/`SimpleDateFormat`（[J14](./J14-精通-ThreadLocal.md) 提到的痛点解决了）。

Java 8 至今仍是存量最大的版本，这些特性是日常开发标配。

---

## 三、Lambda 与函数式接口

Lambda 让"行为作为参数传递"成为可能：

```java
// 匿名内部类 → Lambda
list.sort((a, b) -> a.compareTo(b));
Runnable r = () -> System.out.println("run");

// 方法引用（Lambda 的简写）
list.forEach(System.out::println);
```

- **函数式接口**：只有一个抽象方法的接口，Lambda 就是它的实例。常用：`Function<T,R>`（转换）、`Predicate<T>`（判断）、`Consumer<T>`（消费）、`Supplier<T>`（提供）。
- Lambda 底层用 `invokedynamic` 实现（不是匿名内部类的语法糖，见 [J21 字节码](./J21-精通-字节码与执行引擎.md)），延迟到运行期生成，更高效。
- **注意**：Lambda 捕获的局部变量必须是 final 或 effectively final（事实不变）。

---

## 四、Stream API

Stream 把集合处理变成声明式的"流水线"：

```java
List<String> result = users.stream()
    .filter(u -> u.getAge() > 18)        // 中间操作：过滤
    .map(User::getName)                  // 中间操作：转换
    .sorted()                            // 中间操作：排序
    .collect(Collectors.toList());       // 终端操作：收集
```

要点：
- **中间操作**（filter/map/sorted）**惰性**——不触发执行，只构建流水线。
- **终端操作**（collect/forEach/reduce/count）才触发实际计算。
- **并行流** `parallelStream()`：底层用 ForkJoinPool 并行处理，适合 CPU 密集大数据量（小数据反而更慢，且注意线程安全）。
- Stream 让代码更声明式、可读，但调试比循环难、且有一定开销（热点循环未必比 for 快）。

---

## 五、Java 9-16 重要特性

| 版本 | 特性 |
|---|---|
| 9 | **模块系统（JPMS）**、集合工厂 `List.of()`、私有接口方法 |
| 10 | **`var`** 局部变量类型推断 |
| 11 | 标准 `HttpClient`、String 增强方法 |
| 14 | **switch 表达式**（`->` 写法，有返回值）、**Helpful NPE** |
| 15 | **文本块（Text Blocks）** `"""..."""`（多行字符串） |
| 16 | **`instanceof` 模式匹配** `if (o instanceof String s)` |

```java
// switch 表达式（Java 14+）
String r = switch (day) {
    case MON, TUE -> "工作日";
    case SAT, SUN -> "周末";
    default -> "未知";
};

// instanceof 模式匹配（Java 16+）——省去强转
if (obj instanceof String s) { System.out.println(s.length()); }

// 文本块（Java 15+）
String json = """
    { "name": "Alice" }
    """;
```

这些特性减少样板、提升可读性，是现代 Java 的日常写法。

---

## 六、Java 17 LTS

Java 17 是当前主流 LTS，关键特性是更好的**数据建模**：

- **Record（记录类）**：不可变数据载体，一行定义带 构造器/getter/equals/hashCode/toString 的类：
  ```java
  record Point(int x, int y) {}  // 自动生成全部样板
  ```
  适合 DTO、值对象，天然不可变（[J03](./J03-精通-String与不可变设计.md) 的不可变思想），线程安全。
- **Sealed Class（密封类）**：限制哪些类能继承/实现，`sealed interface Shape permits Circle, Square {}`——配合 switch 模式匹配做穷尽判断。
- **Pattern Matching（模式匹配）**：instanceof 模式 + switch 模式逐步增强。

Record + Sealed + Pattern Matching 让 Java 在"代数数据类型/不可变建模"上更现代（向 Kotlin/Scala 靠拢）。

---

## 七、Java 21 LTS

Java 21 是重磅 LTS，**虚拟线程**是颠覆性特性：

- **虚拟线程（Virtual Threads）**：极轻量的线程（JVM 调度，可创建数百万个）。阻塞时自动让出底层载体线程（平台线程），让"**一请求一线程**"的简单阻塞写法也能支撑海量并发 IO——颠覆了"必须用线程池/NIO/响应式"的传统（见 [J11 线程池](./J11-精通-线程池.md)、[J28 IO](./J28-精通-Java-IO与NIO.md)）。
  ```java
  // 每个任务一个虚拟线程，不再受线程池大小限制
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
      executor.submit(() -> handleRequest());
  }
  ```
  注意：虚拟线程适合 **IO 密集**；CPU 密集仍用平台线程池。虚拟线程中长时间阻塞**优先用 ReentrantLock 而非 synchronized**（避免 pin 载体线程，见 [J08](./J08-精通-synchronized与volatile.md)/[J10](./J10-精通-显式锁与锁优化.md)）；少用 ThreadLocal（海量虚拟线程下占内存，用 ScopedValue，见 [J14](./J14-精通-ThreadLocal.md)）。
- **结构化并发（Structured Concurrency）**：`StructuredTaskScope` 把一组并发子任务作为一个单元管理（统一等待、取消、异常传播），让并发代码更可靠（演进中）。
- **分代 ZGC**：提升 ZGC 吞吐（见 [J18](./J18-精通-垃圾收集器G1与ZGC.md)）。
- **switch 模式匹配 + record 解构**：`case Point(int x, int y) -> ...` 直接解构。
- **Sequenced Collections**：统一有序集合首尾访问 API（见 [J01](./J01-精通-集合框架与List源码.md)）。

---

## 八、版本选型与迁移

- **选 LTS**：生产用 LTS（8/11/17/21/25），有长期安全更新；不要用中间的非 LTS 版本跑生产。
- **当前推荐**：新项目用 **Java 21**（虚拟线程 + 现代特性）；保守用 **17**；老系统 8 仍大量存在。
- **迁移注意点**：
  - **Java 8 → 17/21**：`javax` → `jakarta`（Spring 6/Boot 3 要求，见 [J25](./J25-精通-Spring-MVC.md)/[J26](./J26-精通-SpringBoot自动配置.md)）；移除的 API（如部分 JEE 模块）；GC 默认变 G1；反射对 JDK 内部类受模块限制。
  - 依赖升级：老库可能不兼容新 JDK（模块化、移除的 API）。
- **GraalVM 原生镜像**：追求极速启动/低内存（Serverless）可选，但反射/动态代理受限（见 [J16](./J16-精通-类加载与双亲委派.md)/[J21](./J21-精通-字节码与执行引擎.md)）。

---

## 陷阱清单

- **生产用非 LTS 版本**：中间版本支持期短，生产应选 LTS。
- **还在用 Date/SimpleDateFormat**：SimpleDateFormat 非线程安全（[J14](./J14-精通-ThreadLocal.md)）。用 java.time（LocalDateTime/DateTimeFormatter，不可变线程安全）。
- **滥用 parallelStream**：小数据量并行反而慢；共享可变状态线程不安全；默认共用 ForkJoinPool 公共池可能互相影响。
- **Lambda 捕获可变变量**：捕获的局部变量必须 effectively final。
- **Stream 当万能替代 for**：热点性能敏感循环，Stream 未必比 for 快、且难调试。
- **虚拟线程跑 CPU 密集任务**：不增加并行度，CPU 密集仍用平台线程池。
- **虚拟线程里用 synchronized 长阻塞**：可能 pin 载体线程，用 ReentrantLock。
- **以为升级 JDK 只是换版本号**：javax→jakarta、移除的 API、模块限制等要逐一处理。

---

## 2026 现状

- **Java 17/21 是生产主流**，Java 8 仍有大量存量（迁移中）；Java 25（2025-09）为最新 LTS。
- **虚拟线程改变并发范式**：IO 密集服务越来越多用"阻塞式写法 + 虚拟线程"替代复杂的响应式/线程池调参（见 [J11](./J11-精通-线程池.md)、[J28](./J28-精通-Java-IO与NIO.md)）。
- **Record/Sealed/Pattern Matching 成现代写法**：DTO 用 record、配合 sealed + switch 模式匹配做类型穷尽，代码更简洁安全。
- **Spring Boot 3 基线 Java 17**：用 Spring Boot 3 就必须 17+，且 jakarta 命名空间。
- **GraalVM 原生镜像 + AOT**：Serverless/快启动场景升温，Spring 6 AOT 支持成熟。
- **Project Valhalla/Leyden/Babylon** 等长期项目推进中（值类型、启动优化、异构计算），关注后续 LTS。

---

## 练习题

1. Java 为什么要区分 LTS 版本？生产环境应该如何选择 Java 版本？

<details><summary>参考答案</summary>

Java 自 9 起改为每 6 个月发布一个新版本（快速迭代、尽早交付特性），但如果每个版本都用于生产会带来支持和稳定性问题——非 LTS（中间）版本的官方支持周期很短（通常只到下一个版本发布），不会长期提供安全补丁和 bug 修复。因此 Oracle/OpenJDK 每隔几年（最初 3 年、后调整）指定一个 **LTS（Long-Term Support，长期支持）版本**（如 8、11、17、21、25），对它提供多年的安全更新和支持。区分 LTS 的目的就是：让追求快速特性的开发者能用最新版尝鲜，让生产系统能选择一个长期稳定、持续获得安全补丁的版本。**生产选择原则**：①**只用 LTS 版本**跑生产（8/11/17/21/25），避免用支持期短的中间版本；②新项目推荐用较新的 LTS（如 Java 21，可享受虚拟线程等现代特性和更好的 GC），对稳定性极度保守的可选 17；③老系统大量仍在 Java 8，迁移要评估成本（javax→jakarta、移除的 API 等）；④要用 Spring Boot 3 则必须 Java 17+。总之"选 LTS、按需取较新的、避开非 LTS"是生产选型的基本准则。

</details>

2. Java 8 带来了哪些革命性特性？java.time 相比旧的 Date/SimpleDateFormat 好在哪？

<details><summary>参考答案</summary>

Java 8 的革命性特性主要是引入了**函数式编程**和一系列现代化 API：①**Lambda 表达式**——可以把"行为/函数"作为参数传递（如 `(a,b)->a+b`），大幅简化匿名内部类的样板；②**Stream API**——以声明式、链式的方式处理集合（filter/map/reduce/collect），代码更简洁可读；③**函数式接口**（Function、Predicate、Consumer、Supplier 等，配 @FunctionalInterface）——Lambda 的类型载体；④**Optional**——优雅地表达"可能为空"，减少 NPE 和繁琐的 null 判断；⑤**接口默认方法（default）**——允许接口在不破坏已有实现类的前提下新增带实现的方法；⑥**全新的日期时间 API（java.time）**。**java.time 相比 Date/SimpleDateFormat 的好处**：①**不可变 + 线程安全**——LocalDate/LocalDateTime/Instant 等都是不可变对象，可安全地在多线程间共享；而旧的 java.util.Date 是可变的，尤其 **SimpleDateFormat 不是线程安全的**（内部有可变状态），多线程共享会出现解析/格式化错乱（这正是过去要用 ThreadLocal 包裹 SimpleDateFormat 的原因，见 J14），DateTimeFormatter 则本身线程安全、无此问题；②**API 设计清晰合理**——明确区分日期（LocalDate）、时间（LocalTime）、日期时间（LocalDateTime）、时刻（Instant）、时区（ZonedDateTime）等概念，方法语义清晰、支持流畅的链式操作和丰富的日期运算；而旧 Date 的 API 设计混乱（月份从 0 开始、年份偏移 1900 等坑）、Calendar 笨重。所以现代 Java 开发应统一用 java.time 取代 Date/Calendar/SimpleDateFormat。

</details>

3. Stream 的中间操作和终端操作有什么区别？什么是惰性求值？

<details><summary>参考答案</summary>

Stream 的操作分两类。**中间操作（Intermediate）**：如 filter、map、sorted、distinct、limit 等，它们的返回值仍是一个 Stream，用于构建处理流水线；中间操作是**惰性（lazy）的**——调用它们时**并不会真正执行任何计算**，只是记录下"要做什么"，把操作串联成一条流水线。**终端操作（Terminal）**：如 collect、forEach、reduce、count、findFirst、anyMatch 等，它们会触发整个流水线的实际执行并产生最终结果（一个集合、一个值或副作用），终端操作执行后 Stream 就被消费、不能再用。**惰性求值（lazy evaluation）**：指中间操作不会立即执行，只有当遇到终端操作时，才会从数据源开始、按流水线对元素进行处理。惰性带来两个好处：①**可优化**——多个中间操作可以被融合、对每个元素一次性走完整条流水线（而不是每个中间操作都完整遍历一遍集合再传给下一个），减少遍历次数和中间集合；②**支持短路**——像 limit、findFirst、anyMatch 这类操作可以在满足条件后提前结束，不必处理全部元素（甚至能处理无限流，如 `Stream.iterate(...).limit(n)`）。例如 `stream.filter(...).map(...).findFirst()`，惰性求值下可能只处理到第一个满足条件的元素就返回，而不会先 filter 全部再 map 全部。理解"中间操作惰性、终端操作触发执行"能避免"写了 stream.filter 却没有终端操作导致什么都没发生"的常见错误。

</details>

4. Record（记录类）是什么？它适合什么场景？

<details><summary>参考答案</summary>

Record（记录类，Java 16 正式、17 LTS 纳入）是一种特殊的、简洁的**不可变数据载体类**。用 `record Point(int x, int y) {}` 一行声明，编译器就会**自动生成**：①私有 final 字段（x、y）；②全参构造方法（规范构造器）；③每个字段的访问器方法（`x()`、`y()`，注意是字段名而非 getX）；④基于所有字段的 `equals()` 和 `hashCode()`；⑤合理的 `toString()`。也就是说，它把过去写一个不可变数据类需要的大量样板代码（构造器、getter、equals、hashCode、toString）全部自动生成，极大减少模板。Record 的特点：**字段都是 final、对象不可变**（一旦创建不能修改），因此天然线程安全（呼应不可变设计的好处，见 J03）；它隐式继承 java.lang.Record、不能再继承别的类（但可实现接口）；可以自定义/紧凑构造器做校验。**适合的场景**：①**DTO / VO**（数据传输对象、视图对象）——接口出入参的纯数据结构；②**值对象（Value Object）**——如坐标、金额、范围等表示"一组值"的不可变对象；③多返回值的载体、map 的复合 key、配置项等；④配合 sealed 接口 + switch 模式匹配（`case Point(int x, int y) -> ...` 解构）做代数数据类型建模。**不适合**需要可变状态、复杂行为或要继承类的实体（如 JPA 实体通常不用 record）。总之 Record 让"定义不可变数据类"变得一行搞定，是现代 Java 简化数据建模的利器。

</details>

5. 虚拟线程（Virtual Threads）是什么？它解决了什么问题？使用时要注意什么？

<details><summary>参考答案</summary>

虚拟线程是 Java 21 正式引入的轻量级线程，由 JVM（而非操作系统）管理和调度。它非常轻量——栈按需增长、创建和切换成本极低，可以轻松创建数百万个，而传统的平台线程（对应一个 OS 线程）受内存（每个约 1MB 栈）和系统资源限制，通常只能有几千个。虚拟线程运行时会被挂载到少量"载体线程"（平台线程）上执行，**当它执行阻塞操作（如网络/文件 IO、阻塞队列、sleep）时，JVM 会自动把它从载体线程卸载、让载体线程去运行其他虚拟线程**，等阻塞结束再恢复。**解决的问题**：传统高并发 IO 服务面临两难——用 BIO"一连接一线程"写法简单但线程数受限扛不住高并发（C10K，见 J28）；用 NIO/响应式能高并发但编程复杂、心智负担重。虚拟线程让开发者可以用最直观的"**一请求一线程、顺序阻塞**"的简单代码风格，却能支撑**海量并发 IO**（百万级），因为阻塞的虚拟线程不会占用宝贵的 OS 线程。这极大简化了高并发 IO 服务的开发（见 J11、J28）。**注意事项**：①虚拟线程只对 **IO 密集**有益，**CPU 密集任务不要用**——它不增加 CPU 并行度，CPU 密集仍应用平台线程池（线程数≈核数）；②**长时间阻塞时优先用 ReentrantLock 而非 synchronized**——早期在 synchronized 块内阻塞会"pin（钉住）"载体线程导致无法卸载（后续版本改善中），而 ReentrantLock 等不会（见 J08/J10）；③**慎用 ThreadLocal**——海量虚拟线程各自持有 ThreadLocal 副本会占用大量内存，推荐改用 ScopedValue（见 J14）；④不要池化虚拟线程（它本就廉价，直接一任务一个即可，用 newVirtualThreadPerTaskExecutor）；⑤底层网络框架（Netty）仍是 NIO，虚拟线程改变的是应用层并发模型。

</details>

6. 从 Java 8 升级到 Java 17/21 需要注意哪些主要问题？

<details><summary>参考答案</summary>

主要注意点：①**`javax` → `jakarta` 命名空间迁移**——这是最大的坑之一。从 Java EE 到 Jakarta EE，相关 API 的包名从 javax.\* 改为 jakarta.\*（如 javax.servlet → jakarta.servlet、javax.persistence → jakarta.persistence）。如果升级到 Spring 6 / Spring Boot 3（它们要求 Java 17+），就必须把代码和依赖里的 javax 相关导入改成 jakarta（见 J25/J26），这往往牵涉大量改动和第三方库版本升级。②**被移除/废弃的 API**——Java 在演进中移除了一些模块和 API（如 Java 11 移除了 Java EE/CORBA 模块 java.xml.bind(JAXB)、java.activation 等，需要单独引入依赖；一些过时方法被删），升级后编译可能报找不到类，需要替换或补依赖。③**模块系统（JPMS）的强封装影响反射**——Java 9+ 模块化后，对 JDK 内部 API（如 sun.misc.Unsafe）和未 open 的包做深度反射会受限/报警告甚至失败，依赖反射"破坏封装"的库（旧版 CGLIB、某些序列化框架）可能出问题，需升级这些库或加 --add-opens。④**默认 GC 变化**——Java 9+ 默认收集器从 Parallel 变为 G1（见 J18），GC 行为和调优参数可能要重新评估；一些旧 GC 参数（CMS 相关）已失效（CMS 在 14 被移除）。⑤**第三方依赖兼容性**——老版本的库可能不支持新 JDK（字节码版本、模块、移除的 API），需要升级到兼容版本，要做全面的依赖升级和回归测试。⑥**编译/运行参数与工具链**——构建工具、插件、JaCoCo/Lombok 等也要升到支持新 JDK 的版本。建议升级策略：先升 JDK 跑通（不改代码、处理编译/运行错误），再视需要升级框架（如上 Spring Boot 3 一并处理 jakarta 迁移），全程配合充分的自动化测试回归，分步进行而非一步到位。

</details>
