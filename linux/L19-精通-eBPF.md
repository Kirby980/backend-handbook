# 精通 eBPF：虚拟机与 verifier、map、程序类型、CO-RE/BTF、bpftrace/bcc、生产可观测

> 课程编号：L19
> 路线图来源：Linux · 模块七 可观测与生产
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：CO-RE+BTF、bpftrace 2.x、eBPF LSM、Cilium/Pixie/Parca）

---

## 引言：不改代码、不重启，看见内核里的一切

传统排障的尴尬：想知道「哪个进程在反复 open 某文件」「块设备 I/O 延迟分布」「谁在建 TCP 连接」，要么改代码加日志（要发版），要么 `strace`（开销大、只能单进程），要么抓全量包（数据爆炸）。**eBPF** 改变了这一切：把一段经过安全校验的小程序**动态加载进内核**，挂在系统调用、内核函数、网络收发点、用户函数上，事件触发时就地采集、聚合，几乎零侵入、低开销。

eBPF 是 2020 年代 Linux 可观测与网络的基石技术：Cilium 用它做 K8s 网络与 LB，Pixie/Parca 用它做无侵入观测与持续 profiling。本篇讲清它的运行机制（尤其**为什么你的程序会被 verifier 拒绝**）、程序类型挂载点、CO-RE 如何实现「一次编译到处运行」，以及用 `bpftrace` 写出立竿见影的单行命令。probe 挂在哪由系统调用/内核函数决定，见 [L01 架构与系统调用](./L01-精通-Linux-架构与系统调用.md)；火焰图/perf 见 [L18 性能诊断](./L18-精通-性能诊断方法论与工具.md)；XDP 见 [L11 网络栈](./L11-精通-Linux-网络协议栈.md)。

---

## 第一章 从 cBPF 到 eBPF

最早的 BPF（classic BPF，cBPF）是 `tcpdump` 背后的包过滤虚拟机——一个极简指令集，在内核里决定「这个包要不要给用户态」。2014 年起，eBPF（extended BPF）把它扩展成一个**通用的内核内虚拟机**：

- 11 个 64 位寄存器、栈、更丰富的指令；
- 可挂载到几乎任何内核事件（不只网络）；
- 通过 **map** 与用户态交换数据；
- 通过 **helper 函数**调用内核能力。

它是**事件驱动**的：程序不会自己运行，而是在挂载点的事件发生时被内核调用。

---

## 第二章 运行机制：加载、verifier、JIT、map、helper

### 2.1 完整生命周期

```
C 源码 ──clang/llvm──► eBPF 字节码(.o)
   │ libbpf 加载
   ▼
内核 verifier 校验 ──(拒绝)──► 加载失败，报错
   │ (通过)
   ▼
JIT 编译为本机指令 ──► 挂到事件(kprobe/tracepoint/XDP...)
   │ 事件触发时执行，结果写入 map
   ▼
用户态读 map ──► 展示/聚合
```

### 2.2 verifier：为什么你的程序被拒

verifier 是 eBPF 安全的核心：它在加载时**静态分析所有可能的执行路径**，确保程序不会崩溃内核、不会死循环、不越界。被拒的常见原因：

| 报错类别 | 原因 | 解法 |
|---|---|---|
| `back-edge`/循环相关 | 早期禁止循环；现支持**有界循环**（5.3+），但边界要可证明 | 用 `#pragma unroll` 或可被验证上界的循环、`bpf_loop()` helper |
| `invalid mem access` | 访问指针前没做 NULL 检查 / 越界 | 访问 map value、`args` 前先判空、加边界检查 |
| `R1 !read_ok` | 读未初始化的栈/寄存器 | 先初始化变量 |
| 栈超限 | eBPF 栈仅 **512 字节** | 大数据放 map，不放栈 |
| 指令数超限 | 复杂度上限（百万级指令） | 拆分、简化、用 tail call |

> 「verifier 拒绝」是 eBPF 开发最常见的痛。记住核心：**任何指针解引用前必须让 verifier 相信它非空且在界内**。

**verifier 怎么工作**：它从入口指令开始做**符号执行**，跟踪每个寄存器的类型与取值范围（如「R3 是指向 map value 的指针，偏移 0~64」），对每条分支都验证内存访问合法、helper 参数类型匹配；并做路径剪枝（状态等价的路径合并）以控制状态爆炸。这也解释了为何**越复杂的程序越易被拒**——路径状态太多会触达复杂度上限。调试技巧：加载失败时内核会输出 verifier log（`bpftool prog load` 或 libbpf 打印），逐行看它在哪条指令、对哪个寄存器不满意。

### 2.3 map：内核与用户态的桥

map 是 eBPF 的「内存」，也是与用户态通信的唯一正道：

| 类型 | 用途 |
|---|---|
| `HASH` / `LRU_HASH` | 键值存储（如 per-pid 统计），LRU 自动淘汰 |
| `ARRAY` / `PERCPU_ARRAY` | 索引数组；per-cpu 版避免竞争、用于高频计数 |
| `PERF_EVENT_ARRAY` | 向用户态推送事件（老式） |
| `RINGBUF`（5.8+） | 高效事件流，**取代 perf buffer**，多生产者单消费者 |
| `STACK_TRACE` | 保存调用栈（火焰图基础） |

### 2.4 helper 函数

eBPF 程序不能随意调内核函数，只能调白名单 **helper**：`bpf_map_lookup_elem`、`bpf_probe_read_kernel`、`bpf_ktime_get_ns`、`bpf_get_current_pid_tgid`、`bpf_perf_event_output`、`bpf_ringbuf_submit` 等。这是受控、安全的内核能力出口。

### 2.5 程序组合：tail call 与 BPF-to-BPF

单个 eBPF 程序受指令数与 512 字节栈限制，复杂逻辑靠两种机制拆分：

- **BPF-to-BPF 调用**：一个程序内可调用另一个 eBPF 函数（像普通函数调用），verifier 分别校验各函数。
- **tail call**：经 `BPF_MAP_TYPE_PROG_ARRAY` 跳到另一个程序且**不返回**（类似 `goto`），用于状态机式处理——XDP 多阶段包处理（[L11](./L11-精通-Linux-网络协议栈.md)）常用。

**全局变量**：现代 libbpf 支持在 `.data`/`.rodata` 段放全局变量；`.rodata` 里的编译期常量可被 verifier 用于死代码消除（按内核特性裁剪分支），也便于用户态在加载前配置参数，比每次查 map 更快。

---

## 第三章 程序类型与挂载点

eBPF 的威力在于「能挂的地方多」：

| 类型 | 挂载点 | 典型用途 |
|---|---|---|
| **kprobe / kretprobe** | 任意内核函数入口/返回 | 跟踪内核函数调用/耗时（如 `vfs_read`） |
| **uprobe / uretprobe** | 用户态函数 | 跟踪应用函数（如 libc `malloc`、Go 函数） |
| **tracepoint** | 内核静态埋点 | 稳定接口，优于 kprobe（如 `syscalls:sys_enter_read`） |
| **raw_tracepoint** | 原始 tracepoint | 更低开销，拿原始参数 |
| **XDP** | 网卡驱动收包最早点 | DDoS 过滤、L4 LB（见 [L11](./L11-精通-Linux-网络协议栈.md)） |
| **tc (clsact)** | 流量控制层 | 容器网络策略、整形（Cilium） |
| **SOCK_OPS / sk_skb** | socket 层 | TCP 调优、socket 重定向 |
| **LSM** | 安全钩子 | 运行时安全策略（eBPF LSM） |
| **perf_event** | 采样中断 | CPU profiling（见 [L18](./L18-精通-性能诊断方法论与工具.md)） |

**tracepoint 优于 kprobe**：tracepoint 是内核维护的稳定埋点，跨版本参数稳定；kprobe 挂在具体函数上，函数改名/内联就失效。能用 tracepoint 就别用 kprobe。

### 3.1 一个完整的 XDP 程序（丢弃指定端口）

```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

SEC("xdp")
int xdp_drop_port(struct xdp_md *ctx) {
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;   // 边界检查（verifier 强制）
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;
    if (ip->protocol != IPPROTO_TCP) return XDP_PASS;

    struct tcphdr *tcp = (void *)ip + ip->ihl * 4;
    if ((void *)(tcp + 1) > data_end) return XDP_PASS;
    if (tcp->dest == bpf_htons(9999)) return XDP_DROP;   // 丢弃目标端口 9999

    return XDP_PASS;
}
char LICENSE[] SEC("license") = "GPL";
```

**每次解引用前都做 `> data_end` 边界检查**是 XDP 程序通过 verifier 的铁律（见第二章）。挂载：`ip link set dev eth0 xdp obj xdp_drop.o sec xdp`。它在收包最早点执行，DDoS 过滤可达千万级 pps（呼应 [L11](./L11-精通-Linux-网络协议栈.md)）。

### 3.2 map 的读写（内核侧）

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);
    __type(value, __u64);
    __uint(max_entries, 10240);
} pkt_count SEC(".maps");

__u32 key = ip->saddr;
__u64 *cnt = bpf_map_lookup_elem(&pkt_count, &key);
if (cnt) __sync_fetch_and_add(cnt, 1);                  // 已存在：原子自增
else { __u64 init = 1; bpf_map_update_elem(&pkt_count, &key, &init, BPF_ANY); }
```

用户态再用 `bpf_map_lookup_elem`/`bpf_map_get_next_key` 遍历读出——这就是「按源 IP 统计流量」类工具的核心骨架。高频路径用 `PERCPU_HASH` 免锁、读取时再汇总。

---

## 第四章 CO-RE 与 BTF：一次编译，到处运行

早期 eBPF 工具（老 bcc）要在**目标机现场带着内核头文件用 clang 编译**，笨重且依赖内核头。**CO-RE（Compile Once - Run Everywhere）** 解决了这个问题，三件套：

- **BTF（BPF Type Format）**：内核把自己的类型信息（结构体布局、字段偏移）编译进 `vmlinux`（`/sys/kernel/btf/vmlinux`）；
- **libbpf**：加载时根据目标内核的 BTF 做**字段重定位（relocation）**，自动修正结构体字段偏移；
- **`vmlinux.h`**：从 BTF 生成的全内核类型头，开发时引用它即可。

效果：在一台机器上 `clang` 编译一次 `.o`，能在不同内核版本上加载运行——因为字段偏移在加载时按目标 BTF 修正。这是现代 eBPF 工程（libbpf-bootstrap、Cilium、bcc 新工具）的标准范式。

```c
// CO-RE 读取结构体字段（libbpf 会在加载时重定位偏移）
struct task_struct *task = (void *)bpf_get_current_task();
pid_t ppid = BPF_CORE_READ(task, real_parent, tgid);
```

**最小 libbpf 程序的两半**：现代 eBPF 工程分「内核侧 `.bpf.c` + 用户侧 loader」：

```c
// hello.bpf.c —— 内核侧，编译成 BTF-enabled 的 .o
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
SEC("tracepoint/syscalls/sys_enter_execve")   // SEC() 声明挂载点
int handle_exec(void *ctx) {
    char comm[16];
    bpf_get_current_comm(&comm, sizeof(comm));
    bpf_printk("exec by %s", comm);
    return 0;
}
char LICENSE[] SEC("license") = "GPL";         // 多数 helper 要求 GPL
```

用户侧用 `bpftool gen skeleton hello.bpf.o > hello.skel.h` 生成骨架，几行 C 或 Go（`cilium/ebpf` 库）即可 `open → load → attach → 读 ringbuf`。`SEC()` 宏告诉 libbpf 把程序挂到哪、`LICENSE` 段是调用 GPL-only helper 的前提。

---

## 第五章 bpftrace：单行命令的威力

`bpftrace` 是 eBPF 的「awk」——高级语言封装，一行就能干活。语法：`probe /filter/ { action }`。

### 5.1 内置变量与聚合

- 变量：`pid`、`tid`、`comm`（进程名）、`nsecs`、`args`（探针参数）、`retval`、`arg0..argN`；
- map：`@name[key] = value`；
- 聚合函数：`count()`、`sum()`、`avg()`、`hist()`（2 的幂直方图）、`lhist()`（线性直方图）。

### 5.2 立竿见影的单行

```bash
# 1) 谁在频繁 read？按进程名统计 read 次数
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @[comm] = count(); }'

# 2) 实时打印所有新进程执行（execsnoop 雏形）
bpftrace -e 'tracepoint:syscalls:sys_enter_execve {
    printf("%-16s %s\n", comm, str(args->filename)); }'

# 3) 块 I/O 大小分布直方图
bpftrace -e 'tracepoint:block:block_rq_issue { @bytes = hist(args->bytes); }'

# 4) 文件 open 延迟分布（kprobe + kretprobe 配对，呼应 L07/L18）
bpftrace -e '
  kprobe:do_sys_openat2 { @start[tid] = nsecs; }
  kretprobe:do_sys_openat2 /@start[tid]/ {
      @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# 5) 统计每个进程的 TCP 重传（定位弱网，呼应 L12/L13）
bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm] = count(); }'
```

**多 probe 脚本**（存为 `.bt` 文件，`bpftrace syscount.bt` 运行）——每 10 秒打印各进程系统调用次数：

```
// syscount.bt
BEGIN { printf("counting syscalls... Ctrl-C to end\n"); }
tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }
interval:s:10 { print(@); clear(@); }   // 周期快照后清零
END { clear(@); }
```

`BEGIN`/`END` 在脚本起止运行，`interval:s:N` 周期触发——配合 `print`/`clear` 把 bpftrace 从「单行」升级为「监控脚本」。

### 5.3 bcc 工具箱（开箱即用）

bcc 提供大量成品工具，无需自己写：

| 工具 | 作用 | 呼应章节 |
|---|---|---|
| `execsnoop` | 实时新进程执行 | L02 |
| `opensnoop` | 实时文件打开 | L07 |
| `biolatency` / `biosnoop` | 块 I/O 延迟分布/明细 | L10 |
| `tcplife` / `tcpconnect` / `tcpretrans` | 连接寿命/建连/重传 | L12/L13 |
| `runqlat` | 调度延迟（在运行队列等多久） | L03/L18 |
| `profile` | 采样调用栈出火焰图 | L18 |
| `cachestat` | page cache 命中率 | L05/L07 |

---

## 第六章 生产可观测与安全

### 6.1 生态（2026）

| 项目 | 用途 |
|---|---|
| **Cilium** | K8s 网络 / NetworkPolicy / L4-L7 LB（eBPF 替代 kube-proxy），Hubble 做网络可观测 |
| **Pixie** | K8s 无侵入应用观测（自动追踪 HTTP/SQL 等协议） |
| **Parca / Pyroscope** | 持续 profiling（always-on，eBPF 采样全集群 CPU） |
| **Coroot** | 基于 eBPF 的自动服务拓扑与 SLO |

这些与 [cloud-native C03（Cilium 网络）](../cloud-native/C03-精通-K8s-网络与-Service.md)、[C11（可观测）](../cloud-native/C11-精通-K8s-可观测性.md) 紧密相关。

### 6.2 开销与安全边界

- **开销**：tracepoint/kprobe 在高频路径（如每个包、每次调度）上仍有成本，采集前要评估事件频率，善用 per-cpu map 与采样。
- **安全**：verifier 保证程序不崩内核，但加载 eBPF 需要 `CAP_BPF`/`CAP_SYS_ADMIN`，是强能力；`eBPF LSM` 反过来可用于加固。容器环境通常默认不放开 eBPF 加载权限。

---

## 生产实践

**案例：偶发磁盘延迟，应用层只看到「慢」。**
用 `biolatency` 直接出块设备 I/O 延迟直方图，确认是否设备侧慢；再 `biosnoop` 看是哪个进程、哪个扇区的 I/O 慢——全程不改应用、不重启（呼应 [L10](./L10-精通-块设备与IO调度.md)）。

**案例：想知道某服务到底在 open 哪些文件、建哪些连接。**
`opensnoop -p <pid>` + `tcpconnect -p <pid>`，几秒钟看清行为，比读代码快。

**案例：全集群 CPU 热点。**
Parca/Pyroscope 持续采样，eBPF `profile` 在每台机上低开销采栈，聚合成集群级火焰图，定位跨服务的 CPU 大头。

---

## 陷阱清单

1. **现象**：eBPF 程序加载失败 `back-edge`/循环报错 → **原因**：verifier 无法证明循环有界 → **修法**：`#pragma unroll`、`bpf_loop()` 或可验证上界。
2. **现象**：`invalid memory access` → **原因**：解引用指针前没判空/越界 → **修法**：先 NULL 检查、加边界判断，用 `bpf_probe_read_kernel`。
3. **现象**：老 bcc 工具在生产机要装内核头、编译慢 → **原因**：非 CO-RE → **修法**：用 libbpf+CO-RE/BTF 的新工具。
4. **现象**：kprobe 挂的函数升级内核后失效 → **原因**：函数改名/内联 → **修法**：优先用 tracepoint（稳定接口）。
5. **现象**：高频探针拖慢系统 → **原因**：挂在每包/每次调度等热点，事件量巨大 → **修法**：采样、用 raw_tracepoint、per-cpu 聚合、缩小过滤范围。
6. **现象**：bpftrace 直方图为空 → **原因**：filter 没命中或 kprobe 名错 → **修法**：`bpftrace -l 'kprobe:*openat*'` 先列可用探针名。
7. **现象**：容器内加载 eBPF 失败 → **原因**：缺 `CAP_BPF`/`CAP_SYS_ADMIN`（默认不放开）→ **修法**：在宿主或特权调试容器中运行采集器。
8. **现象**：`bpf_printk` 的输出找不到 → **原因**：它写到 `/sys/kernel/debug/tracing/trace_pipe` → **修法**：`cat` 该文件查看；生产用 ringbuf 而非 printk。
9. **现象**：加载报 GPL 相关错误 → **原因**：用了 GPL-only helper 却没声明 license → **修法**：加 `char LICENSE[] SEC("license") = "GPL";`。

---

## 2026 现状

- **CO-RE + BTF 已是标准范式**，「一次编译到处运行」让 eBPF 工具真正可移植。
- `bpftrace`（2.x）功能持续增强，是即时排障首选；bcc 工具多数已 CO-RE 化。
- **RINGBUF** 取代 perf buffer 成为事件流首选。
- eBPF 进入网络（Cilium）、安全（eBPF LSM、Tetragon）、持续 profiling（Parca/Pyroscope）、可观测（Pixie/Coroot/Hubble）全方位生产落地。
- `sched_ext`（见 [L03](./L03-精通-CPU-调度-CFS-到-EEVDF.md)）让 eBPF 甚至能写 CPU 调度器——eBPF 的边界还在扩张。

---

## 练习题

1. （⭐）cBPF 与 eBPF 的关系是什么？eBPF 为什么是「事件驱动」的？
2. （⭐）map 在 eBPF 里扮演什么角色？列举三种常见 map 类型及用途。
3. （⭐⭐）verifier 的职责是什么？列出至少三种会被它拒绝的程序写法及修法。
4. （⭐⭐）为什么说「能用 tracepoint 就别用 kprobe」？各自的稳定性与适用场景。
5. （⭐⭐）CO-RE 靠哪三样东西实现「一次编译到处运行」？BTF 在其中起什么作用？
6. （⭐⭐⭐）写一个 bpftrace 单行，统计某内核函数（如 `vfs_read`）的调用延迟分布直方图，并解释 kprobe/kretprobe 如何配对计时。
7. （⭐⭐⭐）线上偶发磁盘慢，仅用 eBPF/bcc 工具，给出从「确认是否设备侧慢」到「定位具体进程/IO」的排查链路。
8. （⭐⭐⭐）在高频路径（每个网络包）上挂 eBPF 有什么风险？列出至少三种降低开销的手段。
9. （⭐⭐）tail call 与 BPF-to-BPF 调用各解决什么问题？为什么单个 eBPF 程序需要这两种拆分机制？
10. （⭐⭐⭐）描述现代 libbpf + CO-RE 程序的「内核侧 .bpf.c + 用户侧 loader」结构，`SEC()` 宏与 GPL license 段各起什么作用？
