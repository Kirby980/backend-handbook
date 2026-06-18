# 精通块设备与 I/O 调度：bio、blk-mq、none/mq-deadline/bfq、iostat、biolatency

> 课程编号：L10
> 路线图来源：Linux · 模块三 文件与 I/O
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：blk-mq 是唯一块层路径、单队列 legacy 已删、NVMe 普及 none 调度器常见、cgroup v2 io.max 限流）

---

## 引言：%util 100% 不等于磁盘满载

值班时最容易被误导的指标，是 `iostat` 里的 **%util**。

```bash
$ iostat -x 1
Device  r/s   w/s   rkB/s   wkB/s  r_await w_await aqu-sz  %util
nvme0n1 1200  300   48000   12000   0.08    0.12    0.18   99.50
```

`%util` 99.5%——很多人一看就拍板："磁盘满了，IO 瓶颈，赶紧加盘！"然后加了盘问题依旧，因为这是**误读**。

`%util` 的定义是"采样周期内，块设备至少有一个 IO 在处理的时间占比"。在**单队列**机械盘时代，一次只能处理一个 IO，`%util` 接近 100% 确实意味着盘忙到没有空闲、基本饱和。但 **NVMe SSD 有几十上百个硬件队列、能同时处理成百上千个并发 IO**——只要任意时刻有一个 IO 在跑，`%util` 就是 100%，可它的真实利用率可能才 5%！一块能跑 50 万 IOPS 的 NVMe，你给它 1000 IOPS，`%util` 照样能飙到 100%，但它闲得发慌。

在 NVMe / 多队列时代，判断磁盘是否真饱和，要看 **aqu-sz（平均队列深度）、await（IO 平均耗时）、以及实际 IOPS/带宽 与设备额定值的比值**，而不是 `%util`。这就是本篇要建立的核心心智模型。

我们从块层架构（bio → request → request_queue）讲起，拆解 2026 年唯一的块层路径 **blk-mq 多队列**（单队列 legacy 已被内核删除）、四种 I/O 调度器（none / mq-deadline / kyber / bfq）的原理与选型、IO 合并排序预读 plug/unplug，再到诊断工具（iostat 各列精确含义、biolatency/biosnoop）、NVMe 现代存储与 cgroup `io.max` 限流（呼应 L17）。读完你应该能正确解读 iostat、用 bcc 工具定位"await 高到底卡在哪一段"，并为不同设备选对调度器。

---

## 第一章 块层架构：bio、request、request_queue、gendisk

### 1.1 一次写从 page cache 到盘片的路径

接上 L07：buffered write 标脏的 page、或 `fsync`/O_DIRECT 触发的写，最终要变成对块设备的请求。块层（block layer）就是文件系统与设备驱动之间的中间层。

```
 文件系统 (ext4/xfs)  /  O_DIRECT
        │  提交 IO（按 page/folio）
        ▼
   ┌─────────┐  bio：描述"把这些内存页读/写到设备的这些扇区"
   │  bio    │  （块层的基本 IO 单位）
   └─────────┘
        │  合并 / 排序
        ▼
   ┌─────────┐  request：一个或多个相邻 bio 合并成一个请求
   │ request │
   └─────────┘
        │  经 I/O 调度器
        ▼
   ┌──────────────┐  blk-mq：软件队列 → 硬件队列
   │ request_queue│
   └──────────────┘
        │
        ▼
   设备驱动 (nvme / scsi) → 硬件 → 盘片/闪存
```

### 1.2 bio：块层的基本 IO 单位

`bio`（block I/O）描述一次 IO 涉及的内存与设备位置：

```c
// include/linux/blk_types.h (简化)
struct bio {
    struct bio          *bi_next;     // 链到下一个 bio
    struct block_device *bi_bdev;     // 目标块设备
    blk_opf_t           bi_opf;       // 操作 + 标志 (READ/WRITE/FLUSH/FUA...)
    struct bvec_iter    bi_iter;      // 起始扇区 bi_sector + 剩余长度
    bio_end_io_t        *bi_end_io;   // 完成回调
    struct bio_vec      *bi_io_vec;   // ★ 向量：每段 (page, offset, len)
    unsigned short      bi_vcnt;      // 段数
};
```

`bio_vec` 是关键——一个 bio 可以描述**多段分散的内存页**（scatter-gather），对应设备上**连续的扇区**。这让"一个文件页缓存里分散的脏页"能合并成"对盘上连续区域的一次传输"，是 IO 合并的物理基础。`FLUSH`/`FUA` 标志对应 L07 讲的 fsync 屏障——`FLUSH` 让设备把易失写缓存刷到持久介质，`FUA`（Force Unit Access）让这一笔写直接落持久介质。

### 1.3 request 与 request_queue、gendisk

- **request**：调度层的单位。相邻的 bio（设备扇区连续）会被**合并**成一个 request，减少下发给硬件的请求数。
- **request_queue**：每个块设备一个，是块层提交、合并、调度、下发的中枢。
- **gendisk**：代表一个磁盘设备（如 `nvme0n1`），关联其分区（`nvme0n1p1`…）和 `request_queue`。

```bash
# 块设备的队列参数都暴露在 sysfs
$ ls /sys/block/nvme0n1/queue/
nr_requests  scheduler  read_ahead_kb  rotational  nr_zones
max_sectors_kb  nomerges  ...
$ cat /sys/block/nvme0n1/queue/rotational   # 0=SSD/NVMe, 1=机械盘
0
$ cat /sys/block/nvme0n1/queue/nr_requests  # 队列深度
1023
```

---

## 第二章 blk-mq 多队列：软件队列 / 硬件队列

### 2.1 为什么单队列 legacy 被删

老块层是**单请求队列**：所有 CPU 提交的 IO 都进一个全局 `request_queue`，由一把队列锁保护。机械盘时代这没问题——盘本来就一次干一个 IO，瓶颈在磁头寻道，软件这点锁竞争无所谓。

但 SSD/NVMe 来了之后，单队列成了灾难：
- 一块 NVMe 能跑几十万到上百万 IOPS，多核同时提交，**那把全局队列锁成了热点**，锁竞争把 CPU 烧光，IOPS 上不去。
- NVMe 硬件本身就支持**多个提交/完成队列**（每核一对），单队列白白浪费了硬件并行能力。

于是 **blk-mq（Multi-Queue Block Layer，多队列块层）** 诞生（3.13 引入，逐步成熟）。到 2026 年，**单队列 legacy 路径已被内核彻底删除（5.0 移除），blk-mq 是唯一的块层路径**——所有块设备，无论 NVMe、SATA SSD 还是机械盘，都走 blk-mq。

### 2.2 两级队列：软件队列 + 硬件队列

blk-mq 用两级队列消除全局锁：

```
   CPU0   CPU1   CPU2   CPU3      ← 每 CPU 一个软件队列（无锁/低锁）
    │      │      │      │
   ┌▼─┐  ┌▼─┐  ┌▼─┐  ┌▼─┐
   │SW│  │SW│  │SW│  │SW│         软件队列 (ctx)：本地提交，避免跨核争锁
   └┬─┘  └┬─┘  └┬─┘  └┬─┘
    └──┬───┴──┬───┴───┘
       ▼      ▼
    ┌─────┐ ┌─────┐                硬件队列 (hctx)：映射到设备的硬件队列
    │ HW0 │ │ HW1 │  ...           NVMe 每个 hctx 对应一对 SQ/CQ
    └──┬──┘ └──┬──┘
       ▼       ▼
    NVMe 硬件队列（设备并行处理）
```

- **软件队列（per-CPU ctx）**：每个 CPU 一个，IO 提交先进本地软件队列，**避免跨核抢锁**。
- **硬件队列（hctx）**：映射到设备真实的硬件队列。NVMe 通常每核一对队列，软硬件队列接近 1:1，几乎全程无锁。SATA SSD/机械盘只有 1 个硬件队列，多个软件队列汇聚到它。

```bash
$ ls /sys/block/nvme0n1/mq/      # 每个目录是一个硬件队列
0  1  2  3  4  5  6  7   # 8 个硬件队列（通常 = CPU 数或队列数）
$ cat /sys/block/nvme0n1/mq/0/nr_tags   # 该硬件队列的标签数（可并发 IO 数）
1023
```

blk-mq 用**标签（tag）**集合管理在途请求——每个 in-flight IO 占一个 tag，tag 用完即队列满。这天然支持了 NVMe 的高并发。

---

## 第三章 I/O 调度器：none / mq-deadline / kyber / bfq

### 3.1 调度器在 blk-mq 里的角色

调度器决定 request 从软件队列到硬件队列的**顺序与合并策略**。blk-mq 时代的四个调度器：

| 调度器 | 思想 | 开销 | 适合设备 | 一句话 |
|---|---|---|---|---|
| **none** | 不调度，FIFO 直通硬件 | 最低 | **NVMe / 高速 SSD** | 设备自己够快够并行，软件别添乱 |
| **mq-deadline** | 给每个 IO 设截止期，读优先，防饿死 | 低 | **SATA SSD / 机械盘 / 通用** | 保延迟、防写饿读 |
| **kyber** | 按目标延迟自适应限流，简单 | 低 | 高速多队列设备 | 轻量、自调 |
| **bfq** | 按进程公平分配带宽 + 权重 | **高** | **桌面 / 交互式 / 机械盘** | 公平、低延迟交互，CPU 开销大 |

### 3.2 none：NVMe 的默认与最优

`none`（也叫 noop）**不做任何重排，FIFO 直接下发**。逻辑：NVMe 有海量硬件队列、内部有自己的 FTL 调度、寻道开销为零——**软件层再排序纯属添乱、还增加延迟和 CPU**。所以 NVMe 上 `none` 通常是最优，也是现代发行版对 NVMe 的默认。

```bash
$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq      # 方括号是当前生效的
```

### 3.3 mq-deadline：防止读被写饿死

机械盘 / SATA SSD 上，`mq-deadline` 是稳妥默认。它给每个 IO 打**截止期**（读默认更短，如 500ms；写更长，如 5s），主体按 LBA（扇区号）排序合并以减少寻道，但一旦某 IO 接近截止期就**插队优先处理**，防止它被无限推迟（饿死）。

为什么读优先：读通常是同步阻塞的（进程等着数据才能继续，D 状态），写多是异步回写（writeback，进程不等）。让读优先能显著改善交互延迟。

```bash
$ cat /sys/block/sda/queue/iosched/read_expire   # 读截止期(ms)
500
$ cat /sys/block/sda/queue/iosched/write_expire
5000
```

### 3.4 bfq：公平与交互延迟，但 CPU 贵

**BFQ（Budget Fair Queueing）** 给每个进程/cgroup 分配 IO "预算"，按权重公平分带宽，并对交互式应用做延迟优化（点开应用立刻响应而不被后台拷贝拖死）。**桌面体验极佳**——后台大文件复制时前台仍流畅。但它**算法复杂、每 IO 的 CPU 开销显著**，在高 IOPS 的 NVMe 服务器上反而成为瓶颈（CPU 被调度器吃掉，IOPS 上不去）。所以 **bfq 是桌面/交互场景的选择，不是服务器 NVMe 的选择**。

### 3.5 选型决策

```
设备是 NVMe / 高速 SSD？
   └─ 是 → none（默认最优，低延迟低开销）
        └─ 需要 cgroup IO 权重隔离且能接受开销？→ 可试 mq-deadline/bfq
设备是 SATA SSD / 机械盘（服务器）？
   └─ mq-deadline（防读饿死，稳妥）
桌面 / 交互式 / 单盘混合负载？
   └─ bfq（前台流畅，代价是 CPU）
```

切换调度器（临时）：

```bash
$ echo mq-deadline > /sys/block/sda/queue/scheduler
# 永久：udev 规则，按 rotational 区分
# /etc/udev/rules.d/60-ioscheduler.rules:
# ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
# ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
```

> 注意：cgroup v2 的 IO 权重控制（`io.weight`，呼应 L17）依赖支持权重的调度器（如 bfq）。NVMe + none 下 `io.weight` 不起作用，只能用 `io.max`（绝对限流）。这是 cgroup IO 隔离的一个常见坑。

---

## 第四章 I/O 合并与排序、预读、plug/unplug

### 4.1 IO 合并：相邻请求拼成一个

块层下发硬件前会尝试**合并**：
- **前向/后向合并**：新 bio 的扇区范围与队列里某个 request 首尾相接，就拼进去，一次传输更多数据，减少请求数。
- **排序**：机械盘上按 LBA 排序减少寻道（电梯算法）；NVMe 上意义不大（none 不排）。

```bash
# 看合并效果：rrqm/s wrqm/s 是每秒合并的读/写请求数
$ iostat -x 1
Device  rrqm/s  wrqm/s  r/s   w/s  ...
sda      120.0   450.0  200   50    # 大量合并，说明 IO 模式连续，good
```

可禁合并（排障/测试）：`echo 1 > /sys/block/sda/queue/nomerges`（2 为完全禁止）。

### 4.2 预读 readahead

顺序读时，内核**预读**后续数据进 page cache，让下次 read 命中缓存（呼应 L07）。预读窗口大小：

```bash
$ cat /sys/block/nvme0n1/queue/read_ahead_kb
128
$ blockdev --getra /dev/nvme0n1   # 以 512 字节扇区为单位
256
```

- **顺序读**（日志扫描、大文件拷贝、全表扫描）：调大 `read_ahead_kb`（如 512KB~1MB）能大幅提升吞吐。
- **随机读**（数据库点查、OLTP）：预读纯属浪费带宽和 page cache（读了用不上的数据），应**调小或关闭**。内核会根据访问模式自适应收缩预读窗口，但负载已知时手动调更稳。

### 4.3 plug / unplug：攒一批再下发

**plug（蓄流）**：进程提交 IO 时，块层先把请求暂存在一个**进程本地的 plug 列表**里，而不是立刻下发——给"合并"创造机会。等一批攒够、或进程要睡眠/显式 flush 时，**unplug** 一次性把合并好的请求下发给调度器/硬件。

```c
// 内核里典型用法
blk_start_plug(&plug);
submit_bio(bio1);   // 进 plug 列表，暂不下发
submit_bio(bio2);   // 可能与 bio1 合并
submit_bio(bio3);
blk_finish_plug(&plug);  // unplug：合并后批量下发
```

好处：减少下发次数、提升合并率。对 io_uring 批量提交（L09）尤其友好——一批 SQE 转成的 bio 容易在 plug 阶段合并成大请求。

---

## 第五章 诊断：iostat、iotop、blktrace、biolatency/biosnoop

### 5.1 iostat -x 各列精确含义

这是块 IO 诊断的第一工具，逐列吃透：

```bash
$ iostat -x 1
Device  r/s  w/s  rkB/s  wkB/s  rrqm/s wrqm/s  r_await w_await aqu-sz  rareq-sz  %util
nvme0n1 1200 300  48000  12000   10     50      0.08    0.12    0.18    40        99.5
```

| 列 | 含义 | 怎么看 |
|---|---|---|
| `r/s` `w/s` | 每秒完成的读/写次数（IOPS） | 与设备额定 IOPS 比，看是否接近上限 |
| `rkB/s` `wkB/s` | 每秒读/写带宽 | 与设备额定带宽比 |
| `rrqm/s` `wrqm/s` | 每秒合并的读/写请求 | 高=IO 连续、合并好 |
| `r_await` `w_await` | 读/写**平均耗时**（含排队 + 设备处理，ms） | **最关键的延迟指标**，飙高即慢 |
| `aqu-sz` | 平均队列深度（在途 IO 数） | **判断饱和的关键**，远小于设备并行能力=没满 |
| `rareq-sz` | 平均单次请求大小（KB） | 小=随机/碎，大=顺序 |
| `%util` | 至少一个 IO 在跑的时间占比 | **NVMe 上会骗人！见下** |

**关键关系**（利特尔法则）：`await ≈ aqu-sz / (r/s + w/s) × 1000`（量级关系）。`aqu-sz` 反映"积压多深"。

**`%util` 的正确解读**：
- 机械盘 / 单队列：`%util` 接近 100% ≈ 饱和，可信。
- **NVMe / 多队列**：`%util` 100% **只表示"任意时刻有 IO 在跑"**，因为能并发几百个 IO，单个 IO 串起来就能占满时间。**此时要看 `aqu-sz` 和实际 IOPS/带宽 vs 额定值**——`aqu-sz` 才 0.18，说明队列几乎空着，盘根本没忙。

### 5.2 iotop：哪个进程在搞 IO

```bash
$ iotop -oP        # -o 只显示有 IO 的，-P 按进程
  PID  PRIO  USER   DISK READ  DISK WRITE  COMMAND
  1234 be/4  pg      0.00 B/s   45.00 M/s   postgres: writer
```

定位"谁在制造 IO"，配合 cgroup 看是哪个容器/服务。

### 5.3 blktrace / blkparse：块层全链路追踪

`blktrace` 记录每个 IO 在块层各阶段的时间戳（Queue→Insert→Merge→Dispatch→Complete），能精确定位延迟卡在哪段：

```bash
$ blktrace -d /dev/nvme0n1 -o - | blkparse -i -
# 输出含各阶段事件：Q(queued) G(get request) I(inserted) D(dispatched) C(completed)
# 看 D→C 是设备处理时间，Q→D 是软件排队时间
```

`btt`（blktrace 后处理）能算出 Q2D（排队）、D2C（设备）、Q2C（总）各段平均延迟——区分"卡在软件排队"还是"卡在设备本身"。

### 5.4 biolatency / biosnoop（bcc/eBPF）：现代利器

bcc 工具基于 eBPF（见 L19），更轻量、更精确：

```bash
# biolatency：块 IO 延迟直方图，一眼看延迟分布与长尾
$ /usr/share/bcc/tools/biolatency 10 1
     usecs        : count     distribution
       128 -> 255 : 1203     |****************            |
       256 -> 511 : 2800     |****************************|  ← 主峰
      ...
   8192 -> 16383  : 12       |                            |  ← 长尾，少量慢 IO

# biosnoop：每个 IO 的进程、设备、扇区、大小、延迟，逐条
$ /usr/share/bcc/tools/biosnoop
TIME(s)  COMM        PID   DISK   T SECTOR    BYTES  LAT(ms)
0.000    postgres    1234  nvme0n1 W 1048576  8192   0.15
0.001    kworker     56    nvme0n1 W 2097152  4096   12.30   ← 这笔慢

# 按设备 + 进程的延迟，定位是哪个盘哪个进程慢
$ bpftrace -e 'tracepoint:block:block_rq_complete {
    @[args.dev] = hist(args.sector); }'   # 示意：可统计延迟/扇区分布
```

**诊断套路**：iostat 看到 `await` 高 → biolatency 看是整体慢还是长尾 → biosnoop 抓具体哪些 IO/哪个进程慢 → blktrace/btt 区分软件排队 vs 设备处理。

---

## 第六章 NVMe 与现代存储、cgroup io.max 限流

### 6.1 NVMe 为什么快

NVMe（基于 PCIe）相对 SATA/SAS 的本质优势：

- **多队列**：最多 64K 个队列、每队列 64K 深度——天生匹配 blk-mq 多硬件队列，每核一对队列几乎无锁。
- **低延迟协议**：精简命令集、直连 PCIe，省掉 SATA/AHCI 的协议开销。
- **高并发**：内部并行通道多，能同时处理成百上千 IO——这正是 `%util` 在 NVMe 上失真的根源。

所以 NVMe 上：调度器用 `none`、关注 `aqu-sz` 而非 `%util`、预读按负载调、用 io_uring（L09）批量异步喂满队列。

### 6.2 cgroup v2 io 控制：io.max 与 io.weight

容器/服务的 IO 隔离靠 cgroup v2 的 `io` 控制器（详见 L17）。两种机制：

**`io.max`（绝对限流）**——给某 cgroup 限定 IOPS / 带宽上限：

```bash
# 限制某 cgroup 对 nvme0n1（主次设备号 259:0）的写：最多 1000 IOPS、50MB/s
$ echo "259:0 wbps=52428800 wiops=1000" > /sys/fs/cgroup/mygroup/io.max
$ cat /sys/fs/cgroup/mygroup/io.max
259:0 rbps=max wbps=52428800 riops=max wiops=1000

# 设备号怎么查
$ lsblk -o NAME,MAJ:MIN
nvme0n1 259:0
```

`io.max` 是硬上限，超过就 throttle（节流），**任何调度器下都生效**，是 NVMe 上做 IO 隔离的主要手段。

**`io.weight`（相对权重）**——按权重分配带宽，但**依赖支持权重的调度器（bfq）**。NVMe + none 下 `io.weight` 不生效，这是常见坑：

```bash
$ echo "default 100" > /sys/fs/cgroup/mygroup/io.weight   # 权重，需 bfq 才有效
```

### 6.3 io.latency 与 PSI

cgroup v2 还有 `io.latency`（给某 cgroup 设 IO 延迟目标，超标则限制其他 cgroup 让路）和 **PSI**（Pressure Stall Information）的 IO 压力指标：

```bash
$ cat /sys/fs/cgroup/mygroup/io.pressure
some avg10=12.34 avg60=8.20 avg300=3.10 total=...
full avg10=5.00 ...
# avg10=12.34 表示过去 10 秒有 12.34% 的时间，任务因等 IO 而停滞
```

PSI 的 `io.pressure` 是判断"这个 cgroup 是否被 IO 拖累"的金标准——比 `%util` 可靠得多，直接反映"任务因等 IO 停了多久"。systemd-oomd、各类弹性伸缩都用 PSI 做信号（呼应 L05/L06/L17）。

整条链路闭环：page cache 脏页（L07）→ writeback 提交 bio → blk-mq 软/硬件队列 → 调度器 → NVMe 硬件队列 → 完成中断 → CQE，全程受 cgroup io 控制器与 PSI 观测。

---

## 生产实践

1. **NVMe 一律 none 调度器**：用 udev 规则按 `rotational` 自动设——NVMe/SSD 用 none，机械盘用 mq-deadline。别让 bfq 在 NVMe 服务器上吃 CPU。

2. **判断磁盘饱和看 aqu-sz + IOPS/带宽，不看 %util**：NVMe 上 `%util` 100% 是常态噪声。真饱和的信号是 `await` 升高 + `aqu-sz` 持续很深 + IOPS/带宽逼近设备额定值。

3. **预读按负载调**：顺序大吞吐（备份、扫描、流式）调大 `read_ahead_kb`；随机小 IO（OLTP、点查）调小，避免预读浪费带宽与 page cache。

4. **biolatency 纳入常态监控**：块 IO 延迟直方图（尤其 p99/p999 长尾）比平均 await 更能预警。长尾 IO 是数据库延迟毛刺的常见元凶。

5. **容器 IO 隔离用 io.max**：NVMe + none 下 `io.weight` 无效，用 `io.max` 给吵闹邻居（noisy neighbor）限 IOPS/带宽；用 `io.pressure`（PSI）监控哪个 cgroup 真被 IO 拖累（呼应 L17）。

6. **io_uring + O_DIRECT 喂满 NVMe**：要榨干 NVMe 的高并发，用 io_uring 批量异步提交（L09）保持足够 `aqu-sz`，绕过 page cache 用 O_DIRECT 自管缓冲（L07），别用单线程同步 IO（队列深度上不去，盘闲着）。

---

## 陷阱清单

1. **%util 100% 误判磁盘饱和**
   - 现象：iostat 显示 `%util` 99%+，判定 IO 瓶颈，加盘后无改善。
   - 原因：NVMe 多队列下，只要任意时刻有一个 IO 在跑 `%util` 就 100%，与真实利用率无关。
   - 修法：看 `aqu-sz`（队列深度）、`await`（延迟）、实际 IOPS/带宽 vs 设备额定值；`aqu-sz` 很小说明盘根本没忙。

2. **await 高，分不清卡在哪**
   - 现象：`await` 飙到几十毫秒，但不知是设备慢还是软件排队。
   - 原因：`await` = 软件排队时间 + 设备处理时间之和，单看分不清。
   - 修法：blktrace + btt 拆 Q2D（排队）与 D2C（设备）；或 biosnoop 看单 IO 延迟；D2C 高=设备/盘问题，Q2D 高=队列拥塞/调度器/限流。

3. **NVMe 上用了 bfq，CPU 被调度器吃光**
   - 现象：NVMe 服务器 IOPS 上不去，CPU 高，软中断/内核态占用大。
   - 原因：bfq 算法复杂、每 IO CPU 开销大，拖累高 IOPS 设备。
   - 修法：NVMe 改 `none`；bfq 只留给桌面/交互式机械盘场景。

4. **cgroup io.weight 不生效**
   - 现象：设了 `io.weight` 做 IO 隔离，但毫无效果。
   - 原因：`io.weight` 依赖支持权重的调度器（bfq），NVMe + none 下不生效。
   - 修法：NVMe 用 `io.max`（绝对限流）做隔离；非要按权重则切 bfq（评估 CPU 代价）。

5. **随机负载开大预读，带宽被浪费**
   - 现象：OLTP 数据库吞吐不及预期，`rareq-sz` 大但命中率低。
   - 原因：`read_ahead_kb` 过大，随机点查预读了大量用不上的数据。
   - 修法：随机负载调小或关预读；顺序负载才调大。

6. **队列深度上不去，盘闲着却慢**
   - 现象：单进程同步 IO，`aqu-sz` 始终接近 1，吞吐远低于 NVMe 额定。
   - 原因：同步阻塞 IO 一次只有一个在途请求，喂不满 NVMe 的并行能力。
   - 修法：用 io_uring（L09）/ 多线程 / 异步 IO 提高并发，把 `aqu-sz` 喂到设备并行度量级。

7. **nr_requests / 队列太浅限制吞吐**
   - 现象：高负载下 `aqu-sz` 顶到队列上限就不再涨，吞吐被卡。
   - 原因：`nr_requests`（块层队列深度）或设备 tag 数太小。
   - 修法：适当调大 `/sys/block/*/queue/nr_requests`（在设备和内存允许范围内）；确认 NVMe tag 数。

8. **fsync 频繁导致 FLUSH 风暴**
   - 现象：写吞吐低、await 高，biosnoop 见大量 FLUSH/FUA 小 IO（呼应 L07 fsync 放大）。
   - 原因：应用每写一点就 fsync，每次触发设备 FLUSH，打断流水线。
   - 修法：批量提交后再 fsync；用 `fdatasync`；数据库组提交（group commit）；确认未误用机械盘。

---

## 2026 现状

- **blk-mq 是唯一块层路径**：单队列 legacy 早已从内核删除，所有块设备走多队列；软/硬件队列 + tag 机制是块层基石。
- **NVMe 普及，none 是常见默认**：发行版对 NVMe 默认 `none`；判断饱和的心智从"%util"彻底转向"aqu-sz + IOPS/带宽 + PSI io.pressure"。
- **eBPF 工具成主流诊断手段**：biolatency / biosnoop / bpftrace 取代部分 blktrace 场景，更轻量、可在线、可按进程/cgroup 维度切分（呼应 L19）。
- **cgroup v2 io 控制成熟**：`io.max`（任意调度器生效）做容器 IO 隔离主力；`io.latency`、`io.pressure`（PSI）支撑延迟目标与弹性；`io.weight` 受限于 bfq（呼应 L17）。
- **io_uring + O_DIRECT** 成为高性能存储引擎喂满 NVMe 的标准组合（呼应 L07/L09）：批量异步保持深队列，绕开 page cache 自管缓冲。
- **新介质与分层**：ZNS（分区命名空间 SSD）、NVMe-oF（网络块设备）、持久内存/CXL 内存语义存储等在特定场景落地；folio 化的 IO 路径（L07）改善大 IO 与大页协同。

---

## 练习题

1. 为什么 `%util` 在机械盘上可信、在 NVMe 上会"骗人"？要判断一块 NVMe 是否真饱和，你会看哪几个 iostat 列，以及和什么基准对比？

2. 解释 blk-mq 的软件队列与硬件队列两级设计如何消除老单队列的全局锁瓶颈。为什么单队列 legacy 路径会被内核删除？

3. none / mq-deadline / kyber / bfq 四个调度器各自的核心思想与适用设备是什么？为什么 NVMe 服务器不该用 bfq、桌面却推荐 bfq？

4. mq-deadline 为什么要让读优先于写？这与"读多是同步、写多是异步回写"有什么关系（呼应 L07 writeback）？

5. plug / unplug 机制做什么？它如何配合 IO 合并提升吞吐？为什么它对 io_uring 批量提交（L09）特别友好？

6. cgroup v2 的 `io.max`、`io.weight`、`io.latency`、`io.pressure` 各是什么？为什么在 NVMe + none 环境下做容器 IO 隔离要用 `io.max` 而不是 `io.weight`？

7. **实战**：写一组命令，用 udev 规则让系统对所有 NVMe 设备自动设 `none` 调度器、对机械盘自动设 `mq-deadline`，并给某容器 cgroup 限制对 `nvme0n1` 的写带宽为 100MB/s。

8. **排障**：某 PostgreSQL 实例查询偶发延迟毛刺，`iostat` 显示 `nvme0n1` 的 `%util` 长期 100% 但 `aqu-sz` 只有 0.3、`r/s+w/s` 远低于盘的额定 IOPS。请判断这是不是磁盘瓶颈，并给出用 biolatency / biosnoop / blktrace 进一步定位毛刺根因的完整排查流程。

---

## 参考答案

1. 机械盘是单队列、一次只能处理一个 IO，盘忙到没空闲时 `%util` 才接近 100%，与饱和强相关，故可信。NVMe 有几十上百个硬件队列、能同时处理成百上千并发 IO，只要任意时刻有一个 IO 在跑 `%util` 就是 100%，哪怕真实利用率才 5%，所以它会"骗人"。判断 NVMe 是否真饱和要看：`aqu-sz`（平均队列深度，远小于设备并行能力说明没满）、`r_await`/`w_await`（IO 平均耗时，飙高才是慢）、以及实际 `r/s+w/s` 和 `rkB/s+wkB/s` 与设备额定 IOPS/带宽的比值——逼近额定值才是真饱和。

2. 老单队列把所有 CPU 提交的 IO 都塞进一个全局 `request_queue`，由一把队列锁保护；NVMe 几十万 IOPS 下多核同时提交，这把全局锁成了热点，锁竞争烧光 CPU，IOPS 上不去。blk-mq 用两级队列消除：每个 CPU 一个**软件队列（per-CPU ctx）**，IO 先进本地软件队列，避免跨核抢锁；软件队列再映射到**硬件队列（hctx）**，NVMe 每核一对 SQ/CQ，软硬件队列接近 1:1，几乎全程无锁，并充分利用 NVMe 的多队列并行能力。单队列 legacy 被删是因为它无法发挥 SSD/NVMe 的硬件并行，是性能瓶颈，blk-mq 已能覆盖所有设备（包括机械盘只用 1 个硬件队列），保留双路径徒增维护负担，故 5.0 移除。

3. none：不调度、FIFO 直通硬件，开销最低，适合 NVMe/高速 SSD（设备自己够快够并行、内部有 FTL 调度、无寻道，软件再排序纯属添乱）。mq-deadline：给每个 IO 设截止期、读优先防饿死、主体按 LBA 排序合并，开销低，适合 SATA SSD/机械盘/通用。kyber：按目标延迟自适应限流、轻量自调，适合高速多队列设备。bfq：按进程/cgroup 公平分配带宽+权重并优化交互延迟，CPU 开销高，适合桌面/交互式/机械盘。NVMe 服务器不该用 bfq：bfq 算法复杂、每 IO 的 CPU 开销显著，在高 IOPS 下 CPU 被调度器吃掉反成瓶颈。桌面推荐 bfq：它让后台大文件复制时前台仍流畅、交互应用立刻响应，体验极佳，桌面 IOPS 不高故 CPU 代价可接受。

4. 读多是同步阻塞的——进程要等到数据才能继续（处于 D 状态），读延迟直接转化为用户/应用可感知的卡顿。写多是异步回写（writeback，呼应 L07），脏页由内核后台刷盘，提交写的进程通常不等待。所以 mq-deadline 给读设更短的截止期（如 500ms）、写更长（如 5s），让读优先插队，避免大量异步写把同步读无限推迟（写饿读），从而显著改善交互/查询延迟。

5. plug（蓄流）：进程提交 IO 时块层先把请求暂存在进程本地的 plug 列表里而不立即下发，给"合并"创造时间窗口；等攒够一批、或进程要睡眠/显式 flush 时 unplug，一次性把合并好的请求批量下发给调度器/硬件。它配合 IO 合并：相邻 bio（扇区首尾相接）在 plug 阶段被前向/后向合并成更大的 request，减少下发硬件的请求数、一次传输更多数据。对 io_uring 批量提交特别友好：一批 SQE 转成的多个 bio 在同一个 plug 窗口内提交，容易合并成大请求，提升合并率、降低下发次数。

6. `io.max`：绝对限流，给某 cgroup 限定对某设备的 IOPS/带宽硬上限（如 `wbps`/`wiops`），超过即节流，**任何调度器下都生效**。`io.weight`：相对权重，按权重比例分配带宽，但**依赖支持权重的调度器（bfq）**。`io.latency`：给某 cgroup 设 IO 延迟目标，超标则限制其他 cgroup 让路。`io.pressure`（PSI）：反映该 cgroup 因等 IO 而停滞的时间占比，是判断"是否被 IO 拖累"的金标准。NVMe + none 环境下 `io.weight` 不生效（none 不支持权重调度），所以 IO 隔离要用与调度器无关的 `io.max` 做绝对限流。

7. udev 规则（按 rotational 自动设调度器）：
   ```bash
   # /etc/udev/rules.d/60-ioscheduler.rules
   ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
   ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
   ```
   重载并触发：`udevadm control --reload && udevadm trigger`。临时验证：`echo none > /sys/block/nvme0n1/queue/scheduler`。给容器 cgroup 限写带宽 100MB/s（先查设备号）：
   ```bash
   lsblk -o NAME,MAJ:MIN          # 假设 nvme0n1 是 259:0
   echo "259:0 wbps=104857600" > /sys/fs/cgroup/<容器cgroup>/io.max
   ```

8. 这**不是磁盘瓶颈**：`%util` 100% 在 NVMe 上是常态噪声，而 `aqu-sz` 只有 0.3（队列几乎空着）、IOPS 远低于额定，三者一致说明盘其实很闲，毛刺另有原因（很可能是少量长尾 IO，如 fsync/FLUSH、后台 checkpoint、邻居 cgroup 抢占等）。排查流程：① `biolatency 10 1` 看块 IO 延迟直方图——若主峰正常但有少量落在高延迟桶，确认是**长尾**毛刺而非整体慢；② `biosnoop` 逐条抓 IO，看哪些 IO 延迟高（LAT 列）、属于哪个进程（COMM/PID，区分是 postgres 自身还是 kworker 回写）、是读还是写、扇区/大小（识别是否 FLUSH/FUA 小 IO）；③ `blktrace -d /dev/nvme0n1 | blkparse` 配合 `btt` 拆分各阶段延迟——`D2C`（dispatched→completed）高说明卡在设备本身，`Q2D`（queued→dispatched）高说明卡在软件排队/调度/限流；④ 结合 `cat /sys/fs/cgroup/.../io.pressure`（PSI）确认该实例是否真因等 IO 停滞，并检查是否有 noisy neighbor 或 `io.max` 限流。综合定位毛刺根因（常见为 fsync/FLUSH 风暴或后台 checkpoint 的长尾写，呼应 L07）。
