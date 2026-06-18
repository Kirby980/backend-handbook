# Java 后端 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月（Java 21/25 LTS）**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Java 进阶]) --> M1[模块1: 语言核心]
    M1 --> J01[J01 集合/List]
    M1 --> J02[J02 HashMap]
    M1 --> J03[J03 String]
    M1 --> J04[J04 泛型]
    M1 --> J05[J05 异常]
    M1 --> J06[J06 反射/代理]

    J06 --> M2[模块2: 并发编程]
    M2 --> J07[J07 JMM]
    M2 --> J08[J08 synchronized]
    M2 --> J09[J09 AQS]
    M2 --> J10[J10 显式锁]
    M2 --> J11[J11 线程池]
    M2 --> J12[J12 并发容器]
    M2 --> J13[J13 CAS]
    M2 --> J14[J14 ThreadLocal]

    M2 --> M3[模块3: JVM]
    M3 --> J15[J15 内存结构]
    M3 --> J16[J16 类加载]
    M3 --> J17[J17 GC算法]
    M3 --> J18[J18 收集器]
    M3 --> J19[J19 调优排查]
    M3 --> J20[J20 对象布局]
    M3 --> J21[J21 字节码]

    M3 --> M4[模块4: 框架]
    M4 --> J22[J22 IOC]
    M4 --> J23[J23 AOP]
    M4 --> J24[J24 事务/循环依赖]
    M4 --> J25[J25 Spring MVC]
    M4 --> J26[J26 Spring Boot]
    M4 --> J27[J27 MyBatis]

    M4 --> M5[模块5: IO与演进]
    M5 --> J28[J28 IO/NIO]
    M5 --> J29[J29 版本特性]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#e1bee7
    style J02 fill:#ffcdd2
    style J09 fill:#ffcdd2
    style J18 fill:#ffcdd2
    style J24 fill:#ffcdd2
```

---

## 🧵 并发知识体系（模块二全景）

```mermaid
graph TB
    JMM[JMM 内存模型<br>可见性/有序性/原子性] --> V[volatile<br>可见性+禁重排]
    JMM --> S[synchronized<br>原子性+可见性]
    S --> Upgrade[锁升级<br>偏向→轻量→重量]
    JMM --> CAS[CAS 无锁]
    CAS --> Atomic[原子类<br>AtomicXxx/LongAdder]
    CAS --> AQS[AQS<br>state+CLH队列]
    AQS --> Lock[ReentrantLock<br>读写锁]
    AQS --> Tools[CountDownLatch<br>Semaphore]
    Lock --> Pool[线程池<br>ThreadPoolExecutor]
    Atomic --> CC[并发容器<br>ConcurrentHashMap]
    CAS --> CC

    style JMM fill:#fff3e0
    style AQS fill:#bbdefb
    style CAS fill:#c8e6c9
```

---

## 🔒 synchronized 锁升级（J08）

```mermaid
stateDiagram-v2
    [*] --> 无锁
    无锁 --> 偏向锁: 首个线程进入(对象头记线程ID)
    偏向锁 --> 轻量级锁: 出现竞争(CAS自旋)
    轻量级锁 --> 重量级锁: 自旋失败/竞争激烈(进入monitor阻塞)
    重量级锁 --> [*]
    note right of 重量级锁
        重量级锁依赖 OS 互斥量
        线程阻塞/唤醒有上下文切换开销
    end note
```

---

## 🧩 AQS 独占锁加锁流程（J09）

```mermaid
flowchart TD
    Try[tryAcquire<br>CAS 改 state] -->|成功| Got[拿到锁]
    Try -->|失败| Enq[加入 CLH 队列尾部]
    Enq --> Park[前驱是 head 再试一次<br>否则 LockSupport.park 阻塞]
    Park --> Wake[前驱释放锁时<br>unpark 唤醒后继]
    Wake --> Try2[被唤醒后重试 tryAcquire]
    Try2 -->|成功| Got
    Try2 -->|失败| Park

    style Try fill:#c8e6c9
    style Enq fill:#fff9c4
    style Park fill:#ffccbc
```

---

## 🧠 JVM 运行时内存结构（J15）

```mermaid
flowchart TB
    subgraph 线程共享
        Heap[堆 Heap<br>新生代Eden/S0/S1 + 老年代<br>对象实例]
        Meta[方法区/元空间 Metaspace<br>类元数据/常量池/静态变量]
    end
    subgraph 线程私有
        Stack[虚拟机栈<br>栈帧:局部变量表/操作数栈]
        Native[本地方法栈]
        PC[程序计数器]
    end
    Direct[直接内存<br>堆外 NIO]

    style Heap fill:#ffcdd2
    style Meta fill:#fff9c4
    style Stack fill:#bbdefb
```

---

## ♻️ 线程池执行流程（J11）

```mermaid
flowchart TD
    Submit[提交任务] --> C1{核心线程<br>已满?}
    C1 -->|否| Core[创建核心线程执行]
    C1 -->|是| C2{工作队列<br>已满?}
    C2 -->|否| Queue[入队等待]
    C2 -->|是| C3{达到最大<br>线程数?}
    C3 -->|否| Max[创建非核心线程执行]
    C3 -->|是| Reject[执行拒绝策略]

    style Core fill:#c8e6c9
    style Queue fill:#fff9c4
    style Reject fill:#ffcdd2
```

---

## 🌱 Spring Bean 生命周期（J22）

```mermaid
flowchart LR
    Def[BeanDefinition] --> Inst[实例化<br>构造]
    Inst --> Pop[属性填充<br>依赖注入]
    Pop --> Aware[Aware 回调]
    Aware --> Before[BeanPostProcessor<br>前置]
    Before --> Init[初始化<br>@PostConstruct/afterPropertiesSet]
    Init --> After[BeanPostProcessor<br>后置 → AOP 代理]
    After --> Use[就绪/使用]
    Use --> Destroy[销毁<br>@PreDestroy]

    style Pop fill:#fff9c4
    style After fill:#ffccbc
```

---

## 🔄 Spring 循环依赖三级缓存（J24）

```mermaid
flowchart TB
    A[创建 A] --> A1[A 实例化后<br>提前暴露到三级缓存<br>singletonFactories]
    A1 --> A2[A 注入 B → 去创建 B]
    A2 --> B1[B 实例化<br>注入 A]
    B1 --> Get[从三级缓存拿到 A 的早期引用<br>升到二级缓存]
    Get --> Bdone[B 创建完成]
    Bdone --> Adone[A 继续完成注入]

    style A1 fill:#fff9c4
    style Get fill:#c8e6c9
```

---

## 🎓 学习顺序建议

```mermaid
graph LR
    A[J01-J06<br>语言核心] --> B[J07-J14<br>并发]
    B --> C[J15-J21<br>JVM]
    C --> D[J22-J27<br>Spring]
    D --> E[J28-J29<br>IO/演进]

    style A fill:#c8e6c9
    style B fill:#bbdefb
    style C fill:#fff9c4
    style D fill:#ffccbc
```

按顺序读 = 完整 Java 后端体系；面试突击 = 看 [INDEX.md](./INDEX.md) 的「路径 A」。
