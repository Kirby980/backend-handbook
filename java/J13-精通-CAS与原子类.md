# 精通 CAS 与原子类

> CAS（Compare-And-Swap）是 Java 无锁并发的基石——`AtomicInteger`、并发容器、[AQS](./J09-精通-AQS原理.md) 全靠它。它用一条 CPU 原子指令实现"乐观锁"，避免了加锁的阻塞开销。但它也有 ABA、自旋、单变量三大问题。本篇讲透 CAS 原理与原子类。
>
> **📅 基准：Java 17/21。**

---

## 一、CAS 是什么

**CAS（Compare-And-Swap，比较并交换）** 是一种**无锁（lock-free）**的原子操作：在更新一个变量时，只有当它的当前值等于预期值时，才把它改成新值，否则不改。整个"比较 + 交换"是一条 CPU 原子指令（x86 的 `cmpxchg`），不可被打断。

它是**乐观锁**思想：不像 synchronized 那样先加锁（悲观地假设会冲突），而是乐观地直接尝试修改，失败了再重试——没有线程阻塞和上下文切换，高并发下性能好。

---

## 二、CAS 的三个操作数

CAS(V, A, B)：
- **V**：要修改的内存变量。
- **A**：预期的旧值（expected）。
- **B**：要写入的新值（new）。

逻辑：`if (V == A) { V = B; return true; } else { return false; }`，且这一切**原子执行**。

```java
// AtomicInteger.incrementAndGet 的本质（自旋 CAS）
public final int incrementAndGet() {
    int oldValue, newValue;
    do {
        oldValue = get();          // 读当前值
        newValue = oldValue + 1;   // 算新值
    } while (!compareAndSet(oldValue, newValue)); // CAS：失败就重试
    return newValue;
}
```

如果 CAS 失败（说明期间被别的线程改了），就**自旋重试**——重新读、重新算、再 CAS，直到成功。这就是无锁的"乐观重试"。

---

## 三、原子类

`java.util.concurrent.atomic` 包提供基于 CAS 的原子类，无锁实现线程安全的自增/更新：

| 类别 | 类 |
|---|---|
| 基本类型 | AtomicInteger、AtomicLong、AtomicBoolean |
| 引用类型 | AtomicReference、AtomicStampedReference、AtomicMarkableReference |
| 数组 | AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray |
| 字段更新器 | AtomicIntegerFieldUpdater 等 |
| 高性能累加器 | LongAdder、LongAccumulator、DoubleAdder |

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();        // 原子 +1（CAS 自旋）
counter.compareAndSet(1, 5);      // CAS：当前是 1 才改成 5
counter.getAndAdd(10);            // 原子加 10，返回旧值
```

用 `AtomicInteger` 替代 `synchronized int i++`：无锁、无阻塞，并发自增性能更好（中低竞争下）。

---

## 四、CAS 的三大问题

| 问题 | 说明 |
|---|---|
| **ABA 问题** | 值从 A→B→A，CAS 以为没变（实际变过两次），见第五节 |
| **自旋开销** | 高竞争下 CAS 反复失败、长时间自旋空耗 CPU |
| **只能保证一个变量原子** | CAS 只对单个变量原子，多个变量的复合操作保证不了 |

应对：
- ABA → 加版本号（AtomicStampedReference）。
- 自旋开销 → 竞争极高时不如用锁，或用 LongAdder 分散竞争。
- 多变量 → 把多个变量封装成一个对象用 AtomicReference，或用锁。

---

## 五、ABA 问题与解决

**ABA**：线程1 读到值 A，准备 CAS 改成 C；期间线程2 把 A→B→A 又改回来了。线程1 的 CAS 看到值还是 A，认为"没变过"、成功执行——但其实中间已经变化过，可能破坏了某些不变量。

经典场景：无锁栈/链表的指针，节点被弹出又压回，CAS 误判导致结构错乱。

**解决：加版本号/戳**。用 `AtomicStampedReference`，每次修改不仅改值还改一个递增的 stamp（版本号），CAS 时**同时比较值和版本号**：

```java
AtomicStampedReference<Integer> ref = new AtomicStampedReference<>(100, 0);
int[] stampHolder = new int[1];
int val = ref.get(stampHolder);    // 取值和当前版本号
int stamp = stampHolder[0];
// CAS：值和版本号都匹配才成功
ref.compareAndSet(val, val + 1, stamp, stamp + 1);
```

即使值 A→B→A，版本号也从 0→1→2 变了，CAS 因版本号不匹配而失败，避免 ABA。`AtomicMarkableReference` 是简化版（只用一个 boolean 标记是否被改过）。

---

## 六、LongAdder vs AtomicLong

高并发计数时，`AtomicLong` 的所有线程都 CAS 同一个变量，**竞争激烈、大量自旋失败**，性能下降。

**LongAdder（Java 8）** 用**分段累加（cell 分散）**优化：内部维护一个 `base` 和一个 `Cell[]` 数组，不同线程 CAS 到不同的 Cell（分散热点），求和时把 base + 所有 Cell 加起来。

| 维度 | AtomicLong | LongAdder |
|---|---|---|
| 结构 | 单个变量 | base + Cell[] 分段 |
| 高并发写 | 竞争激烈、自旋多 | 分散竞争，吞吐高 |
| 读 sum | 实时精确 | 求和时弱一致（可能不含正在写的） |
| 内存 | 小 | 大（Cell 数组） |
| 适用 | 低并发、需精确读 | **高并发计数**（统计、监控） |

**高并发只增计数**（如 QPS 统计、监控指标）优先 `LongAdder`；需要精确实时值或低并发用 `AtomicLong`。ConcurrentHashMap 的 size 也用了类似分段累加思想（见 [J02](./J02-精通-HashMap与ConcurrentHashMap.md)、[J12](./J12-精通-并发容器.md)）。

---

## 七、Unsafe 与 VarHandle

- **Unsafe**：CAS 的底层靠 `sun.misc.Unsafe` 类（如 `compareAndSwapInt`），它直接操作内存、绕过 JVM 安全检查，是原子类和 AQS 的底层。但 Unsafe 是内部 API、危险，不应直接用。
- **VarHandle（Java 9+）**：官方提供的替代 Unsafe 的标准 API，提供细粒度的内存访问语义（plain/opaque/acquire-release/volatile）和 CAS 操作。新版本的原子类、AQS 内部逐步改用 VarHandle，是现代 Java 做底层并发的推荐方式。

---

## 八、CAS 与锁的对比

| 维度 | CAS（乐观锁） | synchronized/Lock（悲观锁） |
|---|---|---|
| 思路 | 先改，冲突了重试 | 先加锁，独占后改 |
| 阻塞 | 不阻塞（自旋） | 阻塞 |
| 上下文切换 | 无 | 有（重量级锁） |
| 适用 | 竞争不激烈、操作简单 | 竞争激烈、临界区复杂 |
| 风险 | ABA、自旋空耗 | 死锁、阻塞开销 |

**经验**：竞争不激烈 + 操作简单（计数、标志、单变量更新）→ CAS/原子类；竞争激烈 + 临界区复杂（多个操作要一起原子）→ 用锁。两者常配合（如 AQS 用 CAS 改 state、失败才入队阻塞）。

---

## 陷阱清单

- **以为 CAS 没有任何问题**：有 ABA、自旋开销、单变量限制三大问题。
- **忽视 ABA**：无锁栈/链表等结构对 ABA 敏感。用 AtomicStampedReference 加版本号。
- **高竞争下死用 AtomicLong**：大量自旋失败拖性能。高并发计数用 LongAdder。
- **用 CAS 保证多变量原子**：CAS 只对单变量原子，多变量封装成对象用 AtomicReference 或用锁。
- **自旋无上限**：极高竞争下 CAS 一直失败空耗 CPU，此时锁可能更优。
- **LongAdder 当精确实时读用**：sum 是弱一致的，统计累加可以、需精确瞬时值用 AtomicLong。
- **直接用 sun.misc.Unsafe**：内部危险 API，用 VarHandle 或现成原子类。

---

## 2026 现状

- **VarHandle 全面替代 Unsafe**：现代并发底层用 VarHandle（Java 9+）做 CAS 和细粒度内存访问，Unsafe 逐步被限制/封装。
- **LongAdder 是高并发计数标准**：监控、限流计数器普遍用 LongAdder（或其思想），避免单点 CAS 热点。
- **原子类仍是无锁编程基石**：在虚拟线程（[J29](./J29-精通-Java版本特性演进.md)）时代，无锁原子操作依然高效适用。
- **CAS 是底层共识**：从 AQS、并发容器到限流、计数，CAS 是 Java 并发"无锁"能力的根基，面试理解它能串起一大片知识。

---

## 练习题

1. 什么是 CAS？它的三个操作数是什么？为什么说它是"乐观锁"？

<details><summary>参考答案</summary>

CAS（Compare-And-Swap，比较并交换）是一种无锁的原子操作：更新一个变量时，仅当该变量的当前值等于预期的旧值时，才将其更新为新值，否则不更新；整个"比较 + 交换"由一条 CPU 原子指令（如 x86 的 cmpxchg）完成，不可被打断。三个操作数：①V——要操作的内存变量（当前值）；②A——预期的旧值（expected）；③B——要写入的新值（new）。逻辑是"若 V==A 则把 V 改为 B 并返回成功，否则返回失败"，且原子执行。说它是乐观锁，是因为它的思路与悲观锁（synchronized 先加锁再操作，假设一定会冲突）相反：CAS **乐观地假设不会有冲突，直接尝试修改**，只有当发现值被别人改过（CAS 失败）时才重试（自旋）；整个过程不加锁、不阻塞线程、无上下文切换。这种"先做、冲突再重试"正是乐观并发控制的体现，在竞争不激烈时性能优于加锁。

</details>

2. AtomicInteger 的 incrementAndGet 是如何用 CAS 实现线程安全自增的？

<details><summary>参考答案</summary>

它用"读取当前值 + 计算新值 + CAS 写回 + 失败自旋重试"的循环实现。伪逻辑：在一个 do-while 循环里，先 `get()` 读取当前值 oldValue，计算 newValue = oldValue + 1，然后调用 `compareAndSet(oldValue, newValue)` 尝试用 CAS 把变量从 oldValue 原子地改成 newValue；如果 CAS 成功（说明从读取到写回期间没有其他线程修改过该值），就返回 newValue；如果 CAS 失败（说明期间有别的线程改了值，oldValue 已过期），就**重新读取最新值、重新计算、再次 CAS**，循环直到成功。这样每次成功的自增都是基于最新值且原子完成的，多线程并发自增不会丢失更新。整个过程无锁、不阻塞，只在冲突时自旋重试，因此在中低竞争下比 synchronized 的 i++ 性能更好（无锁竞争和上下文切换）。底层 compareAndSet 通过 Unsafe/VarHandle 调用 CPU 的原子比较交换指令实现。

</details>

3. CAS 有哪三大问题？分别如何应对？

<details><summary>参考答案</summary>

三大问题：①**ABA 问题**——一个值从 A 被改成 B 又改回 A，CAS 比较时看到值还是 A 会误以为"没变过"而成功，但实际中间已发生过变化，可能破坏不变量（对无锁栈/链表等指针结构尤其危险）。应对：使用带版本号的 AtomicStampedReference，每次修改同时递增版本戳，CAS 时同时比较值和版本号，即使值变回 A 版本号也变了，从而识别出变化。②**自旋开销**——在高竞争下 CAS 反复失败、线程长时间自旋空转，浪费 CPU。应对：竞争极高时改用锁（阻塞而非空转），或用 LongAdder 等分散竞争的结构降低单点冲突，或限制自旋次数后转阻塞。③**只能保证单个变量的原子性**——CAS 一次只能原子地更新一个变量，无法保证多个变量的复合操作原子。应对：把需要一起更新的多个变量封装成一个对象，用 AtomicReference 对这个对象引用做 CAS；或干脆用锁来保护这段复合操作。理解这三个问题能避免无锁编程中的隐蔽 bug。

</details>

4. 什么是 ABA 问题？AtomicStampedReference 如何解决它？

<details><summary>参考答案</summary>

ABA 问题：线程 T1 读取共享变量的值为 A，准备用 CAS 把它改成 C；在 T1 执行 CAS 之前，线程 T2 先把值从 A 改成 B、又改回 A。此时 T1 执行 CAS 比较，发现值仍然是 A（等于预期），于是 CAS 成功——但实际上这个值在中途已经被修改过两次，只是又"恰好"变回了 A。在单纯的数值场景这可能无害，但在无锁数据结构（如无锁栈、链表）中很危险：例如栈顶节点 A 被弹出（变 B）、又有个新节点复用了 A 的地址被压回（变 A），T1 的 CAS 误认为栈顶没变而成功，可能导致已释放节点被错误链接、结构损坏。解决：用 **AtomicStampedReference**，它把"引用值 + 整型版本戳（stamp）"绑定，每次修改在改值的同时递增 stamp。CAS 时**同时比较值和版本戳**，只有两者都与预期一致才成功。这样即使值经历 A→B→A，版本戳已从 0→1→2 改变，T1 的 CAS 会因版本戳不匹配而失败，从而识别出"中间发生过变化"，避免 ABA。AtomicMarkableReference 是其简化版（用一个布尔标记代替版本号，只关心"是否被改过"）。

</details>

5. LongAdder 相比 AtomicLong 在高并发计数时有什么优势？代价是什么？

<details><summary>参考答案</summary>

优势：在高并发计数场景，AtomicLong 的所有线程都对**同一个**变量做 CAS，竞争激烈时大量 CAS 失败、反复自旋，成为热点瓶颈、吞吐下降。LongAdder 采用**分段累加（热点分散）**：内部维护一个 base 值和一个 Cell[] 数组，不同线程会被散列到不同的 Cell 上各自 CAS 累加（竞争分散到多个 Cell，单个 Cell 的冲突大幅减少），需要总值时再把 base 和所有 Cell 求和。这样把"对一个变量的争抢"分散成"对多个 Cell 的并行更新"，高并发下吞吐远高于 AtomicLong（竞争越激烈优势越明显）。代价：①**读取（sum）是弱一致的**——求和时可能没把正在并发写入的 Cell 算进去，得到的是近似值，不保证某一瞬间的精确实时值；②**内存占用更大**（要维护 Cell 数组）；③只适合累加统计类操作，不适合需要 CAS 返回旧值做逻辑判断的场景。因此：高并发的只增计数（如 QPS 统计、监控指标）优先用 LongAdder；需要精确实时读或低并发、或要基于 compareAndSet 做控制的场景用 AtomicLong。

</details>

6. CAS（乐观锁）和 synchronized（悲观锁）应该如何选择？

<details><summary>参考答案</summary>

按竞争程度和操作复杂度选择。**CAS/原子类（乐观锁）** 适合：竞争不激烈、临界操作简单（针对单个变量的计数、标志位、引用更新等）。它不加锁、不阻塞、无上下文切换，冲突时只是自旋重试，在中低竞争下性能优于加锁。但要注意它的 ABA、自旋空耗、只能保证单变量原子等问题。**synchronized/Lock（悲观锁）** 适合：竞争激烈、或临界区包含多个操作需要一起原子（复合操作）、逻辑复杂。加锁能保证整段临界区互斥，逻辑清晰；在高竞争下，阻塞等待比让大量线程空转自旋（CAS 反复失败）更省 CPU。选择经验：①只是计数/单变量更新、竞争不高 → 用原子类（CAS）；②要保护一段复合逻辑、或竞争很激烈、临界区较长 → 用锁。实际上两者常结合：例如 AQS 先用 CAS 快速尝试改 state（乐观、快路径），失败了才入队阻塞（悲观、慢路径），兼顾了低竞争时的高性能和高竞争时的有序阻塞。所以不是二选一对立，而是按场景搭配。

</details>
