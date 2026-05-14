# 精通 Goroutines 与 GMP 调度

> 课程编号：G11
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Goroutines 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：60 分钟

---

## 引言：一段 100 万个并发

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1_000_000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println(runtime.NumGoroutine())
}
```

启动 100 万个 goroutine，内存峰值仅约 2-3 GB（每个初始栈 2KB），完成不超过几秒。同样的事用 OS 线程做绝对不可能。Go 调度器到底是怎么做到的？这一章把 G/M/P 模型、栈伸缩、抢占、syscall 解绑、work stealing 一一拆开。

---

## 第一章：goroutine 是什么

### 1.1 用户态轻量级线程

每个 goroutine 是一个 `g` 结构（`runtime/runtime2.go`），包含：

- 自己的栈（初始 2KB，可扩展至 1GB）
- 程序计数器 PC、栈指针 SP
- 状态（运行 / 可运行 / 等待 / 死亡）
- 关联的 M 和 P 指针
- 私有数据（defer 链、panic 链、goroutine 本地 stats）

**大小**：每个 `g` 结构约 400 字节 + 初始栈 2KB ≈ 2.5KB。

### 1.2 启动开销

```go
go fn()
```

约 ~1µs（runtime 调用 `runtime.newproc` 分配 g 并放入运行队列）。对比 OS 线程的 `pthread_create` 约 10-100µs，goroutine 启动便宜 10-100 倍。

### 1.3 与 OS 线程的关系

Go 程序使用 **M:N 调度**——M 个 goroutine 映射到 N 个 OS 线程。N 由 `GOMAXPROCS` 决定（默认等于 CPU 核数）。

---

## 第二章：G/M/P 模型

### 2.1 三个核心抽象

- **G**（Goroutine）：用户编写的并发单元
- **M**（Machine）：OS 线程；执行机器指令的实体
- **P**（Processor）：逻辑处理器；持有可运行 G 的本地队列

```
      G G G G G G G G            ← 用户创建的 goroutine
       \  \  \  \  /
        runqueue (local)         ← P 持有的本地队列
            |
        P  P  P  P               ← GOMAXPROCS 个 P
        |  |  |  |
        M  M  M  M               ← OS 线程
        |  |  |  |
       OS CPU CORES              ← 物理 CPU
```

### 2.2 为什么有 P

早期版本（Go 1.0）只有 G 和 M，全局一个 runqueue。多核扩展性差（锁竞争）。Go 1.1 引入 P，每个 P 自己有 local runqueue，几乎无锁。

P 还持有 mcache（per-P 内存分配缓存），让小对象分配也无锁。

### 2.3 状态机

P 的状态：`_Pidle`、`_Prunning`、`_Psyscall`、`_Pgcstop`、`_Pdead`

G 的状态：`_Gidle`、`_Grunnable`、`_Grunning`、`_Gsyscall`、`_Gwaiting`、`_Gdead`

调度循环：M 选一个 P → 从 P 的 runq 拿 G → 执行 G → G 结束/挂起 → 拿下一个 G

---

## 第三章：本地队列与 work stealing

### 3.1 每个 P 的 local runq

P 的本地队列容量 256 个 G。生产者（如新 `go` 启动）优先放到当前 P 的 local runq。

```
P0 runq: [G1, G2, G3]
P1 runq: [G4, G5]
P2 runq: [] ← 空了
P3 runq: [G6, G7, G8, G9, G10]
```

### 3.2 work stealing

当 P 的 local runq 为空，M 不会闲着：

1. 先尝试从**全局队列**取（global runq，所有 P 共享，需要全局锁）
2. 然后做 **work stealing**——随机选一个其他 P，从它的 local runq 偷一半 G

```
P0: []                  P3: [G6, G7, G8, G9, G10]
     ↑                                       |
     └────── steal half ─────────────────────┘
P0: [G9, G10]           P3: [G6, G7, G8]
```

这种"主动找活干"避免任意 M 空转。

### 3.3 全局队列

全局队列吸收：
- 长时间阻塞的 G 唤醒后无明确归属
- local runq 满了的溢出
- 网络轮询器准备好的 G

全局队列访问要锁，所以是后备路径。

### 3.4 调度时机

什么时候发生调度（即 M 从一个 G 切到另一个 G）？

- G 主动 `runtime.Gosched()`
- G 阻塞（channel、syscall、锁、time.Sleep）
- G 触发抢占
- G 函数返回结束

---

## 第四章：抢占机制

### 4.1 协作式抢占（Go 1.13 之前）

早期 Go 是**协作式调度**——只在**函数调用栈检查点**才能切换。一个紧循环不调用任何函数就永远占用 M：

```go
go func() { for {} }()   // Go 1.13 前会卡住一个 CPU 不放
```

`runtime.NumGoroutine()` 增长正常，但其他 goroutine 拿不到 CPU。

### 4.2 异步抢占（Go 1.14+）

Go 1.14 引入**基于信号的异步抢占**：

- runtime 周期性（约 10ms）向占用 CPU 太久的 M 发 SIGURG 信号
- 信号处理器修改被打断的 G 的状态为可抢占
- M 在适当的安全点跳回 schedule()

现在那个紧循环不会再独占 CPU。

### 4.3 不可抢占点

仍有些代码段不能被抢占：
- runtime 关键代码（持有 GC 信号量等）
- 没有"安全点"的汇编/CGO 代码

CGO 调用尤其要注意——长时间的 C 函数会卡住 M 直到 C 返回。

---

## 第五章：syscall 与 P 解绑

### 5.1 阻塞 syscall

当 G 进入阻塞 syscall（如 `read()`、`write()`）：

1. M 在 syscall 中挂起，等待 OS 返回
2. runtime **解绑 M 和 P**——P 可以被另一个 M 拿去跑别的 G
3. syscall 返回后，M 试图重新拿到一个 P；如果没有空 P，把当前 G 放入全局队列，自己进入休眠

这是为什么"一个 goroutine 阻塞不会冻结整个程序"。

### 5.2 网络 I/O 不阻塞 M

网络 I/O 走 **netpoller**（基于 epoll/kqueue/IOCP），M 立刻挂起 G 并被释放给别的 G 用。当网络数据到达，netpoller 把对应 G 重新放进 runqueue。

```go
// 这个 goroutine 阻塞在 Read，但 M 不被占用
go func() {
    var buf [1024]byte
    conn.Read(buf[:])   // M 已经去跑别的 G 了
}()
```

这是 Go HTTP 服务器能轻松扛 C10M 的关键。

### 5.3 创建 M

如果所有 P 都被解绑的 M 占用且新 G 还在等：runtime 自动**创建新的 M**。M 池上限是 10000（`GoMaxThreads`）。

```go
import _ "runtime/debug"
debug.SetMaxThreads(10000)   // 实际设置
```

如果应用大量阻塞 syscall，M 数量可能远超 GOMAXPROCS。

---

## 第六章：栈伸缩

### 6.1 初始 2KB

每个 goroutine 初始栈 2KB（远小于 OS 线程的 8MB 默认栈）。**100 万个 goroutine = ~2GB**，可以接受。

### 6.2 栈检查

每个函数序言（prologue）有"栈检查"：检测剩余栈是否足够。不够则触发栈增长。

### 6.3 栈复制

栈增长的实现：

1. 分配新栈（通常 2 倍）
2. 把整个旧栈内容复制到新栈
3. 修复所有指向旧栈内部的指针（runtime 扫描寄存器和栈本身）
4. 释放旧栈

这是为什么 Go 不允许"用 uintptr 长期保存栈地址"——栈搬迁后地址失效。

### 6.4 最大栈

默认上限 1GB（`runtime/debug.SetMaxStack`）。超出 panic"stack overflow"。

### 6.5 栈收缩

GC 时如果栈使用率 < 1/4，会**收缩到一半**，释放内存。这是 goroutine 长期空闲后内存能回落的原因。

---

## 第七章：GOMAXPROCS

### 7.1 含义

`GOMAXPROCS` = P 数量 = **同时执行 Go 代码的 OS 线程上限**。注意：不是 M 上限，因 syscall 等可以解绑 P。

### 7.2 默认值

Go 1.5+ 默认 = `runtime.NumCPU()` = OS 报告的逻辑核心数。

### 7.3 容器陷阱

Linux container 通过 cgroup 限制 CPU。但 `runtime.NumCPU()` 看到的是宿主机核数——容器内 GOMAXPROCS 设置过高导致频繁上下文切换。

**Go 1.24 及之前**：用 `go.uber.org/automaxprocs` 库自动从 cgroup 读取 quota（Uber 出品，长期社区方案）。

**Go 1.25+**：runtime 内置 **container-aware GOMAXPROCS**。Linux 上自动读 cgroup v1 (`cpu.cfs_quota_us` / `cpu.cfs_period_us`) 或 v2 (`cpu.max`)，按 quota / period 向上取整（如 1.5 核 → 2）。**而且会周期性重检**（默认 ≤ 1s/次），所以 K8s 在线改 limit 也能跟上。

> 仅当 `go.mod` 声明 `go 1.25` 或更高，且未显式设置 `GOMAXPROCS` 环境变量或调用 `runtime.GOMAXPROCS()` 时生效。可用 `GODEBUG=containermaxprocs=0` 退回旧行为，或 `updatemaxprocs=0` 关掉周期性更新。CPU **request**（cgroup `cpu.shares`/`cpu.weights`）不影响——只看 **limit**（quota）。

> 参考：[Container-aware GOMAXPROCS（Go 官方博客）](https://go.dev/blog/container-aware-gomaxprocs)、[Go 1.25 release notes — Runtime](https://go.dev/doc/go1.25#runtime)。注意旧资料常误传"Go 1.22+ 容器自动适配"——**Go 1.22 仅修了循环变量语义，没有 cgroup 感知**，请勿混淆。

### 7.4 手动设置

```go
runtime.GOMAXPROCS(4)
```

或环境变量 `GOMAXPROCS=4`。一般无需手动调整——默认是好的。

---

## 第八章：goroutine 泄漏

### 8.1 典型场景

```go
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch   // 永远阻塞，没人 send
    }()
    return   // ch 没人引用，但 goroutine 仍在
}
```

调用 `leak()` N 次 → N 个永久阻塞的 goroutine。每个占用 ~2.5KB + 栈。运行几小时可能耗尽内存。

### 8.2 检测方法

```go
import _ "net/http/pprof"

go http.ListenAndServe("localhost:6060", nil)
// curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

会列出所有 goroutine 的 stack trace 和当前阻塞位置。同样调用栈出现成千上万次 = 找到泄漏点。

或在测试中：

```go
func TestNoLeak(t *testing.T) {
    before := runtime.NumGoroutine()
    doWork()
    runtime.GC()
    time.Sleep(100 * time.Millisecond)
    after := runtime.NumGoroutine()
    if after > before {
        t.Errorf("leaked %d goroutines", after-before)
    }
}
```

更严格：`go.uber.org/goleak`。

### 8.3 防止泄漏的设计原则

- 每个长任务接受 `context.Context`
- 永远不要在没有 timeout/cancel 的情况下 `<-ch`
- channel 关闭由发送方负责
- 使用 `select` + `<-ctx.Done()` 兜底

---

## 第九章：实用 API

### 9.1 runtime.GOMAXPROCS

```go
old := runtime.GOMAXPROCS(2)
defer runtime.GOMAXPROCS(old)
```

### 9.2 runtime.Gosched

```go
runtime.Gosched()   // 主动让出 CPU 给其他 G
```

紧循环里偶尔调用，配合协作式调度。Go 1.14+ 异步抢占后用得很少。

### 9.3 runtime.LockOSThread

```go
runtime.LockOSThread()
// 当前 goroutine 永远绑定到当前 OS 线程
```

用于：
- `syscall.Setuid` 等线程局部状态
- CGO 调用必须在固定线程（如 OpenGL 上下文）
- 不希望 runtime 在线程间迁移

### 9.4 runtime.NumGoroutine / NumCPU

```go
fmt.Println(runtime.NumGoroutine())   // 当前 g 数量
fmt.Println(runtime.NumCPU())          // CPU 核数
```

---

## 第十章：生产级最佳实践

1. **不要无限创建 goroutine**：用 worker pool 或 semaphore 限制并发。
2. **每个长生命周期 goroutine 加 context**：能优雅退出。
3. **每个 goroutine 顶部 defer recover**：防止 panic 杀进程。
4. **channel 关闭由发送方负责**：接收方关闭可能 panic。
5. **避免 goroutine 持有大对象引用**：会阻止 GC。
6. **网络 I/O 自动非阻塞**：尽情用 goroutine 处理每个连接。
7. **CPU 密集任务限制到 GOMAXPROCS 个 goroutine**：超过 = 上下文切换浪费。
8. **生产监控 NumGoroutine**：突然飙升通常是泄漏。
9. **容器部署设 GOMAXPROCS** 或升级 Go 1.25+（cgroup-aware）。
10. **pprof goroutine endpoint 是金标准**：随时能看泄漏。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：循环里 go func 捕获循环变量（Go 1.21 前）
```go
for i := 0; i < 10; i++ {
    go func() { print(i) }()   // 常 10 10 10...
}
```

### ❌ 陷阱 2：永远阻塞的 send/recv
```go
ch := make(chan int)
go func() { ch <- 1 }()
return   // goroutine 泄漏
```

### ❌ 陷阱 3：不限制并发耗尽内存
```go
for _, url := range urls {
    go fetch(url)   // 100 万 URL = 100 万 goroutine
}
```

### ❌ 陷阱 4：跨 goroutine 共享变量不加锁
data race，用 `-race` 检查。

### ❌ 陷阱 5：CGO 长函数阻塞 M
C 函数没有 Go 安全点，M 被霸占。

### ❌ 陷阱 6：goroutine 内 panic 不 recover → 整个程序挂
每个 goroutine 顶部都要 defer recover。

### ❌ 陷阱 7：容器中默认 GOMAXPROCS 错误
跑了 32 个 P 但实际限制 2 核 → 频繁上下文切换。用 automaxprocs 或 Go 1.25+。

---

## 第十二章：练习题

**练习 1**：以下输出？说明原因。
```go
go fmt.Println("hi")
```

**练习 2**：写一个 worker pool，固定 N 个 goroutine 处理 channel 中的任务。

**练习 3**：以下函数有何问题？修复。
```go
func process(urls []string) []string {
    var results []string
    var wg sync.WaitGroup
    for _, u := range urls {
        wg.Add(1)
        go func() {
            defer wg.Done()
            results = append(results, fetch(u))
        }()
    }
    wg.Wait()
    return results
}
```

**练习 4**：用 `runtime.NumGoroutine` 和 channel 实现一个"等待所有 goroutine 完成"的 helper（不能用 WaitGroup）。

**练习 5**：解释为什么 `runtime.LockOSThread` 后必须 `UnlockOSThread`。如果忘了会怎样？

---

## 参考答案

**练习 1**：可能什么都不输出。main goroutine 不等子 goroutine 就退出了。修正：用 sync.WaitGroup 或 `time.Sleep`（仅 demo）。

**练习 2**：
```go
func worker(jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    defer func() { if r := recover(); r != nil { /* log */ } }()
    for j := range jobs {
        results <- j.Process()
    }
}

func Run(jobs []Job, n int) []Result {
    jobCh := make(chan Job, len(jobs))
    resCh := make(chan Result, len(jobs))
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go worker(jobCh, resCh, &wg)
    }
    for _, j := range jobs { jobCh <- j }
    close(jobCh)
    wg.Wait()
    close(resCh)
    var out []Result
    for r := range resCh { out = append(out, r) }
    return out
}
```

**练习 3**：① `u` 循环变量捕获（Go 1.21 前）；② `results` 多 goroutine 并发 append data race；③ 闭包没传 `u` 进去。修正：
```go
results := make([]string, len(urls))
for i, u := range urls {
    wg.Add(1)
    go func(i int, u string) {
        defer wg.Done()
        results[i] = fetch(u)   // 各 goroutine 写不同下标
    }(i, u)
}
```

**练习 4**：思路：每个 goroutine 启动前 +1，结束 -1（用 atomic int64 + sync.Cond 或 channel）。最优解通常仍是 WaitGroup；强行不用会更脆弱。

**练习 5**：LockOSThread 让 goroutine 绑死一个 M。如果不 Unlock，goroutine 退出时该 M 会被销毁（不能复用），增加线程创建开销。长期累积可能耗尽 M。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Goroutine | 用户态线程；初始 2KB 栈；启动 ~1µs |
| G/M/P | G=任务 / M=OS 线程 / P=逻辑 CPU |
| 调度 | 本地队列 + work stealing + 全局队列 |
| 抢占 | Go 1.14+ 异步抢占（SIGURG）；不再被紧循环卡死 |
| Syscall | 阻塞时 M-P 解绑；网络 I/O 走 netpoller 不阻塞 M |
| 栈 | 2KB 起 → 1GB；增长时复制；空闲时收缩 |
| GOMAXPROCS | 默认 = CPU 核；容器要用 Go 1.25+（cgroup-aware）或 automaxprocs |
| 泄漏 | pprof goroutine 是排查金标准 |

下一篇 **G12 — 精通 Go Channels 与 select** 会拆开 channel 的 hchan 结构、send/recv 状态机、buffered vs unbuffered、select 随机选择算法、close 的细节。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G11-精通-Goroutines-与-GMP-调度.md`
