# 精通内核同步与 futex：原子操作、自旋锁、RCU、内存屏障、futex、惊群

> 课程编号：L15
> 路线图来源：Linux · 模块五 IPC 与内核同步
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：futex2 系统调用演进、RCU 在内核广泛使用、PI futex 成熟）

---

## 引言：你写的 `mutex.lock()`，在内核里到底是什么

打开任何一门语言的并发教程，第一课都是「加锁」：

```go
var mu sync.Mutex
mu.Lock()
// 临界区
mu.Unlock()
```

```c
pthread_mutex_lock(&m);
/* 临界区 */
pthread_mutex_unlock(&m);
```

这两行代码每秒在你的服务里执行几百万次。但极少有人能回答一个看似简单的问题：**`Lock()` 在没有竞争时，会不会陷入内核？**

直觉上「加锁要找操作系统」，所以应该陷入内核。但如果真的每次 `Lock`/`Unlock` 都做一次系统调用，那加锁开销就是几百纳秒到微秒级——对一个无竞争的锁来说贵得离谱。

**真相是：现代 `pthread_mutex` 在无竞争时完全在用户态完成，一次系统调用都不做。** 它靠一条原子的 CAS（compare-and-swap）指令把锁状态从「空闲」改成「持有」，成功就直接进临界区，整个过程纯用户态、几纳秒。**只有当真的有竞争——锁已被别人持有、当前线程需要睡眠等待时——才陷入内核**，调用一个叫 **futex**（fast userspace mutex）的系统调用，让内核把线程挂起、等持锁者释放时再唤醒。

这就是本篇要彻底讲透的东西：从最底层的「为什么需要同步」（竞态、内存可见性、缓存一致性），到原子操作与 CAS，到内核里的各种锁（自旋锁、读写锁、seqlock、mutex），到读多写少的杀手锏 RCU，到让所有这些正确工作的内存屏障，最后落到 futex——把用户态锁和内核等待队列粘在一起的那块「胶水」。结尾再讲一个贯穿网络/IPC/同步的经典问题：惊群（thundering herd）。

这是整个 Linux 课程里最硬核的一篇。但理解它之后，你看 `sync.Mutex` 源码、看内核锁、看无锁数据结构，都会有「原来如此」的通透感。

---

## 第一章 并发的根源：竞态、可见性与缓存一致性

### 1.1 临界区与竞态条件

最经典的例子：两个线程对同一个全局变量 `counter++` 各执行 100 万次，结果几乎从不是 200 万。

因为 `counter++` 不是一条指令，它是「读-改-写」三步：

```
load   counter → reg      ; 读
add    reg, 1             ; 改
store  reg → counter      ; 写
```

两个线程的这三步可以任意交错。若线程 A 读到 counter=5，还没写回，线程 B 也读到 5，各自加到 6 写回——两次自增只涨了 1。这就是**竞态条件（race condition）**：结果依赖于执行流的相对时序。需要保护的、不能被并发交错执行的代码段叫**临界区（critical section）**。

### 1.2 可见性：写了不等于别人能看见

竞态还不是全部。即使没有交错，在 SMP（多核）系统上还有**内存可见性**问题：CPU 0 写了一个变量，CPU 1 不一定立刻能读到新值。

```c
/* 初始 ready=0, data=0 */
/* CPU 0 */                  /* CPU 1 */
data  = 42;                  while (ready == 0) ;   /* 自旋等 */
ready = 1;                   use(data);             /* 期望读到 42 */
```

直觉上 CPU 1 看到 `ready==1` 后，`data` 一定是 42。但**这在弱内存序架构（如 ARM）上可能失败**，甚至在 x86 上因编译器重排也可能出错。原因有二：

1. **编译器重排**：编译器为优化可能把 `data=42` 和 `ready=1` 调换顺序（它看不出俩变量有依赖）；
2. **CPU 重排 + 缓存延迟**：CPU 可能乱序执行/提交写操作，且写先进 store buffer，其它核要等缓存一致性协议传播才看得到。

所以「能不能看见」和「按什么顺序看见」是两个独立于「会不会交错」的问题。解决前者靠内存屏障与原子操作（第二、五章）。

### 1.3 SMP 与缓存一致性（MESI）

现代 CPU 每个核有自己的 L1/L2 cache，多核共享 L3 和内存。**缓存一致性协议（MESI 及其变种）**保证：同一个内存地址在多个核的缓存里不会出现「都认为自己是对的但值不同」。

MESI 四态：

| 状态 | 含义 |
|---|---|
| **M**odified | 本核独占且已修改，内存里是旧的 |
| **E**xclusive | 本核独占且未改，与内存一致 |
| **S**hared | 多核共享只读副本 |
| **I**nvalid | 无效，需重新加载 |

一个核要写某缓存行，必须先让其它核的该行变 Invalid（发 invalidate 消息），拿到独占权（M 态）。**这就是为什么多核频繁写同一缓存行极慢**——缓存行在核间「乒乓」（cache line bouncing），每次写都要协议往返。

**伪共享（false sharing）**是它的恶劣变体：两个无关变量恰好落在同一条缓存行（通常 64 字节），两个核各写各的变量，却因为同一缓存行而疯狂互相 invalidate，性能暴跌。修法是缓存行对齐填充：

```c
struct counters {
    long a __attribute__((aligned(64)));   /* 各占一条 cache line */
    long b __attribute__((aligned(64)));
};
```

Go 里同理，常见 `_ [64]byte` 填充或 `//go:align`。这是高并发数据结构的必修课。理解缓存一致性，才理解为什么「锁的争用代价」远不止「等锁」，还有缓存行乒乓。

---

## 第二章 原子操作：CAS、内存序与语言原语

### 2.1 原子操作是同步的地基

**原子操作（atomic operation）**是不可被打断、不会被其它核看到「中间态」的操作。硬件层面靠特殊指令实现，x86 上是带 `LOCK` 前缀的指令（`lock xadd`、`lock cmpxchg` 等），它锁住缓存行（或总线）保证读-改-写一体完成。

内核里用 `atomic_t` 类型：

```c
atomic_t v = ATOMIC_INIT(0);
atomic_inc(&v);                  /* 原子自增 */
int old = atomic_fetch_add(3, &v);  /* 原子加 3 并返回旧值 */
atomic_dec_and_test(&v);         /* 减 1，结果为 0 返回 true（引用计数常用） */
```

有了原子自增，第一章的 `counter++` 竞态就解决了——但原子操作能做的远不止计数。

### 2.2 CAS：所有无锁算法的基石

**compare-and-swap（CAS）**是最重要的原子原语：「如果内存里的值等于期望值，就把它改成新值，并告诉我成功与否」。一条指令完成「比较+交换」：

```c
/* 伪代码语义（原子执行） */
bool CAS(int *ptr, int expected, int new) {
    if (*ptr == expected) { *ptr = new; return true; }
    return false;
}
```

x86 上就是 `lock cmpxchg`。CAS 是**无锁（lock-free）编程的核心**——所有不加锁的并发数据结构（无锁队列、无锁栈、引用计数）本质都是「读当前值 → 算新值 → CAS 写回，失败就重试」的循环：

```c
/* 无锁地把 counter 加 1（自旋 CAS 重试） */
int old, new;
do {
    old = atomic_read(&counter);
    new = old + 1;
} while (!atomic_try_cmpxchg(&counter, &old, new));
```

**ABA 问题**是 CAS 的经典陷阱：值从 A 变成 B 又变回 A，CAS 以为「没变过」而成功，但实际中间发生了变化（如指针被释放又重新分配到同地址）。解法是加版本号（带标记的指针）或用 RCU/hazard pointer 延迟释放（见第四章）。

### 2.3 内存序：原子操作也要管「顺序」

原子操作保证「这一个操作」不可分割，但**多个原子操作之间、以及原子操作与普通访存之间的顺序**，需要用**内存序（memory ordering）**约束。C11/C++11 标准化了内存序，Go 的 `sync/atomic` 在语义上对齐顺序一致性。常见内存序：

| 内存序 | 含义 | 开销 |
|---|---|---|
| `relaxed` | 只保证本操作原子，不约束与其它访存的顺序 | 最低 |
| `acquire` | 本读操作之后的访存不能重排到它之前（「获取」） | 中 |
| `release` | 本写操作之前的访存不能重排到它之后（「释放」） | 中 |
| `acq_rel` | 同时具备 acquire + release（如 CAS） | 中 |
| `seq_cst` | 顺序一致，全局唯一总顺序，最强 | 最高 |

**acquire/release 配对**是最常用的模式，正好解决第一章 §1.2 的可见性问题：

```c
#include <stdatomic.h>
atomic_int ready = 0;
int data = 0;
/* 生产者 */                          /* 消费者 */
data = 42;                            while (!atomic_load_explicit(
atomic_store_explicit(&ready, 1,          &ready, memory_order_acquire)) ;
    memory_order_release);            use(data);   /* 保证读到 42 */
```

`release` 保证 `data=42` 不会重排到 `ready=1` 之后；`acquire` 保证消费者读到 `ready==1` 后，对 `data` 的读不会重排到前面。**这对配对建立了「happens-before」关系**——这正是 Go 内存模型、Java `volatile`、C++ atomic 背后的同一套理论。

### 2.4 Go 的 sync/atomic 与 C11 atomics 对比

```go
import "sync/atomic"

var counter atomic.Int64          // Go 1.19+ 的类型化原子
counter.Add(1)
old := counter.Load()
swapped := counter.CompareAndSwap(old, old+1)   // CAS
```

Go 的 `sync/atomic` 操作在 Go 内存模型里具有顺序一致语义（详见 golang [G13 sync 包](../golang/G13-精通-Go-sync-包.md) 与 Go 内存模型）。它和 C11 的对应关系：

| 操作 | Go | C11 |
|---|---|---|
| 原子加 | `atomic.Int64.Add` | `atomic_fetch_add` |
| 原子读/写 | `Load` / `Store` | `atomic_load` / `atomic_store` |
| CAS | `CompareAndSwap` | `atomic_compare_exchange_*` |
| 交换 | `Swap` | `atomic_exchange` |

Go 不暴露 relaxed/acquire/release 这些细粒度内存序（默认给你最强保证，降低出错概率），而 C/C++/Rust 暴露全套以追求极致性能。**这是「易用 vs 极致控制」的取舍**。Go 的 `sync.Mutex`、`sync.Once`、channel 底层全建立在这些原子操作 + runtime 的 futex 封装之上。

---

## 第三章 内核锁：自旋锁、读写锁、seqlock、mutex

内核里不能用 `pthread_mutex`（那是用户态库），内核有自己一套锁。核心区别在一个问题：**等锁时是「忙等（自旋）」还是「睡眠」？** 这取决于上下文能不能睡眠。

### 3.1 自旋锁 spinlock：忙等，绝不睡眠

**自旋锁**在拿不到锁时**不睡眠，而是原地空转（自旋）反复检查**，直到锁可用。它的铁律：

- **持有自旋锁期间绝对不能睡眠**（不能调用可能阻塞的函数、不能缺页、不能 `kmalloc(GFP_KERNEL)`）；
- 适用于**临界区极短**的场景（持锁时间 < 一次上下文切换的开销，几十纳秒级）；
- **可以在中断上下文用**（中断处理不能睡眠，所以只能用自旋锁这种不睡眠的锁）。

```c
spinlock_t lock;
spin_lock_init(&lock);

spin_lock(&lock);
/* 极短临界区：改个链表指针、更新个计数 */
spin_unlock(&lock);

/* 中断也会访问这个数据时，必须关本地中断，否则中断里再 spin_lock 会死锁自己 */
unsigned long flags;
spin_lock_irqsave(&lock, flags);
spin_unlock_irqrestore(&lock, flags);
```

**为什么自旋而不睡眠？** 因为如果临界区只有 50ns，而一次「睡眠 + 被唤醒」的上下文切换开销是几微秒，那「忙等 50ns」远比「睡过去再被叫醒」划算。Linux 的自旋锁实现是 **ticket spinlock / qspinlock（队列自旋锁）**，保证 FIFO 公平、减少缓存行乒乓。

**关键约束**：自旋是浪费 CPU 的——只在多核（SMP）有意义。单核内核里 `spin_lock` 退化为只是禁抢占（自旋等谁呢？持锁者在同一个核上，得让它先跑完）。

### 3.2 读写锁 rwlock：读共享、写独占

很多数据**读远多于写**。读写锁允许**多个读者同时持有**，但**写者独占**（写时无读者也无其它写者）：

```c
rwlock_t lock;
read_lock(&lock);   /* 多个读者可同时进 */
/* 只读临界区 */
read_unlock(&lock);

write_lock(&lock);  /* 写者独占 */
/* 读写临界区 */
write_unlock(&lock);
```

读写锁的问题：**写者可能饿死**（读者源源不断，写者永远等不到独占），且读者也要做原子操作维护计数，**读端并非零开销**。读多写少的极致优化是 RCU（第四章），它让读端几乎零开销。

### 3.3 seqlock 顺序锁：写者优先，读者乐观重试

**seqlock** 针对「写很少、写者不能饿死、读者可以重试」的场景（典型：内核里的 `jiffies`、时间戳、`gettimeofday` 数据）。机制是一个**序列号**：

```c
/* 写者：进临界区序列号 +1（变奇数），出临界区再 +1（变偶数） */
write_seqlock(&sl);
/* 更新数据 */
write_sequnlock(&sl);

/* 读者：读前后各取一次序列号，若变了或为奇数则重试 */
unsigned seq;
do {
    seq = read_seqbegin(&sl);
    /* 读数据（可能读到正在被写的半成品，没关系，下面会检测到并重试） */
} while (read_seqretry(&sl, seq));
```

- 读者**完全不阻塞写者**（不像 rwlock 读者会挡住写者），写者随时能写；
- 读者发现「读期间序列号变了（被写过）」就重试；
- **代价**：读者可能重试，且读的数据必须能容忍「读到半成品后丢弃」（不能有副作用、不能解可能失效的指针）。

### 3.4 mutex 与 semaphore：可睡眠的锁

**mutex（内核互斥锁）**：拿不到锁时**睡眠**让出 CPU，适用于**临界区较长、可能睡眠**的进程上下文。**不能在中断上下文用**（中断不能睡眠）。内核 mutex 还有优化：短暂竞争时先**乐观自旋**（owner 在别的核上跑且没睡，就自旋等一会儿，赌它马上释放），等不到才真睡——这叫 adaptive mutex，融合了自旋和睡眠的优点。

**semaphore（内核信号量）**：计数信号量，可允许 N 个持有者，也可睡眠。`down()`（获取，计数减一，为负则睡）/ `up()`（释放，唤醒等待者）。互斥用现在多用 mutex（语义更清晰、有死锁检测、有 owner 概念），semaphore 用于资源计数。

### 3.5 内核锁全景对比

| 锁 | 等待方式 | 中断上下文可用 | 适用临界区 | 读写区分 | 典型场景 |
|---|---|---|---|---|---|
| **spinlock** | 忙等（自旋） | ✅ | 极短（ns 级） | 否 | 中断处理、极短临界区 |
| **rwlock** | 自旋（读写各自） | ✅ | 短，读多 | ✅ | 读多写少的短临界区 |
| **seqlock** | 读者乐观重试 | ✅（读端） | 写极少 | ✅ | jiffies、时间戳 |
| **mutex** | 睡眠（先乐观自旋） | ❌ | 较长，可睡眠 | 否 | 进程上下文一般互斥 |
| **semaphore** | 睡眠 | ❌ | 较长，资源计数 | — | 计数型资源限制 |
| **RCU** | 读端无锁 | ✅（读端） | 读极多写极少 | ✅（极端） | 路由表、链表、配置 |

**选型心法**：能不能睡眠？不能（中断/极短）→ spinlock；能且临界区长 → mutex；读远多于写且读端要极致快 → RCU；写极少且读可重试 → seqlock。

---

## 第四章 RCU 深入：读多写少的杀手锏

### 4.1 RCU 解决什么：让读端「零开销」

内核里大量数据结构是**读极多、写极少**的：路由表（每个包都查，很少更新）、内核模块链表、配置、文件描述符表……对这些，连「读者做原子操作维护计数」都嫌贵。**RCU（Read-Copy-Update）**做到了**读端几乎零开销**——读者不加任何锁、不做任何原子操作、不写任何共享状态，几乎和普通指针解引用一样快。

代价是把复杂度全压到写者那边。核心思想：**写者不原地修改数据，而是「复制一份 → 改副本 → 原子地把指针切过去 → 等所有老读者走完后再释放旧副本」**。这就是 Read-Copy-Update 的字面意思。

### 4.2 读端：rcu_read_lock 几乎什么都不做

```c
/* 读者 */
rcu_read_lock();                         /* 标记进入读临界区——开销近乎为零 */
struct foo *p = rcu_dereference(gp);     /* 安全地读全局指针（带依赖屏障） */
if (p) do_something(p->field);           /* 用 p，期间 p 指向的数据保证有效 */
rcu_read_unlock();                       /* 退出读临界区 */
```

`rcu_read_lock()`/`rcu_read_unlock()` 在经典 RCU 实现里**几乎是空操作**（主要是禁止抢占/标记），不竞争任何锁。`rcu_dereference` 带一个轻量的依赖屏障（保证读到指针后再读它指向的数据，不被重排）。**读端没有任何写操作、没有缓存行乒乓**——这就是它能扛海量读的原因。

### 4.3 写端与宽限期 grace period

```c
/* 写者：发布新版本 */
struct foo *new = kmalloc(...);          /* 1. 复制/构造新数据 */
*new = *old; new->field = newval;        /*    改副本 */
rcu_assign_pointer(gp, new);             /* 2. 原子地切指针（带 release 屏障） */
synchronize_rcu();                       /* 3. 等待「宽限期」——所有老读者退出 */
kfree(old);                              /* 4. 现在没人引用 old 了，安全释放 */
```

关键是第 3 步的**宽限期（grace period）**：指针切换后，可能还有读者持有指向旧数据的指针（它们在切换前就 `rcu_dereference` 了）。写者必须等到**所有「切换前进入读临界区的读者」都退出**，才能释放旧数据。这段等待就是宽限期。

内核怎么知道「所有老读者都走完了」？经典 RCU 的精妙之处：**读临界区内禁止睡眠/阻塞**，所以「每个 CPU 都经历过一次上下文切换（quiescent state，静止态）」就意味着该 CPU 上的所有老读临界区都结束了。当每个 CPU 都报告过一次静止态，宽限期结束。

```
时间 →
写者:  rcu_assign_pointer(gp,new) ──────── synchronize_rcu() 返回 ── kfree(old)
读者A: [rcu_read_lock ... 用old ... unlock]                  ↑
读者B:        [rcu_read_lock ... 用new ... unlock]           │
                            └── 所有持 old 的读者退出 ───────┘
                                （宽限期 grace period）
```

两种释放方式：`synchronize_rcu()`（同步等待，写者阻塞直到宽限期结束）或 `call_rcu(&old->rcu, free_cb)`（异步，注册回调，宽限期结束后内核帮你调）。高频写路径用 `call_rcu` 避免阻塞。

### 4.4 典型场景与 RCU 链表

RCU 保护的链表插入/删除特别能体现「读者无锁、写者小心」：

```c
/* 删除一个节点（写者持写锁防写写冲突，但读者无需任何锁） */
list_del_rcu(&node->list);     /* 从链表摘除，但不释放——可能还有读者在遍历 */
synchronize_rcu();             /* 等宽限期 */
kfree(node);                   /* 安全释放 */

/* 读者遍历，无锁 */
rcu_read_lock();
list_for_each_entry_rcu(p, &head, list) { use(p); }
rcu_read_unlock();
```

经典应用：网络路由表/FIB（[L11 网络协议栈](./L11-精通-Linux-网络协议栈.md) 每个包都查路由）、`dcache`（dentry cache，[L07 VFS](./L07-精通-VFS-与文件系统.md)）、SELinux 策略、内核模块链表。**RCU 是 Linux 可扩展性（scalability）的核心武器之一**——它让多核读访问几乎线性扩展，没有锁争用瓶颈。

### 4.5 RCU 的限制

- **读临界区内不能阻塞**（经典 RCU；可睡眠的变体 SRCU 允许，但有额外开销）；
- **不适合写多场景**：写者复制 + 等宽限期成本高，写频繁就退化；
- 释放有延迟（宽限期），内存回收不是即时的；
- 写者之间仍需自己的同步（RCU 只解决读写并发，不解决写写并发）。

用户态也有 RCU（liburcu），但内核 RCU 是它的主战场。

---

## 第五章 内存屏障：对抗重排

### 5.1 三种重排

前面反复提到「重排」，它来自三个层次：

1. **编译器重排**：编译器为优化调整指令顺序（在单线程语义不变的前提下）；
2. **CPU 乱序执行**：现代 CPU 乱序执行指令、推测执行；
3. **内存系统重排**：写进 store buffer 延迟提交、各核看到写的顺序不同（取决于架构内存模型）。

单线程下这些重排都「无害」（保证单线程结果不变），但**多线程下会暴露**——一个线程精心安排的写顺序，另一个线程看到的可能是乱的。

### 5.2 编译器屏障 barrier()

最弱的屏障，只阻止**编译器**重排，不影响 CPU：

```c
#define barrier() __asm__ __volatile__("" ::: "memory")
```

Linux 内核里 `barrier()`、C 里 `atomic_signal_fence`、给变量加 `volatile` 都属此类。它**不生成任何 CPU 指令**，只是告诉编译器「这里前后的内存访问不许跨越重排」。**单靠它在 SMP 上不够**——它管不住 CPU 和缓存。

### 5.3 CPU 内存屏障

内核提供（对应硬件屏障指令）：

| 屏障 | 作用 | x86 大致对应 |
|---|---|---|
| `smp_mb()` | 全屏障：之前的读写都先于之后的读写完成 | `mfence`（x86 强序，多数已隐含） |
| `smp_rmb()` | 读屏障：约束读-读顺序 | 编译器屏障（x86 读不重排） |
| `smp_wmb()` | 写屏障：约束写-写顺序 | 编译器屏障（x86 写不重排） |
| `smp_load_acquire()` | acquire 语义的读 | — |
| `smp_store_release()` | release 语义的写 | — |

**x86 是强内存序（TSO）**，硬件已保证「读不和读重排、写不和写重排」，所以 `smp_rmb`/`smp_wmb` 在 x86 上常退化为编译器屏障；但 **ARM/RISC-V 是弱内存序**，必须真发屏障指令。**写跨平台代码绝不能依赖 x86 的强序「碰巧能跑」**——同一段代码挪到 ARM 服务器（如 Graviton）就可能挂。

### 5.4 acquire/release 配对：可见性的正解

回到第一章 §1.2 的可见性问题，内核里的正确写法：

```c
/* 写者 */                                 /* 读者 */
data = 42;                                 if (smp_load_acquire(&ready))
smp_store_release(&ready, 1);                  use(data);   /* 保证看到 42 */
```

`store_release` 保证它**之前的所有写（data=42）**对「看到这个 release 的读者」可见；`load_acquire` 保证它**之后的所有读**能看到 release 方在 release 之前的写。这对配对就是「正确发布数据」的标准范式——RCU 的 `rcu_assign_pointer`/`rcu_dereference` 内部就是 release/acquire（依赖屏障）。

```
smp_store_release(&ready, 1)  ──happens-before──►  smp_load_acquire(&ready) 返回 1
        ▲                                                    │
   之前的 data=42 ──────────────保证可见───────────────────► use(data) 读到 42
```

**记住一句话**：原子操作保证「操作本身不可分割」，内存屏障保证「操作之间的顺序与可见性」。两者缺一不可，无锁编程的 bug 几乎都出在后者。

---

## 第六章 futex：用户态锁的内核支撑

### 6.1 futex 是什么：快路径在用户态，慢路径才进内核

回到开篇的问题。`pthread_mutex` 的设计目标是：**无竞争时零系统调用，有竞争时才求助内核**。实现这一点的内核机制就是 **futex（fast userspace mutex）**。

核心 idea：锁的状态是用户态内存里的一个 32 位整数（futex word）。

- **加锁快路径**：用 CAS 把这个整数从「0（空闲）」改成「1（持有）」。成功 → 直接进临界区，**纯用户态，无系统调用**。
- **加锁慢路径**：CAS 失败（锁被占）→ 调用 `futex(FUTEX_WAIT)` 系统调用，把当前线程挂到内核为这个 futex word 地址维护的等待队列上，睡眠。
- **解锁快路径**：CAS 把整数改回 0。如果发现「没有等待者」标记 → **纯用户态，无系统调用**。
- **解锁慢路径**：若有等待者 → 调用 `futex(FUTEX_WAKE)` 唤醒一个（或多个）等待线程。

```
线程 A 加锁 (无竞争):  CAS 0→1 成功 ───────────────► 进临界区  [纯用户态]
线程 B 加锁 (有竞争):  CAS 0→1 失败 ──► futex(WAIT) ──► 内核挂起睡眠
线程 A 解锁 (有等待者): CAS 1→0 ──────► futex(WAKE) ──► 内核唤醒 B
```

**这就是为什么无竞争的锁几乎免费**：99% 的加锁解锁是 CAS 一条指令的事，只有真冲突时才付系统调用的代价。这个设计 2002 年进内核，是 Linux 高性能同步的基石。

### 6.2 futex 的两个核心操作

`futex(2)` 系统调用的两个最核心操作码：

```c
#include <linux/futex.h>
#include <sys/syscall.h>

/* FUTEX_WAIT: 「如果 *uaddr 仍等于 val，就睡眠等待」
   关键的「比较+睡眠」是原子的——防止「检查后、睡眠前」值被改导致永久睡死 */
syscall(SYS_futex, uaddr, FUTEX_WAIT, val, timeout, NULL, 0);

/* FUTEX_WAKE: 唤醒最多 nr 个等在 uaddr 上的线程 */
syscall(SYS_futex, uaddr, FUTEX_WAKE, nr, NULL, NULL, 0);
```

`FUTEX_WAIT` 的「先比较 `*uaddr == val` 再睡」是原子的，这是关键。否则会有 race：线程检查到锁被占、正准备睡，此时持锁者释放并 WAKE——但 WAKE 时还没人在等待队列上，唤醒落空，然后这个线程才睡下去，**永久睡死（lost wakeup）**。futex 在内核里持队列锁完成「比较+入队睡眠」保证不丢唤醒。

### 6.3 glibc 的 pthread_mutex 怎么基于 futex

简化版 pthread_mutex 加锁逻辑（概念示意，真实实现有更多状态）：

```c
/* 三态：0=空闲, 1=持有无等待者, 2=持有有等待者 */
void lock(int *futex) {
    int c;
    /* 快路径：0→1 */
    if ((c = cmpxchg(futex, 0, 1)) != 0) {
        /* 有竞争 */
        if (c != 2)
            c = xchg(futex, 2);          /* 标记「有等待者」 */
        while (c != 0) {
            futex_wait(futex, 2);        /* 慢路径：睡眠 */
            c = xchg(futex, 2);
        }
    }
}
void unlock(int *futex) {
    if (atomic_fetch_sub(futex, 1) != 1) {   /* 从 1→0 快路径，若不是 1 说明有等待者 */
        *futex = 0;
        futex_wake(futex, 1);            /* 慢路径：唤醒一个 */
    }
}
```

用「0/1/2」三态而不是「0/1」两态，就是为了让**解锁时能在用户态判断「有没有等待者」**——没等待者（state==1）就免掉 `FUTEX_WAKE` 系统调用。这是「无竞争免系统调用」的关键技巧。Go runtime 的 `sync.Mutex`、`runtime.futex`（Linux 上）也是同一套思路。

### 6.4 PI futex：优先级继承，对抗优先级反转

**优先级反转（priority inversion）**：低优先级线程 L 持有锁，高优先级线程 H 等这把锁，此时中优先级线程 M 抢占了 L（M 不需要锁），导致 H 被 M 间接拖住——高优先级反被中优先级阻塞。著名的火星探路者（Mars Pathfinder）故障就是这个。

**PI futex（priority inheritance futex，`FUTEX_LOCK_PI`/`FUTEX_UNLOCK_PI`）**的解法：当 H 等待 L 持有的 PI 锁时，**临时把 L 的优先级提升到 H 的水平**，让 L 尽快跑完临界区释放锁，释放后再恢复 L 原优先级。`pthread_mutex` 设 `PTHREAD_PRIO_INHERIT` 属性即用 PI futex。实时系统（`SCHED_FIFO`/`SCHED_RR`，见 [L03 调度](./L03-精通-CPU-调度-CFS-到-EEVDF.md)）必备。

### 6.5 futex2：演进中的下一代

经典 `futex(2)` 接口有些历史包袱：操作码挤在一个系统调用里、futex word 固定 32 位、对 NUMA 和大规模等待支持有限、不易与现代异步框架结合。**futex2** 是正在演进的一组新系统调用（`futex_waitv` 等已合入较新内核），目标包括：

- **`futex_waitv`**：一次等待**多个 futex**（之前要等多个锁得轮询或变通），这对游戏/图形/Wine（模拟 Windows 的 `WaitForMultipleObjects`）意义重大——Wine/Proton 的性能改善很大程度受益于此；
- 更干净的接口、支持可变大小 futex（8/16/32/64 位）的演进方向；
- 更好地与 io_uring 等异步机制整合的可能。

到 2026 年，`futex_waitv` 已在主流内核可用并被 glibc/Wine 利用；完整的 futex2 接口族仍在逐步铺开。**对应用开发者而言，你几乎永远不会直接调 futex——它是 libc/runtime 的底层**，但理解它能让你看懂锁的性能特征（为什么无竞争锁便宜、为什么高竞争锁会因系统调用而雪崩）。

---

## 第七章 惊群问题全景

**惊群（thundering herd）**：多个执行流在等同一个事件，事件发生时**全部被唤醒**，但只有一个能真正处理，其余白白醒来又睡回去——浪费大量 CPU 和缓存。它在网络、IPC、同步三处都出现，是贯穿本课程的经典问题。

### 7.1 accept 惊群

多个进程 `fork` 后都 `accept` 同一个监听 socket。一个连接到来，**历史上**内核把所有阻塞在 `accept` 的进程都唤醒，结果只有一个 `accept` 成功，其余拿到 `EAGAIN` 又睡回去。

**现状（2026）**：Linux 内核**早已修复了 `accept` 本身的惊群**——内核只唤醒一个等待者。但 `epoll` + `accept` 的组合仍有惊群（见下）。彻底的现代方案是 **`SO_REUSEPORT`**：每个进程各自 `bind` 同一端口，内核为每个 socket 维护独立队列，按连接哈希分发到不同进程——**根本不存在多个进程抢同一个队列**（详见 [L13 Socket 与连接管理](./L13-精通-Socket-与连接管理.md)）。Nginx、各类高性能服务器都用 `SO_REUSEPORT` 做多进程负载均衡。

### 7.2 epoll 惊群

多个进程/线程各自 `epoll_wait` 同一个 fd（或同一个 epoll 实例被 fork 共享）。一个事件到来，多个 `epoll_wait` 被唤醒。解法是 **`EPOLLEXCLUSIVE`** 标志（4.5+）：

```c
struct epoll_event ev = { .events = EPOLLIN | EPOLLEXCLUSIVE };
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
/* 此后该 fd 就绪时，内核只唤醒一个（或少数几个）epoll_wait，而非全部 */
```

`EPOLLEXCLUSIVE` 让内核「只唤醒一个等待者」，大幅缓解多 worker 监听同一 fd 的惊群。详见 [L08 I/O 多路复用](./L08-精通-IO-多路复用.md) 的 epoll 惊群与 `EPOLLEXCLUSIVE` 讨论。

### 7.3 futex 惊群

`FUTEX_WAKE` 唤醒 N 个等待者时，如果唤醒太多（比如条件变量 `pthread_cond_broadcast` 唤醒所有等待者），它们醒来后又去抢同一把互斥锁，**绝大多数抢不到又睡回去**，发生惊群式的「唤醒 → 抢锁失败 → 再睡」。

经典优化是 **`FUTEX_REQUEUE` / `FUTEX_CMP_REQUEUE`**：`pthread_cond_signal`/`broadcast` 唤醒条件变量的等待者时，不是把它们全唤醒去抢 mutex，而是**把它们从「条件变量的 futex 队列」直接「转移（requeue）」到「mutex 的 futex 队列」上**——只唤醒一个，其余原地转挂到 mutex 队列等。这样避免「N 个线程同时醒来抢一把锁」的惊群。glibc 的条件变量实现就用了这个。

### 7.4 惊群缓解手段总览

| 场景 | 惊群表现 | 缓解手段 |
|---|---|---|
| `accept` | 多进程被唤醒抢一个连接 | 内核已修复唤醒一个；多进程负载均衡用 `SO_REUSEPORT` |
| `epoll_wait` | 多 worker 监听同 fd 全醒 | `EPOLLEXCLUSIVE`；或每 worker 独立 epoll + `SO_REUSEPORT` |
| 条件变量 / futex | broadcast 唤醒全部去抢锁 | `FUTEX_REQUEUE`（glibc 已用）；尽量用 signal 而非 broadcast |

**通用原则**：让「等待的多方」尽量不要竞争同一个共享队列（`SO_REUSEPORT` 拆队列），或在唤醒时只唤醒「真正能干活的那一个」（`EPOLLEXCLUSIVE`、`FUTEX_REQUEUE`）。

---

## 生产实践

### 实践一：定位锁争用（off-CPU / futex 分析）

服务吞吐上不去、CPU 没打满，常常是**锁争用**——线程都阻塞在 futex 上睡觉（off-CPU）。定位：

```bash
# 用 bpftrace 统计 futex 系统调用的次数和阻塞——FUTEX_WAIT 多说明竞争激烈
bpftrace -e 'tracepoint:syscalls:sys_enter_futex /args->op & 0x7f == 0/ {
    @wait[comm] = count();         # FUTEX_WAIT(op&0x7f==0) 次数按进程统计
}'

# off-CPU 火焰图：看线程在哪些调用栈上「睡着等锁」（详见 L18）
# offcputime-bpfcc -p <pid> 30
```

`FUTEX_WAIT` 频繁 = 锁竞争激烈 = 该考虑：缩小临界区、分片锁（sharding）、换无锁结构、或用 RCU/读写锁。Go 程序可用 `go tool pprof` 的 mutex/block profile（详见 golang [G13 sync 包](../golang/G13-精通-Go-sync-包.md)）。

### 实践二：消除伪共享

高并发计数器/统计场景，把每个核/每个分片的热点变量按 64 字节缓存行对齐隔离：

```c
struct percpu_counter {
    atomic_long_t count;
    char pad[64 - sizeof(atomic_long_t)];   /* 填充到一条 cache line */
} __attribute__((aligned(64)));
struct percpu_counter counters[NR_CPUS];     /* 每核一个，互不 invalidate */
```

用 `perf c2c`（cache-to-cache）可直接定位伪共享热点缓存行。

### 实践三：进程间共享 mutex（共享内存 + futex）

跨进程互斥（[L14 信号与 IPC](./L14-精通-信号与IPC.md) 的共享内存）：

```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setpshared(&attr, PTHREAD_PROCESS_SHARED);   /* 跨进程 */
pthread_mutexattr_setrobust(&attr, PTHREAD_MUTEX_ROBUST);      /* 健壮锁！ */
pthread_mutex_t *m = shm_ptr;          /* 锁放在共享内存里 */
pthread_mutex_init(m, &attr);
```

**务必设 `PTHREAD_MUTEX_ROBUST`**：否则持锁进程崩溃后，锁永远「被持有」，其它进程死等。robust 锁下，持锁者死亡时其它进程 `lock` 返回 `EOWNERDEAD`，可调 `pthread_mutex_consistent` 恢复。底层正是 futex 的 robust list 机制——内核在线程退出时帮忙清理它持有的 robust futex。

---

## 陷阱清单

**1. 持自旋锁期间睡眠（缺页 / kmalloc(GFP_KERNEL) / 调用阻塞函数）**
- 现象：系统死锁、`BUG: scheduling while atomic`、软死锁告警。
- 原因：自旋锁持有期间禁抢占，睡眠会导致该 CPU 卡死或死锁。
- 修法：自旋锁临界区只做不睡眠的极短操作；需睡眠改用 mutex；分配用 `GFP_ATOMIC`。

**2. 在 x86 上「碰巧能跑」的无锁代码移植到 ARM 就崩**
- 现象：代码在 x86 服务器稳定，上 ARM（Graviton 等）偶发数据错乱。
- 原因：x86 强内存序（TSO）隐藏了缺失的屏障，ARM 弱内存序暴露重排。
- 修法：无锁代码严格用原子操作 + acquire/release 屏障，不依赖架构内存模型。

**3. 缺内存屏障导致「发布的数据读者看到半成品」**
- 现象：读者读到「指针已更新但指向的数据未写完」。
- 原因：`data=...; ptr=new;` 两个写被重排，或读者侧无 acquire。
- 修法：写者 `smp_store_release(&ptr, new)`，读者 `smp_load_acquire(&ptr)`；RCU 用 `rcu_assign_pointer`/`rcu_dereference`。

**4. 伪共享导致多核性能不升反降**
- 现象：增加核数/线程数，吞吐反而下降，`perf c2c` 显示热点缓存行跨核传输。
- 原因：无关的热点变量落在同一缓存行，跨核写互相 invalidate（cache line bouncing）。
- 修法：热点变量 64 字节对齐填充，或 per-CPU 数据结构。

**5. CAS 的 ABA 问题**
- 现象：无锁栈/队列偶发把已释放节点当成有效节点用。
- 原因：值 A→B→A，CAS 误判「没变过」。
- 修法：带版本号的指针（double-width CAS）、hazard pointer，或 RCU 延迟释放。

**6. 进程间共享 mutex 没设 ROBUST，持锁进程崩了全员死锁**
- 现象：一个进程崩溃后，其它访问共享数据的进程全部卡死。
- 原因：普通进程共享 mutex 在持有者死亡后状态永远「锁定」。
- 修法：`PTHREAD_MUTEX_ROBUST` + 处理 `EOWNERDEAD` + `pthread_mutex_consistent`。

**7. 读写锁场景写者饿死**
- 现象：读流量大时，写操作长时间得不到执行。
- 原因：rwlock 读者源源不断，写者拿不到独占。
- 修法：用写者优先的 rwlock 变体；或读多写少改用 RCU；评估 seqlock。

**8. 误把 RCU 当万能锁，在读临界区里睡眠**
- 现象：`rcu_read_lock` 区间内调用可能阻塞的函数，触发告警或数据错乱。
- 原因：经典 RCU 读临界区禁止阻塞（宽限期判定依赖此前提）。
- 修法：读临界区只做不睡眠的快速读取；需睡眠用 SRCU 或换锁。

**9. 条件变量用 broadcast 引发惊群**
- 现象：`pthread_cond_broadcast` 后大量线程被唤醒抢一把锁，CPU 飙高吞吐降。
- 原因：N 个线程同时醒来抢同一 mutex，N-1 个抢不到再睡。
- 修法：能用 `signal` 就别 `broadcast`；依赖 glibc 的 `FUTEX_REQUEUE` 优化；重新设计唤醒粒度。

**10. 多 worker 监听同一 fd 未用 EPOLLEXCLUSIVE，epoll 惊群**
- 现象：多进程 worker，新连接到来时多个被唤醒，只有一个干活，CPU 浪费。
- 原因：共享监听 fd 的 epoll 唤醒所有等待者。
- 修法：加 `EPOLLEXCLUSIVE`；或每 worker 独立 epoll + `SO_REUSEPORT` 拆队列。

---

## 2026 现状

- **RCU 是内核可扩展性的支柱**：从路由表、VFS dcache 到大量子系统，RCU 让多核读访问近线性扩展，6.x 内核里 RCU 的应用面持续扩大，可睡眠的 SRCU 也在更多场景落地。
- **futex2 / `futex_waitv` 落地**：「一次等待多个 futex」已在主流内核可用，glibc 与 Wine/Proton 受益显著；完整 futex2 接口族（可变大小 futex 等）仍在演进。应用层依然几乎不直接碰 futex——它是 libc/runtime 的底层。
- **qspinlock 是内核自旋锁默认实现**：队列自旋锁解决了 ticket spinlock 在高竞争下的缓存行乒乓，公平且可扩展。
- **弱内存序架构成为一等公民**：ARM64 服务器（Graviton、Ampere）大规模生产，弱内存模型不再是「边缘情况」——内存屏障的正确性从「学术细节」变成「线上事故来源」。跨架构正确性是 2026 年并发代码的硬要求。
- **语言级内存模型成熟**：Go、Rust、C/C++ 的内存模型都已稳定，`sync/atomic`、`std::sync::atomic`、C11 atomics 是日常工具。Go 1.19+ 的类型化原子（`atomic.Int64` 等）降低了误用。
- **eBPF 观测同步**：用 bpftrace 追踪 futex、用 `offcputime` 做 off-CPU 火焰图定位锁争用，已是 SRE 标准手段（见 [L18](./L18-精通-性能诊断方法论与工具.md)、[L19](./L19-精通-eBPF.md)）。

---

## 练习题

1. **无竞争锁的开销**：写一个程序，单线程对一把 `pthread_mutex` 做一亿次 lock/unlock，用 `strace -c` 统计期间 `futex` 系统调用的次数。解释为什么次数远小于一亿（接近 0）。

2. **CAS 自增**：用 C11 `atomic_compare_exchange` 写一个无锁自增循环，对比 `pthread_mutex` 保护的自增，在多线程下测吞吐与正确性。再故意用 `memory_order_relaxed` 做一个「只计数不发布数据」的场景，说明此时为何 relaxed 足够。

3. **可见性 bug 复现**：写第一章 §1.2 的 `data/ready` 例子，消费者用普通 `while(ready==0)` 自旋。在 ARM 机器（或开足够激进优化）上尝试复现「读到 data 不是 42」。再用 acquire/release 修复。

4. **锁选型**：分别给出最适合用 spinlock、mutex、rwlock、seqlock、RCU 的一个真实内核/应用场景，并说明为什么不用其它锁。

5. **RCU 流程**：画时序图说明 `synchronize_rcu()` 如何保证「所有持有旧指针的读者退出后才释放旧数据」。解释经典 RCU 为什么要求读临界区不能睡眠，以及这个前提如何用于判定宽限期结束。

6. **futex 三态**：解释 pthread_mutex 用「0/1/2」三态而非「0/1」的原因，重点说明它如何让「无等待者时解锁免去 `FUTEX_WAKE` 系统调用」。

7. **排障题（吞吐上不去 CPU 没满）**：一个 Go 服务 QPS 卡在某个值上不去，但 CPU 利用率只有 40%，看起来「有劲使不出」。用 off-CPU 火焰图 / `go tool pprof` 的 block/mutex profile 怀疑锁争用。给出完整定位流程，并列出至少三种缓解锁争用的改造方向（缩小临界区、分片锁、无锁结构、RCU/读写锁、改用 channel 等）。

8. **排障题（伪共享）**：一个并发计数模块，把线程数从 4 加到 16，吞吐不升反降。`perf c2c` 显示某个结构体所在缓存行有大量跨核传输（HITM）。诊断为伪共享，给出用缓存行对齐填充修复的代码，并解释为什么填充后吞吐随核数恢复线性增长。

---

## 参考答案

1. **无竞争锁的开销**：单线程无竞争时，`lock` 用一条 CAS 把 futex word 从 0 改成 1 即成功，`unlock` 用一条原子操作改回 0，全程纯用户态，根本不触发 `futex` 系统调用。`futex(FUTEX_WAIT/FUTEX_WAKE)` 只在 CAS 失败（锁被占、需睡眠）或解锁时发现「有等待者」标记才调用。单线程没有竞争，所以 `strace -c` 统计到的 `futex` 次数接近 0，远小于一亿。这正是「无竞争免系统调用」的快路径设计。

2. **CAS 自增**：无锁自增是「读当前值 → 算 old+1 → `atomic_compare_exchange` 写回，失败重试」的循环（正文 §2.2）。多线程下两种方式结果都应等于 N×次数（正确），但无竞争/低竞争时 CAS 通常吞吐更高（无系统调用、无睡眠唤醒），高竞争时 CAS 因重试增多也会退化。第二个场景里「只计数、不依赖该计数去发布/读取其它数据」时，各次自增之间没有需要约束顺序的其它访存，只需保证「这一个自增原子且不丢」，因此 `memory_order_relaxed` 足够——relaxed 仍保证操作本身原子，只是不提供 acquire/release 那样的跨变量顺序与可见性约束，而这里不需要。

3. **可见性 bug 复现**：消费者 `while(ready==0);` 后 `use(data)`，生产者 `data=42; ready=1;`。在弱内存序架构（ARM）上，生产者两个写可能被重排、或消费者侧读被重排，导致看到 `ready==1` 时 `data` 还不是 42（正文 §1.2 指出编译器重排 + CPU/缓存重排两个原因）。修复用 acquire/release 配对：生产者 `data=42; smp_store_release(&ready,1);`，消费者 `if (smp_load_acquire(&ready)) use(data);`（用户态对应 C11 `memory_order_release`/`acquire`）。release 保证 `data=42` 不重排到 `ready=1` 之后，acquire 保证看到 `ready==1` 后对 `data` 的读不前移，建立 happens-before。

4. **锁选型**（各举一例，理由对照正文 §3.5）：
   - spinlock：中断处理里更新一个计数/摘链表指针。临界区极短（ns 级），且中断上下文不能睡眠，只能用不睡眠的锁。
   - mutex：进程上下文里较长、可能睡眠的互斥（如改一个需要分配内存的数据结构）。临界区长，自旋会浪费 CPU，且允许睡眠。
   - rwlock：读多写少、读临界区较短的共享数据。允许多读者并发，又比 RCU 简单。
   - seqlock：`jiffies`/时间戳这类「写极少、写者不能被读者挡住、读者可重试」的数据。读者不阻塞写者。
   - RCU：路由表/FIB、dcache 这类读极多写极少的结构，要求读端几乎零开销、多核近线性扩展，rwlock 的读端原子计数都嫌贵。

5. **RCU 流程**：时序见正文 §4.3 图——`rcu_assign_pointer` 切指针后，可能仍有「切换前就 `rcu_dereference` 拿到旧指针」的读者在用旧数据；`synchronize_rcu()` 阻塞写者直到宽限期结束（所有这些老读者退出），之后 `kfree(old)` 才安全。经典 RCU 要求读临界区不能睡眠，是因为宽限期判定依赖这一前提：读临界区内不睡眠，那么「某 CPU 经历一次上下文切换/静止态（quiescent state）」就证明该 CPU 上切换前的所有老读临界区都已结束；当每个 CPU 都报告过一次静止态，宽限期即结束。若读临界区可睡眠，这个推断不再成立。

6. **futex 三态**：三态是 0=空闲、1=持有无等待者、2=持有有等待者（正文 §6.3）。两态（0/1）的话，解锁者无法在用户态判断是否有线程在内核等待队列上，只能每次解锁都保守地调 `FUTEX_WAKE`，即使没人等也付出一次系统调用。引入「2=有等待者」后，加锁慢路径会把状态置为 2，解锁时若发现状态是 1（仅自己持有、无等待者）就直接改回 0 并跳过 `FUTEX_WAKE`，纯用户态完成；只有状态为 2 时才需要 `FUTEX_WAKE` 唤醒。这正是「无等待者时解锁免系统调用」的关键。

7. **排障题（吞吐上不去 CPU 没满）**：定位流程——(1) 现象是 CPU 不满但吞吐封顶，典型 off-CPU 等待（线程睡在 futex 上）；(2) `go tool pprof` 抓 block profile（`runtime.SetBlockProfileRate`）和 mutex profile（`runtime.SetMutexProfileFraction`），看哪段调用栈阻塞/争用时间最长；(3) 系统层用 `offcputime`（eBPF）做 off-CPU 火焰图，或 `bpftrace` 统计 `FUTEX_WAIT` 次数按 goroutine/函数定位热点锁（正文「生产实践一」）；(4) 确认是某把 mutex 的争用。缓解方向（至少三种，对照正文）：缩小临界区（只把真正需要互斥的部分放锁内）、分片锁/sharding（按 key 哈希拆成多把锁降低争用）、改无锁结构或原子操作、读多写少改 rwlock 或 RCU、用 channel 把共享状态改为消息传递归一到单 goroutine。

8. **排障题（伪共享）**：诊断——`perf c2c` 的 HITM（命中其它核 Modified 缓存行）说明多个核反复抢同一条缓存行的所有权，即两个本应无关的热点变量落在同一条 64 字节缓存行上发生伪共享（正文 §1.3、生产实践二）。修复用缓存行对齐填充，让每个核/分片的计数器独占一条缓存行：

   ```c
   struct percpu_counter {
       atomic_long_t count;
       char pad[64 - sizeof(atomic_long_t)];   /* 填充到一整条 cache line */
   } __attribute__((aligned(64)));
   struct percpu_counter counters[NR_CPUS];     /* 每核一个，互不 invalidate */
   ```

   填充后每个核写自己的缓存行，不再触发对其它核缓存行的 invalidate（不再 cache line bouncing），写操作不需要协议往返抢独占权，因此各核的更新彼此独立、互不拖累，吞吐才能随核数恢复近线性增长。
