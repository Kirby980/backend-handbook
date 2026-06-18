# 精通文件描述符与 I/O 多路复用：五种 I/O 模型、select/poll、epoll、LT/ET、惊群

> 课程编号：L08
> 路线图来源：Linux · 模块三 文件与 I/O
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：fd 表、epoll 红黑树+就绪链表、EPOLLEXCLUSIVE、SO_REUSEPORT、Go netpoller / Nginx / Redis 事件循环；io_uring 接棒见 L09）

---

## 引言：从 C10K 到 C10M，一个线程喂饱百万连接

1999 年 Dan Kegel 提出 **C10K 问题**：一台服务器如何同时处理一万个并发连接？当时主流模型是"一个连接一个线程"——一万连接就要一万个线程，光线程栈就吃几十 GB，上下文切换把 CPU 拖垮。二十多年后的今天，单机扛百万连接（**C10M**）是 Nginx、Redis、各类网关的家常便饭，靠的不是更猛的硬件，而是一套截然不同的 I/O 范式：**事件驱动 + I/O 多路复用**——一个（或少数几个）线程，通过 `epoll` 同时盯着百万个 fd，谁就绪处理谁。

来看一个反直觉的数字对比。一万连接、各自偶尔发点数据：

| 模型 | 线程数 | 内存（仅栈） | CPU 主要开销 | 能否上百万连接 |
|---|---|---|---|---|
| thread-per-connection | 10000 | ~80 GB（8MB/栈）| 上下文切换、调度 | 不能 |
| select / poll 单线程 | 1 | 极小 | **每轮 O(n) 扫描全部 fd** | 不能（fd 多了扫描爆炸） |
| epoll 事件循环 | 1（+少量 worker）| 极小 | **只处理就绪的 fd，O(1) 拿就绪** | 能 |

核心矛盾在于：阻塞式 IO 让一个线程只能盯一个 fd（阻塞在 `read` 上），于是连接数 = 线程数。要让一个线程盯住成千上万个 fd，就必须有两样东西：**非阻塞 fd**（read 没数据立刻返回而不是睡死）+ **一个高效的"谁就绪了"通知机制**（这就是多路复用器 select/poll/epoll）。

本篇从 fd 的内核表示讲起，厘清五种 I/O 模型的精确定义（尤其"同步 vs 异步"这对常被混淆的概念），然后逐一拆解 select/poll 的 O(n) 之痛、epoll 为何 O(1)、LT 与 ET 的精确语义与经典 bug，最后讲惊群、`EPOLLEXCLUSIVE` / `SO_REUSEPORT` 与 Reactor 模式，并对照 Go netpoller、Nginx、Redis 的事件循环。读完你应该能解释清楚自己用的语言 runtime 之下，那台"多路复用引擎"到底怎么转。

---

## 第一章 文件描述符：fd 表、struct file 共享、O_NONBLOCK、上限

### 1.1 fd 是什么：进程局部的小整数

文件描述符（fd）是一个进程局部的非负整数，作为索引指向内核的"打开文件表项"。回顾 L07 的三级映射：

```
进程 fd 表 (files_struct.fdt.fd[])
   fd=0 ─┐
   fd=1 ─┼─► struct file ──► struct inode/socket
   fd=2 ─┘     (f_pos, f_flags, f_op...)
   fd=3 ─────► struct file ──► ...
```

`fd[i]` 是 `struct file *`。多个 fd 可指向同一个 `struct file`（`dup`/`fork` 后共享，共享偏移量），不同 `struct file` 可指向同一 inode（独立 open，独立偏移）。

对网络编程而言，socket 也是 fd——它的 `struct file` 的 `f_op` 是 socket 专用操作表，背后是 `struct socket` / `struct sock`。"一切皆文件"让 `epoll` 能同时管理普通文件、socket、pipe、eventfd、timerfd、signalfd（虽然普通磁盘文件对 epoll 意义有限，见后文）。

### 1.2 O_NONBLOCK：多路复用的前提

默认 fd 是**阻塞**的：`read` 没数据时进程睡眠直到有数据。**非阻塞** fd（`O_NONBLOCK`）则在没数据时立即返回 `-1` 并置 `errno = EAGAIN`（或 `EWOULDBLOCK`，二者等值）。

```c
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);   // 设为非阻塞

ssize_t n = read(fd, buf, sizeof(buf));
if (n < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) {
    // 暂时没数据，不是错误，回到事件循环等下次就绪
}
```

**为什么多路复用必须配非阻塞**：事件循环里，epoll 告诉你"fd 就绪了"，你去 read。但在 ET 模式下你要循环读到 EAGAIN 为止（见第五章）；即使 LT，多个 fd 共享一个处理线程时，任何一次 read 阻塞都会拖垮整个事件循环（一个慢连接卡死所有连接）。所以事件驱动框架里所有被 epoll 管理的 fd 都设非阻塞。

### 1.3 fd 上限：ulimit 与 /proc

fd 数量有两道限制：

```bash
# 1) 进程级软/硬上限（RLIMIT_NOFILE）
$ ulimit -n          # 软限，默认常见 1024
1024
$ ulimit -Hn         # 硬限
524288

# 2) 系统级总上限
$ cat /proc/sys/fs/file-max          # 全系统可打开的 file 对象数
9223372036854775807   # 现代内核很大，基本不再是瓶颈
$ cat /proc/sys/fs/nr_open           # 单进程 fd 表能扩到的上限
1048576
```

百万连接的第一道坎就是 `ulimit -n`——默认 1024 直接卡死。systemd 服务用 `LimitNOFILE=` 调整（见 L20）：

```ini
[Service]
LimitNOFILE=1048576
```

```bash
# 查看进程实际打开了多少 fd
$ ls /proc/<pid>/fd | wc -l
$ cat /proc/<pid>/limits | grep "open files"
```

**陷阱**：很多"连接数上不去"的事故，根因是某层（容器、systemd、shell）的 `nofile` 没调够，到上限后 `accept` 返回 `EMFILE`（进程级）或 `ENFILE`（系统级），新连接全被拒。

---

## 第二章 五种 I/O 模型：同步与异步的精确定义

POSIX 与《UNIX 网络编程》定义了五种 I/O 模型。先把两个易混概念钉死：

- **阻塞 vs 非阻塞**：发起 IO 的调用在"数据没准备好"时，是**睡眠等待**还是**立即返回 EAGAIN**。
- **同步 vs 异步**：真正的**数据拷贝**（内核缓冲区 ↔ 用户缓冲区）这一步，是**由发起 IO 的线程自己做（同步）**，还是**内核帮你做完再通知你（异步）**。

一次 IO 概念上分两阶段：**① 等数据就绪**（如等网卡收到包、数据进内核 socket buffer）；**② 拷贝数据**（内核 buffer → 用户 buffer）。

```
                     阶段① 等就绪        阶段② 拷贝数据      谁拷贝
1 阻塞 IO            线程睡眠等          线程同步拷贝        线程
2 非阻塞 IO          轮询(EAGAIN忙等)    线程同步拷贝        线程
3 I/O 多路复用       select/epoll 等     线程同步拷贝        线程  ★ 仍是同步
4 信号驱动 IO        SIGIO 通知就绪      线程同步拷贝        线程
5 异步 IO (AIO)      内核全包           内核拷贝完才通知     内核  ★ 真异步
```

**关键结论**：前四种都是**同步 IO**——因为阶段②的数据拷贝都是发起线程自己做的（线程会"卡"在拷贝那一下，哪怕拷贝很快）。**只有第五种（真正的异步 IO）阶段②也由内核完成**，线程拿到通知时数据已经在自己的缓冲区里了。epoll 属于多路复用（第 3 种），**它是同步的**——这点常被误解。真正补齐"异步"短板的是 io_uring（见 L09）。

### 2.1 五种模型逐个看

```c
// ① 阻塞：最简单，但一个线程只能盯一个 fd
n = read(fd, buf, len);          // 没数据就睡，有数据拷贝后返回

// ② 非阻塞：忙轮询，CPU 空转，几乎不单独用
fcntl(fd, F_SETFL, O_NONBLOCK);
while ((n = read(fd, buf, len)) < 0 && errno == EAGAIN) { /* 自旋 */ }

// ③ 多路复用：一个线程等一批 fd（本篇主角）
epoll_wait(epfd, events, maxevents, -1);
for (i...) read(events[i].data.fd, buf, len);  // 拷贝仍同步

// ④ 信号驱动：fd 就绪发 SIGIO，处理函数里再 read。信号语义复杂、
//    丢信号风险、异步信号安全约束多，实践中罕用（见 L14）

// ⑤ 异步 IO：提交请求即返回，内核拷贝完成后通知
//    POSIX AIO/libaio 长期鸡肋（见 L09），io_uring 才真正可用
```

信号驱动 IO（第 4 种）因为信号不排队、可能丢、且信号处理函数受异步信号安全限制（见 L14），生产几乎不用，了解即可。

---

## 第三章 select / poll：O(n) 的拷贝与扫描

### 3.1 select：fd_set 位图与三大限制

`select` 让你传入三组 fd（可读、可写、异常），阻塞直到至少一个就绪：

```c
fd_set rfds;
FD_ZERO(&rfds);
FD_SET(sockfd, &rfds);
struct timeval tv = {5, 0};
int n = select(maxfd + 1, &rfds, NULL, NULL, &tv);
for (int fd = 0; fd <= maxfd; fd++)
    if (FD_ISSET(fd, &rfds)) { /* fd 可读，处理 */ }
```

三大致命限制：

1. **fd 上限 1024**：`fd_set` 是固定大小位图，由编译期常量 `FD_SETSIZE`（通常 1024）决定。fd 超过 1024 就溢出，C10K 直接出局。
2. **每次调用都全量拷贝位图**：用户态的三个 `fd_set` 每次 `select` 都整份拷进内核，返回再拷出——O(n) 拷贝。
3. **返回后要 O(n) 扫描**：内核只告诉你"有几个就绪"，不告诉你"是哪几个"，你得遍历所有 fd 用 `FD_ISSET` 逐个查——O(n) 扫描。

而且 `fd_set` 被内核就地改写成结果，**每次循环都要重新 `FD_SET`**，否则下一轮就错了。

### 3.2 poll：去掉 1024 限制，仍是 O(n)

`poll` 用 `struct pollfd` 数组取代位图，去掉了 1024 上限，但**没解决 O(n)**：

```c
struct pollfd fds[N];
fds[0].fd = sockfd; fds[0].events = POLLIN;
int n = poll(fds, N, 5000);   // 数组每次全量拷进内核
for (int i = 0; i < N; i++)
    if (fds[i].revents & POLLIN) { /* 处理 */ }   // 仍 O(n) 扫描
```

| | select | poll | epoll |
|---|---|---|---|
| fd 上限 | 1024（FD_SETSIZE）| 无硬上限 | 无硬上限 |
| 每次调用拷贝 | 全量 fd_set | 全量 pollfd 数组 | **只在 epoll_ctl 时一次性注册** |
| 找就绪 fd | O(n) 扫描 | O(n) 扫描 | **O(1) 取就绪链表** |
| 数据结构 | 位图 | 数组 | 内核红黑树 + 就绪链表 |
| 适用规模 | 小（<几百） | 中 | 大（百万） |

**O(n) 之痛的本质**：select/poll 是"无状态"的——内核不记得你上次关心哪些 fd，每次调用都要把"全部关心的 fd"重新告诉内核（拷贝），内核检查完又要你"全部扫一遍"找就绪的。当 fd 有 10 万个但每次只有 10 个就绪时，99.99% 的工作是白费的。epoll 的革命就在于把这套"无状态"改成"有状态"。

---

## 第四章 epoll：红黑树 + 就绪链表，为何 O(1)

### 4.1 三个系统调用

epoll 把"关心哪些 fd"这件事在内核里**持久化**成一个对象（`struct eventpoll`），用三个系统调用操作它：

```c
int epfd = epoll_create1(0);                 // 创建 epoll 实例，返回一个 fd

struct epoll_event ev = { .events = EPOLLIN, .data.fd = sockfd };
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev); // 注册 fd（一次，常驻内核）
// EPOLL_CTL_MOD 改、EPOLL_CTL_DEL 删

struct epoll_event events[1024];
int n = epoll_wait(epfd, events, 1024, -1);  // 等就绪，只返回就绪的 fd
for (int i = 0; i < n; i++) { /* 处理 events[i] */ }
```

对比 select/poll：**注册（ctl）与等待（wait）分离**。fd 注册一次就常驻内核红黑树，`epoll_wait` 不再传入 fd 列表、也不返回全部 fd，**只返回就绪的那几个**。

### 4.2 内核数据结构：红黑树 + 就绪链表

```c
// fs/eventpoll.c (简化)
struct eventpoll {
    struct rb_root_cached rbr;     // ① 红黑树：所有注册的 fd（epitem）
    struct list_head      rdllist; // ② 就绪链表：已就绪的 fd
    wait_queue_head_t     wq;      // 阻塞在 epoll_wait 上的进程
};
struct epitem {                    // 每个被监控的 fd 一个
    struct rb_node   rbn;          // 挂在红黑树上
    struct list_head rdllink;      // 就绪时挂到 rdllist
    struct epoll_event event;      // 用户关心的事件 + data
    struct epoll_filefd ffd;       // 目标 fd
};
```

```
         epoll_ctl(ADD/MOD/DEL)              epoll_wait
                 │                               │
                 ▼                               ▼
   ┌──────────────────────────┐      ┌────────────────────┐
   │  红黑树 rbr（所有 fd）     │      │ 就绪链表 rdllist    │
   │  插/删/查 O(log n)        │      │ 只挂就绪的 fd       │
   │  fd5 fd9 fd12 fd88 ...    │      │ fd9 → fd88          │
   └──────────────────────────┘      └────────────────────┘
                 ▲                               ▲
                 │ ep_poll_callback 在 fd 就绪时  │
                 └───────────把 epitem 挂入就绪链表┘
```

### 4.3 ep_poll_callback：就绪事件的"推送"

这是 epoll 高效的核心机制。注册 fd 时，epoll 在该 fd 的等待队列上挂一个回调 `ep_poll_callback`。当 fd 状态变就绪（如 socket 收到数据，协议栈唤醒等待队列），内核就调用这个回调，把对应的 `epitem` **挂到就绪链表 `rdllist`**，并唤醒阻塞在 `epoll_wait` 上的进程。

```
网卡收包 → 协议栈处理 → 数据进 socket 接收缓冲 → 唤醒该 sock 等待队列
   → 触发 ep_poll_callback → epitem 挂入 rdllist → 唤醒 epoll_wait
```

于是 `epoll_wait` 醒来时，**只需把就绪链表里的 epitem 拷给用户**，完全不用遍历全部注册的 fd。这就是 **O(1)（相对 fd 总数）** 的来源——开销只跟"就绪的 fd 数量"成正比，跟"注册的 fd 总数"无关。

### 4.4 epoll 复杂度总结

| 操作 | 复杂度 | 说明 |
|---|---|---|
| `epoll_ctl(ADD/DEL)` | O(log n) | 红黑树插删 |
| fd 就绪 | O(1) | 回调直接挂就绪链表 |
| `epoll_wait` | O(就绪数) | 只拷就绪的，与注册总数无关 |

**为什么用红黑树管 fd**：需要按 fd 快速查找（`EPOLL_CTL_MOD`/`DEL` 要定位到 epitem）、插入、删除，且 fd 数量可达百万——红黑树 O(log n) 且有序、内存紧凑，是平衡之选。就绪集合则用链表（只需头插和遍历，O(1) 插入）。

**注意**：epoll 对**普通磁盘文件**几乎没用——普通文件总是"就绪"的（read 不会因为没数据而阻塞，只会因为 IO 慢而阻塞），epoll 会立刻报就绪但 read 仍可能卡在磁盘 IO。磁盘文件的真正异步要靠 io_uring（L09）。epoll 的主战场是 socket、pipe、eventfd 这类"会有就绪/未就绪状态切换"的 fd。

---

## 第五章 LT vs ET：水平触发与边缘触发

### 5.1 两种语义

epoll 的事件可注册为两种触发模式：

- **LT（Level Triggered，水平触发，默认）**：只要 fd 处于就绪状态（如接收缓冲里还有数据没读完），**每次** `epoll_wait` 都会持续报告它就绪。
- **ET（Edge Triggered，边缘触发，加 `EPOLLET`）**：只在 fd 状态**发生变化的那一刻**（从无数据→有数据）报告**一次**。如果你没把数据读完，下次 `epoll_wait` **不会**再因为"缓冲里还有剩"而报告它，必须等**新数据到达**才再触发。

```
接收缓冲收到 100 字节，你只 read 了 40 字节：
  LT：下次 epoll_wait 仍报就绪（还有 60 字节）→ 你可以慢慢分多次读
  ET：下次 epoll_wait 不报（状态没"变化"）→ 那 60 字节"卡住"了！
      除非又来新数据触发新的"边缘"
```

### 5.2 ET 必须配非阻塞 + 循环读尽

ET 的铁律：**收到就绪后，必须用非阻塞 fd 循环 read 直到返回 EAGAIN**，把缓冲彻底读空，否则剩余数据会"饿死"。

```c
// ET 模式下正确的读循环
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n > 0) {
        process(buf, n);
        // 继续循环，可能还有数据
    } else if (n == 0) {
        // 对端关闭
        close(fd);
        break;
    } else {   // n < 0
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            break;   // ★ 读空了，正常退出，等下次边缘
        } else if (errno == EINTR) {
            continue;
        } else {
            close(fd);  // 真错误
            break;
        }
    }
}
```

写也类似：ET 下 `EPOLLOUT` 只在"从不可写变可写"时触发一次，要循环 write 到 EAGAIN，并按需用 `EPOLL_CTL_MOD` 动态开关 `EPOLLOUT`（没东西要写时关掉，否则会持续触发空转——LT 下尤其要注意这点）。

### 5.3 LT vs ET 取舍

| | LT（默认） | ET |
|---|---|---|
| 编程难度 | 低（读多少随意，没读完下次还报） | 高（必须循环读尽到 EAGAIN） |
| epoll_wait 唤醒次数 | 多（只要有剩就报） | 少（只在边缘报，减少系统调用） |
| 漏读后果 | 无（下次还会提醒） | **数据饿死**（最常见 ET bug） |
| 必须非阻塞 | 建议 | **强制** |
| 典型使用者 | 大多数应用、Redis | Nginx（高性能、追求少唤醒） |

ET 减少 `epoll_wait` 返回次数（一次把数据读完，不重复提醒），在极高并发下能省下可观的系统调用开销，但把"读尽"的责任压给了应用，写错就丢数据或卡连接。**Nginx 用 ET，Redis 用 LT**——各有取舍。新手优先用 LT。

### 5.4 经典 ET bug 集锦

1. **没读到 EAGAIN 就退出**：只 read 一次就回事件循环，剩余数据饿死，连接"假死"。
2. **用了阻塞 fd**：ET + 阻塞 fd，循环读到没数据时 read 会阻塞，整个事件循环卡死。
3. **accept 漏接**：监听 socket 用 ET 时，一次"边缘"可能堆了多个待 accept 连接，必须 `while(accept() != EAGAIN)` 循环接尽，否则有连接被晾着。
4. **EPOLLOUT 空转**：始终注册 `EPOLLOUT`，发送缓冲长期可写导致 `epoll_wait` 疯狂返回，CPU 100%。正确做法是只在有数据待发且上次 write 遇到 EAGAIN 时才挂 `EPOLLOUT`。

---

## 第六章 惊群、EPOLLEXCLUSIVE、SO_REUSEPORT 与事件循环

### 6.1 惊群（thundering herd）

多个进程/线程在同一个监听 socket 上等待 accept（或多个 epoll 等同一 fd），一个连接到来时，内核**唤醒所有**等待者，但只有一个能 accept 成功，其余白白醒来又睡回去——浪费 CPU、加剧调度抖动。这就是惊群。

历史上：
- 内核早已对 **accept 惊群**做了优化（只唤醒一个等待者）。
- 但 **多个 epoll 实例监控同一个共享 fd** 的惊群长期存在：N 个进程各自 epoll 同一监听 socket，新连接唤醒全部 N 个。

### 6.2 两种解法：EPOLLEXCLUSIVE 与 SO_REUSEPORT

**方案一：`EPOLLEXCLUSIVE`（4.5+）**。注册时加此标志，内核保证一个事件只唤醒一个（或少数）等待者，缓解 epoll 共享 fd 的惊群：

```c
struct epoll_event ev = { .events = EPOLLIN | EPOLLEXCLUSIVE };
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
```

**方案二：`SO_REUSEPORT`（推荐，3.9+）**。让多个 socket **bind 到同一 IP:Port**，内核为每个 socket 维护独立的 accept 队列，并用哈希把新连接**负载均衡**地分发到其中一个。每个 worker 进程拥有自己独立的监听 socket + 独立 epoll，从根上消除惊群，还自带负载均衡：

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
int on = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &on, sizeof(on));  // ★ 每个 worker 都设
bind(fd, ...);
listen(fd, backlog);
// 每个 worker 进程一个这样的 fd，内核自动分发连接
```

| | EPOLLEXCLUSIVE | SO_REUSEPORT |
|---|---|---|
| 解决 | 共享 fd 的 epoll 惊群 | 惊群 + 负载均衡 |
| fd 模型 | 多进程共享同一监听 fd | 每进程独立监听 fd |
| 负载均衡 | 否（仍靠内核唤醒策略） | **是**（内核哈希分发） |
| 优雅重启 | 一般 | **更好**（可平滑增减 worker） |
| 内核 | 4.5+ | 3.9+ |

SO_REUSEPORT 是现代高并发服务的主流选择（Nginx `reuseport`、Envoy、各类 Go 服务）。详见 L13 Socket 章节。

### 6.3 Reactor 模式

事件驱动服务的经典架构是 **Reactor**：一个事件循环（epoll_wait）等待就绪事件，分发（dispatch）给对应的处理器（handler）。

```
        ┌─────────────── Reactor 事件循环 ──────────────┐
        │  while(1) {                                    │
        │    n = epoll_wait(epfd, events, ...);          │
        │    for each ready event:                       │
        │      if listen_fd:  accept → 注册新连接到 epoll │
        │      if conn_fd readable: read → 解析 → 业务    │
        │      if conn_fd writable: 继续发送缓冲          │
        │  }                                             │
        └────────────────────────────────────────────────┘
```

变体：
- **单 Reactor 单线程**：Redis 经典模型（命令处理单线程，省锁）。
- **单 Reactor 多线程**：IO 在主线程，业务计算丢给线程池。
- **主从 Reactor（多 Reactor）**：主 Reactor 只 accept，从 Reactor 各自 epoll 一批连接做 IO——Nginx、Netty 的模型。

### 6.4 三大生产事件循环对照

| 系统 | 多路复用 | 触发模式 | 并发模型 | 备注 |
|---|---|---|---|---|
| **Nginx** | epoll | ET | 主从 Reactor，多 worker + SO_REUSEPORT | 极致少唤醒 |
| **Redis** | epoll（ae 抽象层）| LT | 单 Reactor 单线程（6.x 起 IO 多线程拆分读写） | 命令执行单线程不锁，详见 redis 性能模型 |
| **Go runtime** | epoll（netpoller）| ET | M:N 协程 + netpoller 集成调度器 | 见下 |

**Go netpoller**：Go 把 epoll 藏在 runtime 里。当 goroutine 在 socket 上 `Read` 而无数据时，runtime 不会阻塞底层 OS 线程（M），而是把该 goroutine 挂起、注册 fd 到 netpoller（epoll，ET 模式），让 M 去跑别的 goroutine。fd 就绪时 netpoller 唤醒对应 goroutine 重新入调度队列。于是你写"同步阻塞风格"的 `conn.Read()`，runtime 底层却是 epoll 事件驱动——**用同步的代码拿到异步的性能**。这与本系列 golang netpoller 的描述一致，也是 Go 网络高并发的根基。

```go
// 你写的是"阻塞"风格，runtime 底层是 epoll
func handle(conn net.Conn) {
    buf := make([]byte, 4096)
    for {
        n, err := conn.Read(buf)  // goroutine 挂起，M 去跑别人；就绪后被唤醒
        if err != nil { return }
        conn.Write(buf[:n])
    }
}
```

### 6.5 epoll 的天花板与 io_uring 接棒

epoll 已经把"等就绪"做到极致，但它有两个根本局限：

1. **它只是"就绪通知"，不是"完成通知"**——epoll 告诉你"可以读了"，你还得自己 `read`（同步拷贝）。每个就绪 fd 至少一次系统调用，百万 QPS 下系统调用开销可观。
2. **对磁盘文件无能为力**——普通文件没有"就绪"概念，epoll 帮不上忙。

io_uring（见 L09）正是冲着这两点来的：它是真正的**异步完成模型**——你提交"读这个 fd 到这个缓冲区"的请求，内核完成（含数据拷贝）后把结果放进完成队列，全程可零系统调用（SQPOLL）、可批量、磁盘网络通吃。2026 年，高性能新项目的 IO 引擎首选已从 epoll 转向 io_uring，epoll 退居"稳妥通用"的位置。

---

## 生产实践

1. **先调 nofile 再谈并发**：百万连接前，确认容器、systemd（`LimitNOFILE`）、`/proc/sys/fs/nr_open`、应用自身 `setrlimit` 层层够大。漏掉任一层，到上限就 `accept` 返回 `EMFILE`。

2. **监听用 SO_REUSEPORT 多 worker**：CPU 多核场景，每个 worker 独立监听 + 独立 epoll，内核分发连接，既消除惊群又负载均衡，还利于优雅重启。Nginx 加 `reuseport`，自研服务直接 setsockopt。

3. **ET 一律配非阻塞 + 读到 EAGAIN**：用 ET 提升性能时，把"循环读尽到 EAGAIN""accept 接尽""按需开关 EPOLLOUT"做成框架级约束，避免每个 handler 各写各错。

4. **新手 / 业务逻辑复杂时用 LT**：LT 容错性强（没读完下次还提醒），把精力放业务上。Redis 用 LT 跑到极高 QPS，说明 LT 并非性能瓶颈。

5. **epoll 不用于磁盘文件**：需要异步磁盘 IO 用 io_uring（L09）或线程池兜底，别指望 epoll。混合负载（网络 + 磁盘）的服务尤其注意，别让磁盘 read 卡死网络事件循环。

6. **监控就绪处理延迟**：`epoll_wait` 返回到事件处理完的时间，是事件循环健康度的关键指标。单个 handler 耗时过长（如同步磁盘 IO、大计算）会拖垮整个循环，必要时把重活丢线程池。

---

## 陷阱清单

1. **连接数上不去，accept 报 EMFILE / ENFILE**
   - 现象：并发到某数（常 1024）就不再涨，日志大量 `Too many open files`。
   - 原因：`ulimit -n` 软限太小，或容器 / systemd / 应用某层没放开 nofile。
   - 修法：systemd `LimitNOFILE=1048576`；容器 `--ulimit nofile=`；应用启动 `setrlimit(RLIMIT_NOFILE)`；查 `/proc/<pid>/limits` 确认生效。

2. **ET 模式连接"假死"**
   - 现象：客户端发了数据，服务端某些连接收不全、卡住不再响应。
   - 原因：ET 下只 read 一次没读到 EAGAIN，剩余数据饿死，不再触发。
   - 修法：ET 必须循环 read 到 EAGAIN/EWOULDBLOCK；fd 必须非阻塞。

3. **CPU 100% 但吞吐很低（EPOLLOUT 空转）**
   - 现象：epoll_wait 疯狂返回，CPU 占满，业务没增长。
   - 原因：常驻注册了 EPOLLOUT，发送缓冲长期可写，每轮都报"可写"。
   - 修法：只在有数据待发且上次 write 遇 EAGAIN 时挂 EPOLLOUT，发完用 EPOLL_CTL_MOD 摘掉。

4. **监听 socket 漏接连接（ET accept）**
   - 现象：高并发建连时偶发"连接像被吞了"，客户端超时。
   - 原因：ET 下监听 fd 一次边缘堆了多个待 accept，只 accept 一个就返回了。
   - 修法：`while ((cfd=accept(...))>=0) {...}` 循环接到 EAGAIN；或用 LT 监听。

5. **多进程 epoll 共享监听 fd 惊群**
   - 现象：CPU 随 worker 数线性上升但吞吐不涨，每个新连接唤醒所有 worker。
   - 原因：N 个进程 epoll 同一个监听 fd，事件唤醒全部。
   - 修法：改用 SO_REUSEPORT（每进程独立监听 fd + 内核分发）；或注册 EPOLLEXCLUSIVE。

6. **把 select 用在 fd > 1024 的场景导致越界**
   - 现象：fd 编号大时 `FD_SET` 踩坏栈/堆，随机崩溃或行为诡异。
   - 原因：`fd_set` 固定 FD_SETSIZE（1024）位，fd ≥ 1024 越界。
   - 修法：高并发一律用 epoll；非要兼容用 poll（无此限制）；绝不在大 fd 上用 select。

7. **fork 后子进程误用父进程的 epfd**
   - 现象：多进程模型里事件错乱、重复处理或漏处理。
   - 原因：epoll 实例（epfd）跨 fork 共享同一内核对象，多个进程在同一 epoll 上 wait 行为复杂。
   - 修法：每个 worker 进程在 fork 后各自 `epoll_create1` 并注册自己的 fd（配合 SO_REUSEPORT）。

8. **epoll 监控普通文件期望异步**
   - 现象：把磁盘文件 fd 加进 epoll，read 仍阻塞在磁盘 IO，事件循环卡顿。
   - 原因：普通文件总报"就绪"，epoll 不解决磁盘 IO 的阻塞。
   - 修法：磁盘异步用 io_uring（L09）或独立线程池，别和网络事件循环混在一起。

---

## 2026 现状

- **epoll 仍是网络多路复用的稳妥默认**：Nginx、Redis、Envoy、绝大多数语言 runtime（Go netpoller、Node libuv、Java NIO/Netty）底层都是 epoll，成熟、可移植（相对而言）、调试工具齐全。
- **io_uring 在高性能新项目中接棒**：需要榨干单机性能、或网络+磁盘混合 IO 的场景，io_uring 的"完成式异步 + 批量 + 零系统调用（SQPOLL）+ multishot accept/recv"全面超越 epoll（见 L09）。但因安全收紧（`io_uring_disabled` sysctl），部分云/容器环境默认禁用，需 epoll 兜底。
- **SO_REUSEPORT + 多 worker** 是多核高并发的标准姿势，惊群在工程上基本已是历史问题。
- **Go netpoller 持续演进**：runtime 把 epoll（ET）与调度器深度集成，"同步代码、异步性能"成为 Go 网络编程的护城河；社区也有基于 io_uring 的实验性网络栈。
- **eBPF/XDP 在更前置的层做分流**（L11/L19），把部分连接分发、DDoS 过滤下沉到驱动层，与用户态 epoll/io_uring 形成分层。

---

## 练习题

1. 用"等数据就绪 / 拷贝数据"两阶段框架，解释为什么 epoll（I/O 多路复用）属于**同步** I/O，而 io_uring 属于**异步** I/O。

2. select 有哪三大限制？poll 解决了哪一个、没解决哪一个？为什么说 select/poll 是"无状态"而 epoll 是"有状态"？

3. 画出 epoll 的红黑树 + 就绪链表结构，并说明 `ep_poll_callback` 在 fd 就绪时做了什么，从而解释 `epoll_wait` 为何与"注册 fd 总数"无关、只与"就绪数"成正比。

4. LT 与 ET 的精确语义差别是什么？写出 ET 模式下一个正确的读循环（含 EAGAIN/EINTR/对端关闭处理），并说明为什么 ET 必须配非阻塞 fd。

5. 惊群是什么？`EPOLLEXCLUSIVE` 与 `SO_REUSEPORT` 各自如何解决，二者关键区别（尤其负载均衡）是什么？

6. 解释 Go netpoller 如何让你写"阻塞风格"的 `conn.Read()` 却获得 epoll 事件驱动的性能。当一个 goroutine 在 Read 上"阻塞"时，承载它的 OS 线程（M）在做什么？

7. **实战**：写一个最小的 epoll TCP echo server（C 或 Go 伪代码均可），监听 socket 用 LT、连接 fd 用 ET，正确处理 accept 接尽、读到 EAGAIN、对端关闭。

8. **排障**：某服务 CPU 跑满 100% 但 QPS 很低，`strace` 显示 `epoll_wait` 几乎不阻塞、立刻返回且返回的多是同一批 fd 的"可写"事件。请定位根因并给出修法。
