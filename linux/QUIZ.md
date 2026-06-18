# Linux / 操作系统深度课程 · 测验题与答案

> 配合 [INDEX.md](./INDEX.md) 与 [ROADMAP.md](./ROADMAP.md) 使用
>
> **📅 内容基准：2026 年 6 月**——Linux 6.12 LTS / 6.15+、EEVDF、sched_ext、MGLRU、io_uring、cgroup v2、BBR、XDP/eBPF、PSI。

---

## 📖 使用说明

- 每章 **10 道题**，按难度递进：⭐ 基础概念 → ⭐⭐ 原理入门 → ⭐⭐⭐ 常规实战 → ⭐⭐⭐⭐ 复杂排障 → ⭐⭐⭐⭐⭐ 系统级综合
- 题型混合：单选、简答、场景题、命令 / 代码编写
- 答案与详解放在每章末尾的 `<details>` 折叠块里——**请先独立作答，再展开对照**
- 通过标准：**每章 ≥ 7 题正确**；全部通过即可挑战末尾的 **🏆 综合实战题**
- 推荐节奏：读完对应章节后当天做题；一周后回头复习错题

---

## L01 · 精通 Linux 架构与系统调用

1. （⭐）进入内核态有哪几种途径？
2. （⭐）什么是 vDSO？为什么经它的 `clock_gettime` 可以不陷入内核？
3. （⭐⭐）x86-64 上 `syscall` 指令如何传参与返回？`errno` 是怎么来的？
4. （⭐⭐）`int 0x80` 与 `syscall` 指令有何区别？为何现代用后者？
5. （⭐⭐⭐）`strace` 基于什么内核机制？为什么它会显著拖慢被跟踪进程？
6. （⭐⭐⭐）glibc 的 syscall wrapper 起什么作用？
7. （⭐⭐⭐⭐）一个程序大量调用 `gettimeofday` 导致 CPU 偏高，如何优化？原理是什么？
8. （⭐⭐⭐⭐）怎样确认某进程当前卡在哪个系统调用上？
9. （⭐⭐⭐⭐⭐）为什么说 io_uring 改变了「syscall 即陷入」的传统认知？
10. （⭐⭐⭐⭐⭐）系统调用号在不同 CPU 架构间是否一致？syscall ABI 稳定性意味着什么？

<details>
<summary>📝 L01 答案与详解</summary>

1. **系统调用（`syscall` 指令）、异常/陷阱（缺页、除零）、硬件中断**。三者都切到 ring0 由内核处理；vDSO 调用是例外——它在用户态完成，不陷入。
2. vDSO 是内核映射到每个进程地址空间的一小段代码 + `vvar` 共享数据页。`clock_gettime` 直接读 vvar 里内核维护的时间，**无需陷入**，省掉上下文切换开销。
3. 参数依次放 `rdi/rsi/rdx/r10/r8/r9`，调用号放 `rax`，`syscall` 指令进入内核；返回值在 `rax`。glibc wrapper 把负的返回值转成 `errno` + 返回 -1。
4. `int 0x80` 是软中断方式，开销大、走中断门；`syscall`/`sysenter` 是专用快速指令，省去中断处理流程，现代 x86-64 用 `syscall`。
5. 基于 **ptrace**：每次系统调用进出都让被跟踪进程停下、通知 tracer，上下文切换密集，故开销大、不适合生产高频路径。
6. wrapper 封装寄存器约定、`syscall` 指令、errno 转换，并提供 POSIX 风格接口；部分还走 vDSO 或加缓存。
7. 确认是否已走 vDSO（`clock_gettime(CLOCK_MONOTONIC)` 通常走）；若用了陷入内核的时钟源，换 vDSO 友好的时钟，或减少调用频率/缓存时间戳。
8. `cat /proc/<pid>/stack`、`cat /proc/<pid>/syscall`，或 `strace -p`（短时）、`perf trace`；D 状态多半卡在某 I/O syscall。
9. io_uring 用共享内存环批量提交/完成 I/O，**一次 `io_uring_enter` 可处理大量操作甚至零系统调用（SQPOLL）**，打破「每次 I/O 一次陷入」的模型（见 [L09](./L09-精通-io_uring-与异步IO.md)）。
10. **不一致**：系统调用号按架构定义。ABI 稳定意味着内核保证老的系统调用号/语义不变，老二进制无需重编译即可在新内核运行。

</details>

---

## L02 · 精通进程与线程模型

1. （⭐）在 Linux 内核眼里，进程和线程的本质区别是什么？
2. （⭐）`fork` 之后父子进程共享内存吗？COW 是什么？
3. （⭐⭐）`task_struct` 里 `pid` 与 `tgid` 的区别？多线程进程里它们如何取值？
4. （⭐⭐）进程状态 `D`（不可中断睡眠）意味着什么？为什么 `kill -9` 杀不掉？
5. （⭐⭐⭐）僵尸进程是怎么产生的？如何避免？孤儿进程被谁收养？
6. （⭐⭐⭐）`clone` 通过哪些 flag 决定「造线程」还是「造进程」？
7. （⭐⭐⭐⭐）容器主进程为何是 PID 1？这会带来什么信号/回收问题？
8. （⭐⭐⭐⭐）load average 很高但 CPU 很闲，最可能是什么原因？怎么定位？
9. （⭐⭐⭐⭐⭐）`pidfd` 解决了传统按 pid 发信号的什么竞态问题？
10. （⭐⭐⭐⭐⭐）如何查看一个进程打开的所有 fd、内存映射和线程？

<details>
<summary>📝 L02 答案与详解</summary>

1. **都是 task_struct**。线程是与同进程其他线程**共享地址空间（mm_struct）、fd 表等**的 task；进程是不共享这些的 task。Linux 不区分二者的调度实体。
2. 不直接共享，但通过 **COW（写时复制）**：fork 后父子共享物理页且标记只读，任一方写时才复制一份。省去整份拷贝。
3. `tgid` 是线程组 ID（= 主线程 pid，用户态看到的「进程 PID」）；`pid` 是每个线程独有的内核任务 ID。`getpid()` 返回 tgid。
4. 进程在等待**不可被信号打断**的资源（典型是磁盘 I/O）。此时不处理任何信号，故 `kill -9` 也要等 I/O 返回才生效。长期 D 状态常意味着 I/O 卡死（见 [L18](./L18-精通-性能诊断方法论与工具.md)）。
5. 子进程退出后父进程没 `wait()` 回收，残留 task_struct（占 PID）即僵尸。避免：父进程 `wait`/`waitpid`，或处理 `SIGCHLD`。孤儿（父先死）被 init/subreaper 收养并回收。
6. `CLONE_VM`（共享地址空间）、`CLONE_FILES`、`CLONE_THREAD`、`CLONE_SIGHAND` 等都置上 → 线程；都不置 → 进程（类似 fork）。
7. PID namespace 内第一个进程是 PID 1，承担 init 语义：**默认忽略无 handler 的信号、需负责 reap 子进程**。后果：不处理 SIGTERM 则 `docker stop` 要等超时 SIGKILL；不 reap 则僵尸堆积。用 tini/dumb-init 解决（见 [L16](./L16-精通-Namespace.md)）。
8. 大量进程处于 **D 状态**（I/O 等待也计入 load）。用 `ps -eo state,...`/`vmstat`（看 `b` 列）、`iostat` 确认 I/O 瓶颈，`/proc/<pid>/stack` 看卡在哪。
9. 按 pid 发信号有 **pid 重用竞态**（目标退出、pid 被复用，信号误发给新进程）。`pidfd` 是对进程的稳定引用，`pidfd_send_signal` 不会误伤。
10. `ls /proc/<pid>/fd`、`cat /proc/<pid>/maps`（或 `pmap`）、`ls /proc/<pid>/task`（线程）。

</details>

---

## L03 · 精通 CPU 调度：CFS 到 EEVDF

1. （⭐）Linux 有哪几个调度类？它们的优先级顺序是怎样的？
2. （⭐）`nice` 值的范围和含义？数值越小代表什么？
3. （⭐⭐）CFS 用什么数据结构挑选下一个运行的任务？`vruntime` 是什么？
4. （⭐⭐）EEVDF（6.6）相比 CFS 改了什么？它对延迟敏感任务为何更友好？
5. （⭐⭐⭐）`SCHED_FIFO`/`SCHED_RR`/`SCHED_DEADLINE` 各适合什么场景？
6. （⭐⭐⭐）`cpu.max` 的 quota/period 如何限制 CPU？与 CFS bandwidth 什么关系？
7. （⭐⭐⭐⭐）容器 CPU 用量没到 limit 却周期性卡顿，根因和定位方法？
8. （⭐⭐⭐⭐）`cpu.stat` 里 `nr_throttled`/`throttled_usec` 说明什么？
9. （⭐⭐⭐⭐⭐）如何把延迟敏感进程绑到特定 CPU 并隔离干扰？
10. （⭐⭐⭐⭐⭐）sched_ext（6.12）带来了什么新能力？

<details>
<summary>📝 L03 答案与详解</summary>

1. `stop > deadline(dl) > realtime(rt) > fair(CFS/EEVDF) > idle`。高优先类有任务就先跑。
2. -20 ~ +19，**越小优先级越高**（抢得到更多 CPU）。它影响 CFS 的权重。
3. **红黑树**按 `vruntime`（虚拟运行时间，按权重缩放的已运行时间）排序，每次挑 vruntime 最小者。nice 小→权重大→vruntime 增长慢→获得更多 CPU。
4. EEVDF 引入 **lag 与 virtual deadline**，按「eligibility + 最早 deadline」挑选，对要求低延迟、短时间片的任务响应更及时；可配 `latency_nice` 表达延迟偏好。仍属 fair class。
5. FIFO/RR 是固定优先级实时调度（音视频/控制）；DEADLINE 按 runtime/period/deadline 三参数做 EDF（硬实时周期任务）。普通服务用 fair。
6. `cpu.max = "quota period"`：每个 period 给 quota 微秒额度，用尽则 throttle 到下个 period。底层就是 **CFS bandwidth**（见 [L17](./L17-精通-Cgroup-v2.md)）。
7. 多线程在 period 前段并行把 quota 烧光，后段全员饿等→毛刺。定位：看 `cpu.stat` 的 `nr_throttled`；根治：让运行时感知 limit（Go `automaxprocs`、JVM `ActiveProcessorCount`）或调大 limit。
8. 该 cgroup 因 CPU 配额用尽被限流的 period 数与累计被冻结时长——是 CPU throttling 的直接证据。
9. `taskset`/`sched_setaffinity` 绑核 + `isolcpus`/`cpuset.cpus` 隔离 + 关掉该核的 irq（见 [L11](./L11-精通-Linux-网络协议栈.md)），减少抢占与中断干扰。
10. 允许**用 BPF 程序实现自定义 CPU 调度器**（scx），便于针对特定工作负载实验调度策略，无需改内核重编译。

</details>

---

## L04 · 精通虚拟内存与分页

1. （⭐）`malloc` 1GB 立即占用 1GB 物理内存吗？为什么？
2. （⭐）VSZ 与 RSS 的区别？
3. （⭐⭐）多级页表解决了什么问题？TLB 的作用是什么？
4. （⭐⭐）minor fault 与 major fault 的区别？哪个慢、为什么？
5. （⭐⭐⭐）`mmap` 的文件映射与匿名映射分别用于什么？
6. （⭐⭐⭐）COW 在 fork 和 mmap MAP_PRIVATE 中如何体现？
7. （⭐⭐⭐⭐）数据库为什么常建议关闭 THP（透明大页）？
8. （⭐⭐⭐⭐）HugePage 对 TLB 有什么好处？什么场景用显式大页？
9. （⭐⭐⭐⭐⭐）一个进程 RSS 持续增长，如何判断是堆、mmap 还是共享内存？
10. （⭐⭐⭐⭐⭐）TLB shootdown 是什么？为什么多核改页表可能引发性能抖动？

<details>
<summary>📝 L04 答案与详解</summary>

1. **不会**。`malloc` 只分配虚拟地址空间，物理页在首次写入触发缺页时按需分配（demand paging）。
2. VSZ 是虚拟地址空间大小（含未实际占用的映射）；RSS 是实际驻留物理内存的部分。看内存占用要看 RSS（更准确用 PSS）。
3. 多级页表按需分配页表项，避免为整个 64 位空间建巨大单级表。TLB 缓存「虚拟页→物理页」映射，命中则免去多级页表遍历。
4. minor fault：页已在内存（如已在 page cache、或 COW），只需建映射，快；major fault：页在磁盘/swap，需磁盘 I/O，慢。
5. 文件映射：把文件内容映射进地址空间（按需载入、可共享）；匿名映射：堆/大块分配（无后备文件，换出走 swap）。
6. 都用「共享只读 + 写时复制一份私有页」：fork 后父子页只读共享；MAP_PRIVATE 映射写入时复制，不回写原文件。
7. THP 自动合并大页可能带来分配延迟、内存碎片、`khugepaged` 抖动；数据库随机访问模式下收益小、毛刺大，故常设 `madvise` 或关闭。
8. 大页让一个 TLB 项覆盖更大内存，**减少 TLB miss**。大内存、访问密集且地址集中的场景（如大堆 JVM、数据库 buffer pool）用显式 HugePage。
9. `cat /proc/<pid>/smaps_rollup` 看 anon（堆/匿名）、file（文件映射）、shmem（共享）分项；`pmap -x` 看各段（见 [L06](./L06-精通-OOM-与内存诊断.md)）。
10. 改/撤页表项时需让其他 CPU 的 TLB 失效（IPI 通知），即 shootdown。频繁 munmap/改映射的多核程序会因大量 IPI 抖动。

</details>

---

## L05 · 精通物理内存管理与回收

1. （⭐）`free` 输出里 buff/cache 占了很多，是「内存不够」吗？
2. （⭐）`MemAvailable` 与 `MemFree` 有什么区别？
3. （⭐⭐）伙伴系统（buddy）解决什么问题？外碎片是什么？
4. （⭐⭐）slab/slub 分配器为什么存在？它分配什么？
5. （⭐⭐⭐）kswapd 后台回收与直接回收（direct reclaim）的区别？后者为何危险？
6. （⭐⭐⭐）三条水位线 min/low/high 各触发什么行为？
7. （⭐⭐⭐⭐）`vm.swappiness` 调大/调小分别影响什么？延迟敏感服务怎么设？
8. （⭐⭐⭐⭐）MGLRU（多代 LRU）相比传统双链表 LRU 改进了什么？
9. （⭐⭐⭐⭐⭐）脏页回写参数 `dirty_ratio`/`dirty_background_ratio` 配置不当会怎样？
10. （⭐⭐⭐⭐⭐）PSI 的 `some` 与 `full` 含义？如何用它判断内存压力？

<details>
<summary>📝 L05 答案与详解</summary>

1. **不是**。page cache（buff/cache）可在需要时回收，属于「可用」内存。判断内存够不够看 `MemAvailable`，不是 `MemFree`。
2. `MemFree` 是完全空闲；`MemAvailable` 是估算「在不触发 swap 的前提下可供新分配使用」的量（含可回收的 cache）。后者更有意义。
3. buddy 管理物理页框的分配/合并，按 2 的幂成块。外碎片：空闲页总量够但凑不出连续大块，导致大页/连续分配失败，靠 compaction 缓解。
4. 内核频繁分配小对象（dentry、inode、task_struct 等），slab/slub 在页之上做对象级缓存，减少碎片与分配开销。`slabtop` 可看。
5. kswapd 在后台异步回收（低水位触发）；直接回收发生在分配路径上、**阻塞当前进程**直到回收出内存，造成延迟毛刺。
6. 低于 high 时 kswapd 开始后台回收；触到 low 加紧回收；触到 min 触发直接回收/可能 OOM。
7. swappiness 大→更倾向换出匿名页（省内存但增延迟）；小→尽量不 swap。延迟敏感服务常设很小甚至 0/1，或限制 cgroup `memory.swap.max`。
8. MGLRU 用多个「代」更精细地区分页的冷热与访问新鲜度，回收决策更准、扫描更省，尾延迟更稳（6.1+ 渐成默认）。
9. 太高→脏页攒太多，集中回写时 I/O 风暴 + 延迟尖峰；太低→频繁小回写降吞吐。需按存储能力与延迟要求权衡。
10. `some`=至少一个任务因内存停滞的时间占比；`full`=所有任务都停滞。`/proc/pressure/memory` 或 cgroup `memory.pressure` 持续升高即内存吃紧（见 [L17](./L17-精通-Cgroup-v2.md)）。

</details>

---

## L06 · 精通 OOM 与内存诊断

1. （⭐）容器显示 `OOMKilled`，是宿主内存不够吗？
2. （⭐）`oom_score_adj` 的作用？怎样让某进程更/更不容易被 OOM 杀？
3. （⭐⭐）`memory.max` 与 `memory.high` 触发 OOM 的行为差异？
4. （⭐⭐）RSS、PSS、USS 的区别？统计「真实占用」该看哪个？
5. （⭐⭐⭐）如何从 `dmesg` 读懂一次 OOM kill 的决策？
6. （⭐⭐⭐）systemd-oomd 与内核 OOM killer 有何不同？基于什么决策？
7. （⭐⭐⭐⭐）一个进程 RSS 不大却让容器 OOM，可能是什么内存被计入了？
8. （⭐⭐⭐⭐）排查 Go/C 服务内存泄漏，分别用什么工具？
9. （⭐⭐⭐⭐⭐）完整还原一次容器 OOM：从现象到根因的步骤？
10. （⭐⭐⭐⭐⭐）`MemAvailable` 充足但仍 OOM，可能的原因？

<details>
<summary>📝 L06 答案与详解</summary>

1. **不一定**。多为 **cgroup 内 OOM**：进程用量超过 `memory.max`，与宿主总内存无关。看 `memory.events` 的 `oom_kill`。
2. 它调整 OOM 评分：`+1000` 最易被杀，`-1000` 几乎不被杀。给关键进程设负值、给可牺牲的设正值。
3. `max` 是硬限，超且回收不动 → 直接在 cgroup 内 OOM kill；`high` 是软限，超过只是节流+积极回收（拖慢不杀）。
4. RSS 含共享页（多进程重复计）；PSS 把共享页按比例摊分；USS 是进程独占部分。比较「真实占用」用 PSS（`smaps_rollup`）。
5. dmesg 打印候选进程的 RSS/oom_score、被选中的 victim、触发的 cgroup/全局上下文，据此判断谁、因何被杀。
6. systemd-oomd 在**用户态**、基于 **PSI** 压力**提前**动作（压力抬头就杀），优于内核 OOM 的「最后一刻」；可按 cgroup 配置策略。
7. socket 缓冲（`sock`）、内核 slab（dentry/inode 缓存）、tmpfs/shmem 都计入 cgroup memory。看 `memory.stat`（见 [L17](./L17-精通-Cgroup-v2.md)/[L13](./L13-精通-Socket-与连接管理.md)）。
8. Go：`pprof` heap profile；C/C++：valgrind/massif、ASAN、jemalloc/tcmalloc 的 heap profiling；通用：`pmap`/`smaps`、eBPF 记录分配栈。
9. ①`dmesg`/`kubectl describe` 确认 OOMKilled；②看 cgroup `memory.events`/`memory.max`/`memory.current`；③`memory.stat` 看内存去向；④定位泄漏或调大 limit / 用 `memory.high` 软限。
10. cgroup 限额（非全局）、大页/连续内存分配失败、内核内存耗尽、`vm.overcommit` 策略、或瞬时峰值超限。

</details>

---

## L07 · 精通 VFS 与文件系统

1. （⭐）VFS 的四大核心对象是什么？
2. （⭐）硬链接与符号链接的本质区别？
3. （⭐⭐）一次 `open()`+`read()` 在内核里经过哪些对象（fd→？→？）？
4. （⭐⭐）ext4 日志的 `data=ordered/journal/writeback` 三种模式有何取舍？
5. （⭐⭐⭐）page cache 的脏页在什么时机回写？哪些参数控制？
6. （⭐⭐⭐）`fsync`、`fdatasync`、`O_DIRECT`、`O_SYNC` 的语义差异？
7. （⭐⭐⭐⭐）overlayfs 中修改一个 lowerdir 的文件会发生什么（copy-up）？
8. （⭐⭐⭐⭐）磁盘空间没满却报 `No space left`，可能是什么？怎么查？
9. （⭐⭐⭐⭐⭐）数据库 `fsync` 慢导致写延迟，如何定位是文件系统还是设备？
10. （⭐⭐⭐⭐⭐）dentry/inode 缓存膨胀对内存有什么影响？

<details>
<summary>📝 L07 答案与详解</summary>

1. **superblock（文件系统实例）、inode（文件元数据）、dentry（目录项/路径）、file（打开的文件实例）**。
2. 硬链接是指向同一 inode 的多个目录项（同设备、不能跨文件系统、不能链目录）；软链接是存目标路径的独立 inode（可跨设备、可悬空）。
3. `fd → struct file → struct inode（→ 具体 fs 操作）`，命中则走 page cache，否则下到块层（见 [L10](./L10-精通-块设备与IO调度.md)）。
4. ordered（默认，只日志元数据、数据先落盘）最均衡；journal（数据也进日志）最安全最慢；writeback（不保证数据顺序）最快但崩溃可能旧数据。
5. 脏页超过 `dirty_background_ratio` 时后台 flusher 开始回写；超过 `dirty_ratio` 时写进程被阻塞同步回写；或 `fsync`/定时触发。
6. `fsync` 刷数据+元数据；`fdatasync` 只刷数据（省一次元数据写）；`O_DIRECT` 绕过 page cache 直达设备；`O_SYNC` 每次写都同步落盘。
7. **copy-up**：把该文件从只读 lowerdir 复制到可写 upperdir 再改，原 lower 不变（见 [cloud-native C01](../cloud-native/C01-精通-Docker-与-OCI.md)）。
8. **inode 耗尽**（大量小文件）：`df -i` 看 inode 使用率；或有进程占着已删除的大文件（`lsof | grep deleted`）。
9. `fsync` 慢多在设备落盘：用 `biolatency`/`iostat -x` 看设备 `await`（见 [L10](./L10-精通-块设备与IO调度.md)/[L19](./L19-精通-eBPF.md)）；文件系统层用 `filefrag` 看碎片、查 journal 模式。
10. 它们是内核 slab 内存，计入（cgroup）内存；海量小文件/路径遍历会让 dentry 缓存膨胀，甚至撞 `memory.max`（见 [L06](./L06-精通-OOM-与内存诊断.md)）。

</details>

---

## L08 · 精通文件描述符与 I/O 多路复用

1. （⭐）fd 是什么？多个 fd 能指向同一个 `struct file` 吗？
2. （⭐）五种 I/O 模型分别是什么？
3. （⭐⭐）`select`/`poll` 为什么是 O(n)？瓶颈在哪？
4. （⭐⭐）`epoll` 为什么高效？它的红黑树和就绪链表各干什么？
5. （⭐⭐⭐）LT 与 ET 的区别？ET 编程为什么必须配非阻塞 + 循环读尽？
6. （⭐⭐⭐）epoll 惊群是什么？`EPOLLEXCLUSIVE` 怎么解决？
7. （⭐⭐⭐⭐）ET 模式下「漏读」bug 是怎么产生的？
8. （⭐⭐⭐⭐）服务 fd 持续增长（fd 泄漏），如何排查？
9. （⭐⭐⭐⭐⭐）Go 的 netpoller 与 epoll 是什么关系？
10. （⭐⭐⭐⭐⭐）要支撑百万连接，系统层面要做哪些准备？

<details>
<summary>📝 L08 答案与详解</summary>

1. fd 是进程级的整数索引，指向 `struct file`。`dup`/`fork` 后多个 fd 可共享同一 `struct file`（共享文件偏移）。
2. 阻塞、非阻塞、I/O 多路复用（select/poll/epoll）、信号驱动、异步（AIO/io_uring）。
3. 每次调用都要把整个 fd 集合从用户态拷进内核、再线性扫描所有 fd——fd 越多越慢，且每次重复传入。
4. `epoll_ctl` 把 fd 注册进红黑树（增删查 O(log n)）；就绪事件经回调挂到**就绪链表**，`epoll_wait` 只返回就绪的，与总 fd 数无关。
5. LT：只要可读就一直通知；ET：仅在状态变化（新数据到达）时通知一次。ET 必须一次把数据读尽（循环 read 到 EAGAIN），否则剩余数据不再触发→漏读。
6. 多个 epoll 等同一 listen fd，一个事件唤醒全部→空转。`EPOLLEXCLUSIVE`（4.5+）让内核只唤醒一个。
7. ET 下读到一部分就停（没读到 EAGAIN），剩余数据不再产生新事件，连接「卡住」直到下次新数据到来。
8. `ls /proc/<pid>/fd | wc -l`、`lsof -p <pid>`；多为忘记 close（含异常路径）或连接未释放（见 [L13](./L13-精通-Socket-与连接管理.md) CLOSE_WAIT）。
9. Go runtime 内部用 epoll（Linux）实现 netpoller：goroutine 阻塞在网络 I/O 时让出，事件就绪由 netpoller 唤醒——用户写同步代码，底层是 epoll 事件循环。
10. 调大 `nofile`/`fs.file-max`、内存（每连接缓冲）、`somaxconn`/`tcp_max_syn_backlog`、端口范围、用 epoll/io_uring + 多核（SO_REUSEPORT，见 [L13](./L13-精通-Socket-与连接管理.md)）。

</details>

---

## L09 · 精通 io_uring 与异步 I/O

1. （⭐）io_uring 主要解决 epoll 的什么局限？
2. （⭐）SQ 环与 CQ 环各是什么？
3. （⭐⭐）为什么传统 Linux AIO（libaio）被认为「鸡肋」？
4. （⭐⭐）SQPOLL 模式如何做到「几乎零系统调用」提交 I/O？
5. （⭐⭐⭐）注册 buffer / 注册 fd 带来什么好处？
6. （⭐⭐⭐）链式请求（IOSQE_IO_LINK）适合什么场景？
7. （⭐⭐⭐⭐）网络 io_uring 的 multishot 是什么？
8. （⭐⭐⭐⭐）io_uring 的安全争议是什么？为何有 `io_uring_disabled`？
9. （⭐⭐⭐⭐⭐）什么工作负载用 io_uring 收益最大？
10. （⭐⭐⭐⭐⭐）已有 epoll 服务，是否值得迁 io_uring？怎么权衡？

<details>
<summary>📝 L09 答案与详解</summary>

1. epoll 只解决「就绪通知」，真正的 `read`/`write` 仍是同步系统调用；io_uring 让**提交与完成都异步、批量**，覆盖文件 I/O（epoll 对普通文件无效）。
2. 共享内存中的两个环形队列：SQ（提交队列，用户填请求）、CQ（完成队列，内核填结果），通过 mmap 共享，减少拷贝与陷入。
3. 只支持 `O_DIRECT`、接口怪异、很多情况下仍会阻塞，覆盖面窄，故少有人用。
4. 内核起一个 poll 线程轮询 SQ，用户只管往 SQ 填请求、无需 `io_uring_enter`，从而提交路径零系统调用。
5. 预注册免去每次提交时的 fd 查找与 buffer 映射/校验开销，高频 I/O 提升明显。
6. 有顺序依赖的操作（如 open→read→close、或 write 后 fsync）串成链，一次提交、内核按序执行。
7. 一次提交、多次完成（如 multishot accept/recv）：一个 SQE 持续产出 CQE，省去反复重新提交，适合高并发收包/接受连接。
8. 它提供了强大的内核异步能力，历史上出现过若干内核漏洞；部分发行版/云默认收紧，`io_uring_disabled` sysctl 可禁用。
9. 高并发、小 I/O、高 IOPS 的存储/网络服务（数据库、代理、对象存储），批量与零拷贝收益最大。
10. 若 epoll 已够用且无瓶颈，迁移收益有限且有复杂度/兼容性成本；若 I/O 系统调用本身是瓶颈、或需文件异步 I/O，才值得迁。

</details>

---

## L10 · 精通块设备与 I/O 调度

1. （⭐）`iostat` 里 `%util` 100% 一定代表磁盘满载吗？
2. （⭐）bio 是什么？
3. （⭐⭐）blk-mq 多队列解决了单队列的什么瓶颈？
4. （⭐⭐）`none`/`mq-deadline`/`bfq`/`kyber` 各适合什么设备/场景？
5. （⭐⭐⭐）`iostat -x` 的 `await`、`aqu-sz`、`r_await` 含义？
6. （⭐⭐⭐）NVMe SSD 为什么常用 `none` 调度器？
7. （⭐⭐⭐⭐）定位磁盘高延迟，用哪些工具、看哪些指标？
8. （⭐⭐⭐⭐）readahead（预读）调大调小分别影响什么？
9. （⭐⭐⭐⭐⭐）如何用 cgroup `io.max` 限制某服务的磁盘带宽？
10. （⭐⭐⭐⭐⭐）`%util` 高但应用不慢，怎么解释？

<details>
<summary>📝 L10 答案与详解</summary>

1. **不一定**。对多队列/NVMe 设备，`%util` 只表示「有 I/O 在途的时间占比」，并行能力强时 100% 仍可能远未饱和。要结合 `await`/吞吐判断。
2. block I/O 的基本单位，描述一段连续的磁盘读写请求（页/扇区映射），向上接文件系统、向下接驱动。
3. 单队列锁竞争严重（多核都抢一把队列锁）；blk-mq 按 CPU/硬件队列分摊，发挥 NVMe 多队列并行（单队列 legacy 已删）。
4. none：高速 NVMe（少调度靠硬件）；mq-deadline：通用、读优先防饿死（HDD/SATA SSD）；bfq：桌面/延迟敏感按进程公平；kyber：高速设备的轻量延迟目标调度。
5. `await`=平均 I/O 完成时间（含排队）；`aqu-sz`=平均队列深度；`r_await`/`w_await`=读/写各自延迟。
6. NVMe 内部高度并行、硬件队列深，软件再排序收益小反增开销，`none` 直通最快。
7. `iostat -x`（await/util）、`biolatency`/`biosnoop`（bcc，延迟分布与明细，见 [L19](./L19-精通-eBPF.md)）；区分设备侧慢还是队列深。
8. 调大利于顺序大文件吞吐（少 I/O 次数），但随机访问浪费带宽、占 cache；随机负载宜小。
9. 写 `io.max`：`echo "<major:minor> wbps=<字节/秒>" > <cgroup>/io.max`（见 [L17](./L17-精通-Cgroup-v2.md)）。
10. 设备并行处理能力强（NVMe），虽长期有 I/O 在途（util 高），但每个 I/O 延迟低、吞吐充足，应用感知不到慢。

</details>

---

## L11 · 精通 Linux 网络协议栈

1. （⭐）一个数据包从网卡到 `recv()` 大致经过哪些阶段？
2. （⭐）`sk_buff` 是什么？
3. （⭐⭐）NAPI 为什么用「中断 + 轮询」混合？硬中断与软中断如何分工？
4. （⭐⭐）RSS、RPS、RFS 的区别？
5. （⭐⭐⭐）GRO/GSO/TSO 是什么？对 `tcpdump` 抓包有何影响？
6. （⭐⭐⭐）`ksoftirqd` 某核 CPU 打满，如何缓解？
7. （⭐⭐⭐⭐）qdisc 是什么？`fq`/`fq_codel` 解决什么？
8. （⭐⭐⭐⭐）XDP 在协议栈的哪个位置？能做什么？
9. （⭐⭐⭐⭐⭐）网卡多队列如何与 CPU 亲和配合？
10. （⭐⭐⭐⭐⭐）排查丢包，从网卡到协议栈看哪些计数？

<details>
<summary>📝 L11 答案与详解</summary>

1. 网卡 DMA 写入 RX ring → 硬中断 → 触发软中断（NET_RX）→ NAPI 轮询收包 → 协议栈（IP/TCP）→ 放入 socket 接收队列 → 进程 `recv()` 取走。
2. 内核中表示一个网络包的结构，贯穿协议栈各层，含头部指针、数据、元信息，可克隆/共享。
3. 高速网卡中断频率极高，纯中断会中断风暴；NAPI 用一次硬中断触发后转为轮询批量收包（ksoftirqd），降低中断开销。
4. RSS：网卡硬件按哈希把包分到多队列/多核；RPS：软件层把收包分散到多核；RFS：在 RPS 基础上让包送到「正在读它的进程」所在核，提升缓存命中。
5. 收包合并（GRO）/发包分段卸载（GSO/TSO）减少协议栈处理次数；抓包看到的是合并后的「超大包」，与线上真实分段不同。
6. 开启 RPS/RFS 分散软中断到多核、网卡多队列 + 中断亲和、调 `netdev_max_backlog`、必要时 XDP 卸载（见 [L19](./L19-精通-eBPF.md)）。
7. qdisc 是发包排队规则；`fq`（公平队列，配合 BBR pacing）、`fq_codel`（对抗 bufferbloat、降低排队延迟）。
8. 在驱动收包最早点（协议栈之前）。可 DROP（DDoS 过滤）、TX（回弹）、REDIRECT（L4 LB，如 Katran/Cilium）、PASS。
9. 把每个网卡队列的中断绑到固定 CPU（smp_affinity），让收包处理局部化，减少跨核缓存抖动。
10. `ethtool -S`（网卡丢包/overrun）、`/proc/net/softnet_stat`（软中断丢包）、`nstat`/`netstat -s`（协议栈各层丢弃）。

</details>

---

## L12 · 精通 TCP/IP 内核实现与调优

1. （⭐）半连接队列与全连接队列分别在握手哪一步起作用？
2. （⭐）TIME_WAIT 为什么存在？在 Linux 上停留多久？
3. （⭐⭐）全连接队列溢出的现象与确认命令？
4. （⭐⭐）为什么 `tcp_tw_recycle` 不能用？
5. （⭐⭐⭐）BBR 与 cubic 判断拥塞的根本不同？
6. （⭐⭐⭐）单条 TCP 跑不满带宽，如何从 BDP 角度调优？
7. （⭐⭐⭐⭐）RPC 偶发稳定 40ms 延迟的根因与修法？
8. （⭐⭐⭐⭐）SYN flood 时 syncookies 如何起作用？
9. （⭐⭐⭐⭐⭐）SACK 与快速重传分别优化了什么？
10. （⭐⭐⭐⭐⭐）高并发服务器有哪些关键 sysctl，各自风险？

<details>
<summary>📝 L12 答案与详解</summary>

1. 收到 SYN 后连接进半连接队列（SYN_RECV）；收到 ACK 握手完成移入全连接队列等 `accept()`（见 [L13](./L13-精通-Socket-与连接管理.md)）。
2. 保证最后的 ACK 能重传、让旧报文消亡。Linux 固定约 60s（2MSL，`TCP_TIMEWAIT_LEN`），不可用 `tcp_fin_timeout` 改。
3. 客户端偶发连接延迟/超时；`nstat | grep ListenOverflows` 或 `netstat -s | grep -i overflow` 计数增长；`ss -lnt` 看 Recv-Q 逼近 Send-Q。
4. 它在 NAT 环境下会因时间戳判断错误而丢弃合法连接，导致 NAT 后客户端随机连不上；已于 4.12 删除。
5. cubic 基于丢包（假设丢包=拥塞）；BBR 主动测量瓶颈带宽与最小 RTT 调速，弱网/有 bufferbloat 时显著更优。
6. 估算 BDP=带宽×RTT，放开 `tcp_rmem`/`rmem_max` 上限、确认窗口缩放开启、发送端切 BBR。
7. Nagle 与 delayed ACK 互等：发送端因 Nagle 等 ACK、接收端攒 ACK 到 40ms。设 `TCP_NODELAY`。
8. 半连接队列满时不分配 request_sock，把连接信息编码进 SYN+ACK 序列号，对端回 ACK 时校验还原，扛住洪水。
9. SACK 让发送方只重传真正丢失的段（避免回退 N）；快速重传收 3 个重复 ACK 即重传、不等 RTO，恢复更快。
10. `somaxconn`/`tcp_max_syn_backlog`（队列，需配合应用）、`tcp_tw_reuse`（仅 outbound）、`tcp_rmem/wmem`（缓冲，过大耗内存）、`tcp_congestion_control`（算法）等——都应先定位再调。

</details>

---

## L13 · 精通 Socket 与连接管理

1. （⭐）`struct socket` 与 `struct sock` 各负责什么？
2. （⭐）`SO_REUSEADDR` 与 `SO_REUSEPORT` 的区别？
3. （⭐⭐）`listen(fd, backlog)` 的实际生效队列上限由什么决定？
4. （⭐⭐）`SO_REUSEPORT` 为什么能消除 accept 惊群？
5. （⭐⭐⭐）conntrack 表满的现象与日志？
6. （⭐⭐⭐）大量 `CLOSE-WAIT` 的根因？为什么调内核参数没用？
7. （⭐⭐⭐⭐）Unix domain socket 相比 TCP 回环的优势？`SCM_RIGHTS` 能做什么？
8. （⭐⭐⭐⭐）`ss -ti` 能看到哪些连接级内核指标？
9. （⭐⭐⭐⭐⭐）conntrack 表项为何会堆积？如何安全加速回收？
10. （⭐⭐⭐⭐⭐）定位「连接建立失败」该用哪些工具？

<details>
<summary>📝 L13 答案与详解</summary>

1. `struct socket` 是 BSD socket 层（关联 fd/VFS）；`struct sock` 是协议层（收发队列、状态、TCP 用 tcp_sock 扩展）。
2. REUSEADDR：复用 TIME_WAIT 地址、不同 IP 同端口（解决重启 bind 失败）；REUSEPORT：多 socket 同 addr:port + 内核负载均衡（多 worker）。
3. `min(应用传入的 backlog, net.core.somaxconn)`，两者都要调；`ss -lnt` 的 Send-Q 是生效上限。
4. 每个 worker 有独立 listen socket 与独立 accept 队列，内核按哈希把新连接均衡投递，不再唤醒一群（见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）。
5. 偶发丢包/新连接超时但 CPU/带宽不高；`dmesg` 报 `nf_conntrack: table full, dropping packet`。
6. 对端已发 FIN、本端应用没 `close()`（连接泄漏）——这是应用 bug，调内核参数无效，须修代码确保所有路径 close。
7. 不走网络栈、更快、更安全（本机）；`SCM_RIGHTS` 可通过 sendmsg 把 fd 传给另一进程（socket activation、特权分离的基础）。
8. `cwnd`、`rtt`、`retrans`、`rto`、`sacked`、缓冲占用等——定位「这条连接为什么慢」。
9. established 超时默认很长，断开的连接表项迟迟不回收；安全做法是缩短 established/time_wait 超时、调大 `nf_conntrack_max`/`buckets`，或对纯转发用 NOTRACK。
10. `ss -tanp`（状态/队列）、`tcpdump`（抓 SYN/RST）、bcc 的 `tcpconnect`/`tcpretrans`/`tcplife`（见 [L19](./L19-精通-eBPF.md)）。

</details>

---

## L14 · 精通信号与进程间通信

1. （⭐）可靠信号与不可靠信号的区别？
2. （⭐）哪两个信号不可被捕获、阻塞或忽略？
3. （⭐⭐）`signal` 与 `sigaction` 的区别？为什么推荐后者？
4. （⭐⭐）为什么信号 handler 里不能调用 `printf`/`malloc`？什么是异步信号安全函数？
5. （⭐⭐⭐）self-pipe trick / `signalfd` 解决了什么问题？
6. （⭐⭐⭐）`SIGCHLD` 与僵尸回收的关系？
7. （⭐⭐⭐⭐）服务优雅停机应如何处理 `SIGTERM`？
8. （⭐⭐⭐⭐）为什么共享内存是最快的 IPC？它的同步要靠什么？
9. （⭐⭐⭐⭐⭐）`eventfd`/`timerfd`/`pidfd` 如何统一进 epoll 事件循环？
10. （⭐⭐⭐⭐⭐）System V IPC 与 POSIX IPC 的主要区别？

<details>
<summary>📝 L14 答案与详解</summary>

1. 不可靠信号（标准信号）不排队，多次到达可能丢失；实时信号（SIGRTMIN~）会排队、可带数据。
2. **SIGKILL 与 SIGSTOP**——内核保证它们总能终止/停止进程。
3. `signal` 行为跨平台不一致（语义历史包袱）；`sigaction` 可精确控制掩码、`SA_RESTART` 等，行为确定，推荐用它。
4. handler 可能在任意时刻打断主流程，若调用非可重入函数（malloc 持锁、printf 用缓冲）会死锁/破坏状态。只能调 async-signal-safe 函数（如 `write`）。
5. 把信号转成可在事件循环里处理的 fd 事件：self-pipe 在 handler 里写管道、主循环 epoll 该管道；`signalfd` 直接把信号变成可读 fd——避免在 handler 里干复杂事。
6. 子进程退出发 `SIGCHLD`，父进程在其 handler 里 `waitpid` 回收，避免僵尸（见 [L02](./L02-精通-进程与线程模型.md)）。
7. 捕获 SIGTERM，停止接收新请求、把存量请求处理完、释放资源后退出；容器/systemd 在超时后才 SIGKILL（见 [L16](./L16-精通-Namespace.md)/[L20](./L20-精通-systemd-与启动流程.md)）。
8. 数据直接映射到双方地址空间，**读写无需内核拷贝**（零拷贝）。但需自己用信号量/futex 做同步（见 [L15](./L15-精通-内核同步与futex.md)）。
9. 它们都是 fd，可加进同一个 epoll：`eventfd` 做线程/进程间唤醒、`timerfd` 把定时变成 fd 事件、`pidfd` 监听子进程退出——统一事件循环。
10. System V（`shmget`/`msgget`/`semget`）用 key/IPC id、全局命名、需手动清理；POSIX（`shm_open`/`mq_open`/`sem_open`）用路径名、接口更现代、可用 fd 语义。

</details>

---

## L15 · 精通内核同步与 futex

1. （⭐）自旋锁与互斥锁的本质区别？各用于什么上下文？
2. （⭐）什么是原子操作与 CAS？
3. （⭐⭐）futex 的「无竞争快路径」为什么不陷入内核？
4. （⭐⭐）RCU 适合什么场景？读端为何近乎零开销？
5. （⭐⭐⭐）内存屏障解决什么问题？
6. （⭐⭐⭐）seqlock（顺序锁）适合什么读写比例？
7. （⭐⭐⭐⭐）glibc 的 `pthread_mutex` 如何基于 futex 实现？
8. （⭐⭐⭐⭐）伪共享（false sharing）是什么？如何消除？
9. （⭐⭐⭐⭐⭐）PI futex（优先级继承）解决什么问题？
10. （⭐⭐⭐⭐⭐）定位锁争用的工具与方法？

<details>
<summary>📝 L15 答案与详解</summary>

1. 自旋锁忙等（不睡眠），用于不可睡眠的短临界区（中断上下文）；互斥锁会让线程睡眠，用于可能阻塞的长临界区。
2. 不可被打断的「读-改-写」；CAS（compare-and-swap）比较内存值与期望值，相等才写新值，是无锁编程基石。
3. 无竞争时只需一次用户态原子操作（CAS）改锁状态即可获锁，**只有发生竞争需要阻塞/唤醒时才 `futex` 系统调用陷入内核**。
4. 读多写极少（路由表、配置）。读端不加锁、不写共享状态，仅靠宽限期保证旧数据在所有读者退出后才释放，故读端近零开销。
5. 防止编译器/CPU 乱序导致的可见性问题，保证屏障两侧的内存操作顺序（acquire/release 语义）。
6. 读极多写极少且读可重试：读者不加锁、读前后比对序号，写时序号变化则重读。写不能太频繁。
7. 无竞争时用户态 CAS 改锁字；有竞争时调 `futex(FUTEX_WAIT/WAKE)` 让线程睡眠/唤醒——「快路径在用户态，慢路径才进内核」。
8. 不同 CPU 频繁写**同一缓存行**的不同变量，导致缓存行在核间反复失效弹跳。消除：按缓存行（通常 64B）对齐/填充，让热点变量独占缓存行。
9. 低优先级持锁、高优先级等锁、中优先级抢占低优先级 → 优先级反转。PI futex 临时提升持锁者优先级，避免高优先级被无限拖延。
10. `perf lock`、`perf record` 看锁相关栈、`offcputime`（bcc，线程为何离开 CPU）、火焰图找争用热点（见 [L18](./L18-精通-性能诊断方法论与工具.md)/[L19](./L19-精通-eBPF.md)）。

</details>

---

## L16 · 精通 Namespace

1. （⭐）列出 8 种 namespace。
2. （⭐）`clone`/`unshare`/`setns` 的区别？
3. （⭐⭐）PID namespace 的 init（PID 1）语义会带来哪些问题？
4. （⭐⭐）新建的 net namespace 只有 `lo`，容器如何获得对外网络？
5. （⭐⭐⭐）user namespace 如何实现「容器内 root、宿主无特权」？
6. （⭐⭐⭐）手搓容器时 `pivot_root` 前为什么要先 bind mount 新 rootfs、并把 `/` 设为 private？
7. （⭐⭐⭐⭐）容器网络不通，如何用 `nsenter` 进它的网络栈排障？
8. （⭐⭐⭐⭐）rootless 容器为什么需要 `newuidmap`/`newgidmap` 和 slirp4netns？
9. （⭐⭐⭐⭐⭐）cgroup namespace 为什么重要？
10. （⭐⭐⭐⭐⭐）为什么说「namespace 不是容器安全的全部」？

<details>
<summary>📝 L16 答案与详解</summary>

1. mnt、uts、ipc、pid、net、user、cgroup、time。
2. `clone` 建新进程同时进新 ns；`unshare` 让当前进程脱离进新 ns（不建进程）；`setns` 加入已有 ns（`nsenter` 即此）。
3. PID 1 默认忽略无 handler 的信号、需负责 reap 子进程；否则容器 `stop` 要等超时被 SIGKILL、僵尸堆积。用 tini/dumb-init（见 [L02](./L02-精通-进程与线程模型.md)）。
4. veth pair 一端进容器 ns、一端接宿主 bridge，配 IP/默认路由 + 宿主开 `ip_forward` + MASQUERADE NAT（见 [L13](./L13-精通-Socket-与连接管理.md)）。
5. uid_map 把容器内 uid 0 映射到宿主一个非特权 uid；容器内 root 只在本 ns 有 cap，碰宿主资源按真实非特权 uid 鉴权。
6. pivot_root 要求新根是挂载点，故先 bind mount 自身；设 private 防止容器内挂载事件传播污染宿主。
7. `nsenter -t <容器主进程pid> -n ss -tanp` / `tcpdump`——在宿主用宿主工具钻进容器网络栈，无需进容器装包。
8. 普通用户不能随意写 uid_map，靠 setuid 的 `newuidmap`/`newgidmap` 受控映射；无权建 veth/bridge，故用用户态网络栈 slirp4netns/pasta。
9. 没有它容器能看到宿主完整 cgroup 路径（信息泄漏）；有了它容器以为自己的 cgroup 就是根（见 [L17](./L17-精通-Cgroup-v2.md)）。
10. 容器与宿主共享内核，namespace 只隔离视图；安全还需 seccomp、capabilities drop、LSM（AppArmor/SELinux）等纵深防御；`--privileged` 几乎拆掉隔离。

</details>

---

## L17 · 精通 Cgroup v2

1. （⭐）cgroup 与 namespace 在容器里各负责什么？
2. （⭐）cgroup v1 与 v2 在「层级」上的根本区别？
3. （⭐⭐）`cpu.max = "50000 100000"` 等于多少个 CPU？
4. （⭐⭐）`memory.max` 与 `memory.high` 行为差异？
5. （⭐⭐⭐）「no internal process」规则是什么？为何这样设计？
6. （⭐⭐⭐）容器 CPU 没到 limit 却卡顿，如何排查与根治？
7. （⭐⭐⭐⭐）容器 OOMKilled 但宿主内存充足，如何确认是 cgroup 内 OOM？
8. （⭐⭐⭐⭐）v2 在脏页回写计费上相对 v1 的关键改进？为何让 `io.max` 对写负载有效？
9. （⭐⭐⭐⭐⭐）为什么不应直接 `echo` 改 systemd 管理的 cgroup 文件？
10. （⭐⭐⭐⭐⭐）`cgroup.freeze` 与 `cgroup.kill` 各自用途？

<details>
<summary>📝 L17 答案与详解</summary>

1. namespace 隔离视图，cgroup 限制资源（CPU/内存/IO/进程数）；二者合起来构成容器。
2. v1 每个 controller 一棵独立树（进程在各 controller 可不同位置）；v2 单一统一层级，所有 controller 共用一棵树。
3. quota=50000μs / period=100000μs = 每 100ms 用 50ms CPU = **0.5 核**。
4. max 是硬限，超且回收不动 → cgroup 内 OOM kill；high 是软限，超过只节流分配 + 积极回收（拖慢不杀）。
5. 启用 controller 的非根 cgroup，进程只能在叶子节点；避免资源在「中间节点进程」和「子组」之间分配产生歧义。
6. 看 `cpu.stat` 的 `nr_throttled`/`throttled_usec` 确认 throttling；根治：让运行时感知 limit（Go automaxprocs、JVM ActiveProcessorCount）或调大 limit（见 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md)）。
7. `memory.events` 的 `oom_kill` 增长、`memory.current` 逼近 `memory.max`，即 cgroup 内 OOM，与宿主总量无关（见 [L06](./L06-精通-OOM-与内存诊断.md)）。
8. v2 能把异步脏页回写 I/O 计入发起方 cgroup（v1 做不到）；故对「写完即返回、内核慢慢刷盘」的负载，`io.max`/`io.weight` 才真正生效（见 [L05](./L05-精通-物理内存管理与回收.md)/[L10](./L10-精通-块设备与IO调度.md)）。
9. systemd 是 cgroup 唯一写者，会在重载/重启时覆盖手改；应用 `systemctl set-property` 或 unit 指令（见 [L20](./L20-精通-systemd-与启动流程.md)）。
10. `cgroup.freeze` 冻结/恢复整组（docker pause）；`cgroup.kill`（5.14+）可靠杀光组内全部进程（含拒收信号者），比逐个 SIGKILL 更彻底。

</details>

---

## L18 · 精通性能诊断方法论与工具

1. （⭐）USE 方法的三个维度是什么？
2. （⭐）load average 包含哪些进程状态？为什么 load 高 CPU 却可能闲？
3. （⭐⭐）`perf stat` 能看什么？IPC 偏低通常说明什么？
4. （⭐⭐）火焰图怎么读？某个函数「宽」代表什么？
5. （⭐⭐⭐）`strace` 的开销来自哪里？什么时候不该用它？
6. （⭐⭐⭐）CPU / 内存 / I/O / 网络瓶颈分别优先用什么工具定位？
7. （⭐⭐⭐⭐）Brendan Gregg 的「60 秒检查清单」大致覆盖哪些命令？
8. （⭐⭐⭐⭐）一台机器 load 50 但 CPU idle 很高，怎么查根因？
9. （⭐⭐⭐⭐⭐）`perf record -g` 的采样原理是什么？
10. （⭐⭐⭐⭐⭐）如何区分「应用慢」是 CPU、锁、I/O 还是网络导致？

<details>
<summary>📝 L18 答案与详解</summary>

1. **Utilization（使用率）、Saturation（饱和度/排队）、Errors（错误）**——对每类资源逐一检查。
2. 含 R（运行/就绪）和 **D（不可中断睡眠，多为 I/O 等待）**。大量 D 会推高 load 而 CPU 空闲（见 [L02](./L02-精通-进程与线程模型.md)）。
3. PMU 硬件计数（指令数、cache-miss、分支预测、IPC）。IPC 低说明 CPU 大量等内存/cache miss 或流水线停顿，而非真正在算。
4. x 轴是栈样本聚合（宽=占用 CPU 时间多），y 轴是调用栈深度。找最宽的栈顶函数即热点。
5. 基于 ptrace，每次系统调用进出都停被跟踪进程，开销大；生产高频路径别用，改用 `perf trace` 或 eBPF（见 [L19](./L19-精通-eBPF.md)）。
6. CPU：`perf top`/火焰图；内存：`vmstat`/`free`/PSI；I/O：`iostat -x`/`biolatency`；网络：`ss`/`tcpretrans`/`nstat`。
7. 约：`uptime`、`dmesg`、`vmstat 1`、`mpstat`、`pidstat`、`iostat -x`、`free -m`、`sar`、`ss`、`top`——一分钟先建立全局印象。
8. 大量 D 状态进程：`vmstat` 看 `b` 列、`ps` 找 D 进程、`/proc/<pid>/stack` 看卡在哪个 I/O，再 `iostat`/`biolatency` 定位设备。
9. 周期性（按 PMU 事件或定时）中断 CPU、采当前调用栈，海量样本聚合成「谁占 CPU」的统计，`-g` 采带调用链。
10. CPU：on-CPU 火焰图占满；锁：`offcputime` 显示在锁上睡眠；I/O：D 状态 + iostat 高 await；网络：`ss -ti` 高 rtt/重传——按四象限逐一排除。

</details>

---

## L19 · 精通 eBPF

1. （⭐）cBPF 与 eBPF 的关系？eBPF 为何是「事件驱动」？
2. （⭐）map 在 eBPF 里的作用？列举三种类型。
3. （⭐⭐）verifier 的职责？列举三种会被拒的写法。
4. （⭐⭐）为什么「能用 tracepoint 就别用 kprobe」？
5. （⭐⭐⭐）CO-RE 靠哪三样东西实现「一次编译到处运行」？
6. （⭐⭐⭐）写一个 bpftrace 单行，按进程名统计 `execve` 调用。
7. （⭐⭐⭐⭐）在每个网络包上挂 eBPF 有什么风险？三种降开销手段？
8. （⭐⭐⭐⭐）tail call 与 BPF-to-BPF 调用各解决什么？
9. （⭐⭐⭐⭐⭐）libbpf + CO-RE 程序的「内核侧 + 用户侧」结构是怎样的？
10. （⭐⭐⭐⭐⭐）Cilium / Pixie / Parca 各用 eBPF 做什么？

<details>
<summary>📝 L19 答案与详解</summary>

1. cBPF 是 tcpdump 的包过滤虚拟机；eBPF 扩展为通用内核内虚拟机。事件驱动指程序不自运行，而在挂载点（syscall/函数/收包）事件发生时被内核调用。
2. map 是 eBPF 的存储，也是与用户态通信的唯一正道。类型如 HASH、PERCPU_ARRAY、RINGBUF。
3. 静态分析所有路径确保不崩内核/不死循环/不越界。被拒：无界循环、指针未判空就解引用、读未初始化栈。
4. tracepoint 是内核维护的稳定埋点，跨版本参数稳定；kprobe 挂具体函数，改名/内联即失效。
5. **BTF**（内核类型信息）、**libbpf**（加载时按目标 BTF 重定位字段偏移）、**vmlinux.h**（从 BTF 生成的类型头）。
6. `bpftrace -e 'tracepoint:syscalls:sys_enter_execve { @[comm] = count(); }'`。
7. 事件量巨大、累积开销高。手段：采样、用 raw_tracepoint、per-cpu map 聚合、缩小过滤范围。
8. 单程序受指令/栈限制；BPF-to-BPF 像函数调用拆分逻辑；tail call 跳到另一程序不返回（状态机式，如 XDP 多阶段）。
9. 内核侧 `.bpf.c`（`SEC()` 声明挂载点、编译成 BTF-enabled .o）+ 用户侧 loader（`bpftool gen skeleton` 生成骨架，open/load/attach/读 ringbuf）。
10. Cilium：K8s 网络/LB/策略（替代 kube-proxy）；Pixie：无侵入应用观测（自动解析协议）；Parca/Pyroscope：持续 profiling（全集群 CPU 采样）。

</details>

---

## L20 · 精通 systemd 与启动流程

1. （⭐）从上电到 login 的启动链路主要阶段？
2. （⭐）`multi-user.target` 对应传统哪个 runlevel？
3. （⭐⭐）`Wants=`/`Requires=`/`After=` 区别？为何 `After=network.target` 不保证网络可用？
4. （⭐⭐）`Type=simple`/`forking`/`notify`/`oneshot` 的就绪判定各是什么？
5. （⭐⭐⭐）initramfs 解决什么「鸡生蛋」问题？换内核后忘了重建会怎样？
6. （⭐⭐⭐）服务反复 Failed，用哪三个命令定位、各看什么？
7. （⭐⭐⭐⭐）`systemd-analyze blame` 与 `critical-chain` 如何配合？为何不能只看 blame？
8. （⭐⭐⭐⭐）socket activation 的工作流程与好处？
9. （⭐⭐⭐⭐⭐）把服务「沙箱化」可用哪些 systemd 加固指令？
10. （⭐⭐⭐⭐⭐）改坏 `/etc/fstab` 进不去系统，如何救援？

<details>
<summary>📝 L20 答案与详解</summary>

1. 上电 → UEFI/BIOS → bootloader（GRUB/systemd-boot）→ 内核+initramfs → 挂载真正 rootfs（switch_root）→ systemd（PID 1）→ default.target。
2. `multi-user.target` ≈ runlevel 3（文本多用户）；`graphical.target` ≈ runlevel 5。
3. Wants 弱依赖（失败不影响）；Requires 强依赖（依赖失败自己也失败）；After 仅排序、不产生依赖。`network.target` 只表示「网络栈已配置」，不代表网络可用，需 `network-online.target`。
4. simple：ExecStart 即主进程；forking：进程 fork 后台、父退出（需 PIDFile）；notify：进程调 `sd_notify` 才算就绪（最可靠）；oneshot：跑完即退（常配 RemainAfterExit）。
5. 根文件系统可能在需驱动才能访问的设备上（NVMe/LVM/LUKS），驱动却在根里——initramfs 先内存中提供驱动挂根。忘重建会因缺驱动导致启动找不到根/黑屏。
6. `systemctl status`（状态+退出码+尾日志）、`journalctl -u xxx -b`（完整日志）、`systemctl cat xxx`（最终生效配置含 drop-in）。
7. blame 列各服务耗时，但并行的慢服务未必拖慢总时间；critical-chain 显示串行依赖链上的真正瓶颈，二者结合才准。
8. systemd 先监听 socket，首个连接到来才拉起服务并传入已建立的 fd；好处：开机快、按需启动、平滑重启不丢连接（见 [L13](./L13-精通-Socket-与连接管理.md)）。
9. `ProtectSystem=strict`、`PrivateTmp=yes`、`NoNewPrivileges=yes`、`CapabilityBoundingSet=`、`SystemCallFilter=`（seccomp）——本质是 namespace/能力/seccomp（见 [L16](./L16-精通-Namespace.md)）。
10. GRUB 里加 `systemd.unit=emergency.target` 或 `init=/bin/bash` 进最小环境，`mount -o remount,rw /` 后修复 fstab；平时改完先 `mount -a` 验证。

</details>

---

## 🏆 综合实战题

> 贯穿多章的真实排障场景，考察把知识串起来的能力。建议读完全系列后挑战。

### 实战 1：容器周期性延迟毛刺（涉及 L03 / L17 / L05）

某 Go 服务部署在 K8s，`limits.cpu=2`，宿主 32 核。监控显示 P99 每隔约 100ms 出现毛刺，但 CPU 平均使用率只有 60%。请给出排查思路与根因。

<details>
<summary>📝 参考答案</summary>

- 看容器 `cpu.stat` 的 `nr_throttled`/`throttled_usec`——大概率在涨：**CPU throttling**。
- 根因：Go 默认 `GOMAXPROCS=32`（宿主核数），但 cgroup quota 只有 2 核；GC/并发在 100ms period 前段用多核瞬间烧光 200ms 配额，后段被冻结 → 周期性毛刺。
- 根治：`automaxprocs` 或显式 `GOMAXPROCS=2` 让运行时匹配 limit；必要时调大 limit 或用 `cpu.max.burst`（见 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md)/[L17](./L17-精通-Cgroup-v2.md)）。

</details>

### 实战 2：高并发服务连接数上不去（涉及 L08 / L12 / L13）

压测时 QPS 卡在某值，客户端大量 connection timeout，服务端 CPU 不高。请系统排查。

<details>
<summary>📝 参考答案</summary>

- `ss -lnt` 看 LISTEN 的 Recv-Q 是否逼近 Send-Q（全连接队列满）；`nstat | grep ListenOverflows` 确认溢出。
- 检查 `somaxconn` 与应用 `backlog`（取 min）、worker accept 速度、`nofile` 上限。
- 客户端侧排查端口耗尽（`Cannot assign requested address`）→ `tcp_tw_reuse` + 扩端口范围 + 连接池。
- 修复：调大 `somaxconn`+应用 backlog、增 worker、用 `SO_REUSEPORT` 多 listener（见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)/[L13](./L13-精通-Socket-与连接管理.md)）。

</details>

### 实战 3：节点偶发丢包，CPU/带宽都不高（涉及 L11 / L13）

K8s 节点偶发新连接超时与丢包，监控看 CPU、带宽都不满。请定位。

<details>
<summary>📝 参考答案</summary>

- `dmesg | grep conntrack` 看是否 `nf_conntrack: table full`；`cat /proc/sys/net/netfilter/nf_conntrack_count` 对比 `nf_conntrack_max`。
- 另查软中断单核打满：`mpstat -P ALL` 看某核 %soft 高、`/proc/net/softnet_stat` 丢包列。
- 修复：调大 `nf_conntrack_max`/`buckets`、缩短超时；开 RPS/RFS 分散软中断；长远用 Cilium/IPVS 减 conntrack（见 [L11](./L11-精通-Linux-网络协议栈.md)/[L13](./L13-精通-Socket-与连接管理.md)）。

</details>

### 实战 4：仅用 eBPF 定位磁盘抖动（涉及 L10 / L18 / L19）

应用偶发写延迟尖刺，不能改代码、不能重启。请用 eBPF/bcc 工具定位。

<details>
<summary>📝 参考答案</summary>

- `biolatency` 出块设备 I/O 延迟分布，确认是否设备侧偶发高延迟；
- `biosnoop` 看具体是哪个进程、哪个 I/O 慢、延迟多少；
- 结合 `iostat -x` 的 `await`/`aqu-sz` 与文件系统层（`fsync` 慢？见 [L07](./L07-精通-VFS-与文件系统.md)）；
- 若是 page cache 回写风暴，查 `dirty_ratio` 与 flusher（见 [L05](./L05-精通-物理内存管理与回收.md)/[L10](./L10-精通-块设备与IO调度.md)/[L19](./L19-精通-eBPF.md)）。

</details>

### 实战 5：还原一次容器 OOM（涉及 L04 / L05 / L06 / L17）

容器半夜被 OOMKilled，宿主内存监控显示充足。请完整还原与定位。

<details>
<summary>📝 参考答案</summary>

- `kubectl describe`/`dmesg` 确认 OOMKilled 且为 cgroup 内 OOM；
- 看容器 `memory.events`（oom_kill）、`memory.max`、`memory.current`、`memory.peak`；
- `memory.stat` 看内存去向：是应用堆（anon）、page cache（file）、还是 socket/slab（连接多？dentry 膨胀？）；
- 区分真泄漏（RSS 持续涨，用 pprof/valgrind）vs 限额过小（调 `memory.max` 或用 `memory.high` 软限）（见 [L06](./L06-精通-OOM-与内存诊断.md)/[L17](./L17-精通-Cgroup-v2.md)）。

</details>

---

> 🎓 全部做完并能独立讲清每道综合题，你已具备「懂内核原理、能现场排障、会系统调优」的 Linux 实战能力。配合 [INDEX.md](./INDEX.md) 的完读检查清单复盘薄弱项。
