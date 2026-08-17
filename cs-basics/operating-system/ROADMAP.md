# 操作系统（408）路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 7 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([408 操作系统 35 分]) --> M1[模块一: 地基]
    M1 --> OS01[OS01 概述与运行环境]

    OS01 --> M2[模块二: 进程管理 ⭐主战场]
    M2 --> OS02[OS02 进程与线程]
    M2 --> OS03[OS03 CPU 调度]
    M2 --> OS04[OS04 同步与互斥]
    M2 --> OS05[OS05 死锁]

    OS02 --> M3[模块三: 内存管理]
    M3 --> OS06[OS06 内存管理]
    M3 --> OS07[OS07 虚拟内存]

    OS06 --> M4[模块四: 文件与 IO]
    M4 --> OS08[OS08 文件管理]
    M4 --> OS09[OS09 磁盘与 IO 管理]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style OS04 fill:#ffcdd2
    style OS05 fill:#ffcdd2
    style OS07 fill:#ffcdd2
```

---

## 🔀 进程五状态模型（OS02 高频）

```mermaid
stateDiagram-v2
    [*] --> 创建态: 提交作业
    创建态 --> 就绪态: 系统完成 PCB 创建
    就绪态 --> 运行态: 调度程序选中
    运行态 --> 就绪态: 时间片用完 / 被抢占
    运行态 --> 阻塞态: 请求 IO 或等待事件
    阻塞态 --> 就绪态: IO 完成 / 事件到达
    运行态 --> 终止态: 正常结束或异常
    终止态 --> [*]

    note right of 阻塞态
        ❌ 就绪 → 阻塞 不可能
           没运行怎么会主动请求 IO
        ❌ 阻塞 → 运行 不可能
           必须先回到就绪态排队
    end note
```

**必考点**：`就绪 → 阻塞` 与 `阻塞 → 运行` **都不可能**。前者因为只有运行中的进程才能发出 I/O 请求，后者因为唤醒后必须重新参与调度。

---

## 🧵 线程模型对比（OS02）

```mermaid
graph TB
    subgraph 多对一 用户级线程
        U1[线程1] --> K1[内核线程]
        U2[线程2] --> K1
        U3[线程3] --> K1
        N1["❌ 一个阻塞 全部阻塞<br>❌ 不能多核并行<br>✅ 切换快 不进内核"]
    end
    subgraph 一对一 内核级线程
        V1[线程1] --> L1[内核线程1]
        V2[线程2] --> L2[内核线程2]
        N2["✅ 可多核并行<br>✅ 单个阻塞不影响其他<br>❌ 切换要进内核 开销大"]
    end
    subgraph 多对多 组合
        W1[线程1] --> M1[内核线程1]
        W2[线程2] --> M1
        W3[线程3] --> M2[内核线程2]
        N3["✅ 兼顾两者<br>Go 的 GMP 属于这类"]
    end

    style N1 fill:#ffcdd2
    style N2 fill:#fff9c4
    style N3 fill:#c8e6c9
```

---

## ⏱️ 调度算法选型（OS03）

```mermaid
flowchart TD
    Q{优化目标}
    Q -->|简单公平 先来先服务| FCFS["FCFS<br>✅ 公平无饥饿<br>❌ 短作业等太久 护航效应"]
    Q -->|平均周转时间最短| SJF["SJF/SRTN ⭐<br>✅ 平均周转理论最优<br>❌ 长作业饥饿 需预知运行时间"]
    Q -->|兼顾长短作业| HRRN["HRRN 高响应比<br>R = 等待+运行 / 运行<br>✅ 等久了自动升优先级 无饥饿"]
    Q -->|分时系统 响应时间| RR["时间片轮转<br>✅ 响应快 公平<br>⚠️ 时间片过小则切换开销大"]
    Q -->|区分任务重要性| PRI["优先级调度<br>⚠️ 低优先级饥饿 需老化"]
    Q -->|通用 无需预知| MLFQ["多级反馈队列 ⭐<br>综合以上全部优点<br>实际系统的主流思路"]

    style SJF fill:#ffcdd2
    style MLFQ fill:#c8e6c9
```

**指标公式**：
- `周转时间 = 完成时间 − 到达时间`
- `带权周转时间 = 周转时间 ÷ 实际运行时间`（**恒 ≥ 1**，越接近 1 越好）
- `等待时间 = 周转时间 − 运行时间`

---

## 🔒 临界区四原则与实现方案（OS04）

```mermaid
graph TB
    P[临界区四原则] --> P1[空闲让进<br>临界区空闲时应允许进入]
    P --> P2[忙则等待<br>已有进程在内 其他必须等]
    P --> P3[有限等待<br>等待时间有上限 不能饿死]
    P --> P4[让权等待 ⭐<br>不能进入时应释放 CPU<br>否则忙等浪费]

    style P4 fill:#ffcdd2
```

```mermaid
flowchart TD
    S{实现方式}
    S -->|软件| SW["单标志法 违反空闲让进<br>双标志先检查 违反忙则等待<br>双标志后检查 违反空闲让进+有限等待<br>Peterson ✅ 前三条都满足 但仍忙等"]
    S -->|硬件| HW["中断屏蔽 只适用单核<br>TSL / Swap 指令 ✅ 简单可靠<br>共同缺点: 不满足让权等待"]
    S -->|OS 原语| SEM["信号量 P/V ⭐<br>✅ 四原则全满足<br>P: 减1 小于0 则阻塞自己<br>V: 加1 小于等于0 则唤醒一个"]
    S -->|语言级| MON["管程 Monitor<br>✅ 互斥由编译器保证<br>条件变量 wait/signal<br>Java synchronized 的原型"]

    style SEM fill:#c8e6c9
    style MON fill:#bbdefb
```

---

## 🍝 五大经典同步问题（OS04）

```mermaid
graph LR
    C1["生产者-消费者<br>mutex + empty + full<br>⚠️ 互斥 P 必须在同步 P 之后"] --> C2["多生产者多消费者<br>盘子问题 关键在<br>是否需要 mutex"]
    C2 --> C3["读者-写者<br>读者优先 / 写者优先<br>count + 计数锁"]
    C3 --> C4["哲学家进餐<br>破坏循环等待:<br>奇偶异序 / 限制人数 / 同时拿两只"]
    C4 --> C5["吸烟者问题<br>组合信号量分发"]

    style C1 fill:#ffcdd2
    style C4 fill:#fff9c4
```

**PV 操作解题三步**：① 找出**同步关系**（谁必须在谁之前）；② 每个同步关系设一个信号量（初值 = 初始可用资源数）；③ 前 V 后 P 配对写出——**互斥信号量的 P 一定放在同步信号量的 P 之后**，否则会死锁。

---

## ☠️ 死锁四条件与三种策略（OS05）

```mermaid
graph TB
    D[死锁四个必要条件] --> D1[互斥条件<br>资源一次只能一个进程用]
    D --> D2[不剥夺条件<br>不能强抢 只能主动释放]
    D --> D3[请求并保持<br>持有资源的同时申请新资源]
    D --> D4[循环等待<br>存在进程-资源的环形链]

    style D4 fill:#ffcdd2
```

```mermaid
flowchart TD
    S{处理策略}
    S -->|事前破坏条件| P["死锁预防<br>破坏 4 条件之一<br>❌ 资源利用率低 系统吞吐下降"]
    S -->|事中动态判断| A["死锁避免 ⭐<br>银行家算法<br>每次分配前判断是否仍安全"]
    S -->|事后检测处理| DE["死锁检测与解除<br>资源分配图化简<br>解除: 剥夺 / 撤销 / 回退"]
    S -->|不管| I["鸵鸟策略<br>Linux/Windows 的实际选择<br>死锁概率低 处理成本高"]

    P --> P1["破坏互斥: SPOOLing 技术"]
    P --> P2["破坏不剥夺: 申请不到就释放全部已有"]
    P --> P3["破坏请求保持: 静态分配 一次性申请全部"]
    P --> P4["破坏循环等待: 资源顺序编号 必须递增申请"]

    style A fill:#ffcdd2
    style I fill:#eeeeee
```

**银行家算法五步**：① `Need = Max − Allocation`；② 找 `Need ≤ Available` 的进程；③ 假设分配并**回收其全部资源**（`Available += Allocation`）；④ 重复直到所有进程入列（**安全**）或找不到（**不安全**）；⑤ 输出安全序列。

---

## 🧠 内存管理演进（OS06）

```mermaid
graph TD
    A[单一连续分配<br>只能一道程序] --> B[固定分区<br>❌ 内部碎片]
    B --> C[动态分区<br>❌ 外部碎片 需紧凑]
    C --> D[基本分页 ⭐<br>❌ 内部碎片 但很小<br>✅ 无外部碎片]
    C --> E[基本分段<br>✅ 便于共享与保护<br>❌ 外部碎片]
    D --> F[段页式<br>✅ 兼顾两者<br>❌ 三次访存]
    E --> F
    D --> G[请求分页 = 虚拟内存<br>OS07]

    style D fill:#c8e6c9
    style G fill:#ffcdd2
```

### 碎片辨析（最高频概念题）

```mermaid
flowchart LR
    I["内部碎片<br>分配给进程但用不到的部分<br>👉 固定分区 · 分页"]
    E["外部碎片<br>空闲但太小无法分配的部分<br>👉 动态分区 · 分段"]
    F["紧凑/拼接<br>只能解决外部碎片<br>需要动态重定位支持"]
    E --> F

    style I fill:#fff9c4
    style E fill:#ffcdd2
```

---

## 📄 分页地址变换全流程（OS06-OS07）

```mermaid
flowchart TD
    A["逻辑地址<br>页号 P = A / 页大小<br>页内偏移 W = A % 页大小"] --> B{查快表 TLB}
    B -->|命中 ~1 次访存| C["取出块号 b"]
    B -->|未命中| D[查内存中的页表]
    D --> E{页表项有效位}
    E -->|在内存| F["取出块号 b<br>并填入 TLB"]
    E -->|不在内存 缺页| G["缺页中断<br>OS 从外存调入<br>可能需要页面置换"]
    G --> F
    C --> H["物理地址 = b × 页大小 + W"]
    F --> H

    style B fill:#c8e6c9
    style G fill:#ffcdd2
    style H fill:#bbdefb
```

**访存次数**：TLB 命中 = **1 次**（只访数据）；TLB 未命中但页在内存 = **2 次**（页表 + 数据）；两级页表且 TLB 未命中 = **3 次**；段页式 = **3 次**（段表 + 页表 + 数据）。

---

## 🔄 页面置换算法（OS07 必考）

```mermaid
flowchart TD
    Q{置换谁}
    Q -->|未来最久不用的| OPT["OPT 最佳置换<br>✅ 缺页最少 理论下界<br>❌ 无法实现 只作评价标准"]
    Q -->|最早进来的| FIFO["FIFO 先进先出<br>✅ 实现简单<br>❌ 性能差 ⚠️有 Belady 异常"]
    Q -->|最久没被用过的| LRU["LRU 最近最久未使用<br>✅ 性能接近 OPT<br>❌ 需硬件支持 开销大"]
    Q -->|近似 LRU| CLK["CLOCK 时钟/二次机会 ⭐<br>访问位 = 0 就换 = 1 则清零给二次机会<br>✅ 开销小 性能好 实际常用"]
    CLK --> CLK2["改进型 CLOCK<br>再看修改位 优先换<br>未访问且未修改的页 减少写盘"]

    style OPT fill:#eeeeee
    style FIFO fill:#ffcdd2
    style CLK fill:#c8e6c9
```

**Belady 异常**：**分配的物理块增多，缺页次数反而增加**。只有 **FIFO** 会出现。
LRU / OPT 属于**栈算法**（`k` 块时驻留的页集合 ⊆ `k+1` 块时的页集合），数学上保证不会出现该异常。

---

## 📁 文件物理分配方式（OS08）

```mermaid
graph TB
    subgraph 连续分配
        A1["✅ 支持随机访问 顺序读最快"]
        A2["❌ 外部碎片 · 文件难扩展"]
    end
    subgraph 链接分配-隐式
        B1["✅ 无外部碎片 易扩展"]
        B2["❌ 只能顺序访问 · 指针占空间<br>❌ 一个指针坏了全断"]
    end
    subgraph 链接分配-显式FAT
        C1["✅ 把指针集中到 FAT 表 常驻内存<br>✅ 支持随机访问"]
        C2["❌ FAT 占内存 · 卷越大表越大"]
    end
    subgraph 索引分配
        D1["✅ 支持随机访问 易扩展 ⭐"]
        D2["❌ 索引块占空间 · 小文件开销相对大"]
        D3["扩展: 链接方案 / 多层索引 / 混合索引"]
    end

    style C1 fill:#bbdefb
    style D1 fill:#c8e6c9
```

**UNIX 混合索引**：inode 中放 `N` 个直接块 + 1 个一级间址 + 1 个二级间址 + 1 个三级间址。
最大文件大小 = `(N + k + k² + k³) × 块大小`，其中 `k = 块大小 ÷ 每个地址所占字节数`。

---

## 💽 磁盘调度算法（OS09 必考）

```mermaid
flowchart TD
    Q{调度策略}
    Q -->|按请求顺序| F["FCFS<br>✅ 公平 简单<br>❌ 磁头来回跑 平均寻道最长"]
    Q -->|每次选最近的| S["SSTF 最短寻道优先<br>✅ 平均寻道短<br>❌ 远端请求饥饿 磁头黏在中间"]
    Q -->|一个方向走到底再返回| SC["SCAN 电梯算法 ⭐<br>✅ 无饥饿<br>❌ 两端等待时间不均"]
    Q -->|走到底后直接跳回起点| CS["C-SCAN 循环扫描<br>✅ 响应时间更均匀<br>❌ 返回过程有额外移动"]
    Q -->|到最远请求处就返回 不到边界| LK["LOOK / C-LOOK<br>SCAN / C-SCAN 的优化版<br>实际系统常用"]

    style SC fill:#c8e6c9
    style LK fill:#bbdefb
```

**时间构成**：`存取时间 = 寻道时间 + 旋转延迟(平均半圈) + 传输时间`。磁盘调度**只能优化寻道时间**；旋转延迟靠**交替编号 / 错位命名**优化。

---

## 🧊 缓冲区与 SPOOLing（OS09）

```mermaid
graph LR
    subgraph 单缓冲
        S1["处理一块耗时<br>max(C, T) + M"]
    end
    subgraph 双缓冲
        S2["处理一块耗时<br>max(C+M, T)"]
    end
    subgraph 缓冲池
        S3["公用缓冲池<br>收容输入 提取输入<br>收容输出 提取输出"]
    end
    S1 --> S2 --> S3

    style S2 fill:#c8e6c9
```

```mermaid
flowchart LR
    A[独占设备 如打印机] --> B[SPOOLing 假脱机技术]
    B --> C[输入井 / 输出井<br>磁盘上的存储区]
    B --> D[输入进程 / 输出进程<br>模拟外围控制机]
    B --> E[✅ 改造成共享设备<br>✅ 提高 IO 速度<br>✅ 实现虚拟设备]

    style E fill:#c8e6c9
```

---

## 🎓 建议学习顺序

```mermaid
graph LR
    A[OS01<br>运行环境] --> B[OS02-OS03<br>进程与调度]
    B --> C[OS04-OS05<br>同步与死锁]
    C --> D[OS06-OS07<br>内存与虚存]
    D --> E[OS08-OS09<br>文件与 IO]
    E --> F[五类大题<br>各刷 10 道]

    style C fill:#ffcdd2
    style D fill:#ffcdd2
    style F fill:#e1bee7
```

红色 = 最难也最值钱的两段。大题清单见 [INDEX.md 路径 C](./INDEX.md#路径-c只突击大题1-周)。
