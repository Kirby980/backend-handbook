# 计算机组成原理（408）路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 8 月**

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([408 组成原理 45 分]) --> M1[模块一: 全局观]
    M1 --> CO01[CO01 概述与性能指标]

    CO01 --> M2[模块二: 数据与运算]
    M2 --> CO02[CO02 数据表示 IEEE754]
    M2 --> CO03[CO03 运算方法与运算器]

    CO01 --> M3[模块三: 存储]
    M3 --> CO04[CO04 存储系统]
    M3 --> CO05[CO05 Cache 与虚拟存储器]

    CO03 --> M4[模块四: 指令与 CPU]
    M4 --> CO06[CO06 指令系统]
    M4 --> CO07[CO07 中央处理器]

    CO07 --> M5[模块五: 互连与 IO]
    M5 --> CO08[CO08 总线与输入输出]

    style M1 fill:#c8e6c9
    style M2 fill:#bbdefb
    style M3 fill:#fff9c4
    style M4 fill:#ffccbc
    style M5 fill:#e1bee7
    style CO05 fill:#ffcdd2
    style CO07 fill:#ffcdd2
```

> 红色 = 大题最高频（Cache 地址划分、流水线时空图）。

---

## 🏛️ 冯·诺依曼结构与指令执行（CO01）

```mermaid
graph TB
    subgraph CPU
        CU[控制器 CU<br>PC IR 时序]
        ALU[运算器 ALU<br>ACC MQ X]
        Reg[寄存器组]
    end
    Mem[存储器<br>MAR MDR]
    In[输入设备]
    Out[输出设备]

    In -->|数据| Mem
    Mem -->|指令| CU
    Mem <-->|数据| ALU
    CU -->|控制信号| ALU
    CU -->|控制信号| Mem
    ALU --> Out

    style CU fill:#ffcdd2
    style Mem fill:#fff9c4
```

```mermaid
flowchart LR
    F["取指 IF<br>PC → MAR → M → MDR → IR"] --> D["译码 ID<br>OP 送 CU 分析"]
    D --> E["执行 EX<br>取操作数 + 运算"]
    E --> W["写回 WB"]
    W --> P["PC+1 取下一条"]
    P --> F

    style F fill:#c8e6c9
    style E fill:#ffccbc
```

---

## 🔢 机器数与浮点（CO02）

```mermaid
graph TB
    Num[机器数表示] --> Fixed[定点数]
    Num --> Float[浮点数]

    Fixed --> O[原码<br>符号+绝对值<br>0 有两种表示]
    Fixed --> F1[反码<br>负数按位取反]
    Fixed --> C[补码 ⭐<br>反码+1<br>减法变加法 0 唯一]
    Fixed --> M[移码<br>补码符号位取反<br>用于阶码 便于比大小]

    Float --> IEEE["IEEE 754<br>(-1)^S × 1.M × 2^(E-bias)"]
    IEEE --> S1["单精度 32 位<br>1 + 8 + 23  bias=127"]
    IEEE --> S2["双精度 64 位<br>1 + 11 + 52  bias=1023"]

    style C fill:#ffcdd2
    style IEEE fill:#ffcdd2
```

### IEEE 754 特殊值判定

```mermaid
flowchart TD
    E{阶码 E 的值}
    E -->|全 0| Z{尾数 M}
    Z -->|全 0| Zero["±0"]
    Z -->|非 0| Denorm["非规格化数<br>隐含位取 0 指数固定 1-bias"]
    E -->|全 1| N{尾数 M}
    N -->|全 0| Inf["±∞"]
    N -->|非 0| NaN["NaN<br>NaN != NaN"]
    E -->|其他| Norm["规格化数<br>隐含位取 1"]

    style Norm fill:#c8e6c9
    style NaN fill:#ffcdd2
```

---

## ➕ 标志位与溢出判断（CO03）

```mermaid
flowchart TD
    Q{把数据当成什么解释}
    Q -->|有符号数| SF["看 OF 溢出标志<br>OF = 最高位进位 ⊕ 次高位进位<br>或: 双符号位不同即溢出"]
    Q -->|无符号数| CF["看 CF 进位借位标志<br>加法有进位 / 减法有借位"]

    SF --> R1["OF=1 → 有符号溢出<br>正+正=负 或 负+负=正"]
    CF --> R2["CF=1 → 无符号溢出"]

    Note["同一次加法可能<br>OF=1 而 CF=0<br>也可能 OF=0 而 CF=1"]

    style SF fill:#ffcdd2
    style CF fill:#bbdefb
    style Note fill:#fff9c4
```

---

## 🗄️ 存储器层次（CO04-CO05）

```mermaid
graph TB
    R["寄存器<br>~1 周期 · 数百 B"] --> L1["L1 Cache<br>~4 周期 · 32-64 KB"]
    L1 --> L2["L2 Cache<br>~12 周期 · 256KB-1MB"]
    L2 --> L3["L3 Cache<br>~40 周期 · 数十 MB"]
    L3 --> Main["主存 DRAM<br>~200 周期 · 数十 GB"]
    Main --> SSD["SSD/NVMe<br>~10 万周期 · TB"]
    SSD --> HDD["机械硬盘<br>~千万周期"]

    style R fill:#c8e6c9
    style L1 fill:#dcedc8
    style Main fill:#fff9c4
    style HDD fill:#ffcdd2
```

> 「越快 → 越贵 → 越小」。上层是下层的**缓存**，靠**时间局部性 + 空间局部性**才有效。

---

## 🎯 Cache 三种映射与地址划分（CO05 必考）

```mermaid
flowchart TD
    Q{主存块能放到哪些 Cache 行}
    Q -->|只能放固定的一行| D["直接映射<br>行号 = 块号 % 行数<br>硬件简单 · 冲突多"]
    Q -->|任意行都可以| F["全相联<br>无行号字段<br>命中率高 · 比较器最贵"]
    Q -->|固定组内的任意行| S["组相联 ⭐<br>组号 = 块号 % 组数<br>折中 · 主流"]

    D --> DA["地址 = 标记 + 行号 + 块内地址"]
    F --> FA["地址 = 标记 + 块内地址"]
    S --> SA["地址 = 标记 + 组号 + 块内地址"]

    style S fill:#ffcdd2
    style SA fill:#ffcdd2
```

**解题三步**：① 块内地址位数 = log₂(块大小)；② 行号/组号位数 = log₂(行数 或 组数)；③ 标记位 = 物理地址总位数 − 前两者。

---

## ✍️ Cache 写策略（CO05）

```mermaid
flowchart TD
    W{写命中时}
    W -->|同时写 Cache 和主存| WT["写直达 Write Through<br>一致性好 · 写流量大<br>常配写缓冲"]
    W -->|只写 Cache 置脏位| WB["写回 Write Back<br>写流量小 · 替换时写回<br>需脏位 一致性复杂"]

    M{写不命中时}
    M -->|先调块进 Cache 再写| WA["写分配 Write Allocate<br>常与写回搭配"]
    M -->|直接写主存 不调块| NWA["非写分配<br>常与写直达搭配"]

    style WB fill:#c8e6c9
    style WT fill:#bbdefb
```

---

## 🛡️ 存储器校验与 RAID（CO04）

```mermaid
flowchart TD
    D["码距 d = 任意两合法码字的最小不同位数"] --> E["检 e 位错<br/>需 d ≥ e + 1"]
    D --> T["纠 t 位错<br/>需 d ≥ 2t + 1"]
    D --> B["纠 t 检 e<br/>需 d ≥ t + e + 1"]

    E --> P["奇偶校验 d=2<br/>只检【奇数位】错<br/>❌ 不能纠错"]
    T --> H["海明码 d=3<br/>✅ 纠 1 位<br/>2^k ≥ n+k+1<br/>校验位放 2^i 位"]
    B --> EC["SEC-DED d=4<br/>纠1位 + 检2位<br/>= ECC 内存"]

    style H fill:#ffcdd2
    style EC fill:#c8e6c9
```

**海明码解题三步**：① 由 `2^k ≥ n+k+1` 算校验位数 → ② 校验位放第 1、2、4、8… 位，`Pᵢ` 校验「位号二进制第 `log₂i` 位为 1」的所有位 → ③ 接收方把各组校验结果拼成 `S₈S₄S₂S₁`，**这个数就是出错位的位号**。

> 💡 **ECC 内存为什么是 72 位**：64 位数据纠 1 位需 `k=7`，再加 1 位检双错 = 8 位冗余 → `64+8=72`。

```mermaid
flowchart LR
    R0["RAID 0 条带<br/>容量 n · 容错【0】<br/>❌ 无冗余 可靠性更差"]
    R1["RAID 1 镜像<br/>容量 n/2 · 容错 1"]
    R5["RAID 5 分布校验 ⭐<br/>≥3 盘 · 容量 n-1 · 容错【1】<br/>P = D1⊕D2⊕D3"]
    R6["RAID 6 双校验<br/>≥4 盘 · 容量 n-2 · 容错【2】"]
    R10["RAID 10 镜像+条带<br/>≥4 盘 · 容量 n/2<br/>✅ 数据库首选"]

    R0 --> R1 --> R5 --> R6
    R1 --> R10

    style R5 fill:#fff9c4
    style R10 fill:#c8e6c9
    style R0 fill:#ffcdd2
```

**写惩罚**：RAID 5 改一个块要 **2 读 + 2 写**（读旧数据、读旧校验、写新数据、写新校验）→ 随机写慢 → **数据库选 RAID 10**。
**RAID 不能替代备份**——它防硬件故障，防不了误删与勒索软件（错误会被同步到所有盘）。

---

## 🧭 十种寻址方式（CO06）

```mermaid
graph LR
    A[寻址方式] --> A1["立即寻址<br>EA 无 操作数在指令中<br>最快"]
    A --> A2["直接寻址<br>EA = A"]
    A --> A3["间接寻址<br>EA = M A  访存两次"]
    A --> A4["寄存器寻址<br>操作数在 Ri  不访存"]
    A --> A5["寄存器间接<br>EA = Ri"]
    A --> A6["相对寻址<br>EA = PC + A ⚠️PC 已自增"]
    A --> A7["基址寻址<br>EA = BR + A  重定位用"]
    A --> A8["变址寻址<br>EA = IX + A  数组循环用"]
    A --> A9["堆栈寻址<br>SP 隐含"]
    A --> A10["隐含寻址<br>操作数在 ACC"]

    style A6 fill:#ffcdd2
    style A7 fill:#fff9c4
    style A8 fill:#fff9c4
```

**基址 vs 变址易混点**：基址寄存器由**操作系统**填、面向**程序重定位**，形式地址是偏移；变址寄存器由**用户程序**改、面向**数组循环**，形式地址是数组首址。

---

## 🧱 程序的机器级代码表示（CO06 · 2023 考纲明列）

```mermaid
flowchart TB
    subgraph HIGH["高地址"]
        A["调用者的栈帧"]
        B["第 7 个及以后的参数<br/>（前 6 个走寄存器）"]
        C["【返回地址】<br/>★ call 指令自动压入"]
    end
    subgraph FRAME["被调用者的栈帧"]
        D["保存的旧 %rbp ← %rbp 帧底（固定）"]
        E["被调用者保存的寄存器"]
        F["局部变量 -8(%rbp) -16(%rbp)…"]
        G["为下一层准备的参数 ← %rsp 栈顶（浮动）"]
    end
    A --> B --> C --> D --> E --> F --> G

    style C fill:#ffcdd2
    style D fill:#fff9c4
```

| 机制 | 内容 |
|---|---|
| **`call`** | ① `push %rip`（压**返回地址**）② `jmp 目标` |
| **`ret`** | `pop %rip`（从**栈顶**弹出返回地址） |
| **参数传递** | `%rdi` → `%rsi` → `%rdx` → `%rcx` → `%r8` → `%r9`，第 7 个起**压栈** |
| **返回值** | `%rax` |
| **序言** | `push %rbp; mov %rsp,%rbp; sub $N,%rsp` |
| **尾声** | `leave`（= `mov %rbp,%rsp; pop %rbp`）`; ret` |
| **调用者保存** | `rax rcx rdx rsi rdi r8-r11` —— 被调用者可随意改 |
| **被调用者保存** | `rbx rbp r12-r15` —— 用之前必须存、返回前恢复 |

```mermaid
flowchart LR
    IF["if-else"] --> C1["cmp 设标志 + 条件跳转<br/>★ 跳转条件与 C 判断【相反】"]
    SW["switch"] --> C2["case 密集 → 【跳转表】O(1)<br/>case 稀疏 → 一串 cmp/je  O(n)"]
    LP["while / for"] --> C3["都编译成【do-while】形式<br/>前置判断一次 + 末尾跳回<br/>每轮少一次跳转"]

    style C2 fill:#fff9c4
    style C3 fill:#c8e6c9
```

> ⚠️ **三个必踩陷阱**：
> ① **AT&T `mov 源,目的`（有 `%$` 前缀）vs Intel `mov 目的,源`——顺序相反**。`cmpl %esi,%edi` 算的是 `edi - esi`。
> ② **有符号比较用 `jg/jl`，无符号用 `ja/jb`**。
> ③ **`lea` 只算地址不访存**，常被编译器当乘加指令用：`leaq 4(%rax,%rbx,8),%rdx` = `rdx = rax + 8*rbx + 4`。

---

## ⏱️ 指令周期与流水线（CO07）

```mermaid
graph LR
    IC["指令周期"] --> MC1["机器周期 1<br>取指"]
    IC --> MC2["机器周期 2<br>间址"]
    IC --> MC3["机器周期 3<br>执行"]
    IC --> MC4["机器周期 4<br>中断"]
    MC1 --> T["时钟周期<br>节拍 最小时间单位"]

    style IC fill:#e1bee7
    style T fill:#c8e6c9
```

```mermaid
gantt
    title 五段流水线时空图（4 条指令）
    dateFormat X
    axisFormat %s
    section 指令1
    IF :0, 1
    ID :1, 2
    EX :2, 3
    MEM:3, 4
    WB :4, 5
    section 指令2
    IF :1, 2
    ID :2, 3
    EX :3, 4
    MEM:4, 5
    WB :5, 6
    section 指令3
    IF :2, 3
    ID :3, 4
    EX :4, 5
    MEM:5, 6
    WB :6, 7
    section 指令4
    IF :3, 4
    ID :4, 5
    EX :5, 6
    MEM:6, 7
    WB :7, 8
```

**公式**：`k` 段流水线执行 `n` 条指令，`T流水 = (k + n - 1) × Δt`；`加速比 S = (n × k × Δt) / T流水`；`效率 E = S / k`。

---

## ⚠️ 流水线三大冒险（CO07）

```mermaid
flowchart TD
    H{冒险类型}
    H -->|多条指令争同一部件| S["结构冒险<br>解决: 分离指令/数据 Cache<br>或 停顿一拍"]
    H -->|后条指令要用前条的结果| D["数据冒险<br>RAW 写后读 ⭐最常见<br>WAR 读后写 · WAW 写后写"]
    H -->|分支指令改变 PC| C["控制冒险<br>解决: 分支预测 · 延迟槽<br>提前判断分支"]

    D --> D1["转发 forwarding<br>不经寄存器直接送 ALU<br>解决大部分 RAW"]
    D --> D2["load-use 冒险<br>转发也救不了<br>必须停顿 1 拍"]

    style D fill:#ffcdd2
    style D2 fill:#ffcdd2
    style C fill:#fff9c4
```

---

## 🔌 三种 I/O 方式对比（CO08）

```mermaid
flowchart TD
    Q{CPU 参与程度}
    Q -->|CPU 死等 轮询状态位| P["程序查询<br>CPU 利用率最低<br>一次传 1 字"]
    Q -->|设备就绪后打断 CPU| I["中断驱动<br>CPU 执行中断服务程序<br>一次传 1 字 · 指令级切换"]
    Q -->|专用控制器搬数据| D["DMA ⭐<br>CPU 只在传送前后介入<br>一次传 1 个数据块 · 周期级挪用"]

    P --> P1[适合慢速设备/简单系统]
    I --> I1[适合中速设备/键盘鼠标]
    D --> D1[适合高速块设备/磁盘 网卡]

    style D fill:#c8e6c9
    style P fill:#ffcdd2
```

| 维度 | 中断 | DMA |
|---|---|---|
| 数据通路 | 经过 CPU 寄存器 | **不经过 CPU**，设备 ↔ 主存直连 |
| 传送单位 | 字 / 字节 | 数据块 |
| 响应时机 | 一条**指令执行结束**后 | 一个**机器周期结束**后（更快） |
| CPU 介入 | 每次传送都要执行中断服务程序 | 只在传送**开始与结束**时 |
| 优先级 | 低于 DMA | **高于中断** |

---

## 🎓 建议学习顺序

```mermaid
graph LR
    A[CO01<br>全局观] --> B[CO02-CO03<br>数据与运算]
    B --> C[CO04-CO05<br>存储与 Cache]
    C --> D[CO06-CO07<br>指令与 CPU]
    D --> E[CO08<br>总线与 IO]
    E --> F[九类计算题<br>各刷 10 道]

    style A fill:#c8e6c9
    style C fill:#ffcdd2
    style F fill:#e1bee7
```

本科的关键不是"读懂"而是"**算对**"——计算题清单见 [INDEX.md 路径 C](./INDEX.md#路径-c只突击计算题1-周)。
