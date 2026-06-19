# 精通 Python 多线程 threading

> 多线程在 Python 里"名声不好"——因为 GIL,它对 CPU 密集无能为力。但对 **IO 密集**它仍然有效且简单。面试常问:"GIL 下多线程还有用吗?""线程安全靠 GIL 够吗?"本篇讲清 threading 的正确用法与同步,紧承 [P17 GIL](./P17-精通-Python-GIL原理.md),对比 [P21 进程](./P21-精通-Python-多进程multiprocessing.md)、[P22 asyncio](./P22-精通-Python-asyncio与协程.md)。
>
> **📅 基准:2026 年 6 月,CPython 3.13 / 3.14。**

---

## 一、Thread 基础

```python
import threading
def worker(n):
    print(f"working {n}")

t = threading.Thread(target=worker, args=(1,))
t.start()                      # 启动
t.join()                       # 等待结束
```

线程共享进程的内存空间(全局变量、堆对象),通信方便,但也因此需要同步保护共享状态。

---

## 二、GIL 下线程的定位

回顾 [P17](./P17-精通-Python-GIL原理.md):GIL 使**同一时刻只有一个线程执行字节码**。所以:

- **IO 密集 ✅**:线程阻塞在网络/磁盘/DB IO 时**释放 GIL**,其他线程趁机运行——多线程能有效提升 IO 并发吞吐(如同时发多个 HTTP 请求、并发读多个文件)。
- **CPU 密集 ❌**:线程被 GIL 串行化,无法并行计算,多线程无加速甚至更慢——这类任务用多进程([P21](./P21-精通-Python-多进程multiprocessing.md))或释放 GIL 的库。

```python
# 适合多线程:并发 IO
threads = [threading.Thread(target=download, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
```

---

## 三、线程安全:GIL 不等于不用锁

**最大误区**:"有 GIL 所以多线程不用加锁"——**错**(见 [P17](./P17-精通-Python-GIL原理.md))。GIL 只保证**单条字节码**原子,**复合操作不原子**:

```python
counter = 0
def inc():
    global counter
    for _ in range(100000):
        counter += 1            # 非原子!读→加→写三步,可能被打断

threads = [threading.Thread(target=inc) for _ in range(2)]
# 结果通常 < 200000 —— 丢失更新(竞态)
```

`counter += 1` 是"读值→加 1→写回"多步,两线程交错执行会丢更新。`if k not in d: d[k]=v`(check-then-act)、列表的复合修改等同理。**只要并发读写共享可变状态,就必须加锁**。

---

## 四、同步原语

`threading` 提供多种同步工具:

```python
lock = threading.Lock()              # 互斥锁(最常用)
with lock:                           # 临界区:同一时刻只有一个线程进入
    counter += 1

rlock = threading.RLock()            # 可重入锁(同一线程可多次获取)
sem = threading.Semaphore(5)         # 信号量:限制并发数(如最多 5 个并发)
event = threading.Event()            # 事件:线程间信号(set/wait)
cond = threading.Condition()         # 条件变量:等待某条件成立(生产消费)
barrier = threading.Barrier(3)       # 屏障:等齐 N 个线程再一起放行
```

- **`Lock` + `with`**:保护临界区的标准做法。
- **`Semaphore`**:限制对资源的并发数(如限制并发连接)。
- **死锁**:多锁顺序不一致会死锁;固定加锁顺序、用超时、尽量减少锁范围(见 [J10](../java/INDEX.md) 死锁四条件)。

---

## 五、queue.Queue:线程安全的生产消费

`queue.Queue` 是**线程安全**的队列,生产者-消费者模式的首选(内部已处理好锁,无需自己加锁):

```python
import queue, threading
q = queue.Queue(maxsize=100)         # 有界队列,满了阻塞(天然反压)

def producer():
    for item in source: q.put(item)
def consumer():
    while True:
        item = q.get()               # 空了阻塞等待
        process(item)
        q.task_done()
```

用 `Queue` 在线程间传递任务/数据,比共享变量 + 锁更安全清晰。`Queue`/`LifoQueue`/`PriorityQueue` 各适用不同场景。

---

## 六、线程池与优雅管理

直接用 `concurrent.futures.ThreadPoolExecutor` 比手动管理线程更方便(见 [P24](./P24-精通-Python-并发选型与free-threading.md)):

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
with ThreadPoolExecutor(max_workers=10) as ex:
    futures = [ex.submit(fetch, url) for url in urls]
    for fut in as_completed(futures):
        result = fut.result()        # 异常会在 result() 时抛出
# with 退出时自动等待所有任务完成
```

- **daemon 线程**:`t.daemon = True` 的线程在主线程退出时被强制结束(适合后台任务);非 daemon 线程会阻止进程退出。
- **异常**:线程里的异常不会传到主线程;用 Executor 的 `future.result()` 获取异常,或在线程内捕获记录。

---

## 陷阱清单

- **"GIL 所以不用锁"**:复合操作(`+=`、check-then-act)非原子,并发写共享状态必须加锁。
- **CPU 密集用多线程**:GIL 下无加速;用多进程/向量化(见 [P21](./P21-精通-Python-多进程multiprocessing.md))。
- **死锁**:多锁加锁顺序不一致、忘记释放;用 `with lock` 自动释放、固定锁顺序、加超时。
- **共享可变状态满天飞**:难维护易竞态;优先用 `queue.Queue` 传递数据,减少共享。
- **线程异常被吞**:线程内异常不传播到主线程;用 Executor + `result()` 或线程内捕获。
- **手动 new 大量线程**:开销与上限失控;用 `ThreadPoolExecutor` 限制并发。
- **daemon 线程做关键工作**:主线程退出时被强杀,可能丢数据/不清理;关键任务别设 daemon。
- **忘记 join / 不优雅停止**:任务没做完进程就退出。

---

## 2026 现状

- **多线程仍是 IO 密集并发的简单选择**(尤其遗留同步代码),`ThreadPoolExecutor` 用得最多;高并发新代码越来越多转向 **asyncio**(见 [P22](./P22-精通-Python-asyncio与协程.md))。
- **free-threading(去 GIL,见 [P17](./P17-精通-Python-GIL原理.md))** 一旦普及,多线程将能做 CPU 并行——但**更需要正确加锁**(真并行下竞态更危险),线程安全知识反而更重要。
- **`queue.Queue` / `concurrent.futures`** 是线程编程的标准基础设施。
- 与 Java/Go 对照:Java 线程([J11](../java/INDEX.md))、Go goroutine([G11](../golang/INDEX.md))都能真并行;Python 线程因 GIL 只擅长 IO 并发——这是选型时的根本差异(见 [P24](./P24-精通-Python-并发选型与free-threading.md))。

---

## 练习题

1. 在 GIL 存在的情况下,多线程对哪类任务有用、哪类没用?为什么?

<details><summary>参考答案</summary>

对 **IO 密集型任务有用,对 CPU 密集型任务基本没用**。原因在于 GIL 保证同一时刻只有一个线程执行 Python 字节码(见 P17)。**CPU 密集型**(大量纯计算、几乎不阻塞):多个线程会被 GIL 串行化,只能轮流执行字节码,**无法在多核上并行计算**,因此多线程不仅得不到加速,还要承担线程切换和争抢 GIL 的额外开销,常常比单线程还慢——这类任务应改用多进程(每进程独立 GIL,真并行,见 P21)、释放 GIL 的库(NumPy 等)、或 free-threading 构建。**IO 密集型**(大量等待网络/磁盘/数据库等):多线程**有效**,因为**线程在执行阻塞式 IO 操作时会主动释放 GIL**,让其他线程趁这段等待时间运行。这样多个线程可以"你等你的 IO、我跑我的代码",并发地推进多个 IO 操作(如同时发起多个 HTTP 请求、并发读多个文件),显著提升吞吐——虽然任一时刻仍只有一个线程在执行 Python 代码,但 IO 等待并不占用 GIL,所以并发是真实有效的。简言之:**GIL 阻止的是"并行执行 Python 计算",而 IO 等待期间 GIL 被释放,所以 IO 并发不受影响**。这决定了 Python 并发选型:IO 密集用多线程或 asyncio,CPU 密集用多进程/向量化(见 P24)。

</details>

2. "有 GIL 就不用加锁"对吗?举例说明。

<details><summary>参考答案</summary>

**不对**,这是危险的误解(见 P17)。GIL 只保证**单条字节码指令**级别的原子性(同一时刻只有一个线程在执行字节码),但它**不保证由多条字节码组成的复合操作是原子的**——在这些指令之间,GIL 可能因时间片到期或 IO 而被切换给另一个线程,从而产生竞态。**经典例子:`counter += 1`**。这一行看似简单,实际编译成多条字节码:"**读取** counter 当前值 → **加 1** → **写回** counter"。当两个线程并发执行 `counter += 1` 时,可能出现:线程 A 读到 counter=0,还没写回,GIL 切到线程 B,B 也读到 counter=0、加 1 写回成 1,然后切回 A,A 用它之前读到的 0 加 1 也写回 1——本应是 2,结果只有 1,**丢失了一次更新**。所以两个线程各自 `counter += 1` 十万次,最终结果通常**小于** 20 万。其他例子:`if key not in d: d[key] = compute()`(check-then-act,检查和赋值之间可能被别的线程插入)、`lst.append(x); total += len(lst)` 这种多步组合、对共享对象的"读-改-写"等,都不是原子的。**正确做法**:只要存在多个线程**并发读写同一个可变共享状态**,就必须用同步手段保护——用 `threading.Lock`(配合 `with lock:` 保护临界区)、或使用本身线程安全的结构(`queue.Queue`)、或把共享状态改成不共享(每线程局部 + 最后汇总)。补充:某些由 C 实现且"一条字节码完成"的操作(如 `list.append` 单次调用)碰巧是原子的,但**不要依赖这种"碰巧"**(不同操作、不同版本未必成立);而且在 **free-threading(去 GIL)** 下线程真并行,这类竞态会更频繁,**更**需要加锁。

</details>

3. threading 提供了哪些同步原语?Lock 和 RLock、Semaphore 有什么区别?

<details><summary>参考答案</summary>

`threading` 模块提供的主要同步原语:①**`Lock`(互斥锁)**——最基本,任意时刻只能被一个线程持有,用于保护临界区(共享资源的读写),通常配合 `with lock:` 使用(自动获取和释放)。一个已被持有的 Lock 再次 acquire 会阻塞(即使是同一个线程,所以同一线程重复获取会**自死锁**)。②**`RLock`(可重入锁/递归锁)**——与 Lock 类似,但**允许同一个线程多次获取**(内部记录持有线程和获取次数,要释放相同次数才真正解锁)。适用于"一个加了锁的方法内部又调用了另一个也要加同一把锁的方法"的递归/嵌套场景——用普通 Lock 会自死锁,用 RLock 则不会。③**`Semaphore`(信号量)**——维护一个计数器,acquire 减一、release 加一,计数为 0 时 acquire 阻塞;它允许**最多 N 个线程同时**进入(而非互斥的 1 个),用于**限制对某资源的并发数**(如限制最多 5 个线程同时访问数据库/发请求)。`BoundedSemaphore` 还会防止 release 超过初始值。④**`Event`(事件)**——一个线程间的布尔信号:`set()` 置位、`clear()` 复位、`wait()` 阻塞直到被 set,用于"一个线程通知其他线程某事件发生了"(如启动信号、停止信号)。⑤**`Condition`(条件变量)**——更复杂的等待/通知机制,线程可以 `wait()` 等待某条件成立、其他线程改变状态后 `notify()`/`notify_all()` 唤醒,典型用于生产者-消费者(等待"队列非空/非满")。⑥**`Barrier`(屏障)**——让固定数量的线程互相等待,都到达后再一起继续,用于阶段同步。**核心区别**:**Lock vs RLock**——Lock 不可重入(同线程重复获取会死锁),RLock 可被同一线程重入;**Lock/RLock vs Semaphore**——前者是互斥(最多 1 个),Semaphore 允许最多 N 个并发。实践中最常用 Lock(保护临界区)和 Semaphore(限并发);需要嵌套加锁时用 RLock;生产消费优先直接用线程安全的 `queue.Queue`(它内部已封装好锁和条件变量)。

</details>

4. 为什么推荐用 `queue.Queue` 做线程间的生产者-消费者通信?

<details><summary>参考答案</summary>

因为 **`queue.Queue` 是线程安全的**,它在内部已经用锁和条件变量正确地处理了所有并发同步细节,使用者**无需自己加锁**,从而大大降低了写出竞态/死锁 bug 的风险,也让生产者-消费者模式的代码简洁清晰。具体好处:①**内置线程安全**——多个生产者线程并发 `put`、多个消费者线程并发 `get` 都是安全的,不会损坏内部数据,无需你手动用 Lock 保护;②**阻塞与协调**——`get()` 在队列为空时会**阻塞等待**直到有元素(消费者不必忙等轮询),`put()` 在(有界)队列满时会**阻塞等待**直到有空位;这天然实现了线程间的协调;③**有界队列实现反压(backpressure)**——`Queue(maxsize=N)` 设置容量上限,当消费速度跟不上时,生产者会在 `put` 时阻塞,从而**自动限制生产速度、防止内存被无限堆积的任务撑爆**(这是用共享 list 难以优雅做到的);④**解耦**——生产者和消费者只通过队列交互,不直接共享其他可变状态,符合"用通信代替共享内存"的并发设计理念(类似 Go 的 channel),减少了需要加锁保护的共享状态;⑤**配套功能**——`task_done()`/`join()` 可以等待所有任务被处理完,便于优雅收尾;还有 `LifoQueue`(栈)、`PriorityQueue`(优先级)适配不同需求。相比之下,自己用"共享 list + Lock + Condition"手写生产消费,既容易写错(竞态、死锁、忘记 notify、忙等)又冗长。所以 Python 线程编程的最佳实践是:**尽量用 `queue.Queue` 在线程间传递数据/任务,把"共享可变状态 + 手动锁"降到最少**——这样既安全又清晰。

</details>

5. 直接用 threading.Thread 和用 ThreadPoolExecutor 有什么区别?

<details><summary>参考答案</summary>

**`threading.Thread`** 是底层 API——你手动创建每一个线程对象、`start()` 启动、`join()` 等待,自己管理线程的生命周期、数量、结果收集和异常处理。它灵活,但当任务很多时容易出问题:手动为成百上千个任务各建一个线程会**失控**(线程数过多耗资源、上下文切换开销大,且没有数量上限保护);收集每个线程的返回值、处理线程内抛出的异常都要自己额外想办法(线程函数的返回值不会自动传出、异常不会传播到主线程)。**`concurrent.futures.ThreadPoolExecutor`** 是更高层、更推荐的 API,它管理一个**固定大小的线程池**:①**限制并发数**——`max_workers` 控制同时运行的线程数,任务多于线程数时自动排队,避免线程数失控;②**复用线程**——线程被复用执行多个任务,省去频繁创建销毁的开销;③**统一提交与结果获取**——`submit(fn, *args)` 返回一个 **`Future`**,通过 `future.result()` 拿到返回值(若任务抛了异常,`result()` 会**重新抛出**该异常,从而异常不会被无声吞掉),`map()` 可批量提交,`as_completed()` 可按完成顺序迭代结果;④**优雅管理**——用 `with ThreadPoolExecutor() as ex:` 时,退出 with 块会自动等待所有已提交任务完成(`shutdown`),无需手动 join 每个线程;⑤**接口统一**——和 `ProcessPoolExecutor`(多进程,见 P21)接口一致,切换线程/进程只需换一个类。**结论**:除非有特殊的底层控制需求,**优先用 `ThreadPoolExecutor`**(以及更广义的 `concurrent.futures`,见 P24)来跑并发任务——它在并发数控制、线程复用、结果/异常处理、优雅关闭上都比手搓 Thread 省心且不易出错;手动 `Thread` 适合需要长期运行的单个后台线程或需要精细控制的特殊场景。

</details>

6. 多线程里的异常和守护(daemon)线程要注意什么?

<details><summary>参考答案</summary>

**异常方面**:①**子线程中抛出的异常不会传播到主线程**——如果用底层 `threading.Thread`,线程函数里抛的异常默认只会被线程自己处理(打印 traceback 到 stderr,Python 3.8+ 可用 `threading.excepthook` 定制),**主线程感知不到**,程序也不会因此崩溃,这意味着错误可能被悄悄忽略、任务静默失败。所以用 Thread 时应在**线程函数内部用 try/except 捕获并记录/上报**异常,或把异常通过队列/共享结构传回主线程。②**用 `ThreadPoolExecutor` 更好**——提交任务返回 `Future`,任务里的异常会被捕获并存起来,当你调用 `future.result()` 时**重新抛出**,从而能在主线程统一感知和处理异常(别忘了真的去 `result()`,否则异常仍被"吞")。**守护线程(daemon)方面**:①`thread.daemon = True`(必须在 `start()` 前设置)把线程标记为守护线程——**当所有非守护线程(包括主线程)结束时,守护线程会被解释器直接强制终止**,进程随之退出,不会等它们跑完;非守护线程则会**阻止进程退出**(主线程结束后进程仍会等所有非 daemon 线程结束)。②**守护线程的危险**:因为它们是被"强杀"的,**不保证执行清理逻辑、finally 块或释放资源、刷新缓冲**,可能导致数据丢失、文件/连接未正常关闭、状态不一致。所以**关键的、需要保证完成或正确清理的工作绝不能放在 daemon 线程里**;daemon 适合那些"进程退出时可以随时被丢弃"的纯后台辅助任务(如后台心跳、监控、缓存刷新),且要能容忍被中途杀死。③想要优雅停止线程,应使用**显式的停止信号**(如 `threading.Event`,线程循环里定期检查 `event.is_set()` 然后主动退出)+ `join()` 等待,而不是依赖 daemon 强杀或试图强行 kill 线程(Python 没有安全的强制杀线程的 API)。总结:子线程异常要自己捕获或用 Future 的 result 暴露;daemon 线程会被强杀、不保证清理,关键任务别用 daemon,优雅退出靠信号 + join。

</details>
