# 精通 Socket 与连接管理：struct sock、SO_REUSEPORT、accept 队列、惊群、conntrack、排障

> 课程编号：L13
> 路线图来源：Linux · 模块四 网络
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：SO_REUSEPORT、EPOLLEXCLUSIVE、conntrack、ss/bcc）

---

## 引言：listen 的 backlog 到底是什么

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd, ...);
listen(fd, 511);   // 这个 511 到底控制了什么？
```

很多人以为 `listen` 的 backlog 是「最大连接数」。它其实只控制**全连接队列（accept queue）的上限**，且真正生效的是 `min(backlog, net.core.somaxconn)`。理解这一点，要先把 socket 在内核里的样子看清楚：它既是一个文件（有 fd、能 epoll），又是一个网络协议控制块。本篇沿着 `socket/bind/listen/accept/connect` 的内核路径走一遍，讲清 `SO_REUSEPORT` 如何用「内核级负载均衡」消除 accept 惊群，以及 conntrack 表满这一经典线上事故的成因与排查。

握手两个队列的细节见 [L12](./L12-精通-TCP-IP-内核实现与调优.md) 第二章；epoll/惊群与事件循环见 [L08 I/O 多路复用](./L08-精通-IO-多路复用.md)；网络命名空间见 [L16 Namespace](./L16-精通-Namespace.md)。

---

## 第一章 socket 在内核里的双重身份

### 1.1 struct socket 与 struct sock

一个 socket 在内核里是**两个结构体**协作：

| 结构 | 层次 | 职责 |
|---|---|---|
| `struct socket` | BSD socket 层 | 面向 VFS：关联 `struct file`/fd，持有 `proto_ops`（`inet_stream_ops` 等），有 `state` |
| `struct sock` | 网络协议层 | 真正的协议状态：收发队列、`sk_state`、内存计量、定时器；TCP 用 `tcp_sock` 扩展 |

```
fd ──► struct file ──► struct socket ──► struct sock (── tcp_sock)
        (f_op =                (ops =          (sk_receive_queue,
         socket_file_ops)       inet_stream_ops) sk_write_queue, sk_state...)
```

「**一切皆文件**」在这里体现：socket 有自己的 `file_operations`（`socket_file_ops`），所以能 `read`/`write`/`close`/被 `epoll` 监听。但它没有真正的 inode 对应磁盘文件，VFS 抽象详见 [L07 VFS](./L07-精通-VFS-与文件系统.md)。

### 1.2 为什么 socket 能被 epoll 监听

因为 `struct file` 的 `f_op->poll`（现代为 `->poll` / EPOLLET 机制）由 socket 层实现，epoll 通过它注册回调到 `sk->sk_wq` 等待队列。当数据到达、`sk_data_ready` 被调用时唤醒等待者——这正是 [L08](./L08-精通-IO-多路复用.md) 里 epoll 就绪通知的底层来源。

### 1.3 socket 的内存计量

每个 `struct sock` 有收发缓冲与已用计量：`sk_rcvbuf`/`sk_sndbuf`（上限）、`sk_rmem_alloc`/`sk_wmem_alloc`（已用）。`ss -tm` 可看：

```bash
ss -tm
# skmem:(r0,rb131072,t0,tb46080,...)  # rb=接收缓冲上限,r=已用；tb/t 同理发送
```

接收缓冲上限受 `SO_RCVBUF` 与 `net.ipv4.tcp_rmem` 共同影响（见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）；海量连接时这些缓冲会计入内存，甚至撞 cgroup `memory.max`（见 [L17](./L17-精通-Cgroup-v2.md)）。

### 1.4 Unix domain socket（本机 IPC）

`AF_UNIX` socket 用于**同机进程间通信**（Nginx↔php-fpm、容器↔sidecar、`/var/run/docker.sock`），不走网络栈，比 TCP 回环更快：

- 命名：文件系统路径，或 **abstract**（名字以 `\0` 开头，不落地文件，随进程消亡）；
- **`SCM_RIGHTS`**：Unix socket 能通过 `sendmsg` 辅助数据**把文件描述符传给另一进程**——这是 socket activation（[L20](./L20-精通-systemd-与启动流程.md)）与特权分离的关键能力；
- `ss -x` 查看 Unix socket。

---

## 第二章 socket API 的内核路径

### 2.1 五个系统调用，内核各做了什么

```
socket()  → sock_create(): 按 family/type 分配 struct socket + struct sock，
                            sock_map_fd() 关联到一个 fd
bind()    → inet_bind():   把本地 IP:port 写入 sk，检查端口占用/权限(<1024 需 CAP_NET_BIND_SERVICE)
listen()  → inet_listen(): 状态置 LISTEN，初始化 icsk_accept_queue，
                           队列上限 = min(backlog, somaxconn)
accept()  → inet_csk_accept(): 从全连接队列摘一个已完成连接，新建一个 fd 返回；
                           队列空且阻塞则睡眠
connect() → tcp_v4_connect(): 选源端口，发 SYN，进入 SYN_SENT
```

### 2.2 listen 与 backlog 截断（再次强调）

```bash
# 内核全局上限
sysctl net.core.somaxconn          # 例如 4096
```

应用即便 `listen(fd, 65535)`，实际队列上限也是 `min(65535, somaxconn)`。要扩大队列，**两者都要调**。`ss -lnt` 的 `Send-Q` 列就是这个生效后的上限：

```bash
ss -lnt
# State   Recv-Q  Send-Q  Local Address:Port
# LISTEN  0       4096    *:8080         # Send-Q=生效上限, Recv-Q=当前积压
```

### 2.3 端口分配与耗尽

`connect()` 不显式 bind 时，内核从 `net.ipv4.ip_local_port_range` 选临时端口。大量短连接 + TIME_WAIT 占用会耗尽端口，报 `EADDRNOTAVAIL`（Cannot assign requested address）。处理见 [L12](./L12-精通-TCP-IP-内核实现与调优.md) 第三章（`tcp_tw_reuse` + 扩端口范围 + 连接池）。

### 2.4 影响建连行为的几个选项

| 选项 | 作用 |
|---|---|
| `TCP_FASTOPEN` | 允许在 SYN 里带数据省一个 RTT（受中间盒兼容性限制，见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)） |
| `TCP_DEFER_ACCEPT` | 直到有数据到达才把连接交给 `accept()`（过滤空连接、省唤醒） |
| `SO_LINGER` | 控制 `close()` 时是否等数据发完；设 linger=0 会直接发 RST、跳过 TIME_WAIT——危险手法，慎用 |

### 2.5 一个最小 TCP 服务端骨架

```c
int lfd = socket(AF_INET, SOCK_STREAM, 0);          // → struct socket + struct sock
int one = 1;
setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
struct sockaddr_in addr = { .sin_family = AF_INET,
    .sin_port = htons(8080), .sin_addr.s_addr = INADDR_ANY };
bind(lfd, (struct sockaddr *)&addr, sizeof(addr));  // → inet_bind 写入本地端口
listen(lfd, 1024);                                  // → 初始化 accept 队列 min(1024, somaxconn)
for (;;) {
    int cfd = accept(lfd, NULL, NULL);              // → 从全连接队列摘一个，返回新 fd
    handle(cfd);
    close(cfd);                                     // 不 close 会 fd 泄漏 + CLOSE_WAIT
}
```

每一行都对应第二章讲的内核动作。生产服务不会「每连接一线程」，而是把 `accept` 的 fd 设非阻塞、交给 epoll/io_uring 事件循环（[L08](./L08-精通-IO-多路复用.md)/[L09](./L09-精通-io_uring-与异步IO.md)），并用 `SO_REUSEPORT` 起多 worker（见 3.2）。

---

## 第三章 SO_REUSEADDR 与 SO_REUSEPORT

这两个选项名字像，作用完全不同，是高频混淆点。

### 3.1 SO_REUSEADDR

```c
int one = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
```

主要作用：

- **允许 bind 处于 TIME_WAIT 的本地地址**——服务重启时不必等 2MSL，解决「restart 后 bind: Address already in use」。
- 允许多个 socket bind 到**不同 IP** 的相同端口（如分别 bind 到具体网卡 IP）。

几乎所有服务器都应在 bind 前设它。

### 3.2 SO_REUSEPORT（3.9+）：内核级负载均衡

```c
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &one, sizeof(one));
```

允许**多个 socket bind 到完全相同的 IP:port**。内核为这组 socket 建立一个 reuseport group，新连接到来时按四元组 hash **选一个 socket** 投递。关键收益：

- **每个 worker 进程拥有自己独立的 LISTEN socket 和独立的 accept 队列**；
- 新连接被内核均衡到各 worker，**从根本上消除 accept 惊群**；
- worker 崩溃重启可平滑接管（配合 group 管理）。

```
        ┌─────────── 内核 reuseport group (:8080) ───────────┐
        │  四元组 hash 选一个 socket                           │
        └──┬───────────────┬───────────────┬─────────────────┘
           ▼               ▼               ▼
      worker0 listen   worker1 listen   worker2 listen
      独立 accept 队列  独立 accept 队列  独立 accept 队列
```

Nginx（`reuseport`）、多数 Go/Rust 高并发服务都用它来做多进程/多 listener 负载均衡。

**进阶：自定义 reuseport 选择逻辑**。默认按四元组 hash 选 socket，但可用 `SO_ATTACH_REUSEPORT_CBPF`/`SO_ATTACH_REUSEPORT_EBPF` 挂一段 BPF 程序自定义「连接分给哪个 socket」——如按收包 CPU 把连接交给同 NUMA 的 worker（呼应 [L11](./L11-精通-Linux-网络协议栈.md) 多队列与 [L19](./L19-精通-eBPF.md)），减少跨核缓存抖动。

| 选项 | 核心用途 | 典型场景 |
|---|---|---|
| `SO_REUSEADDR` | 复用 TIME_WAIT 地址、不同 IP 同端口 | 服务重启、绑定具体网卡 |
| `SO_REUSEPORT` | 多 socket 同 addr:port + 内核负载均衡 | 多 worker 消除惊群 |

---

## 第四章 accept 惊群与连接建立全链路

### 4.1 什么是惊群

多个进程/线程在**同一个 LISTEN fd** 上 `accept()`（或都用 epoll 监听同一 fd），一个连接到来时若唤醒了所有等待者，只有一个能 accept 成功，其余空转——这就是 accept 惊群，浪费 CPU。

**两类惊群与现代解法**：

| 类型 | 现状 / 解法 |
|---|---|
| 多进程直接 `accept()` 同一 fd | 现代内核对 accept 等待队列用独占唤醒（`WQ_FLAG_EXCLUSIVE`），基本只唤醒一个，惊群已大幅缓解 |
| 多个 epoll 监听同一 fd | 用 `EPOLLEXCLUSIVE`（4.5+）让内核只唤醒一个 epoll 实例 |
| 彻底方案 | `SO_REUSEPORT`：各 worker 独立 listen fd + 独立队列，内核负载均衡，无惊群 |

epoll 惊群与 `EPOLLEXCLUSIVE` 的细节见 [L08](./L08-精通-IO-多路复用.md)；锁/等待队列的独占唤醒机制见 [L15 内核同步](./L15-精通-内核同步与futex.md) 的惊群章节。

### 4.2 连接建立全链路（串起 L11/L12）

```
client connect() ─SYN─►  网卡→软中断→协议栈(L11)
                         server 入半连接队列(SYN_RECV)，回 SYN+ACK
client ─ACK─►            握手完成，移入全连接队列(ESTABLISHED)
server accept()          从全连接队列摘下，返回新 fd
                         worker 用 epoll 监听新 fd，进入读写循环
```

---

## 第五章 conntrack：连接跟踪与表满事故

### 5.1 conntrack 是什么

Netfilter 的连接跟踪（conntrack）为每条流维护一个表项，记录五元组与状态。**NAT、有状态防火墙（`-m state`/`ctstate`）、Kubernetes Service（kube-proxy iptables 模式）都依赖它**。每条连接占一个表项。

```bash
# 当前条目数与上限
cat /proc/sys/net/netfilter/nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max
# 实时查看（需 conntrack-tools）
conntrack -L
conntrack -S      # 各 CPU 统计，含 insert_failed / drop
```

### 5.2 表满事故

当 `nf_conntrack_count` 触及 `nf_conntrack_max`，新连接的包被丢弃，内核日志报：

```
nf_conntrack: table full, dropping packet
```

表现为：高并发网关/NAT 节点偶发丢包、新连接超时，但 CPU/带宽都不高。这是**容器/网关节点**的经典坑（K8s 节点尤甚，呼应 [cloud-native C03](../cloud-native/C03-精通-K8s-网络与-Service.md)）。

**修法**：

```bash
# 调大上限（注意内存：每条目约数百字节）
sysctl -w net.netfilter.nf_conntrack_max=1048576
# 相应调大 hash 桶（一般设为 max/4 ~ max/8）
sysctl -w net.netfilter.nf_conntrack_buckets=262144
# 缩短无用状态的超时，加速回收（按需）
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

> 另一种思路：对确实不需要跟踪的大流量（如纯转发、LVS DR），用 iptables `NOTRACK`（raw 表）跳过 conntrack，从源头减负。

### 5.3 conntrack 的状态与超时

每个表项有状态（`NEW`/`ESTABLISHED`/`RELATED`/…）和**超时**。established TCP 默认超时很长（默认为天级），这正是表项「堆积」的主因——大量早已断开、内核还没回收的连接占着表项：

```bash
sysctl net.netfilter.nf_conntrack_tcp_timeout_established   # 默认很大（天级）
sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait
conntrack -L -p tcp --state ESTABLISHED | wc -l             # 数 established 表项
```

NAT（SNAT/DNAT/MASQUERADE）完全建立在 conntrack 之上：没有连接跟踪，回包就无法被反向转换——这也是它对每条流都要占表项的原因。

---

## 第六章 排障工具

### 6.1 ss——比 netstat 快且信息全

```bash
ss -tanp                 # 所有 TCP，含进程(-p)
ss -tn state established # 仅 ESTABLISHED
ss -lnt                  # LISTEN，看 Recv-Q/Send-Q(队列积压/上限)
ss -ti                   # -i 显示 cwnd/rtt/retrans 等内核 TCP 内部信息
ss -tm                   # -m 显示 socket 内存占用
```

`ss -ti` 能直接看到单条连接的 `cwnd`、`rtt`、`retrans`，定位「这条连接为什么慢」非常有用（呼应 [L12](./L12-精通-TCP-IP-内核实现与调优.md) 的窗口/拥塞）。

### 6.2 tcpdump——抓包看真相

```bash
tcpdump -i eth0 -nn 'tcp port 8080 and (tcp[tcpflags] & tcp-syn != 0)'  # 只抓 SYN
tcpdump -i any -nn host 10.0.0.5 -w cap.pcap                            # 存盘用 wireshark 分析
```

### 6.3 eBPF/bcc——无侵入追踪（2026 主流）

```bash
tcpconnect      # 实时打印每个新建的出向连接(谁连了谁)
tcpaccept       # 实时打印每个被 accept 的入向连接
tcplife         # 每条连接的生命周期、收发字节、时长
tcpretrans      # 实时打印重传(定位丢包/弱网)
```

这些工具不改应用、不抓全量包，开销极低，是定位「偶发重传 / 连接寿命异常 / 谁在建连」的利器。原理见 [L19 eBPF](./L19-精通-eBPF.md)。

---

## 生产实践

**案例：K8s 节点偶发新连接超时，CPU/带宽都不高。**

1. `dmesg | grep conntrack` 看到 `table full, dropping packet`。
2. `cat /proc/sys/net/netfilter/nf_conntrack_count` 逼近 `nf_conntrack_max`。
3. 根因：节点承载大量短连接 + Service NAT，conntrack 表打满。
4. 处置：调大 `nf_conntrack_max`/`buckets`，缩短 `time_wait` 超时；长期评估 IPVS/eBPF（Cilium）替代 iptables 模式以减负。

**案例：Nginx 多 worker，CPU 利用不均、惊群空转。**

1. 改用 `listen 8080 reuseport;`（底层即 `SO_REUSEPORT`）。
2. 各 worker 独立 accept 队列，内核均衡，CPU 分布变均匀，惊群消失。

---

## 陷阱清单

1. **现象**：服务重启报 `Address already in use` → **原因**：上次连接处于 TIME_WAIT 占用地址 → **修法**：bind 前设 `SO_REUSEADDR`。
2. **现象**：以为 `SO_REUSEPORT` = `SO_REUSEADDR`，重启问题没解决 → **原因**：两者作用不同 → **修法**：重启用 REUSEADDR，多 worker 均衡用 REUSEPORT。
3. **现象**：`listen(backlog=65535)` 但队列还是很小 → **原因**：被 `somaxconn` 截断 → **修法**：同步调大 `net.core.somaxconn`（详见 [L12](./L12-精通-TCP-IP-内核实现与调优.md)）。
4. **现象**：网关偶发丢包、新连接超时但负载不高 → **原因**：conntrack 表满 → **修法**：调大 `nf_conntrack_max`/`buckets`，或 NOTRACK 跳过。
5. **现象**：多进程 accept 同一 fd CPU 空转 → **原因**：accept/epoll 惊群 → **修法**：`SO_REUSEPORT` 或 `EPOLLEXCLUSIVE`。
6. **现象**：`CLOSE-WAIT` 越积越多、fd 泄漏 → **原因**：对端已关、应用未 `close()` → **修法**：修应用，确保所有路径都 close（含异常路径）。
7. **现象**：用 `SO_REUSEPORT` 后偶有连接被「黑洞」 → **原因**：某 worker 崩溃但其 socket 仍在 group 中接收哈希到的连接 → **修法**：worker 退出时正确关闭 listen fd，使用支持平滑接管的框架。
8. **现象**：用 Unix socket 的服务报「too many open files」但 TCP 正常 → **原因**：传 fd（SCM_RIGHTS）或大量 Unix 连接耗尽 fd 上限 → **修法**：调 `ulimit -n`/`LimitNOFILE`，排查 fd 泄漏（见 [L08](./L08-精通-IO-多路复用.md)）。
9. **现象**：`SO_LINGER` 设 linger=0「优化」后偶发数据丢失 → **原因**：close 直接发 RST，未发完的数据被丢弃 → **修法**：除非明确要 RST，否则别这么用。

---

## 2026 现状

- `SO_REUSEPORT` 已是高并发服务多 worker 负载均衡的标准做法（Nginx/Envoy/Go 生态普遍使用）。
- accept 惊群在现代内核基本由独占唤醒解决；epoll 惊群用 `EPOLLEXCLUSIVE`。
- conntrack 表满仍是容器/网关节点高频事故；趋势是用 **eBPF（Cilium）/ IPVS** 替代 iptables 模式以减少 conntrack 压力（见 [cloud-native C03](../cloud-native/C03-精通-K8s-网络与-Service.md)）。
- `ss`、bcc/`bpftrace` 工具链成熟，`ss -ti` + `tcpretrans`/`tcplife` 是连接级排障首选。

---

## 练习题

1. （⭐）`struct socket` 与 `struct sock` 各自负责什么？为什么 socket 能被 epoll 监听？
2. （⭐）`SO_REUSEADDR` 和 `SO_REUSEPORT` 的区别？分别解决什么问题？
3. （⭐⭐）`listen(fd, backlog)` 的 backlog 实际生效值由什么决定？如何用 `ss -lnt` 观察？
4. （⭐⭐）画出从 `connect()` 到服务端 `accept()` 返回新 fd 的完整链路，标出半连接/全连接队列的位置。
5. （⭐⭐）`SO_REUSEPORT` 为什么能消除 accept 惊群？它和 `EPOLLEXCLUSIVE` 的适用场景有何不同？
6. （⭐⭐⭐）一台 NAT 网关偶发丢包，`dmesg` 有 `nf_conntrack: table full`。给出完整诊断与多种缓解手段（含治本思路）。
7. （⭐⭐⭐）服务进程 fd 持续增长，`ss` 显示大量 `CLOSE-WAIT`。定位思路是什么？为什么这通常不是内核调参能解决的？
8. （⭐⭐⭐）用哪些 eBPF/bcc 工具可以无侵入地回答：谁在向我建连、每条连接活了多久、是否有重传？分别说明。
9. （⭐⭐）Unix domain socket 相比 TCP 回环有何优势？`SCM_RIGHTS` 能做什么？abstract socket 与路径命名有何区别？
10. （⭐⭐⭐）conntrack 表项为什么会「堆积」？established 超时与表满有什么关系？如何安全地加速回收？

---

## 参考答案

1. `struct socket` 是 BSD socket 层，面向 VFS——关联 `struct file`/fd，持有 `proto_ops`（如 `inet_stream_ops`）和 `state`；`struct sock` 是网络协议层，保存真正的协议状态——收发队列（`sk_receive_queue`/`sk_write_queue`）、`sk_state`、内存计量、定时器（TCP 用 `tcp_sock` 扩展）。socket 能被 epoll 监听，是因为它的 `struct file` 有自己的 `file_operations`（`socket_file_ops`，体现"一切皆文件"），其 poll 实现由 socket 层提供：epoll 通过它把回调注册到 `sk->sk_wq` 等待队列，数据到达时 `sk_data_ready` 被调用并唤醒等待者，从而产生就绪通知。

2. `SO_REUSEADDR`：主要允许 bind 处于 TIME_WAIT 的本地地址（服务重启不必等 2MSL，解决"Address already in use"），也允许多个 socket bind 到不同 IP 的相同端口。`SO_REUSEPORT`（3.9+）：允许多个 socket bind 到**完全相同的 IP:port**，内核建立 reuseport group、按四元组哈希把新连接投递给其中一个 socket，每个 worker 拥有独立 LISTEN socket 和独立 accept 队列，实现内核级负载均衡并消除惊群。区别：REUSEADDR 解决重启/绑定问题，REUSEPORT 解决多 worker 负载均衡与惊群。

3. backlog 实际生效值 = `min(listen() 传入的 backlog, net.core.somaxconn)`——即便应用传 65535，也会被 `somaxconn` 截断，要扩大队列必须两者都调大。用 `ss -lnt` 观察：LISTEN 行的 `Send-Q` 列就是这个生效后的队列上限，`Recv-Q` 是当前积压（已完成握手未被 accept 的连接数）。

4. 链路：client `connect()` 发 SYN → 经网卡/软中断/协议栈（L11）到达 server，server 创建 `request_sock` 放入**半连接队列（SYN queue）**，状态 SYN_RECV，回 SYN+ACK → client 回 ACK → server 收到 ACK，握手完成，连接从半连接队列移入**全连接队列（accept queue）**，状态 ESTABLISHED → server `accept()`（`inet_csk_accept`）从全连接队列摘下一个连接，新建并返回一个 fd → worker 把新 fd 设非阻塞、交给 epoll 进入读写循环。半连接队列在 SYN_RECV 阶段，全连接队列在握手完成等待 accept 阶段。

5. `SO_REUSEPORT` 消除惊群是因为：它让每个 worker 拥有**自己独立的 LISTEN socket 和独立的 accept 队列**，新连接由内核按四元组哈希**只投递给其中一个 socket**，所以只有那一个 worker 被唤醒去 accept，从根本上不存在"唤醒一群只有一个成功"的惊群。`EPOLLEXCLUSIVE` 适用场景不同：它用于多个 epoll 实例**监听同一个共享 LISTEN fd** 的情况，让内核对一个事件只唤醒一个 epoll 实例来缓解惊群（仍共享同一队列、无负载均衡）；REUSEPORT 是各自独立 fd + 内核哈希分发，自带负载均衡，更彻底。

6. 诊断：① `dmesg | grep conntrack` 确认 `nf_conntrack: table full, dropping packet`；② `cat /proc/sys/net/netfilter/nf_conntrack_count` 对比 `sysctl net.netfilter.nf_conntrack_max`，看是否逼近上限；③ `conntrack -S` 看 insert_failed/drop，`conntrack -L -p tcp --state ESTABLISHED | wc -l` 看堆积的 established 表项。缓解手段：调大 `nf_conntrack_max` 和相应的 `nf_conntrack_buckets`（注意每条目占数百字节内存）；缩短无用状态超时（如 `nf_conntrack_tcp_timeout_time_wait`）加速回收；对确实不需要跟踪的大流量（纯转发、LVS DR）用 iptables raw 表 `NOTRACK` 从源头跳过 conntrack。治本思路：用 eBPF（Cilium）或 IPVS 替代 kube-proxy 的 iptables 模式，减少 conntrack 压力。

7. 定位思路：`ss -tanp` 找出大量 `CLOSE-WAIT` 的连接及其所属进程（`-p`），确认是哪个服务；CLOSE-WAIT 表示**对端已发 FIN、本端应用迟迟没调用 `close()`**，连接和 fd 因此泄漏、持续增长。这通常不是内核调参能解决的，因为它是**应用层 bug**——代码在某些路径（尤其异常/错误返回路径）忘了关闭连接/fd；内核已正确进入 CLOSE_WAIT 等应用 close，调任何 sysctl 都不会替应用去 close。修法是审查代码确保所有路径（含异常）都 close。

8. `tcpaccept`：实时打印每个被 accept 的入向连接（谁在向我建连）；`tcplife`：每条连接的生命周期（起止、收发字节数、存活时长），回答"每条连接活了多久"；`tcpretrans`：实时打印重传事件，回答"是否有重传"（定位丢包/弱网）。（另：`tcpconnect` 打印出向新建连接。）这些都是 bcc/eBPF 工具，不改应用、不抓全量包、开销极低。

9. Unix domain socket 优势：用于**同机进程间通信**，不走网络协议栈（无 IP/TCP 头、无校验和、无回环路由），比 TCP 回环更快、开销更低。`SCM_RIGHTS`：能通过 `sendmsg` 的辅助数据**把文件描述符传递给另一个进程**（fd passing），是 socket activation 与特权分离的关键能力。abstract socket 与路径命名的区别：路径命名的 Unix socket 在文件系统里落地一个 socket 文件（需清理、受文件权限控制）；abstract socket 的名字以 `\0` 开头、**不落地文件**，存在于抽象命名空间，随进程消亡而自动消失（无需手动清理）。

10. conntrack 表项堆积的原因：每条经过 NAT/有状态防火墙/K8s Service 的流都占一个表项，且表项有状态超时——只有超时到了内核才回收。established TCP 的默认超时很长（天级），所以大量早已断开、但还没到超时被回收的连接会一直占着表项，造成堆积。established 超时与表满的关系：超时越长，已断连接的"僵尸"表项存活越久、累积越多，越容易把 `nf_conntrack_count` 顶到 `nf_conntrack_max` 导致表满丢包。安全加速回收：适当调小相关超时（如 `nf_conntrack_tcp_timeout_time_wait`，established 超时调整需谨慎评估，避免误删仍活跃的长连接），同时调大 `nf_conntrack_max`/`buckets` 兜底；对不需跟踪的流量用 NOTRACK 减负。
