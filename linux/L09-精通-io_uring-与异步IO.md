# 精通 io_uring 与异步 I/O：SQ/CQ 双环、注册缓冲、SQPOLL、网络 io_uring、安全收紧

> 课程编号：L09
> 路线图来源：Linux · 模块三 文件与 I/O
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：io_uring 成熟、liburing、multishot、zero-copy send、SQPOLL、io_uring_disabled sysctl、容器/云默认收紧）

---

## 引言：epoll 的天花板与 io_uring 的诞生

L08 把 epoll 讲到了极致：红黑树 + 就绪链表，O(1) 拿就绪、百万连接事件驱动。但 epoll 有一道无法逾越的天花板——**它只是"就绪通知"，不是"完成通知"**。

回顾那两阶段：① 等数据就绪、② 拷贝数据。epoll 帮你高效完成了①，但②（真正的 `read`/`write`）还得你自己用一次系统调用同步去做。于是一次完整的 IO 至少两次系统调用（`epoll_wait` + `read`），百万 QPS 下，仅 **系统调用本身的进出内核开销**（保存/恢复寄存器、栈切换，叠加 Meltdown/Spectre 之后的 KPTI 页表切换代价）就能吃掉可观 CPU。更要命的是：**epoll 对磁盘文件无能为力**——普通文件永远"就绪"，read 仍会同步卡在磁盘 IO 上。

那 Linux 不是早有"异步 IO"吗？POSIX AIO、libaio？答案是：它们鸡肋到几乎没人用（下面会拆）。直到 2019 年（5.1）Jens Axboe 提出 **io_uring**，Linux 才第一次拥有了一个**通用、高效、磁盘网络通吃的真异步 IO 接口**。它的核心思想朴素而强大：**用两个内核与用户态共享的环形队列（ring），让"提交请求"和"收割结果"都尽量不陷入内核**。提交 N 个 IO 可以一次系统调用，甚至开了 SQPOLL 后**零系统调用**。

来感受这个量级差异：

| 模型 | 提交 N 个 IO 的系统调用数 | 数据拷贝由谁做 | 磁盘文件 | 批量 |
|---|---|---|---|---|
| 阻塞 read | N 次 | 线程（同步） | 阻塞 | 否 |
| epoll | epoll_wait + N 次 read | 线程（同步） | 不支持 | 部分 |
| libaio | io_submit（1 次）+ io_getevents | 内核 | 仅 O_DIRECT | 是 |
| **io_uring** | **1 次（SQPOLL 下 0 次）** | **内核（真异步）** | **全支持** | **是** |

本篇从"为什么 libaio 鸡肋"切入，拆解 io_uring 的 SQ/CQ 双环架构、三个系统调用、提交与完成流程、链式请求、注册缓冲/文件、SQPOLL 内核轮询，再到 2026 年成熟的网络 io_uring（multishot、zero-copy send）与 epoll 性能对比，最后讲生态（liburing、各语言绑定）与安全争议（`io_uring_disabled`、容器/云默认收紧）。这是本系列难度最高的一篇——但读完你就握住了现代 Linux 高性能 IO 的方向盘。

---

## 第一章 Linux 异步 I/O 历史：为什么 POSIX AIO / libaio 鸡肋

### 1.1 两套"旧 AIO"

io_uring 之前，Linux 上"异步 IO"有两套，都不好用：

**① POSIX AIO（`aio_read`/`aio_write`，glibc 实现）**：glibc 在**用户态用线程池**模拟异步——`aio_read` 不过是把读操作丢给一个后台线程去做阻塞 read。本质上是"线程池伪异步"，没有内核支持，线程多了开销大、扩展性差，几乎没人在严肃场景用。

**② Linux 原生 AIO（libaio，`io_submit`/`io_getevents`）**：这是内核真支持的异步，但限制多到劝退：

- **只对 `O_DIRECT` 有效**：buffered IO 用 libaio 会**静默退化为同步阻塞**——你以为异步，其实 `io_submit` 阻塞了。这是最大的坑。
- **仍可能阻塞**：即使 O_DIRECT，`io_submit` 在元数据读取、块分配、等待 inode 锁等情况下仍会同步阻塞，"异步"名不副实。
- **接口僵硬**：只支持读写，不支持 `fsync`、`accept`、`recv` 等；每次提交/收割仍是系统调用，无法零拷贝共享。

```c
// libaio 的尴尬：必须 O_DIRECT，否则退化同步
int fd = open("data", O_RDONLY | O_DIRECT);   // 不加 O_DIRECT 就同步阻塞
io_context_t ctx = 0;
io_setup(128, &ctx);
struct iocb cb, *cbs[1] = { &cb };
io_prep_pread(&cb, fd, buf, 4096, 0);          // buf 必须块对齐
io_submit(ctx, 1, cbs);                        // 可能仍阻塞
struct io_event ev[1];
io_getevents(ctx, 1, 1, ev, NULL);             // 收割
```

### 1.2 io_uring 想解决的根本问题

io_uring 的设计目标，是一次性补齐旧 AIO 的所有短板：

1. **真异步、通用**：buffered/direct 都异步；不止读写，几乎所有阻塞系统调用（fsync、accept、connect、recv/send、openat、close、统计、超时……）都能异步提交。
2. **少系统调用 / 零系统调用**：用共享 ring 提交与收割，批量提交一次 `io_uring_enter`；开 SQPOLL 后内核线程轮询，提交端零系统调用。
3. **零拷贝共享**：SQ/CQ 环通过 mmap 在内核与用户态间共享内存，提交描述符不需要每次拷贝。
4. **高级特性**：链式请求（一个完成触发下一个）、固定缓冲/固定文件（省去每次的引用计数与映射）、multishot（一次注册多次产出）。

---

## 第二章 io_uring 架构：SQ / CQ 双环

### 2.1 两个环与三块共享内存

io_uring 的核心是两个环形队列，通过 `mmap` 在用户态与内核共享：

- **SQ（Submission Queue，提交队列）**：用户态往里放"我要做什么 IO"。
- **CQ（Completion Queue，完成队列）**：内核往里放"这些 IO 做完了，结果是…"。

```
       用户态                                   内核
  ┌──────────────────┐                  ┌──────────────────┐
  │  应用             │                  │  io_uring 子系统  │
  │                  │   mmap 共享       │                  │
  │  填 SQE ─────────┼──► SQ Ring ──────┼─► 取 SQE 执行 IO  │
  │  推进 SQ tail     │   (索引数组+SQE) │                  │
  │                  │                  │  IO 完成          │
  │  读 CQE ◄─────────┼── CQ Ring ◄──────┼─ 填 CQE          │
  │  推进 CQ head     │                  │  推进 CQ tail     │
  └──────────────────┘                  └──────────────────┘
```

实际有三块 mmap 区域：SQ 环的元数据+索引数组、CQ 环（含 CQE 数组）、SQE 数组本身。SQ 环里存的是**指向 SQE 数组的索引**（多一层间接，便于复用/重排 SQE）。

### 2.2 SQE 与 CQE

```c
// SQE：提交队列条目——描述"做什么 IO"
struct io_uring_sqe {
    __u8  opcode;        // IORING_OP_READ / WRITE / FSYNC / ACCEPT / RECV / SEND ...
    __u8  flags;         // IOSQE_IO_LINK / IOSQE_FIXED_FILE / IOSQE_ASYNC ...
    __u16 ioprio;
    __s32 fd;            // 操作的 fd
    __u64 off;           // 偏移
    __u64 addr;          // 缓冲区地址（或其他参数）
    __u32 len;           // 长度
    __u64 user_data;     // ★ 用户自定义，原样回到 CQE，用于匹配请求
    // ... 联合体随 opcode 变化
};

// CQE：完成队列条目——描述"结果"
struct io_uring_cqe {
    __u64 user_data;     // ★ 与提交时一致，用来认领是哪个请求
    __s32 res;           // 结果：>=0 是返回值(如读到字节数)，<0 是 -errno
    __u32 flags;         // IORING_CQE_F_MORE(multishot还有更多) 等
};
```

`user_data` 是灵魂：你提交时塞一个标识（指针、请求 ID），完成时它原样出现在 CQE 里，你据此知道"是哪个请求完成了、结果怎样"。这让一个环里同时跑成千上万个不同请求、乱序完成也能各自认领。

### 2.3 三个系统调用

io_uring 整个接口只有三个系统调用（平时几乎只用前两个，liburing 还会替你封装）：

```c
// 1. 建立 io_uring 实例，返回一个 fd；entries=环大小
int io_uring_setup(unsigned entries, struct io_uring_params *p);

// 2. 提交 SQ 中的请求 / 等待 CQ 完成（核心）
int io_uring_enter(unsigned fd, unsigned to_submit,
                   unsigned min_complete, unsigned flags, ...);

// 3. 注册/注销资源（固定缓冲、固定文件、eventfd 等），见第三章
int io_uring_register(unsigned fd, unsigned opcode, void *arg, unsigned nr);
```

`io_uring_setup` 返回后，用户态用 `mmap` 把三块共享区映射进来，拿到 SQ/CQ 的头尾指针和 SQE 数组——这些繁琐步骤 **liburing 库一行封装**，所以实战几乎不直接碰裸系统调用。

---

## 第三章 提交与完成：submit、wait、批量、链式、固定资源

### 3.1 用 liburing 写一个最小读文件程序

liburing 是官方用户态库，把环管理、内存屏障、索引推进都封装好了。一个异步读文件：

```c
#include <liburing.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    struct io_uring ring;
    io_uring_queue_init(8, &ring, 0);          // 建实例，环深 8

    int fd = open(argv[1], O_RDONLY);
    char buf[4096];

    // 1) 取一个 SQE，填一个 read 请求
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);  // 读 fd 偏移0，4096字节
    io_uring_sqe_set_data(sqe, (void *)0x1234);        // user_data

    // 2) 提交并等待至少 1 个完成
    io_uring_submit(&ring);                    // 触发 io_uring_enter
    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);            // 阻塞直到有完成

    // 3) 处理结果
    if (cqe->res < 0)
        fprintf(stderr, "read error: %s\n", strerror(-cqe->res));
    else
        printf("read %d bytes, user_data=%p\n", cqe->res, io_uring_cqe_get_data(cqe));

    io_uring_cqe_seen(&ring, cqe);             // 标记已消费，推进 CQ head
    io_uring_queue_exit(&ring);
    return 0;
}
// 编译：gcc x.c -luring -o x
```

注意 `cqe->res` 的语义：`>=0` 是读到的字节数（同 read 返回值），`<0` 是 `-errno`。这与同步系统调用的错误约定一致，只是错误码取了负。

### 3.2 批量提交：io_uring 的核心红利

io_uring 真正的威力在**批量**。准备多个 SQE 再一次 `io_uring_submit`，N 个 IO 一次系统调用：

```c
for (int i = 0; i < N; i++) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fds[i], bufs[i], 4096, offs[i]);
    io_uring_sqe_set_data(sqe, &reqs[i]);
}
io_uring_submit(&ring);                  // ★ N 个请求，1 次 io_uring_enter

// 批量收割
struct io_uring_cqe *cqe;
unsigned head;
int count = 0;
io_uring_for_each_cqe(&ring, head, cqe) {  // 遍历所有已就绪的 CQE
    struct req *r = io_uring_cqe_get_data(cqe);
    handle(r, cqe->res);
    count++;
}
io_uring_cq_advance(&ring, count);         // 一次推进 head
```

提交多、收割多，把系统调用摊薄到接近零。这就是 io_uring 在高 IOPS 下吊打 libaio/epoll 的根本原因。

### 3.3 链式请求 IOSQE_IO_LINK

`IOSQE_IO_LINK` 让一组 SQE 形成**有序链**：前一个成功完成后才执行后一个，任一失败则链上后续被取消。常用于"读完文件 → 接着写到另一个 fd""accept → recv"等天然有依赖的序列，省去用户态在两次完成之间的来回。

```c
// 链式：先 write，成功后再 fsync
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe, fd, buf, len, off);
sqe->flags |= IOSQE_IO_LINK;               // ★ 与下一个 SQE 链起来

sqe = io_uring_get_sqe(&ring);
io_uring_prep_fsync(sqe, fd, 0);           // write 成功后才 fsync
io_uring_submit(&ring);
```

还有 `IORING_OP_LINK_TIMEOUT` 给链上某步加超时、`IOSQE_IO_DRAIN`（屏障，等之前全部完成）等编排原语。

### 3.4 固定缓冲与固定文件（register）

每次 IO，内核都要把用户缓冲区的页**临时 pin 住**（防止换出/迁移）并建立映射，IO 完再解除——高频小 IO 下这是固定开销。**注册固定缓冲（fixed buffers）** 一次性把一批缓冲区 pin 好，之后用 `IORING_OP_READ_FIXED` 直接引用其索引，省去每次 pin/unpin：

```c
struct iovec iov[2] = {
    { .iov_base = buf0, .iov_len = 4096 },
    { .iov_base = buf1, .iov_len = 4096 },
};
io_uring_register_buffers(&ring, iov, 2);  // 注册，一次 pin 住

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, buf0, 4096, 0, /*buf_index=*/0); // 用固定缓冲0
```

**注册固定文件（fixed files）** 同理：把一批 fd 注册成数组，用 `IOSQE_FIXED_FILE` + 索引引用，省去每次 IO 对 `struct file` 的引用计数（`fget`/`fput`）开销——对短连接海量、或对同一组 fd 反复 IO 的场景收益明显：

```c
int fds[3] = { fd0, fd1, fd2 };
io_uring_register_files(&ring, fds, 3);
sqe->flags |= IOSQE_FIXED_FILE;
sqe->fd = 0;   // ← 这里是注册数组的索引，不是真 fd
```

---

## 第四章 SQPOLL：内核轮询，零系统调用提交

### 4.1 SQPOLL 的思想

默认情况下，填好 SQE 后仍要调一次 `io_uring_submit`（即 `io_uring_enter`）来"踢"内核去取请求——这还是一次系统调用。**`IORING_SETUP_SQPOLL`** 让内核启动一个**专门的内核线程（sqthread）持续轮询 SQ 环**：用户态只要把 SQE 写入环、推进 tail 指针（一次内存写，带屏障），内核线程就会自己发现并处理，**提交端零系统调用**。

```
默认模式：
  填 SQE → io_uring_submit()[系统调用] → 内核取走执行

SQPOLL 模式：
  填 SQE → 推进 SQ tail（仅内存写）→ 内核 sqthread 轮询自动发现执行
           ↑ 无系统调用！
```

```c
struct io_uring_params p;
memset(&p, 0, sizeof(p));
p.flags = IORING_SETUP_SQPOLL;
p.sq_thread_idle = 2000;   // 空闲 2000ms 后内核线程休眠
io_uring_queue_init_params(8192, &ring, &p);

// 之后提交：填 SQE 后 io_uring_submit 在 SQPOLL 下通常变成纯内存操作，
// 仅当内核线程已休眠时才需要一次 io_uring_enter 唤醒它（liburing 自动处理）。
```

### 4.2 SQPOLL 的代价与取舍

SQPOLL 不是免费午餐：

- **烧一个 CPU 核**：sqthread 忙轮询会占满一个核（直到 `sq_thread_idle` 空闲后休眠）。低负载时纯浪费，高负载、追求极致延迟时才划算。
- **可绑核**：`IORING_SETUP_SQ_AFF` + `sq_thread_cpu` 把 sqthread 钉在指定 CPU，避免和业务线程抢核、利于缓存局部性。
- **权限**：早期 SQPOLL 需要特权；较新内核普通用户在受限条件下也可用，但容器/受限环境常被策略禁止。

适用场景：超高 IOPS（NVMe 存储引擎）、超低延迟交易系统——用一个核的轮询换掉所有提交系统调用，值得。普通中低负载服务用默认模式即可。

| 模式 | 提交系统调用 | CPU 成本 | 适用 |
|---|---|---|---|
| 默认 | 每批 1 次 io_uring_enter | 低 | 通用 |
| SQPOLL | 0（内核线程轮询） | 烧 1 核 | 超高 IOPS / 超低延迟 |

---

## 第五章 网络 io_uring：accept / recv / send、multishot、zero-copy

### 5.1 io_uring 也能做网络

io_uring 不止磁盘——`IORING_OP_ACCEPT`、`RECV`、`SEND`、`CONNECT`、`RECVMSG`、`SENDMSG` 让整条网络路径异步化。一个 echo 连接的处理：

```c
// 异步 accept
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_accept(sqe, listen_fd, NULL, NULL, 0);
io_uring_sqe_set_data(sqe, make_ctx(ACCEPT));
io_uring_submit(&ring);

// 在 CQE 处理中：res 就是新连接 fd，接着给它提交 recv
io_uring_wait_cqe(&ring, &cqe);
int conn_fd = cqe->res;
struct io_uring_sqe *r = io_uring_get_sqe(&ring);
io_uring_prep_recv(r, conn_fd, buf, sizeof(buf), 0);
```

相比 epoll，这里没有"先 epoll_wait 报就绪、再 recv"的两步——你直接提交"recv 这个连接"，内核完成（含数据已拷进你的 buf）后给一个 CQE。**就绪通知合并成了完成通知**，省一轮系统调用。

### 5.2 multishot：一次注册，多次产出

普通请求是"一次提交一次完成"。**multishot** 让一次提交持续产出多个 CQE，特别适合"重复同类事件"：

- **`IORING_OP_ACCEPT` multishot**：提交一次 multishot accept，之后**每来一个新连接就产出一个 CQE**，无需为每个连接重新提交 accept SQE。海量短连接场景大幅减少提交次数。
- **multishot recv**：一次提交，连接上**每次到数据就产出一个 CQE**，配合**注册缓冲池（provided buffers / buffer ring）**，内核从你预注册的缓冲池里自动挑一块装数据，CQE 告诉你用了哪块——避免"提交 recv 时还不知道往哪存"的难题。

```c
// multishot accept：提交一次，持续产出新连接
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);
io_uring_submit(&ring);
// 此后每个新连接 = 一个 CQE，cqe->flags 含 IORING_CQE_F_MORE 表示"还会有更多"
```

CQE 的 `flags` 带 `IORING_CQE_F_MORE` 表示"这个 multishot 还活着，后面还有"；若该位清零则表示这条 multishot 结束（需重新提交）。

### 5.3 zero-copy send

`IORING_OP_SEND_ZC`（zero-copy send）让发送大块数据时**内核直接 DMA 用户缓冲区**，跳过"用户 buffer → 内核 socket buffer"的那次拷贝。代价是：缓冲区在数据真正发出（网卡确认）前不能复用，所以会产生**两个通知**——一个表示"已提交/数据已被内核接管"，一个（带 `IORING_CQE_F_NOTIF`）表示"缓冲区可以重用了"。适合大报文、高吞吐发送（如文件服务器、视频流）。

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_send_zc(sqe, conn_fd, big_buf, big_len, 0, 0);
// 收到两个 CQE：发送结果 + 缓冲区释放通知(F_NOTIF)，后者到了才能重用 big_buf
```

### 5.4 io_uring vs epoll 性能对比（定性）

| 维度 | epoll | io_uring（网络） |
|---|---|---|
| 模型 | 就绪通知（同步拷贝） | 完成通知（异步，内核拷贝） |
| 一次 IO 系统调用 | epoll_wait + recv（2） | 合并，批量摊薄到 ~0 |
| accept 海量短连接 | 每连接 accept | multishot 一次提交持续产出 |
| 大块发送拷贝 | 有一次拷贝 | zero-copy 可省 |
| 磁盘 + 网络混合 | 磁盘要另开线程池 | 同一环统一处理 |
| 成熟度 / 生态 | 极成熟、到处可用 | 成熟但部分环境被禁 |

**量级而非精确**：在连接极多、IO 极密集、或网络磁盘混合的负载下，io_uring 相比 epoll 通常能显著降低系统调用次数和 CPU 占用、提升吞吐——具体收益高度依赖负载形态，需实测。中低负载下二者差距不大，epoll 的简单与普适仍有价值（呼应 L08）。

---

## 第六章 生态与安全：liburing、各语言绑定、io_uring_disabled

### 6.1 liburing 与语言绑定

- **liburing**（Jens Axboe 维护）：C 用户态库，封装环管理、内存屏障、prep helper、buffer ring。**实战标准入口**，几乎没人直接写裸系统调用。
- **各语言绑定**：
  - Rust：`tokio-uring`、`io-uring` crate、`glommio`（thread-per-core 运行时）。
  - C++：`liburingcxx`、各类协程框架接入。
  - Go：Go runtime 的网络默认仍是 epoll（netpoller），社区有 io_uring 实验性库；Go 团队对把 io_uring 引入标准 runtime 持谨慎态度（安全 + 跨平台 + 调度集成复杂）。
  - 其他：Node、Java（部分通过 JNI/Netty 实验）等均有探索。
- **落地案例**：存储引擎、数据库、对象存储、代理网关、`fio`（自带 io_uring 引擎）、部分 NVMe-oF / SPDK 旁路方案的对照基准。

### 6.2 安全争议：为什么云和容器默认收紧

io_uring 强大也带来了**攻击面**：它能在内核里异步执行大量操作（含 SQPOLL 内核线程），历史上出现过若干内核漏洞（提权、绕过 seccomp 等）。关键问题：

- **seccomp 绕过隐患**：传统沙箱用 seccomp 过滤系统调用，但 io_uring 提交的操作不是常规系统调用路径，早期可能绕过基于 syscall 的过滤策略（后续内核加了 `IORING_OP` 级别的限制能力与 seccomp 协同改进，但历史包袱让安全团队警惕）。
- **内核异步执行面大**：opcode 众多、链式/multishot 逻辑复杂，是漏洞高发区。

于是出现了**全局收紧开关** `io_uring_disabled`（sysctl）：

```bash
$ sysctl kernel.io_uring_disabled
kernel.io_uring_disabled = 0   # 0=允许；1=非特权禁用(需CAP)；2=全局完全禁用

# 取值语义：
#   0：所有进程可用（默认/宽松）
#   1：仅有 CAP_SYS_ADMIN（或属于 io_uring_group）的进程可用，其余 io_uring_setup 返回 EPERM
#   2：完全禁用，任何 io_uring_setup 都失败
```

2026 年的现实：
- **多家云厂商、托管 K8s、安全发行版默认把 io_uring 收紧（设为 1 或 2）**，Google、部分容器平台公开建议在不需要时禁用。
- **容器运行时 / 安全基线**（如部分 OCI seccomp 默认 profile）倾向限制或禁用 io_uring 相关系统调用。
- **后果**：你的高性能 io_uring 程序在本机跑得飞起，上了某些云/容器却 `io_uring_setup` 返回 `EPERM`/`ENOSYS`。

**工程建议**：依赖 io_uring 的服务必须**做能力探测 + 优雅降级到 epoll**。启动时尝试 `io_uring_queue_init`，失败则回退 epoll 路径。liburing 程序要检查 `io_uring_setup` 的返回，把 io_uring 当"可选加速"而非"硬依赖"，除非你完全掌控部署环境。

```c
struct io_uring ring;
if (io_uring_queue_init(256, &ring, 0) < 0) {
    // io_uring 不可用（被 io_uring_disabled 禁用 / 内核太老 / 容器限制）
    fprintf(stderr, "io_uring unavailable, falling back to epoll\n");
    run_epoll_loop();    // ★ 优雅降级
} else {
    run_iouring_loop(&ring);
}
```

---

## 生产实践

1. **永远准备 epoll 降级路径**：因 `io_uring_disabled` 在云/容器普遍收紧，把 io_uring 作为"可选加速层"，启动探测失败即回退 epoll，别让它成为部署硬依赖。

2. **批量提交、批量收割**：io_uring 的红利在批处理。攒一批 SQE 一次 submit、用 `io_uring_for_each_cqe` 批量收割 + 一次 `cq_advance`，把系统调用摊到接近零，别一个 IO 一次 submit。

3. **SQPOLL 仅用于高负载且能绑核**：SQPOLL 烧一个核，只在 IOPS 极高 / 延迟极敏感时启用，并用 `SQ_AFF` 绑到隔离核（配合 `isolcpus`），避免和业务抢核。中低负载用默认模式。

4. **固定缓冲 + 固定文件降固定开销**：对同一组 fd / 缓冲区高频 IO 的场景（存储引擎、连接池），register fixed buffers/files，省掉每次的 pin/unpin 与 fget/fput。

5. **网络用 multishot + buffer ring**：海量连接 accept 用 multishot accept、收数据用 multishot recv + provided buffer ring，大块发送用 zero-copy send，最大化减少提交与拷贝。

6. **fio 做基准对照**：用 `fio --ioengine=io_uring` 对照 `--ioengine=libaio`/`psync` 在你的真实盘上测 IOPS/延迟，量化收益再决定是否上 io_uring，别凭感觉。

---

## 陷阱清单

1. **io_uring_setup 在生产返回 EPERM / ENOSYS**
   - 现象：本机正常，上云/容器后 io_uring 初始化失败。
   - 原因：`kernel.io_uring_disabled` 被设为 1（需 CAP）或 2（全禁），或 seccomp/容器策略拦截。
   - 修法：代码做探测 + 优雅降级 epoll；确需 io_uring 则申请放开 sysctl/调整 seccomp profile，并评估安全风险。

2. **以为 libaio 异步，实际同步阻塞**
   - 现象：用 libaio 但吞吐/延迟没改善，`io_submit` 耗时高。
   - 原因：未用 `O_DIRECT`，libaio buffered 退化为同步；或遇元数据/分配阻塞。
   - 修法：迁移到 io_uring（buffered 也真异步）；非用 libaio 不可时严格 O_DIRECT + 对齐。

3. **CQE 没消费导致 CQ 溢出**
   - 现象：部分完成事件丢失，或 `cqe->res` 行为异常，`cq_overflow` 计数增长。
   - 原因：提交远多于收割，CQ 环被填满，完成事件溢出（进溢出列表或丢弃）。
   - 修法：及时 `io_uring_cqe_seen` / `cq_advance` 推进 head；CQ 环开大（`IORING_SETUP_CQSIZE`）；控制在途请求数不超过环容量。

4. **SQPOLL 烧满一个核还以为是 bug**
   - 现象：开了 SQPOLL 后某 CPU 持续 100%。
   - 原因：sqthread 忙轮询的预期行为（idle 超时前不休眠）。
   - 修法：低负载别用 SQPOLL；用 `sq_thread_idle` 让它及时休眠；绑核隔离；按需关闭。

5. **固定缓冲生命周期搞错**
   - 现象：注册缓冲后偶发数据错乱 / 崩溃。
   - 原因：注册的缓冲区在注册期间被 free / realloc / 移动，或在 IO 在途时改动内容。
   - 修法：注册的缓冲区生命周期必须覆盖整个使用期，IO 在途不得释放/迁移；用完 `io_uring_unregister_buffers`。

6. **zero-copy send 提前复用缓冲区**
   - 现象：对端偶发收到错乱/旧数据。
   - 原因：`SEND_ZC` 下数据真正发出前就改了/复用了发送缓冲区。
   - 修法：等带 `IORING_CQE_F_NOTIF` 的释放通知到达后才复用缓冲区；理解 zero-copy 的两次 CQE 语义。

7. **multishot 结束未重新提交**
   - 现象：accept/recv 突然不再产出 CQE，连接停滞。
   - 原因：CQE 的 `IORING_CQE_F_MORE` 被清零（multishot 因错误/资源耗尽终止）却没重新提交。
   - 修法：检查每个 multishot CQE 的 `F_MORE` 位，清零时重新 prep 提交。

8. **内核太老缺特性**
   - 现象：某 opcode（如 multishot recv、send_zc）返回 `-EINVAL`。
   - 原因：该特性在当前内核版本未实现（io_uring 特性按版本逐步加入）。
   - 修法：用 `io_uring_get_probe` 探测 opcode 支持；按内核版本走特性开关；最低支持版本写进部署文档。

---

## 2026 现状

- **io_uring 已是高性能 I/O 首选**：磁盘（存储引擎、数据库 WAL + O_DIRECT，呼应 L07）、网络（multishot accept/recv、zero-copy send）、混合负载都成熟，liburing 接口稳定，`fio` 内置引擎。
- **网络 io_uring 成熟**：multishot、provided buffer ring、zero-copy send/recv 落地，单环统一收发，省掉 epoll 的"就绪→再 recv"两步。
- **安全收紧成常态**：`kernel.io_uring_disabled` 在多家云、托管 K8s、安全发行版默认设为 1 或 2；容器 seccomp 基线常限制。**"做能力探测 + 降级 epoll" 是 2026 年写 io_uring 程序的必修课**。
- **内核侧持续硬化**：opcode 级权限、与 seccomp 更好的协同、注册资源的生命周期检查等安全改进持续推进，但历史 CVE 让安全团队保持保守默认。
- **与调度/cgroup 协同**：io_uring 的内核 worker、SQPOLL 线程纳入 cgroup 资源核算（CPU、io.max，呼应 L17）；SQPOLL 绑核 + CPU 隔离是低延迟部署的标配。
- **生态分层**：底层 liburing/C、Rust 的 tokio-uring/glommio 成 thread-per-core 高性能服务的基石；Go 仍以 epoll netpoller 为默认，io_uring 在 Go 多为实验性。

---

## 练习题

1. 为什么说 POSIX AIO 是"用户态线程池伪异步"、libaio 是"半残的真异步"？libaio 的两个致命限制是什么，io_uring 各自如何解决？

2. 画出 io_uring 的 SQ/CQ 双环与三块 mmap 共享内存，解释 `user_data` 字段为什么是"灵魂"，以及它如何支持成千上万个请求乱序完成后各自认领。

3. io_uring 一共几个系统调用？分别做什么？在默认模式与 SQPOLL 模式下，"提交 N 个 IO"各需要多少次系统调用？

4. `IOSQE_IO_LINK`（链式）、注册固定缓冲（fixed buffers）、注册固定文件（fixed files）各解决什么问题、各省掉了哪部分开销？

5. SQPOLL 的工作原理是什么？它的代价是什么？在什么负载下值得开、什么负载下纯属浪费？为什么要给 sqthread 绑核？

6. 网络 io_uring 相比 epoll，在"accept 海量短连接"和"发送大块数据"两个场景分别用什么特性优化（multishot / zero-copy），它们各自的注意点是什么？

7. **实战**：用 liburing 写一个最小程序，批量异步读取 8 个文件的前 4KB，用 `user_data` 区分是哪个文件，批量收割并打印每个的读取字节数（给出可编译的 C 代码框架）。

8. **排障**：一个依赖 io_uring 的服务在自建机房跑得很好，迁到某托管 K8s 后启动即崩，日志显示 io_uring 初始化失败。请给出排查步骤（如何确认是 `io_uring_disabled` 还是 seccomp、内核版本），以及一个让服务"在禁用环境下也能跑"的代码改造方案。
