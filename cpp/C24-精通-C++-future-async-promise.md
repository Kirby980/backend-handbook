# 精通 C++ future / async / promise

> 课程编号：C24
> 路线图来源：现代 C++ 全栈深度课程 — 并发
> 难度：⭐⭐⭐⭐
> 预计阅读时间：55 分钟
> 📅 内容基准：**C++23** + GCC 14 / Clang 18 / MSVC 19.4x

---

## 引言：这段 std::async 是并行的吗？

```cpp
#include <future>
#include <chrono>
using namespace std::chrono_literals;

int slow() { std::this_thread::sleep_for(1s); return 42; }

void demo() {
    auto f1 = std::async(slow);            // 1) 一定开新线程吗？
    auto f2 = std::async(slow);
    int a = f1.get();                      // 2) 这里发生了什么？
    int b = f2.get();
}

void demo2() {
    std::async(std::launch::async, slow);  // 3) 这一行总耗时多少？
    std::async(std::launch::async, slow);  // 串行还是并行？
}
```

答案：① **不一定**——默认策略 `async|deferred` 让实现自由选择，可能开线程也可能延迟到 `get()` 时同步执行；② `f1.get()` 阻塞等待结果，取走后再 `f2.get()`，若两者都真异步则总耗时约 1s，若被 deferred 则是 2s 串行；③ **这一行总耗时约 2s 且串行**——`std::async` 返回的临时 `future` 在分号处析构，而 `launch::async` 的 future **析构时会阻塞等待任务完成**！第一个临时 future 析构阻塞 1s，第二个再阻塞 1s。这是 `std::async` 最臭名昭著的陷阱。

`future`/`promise`/`async`/`packaged_task` 是 C++11 引入的**基于"共享状态（shared state）"的异步结果传递机制**：把"计算"和"取结果"解耦，并能**跨线程传递异常**。它比裸 `thread`（C21）更高层——你拿到的是一个"将来会有值的凭据"。本章是协程（C25）之前最后一块并发拼图。

---

## 第一章：共享状态——所有机制的核心

`future`/`promise` 体系的底层是一个堆上分配的**共享状态（shared state）**对象，它持有：结果值（或异常）、就绪标志、一个条件变量、引用计数。

```mermaid
graph LR
    P["promise / packaged_task / async<br>(生产端)"] -->|写入值或异常| S["共享状态<br>shared state<br>值 / 异常 / ready 标志 / CV"]
    S -->|读取/阻塞等待| F["future / shared_future<br>(消费端)"]
    style P fill:#ed8936,color:#fff
    style S fill:#4299e1,color:#fff
    style F fill:#48bb78,color:#fff
```

- **生产端**（promise / packaged_task / async）：在某个时刻调用 `set_value` / `set_exception`，把结果写进共享状态并唤醒等待者。
- **消费端**（future）：`get()` / `wait()` 阻塞直到共享状态就绪，然后取出值（或重新抛出异常）。

理解"共享状态是一个带条件变量的堆对象"，就理解了为什么 `future` 只能 move 不能 copy（独占对共享状态的取值权），为什么 `get()` 会阻塞，为什么异常能跨线程传。

---

## 第二章：std::future 与 std::shared_future

`std::future<T>` 是消费端的句柄，**只能移动、不可拷贝**，`get()` **只能调用一次**（取走后共享状态被释放）。

```cpp
#include <future>

std::future<int> f = std::async(std::launch::async, []{ return 7; });

if (f.valid()) {                 // 是否关联共享状态
    f.wait();                    // 阻塞直到就绪（不取值）
    int x = f.get();             // 取值；之后 f.valid()==false
    // int y = f.get();          // ❌ UB：已经 get 过
}
```

`wait_for` / `wait_until` 用于带超时的等待：

```cpp
auto f = std::async(std::launch::async, slow);
if (f.wait_for(100ms) == std::future_status::ready) {
    use(f.get());
} else {
    // timeout / deferred：还没好
}
```

| `future_status` | 含义 |
|---|---|
| `ready` | 结果已就绪，可 `get` |
| `timeout` | 超时仍未就绪 |
| `deferred` | 任务是延迟执行（尚未启动），`get` 时才同步运行 |

**`std::shared_future<T>`** 允许**多次 get、可拷贝**——多个消费者共享同一结果（如"广播"一个一次性事件）：

```cpp
std::promise<int> p;
std::shared_future<int> sf = p.get_future().share();  // future → shared_future
auto sf2 = sf;                                        // 可拷贝
// 多个线程都能 sf.get() 读到同一个值（值被拷贝出来）
```

---

## 第三章：std::promise——手动设置结果

`promise<T>` 是生产端：你在一个线程里持有 promise，在另一个线程里持有它配出的 future，自己决定何时 `set_value`。

```cpp
#include <future>
#include <thread>

void producer(std::promise<int> p) {
    try {
        int result = compute();
        p.set_value(result);         // 写入值 → 唤醒消费端
    } catch (...) {
        p.set_exception(std::current_exception());  // 写入异常
    }
}

void demo() {
    std::promise<int> p;
    std::future<int> f = p.get_future();   // 在传走 promise 之前取 future
    std::thread t(producer, std::move(p)); // promise 只能 move
    int x = f.get();                       // 阻塞直到 producer 设值
    t.join();
}
```

关键规则：

- `get_future()` **只能调用一次**（再次调用抛 `future_error`）。
- promise 析构时若**从未 set 任何值**，共享状态会被置为带 `broken_promise` 错误的"就绪"——消费端 `get()` 会抛 `std::future_error{broken_promise}`，而不是永久卡死。
- `set_value` 也只能调一次。

`promise<void>` 用作一次性事件信号；`promise<T&>` 传引用。

---

## 第四章：std::packaged_task——把可调用对象包成任务

`packaged_task<R(Args...)>` 把一个可调用对象包装起来：**调用它**就等于运行任务并把返回值/异常写入共享状态。它本身像个函数对象，可以丢给线程、线程池、事件循环执行。

```cpp
#include <future>

std::packaged_task<int(int,int)> task([](int a, int b){ return a + b; });
std::future<int> f = task.get_future();

std::thread t(std::move(task), 2, 3);   // 在别处调用 task → 运行并 set_value
// 或：丢进线程池队列 later 执行
t.join();
int sum = f.get();                       // 5
```

**promise vs packaged_task vs async** 的关系：

| 工具 | 谁触发结果写入 | 典型用途 |
|---|---|---|
| `promise` | 你手动 `set_value` | 完全手控的一次性结果/事件 |
| `packaged_task` | **调用这个 task 对象**时自动写入返回值/异常 | 把"函数+future"打包丢进线程池 |
| `async` | 库帮你**启动并执行**函数 | 一句话发起异步计算 |

可以理解为：`async` ≈ "`packaged_task` + 自动起线程/调度"；`packaged_task` ≈ "`promise` + 自动捕获函数返回值和异常"。

---

## 第五章：std::async 与两个 launch 策略

```cpp
// 三种调用形式
auto f1 = std::async(f, args...);                       // 策略 = async | deferred（实现自选）
auto f2 = std::async(std::launch::async, f, args...);   // 强制新线程异步
auto f3 = std::async(std::launch::deferred, f, args...);// 延迟：get/wait 时在当前线程同步执行
```

| 策略 | 行为 |
|---|---|
| `launch::async` | **保证**在新线程上异步执行（立即启动） |
| `launch::deferred` | **不**起线程；任务被推迟，到 `get()`/`wait()` 时**在调用线程同步运行**；若从不 get 则**永不执行** |
| `async\|deferred`（默认） | 实现自由选择二者之一（可能受系统负载影响） |

**默认策略的危险**在于不确定性：

```cpp
auto f = std::async(heavy);     // 默认策略
// ... 做别的事，期望 heavy 在后台并行
do_other_work();
// 若实现选了 deferred，heavy 此刻根本没跑！直到下面 get 才同步执行
int r = f.get();                // 这里才真正运行 heavy → 没有任何并行
```

更糟的是带 `thread_local` 时：deferred 在调用 `get()` 的线程跑，async 在新线程跑，访问的 `thread_local` 变量完全不同。

**结论**：要并行就**显式写 `std::launch::async`**；要么干脆别用 `std::async`，改用线程池 + `packaged_task`。

```mermaid
graph TD
    A["std::async(策略, f)"] --> B{策略?}
    B -->|launch::async| C["立即新线程执行<br>future 析构时阻塞 join"]
    B -->|launch::deferred| D["不执行<br>get/wait 时当前线程同步跑"]
    B -->|默认 async\|deferred| E["实现自选<br>⚠️ 行为不确定"]
    style C fill:#48bb78,color:#fff
    style D fill:#ed8936,color:#fff
    style E fill:#f56565,color:#fff
```

---

## 第六章：async 返回 future 析构阻塞的陷阱

这是 `std::async` 设计中最反直觉的一点：**由 `std::async` 返回的、且策略为 `async` 的 future，其析构函数会阻塞，直到关联任务完成**（仿佛隐式 `join`）。

```cpp
{
    std::async(std::launch::async, slow);   // 临时 future，分号处析构 → 阻塞等 slow 完成
}   // 看起来"发射后不管"，实际是同步等待！

{
    auto f = std::async(std::launch::async, slow);   // 具名 future
    do_something();
}   // f 在作用域结束析构 → 仍然阻塞等待 slow（即使你不 get）
```

为什么这么设计？为了**避免悬空**：任务里若引用了局部变量，作用域结束后任务还在跑就会访问已销毁的对象。所以标准让 async 的 future 析构同步等待，保证安全（但牺牲了"火后不管"的直觉）。

**注意**：这个"析构阻塞"特性**只对 `std::async` 返回的 future 成立**。`promise::get_future()` 或 `packaged_task::get_future()` 得到的 future，析构**不会阻塞**（它们不拥有运行中的线程）。

引言中 `demo2` 两行串行就是这个原因：每个临时 future 在各自分号处析构并阻塞 1s，共 2s。

**规避**：

```cpp
// 想真正"火后不管"：用 detach 的 thread 或线程池，别用 std::async
std::jthread t(slow);   // C++20，析构自动 join（但仍会等）
// 或把 future 存活到你真正想等待的地方再 get
```

---

## 第七章：异常的跨线程传递

`future` 体系最实用的能力之一：**生产端抛出的异常会被捕获进共享状态，在消费端 `get()` 时原样重新抛出**——异常"穿越"了线程边界。

```cpp
int may_throw() {
    if (bad()) throw std::runtime_error("boom");
    return 1;
}

auto f = std::async(std::launch::async, may_throw);
try {
    int x = f.get();           // 若任务抛了异常，这里重新抛出
} catch (const std::runtime_error& e) {
    // 在消费线程捕获到生产线程抛的异常
}
```

底层机制：`async` / `packaged_task` 内部用 `try/catch(...)` 包住任务，捕获后调用 `set_exception(std::current_exception())`，把异常存进共享状态（用 `std::exception_ptr`）；`get()` 检测到存的是异常就 `std::rethrow_exception`。

手动版（promise）：

```cpp
try {
    p.set_value(work());
} catch (...) {
    p.set_exception(std::current_exception());  // 关键：捕获并转交
}
```

对比裸 `std::thread`：线程函数里抛出且未捕获的异常会直接 `std::terminate` 整个程序——**没有跨线程传递机制**。这正是 future 体系相对裸线程的核心优势之一。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：默认 std::async 当成"一定并行"

```cpp
auto f = std::async(heavy);   // 可能被 deferred，get 前根本没跑
```
要并行就 `std::async(std::launch::async, heavy)`。

### ❌ 陷阱 2：async 临时 future 析构阻塞（误以为火后不管）

```cpp
std::async(std::launch::async, slow);   // 分号处析构阻塞，等同同步！
std::async(std::launch::async, slow);   // 两行串行
```

### ❌ 陷阱 3：future 的 get 调用两次

```cpp
int a = f.get();
int b = f.get();   // ❌ UB（共享状态已释放）。要多次读用 shared_future
```

### ❌ 陷阱 4：promise 析构未 set 值

```cpp
std::promise<int> p;
auto f = p.get_future();
// ... 忘了 set_value，p 析构 → f.get() 抛 broken_promise（不是死等，但通常是 bug）
```

### ❌ 陷阱 5：先 move promise 再取 future

```cpp
std::promise<int> p;
std::thread t(producer, std::move(p));
auto f = p.get_future();   // ❌ p 已被移走，UB / 抛异常。必须先 get_future 再 move
```

### ❌ 陷阱 6：deferred + thread_local / 期望后台进度

```cpp
auto f = std::async(std::launch::deferred, monitor);  // 永远不在后台跑
// 直到 get() 才在当前线程同步执行
```

---

## 第九章：练习题

**练习 1**：`std::async` 默认（不指定 launch）策略下，任务一定会在新线程执行吗？为什么这是个坑？

**练习 2**：下面这段总耗时大约多少？为什么？
```cpp
std::async(std::launch::async, []{ std::this_thread::sleep_for(1s); });
std::async(std::launch::async, []{ std::this_thread::sleep_for(1s); });
```

**练习 3**：`future`、`shared_future`、`promise`、`packaged_task` 各自的角色是什么？`promise` 和 `packaged_task` 的区别？

**练习 4**：找 bug：
```cpp
std::promise<int> p;
std::thread t([&]{ p.set_value(42); });
auto f = p.get_future();
int x = f.get();
t.join();
```

**练习 5**：裸 `std::thread` 的线程函数抛出未捕获异常会怎样？`std::async` 的任务抛异常又会怎样？差别说明了什么？

---

## 参考答案与解析

**练习 1**：**不一定**。默认策略是 `launch::async | launch::deferred`，实现可自由选择。若选了 deferred，任务直到 `get()`/`wait()` 才在调用线程同步执行——你期望的后台并行根本没发生，且行为随平台/负载变化。要确定性并行必须显式 `std::launch::async`。

**练习 2**：**约 2 秒，串行**。两个 `std::async(launch::async, ...)` 返回临时 future，第一个临时 future 在它所在语句的分号处析构，而 async 的 future 析构会阻塞等待任务完成（1s）；之后第二个同理再等 1s。所以是串行 2s，而非并行 1s。若把 future 存到具名变量、最后一起 get，才能并行。

**练习 3**：`future`——消费端句柄，只移动、`get` 一次。`shared_future`——可拷贝、可多次 `get`，多消费者共享。`promise`——生产端，手动 `set_value`/`set_exception`。`packaged_task`——把可调用对象包装成"调用即写结果"的任务，便于丢给线程池。区别：promise 由你手动设置结果；packaged_task 在被调用执行时自动把函数返回值/异常写入共享状态（≈ promise + 自动捕获返回值/异常）。

**练习 4**：这段**实际是正确的**——`get_future()` 在 `set_value` 之前还是之后调用都行，只要在 `p` 被销毁前、且只调一次。这里 `p` 按引用被 lambda 捕获（没有 move），主线程仍持有 `p`，`get_future()` 合法。真正要警惕的是把 `p` **move** 进线程后再 `get_future()`。（若题目把 `[&]` 改成按值 move 捕获 `p`，则 `p.get_future()` 就是 UB。）

**练习 5**：裸 `std::thread`：线程函数中未捕获的异常会传播到线程顶层，触发 `std::terminate` 终止**整个进程**——异常无法跨线程传递。`std::async`/`packaged_task`：内部用 `catch(...)` + `set_exception` 捕获异常存入共享状态，消费端 `get()` 时 `rethrow_exception` 重新抛出。差别说明 future 体系提供了**跨线程异常传递**，这是它相对裸线程的关键安全优势。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 共享状态 | 堆上对象（值/异常/ready/CV）；future 只移动、get 一次 |
| future | 消费端；`get`/`wait`/`wait_for`；`valid()` |
| shared_future | 可拷贝、可多次 get；多消费者共享一次性结果 |
| promise | 手动 `set_value`/`set_exception`；先 `get_future` 再 move |
| packaged_task | 调用即写结果；适合丢线程池 |
| async 策略 | 要并行就显式 `launch::async`；默认不确定 |
| **析构陷阱** | **async 的 future 析构阻塞**等待任务（临时 future ⇒ 串行） |
| 异常 | 跨线程传递（`set_exception`/`exception_ptr` → get 重抛） |

---

## 📅 2026 现状/更新

- `std::future` 体系自 C++11 稳定，但被广泛认为"不可组合"——没有 `.then()` 链式延续、没有 `when_all/when_any`（这些在 Concurrency TS 里，未进标准）。
- C++20 **协程**（C25）是更现代的异步范式：可写出看起来同步的异步代码，避免 future 的析构陷阱与回调地狱。
- C++26 引入 **`std::execution`（发送者/接收者，sender/receiver）** 作为结构化、可组合的异步框架，被视为 future/async 的"接班人"（见 C32）。
- 实践准则：**简单一次性后台计算**可用 `std::async(launch::async, ...)`（注意持有 future）；**复杂异步**用线程池 + `packaged_task` 或协程；**别依赖默认 async 策略**。

---

> 🔁 下一篇 **C25 — 精通 C++ 协程**：`co_await`/`co_yield`/`co_return`、`promise_type`、`coroutine_handle`、awaiter（`await_ready`/`await_suspend`/`await_resume`）、生成器与 Task、协程帧分配与对称转移——看编译器如何把一个函数"切成"可暂停恢复的状态机。
>
> 反馈：把"async 的 future 析构会阻塞"和"默认策略可能 deferred"这两条钉死——它们是 90% 异步 bug 的源头。
