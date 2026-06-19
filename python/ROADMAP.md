# Python 后端 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月（Python 3.13 / 3.14，free-threading 转正）**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Python 进阶]) --> M1[模块1: 语言核心]
    M1 --> P01[P01 数据模型]
    M1 --> P02[P02 内置容器]
    M1 --> P04[P04 函数/作用域]
    M1 --> P05[P05 迭代器/生成器]
    M1 --> P06[P06 装饰器/上下文]
    M1 --> P08[P08 异常]

    M1 --> M2[模块2: OOP与类型]
    M2 --> P09[P09 类/属性查找]
    M2 --> P10[P10 继承/MRO]
    M2 --> P11[P11 描述符/property]
    M2 --> P12[P12 元类]
    M2 --> P14[P14 类型注解]

    M2 --> M3[模块3: CPython内幕]
    M3 --> P15[P15 执行/字节码]
    M3 --> P16[P16 内存/GC]
    M3 --> P17[P17 GIL]
    M3 --> P19[P19 性能剖析]

    M3 --> M4[模块4: 并发与异步]
    M4 --> P20[P20 线程]
    M4 --> P21[P21 进程]
    M4 --> P22[P22 asyncio]
    M4 --> P24[P24 选型/free-threading]

    M4 --> M5[模块5: 工程化]
    M5 --> P25[P25 包管理]
    M5 --> P26[P26 测试]

    M5 --> M6[模块6: Web/数据/演进]
    M6 --> P28[P28 Web/ASGI]
    M6 --> P29[P29 数据库/ORM]
    M6 --> P30[P30 版本演进]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#e1bee7
    style M6 fill:#d7ccc8
    style P11 fill:#ffcdd2
    style P16 fill:#ffcdd2
    style P17 fill:#ffcdd2
    style P22 fill:#ffcdd2
    style P24 fill:#ffcdd2
```

---

## 🐍 Python 并发全景（模块四 + GIL）

```mermaid
graph TB
    GIL[GIL 全局解释器锁<br>同一时刻只有一个线程执行字节码] --> T[多线程 threading]
    T --> TIO[✅ IO 密集<br>阻塞时释放 GIL]
    T --> TCPU[❌ CPU 密集<br>被 GIL 串行化]

    GIL --> P[多进程 multiprocessing]
    P --> PCPU[✅ CPU 并行<br>每进程独立解释器/GIL]
    P --> PIPC[代价: IPC/序列化/内存]

    GIL --> A[asyncio 协程]
    A --> AIO[✅ 单线程高并发 IO<br>事件循环 + await]
    A --> ABLOCK[❌ 阻塞调用<br>毁掉事件循环]

    GIL -.3.13+ 可选.-> FT[free-threading<br>no-GIL 构建]
    FT --> FTP[✅ 多线程真并行<br>免 IPC]

    style GIL fill:#ffcdd2
    style FT fill:#c8e6c9
    style AIO fill:#bbdefb
```

---

## 🧠 CPython 内存模型（P16）

```mermaid
flowchart TB
    subgraph 对象
        Obj[PyObject<br>ob_refcnt 引用计数<br>ob_type 类型指针]
    end
    Obj --> RC{引用计数}
    RC -->|减到 0| Free[立即回收]
    RC -->|循环引用<br>计数不为0| GC[分代垃圾回收器]
    GC --> G0[第0代<br>新对象,频繁扫描]
    G0 -->|存活晋升| G1[第1代]
    G1 -->|存活晋升| G2[第2代<br>老对象,少扫描]
    GC -.可达性分析.-> Free

    style Obj fill:#fff9c4
    style RC fill:#c8e6c9
    style GC fill:#ffccbc
```

---

## ⚡ asyncio 事件循环（P22）

```mermaid
flowchart TD
    Loop[事件循环 Event Loop] --> Ready{就绪队列<br>有任务?}
    Ready -->|是| Run[运行任务到下一个 await]
    Run --> Await{遇到 await<br>I/O?}
    Await -->|是,挂起| Reg[向 selector 注册 fd<br>让出控制权]
    Await -->|协程结束| Done[完成,设置结果]
    Reg --> Poll[epoll/kqueue 等待就绪]
    Poll -->|fd 就绪| Wake[唤醒对应任务入就绪队列]
    Wake --> Ready
    Done --> Ready

    Run -.阻塞调用 time.sleep/requests.-> Block[❌ 整个循环卡死<br>所有任务饿死]

    style Loop fill:#bbdefb
    style Reg fill:#c8e6c9
    style Block fill:#ffcdd2
```

---

## 🔎 属性查找顺序（P09/P11）

```mermaid
flowchart TD
    Get[obj.attr] --> DataDesc{类型中有<br>数据描述符?}
    DataDesc -->|是| UseDD[调用描述符 __get__]
    DataDesc -->|否| Inst{实例 __dict__<br>有?}
    Inst -->|是| UseInst[返回实例属性]
    Inst -->|否| NonData{类型中有<br>非数据描述符/类属性?}
    NonData -->|是| UseCls[返回/调用]
    NonData -->|否| GetAttr[__getattr__ 兜底<br>否则 AttributeError]

    style DataDesc fill:#ffcdd2
    style UseInst fill:#c8e6c9
```

> 优先级：**数据描述符 > 实例字典 > 非数据描述符/类属性 > `__getattr__`**。这条线解释了 `property`、方法绑定、`__slots__` 的行为（[P11](./P11-精通-Python-描述符与property.md)）。

---

## 🎓 学习顺序建议

```mermaid
graph LR
    A[P01-P08<br>语言核心] --> B[P09-P14<br>OOP/类型]
    B --> C[P15-P19<br>CPython内幕]
    C --> D[P20-P24<br>并发异步]
    D --> E[P25-P27<br>工程化]
    E --> F[P28-P30<br>Web/数据/演进]

    style A fill:#c8e6c9
    style B fill:#bbdefb
    style C fill:#fff9c4
    style D fill:#ffccbc
    style E fill:#e1bee7
    style F fill:#d7ccc8
```

按顺序读 = 完整 Python 后端体系；面试突击 = 看 [INDEX.md](./INDEX.md) 的「路径 A」。
