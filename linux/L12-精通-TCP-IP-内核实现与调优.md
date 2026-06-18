# 精通 TCP/IP 内核实现与调优：状态机、半连接/全连接队列、cubic/BBR、缓冲区、TIME_WAIT、sysctl

> 课程编号：L12
> 路线图来源：Linux · 模块四 网络
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：BBR/BBRv3、syncookies、accept 队列、TIME_WAIT；强调 `tcp_tw_recycle` 早已删除）

---

## 引言：sysctl 救场之前，先搞懂内核在干什么

线上最常见的几类「网络疑难」，几乎都能归到 TCP 内核行为上：

- 压测 QPS 上不去，`ss -s` 看到大量 `SYN-RECV`，客户端报 connection timeout；
- 服务重启后偶发 `connection refused`，但进程明明在 listen；
- 短连接服务跑一会儿，主调方出现成片 `TIME-WAIT`，新建连接报 `Cannot assign requested address`；
- 跨地域传输带宽明明有 1Gbps，单条 TCP 却只能跑到 20MB/s；
- 某个 RPC 偶发 40ms 延迟，抓包发现是「发完没立刻发完」。

这些现象背后，是**半连接队列与全连接队列**、**TIME_WAIT 的 2MSL**、**拥塞控制算法**、**收发缓冲区自动调整**、**Nagle 与 delayed ACK 的相互作用**。本篇从内核数据结构与状态机讲起，把每个现象对应到具体的内核机制和**真实存在的 sysctl**，最后给一份高并发服务器的调优清单——每个参数都标注含义与风险，而不是抄一份「优化脚本」了事。

收发包如何从网卡到达协议栈、软中断与 `sk_buff` 的细节，见 [L11 网络协议栈](./L11-精通-Linux-网络协议栈.md)；socket API 的内核路径与 `SO_REUSEPORT`、conntrack 见 [L13 Socket 与连接管理](./L13-精通-Socket-与连接管理.md)。

| 2026 现状 | 说明 |
|---|---|
| 默认拥塞控制 | 多数发行版仍是 **cubic**；BBR 需手动启用，已在大流量场景广泛部署，**BBRv3** 持续推进 |
| `tcp_tw_recycle` | **早已删除**（4.12 起移除），任何「优化」里出现它都是过时甚至有害的建议 |
| TCP Fast Open | `tcp_fastopen` 可用，但中间盒兼容性导致收益有限，需评估 |
| 缓冲区自动调整 | `tcp_moderate_rcvbuf` 默认开启，多数场景不需要手动设大 `SO_RCVBUF` |

---

## 第一章 内核态 TCP 状态机

### 1.1 一条连接在内核里是什么

用户态看到的是一个 fd，内核里对应一条「控制块」。socket 层是 `struct socket`，协议层是 `struct sock`（TCP 进一步是 `struct tcp_sock`，它内嵌 `inet_connection_sock` → `inet_sock` → `sock`）。`struct tcp_sock` 保存了一条连接的全部状态：序列号 `snd_nxt`/`rcv_nxt`、拥塞窗口 `snd_cwnd`、慢启动阈值 `snd_ssthresh`、RTT 估计 `srtt_us`、接收窗口、各种定时器等。

```
struct socket (BSD socket 层, 关联 fd)
   └── struct sock (网络层通用)
         └── struct inet_sock
               └── struct inet_connection_sock
                     └── struct tcp_sock   // cwnd / ssthresh / srtt / 序列号...
```

> `struct sock` 与 socket-fd 的关系详见 [L13](./L13-精通-Socket-与连接管理.md) 第一章。本篇聚焦 `tcp_sock` 的状态演化与可调参数。

### 1.2 11 个状态与迁移

TCP 状态机是协议核心。Linux 用 `sk->sk_state` 保存当前状态，取值就是经典的 11 态：

```
                          CLOSED
              passive open │        │ active open: send SYN
                 LISTEN ◄──┘        └──► SYN_SENT
                   │ recv SYN              │ recv SYN+ACK, send ACK
                   │ send SYN+ACK          ▼
                   ▼                    ESTABLISHED ◄───────┐
                SYN_RECV ──recv ACK──► ESTABLISHED          │
                                          │ recv FIN        │ send FIN
                              send FIN    │                 ▼
                          ┌───────────────┤            CLOSE_WAIT
                          ▼               ▼                 │ send FIN
                     FIN_WAIT_1      （同时关闭）            ▼
                          │ recv ACK      │             LAST_ACK
                          ▼               ▼                 │ recv ACK
                     FIN_WAIT_2        CLOSING              ▼
                          │ recv FIN      │ recv ACK     CLOSED
                          ▼               ▼
                     TIME_WAIT ──2MSL──► CLOSED
```

用 `ss` 实时观察状态分布：

```bash
ss -tan state established | wc -l        # ESTABLISHED 数量
ss -tan state time-wait | wc -l          # TIME-WAIT 数量
ss -s                                    # 总览：各状态计数
# 关注异常态：大量 SYN-RECV(半连接被打满/SYN flood)、CLOSE-WAIT(应用没 close)
```

**两个高频排障信号**：

- 大量 `CLOSE-WAIT`：对端已发 FIN，**本端应用迟迟不 `close()`**——几乎总是应用 bug（连接泄漏），不是内核问题。
- 大量 `SYN-RECV`：要么遭遇 SYN flood，要么半连接队列/CPU 处理不过来。

---

## 第二章 三次握手与两个队列

这是面试与线上最常考的一块，也是「连接建立失败」类问题的根。**内核为每个 LISTEN socket 维护两个队列**：

```
        client                         server (LISTEN)
          │   ─── SYN ───────────►       创建请求项，进入【半连接队列 SYN queue】
          │                              状态 SYN_RECV
          │   ◄── SYN + ACK ──────       
          │   ─── ACK ───────────►       握手完成，从半连接队列移入
          │                              【全连接队列 accept queue】，状态 ESTABLISHED
          │                                      │
          │                              应用 accept() 从全连接队列取走
```

### 2.1 半连接队列（SYN queue）

收到 SYN 后，内核创建一个 `request_sock` 放入半连接队列，回 SYN+ACK，等待对端 ACK。相关参数：

| 参数 | 含义 |
|---|---|
| `net.ipv4.tcp_max_syn_backlog` | 半连接队列上限（SYN_RECV 请求数的量级控制） |
| `net.ipv4.tcp_synack_retries` | SYN+ACK 重传次数（影响半开连接存活时长） |
| `net.ipv4.tcp_syncookies` | 半连接队列满时启用 syncookie，不占队列也能完成握手 |

**SYN flood 与 syncookies**：攻击者发大量 SYN 不回 ACK，半连接队列被占满，正常用户的 SYN 被丢弃。`tcp_syncookies=1`（默认）让内核在队列满时不分配 `request_sock`，而是把连接信息编码进 SYN+ACK 的序列号，对端回 ACK 时再校验还原——以此扛过洪水。

### 2.2 全连接队列（accept queue）

握手完成的连接进入全连接队列，等应用 `accept()`。**队列长度 = min(`listen()` 的 backlog 参数, `net.core.somaxconn`)**。这是「为什么我 backlog 设了 65535 却没用」的答案——被 `somaxconn` 截断了。

```bash
# 查看 LISTEN socket 的全连接队列：Recv-Q=当前积压, Send-Q=队列上限
ss -lnt
# State   Recv-Q  Send-Q  Local Address:Port
# LISTEN  0       511     0.0.0.0:8080          <- 上限 511 (nginx 默认 backlog)
```

对 LISTEN socket，`ss` 的 `Recv-Q` 是**当前全连接队列中已完成但未被 accept 的连接数**，`Send-Q` 是队列上限。`Recv-Q` 持续逼近 `Send-Q` 说明应用 accept 不过来。

**溢出行为**：全连接队列满时，新完成握手的连接怎么处理由 `net.ipv4.tcp_abort_on_overflow` 决定：

- `=0`（默认）：内核**丢弃对端的 ACK**，装作没收到，触发对端重传 ACK（给应用争取时间 accept）。客户端表现为偶发延迟。
- `=1`：直接发 RST，客户端立刻报 connection reset。一般不建议开。

**怎么确认溢出**（关键命令）：

```bash
# 这两个计数持续增长 = 全连接队列在溢出
nstat -az | grep -i -E 'ListenOverflows|ListenDrops'
# 或
netstat -s | grep -i -E 'listen|SYNs to LISTEN'
#   xxx times the listen queue of a socket overflowed
#   xxx SYNs to LISTEN sockets dropped
```

**修法**：① 调大应用 `listen(backlog)` 同时调大 `net.core.somaxconn`（二者取小）；② 优化应用 accept 速度 / 增加 worker；③ 必要时调大 `tcp_max_syn_backlog`。

> backlog 参数从哪来、`SO_REUSEPORT` 如何让多 worker 各自拥有独立 accept 队列从而消除惊群，见 [L13](./L13-精通-Socket-与连接管理.md) 第三章。

---

## 第三章 四次挥手与 TIME_WAIT

### 3.1 为什么需要 TIME_WAIT

**主动关闭方**在发完最后一个 ACK 后进入 `TIME_WAIT`，停留 **2MSL**（Maximum Segment Lifetime 的两倍）后才真正释放。两个目的：

1. **保证最后的 ACK 能到达对端**：若 ACK 丢失，对端会重传 FIN，本端还在 TIME_WAIT 才能再回 ACK；
2. **让本次连接的旧报文在网络中消亡**：避免延迟到达的旧报文串到使用相同四元组的新连接里。

Linux 的 MSL 实现为固定 60 秒（`TCP_TIMEWAIT_LEN`，编译期常量），即 TIME_WAIT 约 60s，**不可通过 sysctl 修改**（`tcp_fin_timeout` 调的是 `FIN_WAIT_2` 的超时，不是 TIME_WAIT，这是常见误解）。

### 3.2 大量 TIME_WAIT 的两种场景与正确处理

| 场景 | 现象 | 正确做法 |
|---|---|---|
| 服务端主动关闭短连接 | 服务端堆积 TIME_WAIT | 尽量让**客户端主动关闭**；或改用长连接/连接池 |
| 客户端（如反代回源）发起大量短连接 | 客户端本地端口耗尽，报 `Cannot assign requested address` | `net.ipv4.tcp_tw_reuse=1` + 扩大 `ip_local_port_range`；用连接池 |

相关参数：

| 参数 | 含义 / 建议 |
|---|---|
| `net.ipv4.tcp_tw_reuse` | 允许将处于 TIME_WAIT 的连接**用于新的对外连接**（仅 outbound，依赖 TCP timestamps）。客户端/回源场景安全有效 |
| `net.ipv4.tcp_max_tw_buckets` | TIME_WAIT 最大数量，超出直接拆除并告警。是「兜底阀门」，不是越大越好 |
| `net.ipv4.ip_local_port_range` | 本地临时端口范围，短连接客户端要调大，如 `1024 65535` |
| ~~`net.ipv4.tcp_tw_recycle`~~ | **已删除（4.12+）**。它在 NAT 环境会错误丢弃连接，是历史上著名的坑，切勿再用 |

> ⚠️ 网上大量「TCP 优化」文章仍在教 `tcp_tw_recycle=1`。该参数在 4.12 内核已彻底移除；在更早内核上开启会导致 NAT 后的客户端**随机连不上**。见到即应删除。

```bash
# 客户端/回源机：缓解端口耗尽
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

---

## 第四章 拥塞控制：cubic、BBR、BBRv3

### 4.1 经典模型回顾

TCP 用拥塞窗口 `cwnd` 限制在途未确认数据量。经典流程：**慢启动**（cwnd 指数增长直到 `ssthresh`）→ **拥塞避免**（线性增长）→ 丢包时**快重传/快恢复**（cwnd 减半）。这类「基于丢包」的算法（Reno、**cubic**）假设丢包=拥塞。

**cubic**（Linux 默认）用三次函数控制窗口增长，在高带宽长肥管道（high BDP）上比 Reno 更激进、收敛更好，是多数发行版默认值。

### 4.2 BBR：基于带宽和 RTT 建模

**BBR**（Bottleneck Bandwidth and RTT）不靠丢包判断拥塞，而是主动探测瓶颈带宽和最小 RTT，据此调节发送速率。优势在**有随机丢包但带宽充足**的链路（跨地域、弱网、有 buffer bloat 的链路）上显著优于 cubic——这正是「带宽够却跑不满」的典型解。

```bash
# 查看可用与当前算法
sysctl net.ipv4.tcp_available_congestion_control
#   reno cubic bbr
sysctl net.ipv4.tcp_congestion_control
#   cubic

# 启用 BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr
# 持久化
echo 'net.ipv4.tcp_congestion_control=bbr' >> /etc/sysctl.d/99-tcp.conf
```

> 早期 BBR(v1) 建议配合 `fq` qdisc（`net.core.default_qdisc=fq`）以获得 pacing；较新内核 TCP 层已内建 pacing，对 qdisc 依赖减弱。**BBRv3** 在公平性与丢包响应上持续改进，2026 年仍在推进与逐步落地。

| 算法 | 拥塞信号 | 适用 | 风险 |
|---|---|---|---|
| cubic | 丢包 | 通用、数据中心内部 | 弱网/高丢包链路吞吐差 |
| BBR | 带宽+RTT 建模 | 跨地域、弱网、buffer bloat | 与 cubic 共存时的公平性需关注，建议同链路统一 |

### 4.3 选型建议

- 数据中心**内部**低延迟低丢包：cubic 足够。
- **跨地域 / 公网 / CDN 回源 / 弱网**：BBR 通常带来肉眼可见的吞吐提升。
- 切换是**发送方**行为，只需在发送端（如服务端出向、客户端）配置即可生效。

---

## 第五章 收发缓冲区与窗口

### 5.1 BDP 与缓冲区上限

单条 TCP 的最大吞吐受 `窗口 / RTT` 限制。要跑满带宽，窗口需 ≥ **BDP（带宽时延积）= 带宽 × RTT**。例如 1Gbps × 100ms ≈ 12.5MB。若接收窗口只有 256KB，单流吞吐被卡在 `256KB/100ms ≈ 2.5MB/s`——这就是「带宽够、单流慢」的根因之一。

### 5.2 关键参数

| 参数 | 含义 |
|---|---|
| `net.ipv4.tcp_rmem` | 接收缓冲 `min default max`（字节），内核按需在 min~max 间自动调整 |
| `net.ipv4.tcp_wmem` | 发送缓冲 `min default max` |
| `net.core.rmem_max` / `wmem_max` | `SO_RCVBUF`/`SO_SNDBUF` 能设到的上限 |
| `net.ipv4.tcp_moderate_rcvbuf` | 接收缓冲自动调整开关（默认 1） |
| `net.ipv4.tcp_window_scaling` | 窗口缩放选项（默认 1），突破 64KB 窗口限制，长肥管道必需 |

```bash
# 长肥管道单流吞吐上不去：放开缓冲上限，让自动调整有空间
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.ipv4.tcp_rmem="4096 131072 16777216"
sysctl -w net.ipv4.tcp_wmem="4096 16384 16777216"
```

> ⚠️ 不要用 `setsockopt(SO_RCVBUF)` 手动写死接收缓冲——那会**关闭自动调整**（`tcp_moderate_rcvbuf` 失效），反而常常设得过小。除非你非常清楚 BDP，否则交给内核自动调。

---

## 第六章 Nagle 与 delayed ACK：40ms 延迟之谜

### 6.1 两个「为你好」的优化撞在一起

- **Nagle 算法**：未被确认的小包存在时，攒着小数据不立即发，等凑成大包或收到 ACK 再发——减少小包数量。
- **Delayed ACK**：收到数据不立刻回 ACK，等最多 ~40ms（或有反向数据可捎带）再回——减少纯 ACK 包。

当一端开 Nagle、另一端 delayed ACK，且应用是「写一个小请求→等响应」的模式时，会互相等待：发送方因 Nagle 等 ACK 才发下一个小包，接收方因 delayed ACK 攒着不回——直到 40ms 超时。表现为**请求级偶发 40ms 延迟**。

### 6.2 修法

```c
int one = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof(one)); // 关 Nagle
```

绝大多数 RPC / 交互式服务（Redis、gRPC、数据库驱动）都应设 `TCP_NODELAY`。`TCP_QUICKACK` 可临时关闭 delayed ACK，但它不是持久的（内核会重置），通常优先在发送侧关 Nagle。

---

## 第七章 高并发服务器 sysctl 调优清单

> 原则：**先定位再调参**。下面每个参数都标注含义与风险，按场景取用，不要整段照抄。所有参数名均真实存在，可 `sysctl <name>` 验证。

### 7.1 连接队列与握手

```bash
# 全连接队列上限（需配合应用 listen backlog）
net.core.somaxconn = 1024
# 半连接队列上限
net.ipv4.tcp_max_syn_backlog = 8192
# 网卡收包到协议栈前的积压队列（软中断处理不过来时的缓冲，呼应 L11）
net.core.netdev_max_backlog = 16384
# SYN flood 防护（默认开，保留）
net.ipv4.tcp_syncookies = 1
```

| 参数 | 风险 / 注意 |
|---|---|
| `somaxconn` | 调大需应用同步调大 backlog 才生效；过大只是延后暴露 accept 慢 |
| `netdev_max_backlog` | 与软中断/RPS 配合，单纯调大不解决单核打满（见 L11） |

### 7.2 TIME_WAIT 与端口（客户端/回源侧）

```bash
net.ipv4.tcp_tw_reuse = 1                  # 仅 outbound 复用，安全
net.ipv4.ip_local_port_range = 1024 65535  # 短连接客户端扩端口
net.ipv4.tcp_max_tw_buckets = 262144       # 兜底阀门
# 注意：不要设置 tcp_tw_recycle —— 已删除
```

### 7.3 缓冲区（长肥管道 / 大吞吐）

```bash
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 131072 16777216
net.ipv4.tcp_wmem = 4096 16384 16777216
```

### 7.4 拥塞控制与 keepalive

```bash
net.ipv4.tcp_congestion_control = bbr      # 跨地域/弱网；内网可留 cubic
net.ipv4.tcp_keepalive_time = 600          # 空闲多久开始探活
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_fin_timeout = 30              # FIN_WAIT_2 超时（非 TIME_WAIT！）
```

应用层面记得：长连接服务务必设 SO_KEEPALIVE 或应用层心跳，否则中间 NAT/防火墙会静默丢弃空闲连接。

---

## 第八章 重传、SACK 与零拷贝

### 8.1 丢包恢复：RTO、快速重传与 SACK

- **RTO（重传超时）**：基于 RTT 估计（`srtt` + 方差）动态计算，超时未收到 ACK 即重传并指数退避。RTO 触发代价高（通常意味着一段时间无数据流动）。
- **快速重传**：收到 3 个重复 ACK 即重传，不等 RTO，恢复更快。
- **SACK（选择性确认，`net.ipv4.tcp_sack`）**：接收方告诉发送方「我收到了哪些不连续的段」，发送方只重传真正丢的，避免回退 N 重传一整片。默认开启，长肥管道尤其关键。

```bash
ss -ti                          # 单连接的 rto/rtt/retrans/sacked 等内核 TCP 指标（呼应 L13）
nstat -az | grep -i retrans     # 全局重传计数
```

重传率持续偏高 → 链路丢包或拥塞；结合 `tcpretrans`（bcc，见 [L19](./L19-精通-eBPF.md)）定位是哪些连接在重传。

### 8.2 零拷贝发送

减少「内核↔用户态」数据拷贝可大幅提升吞吐：

| 机制 | 说明 |
|---|---|
| `sendfile()` | 文件→socket 在内核内直接搬运，省两次拷贝（Nginx 静态文件、Kafka 日志发送的基石，呼应 [L07](./L07-精通-VFS-与文件系统.md)） |
| `MSG_ZEROCOPY`（`SO_ZEROCOPY`） | 大块 `send()` 零拷贝，发完经 error queue 异步通知；适合大包高吞吐 |
| `splice()` | 在两个 fd 间经管道零拷贝搬运 |

零拷贝省的是 CPU 与内存带宽：对小包收益有限，对大吞吐（文件服务、消息队列）收益显著。

---

## 生产实践

**案例：压测 QPS 卡在某个值，客户端大量 connection timeout。**

1. 服务端 `ss -lnt` 看到 `Recv-Q` 持续等于 `Send-Q`（如 128）——全连接队列满。
2. `nstat -az | grep ListenOverflows` 计数飞涨——确认溢出。
3. 根因：nginx/应用 `backlog` 默认偏小，且 `net.core.somaxconn` 也小，二者取小被截断；同时 worker accept 速度跟不上。
4. 处置：应用 `listen(backlog=1024)` + `sysctl net.core.somaxconn=1024`，并增加 worker；复测 `Recv-Q` 回落、`ListenOverflows` 不再增长。

**案例：跨区域备份单条 TCP 只跑 20MB/s，带宽明明 1Gbps。**

1. 估算 BDP = 1Gbps × 80ms ≈ 10MB；当前 `tcp_rmem` max 仅 6MB，窗口不够。
2. 放开 `net.core.rmem_max`/`tcp_rmem` 上限，确认 `tcp_window_scaling=1`。
3. 发送端切 BBR：`tcp_congestion_control=bbr`。
4. 单流吞吐提升至接近线速。

---

## 陷阱清单

1. **现象**：`backlog` 设很大但全连接队列还是小 → **原因**：被 `net.core.somaxconn` 截断（取 min）→ **修法**：同步调大 `somaxconn` 并确认应用真的传了大 backlog。
2. **现象**：照搬「优化脚本」后 NAT 后的部分客户端随机连不上 → **原因**：脚本里有 `tcp_tw_recycle=1`（旧内核）→ **修法**：删除该项；新内核该参数已不存在。
3. **现象**：调大 `tcp_fin_timeout` 想减少 TIME_WAIT，无效 → **原因**：TIME_WAIT 时长是固定 60s 的 2MSL，`tcp_fin_timeout` 管的是 FIN_WAIT_2 → **修法**：用 `tcp_tw_reuse`、连接池、让客户端主动关闭。
4. **现象**：RPC 偶发稳定 40ms 延迟 → **原因**：Nagle × delayed ACK 互等 → **修法**：`TCP_NODELAY`。
5. **现象**：`setsockopt(SO_RCVBUF)` 设了大缓冲反而吞吐更差 → **原因**：手动设置关闭了接收缓冲自动调整，且值偏小 → **修法**：去掉手动设置，调 `tcp_rmem` 上限交给内核。
6. **现象**：大量 `CLOSE-WAIT` 堆积、fd 泄漏 → **原因**：对端已关、本端应用没 `close()` → **修法**：修应用（这是代码 bug，非内核调参能解）。
7. **现象**：开了 BBR 但单机多服务吞吐不公平 → **原因**：同链路 cubic/BBR 混用公平性问题 → **修法**：同链路统一算法，关注 BBRv3 进展。
8. **现象**：`SYN-RECV` 暴增、正常连接被拒 → **原因**：SYN flood 或半连接队列小 → **修法**：确认 `tcp_syncookies=1`，调 `tcp_max_syn_backlog`，上游加防护。

---

## 2026 现状

- **cubic 仍是默认**，BBR 在公网/跨地域大流量场景广泛启用，**BBRv3** 在公平性与丢包响应上持续改进、逐步推广。
- `tcp_tw_recycle` 自 4.12 删除已多年，**任何引用它的教程都已过时**。
- 接收缓冲自动调整（`tcp_moderate_rcvbuf`）成熟，绝大多数场景**不应**手动写死 `SO_RCVBUF`。
- TCP Fast Open（`tcp_fastopen`）受中间盒兼容性限制，收益场景有限，需实测。
- eBPF 让 TCP 可观测性大幅增强：`tcpconnect`/`tcplife`/`tcpretrans`（bcc）可无侵入追踪建连、连接寿命与重传，详见 [L13](./L13-精通-Socket-与连接管理.md) 与 [L19 eBPF](./L19-精通-eBPF.md)。

---

## 练习题

1. （⭐）半连接队列与全连接队列分别在 TCP 握手的哪一步起作用？各自的上限由哪些参数决定？
2. （⭐）为什么主动关闭方会进入 TIME_WAIT？停留多久？这个时长能用 `tcp_fin_timeout` 改吗？
3. （⭐⭐）`ss -lnt` 输出中，LISTEN 行的 `Recv-Q` 和 `Send-Q` 各代表什么？怎样据此判断 accept 队列溢出？
4. （⭐⭐）解释 `tcp_abort_on_overflow=0` 与 `=1` 时，全连接队列满后客户端分别看到什么现象。
5. （⭐⭐）BBR 与 cubic 判断拥塞的依据有何根本不同？什么链路上 BBR 优势最明显？
6. （⭐⭐⭐）一条跨地域 TCP 带宽跑不满，给出从 BDP 估算到缓冲/窗口/拥塞算法的完整排查与调优步骤。
7. （⭐⭐⭐）线上出现成片 40ms 延迟的 RPC，抓包该看什么特征？根因与修法是什么？
8. （⭐⭐⭐）你接手一台机器，`/etc/sysctl.conf` 里有 `net.ipv4.tcp_tw_recycle=1`。在 6.x 内核上它会怎样？为什么必须删除？历史上它在 NAT 环境引发过什么问题？

---

## 参考答案

1. 半连接队列（SYN queue）在收到对端 SYN、回 SYN+ACK 之后起作用——内核为该半开连接创建 `request_sock` 放入此队列，状态 SYN_RECV，等待对端 ACK；其上限由 `net.ipv4.tcp_max_syn_backlog` 控制（满时可由 `tcp_syncookies` 兜底）。全连接队列（accept queue）在收到对端 ACK、握手完成之后起作用——连接从半连接队列移入全连接队列，状态 ESTABLISHED，等应用 `accept()` 取走；其上限 = `min(listen() 的 backlog 参数, net.core.somaxconn)`。

2. 主动关闭方在发出最后一个 ACK 后进入 TIME_WAIT，目的有二：① 保证最后的 ACK 能到达对端——若 ACK 丢失，对端会重传 FIN，本端还在 TIME_WAIT 才能再回 ACK；② 让本次连接的旧报文在网络中消亡，避免延迟到达的旧报文串进使用相同四元组的新连接。停留 2MSL，Linux 实现为固定约 60 秒（`TCP_TIMEWAIT_LEN` 编译期常量）。**不能用 `tcp_fin_timeout` 改**——`tcp_fin_timeout` 控制的是 FIN_WAIT_2 状态的超时，不是 TIME_WAIT，这是常见误解。

3. 对 LISTEN socket：`Recv-Q` 是当前全连接队列中已完成握手但尚未被应用 accept 的连接数（当前积压），`Send-Q` 是全连接队列的上限。判断溢出：`Recv-Q` 持续逼近或等于 `Send-Q` 说明应用 accept 不过来、队列将满；更确切的确认是看计数器 `nstat -az | grep -i ListenOverflows`（或 netstat -s 里 "times the listen queue of a socket overflowed"），持续增长即全连接队列在溢出。

4. 全连接队列满后新完成握手的连接的处理由 `tcp_abort_on_overflow` 决定：`=0`（默认）内核**丢弃对端的 ACK**、装作没收到，触发对端重传 ACK（给应用争取 accept 时间），客户端表现为**偶发延迟**（连接建立变慢但最终可能成功）；`=1` 内核直接发 **RST**，客户端立刻收到 **connection reset**。一般不建议设 1。

5. cubic 是"基于丢包"的算法，假设丢包=拥塞，用三次函数控制 cwnd 增长，丢包时减小窗口。BBR 不靠丢包判断，而是主动探测**瓶颈带宽（Bottleneck Bandwidth）和最小 RTT** 来建模，据此调节发送速率。根本不同：cubic 以丢包为拥塞信号，BBR 以带宽+RTT 测量为信号。BBR 优势最明显的链路：**有随机丢包但带宽充足**的链路——跨地域、公网、CDN 回源、弱网、有 bufferbloat 的链路；这类链路上 cubic 会把随机丢包误判为拥塞而压低窗口，BBR 则能跑满带宽。

6. 步骤：① 估算 BDP = 带宽 × RTT（如 1Gbps × 80ms ≈ 10MB），与当前缓冲对比；② 检查并放开缓冲上限——`net.core.rmem_max`/`wmem_max` 和 `net.ipv4.tcp_rmem`/`tcp_wmem` 的 max 要 ≥ BDP，让自动调整有空间（如 `tcp_rmem="4096 131072 16777216"`）；③ 确认 `net.ipv4.tcp_window_scaling=1`（长肥管道突破 64KB 窗口必需）；④ 不要用 `setsockopt(SO_RCVBUF)` 手动写死缓冲（会关闭自动调整且常设得偏小），交给内核自动调；⑤ 发送端切 BBR `sysctl -w net.ipv4.tcp_congestion_control=bbr`（跨地域弱网优于 cubic）；⑥ 复测单流吞吐，并用 `ss -ti` 看实际窗口/rtt/retrans 验证。

7. 抓包特征：请求级偶发稳定 ~40ms 延迟，表现为"发完一个小请求后，要等约 40ms 才收到对端的 ACK 或响应"，发送方因有未确认的小包而不发下一个、接收方迟迟不回 ACK。根因：**Nagle 算法与 delayed ACK 互等**——一端开 Nagle（有未确认小包时攒着不发，等 ACK 或凑大包），另一端 delayed ACK（收到数据不立即回 ACK，最多等约 40ms 或等捎带），在"小请求→等响应"模式下互相等待，直到 delayed ACK 的 40ms 超时。修法：在交互式/RPC 连接上设 `setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, ...)` 关闭 Nagle（Redis/gRPC/数据库驱动等都应设）。

8. 在 6.x 内核上，`net.ipv4.tcp_tw_recycle` 这个参数**已不存在**（4.12 起从内核移除），设置它会被忽略（`sysctl` 报该 key 不存在/无效），不起任何作用。必须删除是因为：它是过时且危险的配置，留着会误导后人以为还能用、并掩盖真正该用的手段（`tcp_tw_reuse`、连接池、让客户端主动关闭）。历史问题：`tcp_tw_recycle=1` 会基于源 IP 的 TCP timestamp 做 per-host 的"快速回收"判断，但在 **NAT 环境**下，NAT 后多个客户端共享同一公网 IP 而各自 timestamp 不同步/乱序，内核会把后到客户端的、timestamp"看似回退"的 SYN 当作旧报文丢弃，导致 NAT 后的部分客户端**随机连不上**——这是著名的坑。正确替代是用 `tcp_tw_reuse`（仅 outbound、依赖 timestamps、对 NAT 安全）。
