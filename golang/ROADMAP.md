# Go 路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
> 所有 Mermaid 图可在 GitHub、VS Code、Obsidian、Typora 等支持 Mermaid 的工具中直接渲染
>
> **📅 内容基准：Go 1.26**（2026-02-10 发布）。覆盖 Go 1.21–1.26 五个版本特性。

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Go 之旅]) --> M1[模块 1: 语言基础]
    
    M1 --> G01[G01 变量/常量/iota]
    M1 --> G02[G02 数据类型/字符串]
    M1 --> G03[G03 切片]
    M1 --> G04[G04 Map]
    M1 --> G05[G05 Struct]
    M1 --> G06[G06 函数/闭包/defer]
    M1 --> G07[G07 指针/接收者]
    M1 --> G08[G08 接口]
    M1 --> G09[G09 泛型]
    M1 --> G10[G10 错误处理]
    
    M1 --> M2[模块 2: 并发]
    M2 --> G11[G11 Goroutines/GMP]
    M2 --> G12[G12 Channels]
    M2 --> G13[G13 sync 包]
    M2 --> G14[G14 context]
    M2 --> G15[G15 并发模式]
    
    M2 --> M3[模块 3: 工程化]
    M3 --> G16[G16 Race Detection]
    M3 --> G17[G17 Modules]
    M3 --> G18[G18 测试]
    M3 --> G19[G19 Benchmarking]
    
    M3 --> M4[模块 4: 性能与底层]
    M4 --> G20[G20 内存管理]
    M4 --> G21[G21 逃逸分析]
    M4 --> G22[G22 pprof]
    M4 --> G23[G23 runtime/trace]
    M4 --> G24[G24 反射]
    M4 --> G25[G25 unsafe]
    M4 --> G26[G26 CGO]
    
    M4 --> M5[模块 5: 生态]
    M5 --> G27[G27 net/http]
    M5 --> G28[G28 gRPC]
    M5 --> G29[G29 数据库]
    M5 --> G30[G30 日志]
    
    M5 --> End([Go 高级工程师])
    
    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef basic fill:#48bb78,stroke:#2f855a,color:#fff
    classDef concur fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef eng fill:#ecc94b,stroke:#b7791f,color:#000
    classDef perf fill:#f56565,stroke:#c53030,color:#fff
    classDef eco fill:#ed8936,stroke:#c05621,color:#fff
    
    class M1,M2,M3,M4,M5 module
    class G01,G02,G03,G04,G05,G06,G07,G08,G09,G10 basic
    class G11,G12,G13,G14,G15 concur
    class G16,G17,G18,G19 eng
    class G20,G21,G22,G23,G24,G25,G26 perf
    class G27,G28,G29,G30 eco
```

---

## 🟢 模块 1：语言基础（G01-G10）依赖图

```mermaid
graph LR
    G01[G01 变量/常量] --> G02[G02 数据类型]
    G02 --> G03[G03 切片]
    G02 --> G04[G04 Map]
    G02 --> G05[G05 Struct]
    G05 --> G07[G07 指针]
    G06[G06 函数/闭包] --> G07
    G07 --> G08[G08 接口]
    G08 --> G09[G09 泛型]
    G06 --> G10[G10 错误/panic]
    G08 --> G10

    classDef basic fill:#48bb78,stroke:#2f855a,color:#fff
    class G01,G02,G03,G04,G05,G06,G07,G08,G09,G10 basic
```

**核心知识点速记**：
- G01：var/const/iota；零值；作用域；shadowing
- G02：int 平台差异；UTF-8；StringHeader；零拷贝
- G03：SliceHeader(Data,Len,Cap)；append 扩容；别名陷阱
- G04：hmap+bucket(8 slot)；hash0 随机化；迭代乱序
- G05：内存对齐；字段重排；嵌入；noCopy
- G06：闭包按引用；defer 三种实现；命名返回值
- G07：值/指针接收者；方法集；可寻址性
- G08：iface/eface(16B)；itab 缓存；nil 装箱陷阱
- G09：类型参数；`~T`；GC shape stenciling
- G10：%w 包装；errors.Is/As/Join；panic 边界

---

## 🔵 模块 2：并发（G11-G15）依赖图

```mermaid
graph LR
    G11[G11 Goroutines] --> G12[G12 Channels]
    G11 --> G13[G13 sync]
    G12 --> G14[G14 context]
    G13 --> G14
    G14 --> G15[G15 并发模式]
    G12 --> G15
    G13 --> G15

    classDef concur fill:#4299e1,stroke:#2b6cb0,color:#fff
    class G11,G12,G13,G14,G15 concur
```

**Go 并发心智模型**：
```mermaid
graph TB
    App[Go 应用] --> Goroutines[Goroutines: 用户态]
    Goroutines --> Scheduler[GMP 调度器]
    Scheduler --> G[G: 任务]
    Scheduler --> M[M: OS 线程]
    Scheduler --> P[P: 逻辑 CPU]
    P -.runq.-> G
    M -.exec.-> G
    P -.binds.-> M
    
    Goroutines -->|通信| Channels[Channels]
    Goroutines -->|同步| Sync[sync.Mutex/RWMutex/WG/Once/Pool]
    Goroutines -->|生命周期| Context[context.Context]
    
    Channels --> Patterns[并发模式]
    Sync --> Patterns
    Context --> Patterns
    Patterns --> FanIn[Fan-in/out]
    Patterns --> Pipeline[Pipeline]
    Patterns --> WorkerPool[Worker Pool]
    Patterns --> Errgroup[errgroup]
```

---

## 🔴 模块 4：性能与底层（G20-G26）依赖图

```mermaid
graph TD
    G20[G20 内存管理/GC] --> G21[G21 逃逸分析]
    G20 --> G22[G22 pprof]
    G21 --> G22
    G22 --> G23[G23 runtime/trace]
    G20 -.-> G19[G19 Benchmarking]
    G19 -.-> G22
    G08[G08 接口] -.装箱.-> G21
    G05[G05 Struct] -.对齐.-> G20
    G24[G24 反射] --> G25[G25 unsafe]
    G25 --> G26[G26 CGO]

    classDef perf fill:#f56565,stroke:#c53030,color:#fff
    class G20,G21,G22,G23,G24,G25,G26 perf
    classDef ref fill:#fbb6ce,stroke:#b83280,color:#000
    class G05,G08,G19 ref
```

**性能调优工作流**：
```mermaid
flowchart TD
    Slow{服务慢?} --> Type{瓶颈类型?}
    
    Type -->|CPU 高| CPUProf[CPU profile]
    Type -->|内存涨| HeapProf[Heap profile]
    Type -->|延迟尾| Trace[runtime/trace]
    Type -->|锁争用| MutexProf[Mutex profile]
    Type -->|goroutine 多| GoroutineProf[Goroutine profile]
    
    CPUProf --> Hotspot[找热点函数]
    HeapProf --> AllocSrc[找分配源]
    Trace --> Schedule[看调度延迟]
    
    Hotspot --> Optimize{优化方向}
    AllocSrc --> Optimize
    Optimize -->|减少分配| Stack[逃逸分析<br>sync.Pool<br>strings.Builder]
    Optimize -->|算法| Algo[换数据结构]
    Optimize -->|并发| Concur[goroutine + channel]
    
    Stack --> Bench[benchmark + benchstat]
    Algo --> Bench
    Concur --> Bench
    Bench -->|p<0.05| Done([完成])
    Bench -->|无显著差| Restart[重新分析]
    Restart --> Type
```

---

## 🟠 模块 5：生态（G27-G30）依赖图

```mermaid
graph LR
    G27[G27 net/http] --> G28[G28 gRPC]
    G27 --> G29[G29 数据库]
    G28 --> G29
    G27 --> G30[G30 日志]
    G28 --> G30
    G29 --> G30
    G14[G14 context] -.-> G27
    G14 -.-> G28
    G14 -.-> G29

    classDef eco fill:#ed8936,stroke:#c05621,color:#fff
    class G27,G28,G29,G30 eco
    classDef ref fill:#bee3f8,stroke:#2b6cb0,color:#000
    class G14 ref
```

**生产级服务架构**（典型 Go 微服务）：
```mermaid
graph TB
    Client[客户端] --> LB[负载均衡]
    LB --> HTTP[net/http Server]
    HTTP --> MW1[Recover Middleware]
    MW1 --> MW2[RequestID + Trace]
    MW2 --> MW3[Auth]
    MW3 --> MW4[RateLimit]
    MW4 --> Handler[业务 Handler]
    
    Handler --> Ctx[context.Context<br>含 trace_id + deadline]
    Ctx --> gRPCClient[gRPC Client]
    gRPCClient --> Downstream[下游服务]
    Ctx --> DB[(database/sql<br>+ pgx)]
    
    Handler --> Logger[slog/zap]
    Logger --> Stdout[JSON to stdout]
    Stdout --> ELK[ELK / Loki]
    
    Handler --> Trace[OpenTelemetry]
    Trace --> Jaeger[Jaeger / Tempo]
    
    Handler --> Metrics[Prometheus]
    Metrics --> Grafana[Grafana]
    
    classDef infra fill:#4a5568,stroke:#2d3748,color:#fff
    classDef code fill:#48bb78,stroke:#2f855a,color:#fff
    classDef obs fill:#ed8936,stroke:#c05621,color:#fff
    
    class LB,Downstream,ELK,Jaeger,Grafana,DB infra
    class HTTP,MW1,MW2,MW3,MW4,Handler,Ctx,gRPCClient,Logger,Trace,Metrics code
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（推荐）

```mermaid
graph LR
    W1[第 1-4 周<br>G01-G10<br>语言基础] --> W5[第 5-7 周<br>G11-G15<br>并发]
    W5 --> W8[第 8 周<br>G16-G19<br>工程化]
    W8 --> W9[第 9-11 周<br>G20-G26<br>性能底层]
    W9 --> W12[第 12 周<br>G27-G30<br>生态]
    
    style W1 fill:#48bb78
    style W5 fill:#4299e1
    style W8 fill:#ecc94b
    style W9 fill:#f56565
    style W12 fill:#ed8936
```

### 路径 B：性能特化

```mermaid
graph LR
    Q1[预备<br>G05/G07/G08] --> Q2[核心<br>G20→G21→G19]
    Q2 --> Q3[工具<br>G22→G23]
    Q3 --> Q4[进阶<br>G13/G24/G25]
    
    style Q1 fill:#48bb78
    style Q2 fill:#f56565
    style Q3 fill:#f56565
    style Q4 fill:#f56565
```

### 路径 C：并发特化

```mermaid
graph LR
    C1[G11 GMP] --> C2[G12 Channels]
    C2 --> C3[G13 sync]
    C3 --> C4[G14 context]
    C4 --> C5[G15 模式]
    C5 --> C6[G16 race]
    
    style C1 fill:#4299e1
    style C2 fill:#4299e1
    style C3 fill:#4299e1
    style C4 fill:#4299e1
    style C5 fill:#4299e1
    style C6 fill:#ecc94b
```

### 路径 D：后端入职急训（4 周）

```mermaid
gantt
    title 1 个月 Go 后端急训
    dateFormat YYYY-MM-DD
    section 第 1 周
    G01-G06 语法基础          :a1, 2026-05-12, 7d
    section 第 2 周
    G07-G10 类型/接口/泛型/错误 :a2, after a1, 7d
    section 第 3 周
    G11-G15 并发              :a3, after a2, 7d
    section 第 4 周
    G18 测试                  :a4, after a3, 2d
    G27 net/http              :a5, after a4, 2d
    G28-G29 gRPC/DB           :a6, after a5, 3d
```

---

## 🧠 知识检索思维导图

```mermaid
mindmap
  root((Go 路线图))
    语言基础
      变量与常量 G01
        var/const/iota
        无类型常量
        作用域
      类型系统 G02
        int 平台差异
        UTF-8 字符串
        rune vs byte
      复合类型
        切片 G03
        Map G04
        Struct G05
      函数与方法
        闭包 defer G06
        接收者 G07
      抽象
        接口 G08
        泛型 G09
      错误 G10
    并发
      Goroutines G11
        GMP
        work stealing
        异步抢占
      Channels G12
      sync G13
        Mutex
        RWMutex
        WaitGroup
        Once
        Pool
        atomic
      context G14
      模式 G15
        pipeline
        fan-in/out
        worker pool
        errgroup
    工程化
      race G16
      modules G17
      测试 G18
      benchmark G19
    性能
      GC G20
      逃逸 G21
      pprof G22
      trace G23
      反射 G24
      unsafe G25
      CGO G26
    生态
      net/http G27
      gRPC G28
      DB G29
      日志 G30
```

---

## 📊 难度与重要性矩阵

> Mermaid 的 `quadrantChart` 在多数渲染器（GitHub、VS Code、Hugo 等）兼容性差，这里改用表格。
> 难度：⭐ 简单 ⭐⭐ 中等 ⭐⭐⭐ 进阶 ⭐⭐⭐⭐ 难 ⭐⭐⭐⭐⭐ 极难
> 重要性：🔥🔥🔥🔥🔥 必学 → 🔥 选学

### 必学 + 难（高优先级，要花时间啃）

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| G08 接口 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | iface / eface 装箱、nil 陷阱 |
| G11 GMP | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 调度器与抢占机制 |
| G12 Channel | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 死锁 / close 语义 |
| G13 sync 原语 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | Mutex / Pool / Once / Map 取舍 |
| G15 并发模式 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | Pipeline / Fan-out / Worker pool |
| G20 GC | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 三色 + 写屏障 + STW |
| G21 逃逸分析 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 性能调优起点 |
| G22 pprof | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | CPU / heap / goroutine 全套 |

### 必学 + 简单（性价比最高）

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| G01 变量 | ⭐ | 🔥🔥🔥🔥🔥 | 语言基础 |
| G02 类型 | ⭐⭐ | 🔥🔥🔥🔥 | type / 别名 / 转换 |
| G10 错误处理 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | error wrap / errors.Is/As |
| G14 context | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 取消 / 超时 / 传值规范 |
| G27 net/http | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | server / client / mux 7+ |
| G29 数据库访问 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | database/sql + sqlx / GORM |

### 必学 + 中等

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| G03 切片 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | append / 共享底层数组 |
| G04 Map | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | hmap 扩容 / 并发 |
| G05 Struct | ⭐⭐⭐ | 🔥🔥🔥🔥 | 内存对齐 / tag |
| G06 函数 | ⭐⭐⭐ | 🔥🔥🔥🔥 | 闭包捕获 / defer |
| G07 指针 | ⭐⭐⭐ | 🔥🔥🔥🔥 | 值 vs 引用 / receiver |
| G16 race detector | ⭐⭐⭐ | 🔥🔥🔥🔥 | 必须 -race 跑测试 |
| G18 测试 | ⭐⭐⭐ | 🔥🔥🔥🔥 | table-driven / fuzz |
| G28 gRPC | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 流 / 拦截器 / metadata |
| G30 日志 | ⭐⭐ | 🔥🔥🔥🔥 | slog 标准化 |

### 选学（按需深入）

| 课程 | 难度 | 重要性 | 何时学 |
|---|---|---|---|
| G09 泛型 | ⭐⭐⭐⭐ | 🔥🔥🔥 | 写库 / 容器时 |
| G17 modules | ⭐⭐ | 🔥🔥🔥 | 工程化协作 |
| G19 benchmark | ⭐⭐⭐ | 🔥🔥🔥 | 性能优化前的基线 |
| G23 trace | ⭐⭐⭐⭐ | 🔥🔥 | 调度异常排查 |
| G24 反射 | ⭐⭐⭐⭐ | 🔥🔥 | 写框架时 |
| G25 unsafe | ⭐⭐⭐⭐⭐ | 🔥 | 极限性能 / 与 C 交互 |
| G26 CGO | ⭐⭐⭐⭐⭐ | 🔥 | 需要复用 C 库时 |

---

## 🔗 跨模块知识连接

某些主题会在多个章节出现——理解"为什么"比死记每章更重要。

```mermaid
graph TD
    EscapeAnalysis[逃逸分析]
    Allocation[堆分配]
    GC[GC 压力]
    
    G08接口装箱 --> Allocation
    G03切片append --> Allocation
    G04map扩容 --> Allocation
    G06闭包捕获 --> Allocation
    G24反射 --> Allocation
    
    Allocation --> GC
    EscapeAnalysis --> Allocation
    
    G21逃逸分析 --> EscapeAnalysis
    G20GC --> GC
    
    GC --> G22pprof分析
    GC --> G23trace观察
    
    Solution[降低分配]
    G13syncPool --> Solution
    G05字段对齐 --> Solution
    G25unsafe转换 --> Solution
    G09泛型代替接口 --> Solution
    Solution --> G19benchmark验证
    
    style EscapeAnalysis fill:#f56565,color:#fff
    style Allocation fill:#ed8936,color:#fff
    style GC fill:#ed8936,color:#fff
    style Solution fill:#48bb78,color:#fff
```

---

## ✅ 阶段性自检

每完成一个模块，应该能回答的"灵魂问题"：

### 模块 1（基础）后

```mermaid
graph LR
    Q1[nil slice vs<br>empty slice<br>差别?] -.-> Y1[json/IsNil]
    Q2[len 中文字符串<br>= ?] -.-> Y2[字节数, 不是 rune]
    Q3[Counter值接收者<br>Inc 为何无效?] -.-> Y3[改的是副本]
    Q4[interface 含 nil 指针<br>== nil?] -.-> Y4[false, 装箱陷阱]
```

### 模块 2（并发）后

```mermaid
graph LR
    Q1[GMP 中 P 数量<br>由谁决定?] -.-> Y1[GOMAXPROCS]
    Q2[向已 close<br>channel send?] -.-> Y2[panic]
    Q3[select 多 case<br>就绪选哪个?] -.-> Y3[随机]
    Q4[context.WithCancel<br>cancel 不调用?] -.-> Y4[goroutine 泄漏]
```

### 模块 4（性能）后

```mermaid
graph LR
    Q1[GOGC=100 含义?] -.-> Y1[堆涨 2x 触发 GC]
    Q2[一行 fmt.Println n<br>为啥 n 逃逸?] -.-> Y2[装箱为 any]
    Q3[pprof CPU<br>flat vs cum?] -.-> Y3[自身 vs 含子调用]
    Q4[uintptr 跨函数<br>调用安全吗?] -.-> Y4[不, GC 失效]
```

---

## 🆕 2026 新特性按版本归口

```mermaid
graph LR
    V123[Go 1.23<br>2024-08] --> F1[iter.Seq + range over func]
    V123 --> F2[unique 包]
    V123 --> F3[slices/maps 迭代器化]

    V124[Go 1.24<br>2025-02] --> G1[Swiss Tables map]
    V124 --> G2[泛型类型别名 GA]
    V124 --> G3[weak 包]
    V124 --> G4[runtime.AddCleanup]
    V124 --> G5[tool 指令]
    V124 --> G6[testing/synctest 实验]
    V124 --> G7[omitzero / sql.Null 泛型]

    V125[Go 1.25<br>2025-08] --> H1[container-aware GOMAXPROCS]
    V125 --> H2[Green Tea GC 实验]
    V125 --> H3[testing/synctest 稳定]
    V125 --> H4[FlightRecorder]
    V125 --> H5[CrossOriginProtection]
    V125 --> H6[WaitGroup.Go]
    V125 --> H7[reflect.TypeAssert]
    V125 --> H8[slog.GroupAttrs]

    V126[Go 1.26<br>2026-02] --> I1[Green Tea GC 默认]
    V126 --> I2[CGO -30%]
    V126 --> I3[crypto/hpke 后量子]
    V126 --> I4[goroutineleak profile]
    V126 --> I5[go fix 重做]
    V126 --> I6[slog.NewMultiHandler]
    V126 --> I7[实验 SIMD]

    style V123 fill:#e8f5e9
    style V124 fill:#e3f2fd
    style V125 fill:#fff3e0
    style V126 fill:#fce4ec
```

---

> 📁 本路线图位于 `/data/workspace/dp4/golang/ROADMAP.md`
> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [QUIZ.md](./QUIZ.md) 测验题
