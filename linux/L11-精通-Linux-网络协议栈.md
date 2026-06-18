# 精通 Linux 网络协议栈：收发包路径、sk_buff、NAPI、软中断、RSS/RPS/RFS、GRO、XDP

> 课程编号：L11
> 路线图来源：Linux · 模块四 网络
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：NAPI 收包、sk_buff、softirq/ksoftirqd、RSS/RPS/RFS/XPS、GRO/GSO/TSO offload、qdisc fq/fq_codel、XDP/AF_XDP）

---

## 引言：一个包从网线到 recv() 的旅程

你的 Go 服务调用 `n, _ := conn.Read(buf)` 拿到了一段 HTTP 请求。这一行代码背后，一个以太网帧刚刚走完了一段惊心动魄的旅程：它从对端网卡发出，穿过交换机，到达本机网卡的 RX FIFO；网卡用 **DMA** 把它直接写进内存里一段预先分配好的环形缓冲区；网卡触发一次**硬中断**告诉 CPU"有货到了"；CPU 的硬中断处理函数却几乎什么都不做，只是关掉中断、调度一个**软中断**（`NET_RX_SOFTIRQ`）就返回了；软中断里 **NAPI poll** 函数把环形缓冲区里的帧一个个取出来、包成 `sk_buff`、送进协议栈；IP 层查路由、TCP 层比对四元组找到你的 socket、把数据挂到 socket 的接收队列；最后唤醒阻塞在 `Read` 上的那个 goroutine 背后的内核线程，数据被拷贝到你的 `buf`。

这一整条路径，在一台跑着 25GbE/100GbE 网卡、每秒收上千万个包的机器上，每秒要重复上千万次。它的每一个环节——中断怎么不被打满、软中断怎么不堆在单核上、`sk_buff` 怎么避免反复分配、大包怎么被合并成一个少走几遍协议栈——都直接决定了你的服务的吞吐和尾延迟。**网络栈是 Linux 内核里 CPU 开销最容易"看不见地"爆掉的子系统**：一个 `top` 里 `%si`（softirq）打到 100% 的单核，可能就是你 P99 延迟突然翻倍的全部原因。

本篇沿着"收包→sk_buff→中断与 NAPI→多核扩展→offload→发包→XDP"这条主线，把这条旅程拆到内核数据结构和可调参数的粒度。读完你应该能：画出一个包从网卡 DMA 到 `recv()` 的完整链路；解释为什么硬中断里"什么都不做"反而是对的；用 RPS/RFS 把打满单核的软中断摊到多核；看懂 `ethtool -S` / `/proc/net/softnet_stat` 里的丢包计数；并理解 XDP 凭什么能在协议栈之前就把 DDoS 流量丢掉。TCP 本身的状态机与拥塞控制留给 [L12 TCP/IP 内核实现与调优](./L12-精通-TCP-IP-内核实现与调优.md)，socket 与连接管理见 [L13 Socket 与连接管理](./L13-精通-Socket-与连接管理.md)，XDP/eBPF 的编程细节见 [L19 eBPF 深度实战](./L19-精通-eBPF.md)。

---

## 第一章 收包路径：从网卡 DMA 到 socket 接收队列

### 1.1 全景图：一个包的七道关卡

先建立整条链路的鸟瞰图，后面每一章都是在放大其中一段：

```
   ┌─────────┐  ① 帧到达
   │  网卡 NIC │  PHY/MAC 收帧，做硬件校验、RSS 哈希选队列
   └────┬────┘
        │ ② DMA 写入（网卡主动写内存，不经 CPU）
        ▼
   ┌──────────────────────────────┐
   │  RX ring（环形描述符队列，每队列一个） │  desc[i] 指向预分配的 buffer
   └────┬─────────────────────────┘
        │ ③ 硬中断 IRQ（"有货到了"）
        ▼
   ┌──────────────┐  ④ 硬中断处理：禁用本队列中断 + napi_schedule()
   │ hardirq (top) │     →几乎不干活，立即返回
   └────┬─────────┘
        │ raise NET_RX_SOFTIRQ
        ▼
   ┌──────────────────────────┐  ⑤ 软中断 NET_RX_SOFTIRQ
   │ softirq / ksoftirqd       │     net_rx_action() → 驱动 poll()
   │   napi->poll(budget)      │     批量收割 RX ring，造 sk_buff
   └────┬─────────────────────┘
        │ ⑥ napi_gro_receive → __netif_receive_skb
        ▼
   ┌──────────────────────────┐  协议栈逐层上送
   │ ip_rcv → tcp_v4_rcv       │     查路由 / netfilter / 比对四元组
   └────┬─────────────────────┘
        │ ⑦ 找到 struct sock，数据入 sk_receive_queue
        ▼
   ┌──────────────────────────┐
   │ socket 接收队列 + 唤醒进程  │  recv()/epoll 被唤醒，拷贝到用户 buf
   └──────────────────────────┘
```

七道关卡中，①②是硬件与 DMA，③④是中断的"上半部"，⑤⑥是软中断与 NAPI（CPU 真正干活的地方），⑦是协议栈与 socket。下面逐段拆。

### 1.2 网卡 DMA 与 RX ring：CPU 还没醒，包已经在内存里了

现代网卡收包**不经过 CPU 拷贝**。驱动在初始化时为每个 RX 队列分配一个**环形描述符队列（RX ring / descriptor ring）**，环里每个描述符（descriptor）指向一块预分配的内存缓冲区（DMA buffer）。网卡收到帧后，通过 DMA 引擎把帧内容直接写进描述符指向的那块内存，然后更新描述符状态位标记"这个 slot 已填充"。

```
RX ring（假设 4096 个描述符）
                head（网卡写到这）           tail（驱动消费到这）
                  ▼                            ▼
  ┌────┬────┬────┬────┬────┬─────────────┬────┬────┐
  │DESC│DESC│DESC│ ●  │ ●  │   ...空...   │ ●  │ ●  │
  └────┴────┴────┴────┴────┴─────────────┴────┴────┘
        │    │    │
        ▼    ▼    ▼
      [buf][buf][buf]   ← DMA buffer（帧内容直接落在这里）
```

环大小用 `ethtool -g` 查看与调整：

```bash
$ ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:             8192
TX:             8192
Current hardware settings:
RX:             1024        # 当前 RX ring 深度
TX:             1024

# 丢包/突发场景下增大 RX ring（注意：大 ring 会增大延迟与缓存压力）
$ sudo ethtool -G eth0 rx 4096 tx 4096
```

ring 太小：突发流量来不及消费就会**溢出丢包**（`rx_no_buffer` / `rx_missed_errors` 计数上涨）。ring 太大：占内存、增大排队延迟，且在 bufferbloat 意义上对延迟不利。它是吞吐与延迟之间的一个旋钮，不是越大越好。

### 1.3 从 poll 到协议栈：sk_buff 诞生

软中断里驱动的 `poll()` 函数从 RX ring 收割描述符，把每个 DMA buffer 包装成一个 `sk_buff`（内核里所有网络数据包的统一载体，下一章详述），然后调用 `napi_gro_receive()` 上送。`napi_gro_receive` 会先尝试 **GRO 合并**（见第五章），合并不了或合并完成的包进入 `__netif_receive_skb_core()`：

- 先过 **tc ingress / XDP generic**（如果挂了的话）；
- 按 `skb->protocol` 分发到对应的协议处理函数。对 IPv4 是 `ip_rcv()`。

`ip_rcv()` 做：校验和检查、过 **netfilter** 的 `PREROUTING` hook（iptables/nftables、conntrack 在此）、查路由（`ip_route_input`）判断本机收还是转发。本机收的包进 `ip_local_deliver()`，按 L4 协议号分发——TCP 走 `tcp_v4_rcv()`，UDP 走 `udp_rcv()`。

`tcp_v4_rcv()` 用包的五元组（源/目的 IP、源/目的端口、协议）在连接哈希表里查到对应的 `struct sock`，做 TCP 状态机处理（序号校验、ACK、拥塞窗口更新——这些是 [L12](./L12-精通-TCP-IP-内核实现与调优.md) 的内容），最终把 payload 对应的 `sk_buff` 挂到 `sk->sk_receive_queue`，并调用 `sk->sk_data_ready()` 唤醒等待的进程或触发 epoll 就绪。

### 1.4 用户态如何被唤醒

阻塞在 `recv()` 上的进程此前已经睡在 socket 的等待队列上。`sk_data_ready` 触发后，调度器把它唤醒；进程从内核态把 `sk_receive_queue` 里的数据拷贝到用户缓冲区（`copy_to_user`），更新 socket 偏移，返回字节数。对 epoll，`sk_data_ready` 会把对应 fd 标记为就绪、唤醒 `epoll_wait`（细节见 [L08 I/O 多路复用](./L08-精通-IO-多路复用.md)）。

至此一个包走完全程。注意整条路上有**两次明显的"批处理"机会**：软中断里 NAPI 一次 poll 收多个包（减少中断次数），以及 GRO 把多个小包合并成一个大包（减少协议栈遍历次数）。这两个"批"是高吞吐的关键，后面详谈。

---

## 第二章 sk_buff：内核网络的万能载体

### 2.1 为什么需要 sk_buff

一个数据包在内核里要被多个协议层处理：链路层加/剥以太网头，IP 层加/剥 IP 头，TCP 层加/剥 TCP 头。如果每层都重新分配内存、拷贝 payload，开销不可接受。`sk_buff`（socket buffer，常简称 skb）的设计目标就是：**让数据只分配一次、拷贝尽量少，各层通过移动指针来"加头/剥头"**。

```c
// include/linux/skbuff.h（大幅简化）
struct sk_buff {
    struct sk_buff      *next, *prev;   // 挂入队列用
    struct sock         *sk;            // 所属 socket（如果有）
    struct net_device   *dev;           // 收/发的网卡
    unsigned int        len;            // 当前数据总长（含各层头）
    unsigned int        data_len;       // 分片/非线性部分长度
    __u16               mac_header;     // 各层头相对 head 的偏移
    __u16               network_header;
    __u16               transport_header;
    unsigned char       *head;          // 缓冲区起点
    unsigned char       *data;          // 当前有效数据起点
    unsigned char       *tail;          // 当前有效数据终点
    unsigned char       *end;           // 缓冲区终点
    refcount_t          users;          // 引用计数
    // ... 还有 cb[]、各种 offload 标志、时间戳等
};
```

关键的是 `head/data/tail/end` 四个指针把一块缓冲区切成三段：

```
 head        data            tail         end
  │           │               │            │
  ▼           ▼               ▼            ▼
  ┌───────────┬───────────────┬────────────┐
  │ headroom  │   实际数据     │  tailroom  │
  └───────────┴───────────────┴────────────┘
   预留给前面    payload + 已加  预留给后面
   要加的头       的协议头        要加的尾
```

- **headroom**（`data - head`）：预留给前面要加的协议头。收包时各层"剥头"就是把 `data` 往后移；发包时各层"加头"（`skb_push`）就是把 `data` 往前移、占用 headroom。
- **tailroom**（`end - tail`）：预留给数据增长或尾部字段。

这套设计让"加 IP 头""加 TCP 头"几乎零拷贝——只是指针运算。常用操作：

| 函数 | 作用 | 典型场景 |
|---|---|---|
| `skb_push(skb, n)` | data 前移 n，占 headroom | 发包时加协议头 |
| `skb_pull(skb, n)` | data 后移 n | 收包时剥协议头 |
| `skb_put(skb, n)` | tail 后移 n，占 tailroom | 往 payload 尾部追加 |
| `skb_reserve(skb, n)` | 预留 headroom | 分配后预留以太网/IP 头空间 |

### 2.2 线性区与非线性区：分片数据

并非所有 payload 都连续放在 `head..end` 这块线性缓冲区里。大包（尤其 GRO/GSO 后）的数据可能分散在多个内存页里，由 `skb_shared_info`（位于线性缓冲区末尾，`end` 之后）的 `frags[]` 数组描述，每个 frag 指向一个页片段。`len` 是总长，`data_len` 是非线性部分长度，线性部分长度 = `len - data_len`。

```
   skb（线性区）                 skb_shared_info.frags[]
  ┌──────────────┐              ┌──────┐ ┌──────┐ ┌──────┐
  │ headers+部分  │   ───────►  │ page │ │ page │ │ page │
  │ payload(线性) │              │ frag │ │ frag │ │ frag │
  └──────────────┘              └──────┘ └──────┘ └──────┘
```

这就是为什么处理 skb 时不能直接 `memcpy(skb->data, ..., skb->len)`——非线性数据不在 `data` 后面。需要连续访问时用 `skb_linearize()`（有拷贝开销）或 `skb_header_pointer()`。

### 2.3 克隆与共享：clone vs copy

skb 的引用计数有两层，区分两种"复制"：

- **`skb_clone()`**：复制 `sk_buff` 结构体本身，但**共享同一块数据缓冲区**（`skb_shared_info` 里的 `dataref` 增加）。两个 skb 各有独立的 `head/data/tail` 指针，但指向同一份数据。典型用途：`tcpdump` 抓包要拿一份、协议栈正常处理拿一份——抓包不能改数据，所以共享即可。
- **`skb_copy()`**：连数据缓冲区一起深拷贝，两份完全独立。
- **`pskb_copy()`**：只拷贝线性部分（头），共享 frags 页。

**写时复制陷阱**：如果一个 skb 被 clone（`skb_cloned()` 为真）后，某层又想修改数据，必须先 `skb_unshare()` / `pskb_expand_head()` 做实际拷贝，否则会破坏共享方的数据。这就是为什么"抓包"理论上不影响转发性能——它走的是 clone 共享路径。

### 2.4 生命周期与分配开销

skb 由 slab 缓存（`skbuff_head_cache`，见 [L05 物理内存管理](./L05-精通-物理内存管理与回收.md) 的 slab）分配；数据缓冲区从页分配器拿。释放走 `kfree_skb()`（异常丢弃，会被 `drop_monitor` / `kfree_skb` tracepoint 捕获）或 `consume_skb()`（正常消费完）。**区分这两个很重要**：用 bpftrace 跟踪 `kfree_skb` 能定位"包在内核哪一层被丢了"：

```bash
# 统计内核丢包的位置（drop reason，5.18+ 提供 reason 枚举）
sudo bpftrace -e 'tracepoint:skb:kfree_skb { @[args->reason] = count(); }'
```

高速路径下，skb 的分配/释放本身就是热点，所以才有 GRO（少造 skb）、page_pool（复用 DMA 页）、以及 XDP（在造 skb 之前就处理）这些优化。

---

## 第三章 中断与 NAPI：硬中断为什么"什么都不干"

### 3.1 硬中断的代价与"中断风暴"

最朴素的收包模型是：每来一个包，网卡触发一次硬中断，中断处理函数把包收进协议栈。这个模型在低速时代没问题，但在 10G+ 网卡、每秒上百万包时是灾难：

- 硬中断会**抢占当前 CPU 上的一切**（包括内核态），频繁中断导致 CPU 大量时间花在保存/恢复上下文、cache 被反复刷新；
- 极端情况下中断到来的速率超过处理速率，CPU 永远在处理中断、永远轮不到上层逻辑——这叫 **receive livelock（接收活锁）**，系统看似没崩但吞吐归零。

解决方案是 **NAPI（New API）**：硬中断只负责"通知"，真正的收包改为软中断里**轮询（polling）批量收割**。

### 3.2 NAPI 的核心机制：中断 + 轮询的混合

NAPI 的精妙在于**自适应地在中断和轮询之间切换**：

```
低负载（包稀疏）：              高负载（包密集）：
  来包 → 硬中断 → 处理            来包 → 硬中断（第一次）
  （中断开销可接受，低延迟）         → 关本队列中断
                                  → 软中断里 poll，一次收一批
                                  → ring 还满？继续 poll
                                  → ring 空了？重新开中断
                                  （轮询期间不再有中断，省开销）
```

具体流程：

1. **硬中断处理函数（上半部，top half）**只做两件事：调用 `napi_schedule()` 把这个 NAPI 实例挂到当前 CPU 的 poll 链表、`__raise_softirq_irqoff(NET_RX_SOFTIRQ)`；并**关闭本队列的硬中断**。然后立即返回。它不碰任何包数据。

2. **软中断处理（下半部，bottom half）** `net_rx_action()` 遍历 poll 链表，对每个 NAPI 调用驱动的 `poll(budget)`。`budget` 是本次最多收多少个包（默认每设备 64，全局上限 `netdev_budget`）。

3. poll 收割 RX ring：
   - 如果收到的包数 **< budget**（说明 ring 快空了，流量不密集），调用 `napi_complete_done()`，**重新开启本队列硬中断**，退出轮询模式；
   - 如果收满 **= budget**（ring 还有货，流量密集），不开中断，把自己留在 poll 链表里，等下一轮 `net_rx_action` 继续 poll。

这样：**流量稀疏时靠中断（低延迟），流量密集时靠轮询（低开销）**，且轮询期间硬中断被抑制，从根本上避免了中断风暴和活锁。

### 3.3 softirq 与 ksoftirqd

软中断（softirq）不是单独的线程，它在以下时机被执行：

- 硬中断返回时（`irq_exit` → `__do_softirq`）——这是最常见的，软中断"借"了刚处理完硬中断的那个 CPU；
- 系统调用返回、内核抢占点等。

`__do_softirq` 有个保护：单次最多处理一定量（受 `MAX_SOFTIRQ_TIME` ~2ms 和 `MAX_SOFTIRQ_RESTART` 次数限制），如果软中断还没处理完就超了，剩下的工作**唤醒 `ksoftirqd/N` 内核线程**（每 CPU 一个）去做。所以：

- `top` 里 `%si` 高 = 软中断在硬中断上下文里跑得多；
- `ksoftirqd/N` 进程 CPU 高 = 软中断量太大，溢出到了线程上下文。

```bash
# 看每 CPU 的 softirq 计数，NET_RX 那一列暴涨说明收包软中断压力大
$ cat /proc/softirqs | grep -E "NET_RX|NET_TX"
# 看 softnet：第1列 processed，第2列 dropped，第3列 time_squeeze
$ cat /proc/net/softnet_stat
```

`/proc/net/softnet_stat` 是排障金矿（每行一个 CPU，十六进制）：
- 第 2 列 **dropped**：因 backlog 满而丢的包（RPS 相关，见第四章）；
- 第 3 列 **time_squeeze**：一次 `net_rx_action` 没在 budget/时间内处理完就被迫退出的次数——非零且增长说明**单核软中断处理不过来**，是该上 RPS/RFS 或多队列的信号。

### 3.4 中断合并（interrupt coalescing）

即便有 NAPI，第一个包仍要触发硬中断。**中断合并**让网卡"攒一攒再中断"：等收到 N 个包、或经过 T 微秒后才触发一次硬中断，进一步降低中断频率。

```bash
$ ethtool -c eth0
rx-usecs: 50          # 收到包后最多等 50us 再中断
rx-frames: 64         # 或攒够 64 个包就中断
adaptive-rx: on       # 自适应：根据流量动态调

# 延迟敏感场景：减小 usecs（更快中断，更低延迟，更高 CPU）
$ sudo ethtool -C eth0 rx-usecs 10
# 吞吐优先场景：增大 usecs（攒批，更省 CPU，延迟略增）
$ sudo ethtool -C eth0 rx-usecs 100 adaptive-rx on
```

这又是一个**延迟 vs CPU/吞吐**的旋钮。许多网卡的 `adaptive-rx`（自适应中断合并）能在两者间自动平衡，是通用场景的合理默认。

---

## 第四章 多队列与多核扩展：RSS / RPS / RFS / XPS

单核处理收包软中断终会到顶（一个核打满 `%si`）。要榨干多核，必须把收包负载**横向铺开到多个 CPU**。Linux 提供硬件和软件两套机制。

### 4.1 RSS：硬件多队列分流

**RSS（Receive Side Scaling）**是网卡硬件特性：网卡有多个 RX 队列，对每个进来的包按五元组算一个哈希（Toeplitz hash），用哈希值选队列。每个 RX 队列绑定一个独立的硬中断（MSI-X），可以把不同队列的中断**亲和（affinity）到不同 CPU**，于是不同 CPU 并行处理不同队列的软中断。

```bash
# 看队列数（Combined 通常是收发合一的队列对数）
$ ethtool -l eth0
Combined:       8

# 调整队列数（一般设为物理核数，最多到网卡上限）
$ sudo ethtool -L eth0 combined 16

# 看每个队列的中断号
$ grep eth0 /proc/interrupts
# 把队列 0 的中断绑到 CPU2（mask=0x4）
$ echo 4 | sudo tee /proc/irq/<irq>/smp_affinity
```

**RSS 是首选**——它在硬件就分好流，软件零开销。前提是网卡支持足够队列。`irqbalance` 守护进程会自动分配中断亲和，但对网络密集型/低延迟场景，**手动把队列中断 pin 到固定核**往往更可控（避免 irqbalance 动态迁移导致 cache 抖动）。一个常见做法是把 NIC 队列中断绑到与处理它的应用同 NUMA 节点的核上。

### 4.2 RPS：软件版 RSS

当网卡队列数不够（比如只有 1 个队列的虚拟网卡、老网卡），或想把单队列的流量再分散，用 **RPS（Receive Packet Steering）**：在软中断收到包后、进协议栈前，**按包哈希选一个目标 CPU**，通过 IPI（处理器间中断）把包丢到目标 CPU 的 **backlog 队列**，由目标 CPU 的软中断继续处理协议栈。

```bash
# 为 eth0 的 rx-0 队列开启 RPS：把流量散到 CPU0-7（掩码 0xff）
$ echo ff | sudo tee /sys/class/net/eth0/queues/rx-0/rps_cpus

# backlog 队列上限（防止目标 CPU 处理不过来时无限堆积）
$ sysctl net.core.netdev_max_backlog   # 默认 1000，高速场景可调大
```

RPS 有 IPI 和跨核 cache 迁移的开销，所以**有 RSS 时优先 RSS**；RPS 是"硬件不给力时的软件补救"或"单队列散核"的手段。

### 4.3 RFS：让数据跟着应用走

RPS 纯按哈希散核，可能把包送到 CPU-A 处理协议栈，而消费它的应用线程跑在 CPU-B——数据要跨核，socket 状态的 cache 来回弹，反而变慢。**RFS（Receive Flow Steering）**修正这点：它维护一张表记录"每条流上次被哪个 CPU 上的应用 `recvmsg` 过"，让同一条流的包尽量**送到运行其应用的那个 CPU** 处理，提升 cache 局部性。

```bash
# 全局流表大小
$ echo 32768 | sudo tee /proc/sys/net/core/rps_sock_flow_entries
# 每队列流表大小（通常 = 全局 / 队列数）
$ echo 2048 | sudo tee /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

硬件版的 RFS 叫 **aRFS（accelerated RFS）**，由支持的网卡直接把流导向正确队列（结合 ntuple filter），省掉软件 steering。

| 机制 | 层次 | 分流依据 | 何时用 |
|---|---|---|---|
| **RSS** | 硬件 | 包五元组哈希 | 首选，多队列网卡 |
| **RPS** | 软件 | 包哈希 → CPU | 队列不足/单队列散核 |
| **RFS** | 软件 | 应用所在 CPU | RPS 基础上提 cache 局部性 |
| **aRFS** | 硬件 | 应用所在 CPU | 网卡支持时，RFS 的硬件加速 |
| **XPS** | 软件 | 发包 CPU → TX 队列 | 发送侧多队列 |

### 4.4 XPS：发送侧的对称机制

**XPS（Transmit Packet Steering）**是发送方向的：让某个 CPU 发的包固定走某个 TX 队列，避免多核争抢同一 TX 队列的锁、并提升 cache 局部性。

```bash
# CPU0-3 用 tx-0 队列
$ echo f | sudo tee /sys/class/net/eth0/queues/tx-0/xps_cpus
```

### 4.5 NUMA 与亲和：别让包跨节点

在多 socket（NUMA）机器上，PCIe 网卡物理挂在某个 NUMA 节点下。如果中断和应用跑在另一个节点，每个包都要跨 NUMA 互联访问内存，延迟与带宽都受损。原则：**网卡中断亲和、RPS/RFS 目标 CPU、以及处理网络的应用线程，尽量都落在网卡所在的 NUMA 节点**。

```bash
# 查网卡在哪个 NUMA 节点
$ cat /sys/class/net/eth0/device/numa_node
0
# 查节点的 CPU 列表
$ lscpu | grep NUMA
```

---

## 第五章 offload：GRO / GSO / TSO / LRO 与校验和

协议栈处理每个包都有固定开销（查表、状态机、函数调用）。**如果能让协议栈"看到"的是少量大包而不是大量小包，每字节的 CPU 成本就大幅下降。** offload 就是干这个的，分收发两个方向。

### 5.1 收方向：GRO 与 LRO

- **LRO（Large Receive Offload）**：网卡硬件把同一条流的多个 TCP 段合并成一个大段再交给内核。缺点：合并是"有损"的——它可能丢弃/合并掉一些头部差异信息，破坏转发与 bridging 语义，所以**做路由器/网桥/容器宿主时不能用 LRO**。

- **GRO（Generic Receive Offload）**：内核软件实现的合并，在 `napi_gro_receive` 里做。它**严格**地只合并那些"合并后能被无损还原成原始包序列"的段（要求各字段一致、时间戳/序号连续），因此对转发安全。GRO 是现代默认，几乎总该开着。

```bash
$ ethtool -k eth0 | grep -E "generic-receive|large-receive|tcp-segmentation|generic-segmentation"
generic-receive-offload: on          # GRO
large-receive-offload: off           # LRO（路由/网桥/容器宿主应关）
tcp-segmentation-offload: on         # TSO
generic-segmentation-offload: on     # GSO
```

GRO 把比如 10 个 1500 字节的 MSS 段合并成一个 ~15KB 的 skb（用前述非线性 frags 装载），协议栈只走一遍，CPU 大降。

### 5.2 发方向：TSO 与 GSO

- **TSO（TCP Segmentation Offload）**：内核把一大块数据（比如 64KB）作为一个"超大段"交给网卡，**由网卡硬件切成符合 MSS 的多个 TCP 段**并各自填好头、算校验和。协议栈只处理一个大段，省 CPU。

- **GSO（Generic Segmentation Offload）**：软件版分段，**尽量推迟分段到驱动层之前才做**（如果网卡支持 TSO 就交给硬件，不支持就软件切）。它是 TSO 在通用层的兜底。

- **校验和 offload（checksum offload）**：网卡硬件算 IP/TCP/UDP 校验和，CPU 不用逐字节算。`tx-checksumming` / `rx-checksumming`。

### 5.3 offload 对抓包的影响（重要的坑）

`tcpdump` 抓包是在协议栈里、offload 之后（收）/之前（发）抓的，于是你会看到**违反直觉的"巨型包"**：

- 收方向开了 GRO：tcpdump 看到一个 15KB 的 TCP 段，但线上实际是 10 个 1500B 的包——因为抓包点在 GRO 合并之后。
- 发方向开了 TSO/GSO：tcpdump 看到一个 64KB 的段，实际网线上是切好的多个 MSS 段——抓包点在分段之前。

所以**用 tcpdump 排查 MTU/分片/MSS 问题时，结果可能误导**。要看"线上真实包"，需临时关 offload：

```bash
# 排查时临时关闭，看到的就是接近线缆真实的包
$ sudo ethtool -K eth0 gro off gso off tso off lro off
# 排查完记得开回来（关掉会显著掉吞吐、涨 CPU）
$ sudo ethtool -K eth0 gro on gso on tso on
```

| offload | 方向 | 在哪做 | 作用 | 抓包可见的假象 |
|---|---|---|---|---|
| GRO | 收 | 内核软件 | 合并小段为大段 | 收到"超大段" |
| LRO | 收 | 网卡硬件 | 合并（有损，转发禁用） | 收到"超大段" |
| TSO | 发 | 网卡硬件 | 切大段为 MSS | 发出"超大段" |
| GSO | 发 | 内核软件 | 软件切段（TSO 兜底） | 发出"超大段" |
| checksum | 收发 | 网卡硬件 | 算校验和 | 发包校验和显示为 0/incorrect |

---

## 第六章 发包路径：从 send() 到网卡

### 6.1 协议栈下行

`send()`/`write()` 进内核后：TCP 层把用户数据拷进 skb（受发送缓冲区 `sk_sndbuf` 与拥塞窗口约束，见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）、加 TCP 头；IP 层加 IP 头、查路由、过 netfilter `OUTPUT`/`POSTROUTING`；然后调 `dev_queue_xmit()` 把 skb 交给**流量控制层（qdisc）**。

```
 send() → tcp_sendmsg → ip_queue_xmit → dev_queue_xmit
                                              │
                                              ▼
                                    ┌──────────────────┐
                                    │   qdisc 队列规则   │ ← 排队/整形/调度
                                    └────────┬─────────┘
                                              │ 出队
                                              ▼
                                    ┌──────────────────┐
                                    │ 驱动 ndo_start_xmit│ → 填 TX ring 描述符
                                    └────────┬─────────┘
                                              │ DMA + 触发发送
                                              ▼
                                        网卡 → 网线
```

发送完成后网卡触发 TX 中断，软中断 `NET_TX_SOFTIRQ` 回收已发送的 skb（释放缓冲区）。

### 6.2 qdisc：排队规则

**qdisc（queueing discipline）**决定包以什么顺序、什么速率离开。它是排队论意义上的"队列管理器"，承担调度、整形（shaping）、AQM（主动队列管理，对抗 bufferbloat）等职责。

```bash
# 看网卡当前 qdisc
$ tc qdisc show dev eth0
qdisc fq_codel 0: root refcnt 2 limit 10240p flows 1024 ...

# 多队列网卡通常默认 mq（每个硬件 TX 队列挂一个子 qdisc）
```

常见 qdisc：

| qdisc | 特点 | 适用 |
|---|---|---|
| `pfifo_fast` | 三条优先级 FIFO，按 ToS 分级，无整形 | 老默认，简单 |
| `fq_codel` | 公平排队 + CoDel AQM，控制队列延迟、抗 bufferbloat | **现代通用默认**，多数发行版默认 |
| `fq` | Fair Queue，per-flow 调度 + pacing，配合 BBR 做发送节奏控制 | BBR / 高吞吐服务器 |
| `mq` | 多队列伞 qdisc，给每个硬件 TX 队列挂独立子 qdisc | 多队列网卡自动用 |
| `htb` | 分层令牌桶，做带宽分级限速 | 需要限速/QoS 时 |

**fq vs fq_codel**：`fq_codel` 重在用 CoDel 算法压低排队延迟（对抗 bufferbloat，桌面/通用路由默认）；`fq` 重在做 per-flow pacing（按节奏发包），是 **BBR 拥塞控制的推荐搭档**——BBR 需要 pacing 才能精确控速。高吞吐服务器配 BBR 时常设 `fq`：

```bash
$ sudo sysctl net.core.default_qdisc=fq
$ sudo sysctl net.ipv4.tcp_congestion_control=bbr
```

### 6.3 BQL：字节队列限制

驱动层还有 **BQL（Byte Queue Limits）**：动态限制 TX ring 里在途字节数，防止驱动 ring 里堆太多数据（那也是一种 bufferbloat）。BQL 让 qdisc 的 AQM 真正起作用——如果数据都堆在驱动 ring 里，qdisc 的智能调度就被架空了。BQL 现代驱动默认开启，一般无需手调。

---

## 第七章 XDP / eBPF：在协议栈之前拦截

### 7.1 XDP 是什么，快在哪

前面所有处理都发生在 skb 创建**之后**。但造 skb、走协议栈本身就有成本。**XDP（eXpress Data Path）**让你在**驱动收到包、还没造 skb 的最早时刻**，运行一段 eBPF 程序直接对原始包（`xdp_md`，指向 DMA buffer）做决策。因为绕过了 skb 分配和大半个协议栈，XDP 处理单包的成本极低，能在单核上做到**千万级 pps 的丢弃/转发**，这是 iptables/nftables 难以企及的。

XDP 有三种运行模式：

| 模式 | 在哪运行 | 性能 | 要求 |
|---|---|---|---|
| **Native（驱动）** | 驱动 poll 里、造 skb 前 | 最高 | 驱动支持 XDP |
| **Offloaded** | 网卡硬件（SmartNIC） | 最高、零 CPU | 特定网卡（如部分 SmartNIC） |
| **Generic（SKB）** | 协议栈早期、已造 skb | 低（仅测试用） | 任何网卡 |

### 7.2 XDP 动作

XDP 程序返回一个动作码，决定包的命运：

| 动作 | 含义 | 典型用途 |
|---|---|---|
| `XDP_DROP` | 立即丢弃 | DDoS 防护：在最前面丢掉攻击流量，成本极低 |
| `XDP_PASS` | 正常上送协议栈 | 不感兴趣的包放行 |
| `XDP_TX` | 从**收到的同一网卡**原路发回 | 反射类负载均衡、回包 |
| `XDP_REDIRECT` | 重定向到另一网卡 / CPU / AF_XDP socket | L4 LB、用户态收包、cpumap |
| `XDP_ABORTED` | 异常丢弃（带 tracepoint） | 程序出错 |

一个极简 XDP 程序（丢掉所有到 UDP 53 端口的包，示意 DDoS 过滤思路）：

```c
// xdp_drop_dns.c —— 用 clang -O2 -target bpf 编译
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int drop_dns(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;   // 边界检查（verifier 要求）
    if (eth->h_proto != __constant_htons(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;
    if (ip->protocol != IPPROTO_UDP) return XDP_PASS;

    struct udphdr *udp = (void *)ip + ip->ihl * 4;
    if ((void *)(udp + 1) > data_end) return XDP_PASS;

    if (udp->dest == __constant_htons(53))
        return XDP_DROP;     // 丢掉到 53 端口的包

    return XDP_PASS;
}
char _license[] SEC("license") = "GPL";
```

```bash
# 加载到网卡（native 模式；用 ip 或 bpftool）
$ sudo ip link set dev eth0 xdp obj xdp_drop_dns.o sec xdp
$ ip link show eth0          # 看到 xdp/id 说明已挂
$ sudo ip link set dev eth0 xdp off   # 卸载
```

注意每个内存访问前的**边界检查**——这是 eBPF verifier 的硬性要求，没有就加载失败（verifier 细节见 [L19 eBPF](./L19-精通-eBPF.md)）。

### 7.3 生产落地：DDoS 防护与 L4 负载均衡

- **DDoS 防护**：Cloudflare 用 XDP 在边缘机器上把攻击包在协议栈之前丢掉，单机能扛住极高 pps 的洪泛而不耗尽 CPU——因为被丢的包从不进协议栈、从不造 skb。
- **L4 负载均衡**：Facebook/Meta 的 **Katran** 是基于 XDP 的 L4 LB，用 `XDP_TX`/`XDP_REDIRECT` 把入向流量按一致性哈希转发到后端，无需经过完整协议栈。
- **容器网络**：**Cilium** 用 XDP + tc eBPF 实现 K8s 的 Service 负载均衡、NetworkPolicy、kube-proxy 替代，把大量包处理从 iptables 链搬到 eBPF（详见 cloud-native C03 K8s 网络）。
- **AF_XDP**：通过 `XDP_REDIRECT` 把包送进用户态 socket，实现内核旁路（kernel bypass）的高性能用户态收包，是 DPDK 之外的轻量选择。

XDP 与 tc eBPF 的分工：XDP 在最早期、只能看入向、性能最高；tc eBPF（`tc` 的 clsact）在 skb 之后、收发双向都能挂、能改包能用更多 helper。两者常配合使用。

---

## 生产实践

**场景：软中断打满单核，P99 延迟突然翻倍 → RPS/RFS 调优**

某 Go 网关服务在流量上涨后 P99 延迟从 5ms 跳到 80ms，但 CPU 总体使用率才 30%。

**第一步：确认是软中断单核瓶颈。**

```bash
$ mpstat -P ALL 1
# 发现 CPU3 的 %soft = 99%，其余核 %soft 几乎为 0 —— 收包软中断全压在一个核
$ cat /proc/net/softnet_stat
# CPU3 对应行的第 3 列（time_squeeze）持续暴涨 —— net_rx_action 处理不完被迫退出
```

典型根因：网卡只有 1 个 RX 队列（虚拟机/云上常见），或 RSS 队列数太少，所有收包软中断都落在同一个核。

**第二步：先看能不能上硬件多队列（RSS）。**

```bash
$ ethtool -l eth0
# 如果 Combined 可以 >1，直接加队列并把中断分散到多核（最优）
$ sudo ethtool -L eth0 combined 8
# 然后把各队列中断 pin 到不同核（或交给 irqbalance）
```

**第三步：单队列网卡没法上 RSS，用软件的 RPS + RFS 散核。**

```bash
# 把 rx-0 的软中断散到 CPU0-7
$ echo ff | sudo tee /sys/class/net/eth0/queues/rx-0/rps_cpus
# 开 RFS，让流跟着应用走，减少跨核 cache 抖动
$ echo 32768 | sudo tee /proc/sys/net/core/rps_sock_flow_entries
$ echo 4096  | sudo tee /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
# backlog 调大，防目标核瞬时堆积丢包
$ sudo sysctl -w net.core.netdev_max_backlog=4000
```

**第四步：验证。** 再看 `mpstat`，`%soft` 应从单核 99% 摊到多核各 ~15%；`softnet_stat` 的 time_squeeze 不再增长；P99 回落。

**配套检查**：确认 GRO 开着（`ethtool -k`）以减少每核的软中断处理量；NUMA 机器上确认 `rps_cpus` 选的核与网卡同节点；中断合并 `adaptive-rx on` 兜底。这套组合（多队列 / RPS+RFS / GRO / NUMA 亲和）是网络软中断瓶颈的标准处方。

---

## 陷阱清单

1. **现象**：`tcpdump` 看到 64KB 的"巨型 TCP 包"，怀疑 MTU 配错。
   **原因**：开了 GRO/TSO，抓包点在合并之后（收）/分段之前（发），看到的不是线缆真实包。
   **修法**：排查 MTU/分片问题时临时 `ethtool -K eth0 gro off gso off tso off`，看到接近真实的包，排查完务必开回（关掉显著掉吞吐）。

2. **现象**：CPU 总使用率不高，但某个核 `%si` 100%，延迟毛刺。
   **原因**：收包软中断全压在单核（单 RX 队列 / RSS 队列太少 / 中断未分散）。
   **修法**：`ethtool -L` 加 RSS 队列并分散中断；不行则 RPS（`rps_cpus`）+ RFS 软件散核。看 `/proc/net/softnet_stat` 第 3 列 time_squeeze 确认。

3. **现象**：`/proc/net/softnet_stat` 第 2 列（dropped）持续增长，应用层偶发丢包。
   **原因**：RPS backlog 队列满，目标 CPU 处理不过来。
   **修法**：调大 `net.core.netdev_max_backlog`；同时排查目标核是否本身就过载（要再加散核范围或减负）。

4. **现象**：宿主机做容器/路由转发，开了 LRO 后出现诡异的转发错误、TCP 性能异常。
   **原因**：LRO 是有损合并，破坏转发语义；转发/网桥/容器宿主场景不能用 LRO。
   **修法**：`ethtool -K eth0 lro off`，改用无损的 GRO（GRO 对转发安全）。

5. **现象**：丢包但 `ifconfig`/应用层都看不出在哪丢的。
   **原因**：包可能在协议栈某层被 `kfree_skb` 丢弃（校验和、conntrack、队列满等），常规计数看不到位置。
   **修法**：`bpftrace -e 'tracepoint:skb:kfree_skb { @[args->reason]=count(); }'` 按 drop reason 定位；或看 `nstat -az` / `ethtool -S` 的细分计数。

6. **现象**：开了 BBR 但吞吐/公平性不如预期，发包"一阵一阵"。
   **原因**：qdisc 用的是 `pfifo_fast`，没有 pacing，BBR 的发送节奏控制失效。
   **修法**：`sysctl net.core.default_qdisc=fq`（BBR 推荐搭配 fq 做 pacing），重置网卡或重启使生效。

7. **现象**：NUMA 多 socket 机器上网络延迟比单 socket 还高。
   **原因**：网卡中断/软中断/应用线程分散在网卡所在节点之外，每包跨 NUMA 访存。
   **修法**：查 `/sys/class/net/eth0/device/numa_node`，把中断亲和、`rps_cpus`、应用线程都收到同节点核。

8. **现象**：增大了 RX ring（`ethtool -G rx 8192`）后吞吐没涨，延迟反而变差。
   **原因**：ring 过大引入排队延迟（bufferbloat），且并非瓶颈所在。
   **修法**：ring 只在确有 `rx_missed_errors`/溢出丢包时才加；瓶颈在软中断处理时应去做散核而非加 ring。

---

## 2026 现状

- **NAPI/软中断模型稳定**：收包仍是"硬中断触发 + softirq/NAPI 轮询批处理"，这套机制十多年来基本未变，是理解一切网络性能问题的地基。`netdev_budget` / NAPI 阈值仍是高 pps 调优旋钮；近年的改进集中在 **page_pool** 与 DMA 页复用、以及 **threaded NAPI**（`/sys/class/net/*/threaded`，把 NAPI poll 放到专用内核线程而非 ksoftirqd，便于调度隔离与亲和控制）。
- **GRO/GSO/TSO 与校验和 offload 是普遍默认**，LRO 在转发/容器宿主场景默认关闭。`fq_codel` 是多数发行版默认 qdisc，BBR + `fq` 在高吞吐服务器广泛部署（BBRv3 持续推进，见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）。
- **XDP/eBPF 大规模落地**：DDoS 防护（Cloudflare 边缘）、L4 LB（Meta Katran）、K8s 数据面（Cilium 替代 kube-proxy）已是生产标配。AF_XDP 成为内核旁路高性能收包的主流轻量方案。XDP 程序的 CO-RE/BTF 可移植性成熟（见 [L19 eBPF](./L19-精通-eBPF.md)）。
- **可观测进步**：`skb:kfree_skb` 的 **drop reason** 枚举（5.18+）让"内核在哪丢包"可精确定位；`nstat`、`ss`、bpftrace/bcc 的网络工具箱（`tcptop`、`tcpretrans`、`tcplife`）成为排障标配。
- **多队列与 NUMA 亲和**在 25/100GbE 普及后愈发关键；`irqbalance` 仍是通用默认，但延迟敏感/高 pps 服务普遍改用手动中断 pin + RFS + 应用线程亲和的"对齐到 NUMA 节点"方案。

---

## 练习题

1. 画出一个 TCP 数据包从网卡 DMA 到应用 `recv()` 的完整路径，标出硬中断、软中断、NAPI poll、协议栈、socket 接收队列各在哪一步，并说明哪两步是"批处理"优化点。

2. 解释 NAPI 中"硬中断里几乎什么都不做、只关中断 + 调度软中断"为什么是对的。如果回到"每包一中断"模型，在高 pps 下会发生什么？请说出 receive livelock 的成因。

3. `sk_buff` 的 `head/data/tail/end` 四个指针把缓冲区分成哪三段？发包时各层"加协议头"对应哪个指针的哪种移动、消耗哪段空间？为什么这样设计能减少拷贝？

4. `skb_clone()` 和 `skb_copy()` 的区别是什么？`tcpdump` 抓包为什么理论上不影响转发数据？如果 clone 之后某层要改包，必须先做什么操作，否则会发生什么？

5. 你在 `mpstat` 看到 CPU5 的 `%soft` 是 98%，其余核接近 0，应用 CPU 不忙但延迟高。给出从确认根因到修复的完整命令序列，并说明 RSS、RPS、RFS 各自解决什么、优先级如何。

6. `/proc/net/softnet_stat` 的第 2 列和第 3 列分别代表什么？各自非零增长时分别指向什么问题、对应什么调优动作？

7. 对比 GRO 与 LRO：为什么 GRO 对转发安全而 LRO 不安全？做 K8s 宿主机时这两个该如何设置？开了 TSO 后用 `tcpdump` 排查 MSS 问题为什么会被误导，正确做法是什么？

8. XDP 的四个主要动作（DROP/PASS/TX/REDIRECT）各自典型用途是什么？为什么 XDP 做 DDoS 丢包的单核成本远低于 iptables？写一段 XDP 程序时，为什么每次访问包数据前都要做边界检查？
