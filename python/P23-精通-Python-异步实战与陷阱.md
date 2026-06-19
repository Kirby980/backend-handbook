# 精通 Python 异步实战与陷阱

> 会写 `async/await` 只是入门——生产里的异步要处理取消、超时、并发控制、阻塞调用隔离、异步生态选型。本篇是 [P22 asyncio](./P22-精通-Python-asyncio与协程.md) 的实战延伸,聚焦"怎么写对、怎么不踩坑",对接 [P28 FastAPI](./P28-精通-Python-Web框架与WSGI-ASGI.md)、[P29 异步 DB](./P29-精通-Python-数据库访问与ORM.md)。
>
> **📅 基准:2026 年 6 月,CPython 3.13 / 3.14,TaskGroup/asyncio.timeout 主流。**

---

## 一、结构化并发:TaskGroup 优先

并发跑一组任务,**优先用 `TaskGroup`(3.11+)**而非裸 `gather`——它能在出错时自动取消其余任务、不泄漏:

```python
async def main():
    async with asyncio.TaskGroup() as tg:        # 结构化并发
        tg.create_task(fetch("a"))
        tg.create_task(fetch("b"))
    # 退出时:等所有完成;任一失败→取消其余→抛 ExceptionGroup
```

旧写法 `gather` 仍可用,但失败时其余任务不会自动取消(可能泄漏);`gather(..., return_exceptions=True)` 收集异常而不中断。处理 TaskGroup 抛出的 `ExceptionGroup` 用 `except*`(见 [P08](./P08-精通-Python-异常处理.md))。

---

## 二、取消与超时

异步任务必须能被取消、能超时,否则一个慢请求拖垮全局:

```python
# 超时(3.11+ 推荐 asyncio.timeout 上下文)
async with asyncio.timeout(5):       # 5 秒内没完成就取消并抛 TimeoutError
    await slow_operation()

# 旧式:wait_for
result = await asyncio.wait_for(slow_operation(), timeout=5)

# 取消会在协程内抛 CancelledError —— 清理放 try/finally
async def task():
    try:
        await work()
    except asyncio.CancelledError:
        cleanup()                    # 释放资源
        raise                        # 必须重新抛出!别吞掉取消
    finally:
        await close()
```

要点:**取消通过在 `await` 点抛 `CancelledError` 实现**;协程应在 `finally` 里清理资源,且**捕获 `CancelledError` 后要重新 raise**(吞掉它会破坏取消机制)。

---

## 三、异步同步原语

多个协程并发访问共享资源,用 **asyncio 版**的同步原语(不是 `threading` 的!):

```python
lock = asyncio.Lock()                # 异步锁
async with lock:                     # 临界区
    await update_shared()

sem = asyncio.Semaphore(10)          # 限制并发数(如最多 10 个并发请求)
async def fetch_limited(url):
    async with sem:
        return await fetch(url)

q = asyncio.Queue()                  # 异步队列(协程间生产消费)
```

注意:虽然 asyncio 单线程、`await` 点之间的代码不会被打断(竞态比多线程少),但**跨 `await` 的"读-改-写"仍可能交错**(中间让出给别的协程),需要 `asyncio.Lock` 保护;限并发用 `asyncio.Semaphore`。

---

## 四、阻塞调用:丢给 executor

**铁律(见 [P22](./P22-精通-Python-asyncio与协程.md)):事件循环线程上不能有阻塞调用。** 无法避免的同步阻塞/CPU 活,丢给线程/进程池:

```python
import asyncio
# 阻塞 IO(同步库)→ 线程池
result = await asyncio.to_thread(requests.get, url)        # 3.9+ 简便写法
# 或 loop.run_in_executor(None, blocking_func, *args)

# CPU 密集 → 进程池(见 P21)
loop = asyncio.get_running_loop()
result = await loop.run_in_executor(process_pool, heavy_compute, data)
```

- 同步阻塞 IO(没有异步版的库)→ `asyncio.to_thread` / 线程池 executor。
- CPU 密集 → **进程池** executor(线程池受 GIL,见 [P17](./P17-精通-Python-GIL原理.md))。
- 这样阻塞发生在别的线程/进程,事件循环不被堵。

---

## 五、异步生态与 async with / async for

asyncio 要发挥作用,**整条链路都要异步**——用异步库:

| 用途 | 异步库 | 别用(同步,会阻塞) |
|---|---|---|
| HTTP 客户端 | `aiohttp` / `httpx`(async) | `requests` |
| PostgreSQL | `asyncpg` / SQLAlchemy async | 同步 psycopg(阻塞) |
| Redis | `redis.asyncio` | 同步 redis |
| Web 框架 | FastAPI / Starlette(见 [P28](./P28-精通-Python-Web框架与WSGI-ASGI.md)) | Flask 同步视图 |

```python
async with aiohttp.ClientSession() as session:   # 异步上下文管理器
    async with session.get(url) as resp:
        data = await resp.json()

async for line in aiofiles_stream:               # 异步迭代器
    process(line)
```

`async with`(异步上下文管理器,`__aenter__/__aexit__`)、`async for`(异步迭代器,`__anext__`)、异步生成器(`async def` + `yield`)把协程能力扩展到资源管理和流式数据。

---

## 六、调试与常见错误

- **`asyncio` debug 模式**:`asyncio.run(main(), debug=True)` 或 `PYTHONASYNCIODEBUG=1`——能检测"协程未 await""阻塞太久的回调"等问题。
- **"coroutine was never awaited" 警告**:调用了协程函数但没 await/没调度——忘了 `await` 或 `create_task`。
- **"Task was destroyed but it is pending"**:任务没等完就被丢弃(常因 `create_task` 没保存引用被 GC)。
- **事件循环卡顿**:多半是混入了阻塞调用;用 debug 模式或 `loop.slow_callback_duration` 排查。

---

## 陷阱清单

- **协程里混入同步阻塞调用**:`requests`/`time.sleep`/同步 DB 卡死事件循环。用异步库或 `to_thread`/executor。
- **用 `threading.Lock` 保护协程**:应该用 `asyncio.Lock`;混用线程原语会阻塞或失效。
- **吞掉 `CancelledError`**:`except CancelledError: pass` 破坏取消;清理后要 `raise`。
- **`create_task` 不保存引用**:任务被 GC,"凭空消失"。保存引用或用 TaskGroup。
- **裸 `gather` 失败不取消其余**:任务泄漏继续跑;用 TaskGroup,或正确处理。
- **没有超时**:一个慢下游拖垮全局;用 `asyncio.timeout`/`wait_for`。
- **在协程里跑 CPU 密集**:霸占事件循环;丢进程池。
- **同步代码里 `await` / 嵌套 `asyncio.run`**:`await` 只能在 async 函数里;`run` 不能嵌套。
- **未限并发打爆下游**:几万协程同时请求;用 `asyncio.Semaphore` 限流。

---

## 2026 现状

- **结构化并发(TaskGroup)+ `asyncio.timeout` + `except*`** 是现代异步的标准写法,取代裸 gather/wait_for。
- **`asyncio.to_thread`(3.9)** 是把阻塞调用塞进线程的简便首选;CPU 活用进程池 executor。
- **异步生态成熟**:FastAPI + httpx + asyncpg/SQLAlchemy async + redis.asyncio 是高并发服务典型栈;`uvloop` 提速事件循环。
- **结构化并发库**(如 AnyIO、早期 Trio 的理念)影响了 asyncio 的演进,AnyIO 让代码可同时跑在 asyncio/Trio 上。
- 与 [P30](./P30-精通-Python-版本特性演进.md):异步是 Python 近十年最大的范式演进之一,仍在持续打磨(可读性、取消语义、与 free-threading 的关系)。

---

## 练习题

1. 为什么推荐用 TaskGroup 而不是裸 gather?

<details><summary>参考答案</summary>

`asyncio.TaskGroup`(3.11+)提供**结构化并发**,相比裸 `asyncio.gather` 在**错误处理、任务生命周期管理、避免泄漏**上更健壮。问题出在 `gather` 的语义:`await asyncio.gather(c1, c2, c3)` 并发跑多个任务,但**当其中一个任务抛出异常时,gather 会立即把该异常向上抛出,而其余仍在运行的任务并不会被自动取消**——它们会继续在后台跑(成为"孤儿任务"),其结果/异常被丢弃,可能浪费资源、产生副作用、甚至引发"Task was destroyed but it is pending"之类的问题,即**任务泄漏**;而且 gather 一次只抛出一个异常(其余任务的异常丢失,除非用 `return_exceptions=True` 收集,但那又不会中断)。**TaskGroup 解决了这些**:用 `async with asyncio.TaskGroup() as tg:` + `tg.create_task(...)`,它把这组任务作为一个**有明确边界的整体**管理——①退出 `async with` 块时**保证等待所有任务完成**(不会有任务被遗漏/泄漏);②**任一任务失败时,自动取消组内其余所有任务**,然后把发生的(可能多个)异常**聚合成一个 `ExceptionGroup` 抛出**(用 `except*` 分类处理,见 P08),不丢失异常信息;③整体的取消会向下传播给子任务,语义清晰。这正是"结构化并发"的思想(类似 Java 21 的 StructuredTaskScope,见 J30):并发任务的生死被约束在一个作用域内,要么全部成功、要么统一失败并清理,绝不泄漏。所以现代 asyncio 代码**优先用 TaskGroup**;`gather` 适合简单的、明确不需要"一个失败取消其余"的并发汇总场景(且要注意异常处理)。

</details>

2. 异步任务的取消和超时怎么处理?捕获 CancelledError 要注意什么?

<details><summary>参考答案</summary>

**超时**:推荐用 **`asyncio.timeout`(3.11+)** 上下文管理器——`async with asyncio.timeout(5): await op()`,如果块内操作在 5 秒内没完成,会**取消该操作并抛出 `TimeoutError`**;旧式用 `asyncio.wait_for(coro, timeout=5)` 也能达到类似效果(超时则取消并抛 TimeoutError)。给可能慢的下游调用加超时很重要,否则一个卡住的请求会长期占用资源、拖累整体。**取消机制**:asyncio 的取消是通过**在协程当前挂起的 `await` 点抛出 `asyncio.CancelledError`** 来实现的——当一个 Task 被 `task.cancel()`(或被 TaskGroup/timeout 取消)时,事件循环会在它下一次处于 await 状态时往里抛 CancelledError,协程因此从该处"被中断"。**捕获 CancelledError 的注意事项**:①**清理资源应放在 `try/finally` 或捕获 CancelledError 的块里**——确保被取消时也能释放锁、关闭连接、回滚等;②**捕获了 CancelledError 后必须重新 `raise`(不要吞掉它)**——`except asyncio.CancelledError: cleanup(); raise`。如果你 `except asyncio.CancelledError: pass` 把它吞掉,就**破坏了取消语义**:调用方以为任务被取消了,实际任务却继续运行(或被认为正常结束),导致逻辑混乱、TaskGroup/timeout 的联动取消失效、资源无法正确回收;③**CancelledError 在 3.8+ 直接继承自 `BaseException`** 而非 `Exception`,正是为了**避免被笼统的 `except Exception` 误吞**(见 P08)——所以写 `except Exception` 兜底业务异常时不会意外拦住取消,这是好事,但也意味着你若确实要在取消时清理,得**显式** `except asyncio.CancelledError`;④清理本身若需要 await,要小心它也可能再被取消(可用屏蔽手段如 `asyncio.shield` 保护关键清理,但要谨慎)。总结:用 `asyncio.timeout`/`wait_for` 加超时;取消靠 CancelledError 在 await 点抛出;在 finally 里清理资源;捕获 CancelledError 后务必重新 raise,绝不吞掉。

</details>

3. 为什么协程里要用 asyncio.Lock 而不是 threading.Lock?asyncio 里还需要锁吗?

<details><summary>参考答案</summary>

**要用 `asyncio.Lock` 而不是 `threading.Lock`**,因为二者的"阻塞/等待"机制完全不同。`threading.Lock` 的 `acquire()` 在锁被占用时会**阻塞当前线程**(让线程真正停下来等待);如果在协程里用它并发生争用,就会**阻塞整个事件循环所在的线程**——这等于在协程里做了阻塞调用,会卡死所有其他协程(见 P22 的阻塞陷阱),完全违背 asyncio 的协作式模型。而 `asyncio.Lock` 是**异步感知**的:它的 `acquire` 是一个协程(`async with lock:`),当锁被占用时,它会**挂起当前协程、让出控制权给事件循环**(而不是阻塞线程),等锁释放时再唤醒——这样等待锁的过程不会堵住事件循环,其他协程仍能运行。所以**协程里的同步必须用 asyncio 版的原语**(`asyncio.Lock`、`asyncio.Semaphore`、`asyncio.Event`、`asyncio.Queue`),不能用 threading 版。**asyncio 里还需要锁吗?**——**有时需要**。好消息是:asyncio 是**单线程协作式**的,代码只在显式的 `await` 点才会切换到别的协程,所以**两个 await 之间的同步代码是"原子"的、不会被打断**,这消除了多线程那种"任意字节码间被抢占"的细粒度竞态,**大多数情况下不需要锁**。但是,如果一段逻辑**跨越了 await**(即"读取共享状态 → await 某操作 → 再根据之前读到的值修改共享状态"),那么在那个 await 让出期间,**别的协程可能插进来修改同一共享状态**,造成竞态——这种"跨 await 的读-改-写"就需要用 `asyncio.Lock` 保护临界区,保证整个序列的互斥。此外 `asyncio.Semaphore` 常用来**限制并发数**(如最多 N 个协程同时访问某资源/打某下游),`asyncio.Queue` 用于协程间安全地生产消费。总结:协程内同步用 asyncio 原语(它们让出而非阻塞);单线程让竞态减少但"跨 await 的共享状态修改"仍需 asyncio.Lock;限并发用 asyncio.Semaphore。

</details>

4. 在异步程序里必须调用一个阻塞的同步函数(如 requests、重计算),怎么办?

<details><summary>参考答案</summary>

不能直接在协程里调用它(会阻塞事件循环、卡死所有协程,见 P22),正确做法是**把这个阻塞调用"外包"到事件循环之外的线程或进程去执行,然后 await 其结果**,这样阻塞发生在别处、事件循环本身不被堵塞。具体:①**阻塞的同步 IO**(如必须用同步的 `requests.get`、某个没有异步版的同步库、同步文件读写):用 **`asyncio.to_thread(func, *args)`(3.9+,最简便)** 或 **`loop.run_in_executor(None, func, *args)`**,把该函数丢到**线程池**里运行——因为它主要是在等 IO(IO 期间会释放 GIL,见 P17),用线程就够了,事件循环可以 await 这个线程任务而继续服务其他协程。例如 `data = await asyncio.to_thread(requests.get, url)`。②**CPU 密集的重计算**:线程池受 GIL 限制无法真正并行计算(见 P17),所以应丢到**进程池**——`loop.run_in_executor(process_pool_executor, heavy_compute, data)`(传入一个 `concurrent.futures.ProcessPoolExecutor`),让计算在独立进程里并行跑、不占事件循环线程的 CPU,再 await 结果(注意进程池有 pickle 序列化开销,见 P21)。③更优的做法当然是**优先寻找异步替代库**(aiohttp/httpx 替代 requests、asyncpg 替代同步 DB、aiofiles 等),让 IO 天然异步、不需要外包线程;只有在确实没有异步版本、或是 CPU 计算时才用 executor。**关键原则**:事件循环所在线程上**只能跑非阻塞、能在 await 点让出的代码**;任何会"占住线程不放"的阻塞 IO 或重计算,都要通过 `to_thread`/`run_in_executor` 转移到线程池(阻塞 IO)或进程池(CPU 计算)去做,主协程用 `await` 等它完成。这样既不破坏事件循环的高并发,又能整合必须用的同步代码。

</details>

5. 用 asyncio 时为什么"整条链路都要异步"?混入同步库会怎样?

<details><summary>参考答案</summary>

因为 asyncio 的高并发依赖"**每个协程在等 IO 时主动让出、让事件循环去推进别的协程**"这一机制,而这要求 IO 操作本身是**异步的(awaitable、非阻塞)**才能正确让出。**如果在协程的执行路径中混入了同步阻塞库**(如用同步的 `requests`、同步数据库驱动、`time.sleep`、同步文件 IO),这些调用**不会让出控制权、而是直接占住事件循环所在的唯一线程干等**——结果就是在它阻塞的整段时间里,**事件循环无法运行任何其他协程**,所有并发任务全部停滞,服务表现为卡顿/无响应/吞吐崩塌(见 P22)。一个阻塞调用就能毁掉整个异步程序的并发性。所以"要么全异步、要么别用 asyncio":**调用链上的每一层、用到的每个 IO 库都必须是异步的**,才能保证从头到尾的让出不被打断。这就是异步编程的所谓"**传染性/染色(function coloring)**"——一旦某个底层操作要异步,调用它的函数就得是 `async def`、调用方又得 `await`……异步性会沿调用链向上"传染",最终往往整条链路、整个框架都得是异步的(同步函数不能直接 await 异步函数)。**实践含义**:用 asyncio 就要选用配套的异步生态——`aiohttp`/`httpx`(异步)代替 `requests`,`asyncpg`/异步 ORM 代替同步 DB 驱动,`redis.asyncio` 代替同步 redis,`aiofiles` 代替同步文件操作,Web 用 FastAPI/Starlette(ASGI)而非同步 Flask 视图(见 P28/P29);**实在必须用的同步库**,则要用 `asyncio.to_thread`/`run_in_executor` 把它隔离到线程/进程池(见上题),绝不能直接在协程里调用。正是这种"全链路异步 + 染色"的负担,让一些人觉得 asyncio 心智成本高,也是 Java 虚拟线程(同步写法、自动让出,见 J30)、Go goroutine 被认为更易用的原因之一。

</details>

6. 异步编程有哪些常见错误信号?如何调试?

<details><summary>参考答案</summary>

常见错误信号及含义:①**`RuntimeWarning: coroutine '...' was never awaited`**——你**调用了协程函数但没有 await 它、也没用 create_task/gather 调度它**,于是协程对象被创建却从未执行(白调用了)。修复:加 `await`,或用 `asyncio.create_task`/`gather`/TaskGroup 调度。②**`Task was destroyed but it is pending!`**——一个任务还没运行完就被销毁了,通常是因为用 `create_task` 创建了任务**但没有保存它的引用**,导致它被垃圾回收;或事件循环结束时仍有未完成任务。修复:保存 task 引用、或用 TaskGroup 管理生命周期、或在退出前 await/取消它们。③**事件循环卡顿/吞吐骤降/超时**——多半是**混入了阻塞调用或 CPU 密集代码**堵住了事件循环。④**`RuntimeError: ... attached to a different loop` / 不能嵌套 run**——跨事件循环使用对象、或在已运行的循环里又调 `asyncio.run`。⑤**取消不生效/资源没释放**——可能吞掉了 CancelledError。**调试手段**:①**开启 asyncio 调试模式**——`asyncio.run(main(), debug=True)` 或设环境变量 `PYTHONASYNCIODEBUG=1`,它会**检测"未被 await 的协程""执行时间过长的回调/任务(疑似阻塞)""选择器慢"等问题并发出警告**,是定位阻塞和遗漏 await 的利器;可调 `loop.slow_callback_duration` 设阈值。②**用日志/`asyncio.current_task()`、`asyncio.all_tasks()`** 观察有哪些任务在跑、是否泄漏。③**定位阻塞**——用 `py-spy`(见 P19)采样看线程到底卡在哪个同步调用上;或在 debug 模式下看"slow callback"警告指向的代码。④**结构上预防**——统一用 TaskGroup 管理任务(避免泄漏)、给所有外部调用加 `asyncio.timeout`、阻塞调用一律走 `to_thread`/executor、用 `asyncio.Semaphore` 限并发。⑤**测试**——用 `pytest-asyncio` 写异步测试(见 P26)。总之:开 debug 模式抓"未 await/阻塞过久/任务泄漏"是最有效的第一步,再结合 py-spy 和任务自省定位具体问题。

</details>
