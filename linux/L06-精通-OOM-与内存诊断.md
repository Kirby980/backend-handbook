# 精通 OOM 与内存诊断：OOM killer、oom_score、cgroup memory、内存泄漏排查、内存火焰图

> 课程编号：L06
> 路线图来源：Linux · 模块二 内存
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：cgroup v2 memory.max/high/low/min 与 memory.events 主流、systemd-oomd 基于 PSI 的用户态 OOM、MGLRU 回收、PSI 压力信号、smaps_rollup 标配）

---

## 引言：容器半夜被 Killed，谁干的？

凌晨 3 点，告警群炸了：某个 Go 服务的 Pod 反复重启，`kubectl describe pod` 里写着：

```
    State:          Running
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
```

`Exit Code 137` = `128 + 9`，意思是进程被信号 9（SIGKILL）杀掉了。但应用日志里干干净净——没有 panic、没有 fatal、没有任何"我要死了"的遗言。进程是被人从背后一枪打死的，连呻吟都没来得及发出。

凶手是谁？这里其实有**两个完全不同的凶手**，新手最容易搞混：

| 凶手 | 触发条件 | 杀谁 | 在哪看日志 |
|---|---|---|---|
| **全局 OOM killer** | 整机物理内存 + swap 耗尽 | 全机进程里 `oom_score` 最高的 | `dmesg` / `journalctl -k` |
| **cgroup OOM killer** | 某个 cgroup 用量触到 `memory.max` | 该 cgroup 内的进程 | `dmesg` + `memory.events` 的 `oom_kill` 计数 |
| **systemd-oomd** | PSI 内存/IO 压力超阈值（用户态） | 整个 cgroup（按 `ManagedOOMMemoryPressure`） | `journalctl -u systemd-oomd` |

容器里的 `OOMKilled` 99% 是第二种——**不是机器内存满了，而是你这个容器的 `memory.max` 满了**。机器可能还有 100 GB 空闲，你的容器照样被杀。这就是为什么"机器内存监控一片绿、容器却被 OOM"这种看似矛盾的现象天天发生。

更微妙的是：**进程不一定是凶手，也可能是受害者**。cgroup OOM 杀的是该 cgroup 内 `oom_score` 最高的进程，不一定是申请内存那一刻的进程。一个内存正常的 sidecar 完全可能替主容器"背锅"被杀。

本章把 OOM 从触发条件、选择算法，一路拆到 cgroup 内存控制、systemd-oomd、内存度量（VSZ/RSS/PSS/USS）、泄漏排查、eBPF 抓分配栈。读完你应该能：

- 解读一段 `dmesg` OOM 日志，指出谁触发、谁被杀、为什么是它
- 算出一个进程的 `oom_score`，并用 `oom_score_adj` 保护/牺牲特定进程
- 区分全局 OOM 与 cgroup OOM，从 `memory.events` 还原一次容器 OOM
- 说清 VSZ/RSS/PSS/USS 的差别，解释 `MemAvailable` 怎么算
- 用 `pmap`/`smaps_rollup`/heap profiler/Go pprof 定位内存泄漏
- 用 eBPF 抓出"谁在分配内存"的火焰图

---

## 第一章 OOM killer：内核的最后手段

### 1.1 OOM 何时触发

OOM（Out Of Memory）killer 是内核在**内存分配失败、且已无法通过回收挽救**时的最后手段。要理解它的触发，先要理解一次内存分配的瀑布流。

当内核要分配物理页（`__alloc_pages`）而空闲页低于水位线时，它会依次尝试（见 [L05 物理内存管理与回收](./L05-精通-物理内存管理与回收.md)）：

```
申请页 → 空闲 < low 水位 → 唤醒 kswapd 异步回收
       → 空闲 < min 水位 → 当前进程进入"直接回收"(direct reclaim)
            ├─ 回收 page cache（干净页直接丢、脏页回写）
            ├─ 换出匿名页到 swap（若有 swap）
            ├─ 收缩 slab（dentry/inode 缓存等）
            └─ 还是凑不够？ → 调用 out_of_memory() → OOM killer
```

关键点：**OOM 是"回收都救不了"之后才发生的**。所以 OOM 现场往往伴随：page cache 已被榨干、swap 已写满（如果有）、PSI 内存压力 100%、CPU 大量耗在 `kswapd`/`direct reclaim`。机器在 OOM 前通常先"假死"——疯狂回收、疯狂换页、什么都卡。

触发 OOM 的两条主线：

- **全局 OOM**：整机 `NodeFree` + 可回收都不够，分配走投无路。
- **cgroup OOM**：某 cgroup 的内存用量要超过 `memory.max`，且 cgroup 内回收（含 swap）也救不了。后者**只在该 cgroup 范围内选牺牲者**，与整机内存无关。

### 1.2 oom_score：怎么选牺牲者

OOM killer 的核心问题是"杀谁"。内核的哲学是：**杀掉能释放最多内存、且"价值最低"的进程，尽量一次解决问题**。

打分函数是 `oom_badness()`（位于 `mm/oom_kill.c`）。2026 年的现代内核里，打分基本就是一句话：

> **基础分 ≈ 进程占用的内存（RSS + swap + 页表），再叠加 `oom_score_adj` 的偏移。**

具体地，归一化到 0~1000 的范围：

```
points = process_pages              # RSS(匿名+文件+shmem) + swap + 页表页
points = points * 1000 / total_pages   # total = 全局总页 或 cgroup limit
points += oom_score_adj * total_pages / 1000   # adj 直接按比例加减
```

可以从 `/proc/[pid]/oom_score` 直接读出每个进程的当前得分（已是综合结果，0~1000）：

```bash
# 列出当前系统 oom_score 最高的 10 个进程（最可能被杀的）
$ for p in /proc/[0-9]*; do
    printf '%s %s\n' "$(cat $p/oom_score 2>/dev/null)" "$(cat $p/comm 2>/dev/null)"
  done | sort -rn | head
667 java
312 chrome
...
```

直觉结论：**谁吃内存多，谁先死**。这很合理——杀一个 8 GB 的进程比杀 50 个 100 MB 的进程更可能一次解决问题，也减少误伤。但这也意味着你那个"内存大户"主进程是天然靶子，哪怕它才是最重要的。

### 1.3 oom_score_adj：人为干预谁该死

`oom_score_adj` 是 `/proc/[pid]/oom_score_adj`，范围 `-1000 ~ +1000`，是给运维/编排系统的"调节旋钮"：

| 取值 | 效果 |
|---|---|
| `-1000` | 等价"永不被选"——分数被压到下界，OOM killer 绕过它 |
| `< 0` | 降低被杀概率（保护，如关键守护进程） |
| `0` | 默认，纯按内存占用打分 |
| `> 0` | 提高被杀概率（牺牲，如批处理/低优先级任务） |
| `+1000` | 最优先被杀 |

`adj` 是按 `total_pages` 比例叠加的：`adj=+1000` 相当于"假装你额外占了全部内存"，必死；`adj=-1000` 把分数压到 0 以下，免死。

```bash
# 保护一个关键进程（如本机监控 agent），让它几乎不会被 OOM 选中
$ echo -900 > /proc/$(pgrep -f node_exporter)/oom_score_adj

# 把一个批处理任务设为"优先牺牲"
$ echo 800 > /proc/$BATCH_PID/oom_score_adj
```

**生产中的真实用法**：

- **Kubernetes** 根据 QoS class 自动设置 `oom_score_adj`：`Guaranteed` Pod 接近 `-998`（最受保护），`Burstable` 按 request/limit 比例算一个中间值，`BestEffort` 是 `1000`（最先被杀）。所以资源不足时，BestEffort Pod 先死——这是设计。
- **systemd** 服务可在 unit 里写 `OOMScoreAdjust=-500` 保护关键服务。
- **数据库**：常把主进程 `adj` 调负，把可重建的子进程（如 PG 的并行 worker）调正。

注意：`adj` 会被子进程继承（fork 时复制），但有些 supervisor 会重置它。

### 1.4 panic_on_oom 与其他旋钮

并非所有系统都想"杀进程续命"。有些场景（高可靠集群、希望快速故障转移）宁愿整机 panic 重启，也不要一个被随机削弱的系统继续跑。

```bash
# /proc/sys/vm/ 下的 OOM 相关旋钮
$ sysctl vm.panic_on_oom        # 0=杀进程(默认) 1=panic(除非cpuset/mempolicy受限) 2=无条件panic
$ sysctl vm.oom_kill_allocating_task  # 1=直接杀"触发分配的那个进程"，不全局选(更快但更随机)
$ sysctl vm.oom_dump_tasks      # 1=OOM时dump所有进程的内存表(默认1，日志很长但有用)
$ sysctl vm.overcommit_memory   # 0=启发式(默认) 1=从不拒绝 2=按ratio严格限制
$ sysctl vm.overcommit_ratio    # overcommit_memory=2时，可分配 = swap + RAM*ratio%
```

`overcommit` 决定了 `malloc` 多大才会失败（见第四章 `CommitLimit`）。默认 `0`（启发式）下，单次"明显荒唐"的大申请会被拒（malloc 返回 NULL），但日常的小额超额分配都放行——这正是 `malloc 100GB` 能成功、写入时才崩的根源。

`panic_on_oom=1` 在内核高可用集群里常见：配合 `kernel.panic=10`（panic 后 10 秒重启），让节点干脆利落地故障转移，而不是变成一台"被 OOM 啃过、状态诡异"的僵尸机。

### 1.5 解读 dmesg OOM 日志

这是排障的核心技能。一段典型的全局 OOM 日志（`dmesg -T` 或 `journalctl -k`）长这样，逐段拆解：

```
[Mon Jun  1 03:14:07 2026] node invoked oom-killer: gfp_mask=0x...(GFP_KERNEL), order=0, oom_score_adj=0
                            ^^^^ 是"node"这个进程在申请内存时触发了 OOM（它是触发者，未必是被杀者）
[...] CPU: 3 PID: 8821 Comm: node ...
[...] Mem-Info:
[...] active_anon:1932841 inactive_anon:5023 ... free:21884 ...   ← 各类页统计（单位:页, ×4KB）
[...] Node 0 Normal free:87536kB min:90112kB low:112640kB ...     ← 关键! free < min，水位击穿
[...] Free swap  = 0kB    Total swap = 0kB                         ← 无 swap 或已耗尽
[...] Tasks state (memory values in pages):
[...] [  pid  ]   uid  tgid total_vm      rss ... oom_score_adj name
[...] [   8821]  1000  8821  2451233  1893122 ...        0       node    ← rss 最高
[...] [   1234]     0  1234    52341     1203 ...     -900       node_exporter
[...] oom-killer: ... 
[...] Out of memory: Killed process 8821 (node) total-vm:9804932kB, anon-rss:7572488kB, file-rss:..., shmem-rss:...
       ^^^^ 真正被杀的是 8821，释放了约 7.5 GB 匿名内存
[...] oom_reaper: reaped process 8821 (node), now anon-rss:0kB, file-rss:0kB, shmem-rss:0kB
```

读这段日志的"四步法"：

1. **谁触发**：看第一行 `invoked oom-killer` 前的进程名/PID——它申请内存时失败。
2. **为什么救不了**：看 `free < min`（水位击穿）、`Free swap = 0`（swap 耗尽）、`active_anon` 占绝对多数（说明是匿名内存撑爆，page cache 已被回收，不是文件缓存问题）。
3. **谁被杀**：看 `Out of memory: Killed process ...`，以及 `anon-rss` 看它释放了多少。
4. **为什么是它**：对照 `Tasks state` 表里各进程的 `rss` 和 `oom_score_adj`——通常是 rss 最大且 adj 不为负的那个。

如果是 **cgroup OOM**，日志里会多出一行标明范围：

```
[...] memory: usage 524288kB, limit 524288kB, failcnt 27
[...] Memory cgroup out of memory: Killed process 9931 (python) ...
[...] oom-kill:constraint=CONSTRAINT_MEMCG,nodemask=...,cpuset=...,mems_allowed=...,oom_memcg=/kubepods.slice/.../pod...,task_memcg=...
                              ^^^^^^^^^^^^^^^^^ MEMCG 约束 = 是 cgroup 触顶，不是整机
```

看到 `CONSTRAINT_MEMCG` 就基本可以断定：**机器内存没满，是这个容器自己的 limit 满了**。`oom_memcg` 字段直接告诉你是哪个 cgroup。

---

## 第二章 cgroup 内存控制：容器 OOM 的真相

### 2.1 cgroup v2 的四道内存闸门

2026 年绝大多数发行版默认 cgroup v2（统一层级，systemd 默认 unified，详见 [L17 Cgroup v2 与资源控制](./L17-精通-Cgroup-v2.md)）。内存控制器在 `/sys/fs/cgroup/<path>/` 下提供四个核心限额文件，理解它们的层次关系是排障的关键：

| 文件 | 语义 | 触顶后果 | 类比 |
|---|---|---|---|
| `memory.min` | **硬保护**：低于此值的内存永不被回收 | 全局回收也绕开这部分 | 不可侵犯的底线 |
| `memory.low` | **软保护**：尽量不回收，除非其他都回收光了 | 优先回收别人 | 优先级 |
| `memory.high` | **软限流**：超过则强烈回收 + **节流**（throttle）申请进程 | 进程变慢，但不被杀 | 限速带 |
| `memory.max` | **硬上限**：超过且回收无效 → **cgroup OOM kill** | 进程被 SIGKILL | 一堵墙 |

层次关系：`min ≤ low ≤ ...实际用量... ≤ high ≤ max`。

```
0 ──min──low──────[ 实际用量 ]──────high──────max
   │     │                          │          │
   保护   软保护                     软限流      硬墙(OOM)
```

**`memory.high` 是被低估的好东西**。它在触墙前先"踩刹车"：进程申请内存时被人为延迟（节流），同时触发激进回收，给系统留出喘息和扩容窗口，而不是直接杀。生产上建议 `high` 设在 `max` 下方一点（如 `max=512M, high=460M`），把硬杀变成软降速。

```bash
# 查看一个容器 cgroup 的内存闸门
$ cd /sys/fs/cgroup/kubepods.slice/.../cri-containerd-<id>.scope
$ cat memory.max memory.high memory.min memory.low
536870912        # max = 512 MiB（K8s limits.memory: 512Mi）
max              # high 未设（"max"表示无限）
0
0
$ cat memory.current        # 当前实际用量（含 page cache）
489234432       # ≈ 466 MiB，逼近上限了
```

### 2.2 memory.events：OOM 的"案发记录"

`memory.events` 是排查容器 OOM 的**第一现场**，它记录了这个 cgroup 触各道闸门的累计次数：

```bash
$ cat /sys/fs/cgroup/.../memory.events
low 0
high 142          # 触发 high 软限流/激进回收 142 次 ← 内存吃紧的早期信号!
max 27            # 用量撞到 max（被拦+回收）27 次
oom 3             # 回收也救不了、进入 OOM 流程 3 次
oom_kill 3        # 实际杀掉进程 3 次 ← 这就是 OOMKilled 的铁证
oom_group_kill 0  # 整组一起杀的次数（memory.oom.group=1 时）
```

排障逻辑：

- `oom_kill > 0` → **确定**这个容器发生过 cgroup OOM，且是被自己的 limit 杀的。
- `high` 计数持续增长但 `oom_kill=0` → 内存吃紧、在软限流挣扎，性能下降但还没死，是**扩容的提前预警**。
- `max` 增长而 `oom_kill` 不增 → 撞墙后靠回收续命，危险但尚可。

K8s 里 `kubectl describe pod` 显示的 `OOMKilled` 与 `Exit Code 137`，其内核侧的真相就在这个文件的 `oom_kill` 计数里。把它纳入监控（Prometheus 的 `container_oom_events_total` 之类），比事后翻 dmesg 强得多。

还有一个细节：`memory.oom.group`。设为 `1` 时，cgroup OOM 会**杀掉整个 cgroup 的所有进程**而非单个，避免"杀了 worker、留下半死的主进程"的残局。容器运行时常对 init 进程所在 cgroup 启用它。

### 2.3 memory.stat：钱花哪了

`memory.current` 只告诉你"用了多少"，`memory.stat` 告诉你"用在哪"——这是判断"是真泄漏还是 page cache"的关键：

```bash
$ cat /sys/fs/cgroup/.../memory.stat
anon          312428032     # 匿名内存（堆/栈）—— 真正"用掉"的，回收要靠 swap
file          178257920     # 文件页缓存(page cache) —— 内存紧张时可直接回收!
kernel_stack    2097152
slab           41943040     # 内核 slab（dentry/inode 等）
sock            1048576
shmem          16777216     # 共享内存/tmpfs
file_mapped    33554432
anon_thp              0     # 透明大页里的匿名
inactive_anon 298745856
active_anon    13682176
inactive_file  98304000
active_file    79953920
...
pgfault       1283746       # 缺页次数
pgmajfault       3021       # major fault（需从磁盘读）
workingset_refault_anon ...
```

**关键洞察**：`memory.current = anon + file + slab + sock + ... `。其中 `file`（page cache）是"软"的——内存紧张时内核会先回收它。所以：

- 一个容器 `memory.current` 逼近 `max`，但 `memory.stat` 里 **大头是 `file`** → 多半不是泄漏，是它读了大文件填满了 page cache，OOM 风险低（会被回收）。
- 大头是 **`anon` 且持续增长** → 真·内存增长（堆），可能是泄漏，回收只能靠 swap（容器常无 swap）→ OOM 风险高。

这解释了一个经典困惑：容器内 `free` 看着没多少，`memory.current` 却快满了——因为 page cache 也算进 cgroup 用量，而 `free` 把它列在 buff/cache。**cgroup 的内存账本包含 page cache**，这点必须牢记。

### 2.4 容器内 OOM vs 全局 OOM 对照

| 维度 | 全局 OOM | cgroup OOM |
|---|---|---|
| 触发 | 整机 free + 可回收 < min 水位 | cgroup 用量 ≥ `memory.max` 且回收无效 |
| 范围 | 全机所有进程参与打分 | 仅该 cgroup 内进程打分 |
| 日志标志 | `constraint=CONSTRAINT_NONE` | `constraint=CONSTRAINT_MEMCG` |
| 看哪 | `dmesg` 的 `Mem-Info`/`Node ... free` | `memory.events` 的 `oom_kill` |
| 典型场景 | 物理机内存超卖、无 limit 的进程暴涨 | 容器 `limits.memory` 设小了/有泄漏 |
| 机器整体 | 真的没内存了，全机受影响 | 机器可能很空闲，只这个容器受影响 |

排障第一刀永远是分清这两者：**`dmesg | grep -i 'killed process'` 看有没有 `CONSTRAINT_MEMCG`**。是 MEMCG 就去查那个容器的 limit 和 `memory.stat`；是 NONE 就查整机谁在吃内存、是否超卖。

---

## 第三章 systemd-oomd：基于 PSI 的用户态 OOM

### 3.1 为什么需要用户态 OOM

内核 OOM killer 有个根本缺陷：**它出手太晚**。等到分配真的失败、水位击穿，机器往往已经在"颠簸地狱"里卡了几十秒甚至几分钟——疯狂换页、page cache 反复抖动、所有进程都慢如蜗牛。用户体验上，机器"早就死了"，内核才慢悠悠地决定杀谁。

`systemd-oomd`（systemd 247+，2026 年主流发行版默认启用）解决的就是"**在内核 OOM 之前，根据压力提前干预**"。它的依据是 **PSI（Pressure Stall Information）**——内核 4.20 引入、现已成压力信号事实标准（见 [L05](./L05-精通-物理内存管理与回收.md)）。

PSI 度量的是"**因为等资源而停顿的时间占比**"，而非"用了多少内存"。这是质的区别：用量满了不一定卡（page cache 满是好事），但只要进程频繁因等内存而停顿，PSI 就升高。

```bash
$ cat /proc/pressure/memory
some avg10=42.31 avg60=18.22 avg300=5.10 total=...
full avg10=23.05 avg60=9.11  avg300=2.30 total=...
#     ^some^ = 至少一个任务因内存停顿的时间占比
#     ^full^ = 所有任务都因内存停顿（系统性卡顿）的时间占比
```

`some avg10=42` 意味着过去 10 秒里有 42% 的时间至少有一个任务卡在等内存上——机器在受苦，但内核 OOM 可能还没触发。

### 3.2 systemd-oomd 怎么工作

`systemd-oomd` 是个用户态守护进程，周期性读取各 cgroup 的 PSI 和内存用量，当超过配置阈值时，**主动杀掉压力最大的那个 cgroup**（整组），把内存还给系统：

```bash
# 全局策略（/etc/systemd/oomd.conf）
$ cat /etc/systemd/oomd.conf
[OOM]
SwapUsedLimit=90%                       # swap 用超 90% → 触发
DefaultMemoryPressureLimit=60%          # 内存 PSI 超 60% → 候选
DefaultMemoryPressureDurationSec=20s    # 且持续 20 秒才动手（避免抖动误杀）
```

要对某个 slice 启用，需在其 unit 里声明"愿意被 oomd 管理"：

```ini
# 例：/etc/systemd/system/user.slice.d/oomd.conf
[Slice]
ManagedOOMMemoryPressure=kill           # 按内存 PSI 杀
ManagedOOMMemoryPressureLimit=50%       # 该 slice 自己的阈值
ManagedOOMSwapUsedLimit=90%             # 按 swap 用量杀
```

查看 oomd 的动作记录：

```bash
$ journalctl -u systemd-oomd
... systemd-oomd[612]: Killed /user.slice/user-1000.slice/.../app.scope due to memory pressure
    for /user.slice being 51.23% > 50.00% for > 20s with reclaim activity
$ oomctl                # 查看 oomd 当前监控的 cgroup 与压力状态
```

### 3.3 三种 OOM 机制的分工

到这里出现了三套"杀进程续命"的机制，它们在不同层次、不同时机各司其职：

```
压力↑                                                          时间线
 │
 │  systemd-oomd            内核 cgroup OOM         内核全局 OOM
 │  (用户态/PSI驱动)         (memory.max触顶)        (整机水位击穿)
 │  最早出手                 容器超限时              最后兜底
 │  杀"整个cgroup"           杀cgroup内单进程        杀全机最高分进程
 └──────────────────────────────────────────────────────────────→
    "还没真满，但很卡了"     "这个容器满了"          "整机彻底没了"
```

| 机制 | 层 | 依据 | 粒度 | 时机 | 2026 状态 |
|---|---|---|---|---|---|
| systemd-oomd | 用户态 | PSI + swap% | 整个 cgroup | 最早 | 桌面/部分服务器默认启用 |
| cgroup OOM | 内核 | memory.max | cgroup 内进程 | 容器超限 | 容器场景主力 |
| 全局 OOM | 内核 | 整机水位 | 全机进程 | 最后兜底 | 永远存在的安全网 |

实践建议：服务器上 systemd-oomd 多用于保护交互/系统会话（避免一个失控进程拖垮整机 ssh 都进不去）；容器资源边界仍靠 cgroup `memory.max`；全局 OOM 是永远的最后防线，不该被指望，但必须留着。

---

## 第四章 内存度量：VSZ/RSS/PSS/USS 与 /proc/meminfo

### 4.1 四个内存指标，别再混淆

监控内存时，"这进程用了多少内存"这个问题没有唯一答案——取决于你问的是哪个指标。共享内存（动态库、共享映射、fork 后的 COW 页）让这件事变得微妙：

| 指标 | 全称 | 含义 | 共享库怎么算 | 用途 |
|---|---|---|---|---|
| **VSZ** | Virtual Size | 虚拟地址空间总大小 | 全算 | 几乎没用（含未兑现的映射） |
| **RSS** | Resident Set Size | 驻留物理内存 | **全算**（多进程重复计） | 粗略，但会高估总和 |
| **PSS** | Proportional Set Size | 比例分摊后的驻留 | 按共享进程数**均摊** | 评估"真实占用"的最佳单值 |
| **USS** | Unique Set Size | 进程独占的内存 | **不算** | "杀了它能立刻回收多少" |

举例：libc 占 2 MB，被 100 个进程共享。

- 每个进程 **RSS** 都把这 2 MB 算上 → 100 个进程 RSS 相加 = 200 MB（实际物理只占 2 MB，**严重高估**）。
- 每个进程 **PSS** 只算 `2MB / 100 = 20 KB` → 100 个进程 PSS 相加 = 2 MB（**正好等于真实物理占用**）。这就是 PSS 牛在哪。
- **USS** 完全不含 libc，只算该进程独享的堆/栈/匿名页 → 反映"干掉它能直接拿回多少"。

所以监控容器/进程内存占用，**PSS 是最诚实的单值**；判断"杀谁释放最多"看 USS；RSS 用于粗看单进程，但**绝不能简单相加**（会把共享内存重复计算成 N 倍）。

```bash
# RSS：快速但粗
$ grep VmRSS /proc/<pid>/status
# PSS + USS：精确，从 smaps_rollup 一次读出（比逐行 smaps 快得多）
$ cat /proc/<pid>/smaps_rollup
Rss:            1893122 kB
Pss:            1871044 kB        # 比例分摊后
Private_Clean:        0 kB
Private_Dirty:  1837120 kB        # Private_Clean + Private_Dirty ≈ USS
Shared_Clean:     45056 kB
Shared_Dirty:     11008 kB
Swap:                 0 kB
SwapPss:              0 kB        # swap 中的部分也按比例算
...
# 用 smem 工具直接看 PSS/USS 排名（最常用）
$ smem -rk -c "pid name uss pss rss" | head
```

`smaps_rollup`（4.14+ 标配）是把 `/proc/<pid>/smaps` 几百个 VMA 段的统计预先汇总好的单文件，读一次就有全进程的 PSS/USS，避免了遍历巨大的 smaps（对大堆进程，smaps 可能几 MB，读取本身就慢）。

### 4.2 /proc/meminfo 逐行解读

`/proc/meminfo` 是整机内存的"资产负债表"。挑后端/SRE 最该认识的几行：

```bash
$ cat /proc/meminfo
MemTotal:       32825384 kB    # 物理内存总量
MemFree:         1284736 kB    # 完全空闲（注意：低不代表危险!）
MemAvailable:   18472104 kB    # 【最重要】预估可用内存（见 4.3）
Buffers:          204800 kB    # 块设备元数据缓存
Cached:         16384000 kB    # page cache（文件缓存）—— 可回收，不是"被吃了"
SwapCached:            0 kB
Active:         12345600 kB
Inactive:        8765400 kB    # LRU 链表（MGLRU 下内部是多代，见 L05）
Active(anon):   10240000 kB
Inactive(anon):   512000 kB
Active(file):    2105600 kB
Inactive(file):  8253400 kB
Dirty:             40960 kB    # 待回写的脏页（过高=回写跟不上）
Writeback:             0 kB    # 正在回写
AnonPages:      10752000 kB    # 匿名页总量（堆/栈，回收靠 swap）
Mapped:          1024000 kB    # mmap 映射的文件页
Shmem:            262144 kB    # tmpfs/共享内存
Slab:            1048576 kB    # 内核 slab
  SReclaimable:   786432 kB    #   可回收 slab（dentry/inode 缓存）—— 紧张时能回收
  SUnreclaim:     262144 kB    #   不可回收 slab
SwapTotal:       8388604 kB
SwapFree:        8388604 kB    # swap 还没用 → SwapFree==SwapTotal
CommitLimit:    24801292 kB    # overcommit=2 时的可分配上限
Committed_AS:   14500000 kB    # 已"承诺"出去的虚拟内存总量
HugePages_Total:       0
```

**最常见的误读**：看到 `MemFree` 只剩 1 GB 就慌着扩容。错！Linux 会把空闲内存几乎全拿去做 page cache（`Cached`），这是**好事**——内存闲着才是浪费。该看的是 `MemAvailable`：它预估了"在不严重影响性能的前提下，还能给新申请挤出多少内存"（含可回收的 page cache 和可回收 slab）。

### 4.3 MemAvailable 怎么算

`MemAvailable`（3.14+ 引入）是内核替你算好的"真实可用量"，省得你瞎估。它的算法（`mm/page_alloc.c` 的 `si_mem_available()`）量级上是：

```
MemAvailable ≈ MemFree
             - (低水位线 low watermark 保留)        # 留给内核应急，不能全花
             + 可回收的 page cache 的大部分          # 活跃文件页留一部分，其余算可用
             + 可回收的 slab (SReclaimable) 的大部分  # 同样扣掉一个水位保留
```

直觉：`MemFree` 太悲观（没算可回收的缓存），`MemFree + Cached` 又太乐观（page cache 不能全回收、回收有代价、还得给内核留水位）。`MemAvailable` 取中间一个保守可信的估值。**监控告警应基于 `MemAvailable`，不是 `MemFree`。** 量级上，一台 32 GB、跑满 page cache 但应用只占 12 GB 的机器，`MemFree` 可能只有 1~2 GB，而 `MemAvailable` 会显示 18 GB 左右——后者才是真相。

### 4.4 共享内存的计量陷阱

共享内存（tmpfs、`shmget`、`MAP_SHARED`）是计量的重灾区，几个典型坑：

1. **`/dev/shm` 和 tmpfs 算匿名还是文件？** tmpfs 的页计入 `Shmem`，且**计入使用它的 cgroup 的 `memory.stat` 的 `shmem`**。一个进程往 `/dev/shm` 写 2 GB，它的 RSS 可能没涨多少，但 cgroup `memory.current` 涨了 2 GB——因为 tmpfs 占用算在创建/写入它的 cgroup 头上。容器里 `/dev/shm` 默认 64 MB 写满会报 "No space left"，但调大后写满则可能直接把容器顶到 `memory.max` 触发 OOM。

2. **多进程共享 segment 的 RSS 重复计**：N 个进程 attach 同一个 1 GB 共享内存，每个 RSS 都 +1 GB，相加 N GB，实际只占 1 GB。**只有 PSS 能正确均摊。**

3. **谁释放谁负责**：共享内存不随单个进程退出而释放（System V shm 尤其），必须显式 `shmctl(IPC_RMID)` 或所有 attach 者退出。泄漏的共享内存在 `ipcs -m` 里看 attach 计数为 0 却仍占空间。

```bash
$ ipcs -m              # System V 共享内存段
$ df -h /dev/shm       # tmpfs 用量
$ cat /sys/fs/cgroup/.../memory.stat | grep shmem
```

排查"RSS 不高但 cgroup 内存满"时，第一个怀疑对象就是 tmpfs/共享内存。

---

## 第五章 内存泄漏排查：从粗到细

### 5.1 第一步：确认是不是泄漏，泄漏在哪一层

"内存涨"不等于"泄漏"。先分类（用第二章的 `memory.stat` + 第四章的指标）：

| 现象 | 大概率原因 | 排查方向 |
|---|---|---|
| `file`(page cache) 涨，`anon` 平 | 读大文件填缓存 | 正常，可回收，不是泄漏 |
| `anon` 持续单调涨，不回落 | 堆泄漏 / 缓存无上限 | heap profiler |
| `slab`/`SUnreclaim` 涨 | 内核对象泄漏（fd、dentry、conntrack） | `slabtop` / `/proc/slabinfo` |
| `shmem` 涨 | tmpfs / 共享内存泄漏 | `ipcs` / `df /dev/shm` |
| RSS 平、VSZ 涨 | 只是地址空间预留 | 不是泄漏 |

判断"单调增长"要看趋势：泄漏的特征是**长期单调上升、GC/重启才回落**；正常缓存是**涨到一个平台就稳住**。

### 5.2 pmap / smaps：进程级别的内存分布

`pmap` 看进程的内存段构成，快速定位"是哪块在涨"：

```bash
$ pmap -x <pid> | sort -rnk3 | head        # 按 RSS 排序，看最大的几段
Address           Kbytes     RSS   Dirty Mode  Mapping
00007f...        2097152 2097152 2097152 rw---   [ anon ]   ← 2GB 匿名段，可疑
000056...         524288  520000  520000 rw---   [ heap ]   ← 堆
00007f...          45056   12000       0 r-x--   libc.so.6
...
```

匿名大段（`[ anon ]`、`[ heap ]`）持续增长 = 堆相关泄漏的信号。对照 `/proc/<pid>/smaps` 能看到每段的 `Pss`/`Private_Dirty`，定位哪段是私有脏页（最可能是泄漏的实在内存）。

### 5.3 C/C++：valgrind / massif / ASAN

**valgrind memcheck**——精确但慢（10~50 倍），适合测试环境/复现场景：

```bash
$ valgrind --leak-check=full --show-leak-kinds=all ./myapp
==1234== LEAK SUMMARY:
==1234==    definitely lost: 8,192 bytes in 4 blocks      # 确定泄漏（指针已丢）
==1234==    indirectly lost: 0 bytes in 0 blocks
==1234==    possibly lost: 1,024 bytes in 2 blocks
==1234==    ... by 0x... : leaky_func (myapp.c:42)         # 直接给出泄漏点调用栈
```

**massif**——堆用量随时间的剖析（"内存涨在哪个函数"，不只是泄漏）：

```bash
$ valgrind --tool=massif ./myapp
$ ms_print massif.out.<pid>     # 文本火焰图：每个时间点的堆构成
```

**AddressSanitizer (ASAN)**——编译期插桩，开销低（约 2 倍），适合 CI 和压测，还能抓越界/use-after-free：

```bash
$ gcc -fsanitize=address -g -O1 myapp.c -o myapp
$ ASAN_OPTIONS=detect_leaks=1 ./myapp     # 退出时报告泄漏栈
```

### 5.4 jemalloc / tcmalloc 的 heap profiling

生产 C/C++ 服务（以及很多用 jemalloc 的 Rust/数据库）可以靠分配器自带的采样剖析，几乎零侵入、可在线开关：

```bash
# jemalloc：环境变量开启采样，定期 dump
$ MALLOC_CONF="prof:true,prof_active:true,lg_prof_sample:19,prof_prefix:/tmp/jeprof" ./myapp
# 运行中按需 dump，或周期 dump，然后:
$ jeprof --show_bytes --pdf ./myapp /tmp/jeprof.*.heap > heap.pdf   # 生成调用图

# tcmalloc（gperftools）：
$ HEAPPROFILE=/tmp/myapp.hprof ./myapp
$ pprof --pdf ./myapp /tmp/myapp.hprof.0001.heap > heap.pdf
```

`lg_prof_sample:19` 表示约每 512 KB（2^19）采样一次分配，开销很低却足以定位大头。这是生产环境抓 C/C++ 泄漏的首选——比 valgrind 实用得多。

### 5.5 Go：pprof heap

Go 服务内置 pprof，是定位 Go 内存增长的标准武器（详见 [golang G22 pprof](../golang/G22-精通-Go-pprof-性能剖析.md)）：

```go
import (
    "net/http"
    _ "net/http/pprof"     // 注册 /debug/pprof/ 路由
)
func main() {
    go func() { http.ListenAndServe("localhost:6060", nil) }()
    // ... 业务
}
```

```bash
# 抓当前堆，看"还活着"的对象分配在哪（inuse_space = 当前占用）
$ go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) top
flat  flat%   sum%        cum   cum%
512MB 64.0%  64.0%      512MB  64.0%  myapp/cache.(*Store).Put   ← 元凶
...
(pprof) list cache.Put          # 看具体哪行
(pprof) web                     # 生成调用图

# 对比两个时间点的 heap，差值直接暴露泄漏增量（最有效的手法）
$ go tool pprof -base heap_t1.pb.gz heap_t2.pb.gz
```

Go 的几个常见"假泄漏 / 真泄漏"：

- **goroutine 泄漏**（最常见）：goroutine 阻塞在 channel/锁上永不退出，每个占栈 + 引用的对象不释放。看 `/debug/pprof/goroutine?debug=1` 的数量是否单调涨。
- **`inuse_space` vs `alloc_space`**：查泄漏看 `inuse`（当前还占着的），查"分配压力大"看 `alloc`（累计分配，反映 GC 压力）。
- **RSS 不降的"假泄漏"**：Go runtime 把内存还给 OS 是惰性的（`MADV_FREE`/`MADV_DONTNEED`），堆已 free 但 RSS 还挂着，这不是泄漏。看 `runtime.MemStats` 的 `HeapInuse` 是否稳定。

### 5.6 内核侧泄漏：slab 与 fd

应用内存正常、整机内存却涨——怀疑内核对象泄漏：

```bash
$ slabtop -o -s c | head        # 按 cache 大小排，看哪类内核对象暴涨
 OBJS  ACTIVE  USE OBJ SIZE  SLABS ... NAME
8.2M    8.2M  99%   0.19K  ...        dentry          ← dentry 缓存巨大（疯狂 stat/open?）
2.1M    2.0M  95%   1.00K  ...        ext4_inode_cache
...
$ cat /proc/slabinfo            # 原始数据
$ ls /proc/<pid>/fd | wc -l     # fd 泄漏（每个 fd 也耗内核内存 + 触 EMFILE）
$ cat /proc/sys/fs/file-nr      # 全局已分配 fd 数
```

`dentry`/`inode` cache 是可回收 slab（`SReclaimable`），内存紧张时会回收，不算硬泄漏；但 conntrack 表满、未关闭的 socket、fd 泄漏是真问题。

---

## 第六章 内存火焰图与 eBPF 记录分配栈

### 6.1 为什么要"内存火焰图"

前面的 heap profiler 各自绑定语言/分配器。如果你想要一个**跨语言、低开销、生产可在线开**的"谁在分配/谁在缺页"的全景图，eBPF 是 2026 年的答案（见 [L19 eBPF 深度实战](./L19-精通-eBPF.md)）。

两类内存火焰图：

- **分配火焰图**：在 `malloc`/`mmap`/`brk` 或 `kmem` tracepoint 上挂 uprobe/kprobe，记录调用栈和分配字节，聚合成火焰图——回答"谁在分配内存"。
- **缺页火焰图**：在缺页路径（`page_fault`/`exceptions:page_fault_user`）上采栈——回答"谁在触发物理内存兑现"，常用于定位 RSS 增长来源。

### 6.2 bpftrace 一行抓分配栈

`bpftrace` 是最快的上手方式。统计用户态 `malloc` 的调用栈与字节数（按栈聚合）：

```bash
# 统计某进程 malloc 的累计字节，按用户态调用栈聚合
$ bpftrace -e '
  uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc {
      @bytes[ustack] = sum(arg0);
  }
  interval:s:10 { print(@bytes); clear(@bytes); exit(); }' -p <pid>

# 内核侧：统计触发缺页最多的进程（哪些进程在"兑现"物理页）
$ bpftrace -e '
  software:major-faults:1 { @major[comm] = count(); }
  software:minor-faults:1 { @minor[comm] = count(); }
  interval:s:5 { print(@major); print(@minor); clear(@major); clear(@minor); }'

# 内核 slab/页分配 tracepoint：谁在向 buddy 要页
$ bpftrace -e 'tracepoint:kmem:mm_page_alloc { @[comm, kstack] = count(); }'
```

`malloc` uprobe 会 hook 进程**每一次** malloc，高频路径下开销不可忽视——生产上更推荐采样式工具或对低频大块分配下手。

### 6.3 bcc 工具与火焰图

bcc 提供现成工具：

```bash
# memleak：周期性报告"已分配但未释放"的栈（带分配字节）——eBPF 版泄漏检测
$ /usr/share/bcc/tools/memleak -p <pid> 5     # 每 5 秒报告一次未释放分配的 top 栈
[03:14:07] Top 10 stacks with outstanding allocations:
    536870912 bytes in 512 allocations from stack
        malloc+0x...
        cache_put+0x...           ← 泄漏点
        handle_request+0x...

# 抓栈生成火焰图（配合 brendangregg/FlameGraph）
$ /usr/share/bcc/tools/stackcount -p <pid> -U "c:malloc" > out.stacks
$ ./flamegraph.pl out.stacks > malloc_flame.svg
```

`memleak` 的原理：hook `malloc`/`free`，记录每次分配的栈和地址，free 时移除；周期性 dump 仍"挂账"的分配——这正是泄漏。它比 valgrind 轻得多，且能 attach 到**正在生产运行**的进程，是 2026 年线上抓 native 泄漏的利器。

连续型采集可对接 **Parca / Pyroscope** 这类持续剖析（continuous profiling）平台，把内存火焰图做成时间序列，回看"泄漏从哪个版本/哪个时刻开始"。

---

## 生产实践

**还原一次 OOM：dmesg → cgroup memory.events 的标准流程**

线上收到 `OOMKilled` 告警，按这个剧本走，5 分钟定位：

```bash
# ① 先分清全局还是 cgroup OOM——这决定了后面所有方向
$ dmesg -T | grep -i 'killed process' | tail
[Mon Jun 1 03:14:07 2026] Out of memory: Killed process 9931 (python) total-vm:..., anon-rss:489MB,...
$ dmesg -T | grep -i 'constraint=' | tail
... constraint=CONSTRAINT_MEMCG ... oom_memcg=/kubepods.slice/.../pod<uid>/...   # → 是 cgroup OOM!

# ② 定位到具体 cgroup，看案发记录
$ cd /sys/fs/cgroup/kubepods.slice/.../pod<uid>/cri-containerd-<id>.scope
$ cat memory.events
oom_kill 3            # 铁证：被自己 limit 杀了 3 次
high 142              # 之前已经在软限流挣扎很久了

# ③ 看钱花哪了——是泄漏(anon) 还是缓存(file)？
$ cat memory.stat | grep -E '^(anon|file|slab|shmem) '
anon          478150656     # 大头是匿名内存 → 真·增长，疑似泄漏
file           10485760     # page cache 很少，排除"读大文件撑爆"
shmem                 0

# ④ 看 limit 与峰值，判断是 limit 太小还是真泄漏
$ cat memory.max memory.peak     # peak (6.x 提供) 是历史峰值
536870912
536870912                        # 峰值正好顶到 max
```

到这一步结论已清晰：**这个容器的匿名内存（堆）涨到了 512 MiB 的 limit，被 cgroup OOM 杀**。下一步进容器用 5.5 节的 heap profiler（Go pprof / jemalloc / memleak）抓泄漏点；短期止血则调大 `limits.memory` 或设 `memory.high` 把硬杀变软限流。

**其他生产要点**：

1. **监控盯对指标**：进程看 PSS（不是 RSS 相加、更不是 VSZ）；整机看 `MemAvailable`（不是 `MemFree`）；容器看 `memory.current` + `memory.events` 的 `oom_kill`/`high`。

2. **关键进程上保护**：监控 agent、日志采集 sidecar 等"被杀就抓瞎"的进程，设 `oom_score_adj=-900`；K8s 里给它们 `Guaranteed` QoS。

3. **用 memory.high 做软着陆**：容器 `memory.high` 设在 `max` 下方约 10%，把"突然被 SIGKILL"变成"先变慢、给监控反应时间"。

4. **PSI 进告警**：`/proc/pressure/memory` 的 `some avg60 > 某阈值` 比"内存用了 X%"更早、更准地预示内存麻烦。

5. **容器尽量配少量 swap 或显式禁用并接受**：K8s 1.30+ 支持 swap，给 Burstable 一点 swap 能把 OOM 转成性能降级；但要清楚 swap 颠簸的延迟代价（见 [L05](./L05-精通-物理内存管理与回收.md)）。

---

## 陷阱清单

1. **现象**：整机内存监控一片绿（`MemAvailable` 充足），容器却频繁 `OOMKilled`。
   **原因**：这是 cgroup OOM——容器自己的 `memory.max` 触顶，与整机内存无关。
   **修法**：`dmesg | grep CONSTRAINT_MEMCG` 确认；查该 cgroup 的 `memory.events`（`oom_kill`）和 `memory.stat`；调大 limit 或修泄漏。别去扩机器内存。

2. **现象**：把多个进程的 RSS 相加得到"总内存占用"，远大于机器物理内存，以为要爆。
   **原因**：RSS 把共享内存（libc、共享映射、COW 页）在每个进程里重复计算，相加严重高估。
   **修法**：用 PSS 评估真实占用（`smaps_rollup` 或 `smem`），共享部分按进程数均摊；判断"杀谁释放最多"用 USS。

3. **现象**：`free` 显示可用内存只剩几百 MB，运维紧急扩容/重启。
   **原因**：误把 `MemFree` 当可用量。Linux 把空闲内存几乎全用作 page cache（`Cached`），紧张时会自动回收。
   **修法**：看 `MemAvailable`（已扣除水位、估算可回收缓存），监控告警基于它而非 `MemFree`。

4. **现象**：容器 RSS 看着不高，`memory.current` 却逼近 `max` 并 OOM。
   **原因**：cgroup 内存账本包含 page cache 和 tmpfs(shmem)；进程往 `/dev/shm` 或读大文件填满了缓存，算在容器头上。
   **修法**：看 `memory.stat` 区分 `anon`/`file`/`shmem`；若是 file 缓存，OOM 风险其实低（可回收），但 shmem(tmpfs) 是实打实占用，需限制 `/dev/shm` 大小。

5. **现象**：Go 服务 `inuse_space` 稳定，但容器 RSS 持续不降，怀疑泄漏。
   **原因**：Go runtime 惰性归还内存（`MADV_FREE`），堆已释放但物理页未立即还给 OS；或 cgroup 无内存压力时内核不催收。
   **修法**：看 `runtime.MemStats.HeapInuse` 是否稳定（稳定=非泄漏）；需要立即归还可设 `GODEBUG=madvdontneed=1` 或施加内存压力；真泄漏看 goroutine 数与 `inuse` 差值。

6. **现象**：被杀的进程不是吃内存最多的那个，运维以为内核选错了。
   **原因**：`oom_score` 受 `oom_score_adj` 影响；某进程 adj 为正（如 K8s BestEffort 是 +1000）会被优先选中，哪怕它内存不是最大。
   **修法**：看 `dmesg` 的 `Tasks state` 表里各进程的 `rss` 与 `oom_score_adj` 综合判断；调整关键进程的 adj 或 QoS class。

7. **现象**：机器在 OOM 前先"假死"几十秒，ssh 都进不去，事后才看到 OOM 日志。
   **原因**：内核 OOM 出手太晚——分配失败前的疯狂直接回收 + swap 颠簸已经让系统停摆，但还没触发 OOM。
   **修法**：部署 systemd-oomd（按 PSI 提前干预）；PSI `full avg10` 进告警；关键交互会话设 `ManagedOOMMemoryPressure`。

8. **现象**：整机内存涨、应用内存却正常，`top` 找不到大户。
   **原因**：内核对象泄漏——slab（dentry/inode）、conntrack 表、fd 泄漏，这些不算在任何进程的 RSS 里。
   **修法**：`slabtop -s c` 看哪类内核对象暴涨；`/proc/sys/fs/file-nr` 看 fd；`conntrack -C` 看连接跟踪表；对症修（如调 `vm.vfs_cache_pressure`、修 fd 泄漏）。

---

## 2026 现状

- **cgroup v2 全面主流**：`memory.max`（硬限）+ `memory.high`（软限流）+ `memory.min`/`low`（保护）+ `memory.events`/`memory.stat` 已是容器内存控制的标准面板，systemd 默认 unified 层级（详见 [L17 Cgroup v2](./L17-精通-Cgroup-v2.md)）。`memory.peak` 等只读指标让历史峰值排障更方便。
- **systemd-oomd 普及**：systemd 247+ 自带、主流发行版默认启用基于 PSI 的用户态 OOM，在内核 OOM 之前按压力提前杀整组，显著改善"机器假死才被救"的体验。
- **PSI 成压力信号事实标准**：`/proc/pressure/{memory,io,cpu}` 与 cgroup 级 PSI 被监控、弹性伸缩、oomd 广泛采用，比"用量百分比"更能预示真实卡顿。
- **MGLRU 改善回收**：多代 LRU（6.1+ 可用、渐成默认）让内存回收更精准、OOM 前的颠簸更少，间接降低误杀（见 [L05](./L05-精通-物理内存管理与回收.md)）。
- **eBPF 持续剖析成熟**：`memleak`、`stackcount` 等 bcc 工具加上 Parca/Pyroscope 的持续内存剖析，让"线上抓 native 泄漏"成为常规操作，逐步取代只能离线跑的 valgrind（见 [L19 eBPF](./L19-精通-eBPF.md)）。
- **K8s 内存语义清晰化**：QoS → `oom_score_adj` 的映射、`OOMKilled` 与 `memory.events` 的对应、1.30+ 的 swap 支持，让"容器为什么被杀"比早年可解释得多。

参见 [L04 虚拟内存与分页](./L04-精通-虚拟内存与分页.md)（VSZ/RSS 的虚拟内存根基）、[L05 物理内存管理与回收](./L05-精通-物理内存管理与回收.md)（回收/水位/PSI/swap）、[L02 进程与线程模型](./L02-精通-进程与线程模型.md)（COW 与共享页）、[L17 Cgroup v2](./L17-精通-Cgroup-v2.md)（内存控制器全貌），以及跨专题 [cloud-native C06 资源管理](../cloud-native/C06-精通-Scheduling-与资源管理.md)、[golang G22 pprof](../golang/G22-精通-Go-pprof-性能剖析.md)。

---

## 练习题

1. 写一段 C 程序，`malloc` 并**逐字节写满** 200 MB，在另一终端用 `cat /proc/<pid>/oom_score` 观察分数变化；再把 `oom_score_adj` 设为 `-1000`，对比分数，解释 adj 如何改变结果。

2. 一台 32 GB 机器 `MemFree` 只剩 800 MB，但 `MemAvailable` 显示 16 GB。解释这台机器到底紧不紧张，`MemAvailable` 这 16 GB 大致来自哪几部分（对照 `/proc/meminfo` 的 `Cached`/`SReclaimable`）。

3. 100 个进程共享一个 50 MB 的动态库。分别估算它们 RSS 之和、PSS 之和、单个进程的 USS（假设各自另有 10 MB 私有堆），解释为什么监控应该用 PSS 之和。

4. 给一个 Go 服务接入 pprof，制造一个 goroutine 泄漏（goroutine 阻塞在无人写入的 channel 上），用 `/debug/pprof/goroutine` 和 `-base` 两次 heap 对比，定位泄漏点。

5. 解释 `memory.high` 和 `memory.max` 的区别：当一个 cgroup 的用量分别超过这两者时，进程各自会经历什么？为什么生产建议 `high < max`？

6. 区分这三种 OOM 机制的触发依据、杀的粒度、出手时机：systemd-oomd、cgroup OOM、全局 OOM。各举一个最适合它登场的场景。

7. （排障）某 Pod 反复 `OOMKilled`（Exit 137），但 `kubectl top pod` 看内存只到 limit 的 70%。给出你的诊断假设（提示：top 的采样间隔 vs 瞬时峰值、page cache、tmpfs），以及用 `memory.events`/`memory.stat`/`memory.peak` 验证的步骤。

8. （排障）整机 `MemAvailable` 持续下降、最终全局 OOM，但 `top`/`ps` 按 RSS 排序的所有进程加起来远不到物理内存。`slabtop` 显示 `dentry` 缓存有 8 GB。解释这是不是泄漏、`dentry` 为何会涨这么大、`SReclaimable` 是否意味着它会被自动回收、你会怎么进一步定位和缓解。

---

## 参考答案

1. 逐字节写满 200 MB 会触发 demand paging 把约 200 MB 匿名页兑现成物理内存，进程 RSS 涨到约 200 MB，`oom_score` 随之上升（基础分≈RSS+swap+页表，归一化到 0~1000）。把 `oom_score_adj` 设为 `-1000` 后，`oom_score` 被压到下界（0），OOM killer 会绕过它——adj 按 `total_pages` 比例叠加到分数上，`-1000` 相当于减去全部内存的等效分，使其"免死"。这说明 adj 是人为偏移 oom_score 的旋钮，可保护（负）或牺牲（正）特定进程。

2. 这台机器其实不紧张。`MemFree` 只有 800 MB 是因为内核把空闲内存几乎全拿去做 page cache（闲着才是浪费）。`MemAvailable` 的 16 GB 来自：`MemFree` 减去 low 水位保留，加上可回收的 page cache（`Cached` 中可回收的大部分，活跃文件页留一部分），加上可回收 slab（`SReclaimable` 的大部分，同样扣掉水位保留）。即 16 GB ≈ free + 大部分 Cached + 大部分 SReclaimable。监控应基于 `MemAvailable` 判断，结论是健康。

3. RSS 之和：每个进程都把 50 MB 共享库全算上，再加各自 10 MB 私有堆，100 进程 = 100×(50+10) = 6000 MB（严重高估，库实际只占 50 MB）。PSS 之和：共享库按进程数均摊，每进程算 50/100=0.5 MB + 10 MB 私有 = 10.5 MB，100 进程 = 1050 MB，正好等于真实物理占用（50 MB 库 + 100×10 MB 私有 = 1050 MB）。单进程 USS：只算独占部分 = 10 MB（不含共享库）。监控应用 PSS 之和的原因：RSS 相加会把共享库重复计算成 100 倍、严重高估，而 PSS 把共享部分均摊，相加恰好等于真实物理占用，是最诚实的总量单值。

4. 接入：导入 `net/http/pprof` 并起一个 HTTP server 暴露 `/debug/pprof/`。制造泄漏：起大量 goroutine 阻塞在 `<-ch`（无人写入的 channel），它们永不退出。定位：(1) 访问 `/debug/pprof/goroutine?debug=1` 看 goroutine 数量随时间单调增长，栈顶都停在同一个 channel 接收处即泄漏点；(2) 在两个时间点各抓一次 heap（`go tool pprof http://.../debug/pprof/heap` 保存），用 `go tool pprof -base heap_t1 heap_t2` 看增量，差值集中在那些 goroutine 持有/引用的对象分配栈。两者共同指向阻塞的 channel 接收代码行。

5. `memory.high` 是软限流：用量超过它时进程申请内存被人为延迟（throttle）并触发激进回收，进程变慢但不被杀。`memory.max` 是硬上限：用量超过它且回收无效时，cgroup OOM killer 杀掉 cgroup 内进程（SIGKILL）。超过 high：进程经历变慢 + 激进回收（软着陆）；超过 max：进程被直接杀。生产建议 `high < max`（如 max 下方约 10%）的原因：让系统在撞硬墙前先"踩刹车"——给监控反应时间、给扩容窗口，把"突然被 SIGKILL"变成"先变慢可观测"，避免硬杀。

6. systemd-oomd：用户态、依据 PSI 内存压力 + swap 使用率、粒度是整个 cgroup、出手最早（"还没真满但很卡"）——适合保护交互/系统会话，防一个失控进程让整机假死到 ssh 都进不去。cgroup OOM：内核态、依据 `memory.max` 触顶、粒度是 cgroup 内单进程、容器超限时出手——适合容器资源边界，限定单容器超额的影响范围。全局 OOM：内核态、依据整机水位击穿、粒度是全机进程打分、最后兜底——适合物理机内存真正耗尽时的最终安全网，不应被指望但必须保留。

7. 诊断假设：(1) `kubectl top` 是周期采样（如 15~30s 间隔），抓不到两次采样之间的瞬时内存峰值，进程可能短时冲到 limit 被杀而 top 只显示 70%；(2) cgroup 内存账本含 page cache 和 tmpfs，进程往 `/dev/shm` 写或读大文件填满缓存，把容器顶到 max，而这些不计入进程 RSS（top 的内存）；(3) 真实增长发生在采样间隙。验证步骤：`cat memory.events` 看 `oom_kill` 是否 > 0（确认 cgroup OOM）；`cat memory.peak` 看历史峰值是否顶到 max（证实瞬时峰值超限）；`cat memory.stat` 区分 `anon`/`file`/`shmem`——若 shmem/file 大则是缓存/tmpfs 撑爆，若 anon 大且接近 max 则是堆增长/瞬时峰值。据此决定调大 limit、限制 `/dev/shm`、或抓堆 profiler。

8. 这通常不是传统意义的应用泄漏，而是内核可回收缓存膨胀（dentry 是目录项缓存）。`dentry` 涨到 8 GB 的原因：某进程在疯狂 `open`/`stat`/遍历海量文件（如扫描大目录树、负缓存大量不存在路径），内核为每个路径项缓存 dentry。`SReclaimable` 表示这部分 slab 在内存紧张时**可以被回收**——理论上内存压力下内核会收缩它，不应直接导致 OOM。但若同时存在大量匿名内存增长、或回收速度跟不上分配速度、或这些 dentry 被引用（pinned）无法回收，仍可能走到 OOM。进一步定位：`slabtop -s c` 确认是 dentry/inode；用 `bpftrace`/`opensnoop` 抓哪个进程在狂 open/stat 文件；查 `vm.vfs_cache_pressure`（调高使内核更积极回收 dentry/inode）。缓解：定位并修复狂扫文件的进程行为；临时可 `echo 2 > /proc/sys/vm/drop_caches` 回收（生产慎用）；调高 `vm.vfs_cache_pressure`；若是不可回收增长则查内核模块/驱动。
