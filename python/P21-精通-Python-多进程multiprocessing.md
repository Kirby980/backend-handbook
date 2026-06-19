# 精通 Python 多进程 multiprocessing

> 多进程是 Python 在 GIL 时代利用多核做 CPU 并行的主力——每个进程有独立解释器和 GIL,能真正并行。代价是进程开销大、内存不共享、要序列化通信。面试问:"CPU 密集怎么并行?""进程间怎么通信?"本篇讲透,紧承 [P17 GIL](./P17-精通-Python-GIL原理.md),对比 [P20 线程](./P20-精通-Python-多线程threading.md)。
>
> **📅 基准:2026 年 6 月,CPython 3.13 / 3.14。**

---

## 一、为什么用多进程

GIL 让多线程无法并行计算(见 [P17](./P17-精通-Python-GIL原理.md))。**多进程绕过 GIL**:每个进程是独立的操作系统进程,**有自己独立的 Python 解释器和独立的 GIL**,因此多个进程能**真正并行**运行在多个 CPU 核上——这是 Python 做 **CPU 密集型并行**的传统主力。

```python
from multiprocessing import Process
def heavy(n): ...                  # CPU 密集计算
if __name__ == "__main__":          # 关键:必须放在 main guard 里
    ps = [Process(target=heavy, args=(i,)) for i in range(4)]
    for p in ps: p.start()
    for p in ps: p.join()
```

---

## 二、启动方式:fork vs spawn

子进程的创建方式影响行为(`multiprocessing.set_start_method`):

- **fork**(Linux 传统默认):复制父进程(写时复制),快;但**继承父进程的全部状态**,与多线程混用、持有锁时 fork 易出问题(子进程可能死锁)。
- **spawn**(Windows/macOS 默认,**3.14 起 Linux 也逐步转向 spawn 默认**):启动全新的解释器、重新 import 模块——更干净、更安全,但慢一些,且**要求被传递的对象可 pickle、代码放在 `if __name__ == "__main__"` 保护下**(否则重新 import 时会递归创建进程)。

**`if __name__ == "__main__":` 守护必不可少**(尤其 spawn):没有它,子进程重新导入主模块时会再次执行创建进程的代码,导致无限递归/报错。

---

## 三、进程间不共享内存

与线程最大的不同:**进程间内存相互隔离**,父子进程、兄弟进程**不共享**普通变量/对象。一个进程改了自己的全局变量,别的进程看不到。所以进程间要"通信"必须用专门的 **IPC** 机制:

- **`Queue`**(`multiprocessing.Queue`):进程安全队列,传递对象(底层走管道 + pickle 序列化)。
- **`Pipe`**:两个进程间的双向管道。
- 这些传递的数据都要**被 pickle 序列化/反序列化**(有开销)。

```python
from multiprocessing import Process, Queue
def worker(q): q.put(compute())
q = Queue()
p = Process(target=worker, args=(q,)); p.start()
result = q.get(); p.join()
```

---

## 四、共享内存

需要进程间共享数据时,用专门的共享内存机制(而非普通变量):

- **`Value` / `Array`**:共享单个值/数组(底层共享内存 + 锁),如 `Value("i", 0)`。
- **`shared_memory`(3.8+)**:共享内存块,配合 NumPy 可**零拷贝**共享大数组(传 NumPy 大数据时避免 pickle 拷贝,大幅提速)。
- **`Manager`**:提供可在进程间共享的 `list`/`dict` 等(通过代理,有网络/序列化开销,较慢但方便)。

```python
from multiprocessing import shared_memory
import numpy as np
shm = shared_memory.SharedMemory(create=True, size=arr.nbytes)
buf = np.ndarray(arr.shape, dtype=arr.dtype, buffer=shm.buf)   # 多进程共享同一块内存
```

---

## 五、进程池:Pool / ProcessPoolExecutor

批量并行任务用进程池,免去手动管理:

```python
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor(max_workers=4) as ex:
    results = list(ex.map(heavy_compute, items))   # 并行映射,自动分发到多进程
# 或 multiprocessing.Pool:
from multiprocessing import Pool
with Pool(4) as pool:
    results = pool.map(heavy_compute, items)
```

- **`ProcessPoolExecutor`**(`concurrent.futures`)与 `ThreadPoolExecutor` **接口一致**,切换线程/进程只改一个类(见 [P24](./P24-精通-Python-并发选型与free-threading.md)),推荐。
- 任务函数和参数、返回值都要**可 pickle**(见下)。

---

## 六、序列化(pickle)的限制与开销

多进程的"阿喀琉斯之踵"是**序列化**:进程间传递的一切(任务函数、参数、返回值)都要经 **pickle** 序列化后传输、再反序列化。

- **开销**:大对象序列化/拷贝/传输有可观成本——**适合"计算量大、数据传输少"的任务**;频繁传大数据或大量小任务,序列化开销可能吃掉并行收益。
- **限制**:**不是所有对象都可 pickle**——lambda、本地函数、打开的文件/socket、某些闭包不能 pickle,会报错(spawn 下尤甚)。任务函数要定义在模块顶层。
- **优化**:大数组用 `shared_memory`/NumPy 零拷贝共享;减少跨进程数据传递;任务粒度别太细(分块批处理)。

---

## 陷阱清单

- **CPU 密集才用多进程**:IO 密集用多进程是浪费(进程开销 + 序列化),用线程/asyncio(见 [P20](./P20-精通-Python-多线程threading.md)/[P22](./P22-精通-Python-asyncio与协程.md))。
- **忘记 `if __name__ == "__main__":`**:spawn 下子进程重新导入主模块会递归创建进程,崩溃。
- **以为进程间共享变量**:它们内存隔离;要共享用 Queue/共享内存/Manager。
- **传不可 pickle 的对象**:lambda、本地函数、文件句柄等不能跨进程传,报错。
- **频繁传大数据**:pickle 序列化开销大;用 shared_memory 或减少传输。
- **任务粒度太细**:大量小任务的调度/序列化开销 > 计算收益;分块。
- **fork + 多线程/持锁**:fork 一个持有锁或多线程的进程易死锁;混用时用 spawn。
- **进程不回收(僵尸/泄漏)**:用 `with Pool()`/Executor 或确保 join/close。

---

## 2026 现状

- **多进程仍是 CPU 密集并行的主力**;`ProcessPoolExecutor` 是推荐入口。计算密集库(NumPy/PyTorch)则在 C 层释放 GIL,多线程也能并行(见 [P17](./P17-精通-Python-GIL原理.md))。
- **`shared_memory` + NumPy** 用于大数组零拷贝跨进程共享,避免 pickle 瓶颈。
- **3.14 起 Linux 默认启动方式转向 spawn**(更安全、与 free-threading 等更兼容)——注意迁移时的 pickle/main guard 要求。
- **free-threading(去 GIL,见 [P17](./P17-精通-Python-GIL原理.md))/ 子解释器(InterpreterPoolExecutor,3.14)** 提供了"免 IPC 的进程内并行"新选择,长期可能减少对多进程的依赖(见 [P24](./P24-精通-Python-并发选型与free-threading.md))。
- 与 Go/Java 对照:它们靠线程就能多核并行、共享内存;Python 因 GIL 要靠多进程 + IPC,编程更重——这是 Python 并发的历史包袱,free-threading 正在改变它。

---

## 练习题

1. 为什么 CPU 密集型任务要用多进程而不是多线程?

<details><summary>参考答案</summary>

因为 **GIL** 的存在(见 P17)使得 CPython 中**多线程无法并行执行 Python 计算**——同一时刻只有一个线程能持有 GIL、跑字节码,所以多个线程做 CPU 密集计算只能被串行化轮流执行,得不到多核加速(还多了切换开销,常常更慢)。而**多进程能绕过 GIL**:每个进程是独立的操作系统进程,**拥有自己独立的 Python 解释器和独立的 GIL**,互不干扰,因此多个进程可以**真正并行地运行在多个 CPU 核心上**,各自全速跑自己的计算,从而把多核算力利用起来,实现 CPU 密集任务的并行加速。所以对于纯计算型任务(图像/视频处理、加解密、数值模拟、批量数据转换等),用 `multiprocessing` 或 `ProcessPoolExecutor` 把任务分发到多个进程并行,是 Python 利用多核的传统标准做法。**代价**:进程比线程**重**(创建/销毁开销大、各自占独立内存)、进程间**内存不共享**(要通过 IPC 通信)、传递数据要 **pickle 序列化**(有开销且有的对象不可序列化)。因此多进程**适合"计算量大、进程间通信少"的任务**;如果任务很小或需要频繁传大量数据,序列化和进程开销可能抵消甚至超过并行收益。**对比**:IO 密集型任务则相反——多线程(或 asyncio)就够了(IO 阻塞时释放 GIL,能有效并发),用多进程反而浪费资源。补充:除了多进程,CPU 密集还可借助**在 C 层释放 GIL 的库**(NumPy、PyTorch 等多线程并行计算)、**Cython/C/Rust 扩展**(释放 GIL)、或新兴的 **free-threading(无 GIL 构建)/子解释器**(见 P24)。

</details>

2. fork 和 spawn 两种进程启动方式有什么区别?为什么需要 `if __name__ == "__main__"`?

<details><summary>参考答案</summary>

`multiprocessing` 创建子进程有不同的"启动方式(start method)":①**fork**(类 Unix 系统上的传统默认,Linux 上长期默认):通过操作系统的 `fork()` **复制父进程**来创建子进程——子进程几乎完整地**继承父进程当时的内存状态**(全局变量、已导入的模块、打开的资源等,采用写时复制)。优点是**快**(不用重新初始化解释器);缺点是继承全部状态可能带来问题,尤其**当父进程是多线程的、或在 fork 时持有锁**时,子进程可能复制了一个"被锁住但永远不会被释放"的锁而**死锁**,与某些库(线程池、日志、CUDA 等)混用也容易出诡异 bug。②**spawn**(Windows 和 macOS 上的默认,**Python 3.14 起 Linux 也逐步转向以 spawn 为默认**):启动一个**全新的、干净的 Python 解释器进程**,只继承运行子进程所必需的最少资源,并会**重新 import 主模块**、通过 pickle 把目标函数和参数传过去。优点是**干净、安全、跨平台一致**(避免了 fork 继承状态的隐患);缺点是**慢一些**(要重启解释器、重新导入),且**要求传递的对象都可 pickle**、目标函数定义在可导入的位置。**为什么需要 `if __name__ == "__main__":`**:在 **spawn**(以及 Windows)下,子进程启动时会**重新导入(执行)主模块**以获得函数定义等。如果你创建子进程的代码(`Process(...).start()`、`Pool(...)` 等)是写在模块顶层、没有被 `if __name__ == "__main__":` 保护的,那么子进程在重新导入主模块时**会再次执行这些"创建子进程"的代码**,于是又去创建新的子进程……造成**无限递归地 fork 子进程**(报错 RuntimeError 或进程爆炸)。把入口逻辑放在 `if __name__ == "__main__":` 块里,可以保证这段代码**只在直接运行主程序时执行一次**,而在被子进程作为模块导入时**不执行**,从而避免递归创建。所以用 multiprocessing 时(尤其要跨平台/用 spawn),**务必把启动进程的代码放进 main guard**,并确保任务函数和参数可 pickle。

</details>

3. 进程间为什么不能像线程那样直接共享变量?有哪些通信/共享方式?

<details><summary>参考答案</summary>

因为**进程之间的内存是相互隔离的**——每个操作系统进程拥有自己独立的虚拟地址空间,一个进程里的普通变量/对象存在它自己的内存中,**其他进程无法直接访问**(这正是进程隔离带来安全性与稳定性的基础)。这与**线程**不同:同一进程内的多个线程**共享同一块内存**(全局变量、堆对象),所以线程间可以直接读写共享变量(但要加锁,见 P20)。因此多进程中,一个进程修改自己的全局变量,别的进程是看不到的,要在进程间传递数据/状态必须借助显式的**进程间通信(IPC)**或共享内存机制:①**`multiprocessing.Queue`**——进程安全的队列,可在进程间传递 Python 对象(底层通过管道传输 + pickle 序列化),是生产者-消费者式通信的常用方式;②**`Pipe`**——两个进程之间的双向连接(管道),点对点通信;③**共享内存原语 `Value` / `Array`**——在共享内存中存放单个值或定长数组(配套有锁),供多进程读写简单的数值/数组;④**`shared_memory`(3.8+)**——分配一块命名共享内存,多个进程可映射访问同一块内存,配合 NumPy 可实现**大数组的零拷贝共享**(避免 pickle 大数据的开销,适合传大矩阵);⑤**`Manager`**——启动一个管理进程,提供可在进程间共享的"代理"对象(如 `Manager().list()`、`Manager().dict()`),用起来像普通容器,但每次访问都经过进程间通信(序列化 + 转发),**方便但较慢**;⑥更广义的还有文件、数据库、Redis、socket 等外部媒介。注意:通过 Queue/Pipe/Manager 传递的对象都要**可 pickle**(lambda、文件句柄等不行),且序列化有开销。所以设计多进程程序时要**尽量减少跨进程的数据传递量**(传大数据用 shared_memory 零拷贝),并选"计算多、通信少"的任务划分。

</details>

4. 多进程的 pickle 序列化带来哪些限制和开销?如何应对?

<details><summary>参考答案</summary>

多进程中,进程间传递的一切——提交给进程池的**任务函数、它的参数、以及返回值**,以及通过 Queue/Pipe 传的数据——都需要在发送端用 **pickle 序列化**成字节、传输到另一进程、再反序列化还原。这带来两类问题:**①限制(不是所有对象都能 pickle)**:`lambda` 匿名函数、定义在函数内部的**嵌套/本地函数**、某些闭包、打开的**文件句柄/socket/锁/数据库连接**、以及一些含不可序列化属性的对象,都**不能被 pickle**,尝试跨进程传递会报 `PicklingError`/`AttributeError`(在 spawn 模式下更严格)。这也是为什么**进程池的任务函数必须定义在模块顶层**(可被重新导入/按限定名 pickle),不能用 lambda 或局部函数。**②开销**:序列化、数据拷贝、进程间传输、再反序列化都要花时间和内存,**对象越大开销越大**;如果频繁在进程间传递大量数据(或有大量小任务、每个都要序列化任务和结果),序列化开销可能**抵消甚至超过**多核并行带来的收益,使多进程不划算。**应对办法**:①**选对任务**——多进程适合"**计算量大、通信数据少**"的任务(传少量参数、返回少量结果、中间大量计算),避免"频繁传大数据"的模式;②**减少跨进程数据**——只传必要的输入/输出,别把大对象来回搬;能在子进程内部读取/生成的数据就别从父进程 pickle 过去;③**大数组用共享内存零拷贝**——用 `multiprocessing.shared_memory` 配合 NumPy 共享大数组,避免对大矩阵做 pickle 拷贝;④**增大任务粒度(分块)**——把很多小任务合并成较大的批,减少任务数从而减少序列化次数和调度开销;⑤**确保可 pickle**——任务函数放模块顶层、用具名函数而非 lambda、避免传不可序列化的资源(在子进程里自己打开文件/连接);⑥必要时**用更快的序列化**(如 `cloudpickle` 支持更多对象、或用 Apache Arrow/共享内存传大数据)。总之:多进程的成本主要在 IPC 序列化,设计上要"少传数据、传得高效、任务够大"。

</details>

5. `ProcessPoolExecutor` 和 `ThreadPoolExecutor` 有什么关系和区别?

<details><summary>参考答案</summary>

二者都属于标准库 `concurrent.futures`,提供**统一、一致的高层并发接口**:都用 `submit(fn, *args)` 提交任务(返回 `Future`,用 `future.result()` 取结果、异常会在 result 时重新抛出)、用 `map(fn, iterable)` 批量并行映射、用 `as_completed` 按完成顺序收集、都支持 `with ... as ex:` 上下文管理(退出时自动等待所有任务完成并关闭),也都用 `max_workers` 控制并发数。它们的接口**几乎完全相同**,这正是 `concurrent.futures` 的设计目标——让你能**只改一个类名就在"多线程"和"多进程"之间切换**(见 P24)。**区别在于底层执行单元**:**`ThreadPoolExecutor`** 用**线程池**执行任务——线程共享进程内存(传参/返回值无需序列化、通信方便),但受 **GIL** 限制,**只适合 IO 密集型**(IO 阻塞时释放 GIL 并发有效),不能并行做 CPU 计算;**`ProcessPoolExecutor`** 用**进程池**执行任务——每个工作进程有独立解释器和 GIL,能**真正并行做 CPU 计算**(绕过 GIL),**适合 CPU 密集型**,但代价是进程开销大、内存不共享、任务函数/参数/返回值都要**可 pickle 且经序列化传输**(有开销和限制,见上),且(spawn 下)需要 `if __name__ == "__main__"` 保护。**选择**:**IO 密集**(网络请求、文件、DB)用 `ThreadPoolExecutor`(轻量、共享内存、无序列化);**CPU 密集**(纯计算)用 `ProcessPoolExecutor`(真并行)。因为接口一致,实践中可以先用线程池验证逻辑、再按任务性质切换到进程池。注意 ProcessPoolExecutor 的任务函数要定义在模块顶层、参数和结果要可 pickle、避免频繁传大数据(否则序列化开销吃掉并行收益)。

</details>

6. 多进程、多线程、asyncio 在内存共享和适用场景上有何根本不同?(综合)

<details><summary>参考答案</summary>

三者是 Python 并发的三条路径,差异源于"是否共享内存、是否受 GIL、调度方式":①**多线程(threading)**——同一进程内的多个线程**共享内存**(全局变量、堆对象),通信方便但**需要加锁保护共享可变状态**(见 P20);受 **GIL** 限制,**同一时刻只有一个线程跑字节码**,所以**只适合 IO 密集型**(IO 阻塞时释放 GIL 实现并发),不能并行做 CPU 计算;由操作系统抢占式调度;线程比进程轻。②**多进程(multiprocessing)**——多个**独立进程**,**内存相互隔离、不共享**(要靠 Queue/Pipe/共享内存等 IPC 通信,数据要 pickle 序列化,有开销和限制);每进程独立 GIL,能**真正并行利用多核**,**适合 CPU 密集型**;进程开销大、启动慢;由操作系统调度。③**asyncio(协程)**——**单线程、单进程**内的**协作式**并发(见 P22):由事件循环在一个线程里调度成千上万个协程,协程在 `await` 一个 IO 操作时**主动让出**控制权给其他协程,从而用极少的资源实现**超高并发的 IO**;因为是单线程,**不存在多线程的数据竞争问题**(在 await 点之间的代码是原子的,共享状态相对安全,但仍要注意),也**几乎没有线程/进程切换开销和内存开销**;但它**只适合 IO 密集**(不能利用多核做 CPU 并行,且一个阻塞调用会卡死整个事件循环),且需要全链路异步(库要支持 async)。**适用场景总结**:**CPU 密集型 → 多进程**(或释放 GIL 的库 / free-threading,见 P24);**IO 密集型、并发量中等、想复用同步代码 → 多线程**;**IO 密集型、超高并发(大量连接/请求)、追求资源效率 → asyncio**。**内存模型**:线程共享内存(需锁)、进程隔离内存(需 IPC)、asyncio 单线程(协作让出、无并行竞争但要避免阻塞)。三者也可组合(如多进程 + 每进程内 asyncio)。选型详见 P24。

</details>
