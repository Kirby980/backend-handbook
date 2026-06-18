# 精通 JVM 调优与故障排查

> 前面讲了内存结构（[J15](./J15-精通-JVM运行时内存结构.md)）、GC 算法（[J17](./J17-精通-垃圾回收算法.md)）、收集器（[J18](./J18-精通-垃圾收集器G1与ZGC.md)），这一篇是**实战**：线上 CPU 飙到 100% 怎么定位？OOM 了怎么找泄漏？频繁 Full GC 怎么排查？掌握这套排查方法论和工具，是高级 Java 工程师的硬实力。
>
> **📅 基准：Java 17/21。**

---

## 一、调优目标与原则

调优的本质是在**吞吐、延迟、内存**三者间权衡，不是盲目调参数。

原则：
- **先监控、再定位、后调优**——没有数据支撑的调优是猜。靠 GC 日志、监控（Prometheus/JFR）找到真正瓶颈。
- **不要过早优化**——默认参数（G1）对多数应用已够好；大多数"GC 问题"根源是代码（内存泄漏、大对象、不合理缓存），而非参数。
- **一次只改一个变量**——改完压测对比，否则分不清哪个起了作用。
- **目标量化**——如 "p99 停顿 < 100ms、Full GC 每天 < 1 次"，有目标才知道调到什么程度。

---

## 二、核心 JVM 参数

```bash
# 堆
-Xms4g -Xmx4g                          # 初始/最大堆（设相等，见 J18）
-Xmn2g                                 # 新生代大小
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=256m  # 元空间

# GC
-XX:+UseG1GC -XX:MaxGCPauseMillis=200  # G1 + 停顿目标
-XX:+UseZGC -XX:+ZGenerational         # 或分代 ZGC（JDK21）

# 故障排查（生产必加）
-XX:+HeapDumpOnOutOfMemoryError        # OOM 自动 dump
-XX:HeapDumpPath=/path/dump.hprof      # dump 路径
-Xlog:gc*:file=/path/gc.log:time       # GC 日志（JDK9+ 统一格式）

# 容器
-XX:MaxRAMPercentage=75.0              # 按容器内存比例设堆（云原生）
```

**生产铁律**：一定要加 `-XX:+HeapDumpOnOutOfMemoryError`——OOM 时自动保存现场，否则事后无从排查。

---

## 三、OOM 的几种类型与定位

| OOM 类型 | 含义 | 常见原因 |
|---|---|---|
| `Java heap space` | 堆放不下 | 内存泄漏、大对象、缓存无限增长、查询数据过多 |
| `GC overhead limit exceeded` | GC 占用 98%+ 时间却回收 < 2% | 堆快满、频繁 Full GC 仍救不回来 |
| `Metaspace` | 元空间满 | 动态生成大量类、类加载器泄漏 |
| `Direct buffer memory` | 直接内存满 | NIO/Netty DirectBuffer 泄漏（堆 dump 看不到，见 [J15](./J15-精通-JVM运行时内存结构.md)） |
| `unable to create native thread` | 线程数超限 | 线程泄漏、线程池配置不当 |

不同 OOM 排查思路不同：堆 OOM 靠 heap dump 分析；元空间 OOM 查类加载器；直接内存/线程 OOM 要看堆外和系统层。

---

## 四、内存泄漏排查

堆 OOM / 内存持续上涨的标准排查流程：

```bash
# 1. 拿到堆快照（OOM 时自动 dump，或手动）
jmap -dump:live,format=b,file=heap.hprof <pid>

# 2. 用 MAT / VisualVM / JProfiler 分析 heap.hprof
```

用 **MAT（Memory Analyzer Tool）** 分析：
- **Histogram**：看哪类对象实例最多、占用最大。
- **Dominator Tree（支配树）**：看哪些对象"支配"了最多内存（谁持有了大量内存出不去）。
- **Leak Suspects 报告**：MAT 自动分析可疑泄漏点。
- **GC Roots 路径**：对可疑对象看它到 GC Roots 的引用链——找到"谁一直引用着它导致回收不掉"。

典型泄漏：static 集合无限增长、ThreadLocal 未 remove（[J14](./J14-精通-ThreadLocal.md)）、监听器未注销、连接未关闭、缓存无淘汰。

---

## 五、CPU 飙高排查

线上 CPU 100% 的经典四步定位（高频面试题）：

```bash
# 1. 找到 CPU 高的 Java 进程
top                          # 找到占 CPU 最高的进程 PID

# 2. 找到该进程里 CPU 高的线程
top -Hp <pid>               # -H 显示线程，找到占 CPU 高的线程 TID

# 3. 线程 ID 转成 16 进制（jstack 里是 16 进制）
printf "%x\n" <tid>         # 如 12345 → 3039

# 4. dump 线程栈，搜这个 16 进制 ID
jstack <pid> | grep -A 30 <16进制tid>   # 看这个线程在执行什么
```

定位到具体线程在执行的代码，常见原因：死循环、正则回溯、频繁 GC（GC 线程占 CPU）、序列化/计算热点。

如果是 **GC 导致的 CPU 高**：jstack 看到 GC 线程忙，再结合 GC 日志确认是否频繁 Full GC，转去查内存问题。

---

## 六、常用工具

| 工具 | 用途 |
|---|---|
| `jps` | 列出 Java 进程 |
| `jstat -gcutil <pid> 1000` | 实时看 GC 情况（各区占用、GC 次数/耗时） |
| `jstack <pid>` | 线程栈快照（查死锁、CPU 高、阻塞） |
| `jmap -dump` / `jmap -histo` | 堆转储 / 对象统计 |
| `jcmd <pid>` | 多功能（GC、dump、JFR 启停，官方推荐统一入口） |
| `jinfo` | 查看/修改 JVM 参数 |
| **Arthas** | 阿里开源，线上动态诊断神器（watch 方法入参/耗时、trace 调用链、热更新、火焰图），不停机排查 |
| **JFR（Java Flight Recorder）** | JDK 内置低开销性能记录器，生产可常开，事后用 JMC 分析 |
| MAT / VisualVM / JProfiler | 堆分析、性能剖析 |

**Arthas 和 JFR 是 2026 的主力**：Arthas 不停机在线诊断（看方法耗时、慢调用、热点），JFR 低开销持续记录、事后回溯。

---

## 七、GC 日志分析

GC 日志（`-Xlog:gc*`）是诊断 GC 问题的金矿，关注：

- **GC 频率**：Young GC 太频繁 → 新生代太小或对象创建过快；Full GC 频繁 → 严重问题（内存泄漏/堆太小/大对象）。
- **GC 停顿时长**：单次 STW 多久，是否超 SLA。
- **回收效果**：每次 GC 后堆占用降了多少；若 Full GC 后老年代几乎没降 → 内存泄漏（存活对象一直在涨）。
- **吞吐**：GC 时间占总运行时间的比例（健康一般 < 5%）。

用 **GCEasy**、**GCViewer** 等工具可视化分析 GC 日志，快速发现停顿和频率异常。

---

## 八、典型案例

| 现象 | 排查路径 | 常见根因 |
|---|---|---|
| **频繁 Full GC** | GC 日志看频率 + Full GC 后老年代是否下降 → jmap dump → MAT | 内存泄漏（存活对象涨）、堆/新生代太小导致过早晋升、大对象、元空间 |
| **OOM: heap space** | 自动 dump → MAT 支配树 + GC Roots 路径 | static 集合、缓存无淘汰、ThreadLocal 泄漏、大结果集 |
| **CPU 100%** | top -Hp → jstack 定位线程 | 死循环、正则回溯、频繁 GC、热点计算 |
| **死锁** | jstack 自动检测 "Found Java-level deadlock" | 多锁加锁顺序不一致（见 [J10](./J10-精通-显式锁与锁优化.md)） |
| **响应变慢** | JFR/Arthas trace 方法耗时 + GC 日志 | 慢查询、锁竞争、GC 停顿、外部依赖慢 |
| **元空间 OOM** | 看类加载数（jstat -class）→ 查动态类来源 | 动态代理/字节码生成失控、类加载器泄漏 |

---

## 陷阱清单

- **不加 HeapDumpOnOutOfMemoryError**：OOM 后没有现场，无法事后排查。生产必加。
- **盲目调 GC 参数**：不先定位就改参数，越调越乱。先监控找根因（多数是代码问题）。
- **Full GC 后老年代不降还以为正常**：这是内存泄漏的典型信号。
- **直接内存/元空间 OOM 去翻堆 dump**：堆 dump 看不到堆外和元空间，要用 NMT/jstat -class。
- **jstack 找线程不转 16 进制**：top 的线程 ID 是十进制，jstack 里是 16 进制，要转换。
- **线上用重型工具 attach**：jmap dump 大堆会 STW 卡顿，慎在高峰期对大堆 dump；优先用低开销的 JFR/Arthas。
- **一次改多个参数**：分不清效果。一次一个变量 + 压测对比。

---

## 2026 现状

- **Arthas + JFR 是线上诊断主力**：Arthas 不停机看方法耗时/调用链/热点、热修复；JFR 低开销常开记录，JMC 事后分析，二者覆盖绝大多数线上问题。
- **可观测一体化**：JVM 指标（GC、堆、线程）通过 Micrometer 暴露到 Prometheus/Grafana，结合 OpenTelemetry 链路追踪（见 [microservices 可观测](../microservices/18-精通-可观测三支柱.md)），异常自动告警。
- **容器化排查**：云原生下 JVM 跑在容器，要正确设置容器内存感知（-XX:MaxRAMPercentage）避免 OOMKilled；排查需进容器或用 sidecar/ephemeral container。
- **统一 GC 日志**：JDK 9+ 的 `-Xlog` 统一日志框架，配合 GCEasy 在线分析。
- **AI 辅助诊断**：开始出现用大模型分析 GC 日志/线程 dump、给出诊断建议的工具。

---

## 练习题

1. 线上一个 Java 服务 CPU 飙到 100%，请描述你的完整定位步骤。

<details><summary>参考答案</summary>

经典四步定位：①**找进程**——用 `top`（或 `top -c`）找到占用 CPU 最高的 Java 进程，记下它的 PID。②**找线程**——用 `top -Hp <pid>`（-H 显示线程级 CPU）找到该进程内占用 CPU 最高的一个或几个线程，记下线程的 TID（十进制）。③**转换线程 ID**——jstack 输出里线程的 nid 是 16 进制，所以用 `printf "%x\n" <tid>` 把十进制 TID 转成 16 进制。④**dump 线程栈定位代码**——用 `jstack <pid>` 导出线程快照，搜索上一步的 16 进制线程 ID（`jstack <pid> | grep -A 30 <16进制id>`），找到该线程当前正在执行的方法调用栈，从而定位到具体是哪段代码在狂占 CPU。常见根因：死循环、正则灾难性回溯、频繁 GC（此时占 CPU 的是 GC 线程，需结合 GC 日志确认并转向内存排查）、热点计算/序列化等。补充：也可用 Arthas 的 `thread -n 3`（直接显示最忙的几个线程及栈）或生成火焰图，更快定位，且不需要手动转进制。定位到代码后再针对性修复（修死循环、优化算法、改正则、解决频繁 GC 的内存问题等）。

</details>

2. 发生堆 OOM 后，如何排查是哪里发生了内存泄漏？

<details><summary>参考答案</summary>

标准流程：①**获取堆快照（heap dump）**——生产环境应提前配置 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...`，OOM 时自动 dump 出 .hprof 文件保存现场；或在内存持续上涨时用 `jmap -dump:live,format=b,file=heap.hprof <pid>` 手动获取（注意大堆 dump 会 STW，谨慎在高峰操作）。②**用分析工具打开**——用 MAT（Memory Analyzer Tool）、VisualVM 或 JProfiler 加载 hprof。③**分析定位**——看 **Histogram**（按类统计实例数和占用大小，找出异常多/大的对象类型）；看 **Dominator Tree（支配树）**（找出"支配"了大量内存的对象，即谁持有了大片内存导致无法回收）；看 MAT 的 **Leak Suspects** 自动报告；对可疑大对象查它到 **GC Roots 的引用链**，搞清楚是"谁一直强引用着它"导致回收不掉（这是找泄漏根因的关键）。④**结合代码定位根因**——常见泄漏：静态集合（static Map/List）无限增长、缓存无淘汰策略、ThreadLocal 用完没 remove（线程池下，见 J14）、监听器/回调未注销、连接/流未关闭、一次查询加载过多数据等。⑤**验证修复**——修复后观察内存曲线和 Full GC 后老年代是否能正常回落。辅助判断：如果每次 Full GC 后老年代占用持续不降反升，基本可确定是内存泄漏（存活对象一直累积）。

</details>

3. 列举你常用的 JVM 故障排查工具及其用途。

<details><summary>参考答案</summary>

①**jps**——列出当前所有 Java 进程及其 PID。②**jstat**——实时监控 JVM 统计信息，如 `jstat -gcutil <pid> 1000` 每秒输出各内存区使用率、GC 次数和耗时，用于快速观察 GC 行为；`jstat -class` 看类加载数（排查元空间问题）。③**jstack**——打印线程栈快照，用于排查 CPU 飙高（定位忙线程）、死锁（能自动检测并提示 "Found one Java-level deadlock"）、线程阻塞。④**jmap**——`jmap -dump` 导出堆快照供 MAT 分析、`jmap -histo` 看对象直方图统计。⑤**jcmd**——官方推荐的多功能统一诊断入口，可做 GC、堆 dump、启停 JFR、查看参数等。⑥**jinfo**——查看和动态修改部分 JVM 参数。⑦**Arthas**（阿里开源）——线上动态诊断神器，能 `watch` 方法的入参/返回/耗时、`trace` 调用链找慢点、`thread` 看忙线程、生成火焰图、甚至热更新代码，全程不停机，是线上排查主力。⑧**JFR（Java Flight Recorder）**——JDK 内置的低开销飞行记录器，可在生产长期开启持续记录运行数据，事后用 JMC（Mission Control）分析性能、GC、锁、热点。⑨**MAT / VisualVM / JProfiler**——堆 dump 分析和性能剖析的图形化工具。⑩**GCEasy/GCViewer**——可视化分析 GC 日志。实际排查中，jstack/jstat/jmap 是基础三件套，Arthas 和 JFR 是现代低开销在线诊断的首选。

</details>

4. 如何判断频繁 Full GC 是不是内存泄漏导致的？

<details><summary>参考答案</summary>

关键看**每次 Full GC 之后老年代（或整堆）的占用是否能有效回落**。正常情况下，Full GC 应该能回收掉大量不再使用的对象，使老年代占用明显下降、回到一个较低水平；如果观察 GC 日志发现：Full GC 越来越频繁，但**每次 Full GC 后老年代占用几乎不降、甚至持续走高**（存活对象总量随时间不断累积、回收不掉），这就是**内存泄漏的典型信号**——说明有对象一直被强引用着无法回收，越积越多，逼得 JVM 不断 Full GC 试图腾空间却徒劳，最终走向 OOM。排查步骤：①开启并分析 GC 日志（`-Xlog:gc*`），用 GCEasy 等观察 Full GC 频率和每次回收后的堆占用趋势曲线；②若确认老年代持续上涨，用 jmap dump 堆 + MAT 分析支配树和 GC Roots 引用链，找出一直累积、回收不掉的对象及其持有者；③定位代码（静态集合、缓存无淘汰、ThreadLocal 未清等）。对比：如果 Full GC 后老年代能正常大幅回落，那频繁 Full GC 更可能是堆/新生代太小、对象晋升过快、大对象过多或参数不当（调优参数即可），而非泄漏。所以"Full GC 后内存降不下来"是区分"泄漏"与"参数问题"的核心判据。

</details>

5. 为什么生产环境一定要配置 -XX:+HeapDumpOnOutOfMemoryError？

<details><summary>参考答案</summary>

因为内存问题（OOM、内存泄漏）往往是**偶发、难复现**的——它可能在运行很久后、在特定数据量或流量下才触发，而 OOM 发生的那一刻，堆里的对象分布、引用关系正是定位泄漏根因的关键现场。如果没有配置自动 dump，OOM 发生后进程可能崩溃/被重启，现场（当时的堆内容）就**永久丢失**了，事后只能靠日志猜测、很难定位真正原因，甚至要等下次再 OOM。配置 `-XX:+HeapDumpOnOutOfMemoryError`（配合 `-XX:HeapDumpPath` 指定保存路径）后，JVM 会在抛出 OutOfMemoryError 的那一刻**自动把整个堆转储成 .hprof 文件**，完整保存崩溃现场；之后就能用 MAT 等工具离线分析这个 dump，通过支配树、GC Roots 引用链精确找出"是哪类对象占满了堆、被谁一直引用着"，从而定位泄漏代码。它几乎零日常开销（只在 OOM 时才 dump 一次），却能把"无法排查"变成"有据可查"，是生产环境的必备保命参数。注意 dump 文件可能很大（接近堆大小），要确保磁盘空间充足、路径可写。

</details>

6. JVM 调优应该遵循哪些基本原则？为什么说"多数 GC 问题的根源在代码而非参数"？

<details><summary>参考答案</summary>

基本原则：①**先监控、再定位、后调优**——用 GC 日志、jstat、JFR、监控指标等先收集数据、找到真正的瓶颈，不要凭感觉猜着调参；②**不要过早优化**——现代默认收集器（G1）和默认参数对绝大多数应用已经够好，没有明确性能问题就不必动；③**目标量化**——先定义清晰的优化目标（如 p99 停顿 < 100ms、Full GC 频率、吞吐占比），才能衡量调优是否达标、何时停止；④**一次只改一个变量并压测对比**——同时改多个参数会让你分不清是哪个起作用甚至互相干扰。"多数 GC 问题根源在代码而非参数"的原因：很多看似 GC/内存的问题，本质是代码层面的缺陷——比如内存泄漏（static 集合无限增长、ThreadLocal 未 remove、缓存无淘汰、连接未关闭）导致存活对象不断累积、频繁 Full GC 最终 OOM；创建大量临时大对象、不合理的大集合、一次查询加载过多数据导致频繁 GC；这些靠调 GC 参数（加大堆、换收集器）只能暂时缓解、治标不治本，问题会再次出现。真正的解法是定位并修复代码（修泄漏、加缓存淘汰、分页查询、减少大对象）。所以排查时应先怀疑代码、用 dump/profiling 找根因，而不是一上来就调 JVM 参数——参数调优是在代码合理的前提下做的"锦上添花"，不能替代修复代码缺陷。

</details>
