# Linux / 操作系统深度课程 · 总目录

> 面向 Go / 后端 / SRE 工程师的 Linux 内核与系统编程进阶，共 20 篇万字长文
> 每篇约 10000-15000 字，含底层原理、内核机制、C / Go / shell 代码示例、性能诊断、生产实践、陷阱清单与练习题
> 适合从"会敲 Linux 命令"到"懂内核、能排障、会调优"的系统性进阶
>
> **📅 内容基准：2026 年 6 月**——Linux **6.12 LTS**（2024-11）为生产主流基线、**6.15+** 为最新；**EEVDF 调度器**（6.6 取代 CFS fair class）、**sched_ext / BPF 可编程调度**（6.12）、**MGLRU**（多代 LRU）、**io_uring** 成熟、**cgroup v2** 全面主流（systemd 默认）、**folio** 内存重构、**BBRv3**、**XDP / eBPF** 与 **bpftrace / bcc** 主流、**PSI** 压力指标。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| L01 | [精通 Linux 架构与系统调用](./L01-精通-Linux-架构与系统调用.md) | ⭐⭐⭐ | 用户态/内核态 / syscall / vDSO / ABI / 内核子系统 / strace |
| L02 | [精通进程与线程模型](./L02-精通-进程与线程模型.md) | ⭐⭐⭐⭐ | task_struct / fork-clone-exec / COW / 进程状态机 / 线程组 / 僵尸孤儿 |
| L03 | [精通 CPU 调度：CFS 到 EEVDF](./L03-精通-CPU-调度-CFS-到-EEVDF.md) | ⭐⭐⭐⭐ | 调度类 / vruntime / EEVDF / sched_ext / cgroup 配额 / 调度延迟 |
| L04 | [精通虚拟内存与分页](./L04-精通-虚拟内存与分页.md) | ⭐⭐⭐⭐ | 地址空间 / 多级页表 / TLB / 缺页 / mmap / HugePage / THP |
| L05 | [精通物理内存管理与回收](./L05-精通-物理内存管理与回收.md) | ⭐⭐⭐⭐ | 伙伴系统 / slab / page cache / kswapd / 水位线 / MGLRU / swap / PSI |
| L06 | [精通 OOM 与内存诊断](./L06-精通-OOM-与内存诊断.md) | ⭐⭐⭐⭐ | OOM killer / oom_score / cgroup memory / 泄漏排查 / 内存火焰图 |
| L07 | [精通 VFS 与文件系统](./L07-精通-VFS-与文件系统.md) | ⭐⭐⭐⭐ | VFS / inode / dentry / ext4 / xfs / 日志 / page cache 回写 / fsync |
| L08 | [精通文件描述符与 I/O 多路复用](./L08-精通-IO-多路复用.md) | ⭐⭐⭐⭐ | fd 表 / 五种 I/O 模型 / select-poll-epoll / LT-ET / 惊群 |
| L09 | [精通 io_uring 与异步 I/O](./L09-精通-io_uring-与异步IO.md) | ⭐⭐⭐⭐⭐ | Linux AIO / SQ-CQ 环 / 注册缓冲 / SQPOLL / vs epoll |
| L10 | [精通块设备与 I/O 调度](./L10-精通-块设备与IO调度.md) | ⭐⭐⭐⭐ | 块层 / bio / blk-mq / mq-deadline-bfq-kyber / NVMe / iostat |
| L11 | [精通 Linux 网络协议栈](./L11-精通-Linux-网络协议栈.md) | ⭐⭐⭐⭐⭐ | 收发包路径 / sk_buff / NAPI / 软中断 / RSS-RPS-RFS / GRO / XDP |
| L12 | [精通 TCP/IP 内核实现与调优](./L12-精通-TCP-IP-内核实现与调优.md) | ⭐⭐⭐⭐⭐ | TCP 状态机 / cubic-BBR / 缓冲区 / accept 队列 / TIME_WAIT / sysctl |
| L13 | [精通 Socket 与连接管理](./L13-精通-Socket-与连接管理.md) | ⭐⭐⭐⭐ | socket 内核路径 / SO_REUSEPORT / 握手全链路 / conntrack / ss-tcpdump |
| L14 | [精通信号与进程间通信](./L14-精通-信号与IPC.md) | ⭐⭐⭐⭐ | 信号(可靠/实时) / 异步信号安全 / pipe / 共享内存 / eventfd / signalfd |
| L15 | [精通内核同步与 futex](./L15-精通-内核同步与futex.md) | ⭐⭐⭐⭐⭐ | 原子操作 / 自旋锁 / RCU / 内存屏障 / futex / 惊群 |
| L16 | [精通 Namespace](./L16-精通-Namespace.md) | ⭐⭐⭐⭐ | 8 种 namespace / clone-setns-unshare / 手搓容器 / user ns / rootless |
| L17 | [精通 Cgroup v2 与资源控制](./L17-精通-Cgroup-v2.md) | ⭐⭐⭐⭐ | v1 vs v2 / 统一层级 / cpu-memory-io-pids / PSI / systemd 联动 |
| L18 | [精通性能诊断方法论与工具](./L18-精通-性能诊断方法论与工具.md) | ⭐⭐⭐⭐ | USE/RED / load average / /proc / perf / 火焰图 / strace / 瓶颈四象限 |
| L19 | [精通 eBPF 深度实战](./L19-精通-eBPF.md) | ⭐⭐⭐⭐⭐ | eBPF 虚拟机 / verifier / map / kprobe-uprobe-tracepoint-XDP / bpftrace |
| L20 | [精通 systemd 与启动流程](./L20-精通-systemd-与启动流程.md) | ⭐⭐⭐⭐ | UEFI / initramfs / systemd / unit-target / cgroup 集成 / journald |

---

## 🗺️ 按模块组织

### 🟢 模块一：基石——架构 / 进程 / 调度（L01-L03）

> "进程在 CPU 上跑"这句话，每个词都值得拆到内核里看一遍。

- **L01 架构与系统调用**：用户态 / 内核态的边界、`syscall` 指令与中断门、vDSO 如何让 `gettimeofday` 不陷入内核、ABI 与寄存器传参、内核子系统全景图、strace / ltrace 的 ptrace 原理
- **L02 进程与线程模型**：`task_struct` 关键字段、`fork` / `vfork` / `clone3` 的统一、COW 写时复制、线程是"共享地址空间的 task"、进程状态机（R/S/D/Z/T）、`D` 状态不可中断之谜、僵尸与孤儿、进程组 / 会话 / 控制终端
- **L03 CPU 调度**：调度类层级（stop / dl / rt / fair / idle）、CFS 的红黑树与 vruntime、**EEVDF**（6.6 起取代 CFS 选取逻辑）的 lag / deadline 模型、**sched_ext**（6.12，用 BPF 写调度器）、nice / 权重、CPU 亲和与 NUMA、cgroup CPU 配额（`cpu.max`）、调度延迟诊断（`schedstat` / `perf sched` / runqlat）

### 🔵 模块二：内存（L04-L06）

> 后端进程 99% 的"诡异问题"——延迟毛刺、OOM、swap 颠簸——根子都在内存。

- **L04 虚拟内存与分页**：64 位地址空间布局（text/data/heap/mmap/stack）、4 级 / 5 级页表、页表项标志位、TLB 与 TLB shootdown、缺页中断（minor / major）、`mmap` 文件映射与匿名映射、demand paging、HugePage（显式）与 THP（透明大页）的取舍
- **L05 物理内存管理与回收**：伙伴系统（buddy）与碎片、slab / slub 分配器、page cache 与 `free` 的 buff/cache 之谜、kswapd 与直接回收、三条水位线（min/low/high）、**MGLRU** 多代 LRU、swap 与 swappiness、内存压力 **PSI**
- **L06 OOM 与内存诊断**：OOM killer 选择算法、`oom_score` / `oom_score_adj`、cgroup v2 `memory.max` / `memory.high` 与容器内 OOM、RSS / PSS / USS、`/proc/meminfo` 逐行解读、内存泄漏排查（`pmap` / `smaps_rollup` / valgrind / heap profiler）、内存火焰图

### 🟡 模块三：文件与 I/O（L07-L10）

> 从"一个 `read()` 到底发生了什么"，到把 100 万连接喂饱的 io_uring。

- **L07 VFS 与文件系统**：VFS 四大对象（superblock / inode / dentry / file）、dentry cache、硬链接 vs 软链接、ext4 的 extent 与日志（journal）、xfs / btrfs 取舍、page cache 与脏页回写（`dirty_ratio` / writeback）、`fsync` / `fdatasync` / `O_DIRECT` / `O_SYNC` 语义
- **L08 文件描述符与 I/O 多路复用**：fd 表与 `struct file` 共享、五种 I/O 模型（阻塞 / 非阻塞 / 多路复用 / 信号驱动 / 异步）、`select` / `poll` 的 O(n) 之痛、**epoll** 的红黑树 + 就绪链表、LT vs ET、`epoll` 惊群与 `EPOLLEXCLUSIVE`
- **L09 io_uring 与异步 I/O**：Linux 传统 AIO（`libaio`）为何鸡肋、**io_uring** 的 SQ / CQ 双环、`io_uring_enter`、注册 buffer / fd、SQPOLL 内核轮询模式、与 epoll 的性能对比、安全争议与生产开关
- **L10 块设备与 I/O 调度**：块层架构、bio / request、**blk-mq** 多队列、I/O 调度器（mq-deadline / bfq / kyber / none）选型、NVMe 多队列、`iostat` / `blktrace` / `biolatency` 诊断、`%util` 的误读

### 🟣 模块四：网络（L11-L13）

> 一个包从网卡到你的 `recv()`，中间内核做了一百件事。

- **L11 Linux 网络协议栈**：收包路径（网卡 → DMA → 软中断 → 协议栈 → socket buffer）、`sk_buff` 结构、NAPI 轮询、硬中断 / 软中断（softirq / ksoftirqd）、多队列网卡 RSS / RPS / RFS / XPS、GRO / GSO / TSO offload、**XDP** 与 eBPF 在驱动层的拦截
- **L12 TCP/IP 内核实现与调优**：内核态 TCP 状态机、三次握手与半连接 / 全连接队列、拥塞控制（cubic / **BBR** / BBRv3）、收发缓冲区自动调整、`tcp_tw_reuse` 与 TIME_WAIT、Nagle 与 delayed ACK、核心 sysctl 调优清单
- **L13 Socket 与连接管理**：`socket()` / `bind()` / `listen()` / `accept()` 的内核路径、**SO_REUSEPORT** 多进程负载均衡、`accept` 惊群、conntrack 连接跟踪与表满、`ss` / `tcpdump` / `tcpconnect`（eBPF）抓包与排障

### 🟠 模块五：IPC 与内核同步（L14-L15）

> 多个执行流要"说话"和"不打架"，靠的就是这两章。

- **L14 信号与进程间通信**：信号语义（可靠 / 不可靠 / 实时信号）、`signal` vs `sigaction`、信号在 `task_struct` 的投递、异步信号安全函数、`SIGCHLD` 与 reap、pipe / FIFO / System V & POSIX 共享内存 / 消息队列 / 信号量、`eventfd` / `signalfd` / `timerfd`
- **L15 内核同步与 futex**：原子操作与 CAS、自旋锁 / 顺序锁（seqlock）/ 互斥锁、**RCU**（读多写少的杀手锏）、内存屏障与重排、**futex**（用户态锁的内核支撑，glibc mutex 怎么来的）、惊群问题全景

### 🔴 模块六：容器与隔离（L16-L17）

> 容器不是虚拟机——它就是被 namespace 和 cgroup "圈养"的普通进程。

- **L16 Namespace**：8 种 namespace（mnt / pid / net / ipc / uts / user / cgroup / time）、`clone` / `setns` / `unshare`、**用 50 行 C 手搓一个容器**、PID namespace 的 init 语义、user namespace 与 rootless 容器、`/proc/[pid]/ns`
- **L17 Cgroup v2 与资源控制**：cgroup v1 的混乱与 v2 的统一层级、controllers（cpu / memory / io / pids）、`cpu.max` / `memory.max` / `io.max`、cgroup 与 **PSI** 联动、systemd 如何用 cgroup 管理服务、容器运行时（runc / containerd）怎么落地 limit

### ⚫ 模块七：可观测与生产（L18-L20）

> 没有度量就没有优化——这三章把前面所有知识"接到生产线上"。

- **L18 性能诊断方法论与工具**：Brendan Gregg 的 **USE** 方法与 **RED** 方法、load average 到底算什么（含 D 状态）、`/proc` 与 `/sys` 宝库、`perf`（采样 / 火焰图 / `perf sched` / `perf stat`）、`strace` / `ltrace`、CPU / 内存 / I/O / 网络瓶颈四象限定位法
- **L19 eBPF 深度实战**：eBPF 虚拟机与 JIT、**verifier**（为什么你的程序被拒）、map 类型、程序类型（kprobe / uprobe / tracepoint / XDP / tc / LSM）、**bpftrace** 单行神器、bcc 工具箱、CO-RE 与 BTF、生产可观测（Cilium / Pixie / Parca）
- **L20 systemd 与启动流程**：UEFI → bootloader（GRUB / systemd-boot）→ initramfs → kernel → systemd 全链路、unit 类型（service / socket / timer / mount / target）、依赖与排序、systemd 与 cgroup 集成、journald 日志、service 排错（`systemctl` / `journalctl` / `systemd-analyze`）

---

## 🎯 学习路径建议

### 路径 A：完整通学（3-4 个月）

按编号顺序，每周 1-2 篇。每篇配套：
1. 在本机 / 虚拟机 / 容器里复现示例（`/proc`、`strace`、`perf`、`bpftrace`）
2. 对照 `man7.org` 手册页与内核文档
3. 做练习题与 QUIZ
4. 拿一个真实线上问题把知识点串起来

### 路径 B：后端应用开发者切入（1-2 个月）

> 目标：看懂自己程序"在系统层面"发生了什么。

- **L01 架构与系统调用** → **L02 进程与线程**（理解 runtime 之下）
- **L08 I/O 多路复用** → **L09 io_uring**（理解 Go/Node/Nginx 的事件循环）
- **L12 TCP 调优** → **L13 Socket**（连接数 / 延迟问题）
- **L06 OOM 诊断**（容器为什么被杀）
- **L18 性能诊断**（学会自己定位瓶颈）

### 路径 C：SRE / 性能工程师（2-3 个月）

- **L03 调度** + **L04-L06 内存全套**（资源是 SRE 的命）
- **L10 块设备 I/O** + **L07 文件系统**（存储瓶颈）
- **L11-L13 网络全套**（最高频的线上问题）
- **L17 Cgroup**（容器资源限额排障）
- **L18 诊断方法论** + **L19 eBPF**（现代可观测的天花板）
- **L20 systemd**（服务为什么起不来）

### 路径 D：网络专精（1 个月）

- **L11 协议栈**（包怎么进来）
- **L12 TCP/IP 调优**（连接与拥塞）
- **L13 Socket / conntrack**（连接管理与排障）
- 配合 **L19 eBPF** 的 XDP / tc 章节 + cloud-native C03（K8s 网络）

### 路径 E：容器底层专精（1 个月）

- **L02 进程**（容器就是进程）
- **L16 Namespace** + **L17 Cgroup**（手搓容器）
- **L07 VFS**（overlayfs 的根基）
- **L13 网络命名空间**（容器网络）
- 配合 cloud-native C01（Docker/OCI）、C06（资源管理）

### 路径 F：eBPF / 可观测工程师（3-4 周）

- **L18 性能诊断方法论**（先建立心智模型）
- **L01 系统调用** + **L02 进程**（probe 挂哪里）
- **L19 eBPF 深度实战**（核心）
- 配合 **L11 网络栈**（XDP / tc）+ cloud-native C11（K8s 可观测）

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)
- **权威文档**：
  - [Linux Kernel Documentation](https://docs.kernel.org/)
  - [man7.org（Linux man-pages）](https://man7.org/linux/man-pages/)
  - [The Linux Kernel Map](https://makelinux.github.io/kernel/map/)
  - [kernel.org](https://www.kernel.org/)
- **必读书 / 站**：
  - Brendan Gregg, *Systems Performance (2nd)* 与 [brendangregg.com](https://www.brendangregg.com/)（火焰图 / USE / bcc 作者）
  - *Understanding the Linux Kernel*、*Linux Kernel Development (Robert Love)*
  - *The Linux Programming Interface (Michael Kerrisk)*——系统编程圣经
  - [LWN.net](https://lwn.net/)——跟踪内核最新变化
- **eBPF 生态**：
  - [bpftrace](https://github.com/bpftrace/bpftrace) / [bcc](https://github.com/iovisor/bcc)
  - [ebpf.io](https://ebpf.io/) / [Cilium](https://cilium.io/)

---

## 🛠️ 工具速查

| 任务 | 工具 / 命令 |
|---|---|
| 进程 / 线程 | `ps` / `top` / `htop` / `pidstat` / `/proc/[pid]/` |
| 调度延迟 | `perf sched` / `runqlat`（bcc）/ `schedstat` / `chrt` |
| CPU 剖析 | `perf top` / `perf record -g` / 火焰图 / `pidstat -u` |
| 内存 | `free` / `vmstat` / `/proc/meminfo` / `pmap` / `smem` / `slabtop` |
| OOM | `dmesg`(查 OOM)/ `oom_score_adj` / cgroup `memory.events` |
| 文件 / FS | `lsof` / `df` / `du` / `stat` / `filefrag` / `iostat` |
| I/O | `iostat -x` / `iotop` / `blktrace` / `biolatency`（bcc） |
| 网络 | `ss` / `ip` / `tcpdump` / `ethtool` / `nstat` / `tcpconnect`(bcc) |
| 系统调用 | `strace` / `ltrace` / `perf trace` |
| 通用 trace | `perf` / `ftrace` / `bpftrace` / `bcc` / `trace-cmd` |
| cgroup / ns | `systemd-cgls` / `systemd-cgtop` / `lsns` / `nsenter` / `unshare` |
| 压力 / 负载 | `stress-ng` / `fio`（磁盘）/ `iperf3` / `sysbench` |
| 启动 / 服务 | `systemctl` / `journalctl` / `systemd-analyze` / `dmesg` |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 解释一次 `read()` 系统调用从用户态到磁盘再返回的完整路径
- [ ] 说清 vDSO 为什么能让 `clock_gettime` 不陷入内核
- [ ] 画出 `fork` 后父子进程的内存（COW）演变
- [ ] 解释 EEVDF 相比 CFS 改了什么，以及 `nice` 如何影响调度
- [ ] 用 `runqlat` / `perf sched` 定位一次调度延迟毛刺
- [ ] 区分 minor / major fault，并解释 THP 可能带来的延迟毛刺
- [ ] 读懂 `/proc/meminfo`，解释 `MemAvailable` 怎么算
- [ ] 还原一次容器 OOM：从 `dmesg` 到 cgroup `memory.events`
- [ ] 讲清 epoll 的红黑树 + 就绪链表，以及 LT/ET 的差异与陷阱
- [ ] 写一个最小 io_uring 程序并解释 SQ/CQ 环
- [ ] 描述一个包从网卡 DMA 到 `recv()` 的全链路（含 NAPI / softirq）
- [ ] 解释全连接 / 半连接队列溢出的现象与 sysctl 调法
- [ ] 比较 BBR 与 cubic 的拥塞控制思路
- [ ] 用 50 行 C（clone + namespace）跑起一个"容器"
- [ ] 解释 cgroup v2 `cpu.max` / `memory.high` 如何限流
- [ ] 用 USE 方法对一台高负载机器做系统化体检
- [ ] 写一个 bpftrace 单行统计某 syscall 的延迟分布
- [ ] 用 `systemd-analyze` 找出拖慢开机的服务

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **L01 系统调用** | `syscall` 指令为主（非 `int 0x80`）；vDSO / vvar 标配；io_uring 改变"syscall 即陷入"的传统认知 |
| **L03 调度** | **EEVDF 自 6.6 取代 CFS 的挑选逻辑**（仍属 fair class）；**sched_ext（6.12 GA）允许用 BPF 写调度器**；`SCHED_DEADLINE` 在实时场景普及 |
| **L05 内存回收** | **MGLRU（多代 LRU）自 6.1 可用、渐成默认**，回收更精准；folio 重构持续推进；PSI 成为压力信号事实标准 |
| **L06 OOM** | cgroup v2 `memory.high`（软限流）+ `memory.max`（硬限）+ `memory.events`；systemd-oomd 用户态 OOM（按 PSI） |
| **L09 io_uring** | 已成高性能 I/O 首选；部分发行版 / 云因安全默认收紧（`io_uring_disabled` sysctl）；网络 io_uring（多 accept / zero-copy）成熟 |
| **L10 块 I/O** | blk-mq 是唯一路径（单队列 legacy 已删）；NVMe 普及；`none` 调度器在 NVMe 上常见 |
| **L11 网络栈** | XDP / eBPF 在 DDoS 防护、L4 LB（Katran / Cilium）大规模落地；GRO / GSO 默认 |
| **L12 TCP** | **BBR 广泛部署、BBRv3 推进**；`tcp_tw_recycle` 早已删除；默认 cubic |
| **L16 Namespace** | time namespace（5.6+）已稳定；user namespace + rootless（Podman）主流；cgroup namespace 标配 |
| **L17 Cgroup** | **cgroup v2 全面主流，systemd 默认 unified**；v1 进入维护；PSI 驱动的弹性伸缩 |
| **L19 eBPF** | CO-RE + BTF 让"一次编译到处运行"成真；eBPF LSM、sched_ext、bpftrace 2.x；可观测（Pixie / Parca / Coroot）崛起 |
| **L20 systemd** | systemd 250+ 普及；systemd-boot / UKI（统一内核镜像）兴起；systemd-oomd / homed / 网络栈扩张 |
