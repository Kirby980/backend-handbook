# 精通 Cgroup v2：统一层级、cpu/memory/io/pids、throttling、PSI、systemd 与容器落地

> 课程编号：L17
> 路线图来源：Linux · 模块六 容器与隔离
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：cgroup v2 全面主流、systemd 默认 unified、PSI、memory.high 软限）

---

## 引言：容器的「0.5 核 / 512Mi」是怎么落地的

Kubernetes 里写 `resources.limits: { cpu: "500m", memory: "512Mi" }`，这两个数字最终变成内核里两个文件的内容：`cpu.max` 和 `memory.max`。限制它们的子系统就是 **cgroup（control group）**。如果说 [L16 Namespace](./L16-精通-Namespace.md) 负责「隔离视图」（让进程以为自己独占系统），那 cgroup 负责「限制资源」（CPU、内存、I/O、进程数）。namespace + cgroup = 容器的两条腿。

本篇讲清 cgroup v1 为什么乱、v2 的统一层级好在哪、四个核心 controller 怎么用，以及一个最高频的生产问题：**为什么容器 CPU 没到 limit 却频繁卡顿（CPU throttling）**。CPU 配额底层是 CFS bandwidth，见 [L03 调度](./L03-精通-CPU-调度-CFS-到-EEVDF.md)；内存超限触发的 OOM 见 [L06 OOM](./L06-精通-OOM-与内存诊断.md)；io 限速见 [L10 块设备](./L10-精通-块设备与IO调度.md)；K8s 的 requests/limits 映射见 [cloud-native C06](../cloud-native/C06-精通-Scheduling-与资源管理.md)。

---

## 第一章 cgroup 是什么

cgroup 把进程组织成**树形层级**，对每个节点（一个目录）施加资源控制。控制能力由 **controller（子系统）** 提供：cpu、memory、io、pids 等。接口是文件系统 `/sys/fs/cgroup`。

```bash
# v2 下：创建一个 cgroup，把当前 shell 放进去，限到 0.5 核
cd /sys/fs/cgroup
mkdir demo
echo "+cpu +memory" > cgroup.subtree_control   # 在父级开启要下发的 controller
echo $$ > demo/cgroup.procs                     # 把当前进程加入
echo "50000 100000" > demo/cpu.max              # quota=50ms / period=100ms = 0.5 核
```

把一个进程写入 `cgroup.procs`，它（及其后续子进程）就受该 cgroup 约束。

### 1.1 三个核心概念与历史

cgroup 由 Google 工程师在 2007 年以「process containers」之名引入，2008 年合入 2.6.24。它建立在三个概念上：

| 概念 | 含义 |
|---|---|
| **cgroup** | 一组进程的集合（文件系统里就是一个目录） |
| **controller**（subsystem） | 一种资源的控制器：cpu、memory、io、pids… |
| **hierarchy** | cgroup 组成的树。v1 每个 controller 一棵树；v2 全部共用一棵 |

查看任意进程归属哪个 cgroup：

```bash
cat /proc/$$/cgroup
# 0::/user.slice/user-1000.slice/session-3.scope   # v2 只有一行 "0::<path>"
```

### 1.2 进程粒度与线程粒度

`cgroup.procs` 是**进程级**归属（一个 TGID 的所有线程一起进）。若要按**线程**精细划分（罕见，如把不同线程放进不同 cpu 分组），需把 cgroup 切到 **threaded** 模式：

```bash
echo threaded > demo/sub/cgroup.type   # 该子树变为线程域
echo <tid> > demo/sub/cgroup.threads   # 按线程加入
```

`cgroup.type` 取值 `domain`（默认）/`threaded`/`domain threaded`/`domain invalid`。绝大多数场景用进程级即可。

### 1.3 cgroup.events：感知状态

```bash
cat demo/cgroup.events
# populated 1     # 内部是否还有活进程（1=有）；常用于「等 cgroup 空了再回收」
# frozen 0        # 是否被冻结（见 cgroup.freeze）
```

`populated` 配合 inotify 可让管理程序在 cgroup 变空时收到通知——容器运行时据此判断容器真正退出。

---

## 第二章 v1 vs v2：从混乱到统一层级

### 2.1 v1 的问题

cgroup v1 里**每个 controller 是独立的层级树**：cpu 一棵树、memory 一棵树、blkio 一棵树……一个进程在不同 controller 里可以处于不同位置。这导致：

- controller 间无法协同（如 memory 与 io 回写的配合）；
- 管理复杂、语义不一致；
- 容器运行时与 systemd 各自挂载，冲突频发。

### 2.2 v2 的统一层级（unified hierarchy）

cgroup v2 **只有一棵树**，所有 controller 作用在同一层级上。关键规则：

| 规则 | 说明 |
|---|---|
| 单一层级 | 所有 controller 共享一棵 cgroup 树 |
| `cgroup.controllers` | 本 cgroup 可用的 controller |
| `cgroup.subtree_control` | 向子节点下发哪些 controller（`+cpu`/`-cpu`） |
| **no internal process** | 启用了 controller 的非根 cgroup，进程只能待在**叶子**节点 |

```
/sys/fs/cgroup/                 (root)
├── cgroup.controllers          # 可用：cpu memory io pids ...
├── cgroup.subtree_control      # 下发给子级
├── system.slice/               # systemd 管理的服务
│   └── nginx.service/
│       ├── cpu.max
│       ├── memory.max
│       └── cgroup.procs
└── kubepods.slice/             # K8s 的 Pod cgroup
```

2026 年 cgroup v2 已**全面主流**，systemd 默认 unified，多数发行版与 K8s（含 kubelet `cgroupDriver: systemd`）都在 v2 上运行；v1 进入维护态。判断当前模式：

```bash
stat -fc %T /sys/fs/cgroup/      # cgroup2fs = v2 ; tmpfs = v1(混合)
```

### 2.3 v1/v2 全面对比

| 维度 | cgroup v1 | cgroup v2 |
|---|---|---|
| 层级 | 每 controller 一棵独立树 | 单一统一树 |
| 进程归属 | 各 controller 可不同 | 全局唯一位置 |
| 内部进程 | 允许中间节点有进程 | 非根启用 controller 后只能在叶子 |
| memory+io 协同 | 做不到（脏页回写计费错乱） | 可协同（回写计入发起 cgroup） |
| 接口 | `cpu.cfs_quota_us` 等分散命名 | `cpu.max` 等统一命名 |
| 线程粒度 | 直接支持 | 需 threaded 模式 |
| 现状 | 维护态 | **主流默认** |

### 2.4 controller 的「自顶向下」启用

v2 里 controller 必须**从根往下逐层下发**。父级用 `cgroup.subtree_control` 声明「我把哪些 controller 交给子级」，子级的 `cgroup.controllers` 才会出现它：

```bash
cd /sys/fs/cgroup
cat cgroup.controllers              # 根上可用的：cpuset cpu io memory pids ...
echo "+cpu +memory" > cgroup.subtree_control   # 下发给直接子级
mkdir demo
cat demo/cgroup.controllers         # 现在 demo 里能看到 cpu memory
echo "+cpu" > demo/cgroup.subtree_control      # demo 再下发给它的子级
```

**no internal process 规则**的由来：若一个 cgroup 既有子 cgroup 又直接挂进程，资源在「进程」和「子组」之间如何分配会产生歧义。v2 强制进程只能待在叶子，消除歧义。根 cgroup 是唯一例外。

### 2.5 混合模式（hybrid）

过渡期系统可能是 hybrid：一部分 controller 走 v2、一部分仍 v1。`mount | grep cgroup` 能看到 `cgroup2`（`/sys/fs/cgroup` 或其下 `unified/`）与若干 v1 挂载并存。新系统已基本全 v2（`systemd.unified_cgroup_hierarchy=1` 是默认）。

---

## 第三章 四个核心 controller

### 3.1 cpu：weight 与 max

| 文件 | 含义 |
|---|---|
| `cpu.weight` | 相对权重（默认 100，范围 1–10000），CPU 紧张时按权重分配。对应 v1 `cpu.shares` |
| `cpu.max` | `"$quota $period"`（μs），如 `50000 100000` = 每 100ms 最多用 50ms CPU = 0.5 核。`max` 表示不限 |
| `cpu.stat` | 统计：`usage_usec`、`nr_throttled`、`throttled_usec`（**排查 throttling 的关键**） |

`cpu.max` 底层就是 **CFS bandwidth**（带宽控制），原理见 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md)。

**CFS bandwidth 怎么工作**：内核每隔一个 `period`（默认 100ms）给该 cgroup 补满 `quota` 的运行时间额度。cgroup 内的线程消耗额度，**一旦在本 period 内用尽，所有线程被 throttle（冻结）直到下个 period 补充**。这正是「CPU 用量没到 limit 却卡顿」的机理——多线程在 period 前段并行把额度瞬间烧光，后段全员饿等。

```bash
cat demo/cpu.stat
# usage_usec 12345678      # 累计使用(μs)
# nr_periods 1000          # 经历的 period 数
# nr_throttled 37          # 被限流的 period 数  ← 持续增长 = 有 throttling
# throttled_usec 540000    # 累计被冻结时长     ← 延迟毛刺的元凶
```

**`cpu.max.burst`**（5.14+）允许积累未用完的额度，应对突发：设一个 burst 值后，空闲 period 攒下的额度可在繁忙 period 突发使用，缓解「平均不超限但瞬时被 throttle」。

```bash
echo "50000 100000" > demo/cpu.max         # 0.5 核
echo 20000 > demo/cpu.max.burst            # 允许突发 20ms
```

**`cpu.weight`**（默认 100）只在 CPU 竞争时按比例分配，不设上限；`cpu.weight.nice` 提供与 nice 值对齐的另一种表达。**weight 决定「抢得到多少」，max 决定「最多用多少」**——K8s 的 requests→weight、limits→max 正是这个分工。

> 相关的 `cpuset` controller（`cpuset.cpus`/`cpuset.mems`）把 cgroup 绑定到特定 CPU 核与 NUMA 节点，延迟敏感服务常用（呼应 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md) 的 CPU 亲和与 NUMA）。

### 3.2 memory：max / high / low / min

| 文件 | 含义 |
|---|---|
| `memory.max` | **硬上限**。超过且回收不回来 → 该 cgroup 内触发 **OOM kill**（容器 OOMKilled） |
| `memory.high` | **软上限**。超过不直接杀，而是**节流分配 + 积极回收**，把内存压回 high 以下（拖慢但不死） |
| `memory.low` / `memory.min` | 回收**保护**：尽量/绝对不回收到该值以下 |
| `memory.current` | 当前用量 |
| `memory.events` | 计数：`high`、`max`、`oom`、`oom_kill`（**确认是否被 OOM 的依据**） |
| `memory.stat` | 细分：anon、file、slab、sock... |

`memory.max` 与 `memory.high` 的区别是面试高频点：**max 是「红线，越过就杀」，high 是「减速带，越过就限速回收」**。容器编排里 limit 通常映射到 `memory.max`。

**四个阈值构成一条「保护—限流—击杀」的梯度**：

```
   用量低 ───────────────────────────────► 用量高
   │ min │ low │        正常         │ high │  max │
   └─────┴─────┴────────────────────┴──────┴──────┘
     ▲      ▲                          ▲       ▲
     │      │                          │       └ 硬上限：超且回收不动 → cgroup OOM kill
     │      │                          └ 软上限：超 → 节流分配 + 积极回收
     │      └ low：软保护，全局回收时尽量不动这部分
     └ min：硬保护，绝不回收（即便全局内存吃紧）——给关键服务保底
```

`memory.min`/`memory.low` 用于**保护**关键 cgroup 不被回收饿死（如把数据库的 page cache 保住）；`memory.high`/`memory.max` 用于**限制**。

**`memory.swap.max`**：单独限制该 cgroup 能用多少 swap（设 0 = 禁止换出，常用于延迟敏感服务，强制其内存常驻，超限直接走 OOM 而非颠簸）。

**`memory.reclaim`**（5.19+）：向它写入字节数可**主动触发回收**，运维可用于「软驱逐」而不杀进程：

```bash
echo "100M" > demo/memory.reclaim     # 主动从该 cgroup 回收约 100M
```

**关键观测文件**：

```bash
cat demo/memory.current      # 当前用量
cat demo/memory.peak         # 历史峰值(便于定容)
cat demo/memory.events
# low 0
# high 12         # 触发 high 限流的次数 ← 频繁说明 high 设小了/确实吃紧
# max 3           # 触达 max 的次数
# oom 1           # 触发 OOM 的次数
# oom_kill 1      # 实际杀进程次数 ← 容器 OOMKilled 的铁证
cat demo/memory.stat         # anon/file/kernel/slab/sock/shmem... 逐项，定位内存去向
```

### 3.2.1 内核内存与 socket 内存

v2 默认把**内核内存（slab、内核栈）和 socket 缓冲**也计入 `memory.current`。这意味着：一个连接数爆炸的服务，可能因 `sock`（TCP 缓冲，呼应 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）和 `slab`（如 dentry 缓存，呼应 [L07](./L07-精通-VFS-与文件系统.md)）撑爆 `memory.max` 被 OOM，而应用堆内存并不大——看 `memory.stat` 才能识破。

### 3.3 io：限速与权重

| 文件 | 含义 |
|---|---|
| `io.max` | 按设备限速：`"<major:minor> rbps=… wbps=… riops=… wiops=…"` |
| `io.weight` | 权重分配（需 bfq 等支持，见 [L10](./L10-精通-块设备与IO调度.md)） |
| `io.stat` | 各设备 I/O 统计 |

```bash
# 限制某 cgroup 对 8:0 设备写不超过 10MB/s
echo "8:0 wbps=10485760" > demo/io.max
```

**四种限速维度**：`rbps`/`wbps`（读/写带宽，字节/秒）、`riops`/`wiops`（读/写 IOPS）。设备号 `8:0` 用 `lsblk` 或 `ls -l /dev/sda` 的主次设备号确定。

更高级的两种「目标式」控制（比硬限速更智能）：

| 文件 | 机制 |
|---|---|
| `io.latency` | 给 cgroup 设一个**延迟目标**（如 `8:0 target=10`，10ms）。当它延迟达标时不干预；某低优先 cgroup 拖慢了它，内核就**节流低优 cgroup**。适合「保护关键服务的 I/O 延迟」 |
| `io.cost`（blk-iocost） | 基于成本模型按 `io.weight` 公平分配磁盘吞吐，比硬限速更能压榨设备又保公平。需 `io.cost.qos`/`io.cost.model` 标定 |

**v2 的杀手锏——回写计费**：cgroup v1 无法把异步脏页回写（writeback）的 I/O 正确归属到产生脏页的 cgroup（回写发生在 flusher 线程上下文）。v2 因 memory 与 io 在同一层级，**能把回写 I/O 计入发起方 cgroup**，于是 `io.max`/`io.weight` 对「写大量数据再让内核慢慢刷盘」的负载才真正有效。这也是 v2 相对 v1 的核心优势之一（呼应 [L05](./L05-精通-物理内存管理与回收.md) 脏页回写、[L10](./L10-精通-块设备与IO调度.md) blk-mq）。

```bash
cat demo/io.stat
# 8:0 rbytes=4096000 wbytes=8192000 rios=100 wios=200 dbytes=0 dios=0
```

### 3.4 pids：防 fork bomb

```bash
echo 100 > demo/pids.max     # 该 cgroup 最多 100 个进程/线程
```

容器默认会设 `pids.max` 防止 fork bomb 拖垮节点。

```bash
cat demo/pids.current        # 当前进程/线程数
cat demo/pids.events
# max 5      # 因触达 pids.max 而被拒绝 fork 的次数
```

### 3.5 进程控制：freeze 与 kill

cgroup v2 还提供两个「整组操作」的开关，是容器 pause/stop 的内核基础：

| 文件 | 作用 |
|---|---|
| `cgroup.freeze` | 写 `1` **冻结整组**进程（类似对全组 SIGSTOP，但更彻底，不可被进程绕过），写 `0` 解冻。`docker pause` 即此 |
| `cgroup.kill`（5.14+） | 写 `1` **可靠杀死组内全部进程**（含拒绝处理信号的进程），是「彻底干掉一个容器」的保证 |

```bash
echo 1 > demo/cgroup.freeze   # 暂停整个 cgroup
echo 0 > demo/cgroup.freeze   # 恢复
echo 1 > demo/cgroup.kill     # 杀光（不给逃逸机会）
```

### 3.6 其他常用 controller

| controller | 用途 |
|---|---|
| `cpuset` | 绑定 CPU 核与 NUMA 节点（`cpuset.cpus`/`cpuset.mems`），延迟敏感/NUMA 亲和 |
| `hugetlb` | 限制大页用量（呼应 [L04](./L04-精通-虚拟内存与分页.md) HugePage） |
| `misc` | 杂项标量资源（如 AMD SEV 加密虚机的 ASID 配额） |
| `rdma` | 限制 RDMA 资源 |

---

## 第四章 PSI：cgroup 级压力指标

每个 cgroup 都有 `cpu.pressure`、`memory.pressure`、`io.pressure`（PSI，详见 [L05](./L05-精通-物理内存管理与回收.md)）：

```bash
cat demo/memory.pressure
# some avg10=12.3 avg60=8.1 avg300=2.0 total=...
# full avg10=5.0  ...
```

- `some`：**至少一个**任务因该资源停滞的时间占比；
- `full`：**所有**任务都停滞的时间占比（更严重）。

PSI 让「资源到底卡没卡」可量化，是弹性伸缩、`systemd-oomd`（基于 PSI 决定杀谁）的依据。

**字段含义**：

| 字段 | 含义 |
|---|---|
| `avg10` / `avg60` / `avg300` | 过去 10 / 60 / 300 秒内的停滞时间百分比 |
| `total` | 累计停滞微秒数；两次采样求差比 avg 更精确，适合接入监控 |

**poll 触发器**：可对 `*.pressure` 文件用 `poll()`/`epoll`（呼应 [L08](./L08-精通-IO-多路复用.md)）注册阈值，例如「1 秒窗口内 `some` 停滞超过 150ms 就唤醒我」：

```
some 150000 1000000   # 150ms / 1s 窗口；写入后 poll 该 fd
```

`systemd-oomd`、Android 的 lmkd 正是用 PSI 触发器在**内存压力刚抬头**时就动作，而非等到内核 OOM 那一刻（呼应 [L06](./L06-精通-OOM-与内存诊断.md)、[L05](./L05-精通-物理内存管理与回收.md)）。

---

## 第五章 cgroup + systemd + 容器运行时

### 5.1 systemd 用 cgroup 管理一切服务

systemd 把每个 service 放进自己的 cgroup（`<name>.service`），用 slice 组织层级（`system.slice`/`user.slice`），资源控制通过 unit 指令下发（呼应 [L20 systemd](./L20-精通-systemd-与启动流程.md)）：

```ini
# /etc/systemd/system/myapp.service 的 [Service] 段
CPUQuota=50%          # → cpu.max
MemoryMax=512M        # → memory.max
MemoryHigh=400M       # → memory.high
IOWeight=100          # → io.weight
TasksMax=200          # → pids.max
```

```bash
systemctl set-property myapp.service MemoryMax=1G   # 运行时改
systemd-cgls                                        # 看 cgroup 树
systemd-cgtop                                       # 按 cgroup 看实时资源
```

systemd 把所有进程组织进三类 unit + slice 层级：

| 单元 | 含义 |
|---|---|
| `.service` | 服务进程组 |
| `.scope` | systemd 之外创建、但交给它管理的进程组（容器、用户会话、`systemd-run --scope`） |
| `.slice` | 纯分组节点，构成资源层级：`-.slice` → `system.slice`/`user.slice`/`machine.slice` → 具体 unit |

完整资源指令（各自落到对应 cgroup 文件）：

| systemd 指令 | cgroup 文件 |
|---|---|
| `CPUWeight=` / `CPUQuota=` | `cpu.weight` / `cpu.max` |
| `MemoryMin=` / `MemoryLow=` | `memory.min` / `memory.low` |
| `MemoryHigh=` / `MemoryMax=` | `memory.high` / `memory.max` |
| `MemorySwapMax=` | `memory.swap.max` |
| `IOWeight=` / `IOReadBandwidthMax=` | `io.weight` / `io.max` |
| `TasksMax=` | `pids.max` |
| `AllowedCPUs=` / `AllowedMemoryNodes=` | `cpuset.cpus` / `cpuset.mems` |

**`Delegate=yes`**：把一个 unit 的 cgroup 子树「委派」给它自管。容器运行时（systemd 作 cgroup driver 时）和嵌套 systemd 靠它在自己的子树里自由创建 cgroup，而不与外层 systemd 抢写（systemd 是 cgroup 的唯一写者，见陷阱清单）。

### 5.2 容器运行时与 K8s

`runc`/`crun` 在启动容器时创建 cgroup 并写入 limit；containerd/kubelet 据 Pod 的 requests/limits 计算并下发：

- `requests.cpu` → `cpu.weight`（相对权重，影响争抢时的分配）
- `limits.cpu` → `cpu.max`（硬配额，触发 throttling）
- `limits.memory` → `memory.max`（触发 OOMKilled）

映射细节见 [cloud-native C06](../cloud-native/C06-精通-Scheduling-与资源管理.md)。

**K8s QoS 与 cgroup 层级**：kubelet 按 Pod 的 requests/limits 分三档 QoS，落到不同 cgroup 子树：

| QoS | 条件 | 表现 |
|---|---|---|
| `Guaranteed` | requests == limits | 限额精确，最不易被驱逐 |
| `Burstable` | requests < limits | 可突发到 limit，紧张时较易被回收 |
| `BestEffort` | 无 requests/limits | 无保障，最先被牺牲 |

```
kubepods.slice/
├── kubepods-guaranteed.slice/
│   └── ...-pod<uid>.slice/        # 每个 Pod 一个
│       └── <container-id>/        # 每个容器一个：写 cpu.max / memory.max
├── kubepods-burstable.slice/
└── kubepods-besteffort.slice/
```

节点级驱逐（eviction）与这套层级、PSI 紧密相关，详见 [cloud-native C06](../cloud-native/C06-精通-Scheduling-与资源管理.md)。

---

## 生产实践

**案例（最高频）：容器 CPU 用量没到 limit，却频繁延迟毛刺。**

1. 看 `cpu.stat`：`nr_throttled` 与 `throttled_usec` 持续增长——确认被 **CPU throttling**。
2. 根因：`limits.cpu=500m`（quota=50ms/100ms），但应用是**多线程**（如 Go 默认 `GOMAXPROCS=`宿主核数，或 JVM 按宿主核数起线程池），在 period 前段并行把 50ms quota 瞬间用光，剩余时间被冻结，造成毛刺。
3. 处置：① 让运行时感知 limit——Go 用 `automaxprocs` 或显式设 `GOMAXPROCS` 匹配 limit 核数；JVM 用 `-XX:ActiveProcessorCount`；② 适当调大 limit 或放宽 period；③ 对延迟敏感服务考虑 `static` CPU manager（C06）。这与 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md) 的 CFS bandwidth 同源。

**案例：容器 OOMKilled，但宿主内存充足。**

1. `cat memory.events` 看 `oom_kill` 增长——是 **cgroup 内 OOM**，非全局。
2. 根因：`memory.max` 设得太小，容器内进程峰值超限。
3. 处置：调大 limit，或用 `memory.high` 给一个软上限提前限流回收（拖慢而非杀）。还原全过程见 [L06](./L06-精通-OOM-与内存诊断.md)。

**案例：离线批处理把磁盘打满，拖垮同节点在线服务。**

1. 在线服务 P99 抖动，查 `io.pressure` 的 `some` 飙高。
2. 根因：批处理 cgroup 无 I/O 限制，抢占磁盘带宽（呼应 [L10](./L10-精通-块设备与IO调度.md)）。
3. 处置：给批处理设 `io.max` 硬限速，或给在线服务设 `io.latency` 延迟目标让内核自动节流低优任务；长远用 `io.cost` 做公平。验证 `io.pressure` 回落。

**案例：连接数暴涨的网关被 OOMKilled，但应用堆内存不大。**

1. `memory.events` 的 `oom_kill` 增长，但 `memory.stat` 显示 `anon` 不高、`sock`/`slab` 很大。
2. 根因：v2 把 socket 缓冲（[L12](./L12-精通-TCP-IP-内核实现与调优.md)）与内核 slab 计入 `memory.max`。
3. 处置：调大 limit、优化连接数与缓冲、排查 dentry/inode 缓存膨胀（[L07](./L07-精通-VFS-与文件系统.md)）。

---

## 陷阱清单

1. **现象**：容器 CPU 利用率不高却卡顿 → **原因**：CPU throttling（多线程瞬时打满 quota）→ **修法**：让运行时感知 limit（GOMAXPROCS/ActiveProcessorCount），看 `cpu.stat` 验证。
2. **现象**：`memory.high` 设了但进程没被杀、只是巨慢 → **原因**：high 是软限只限流回收，不 kill → **修法**：理解 high vs max 语义；要硬限用 `memory.max`。
3. **现象**：在启用 controller 的非根 cgroup 直接放进程失败 → **原因**：v2 的 no-internal-process 规则 → **修法**：进程只放叶子 cgroup。
4. **现象**：`echo +cpu > cgroup.subtree_control` 报错 → **原因**：父级未拥有该 controller 或有内部进程 → **修法**：先在更上层下发、清空内部进程。
5. **现象**：在 v2 系统上找 `cpu.cfs_quota_us`（v1 文件）找不到 → **原因**：v2 改名为 `cpu.max` → **修法**：用 v2 文件名；确认 `stat -fc %T /sys/fs/cgroup`。
6. **现象**：容器 fork bomb 拖垮节点 → **原因**：未设 `pids.max` → **修法**：运行时/编排设置 pids 限制。
7. **现象**：手动改了 `/sys/fs/cgroup` 下 systemd 管理的 cgroup，重启失效 → **原因**：systemd 是 cgroup 的「唯一写者」，会覆盖 → **修法**：用 `systemctl set-property` 或 unit 文件，别直接写文件。
8. **现象**：设 `memory.swap.max=0` 后服务更易 OOM → **原因**：禁了 swap，内存峰值无缓冲直接撞 max → **修法**：这是权衡（要延迟稳定就接受，要扛峰值则留 swap）。
9. **现象**：v1 上设 `io.max` 对「写完即返回、内核慢慢刷盘」的负载无效 → **原因**：v1 回写 I/O 无法归属发起 cgroup → **修法**：迁 v2（回写计费）。
10. **现象**：`echo 1 > cgroup.kill` 找不到该文件 → **原因**：内核 < 5.14 → **修法**：用 `cgroup.procs` 遍历逐个 SIGKILL，或升级内核。

---

## 2026 现状

- **cgroup v2 全面主流**，systemd 默认 unified，K8s 推荐 `cgroupDriver: systemd` 且在 v2 上运行；v1 维护态。
- `memory.high` 软限 + `systemd-oomd`（基于 PSI）的「优雅降级式」内存管理逐步普及，替代「直接 OOM kill」。
- PSI 成为弹性伸缩与过载保护的标准信号。
- CPU throttling 仍是容器化最常见的性能坑，「让运行时感知 cgroup limit」（automaxprocs 等）已是工程共识。

---

## 练习题

1. （⭐）cgroup 与 namespace 在容器中各负责什么？为什么说二者缺一不可？
2. （⭐）cgroup v1 与 v2 在「层级」上的根本区别是什么？如何判断当前系统用的是哪个？
3. （⭐⭐）`cpu.max` 的两个数字各是什么含义？`"50000 100000"` 等于多少个 CPU？
4. （⭐⭐）`memory.max` 与 `memory.high` 的行为差异？超限后分别发生什么？
5. （⭐⭐）cgroup v2 的「no internal process」规则是什么？为什么这么设计？
6. （⭐⭐⭐）容器 CPU 没到 limit 却卡顿，给出从 `cpu.stat` 到运行时配置的完整排查与根治步骤，并说明它和 CFS bandwidth、Go GOMAXPROCS 的关系。
7. （⭐⭐⭐）容器 OOMKilled 但宿主内存充足，如何用 `memory.events`/`memory.current` 确认是 cgroup 内 OOM？怎么处置？
8. （⭐⭐⭐）为什么不应直接 `echo` 修改 systemd 管理的 service cgroup 文件？正确的运行时调整方式是什么？
9. （⭐⭐）`memory.min`/`low`/`high`/`max` 四个阈值构成怎样的「保护—限流—击杀」梯度？哪些用于保护、哪些用于限制？
10. （⭐⭐⭐）解释 cgroup v2 相对 v1 在「脏页回写计费」上的关键改进，以及它为何让 `io.max` 对写负载真正有效（结合 [L05](./L05-精通-物理内存管理与回收.md)/[L10](./L10-精通-块设备与IO调度.md)）。
11. （⭐⭐⭐）`cgroup.freeze` 与 `cgroup.kill` 各自的用途？为什么 `cgroup.kill` 比逐个进程发 SIGKILL 更可靠？
