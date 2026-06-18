# Linux / 操作系统路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月**——Linux 6.12 LTS / 6.15+、EEVDF、sched_ext、MGLRU、io_uring、cgroup v2、BBR、XDP/eBPF、PSI。

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 Linux 内核之旅]) --> M1[模块 1: 基石 架构/进程/调度]

    M1 --> L01[L01 架构与系统调用]
    M1 --> L02[L02 进程与线程]
    M1 --> L03[L03 CPU 调度 CFS→EEVDF]

    L03 --> M2[模块 2: 内存]
    M2 --> L04[L04 虚拟内存与分页]
    M2 --> L05[L05 物理内存与回收]
    M2 --> L06[L06 OOM 与诊断]

    L06 --> M3[模块 3: 文件与 I/O]
    M3 --> L07[L07 VFS 与文件系统]
    M3 --> L08[L08 I/O 多路复用]
    M3 --> L09[L09 io_uring]
    M3 --> L10[L10 块设备与 I/O 调度]

    L10 --> M4[模块 4: 网络]
    M4 --> L11[L11 网络协议栈]
    M4 --> L12[L12 TCP/IP 调优]
    M4 --> L13[L13 Socket 与连接]

    L13 --> M5[模块 5: IPC 与同步]
    M5 --> L14[L14 信号与 IPC]
    M5 --> L15[L15 内核同步与 futex]

    L15 --> M6[模块 6: 容器与隔离]
    M6 --> L16[L16 Namespace]
    M6 --> L17[L17 Cgroup v2]

    L17 --> M7[模块 7: 可观测与生产]
    M7 --> L18[L18 性能诊断方法论]
    M7 --> L19[L19 eBPF 深度实战]
    M7 --> L20[L20 systemd 与启动]

    L20 --> End([懂内核 / 能排障 / 会调优])

    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef base fill:#48bb78,stroke:#2f855a,color:#fff
    classDef mem fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef io fill:#ecc94b,stroke:#b7791f,color:#000
    classDef net fill:#9f7aea,stroke:#6b46c1,color:#fff
    classDef ipc fill:#ed8936,stroke:#c05621,color:#fff
    classDef ct fill:#f56565,stroke:#c53030,color:#fff
    classDef obs fill:#38b2ac,stroke:#234e52,color:#fff

    class M1,M2,M3,M4,M5,M6,M7 module
    class L01,L02,L03 base
    class L04,L05,L06 mem
    class L07,L08,L09,L10 io
    class L11,L12,L13 net
    class L14,L15 ipc
    class L16,L17 ct
    class L18,L19,L20 obs
```

---

## 🟢 模块 1：基石——架构 / 进程 / 调度（L01-L03）

```mermaid
graph LR
    L01[L01 架构/系统调用<br>用户态↔内核态] --> L02[L02 进程/线程<br>task_struct/clone]
    L02 --> L03[L03 调度<br>EEVDF/sched_ext]

    classDef base fill:#48bb78,stroke:#2f855a,color:#fff
    class L01,L02,L03 base
```

**用户态 → 内核态的几条路**：

```mermaid
flowchart TD
    App[用户态进程] -->|syscall 指令| Sys["系统调用<br>陷入内核 ring0"]
    App -->|vDSO 调用| VDSO["vDSO<br>不陷入，用户态读 vvar"]
    App -->|缺页/除零| Trap["异常 / fault"]
    HW[硬件设备] -->|IRQ| INT["硬中断 → 软中断"]

    Sys --> Kernel[内核处理]
    Trap --> Kernel
    INT --> Kernel

    style VDSO fill:#48bb78,color:#fff
    style Sys fill:#4299e1,color:#fff
    style INT fill:#ed8936,color:#fff
```

**进程状态机**：

```mermaid
stateDiagram-v2
    [*] --> R: fork/clone
    R --> S: 等待事件(可中断)
    R --> D: 等待 I/O(不可中断)
    S --> R: 事件就绪/信号
    D --> R: I/O 完成
    R --> T: SIGSTOP
    T --> R: SIGCONT
    R --> Z: exit()
    Z --> [*]: 父进程 wait() 回收

    note right of D
        D 状态收不到信号
        kill -9 也杀不掉
        常见于卡 I/O
    end note
```

---

## 🔵 模块 2：内存（L04-L06）

```mermaid
graph TB
    VA["虚拟地址"] -->|MMU + 页表| PA["物理地址"]
    VA -.miss.-> Fault["缺页中断"]
    Fault -->|匿名页| Anon["分配物理页"]
    Fault -->|文件页| PageCache["page cache"]

    PA --> Buddy["伙伴系统<br>物理页框"]
    Buddy --> Slab["slab/slub<br>内核小对象"]

    Pressure{"内存压力?"} -->|low 水位| Kswapd["kswapd 后台回收"]
    Pressure -->|min 水位| Direct["直接回收(阻塞分配)"]
    Kswapd --> Reclaim["MGLRU 回收<br>文件页/匿名页→swap"]
    Direct --> Reclaim
    Reclaim -->|回收不够| OOM["OOM killer"]

    style PageCache fill:#4299e1,color:#fff
    style OOM fill:#f56565,color:#fff
    style Kswapd fill:#48bb78,color:#fff
```

**缺页中断分类**：

```mermaid
flowchart TD
    PF{缺页类型}
    PF -->|页在内存，仅未建映射| Minor["minor fault<br>快，无磁盘 I/O"]
    PF -->|页在磁盘/swap| Major["major fault<br>慢，触发磁盘 I/O"]
    PF -->|非法访问| Seg["SIGSEGV<br>段错误"]

    style Minor fill:#48bb78,color:#fff
    style Major fill:#ecc94b,color:#000
    style Seg fill:#f56565,color:#fff
```

---

## 🟡 模块 3：文件与 I/O（L07-L10）

```mermaid
graph TB
    App[应用 read/write] --> VFS["VFS 抽象层<br>file/inode/dentry"]
    VFS --> FS["具体文件系统<br>ext4/xfs/btrfs"]
    VFS -.命中.-> PC["page cache"]
    FS --> Block["块层 blk-mq"]
    Block --> Sched["I/O 调度器<br>mq-deadline/bfq/none"]
    Sched --> Driver["NVMe/SCSI 驱动"]
    Driver --> Disk[(物理磁盘)]

    PC -.脏页回写.-> Block

    style VFS fill:#ecc94b,color:#000
    style PC fill:#4299e1,color:#fff
    style Block fill:#9f7aea,color:#fff
```

**五种 I/O 模型与 epoll → io_uring 演进**：

```mermaid
flowchart LR
    Block["阻塞 I/O"] --> NonBlock["非阻塞轮询"]
    NonBlock --> Mux["I/O 多路复用<br>select/poll/epoll"]
    Mux --> SigIO["信号驱动 I/O"]
    SigIO --> AIO["异步 I/O"]
    AIO --> Uring["io_uring<br>SQ/CQ 环 + 真异步"]

    style Mux fill:#ecc94b,color:#000
    style Uring fill:#48bb78,color:#fff
```

**I/O 调度器选型**：

```mermaid
flowchart TD
    Disk{设备类型}
    Disk -->|NVMe SSD 高速| None["none / kyber<br>少调度，靠硬件队列"]
    Disk -->|SATA SSD/HDD 混合| MQD["mq-deadline<br>读优先 + 防饿死"]
    Disk -->|桌面/延迟敏感| BFQ["bfq<br>按进程公平 + 低延迟"]

    style None fill:#48bb78,color:#fff
    style MQD fill:#4299e1,color:#fff
    style BFQ fill:#9f7aea,color:#fff
```

---

## 🟣 模块 4：网络（L11-L13）

```mermaid
graph TB
    NIC[网卡] -->|DMA 写入 ring buffer| Ring["RX ring"]
    Ring -->|硬中断| HardIRQ["硬中断<br>仅触发"]
    HardIRQ -->|触发 softirq| NAPI["NAPI 轮询<br>ksoftirqd"]
    NAPI --> Stack["协议栈<br>IP → TCP/UDP"]
    Stack --> SkBuff["sk_buff 入 socket 接收队列"]
    SkBuff --> Recv["进程 recv() 取走"]

    NAPI -.可选.-> XDP["XDP/eBPF<br>驱动层提前拦截"]

    Stack --> RPS["RPS/RFS<br>软件分流到多核"]

    style NAPI fill:#9f7aea,color:#fff
    style XDP fill:#48bb78,color:#fff
    style Stack fill:#4299e1,color:#fff
```

**TCP 三次握手与两个队列**：

```mermaid
sequenceDiagram
    participant C as Client
    participant SYNQ as 半连接队列<br>(SYN queue)
    participant ACCQ as 全连接队列<br>(accept queue)
    participant App as 服务端 accept()

    C->>SYNQ: SYN
    SYNQ-->>C: SYN+ACK (进半连接队列)
    C->>ACCQ: ACK (握手完成→移入全连接队列)
    App->>ACCQ: accept() 取走连接
    Note over SYNQ,ACCQ: 半连接满→SYN 丢弃<br>全连接满→看 tcp_abort_on_overflow
```

---

## 🟠 模块 5：IPC 与同步（L14-L15）

```mermaid
graph LR
    subgraph IPC 手段
    Pipe["pipe/FIFO"]
    SHM["共享内存<br>最快"]
    MQ["消息队列"]
    Sem["信号量"]
    Sig["信号"]
    EFD["eventfd/signalfd"]
    end

    subgraph 同步原语
    Atomic["原子操作/CAS"]
    Spin["自旋锁"]
    Mutex["互斥锁(futex)"]
    RCU["RCU 读多写少"]
    end

    Atomic --> Spin --> Mutex
    Mutex -.用户态快路径.-> Futex["futex 仅竞争时陷入"]

    style SHM fill:#48bb78,color:#fff
    style RCU fill:#9f7aea,color:#fff
    style Futex fill:#ed8936,color:#fff
```

---

## 🔴 模块 6：容器与隔离（L16-L17）

```mermaid
graph TB
    Proc["普通进程"] -->|clone + flags| NS["8 种 Namespace<br>视图隔离"]
    Proc -->|加入 cgroup| CG["Cgroup v2<br>资源限额"]

    NS --> Mnt["mnt 文件系统"]
    NS --> Pid["pid 进程号"]
    NS --> Net["net 网络栈"]
    NS --> User["user UID 映射"]

    CG --> CPU["cpu.max"]
    CG --> Mem["memory.max/high"]
    CG --> IO["io.max"]

    NS & CG --> Container["= 容器<br>(runc/containerd)"]

    style Container fill:#f56565,color:#fff
    style NS fill:#4299e1,color:#fff
    style CG fill:#48bb78,color:#fff
```

---

## ⚫ 模块 7：可观测与生产（L18-L20）

```mermaid
graph TB
    Target["待诊断系统"] --> USE{"USE 方法"}
    USE --> Util["Utilization 使用率"]
    USE --> Sat["Saturation 饱和度"]
    USE --> Err["Errors 错误"]

    Target --> Tools["工具层"]
    Tools --> Proc["/proc /sys"]
    Tools --> Perf["perf 采样/火焰图"]
    Tools --> Strace["strace 系统调用"]
    Tools --> BPF["eBPF/bpftrace<br>动态插桩"]

    BPF --> Probe["kprobe/uprobe<br>tracepoint/XDP"]

    Target --> Boot["systemd"]
    Boot --> Journal["journald 日志"]
    Boot --> Analyze["systemd-analyze 启动耗时"]

    style USE fill:#38b2ac,color:#fff
    style BPF fill:#9f7aea,color:#fff
    style Perf fill:#ed8936,color:#fff
```

**性能瓶颈四象限定位**：

```mermaid
flowchart TD
    Slow{系统慢?}
    Slow -->|CPU 高| CPU["perf top / 火焰图<br>找热点函数"]
    Slow -->|等待多/load 高但 CPU 闲| IO["iostat/biolatency<br>查 I/O 等待(D 状态)"]
    Slow -->|内存紧/swap| MEM["free/vmstat/PSI<br>查回收与 OOM"]
    Slow -->|网络延迟| NET["ss/tcpdump/tcplife<br>查重传与队列"]

    style CPU fill:#f56565,color:#fff
    style IO fill:#ecc94b,color:#000
    style MEM fill:#4299e1,color:#fff
    style NET fill:#9f7aea,color:#fff
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（3-4 个月）

```mermaid
gantt
    title 3-4 个月完整 Linux 内核通学
    dateFormat YYYY-MM-DD
    section 月 1
    L01-L03 基石(进程/调度)    :a1, 2026-06-15, 21d
    section 月 2
    L04-L10 内存+文件+I/O      :a2, after a1, 35d
    section 月 3
    L11-L17 网络+IPC+容器      :a3, after a2, 35d
    section 月 4
    L18-L20 诊断+eBPF+systemd  :a4, after a3, 21d
```

### 路径 C：SRE / 性能工程师

```mermaid
graph LR
    P1[L03 调度] --> P2[L04-L06 内存]
    P2 --> P3[L10 块 I/O]
    P3 --> P4[L11-L13 网络]
    P4 --> P5[L17 Cgroup]
    P5 --> P6[L18 诊断]
    P6 --> P7[L19 eBPF]

    style P2 fill:#4299e1,color:#fff
    style P7 fill:#9f7aea,color:#fff
```

### 路径 F：eBPF / 可观测工程师

```mermaid
graph LR
    F1[L18 诊断方法论] --> F2[L01 系统调用]
    F2 --> F3[L02 进程]
    F3 --> F4[L19 eBPF]
    F4 --> F5[L11 网络栈 XDP/tc]

    style F4 fill:#9f7aea,color:#fff
```

---

## 🧠 Linux 核心知识思维导图

```mermaid
mindmap
  root((Linux 内核))
    基石
      架构/系统调用 L01
      进程/线程 L02
      CPU 调度 L03
    内存
      虚拟内存 L04
      物理回收 L05
      OOM L06
    文件与IO
      VFS L07
      多路复用 L08
      io_uring L09
      块IO L10
    网络
      协议栈 L11
      TCP调优 L12
      Socket L13
    并发
      信号IPC L14
      同步futex L15
    容器
      Namespace L16
      Cgroup L17
    可观测
      诊断方法 L18
      eBPF L19
      systemd L20
```

---

## 📊 难度与重要性矩阵

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| L01 架构/系统调用 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 一切的入口 |
| L02 进程/线程 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 心智模型基石 |
| L03 CPU 调度 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 延迟问题根因 |
| L04 虚拟内存 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 内存问题前提 |
| L05 物理回收 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | swap/抖动根因 |
| L06 OOM 诊断 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 容器最高频事故 |
| L07 VFS/FS | ⭐⭐⭐⭐ | 🔥🔥🔥 | 存储基础 |
| L08 I/O 多路复用 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 高并发服务核心 |
| L09 io_uring | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 2026 高性能首选 |
| L10 块 I/O | ⭐⭐⭐⭐ | 🔥🔥🔥 | 存储瓶颈 |
| L11 网络栈 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 包路径必懂 |
| L12 TCP 调优 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 最高频线上问题 |
| L13 Socket | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 连接管理 |
| L14 信号/IPC | ⭐⭐⭐⭐ | 🔥🔥🔥 | 优雅退出/通信 |
| L15 同步/futex | ⭐⭐⭐⭐⭐ | 🔥🔥🔥 | 锁的本质 |
| L16 Namespace | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 容器底层 |
| L17 Cgroup | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 资源限额排障 |
| L18 诊断方法论 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 串起所有知识 |
| L19 eBPF | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 现代可观测天花板 |
| L20 systemd | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 服务管理日常 |

---

## 🔗 与已有课程的关系

| Linux 章节 | 关联已有课程 |
|---|---|
| L01 系统调用 | golang/G 系列 runtime、backend 通用 |
| L03 调度 | cloud-native/C06（CPU Requests/Limits/QoS） |
| L05/L06 内存/OOM | cloud-native/C06（memory limit）、redis/postgresql 内存调优 |
| L08 I/O 多路复用 | golang netpoller、redis 性能模型、backend I/O 模型 |
| L09 io_uring | backend 高性能 I/O、数据库引擎 |
| L11-L13 网络 | backend/B01-B02（网络/HTTP）、cloud-native/C03（K8s 网络）、microservices 通信 |
| L12 TCP 调优 | backend/B25（反向代理）、kafka/redis 网络层 |
| L16 Namespace | cloud-native/C01（Docker/OCI） |
| L17 Cgroup | cloud-native/C06（资源管理）、C01（容器） |
| L18 诊断 | backend/B24（可观测性）、golang/G22（pprof） |
| L19 eBPF | cloud-native/C03（Cilium）、C10（Mesh）、C11（可观测） |

---

## 🆕 2026 关键技术演进

```mermaid
graph LR
    subgraph 调度器
    S1[O1 调度器] --> S2[CFS 2.6.23]
    S2 --> S3[EEVDF 6.6]
    S3 --> S4["sched_ext 6.12<br>BPF 写调度器"]
    end

    subgraph 内存回收
    M1[传统双 LRU] --> M2["MGLRU 6.1<br>多代 LRU"]
    M2 --> M3["folio 重构<br>持续推进"]
    end

    subgraph I/O
    I1[阻塞/epoll] --> I2[libaio 鸡肋]
    I2 --> I3["io_uring 5.1+<br>真异步"]
    I3 --> I4["网络 io_uring<br>zero-copy"]
    end

    subgraph 拥塞控制
    C1[Reno] --> C2[CUBIC 默认]
    C2 --> C3[BBR]
    C3 --> C4[BBRv3]
    end

    subgraph 容器/可观测
    E1[cgroup v1] --> E2["cgroup v2<br>unified 主流"]
    E2 --> E3["eBPF CO-RE+BTF"]
    E3 --> E4["Cilium/Pixie/Parca"]
    end

    style S4 fill:#fff3e0
    style M2 fill:#fff3e0
    style I3 fill:#fff3e0
    style C3 fill:#fff3e0
    style E2 fill:#fff3e0
```
