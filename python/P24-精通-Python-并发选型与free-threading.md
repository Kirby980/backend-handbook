# 精通 Python 并发选型与 free-threading

> 线程、进程、asyncio、free-threading——Python 并发模型多到让人选择困难。选错不仅性能差,还可能完全白做(如多线程跑 CPU 密集)。本篇给出决策框架,并讲清 **free-threading(去 GIL)** 如何改变这套逻辑。是模块四的总纲,综合 [P17 GIL](./P17-精通-Python-GIL原理.md)、[P20 线程](./P20-精通-Python-多线程threading.md)、[P21 进程](./P21-精通-Python-多进程multiprocessing.md)、[P22 asyncio](./P22-精通-Python-asyncio与协程.md)。
>
> **📅 基准:2026 年 6 月。free-threading:3.13 实验、3.14 官方支持(可选构建)。**

---

## 一、决策树:先问"IO 还是 CPU"

选并发模型,第一刀切在**任务是 IO 密集还是 CPU 密集**:

```
任务主要在【等待】(网络/磁盘/DB)? → IO 密集
   ├─ 超高并发(上万连接)、有异步生态 → asyncio(P22)
   └─ 并发量中等、想复用同步代码/库  → 多线程(P20)

任务主要在【计算】(加解密/压缩/数值)? → CPU 密集
   ├─ 传统方案 → 多进程(P21)/ 释放 GIL 的库(NumPy)
   └─ 2026 新选择 → free-threading 构建(多线程真并行)
```

**根因是 GIL**(见 [P17](./P17-精通-Python-GIL原理.md)):它让多线程不能并行计算,所以 CPU 密集必须绕过它(多进程/释放 GIL 的 C 库/去 GIL)。IO 密集则不受 GIL 限制(阻塞时释放),线程和 asyncio 都行。

---

## 二、四种模型对比

| 维度 | 多线程 | 多进程 | asyncio | free-threading |
|---|---|---|---|---|
| 内存 | 共享(需锁) | 隔离(IPC) | 共享(单线程) | 共享(需锁) |
| 并行 CPU | ❌(GIL) | ✅(多解释器) | ❌(单线程) | ✅(无 GIL) |
| 适合 | IO 密集 | CPU 密集 | 高并发 IO | CPU 密集 |
| 开销 | 中(线程) | 高(进程+序列化) | 低(协程) | 中(线程+同步) |
| 通信 | 共享变量/Queue | pickle/共享内存 | 共享/Queue | 共享变量/Queue |
| 心智 | 锁、竞态 | 序列化、main guard | 染色、不能阻塞 | 锁、竞态(真并行更严) |

---

## 三、concurrent.futures:统一接口

`concurrent.futures` 用**同一套接口**封装线程池和进程池,切换只改一个类——这是跑批量并发任务的推荐入口:

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# IO 密集:线程池
with ThreadPoolExecutor(max_workers=20) as ex:
    results = list(ex.map(fetch, urls))

# CPU 密集:进程池(只改类名)
with ProcessPoolExecutor(max_workers=8) as ex:
    results = list(ex.map(heavy_compute, items))
```

`submit`→`Future`、`map`、`as_completed` 接口一致(见 [P20](./P20-精通-Python-多线程threading.md)/[P21](./P21-精通-Python-多进程multiprocessing.md))。`Future.result()` 取结果/抛异常。先用线程池验证逻辑,按需切进程池。

---

## 四、free-threading:改变选型逻辑

**free-threading(无 GIL 构建,PEP 703,见 [P17](./P17-精通-Python-GIL原理.md))** 是 2026 的关键变量:

- **3.13 实验、3.14 官方支持**(仍是**可选构建**,默认发行版仍带 GIL)。
- **它让多线程能真正并行做 CPU 计算**——不再需要多进程 + IPC 来利用多核!共享内存、无序列化开销,对"计算 + 频繁共享数据"的任务尤其有吸引力。
- **但代价与注意**:①单线程性能有一定损耗(细粒度同步);②C 扩展需适配;③**真并行后,共享可变状态的竞态更危险——更需要正确加锁**(去 GIL ≠ 自动线程安全)。

**对选型的影响**:一旦成熟普及,"CPU 密集 → 必须多进程"的铁律会松动——多线程 + free-threading 可能成为更轻量的 CPU 并行方案(免去进程开销和 pickle)。当前仍处过渡期,生产仍以多进程/释放 GIL 的库为主。

---

## 五、子解释器:另一条并行路线

- **per-interpreter GIL(PEP 684,3.12)**:让一个进程内可以有**多个子解释器,每个有自己独立的 GIL**——于是子解释器之间能并行执行 Python 代码,且比多进程轻(同进程、可更高效共享)。
- **`InterpreterPoolExecutor`(3.14)**:`concurrent.futures` 新增的执行器,把任务分发到**多个子解释器**并行跑——介于线程(共享但受 GIL)和进程(并行但重 IPC)之间的折中。
- 这是除 free-threading 外、CPython 实现"进程内并行"的另一条路线,适合想要并行又不想付多进程完整开销的场景(但子解释器间共享仍有限制)。

---

## 六、实战权衡与组合

- **典型 Web 服务**:用 asyncio(FastAPI,见 [P28](./P28-精通-Python-Web框架与WSGI-ASGI.md))扛高并发 IO;偶发 CPU 活(如生成 PDF、压缩)用 `run_in_executor` 丢进程池(见 [P23](./P23-精通-Python-异步实战与陷阱.md))。
- **数据处理 / 批计算**:CPU 密集用多进程 `ProcessPoolExecutor` 或 NumPy 向量化 / Polars(释放 GIL 多线程,见 [P19](./P19-精通-Python-性能剖析与优化.md))。
- **爬虫 / 批量 API 调用**:IO 密集,asyncio + `Semaphore` 限并发,或线程池。
- **组合**:常见"多进程 + 每进程内 asyncio"(如 Gunicorn 多 worker 进程,每个跑 Uvicorn 异步事件循环,见 [P28](./P28-精通-Python-Web框架与WSGI-ASGI.md))——用多进程吃满多核、用 asyncio 在每核内扛高并发 IO。
- **不确定先测**:用真实负载压测对比,别凭感觉(见 [P19](./P19-精通-Python-性能剖析与优化.md))。

---

## 陷阱清单

- **CPU 密集用多线程或 asyncio**:GIL/单线程下无并行,白做;用多进程/向量化/free-threading。
- **IO 密集用多进程**:进程开销 + 序列化浪费;用线程或 asyncio。
- **以为 asyncio 能加速 CPU**:它是单线程,CPU 活会霸占事件循环。
- **协程里阻塞 / 进程间传大数据**:分别毁掉并发 / 吃掉并行收益(见 [P23](./P23-精通-Python-异步实战与陷阱.md)/[P21](./P21-精通-Python-多进程multiprocessing.md))。
- **以为 free-threading = 不用锁**:真并行竞态更严重,**更**要加锁(见 [P17](./P17-精通-Python-GIL原理.md))。
- **盲目混用模型**:线程 + 多进程 + 异步乱炖,复杂难调;按层次清晰组合。
- **不压测就选型/调参**:并发数、模型都该用真实负载验证。

---

## 2026 现状

- **现阶段主流选型**:IO 密集高并发 → asyncio;IO 密集普通 → 线程池;CPU 密集 → 多进程 / NumPy 等释放 GIL 的库。`concurrent.futures` 是统一入口。
- **free-threading(3.14 官方支持)** 正在改变格局,生态适配进行中;尝鲜需评估单线程损耗与 C 扩展兼容,**默认仍是带 GIL 版本**。
- **子解释器 `InterpreterPoolExecutor`(3.14)** 提供进程内并行的新折中。
- **Rust 加持的库(Polars 等)** 在数据处理上用原生多线程绕开 GIL,是另一条高性能路线(见 [P19](./P19-精通-Python-性能剖析与优化.md))。
- 与 Go/Java 对照:它们靠"线程/协程 + 真并行 + 共享内存"统一解决并发([G11](../golang/INDEX.md)/[J11](../java/INDEX.md)/[J30](../java/INDEX.md)),模型更简单;Python 长期要在"线程/进程/asyncio"间权衡,free-threading 有望简化这一局面。

---

## 练习题

1. 选择 Python 并发模型时,最关键的判断依据是什么?给出决策框架。

<details><summary>参考答案</summary>

**最关键的判断依据是:任务是 IO 密集型还是 CPU 密集型**(因为这决定了 GIL 是否成为障碍,见 P17)。决策框架:**第一步,判断任务性质**——任务的时间主要花在"等待外部 IO"(网络请求、磁盘读写、数据库、等待响应)还是"做计算"(加解密、压缩、数值运算、图像处理、解析大量数据)?**第二步,按性质选模型**:①**IO 密集型**——GIL 在 IO 阻塞时会释放,所以多线程和 asyncio 都能有效并发。若是**超高并发**(要同时维持上万连接/请求)、且有配套的**异步生态**(aiohttp/asyncpg 等),选 **asyncio**(单线程协程,资源效率最高,见 P22);若并发量中等、想**复用现成的同步代码/库**、或不想引入异步的"染色"复杂度,选**多线程**(ThreadPoolExecutor,见 P20)。②**CPU 密集型**——GIL 使多线程无法并行计算,必须绕过它:传统选 **多进程**(ProcessPoolExecutor,每进程独立 GIL 真并行,见 P21)或使用**在 C 层释放 GIL 的库**(NumPy/PyTorch 向量化、Polars 多线程,见 P19);2026 的新选择是 **free-threading(无 GIL 构建)** 用多线程真并行(见下)。**第三步,考虑工程因素**:数据共享需求(线程/asyncio 共享内存方便、多进程要 IPC 序列化)、并发规模、是否有阻塞库、团队熟悉度、以及通过**真实负载压测**验证(别凭感觉,见 P19)。**统一入口**用 `concurrent.futures`(ThreadPoolExecutor / ProcessPoolExecutor 接口一致,切换只改类名)。**组合**:复杂服务常"多进程 + 每进程内 asyncio"(多核 + 高并发 IO 兼得)。一句话:**先分 IO/CPU——IO 用线程或 asyncio,CPU 用多进程或释放 GIL 的方案**;再按并发规模、生态、共享需求细化。

</details>

2. 多线程、多进程、asyncio 在"内存共享"和"能否并行 CPU"上分别如何?

<details><summary>参考答案</summary>

①**多线程**:**内存共享**——同一进程内的线程共享全部内存(全局变量、堆对象),通信方便,但因此需要用锁保护共享可变状态以避免竞态(见 P20)。**并行 CPU:不能**——受 GIL 限制,同一时刻只有一个线程执行 Python 字节码,无法在多核上并行计算(只对 IO 密集有效,因 IO 阻塞时释放 GIL)。②**多进程**:**内存隔离**——每个进程有独立地址空间,**不共享**普通变量,进程间通信要用 IPC(Queue/Pipe/共享内存),传数据要 pickle 序列化(有开销和限制,见 P21)。**并行 CPU:能**——每进程有独立解释器和独立 GIL,可真正并行运行在多核上,是传统的 CPU 密集并行方案。③**asyncio**:**单线程、共享内存**——所有协程跑在一个线程里,共享内存;因为是协作式调度、只在 await 点切换,竞态比多线程少(但跨 await 的读改写仍需 asyncio.Lock)。**并行 CPU:不能**——单线程,且 CPU 密集会霸占事件循环(没有 await 让出点),把整个服务卡死;它只解决高并发 IO(见 P22)。**对照总结**:多线程=共享内存+不能并行CPU(擅长IO);多进程=隔离内存(需IPC)+能并行CPU(擅长CPU);asyncio=单线程共享+不能并行CPU(擅长高并发IO、最省资源)。这也直接决定了选型:并行 CPU 计算→多进程(或释放 GIL 的库/free-threading);高并发 IO→asyncio;普通 IO 并发/复用同步代码→多线程。注意三者可组合(如多进程吃多核 + 每进程内 asyncio 扛 IO 并发)。

</details>

3. `concurrent.futures` 提供了什么便利?

<details><summary>参考答案</summary>

`concurrent.futures` 提供了一套**高层、统一、易用的并发任务执行接口**,屏蔽了直接操作线程/进程的繁琐细节,主要便利有:①**统一的 Executor 抽象**——`ThreadPoolExecutor`(线程池)和 `ProcessPoolExecutor`(进程池)拥有**完全一致的 API**,这意味着你可以**只改一个类名**就在"多线程"和"多进程"之间切换(IO 密集用前者、CPU 密集用后者),无需重写逻辑,便于先用线程验证、再按任务性质切换。②**池化管理**——通过 `max_workers` 自动管理固定数量的工作线程/进程并复用它们、任务超出则排队,避免手动创建大量线程/进程导致资源失控,也省去手写线程生命周期管理。③**`Future` 对象统一结果与异常处理**——`submit(fn, *args)` 返回一个 `Future`,`future.result()` 取返回值,且**任务中抛出的异常会在调用 `result()` 时被重新抛出**(不会像裸线程那样被无声吞掉),`future.done()`/`cancel()`/`add_done_callback()` 等便于状态管理;`as_completed(futures)` 可按**完成顺序**迭代结果,`map(fn, iterable)` 可批量并行映射(保持输入顺序返回)。④**优雅的上下文管理**——`with Executor() as ex:` 在退出时自动 `shutdown`(等待所有已提交任务完成并释放资源),无需手动 join。⑤**3.14 新增 `InterpreterPoolExecutor`**——把任务分发到多个子解释器(per-interpreter GIL)并行,提供线程与进程之间的第三种折中(见下)。总之它把"提交任务→并发执行→收集结果/异常→优雅关闭"这一整套用统一接口封装好,是 Python 跑批量并发任务的**推荐入口**,比手动管理 `threading.Thread`/`multiprocessing.Process` 省心且不易出错(见 P20/P21)。

</details>

4. free-threading 会如何改变 Python 的并发选型?现在能用了吗?

<details><summary>参考答案</summary>

**free-threading(无 GIL 的 CPython 构建,PEP 703,见 P17)的核心改变是:让多线程能够真正并行执行 Python 计算**,从而**松动"CPU 密集任务必须用多进程"这条长期铁律**。在有 GIL 的世界里,要利用多核做 CPU 并行只能用多进程(每进程独立 GIL),但多进程有明显代价:进程创建开销大、**内存隔离要靠 IPC 通信、数据要 pickle 序列化**(传大数据尤其昂贵)、还要处理 main guard 等。free-threading 下,多线程**共享内存且能并行计算**,意味着 CPU 密集任务可以用**更轻量的多线程方案**——免去进程开销、免去序列化、可直接共享内存中的大对象(对"计算 + 频繁共享/交换数据"的工作负载尤其有吸引力)。所以一旦成熟普及,选型逻辑会变成:CPU 密集既可多进程、也可"多线程 + free-threading",后者更轻;并发模型整体向"靠线程统一解决"(类似 Java/Go)靠拢。**现在能用了吗?**——**部分能,但仍在过渡期**:①**3.13(2024)实验性引入**了无 GIL 的可选构建,**3.14(2025)将其提升为"官方支持"**,但**仍然是一个可选的独立构建**,**默认发行的 CPython 依然带 GIL**,你要专门安装/使用 free-threading 版本;②**注意事项**:free-threading 构建的**单线程性能有一定下降**(细粒度同步开销,社区在优化)、**C 扩展需要适配**才能在无 GIL 下安全工作(NumPy 等主流库正陆续支持,但生态迁移没完成)、并且**你的 Python 代码反而更需要正确加锁**——真并行后多线程同时改共享可变状态的竞态比有 GIL 时更真实危险,"去 GIL ≠ 自动线程安全"。所以**当前生产实践仍以多进程 / 释放 GIL 的库(NumPy/Polars)为 CPU 并行主力**,free-threading 适合评估和尝鲜,需权衡单线程损耗和扩展兼容性。长期看它是 Python 补齐"多核并行"短板的方向。

</details>

5. 什么是子解释器和 per-interpreter GIL?它和 free-threading、多进程有什么关系?

<details><summary>参考答案</summary>

**子解释器(subinterpreter)**是在**同一个进程内**创建多个相对独立的 Python 解释器实例的机制。**per-interpreter GIL(PEP 684,Python 3.12)**是关键改进:它让**每个子解释器拥有自己独立的 GIL**(而不是整个进程共用一个 GIL)。这带来的效果是:**不同子解释器可以在同一进程内、在不同 CPU 核上并行执行 Python 代码**(因为它们各自的 GIL 互不阻塞),从而绕过"一个进程一个 GIL"的并行限制。**`InterpreterPoolExecutor`(Python 3.14)**是 `concurrent.futures` 新增的执行器,把任务分发到一个**子解释器池**中并行执行,提供了便捷的使用入口。**与多进程、free-threading 的关系/定位**——它是介于二者之间的"第三条并行路线":①**相比多进程**——子解释器在**同一进程内**,比创建独立进程**更轻量**(无需 fork/spawn 整个进程、可更高效地共享某些资源),启动和某些通信成本更低;但子解释器之间默认**仍是相当隔离的**(为了各自独立的状态,对象不能随意直接共享,跨解释器通信也有限制和开销,需要专门的通道),所以不像多线程那样能自由共享内存。②**相比 free-threading(无 GIL)**——free-threading 是**彻底去掉 GIL**、让普通多线程在共享内存下真并行(最灵活但单线程性能受损、C 扩展要适配);子解释器则是**保留 GIL 但每解释器一个**、用"多个隔离解释器"换并行(隔离性强、各解释器内部仍是熟悉的带 GIL 单线程语义,但共享数据麻烦)。③它们都是 CPython 实现"利用多核"的不同尝试。**适用场景**:子解释器适合那些**任务之间相对独立、不需要大量共享可变状态**、又想要"比多进程轻、能并行"的场景(如并行处理相互独立的请求/任务)。目前(2026)子解释器及其执行器仍较新、生态和易用性在完善中,实践中 CPU 并行主流仍是多进程/释放 GIL 的库,free-threading 和子解释器是正在成熟的新选项。

</details>

6. 给几个典型场景,说说你会选哪种并发模型。

<details><summary>参考答案</summary>

①**高并发 Web/API 服务(大量并发请求,主要在等下游 IO:DB、缓存、其他服务)**——选 **asyncio**(用 FastAPI/Starlette 等 ASGI 框架,见 P28),单线程协程能用很少资源扛海量并发连接;配套用异步库(asyncpg、httpx、redis.asyncio);偶发的 CPU 活(生成 PDF、图片处理)用 `run_in_executor` 丢进程池,避免堵事件循环。生产部署常用"**多进程(每核一个 worker)+ 每进程内 asyncio**"(如 Gunicorn 管多个 Uvicorn worker),既吃满多核又每核高并发。②**爬虫 / 批量调用大量外部 API(IO 密集)**——asyncio + `asyncio.Semaphore` 限制并发数(避免打爆对方/被限流);或简单点用 `ThreadPoolExecutor`(线程池)并发请求(代码改动小、复用 requests 等同步库)。③**CPU 密集的批处理(图像/视频处理、加解密、大量纯计算)**——多进程 `ProcessPoolExecutor` 分发到多核并行;若是数值计算优先 **NumPy 向量化 / Polars**(释放 GIL、多线程,见 P19);可评估 free-threading。④**数据科学/大规模数值计算**——NumPy/pandas/Polars 向量化为主(把循环下推到底层并行的 C),必要时多进程(Dask/Ray)做分布式。⑤**少量长期运行的后台任务(如监控、心跳、定时刷新)配合主程序**——用多线程(threading.Thread/线程池)即可,简单且共享内存方便。⑥**既要并行计算又要频繁共享大量数据**——这正是 free-threading(共享内存 + 真并行,免 IPC,见 P17)或子解释器有吸引力的场景(当前过渡期可先用多进程 + 共享内存/shared_memory)。**通用原则**:先分清 IO/CPU(IO→线程/asyncio,CPU→多进程/向量化),再按并发规模(超高并发→asyncio)、是否有异步生态、共享数据需求来定;用 `concurrent.futures` 作统一入口;拿不准就用真实负载压测对比(见 P19)。

</details>
