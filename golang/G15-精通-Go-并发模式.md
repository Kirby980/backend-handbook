# 精通 Go 并发模式：fan-in、fan-out、pipeline 与 worker pool

> 课程编号：G15
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Concurrency Patterns 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：把前面四章串成 cookbook

G11（goroutine）、G12（channel）、G13（sync）、G14（context）合起来是 Go 并发的"原子积木"。但真实工程里你不会重新发明轮子——你会用 fan-out / fan-in / pipeline / pub-sub / semaphore 等**经过反复验证的模式**。本章把这些模式以"问题 → 模式 → 代码 → 何时不用"的格式整理成手册。

---

## 第一章：管道（Pipeline）

### 1.1 问题

数据流：`输入 → 阶段1 → 阶段2 → 阶段3 → 输出`，每个阶段独立运行。

### 1.2 实现

```go
func gen(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case <-ctx.Done(): return
            case out <- n:
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case <-ctx.Done(): return
            case out <- n * n:
            }
        }
    }()
    return out
}

func print(ctx context.Context, in <-chan int) {
    for v := range in {
        fmt.Println(v)
    }
}

// 组合
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
print(ctx, square(ctx, gen(ctx, 1, 2, 3, 4)))
```

### 1.3 要点

- 每个阶段是一个 goroutine
- 阶段之间用 channel 连接
- 上游 close → range 自动终止 → 下游也 close
- 每个阶段 `select <-ctx.Done()` 支持提前取消

### 1.4 何时用

数据流式处理：日志聚合、ETL、流处理。每个阶段独立扩展、并发执行。

### 1.5 限制

- 顺序保留（同一阶段单 goroutine 时）
- 中间阶段慢会让 buffer 累积或阻塞上游
- 调试栈深

---

## 第二章：Fan-out（一对多）

### 2.1 问题

一个数据源 → 多个 worker 并发处理。

### 2.2 实现

```go
func fanOut[T any](ctx context.Context, in <-chan T, n int, work func(T)) {
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done(): return
                case v, ok := <-in:
                    if !ok { return }
                    work(v)
                }
            }
        }()
    }
    wg.Wait()
}
```

### 2.3 何时用

- CPU 密集任务：N 个 worker，N = GOMAXPROCS
- I/O 密集任务：N 可以远大于 CPU 数（数百甚至数千）
- 处理顺序无关紧要

### 2.4 何时不用

需要保持顺序（fan-out 后乱序）；任务有依赖。

---

## 第三章：Fan-in（多对一）

### 3.1 问题

多个数据源 → 合并到一个 channel。

### 3.2 实现

```go
func fanIn[T any](ctx context.Context, ins ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    wg.Add(len(ins))
    for _, in := range ins {
        go func(in <-chan T) {
            defer wg.Done()
            for v := range in {
                select {
                case <-ctx.Done(): return
                case out <- v:
                }
            }
        }(in)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

### 3.3 常配合 fan-out

```
gen → fan-out N workers → fan-in → 下游
```

这是处理"独立任务列表"的标准结构：先分发到 N 个 worker 并发处理，再汇总输出。

---

## 第四章：Worker Pool

### 4.1 问题

固定 N 个 worker，处理无界任务队列。

### 4.2 实现

```go
type Job struct{ /* ... */ }
type Result struct{ /* ... */ }

func runPool(ctx context.Context, jobs <-chan Job, n int) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup
    for i := 0; i < n; i++ {
        wg.Add(1)
        go worker(ctx, jobs, out, &wg)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}

func worker(ctx context.Context, jobs <-chan Job, out chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker recovered: %v\n%s", r, debug.Stack())
        }
    }()
    for j := range jobs {
        select {
        case <-ctx.Done(): return
        case out <- process(j):
        }
    }
}
```

### 4.3 关键点

- worker 数固定（按资源决定）
- jobs channel 由生产方 close
- 每个 worker 顶部 defer recover
- ctx 取消时所有 worker 退出
- WaitGroup 等所有 worker 完成后 close out

### 4.4 buffer 大小

`jobs := make(chan Job, ...)` 的缓冲：
- 太小：生产者频繁等待
- 太大：内存占用、积压问题被掩盖
- 经验：`N × 2` 到 `N × 10` 之间起步，根据生产/消费速率调

---

## 第五章：Semaphore（限并发）

### 5.1 问题

不想用 worker pool 那么"重"，只是想限制同时进行的操作数。

### 5.2 用 buffered channel

```go
sem := make(chan struct{}, 10)   // 最多 10 并发

for _, item := range items {
    sem <- struct{}{}
    go func(it Item) {
        defer func() { <-sem }()
        process(it)
    }(item)
}
```

`sem` 满了 send 阻塞 → 新 goroutine 不启动。简洁、零依赖。

### 5.3 用 golang.org/x/sync/semaphore

```go
sem := semaphore.NewWeighted(10)

for _, item := range items {
    if err := sem.Acquire(ctx, 1); err != nil { return err }
    go func(it Item) {
        defer sem.Release(1)
        process(it)
    }(item)
}
```

支持加权（一个任务可以占多个槽位），且 `Acquire` 接受 context。

### 5.4 vs Worker Pool

- semaphore：goroutine 在外部循环里 spawn；适合任务列表已知、各任务大致独立
- worker pool：固定 N 个 worker 长跑；适合无界任务流、需要保留 worker 内部状态

---

## 第六章：Generator（生成器）

### 6.1 问题

按需产生序列：自然数、文件行、数据库分页。

### 6.2 实现

```go
func naturals(ctx context.Context) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 1; ; i++ {
            select {
            case <-ctx.Done(): return
            case out <- i:
            }
        }
    }()
    return out
}

ctx, cancel := context.WithCancel(context.Background())
defer cancel()
for v := range naturals(ctx) {
    fmt.Println(v)
    if v >= 10 { cancel(); break }
}
```

### 6.3 关键

- 必须支持 ctx 取消（否则提前 break 会泄漏 goroutine）
- 永远 buffered=0 或小 buffer（让消费方驱动节奏）

---

## 第七章：Broadcast（一发多收）

### 7.1 问题

一个事件通知 N 个监听者。

### 7.2 用 close(channel) 广播

```go
done := make(chan struct{})

for i := 0; i < N; i++ {
    go func() { <-done; cleanup() }()
}

close(done)   // 同时唤醒所有 N 个
```

最简单、最高效——所有阻塞在 `<-done` 的 goroutine 立即唤醒。但仅支持"一次性事件"。

### 7.3 持续广播：sync.Cond 或 pub/sub 库

如果要"反复广播同一种事件"：

```go
type Broadcaster struct {
    mu        sync.Mutex
    subs      []chan struct{}
}
func (b *Broadcaster) Sub() <-chan struct{} {
    b.mu.Lock(); defer b.mu.Unlock()
    ch := make(chan struct{}, 1)
    b.subs = append(b.subs, ch)
    return ch
}
func (b *Broadcaster) Pub() {
    b.mu.Lock(); defer b.mu.Unlock()
    for _, ch := range b.subs {
        select {
        case ch <- struct{}{}:
        default:   // 订阅者慢，丢一次
        }
    }
}
```

---

## 第八章：Tee（分流）

### 8.1 问题

一个输入 → 两个独立消费者。

### 8.2 实现

```go
func tee[T any](ctx context.Context, in <-chan T) (<-chan T, <-chan T) {
    out1, out2 := make(chan T), make(chan T)
    go func() {
        defer close(out1)
        defer close(out2)
        for v := range in {
            select {
            case <-ctx.Done(): return
            case out1 <- v:
                select {
                case <-ctx.Done(): return
                case out2 <- v:
                }
            }
        }
    }()
    return out1, out2
}
```

注意：必须**先 send 到 out1 再 send 到 out2**（或反过来）。慢的消费者会拖慢另一个——这是正常背压。

---

## 第九章：Or-channel（任一完成）

### 9.1 问题

多个事件源，任意一个触发即返回。

### 9.2 递归实现

```go
func or(channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0: return nil
    case 1: return channels[0]
    }
    orDone := make(chan struct{})
    go func() {
        defer close(orDone)
        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            case <-or(append(channels[3:], orDone)...):
            }
        }
    }()
    return orDone
}
```

任一 channel 关闭 → orDone 关闭。

### 9.3 用途

```go
<-or(timeout, ctx.Done(), userAborted, downstreamErr)
```

把"多个取消源"统一为单个 done channel。

---

## 第十章：Errgroup

### 10.1 问题

并发执行多个函数，任一出错则取消其他，最终返回第一个错。

### 10.2 用 `golang.org/x/sync/errgroup`

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)
for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetch(ctx, url)
    })
}
if err := g.Wait(); err != nil { /* ... */ }
```

- `g.Go` 启动 goroutine 并自动 WaitGroup
- 任一 fn 返回非 nil err → ctx cancel
- `g.Wait()` 等所有完成，返回第一个非 nil error

### 10.3 SetLimit（Go 1.21+）

```go
g.SetLimit(10)   // 同时最多 10 个 goroutine
```

errgroup 内置 semaphore，免去手动限流。

---

## 第十一章：Heartbeat（心跳）

### 11.1 问题

长任务想报告"我还活着"，让外部监控判断是否卡住。

### 11.2 实现

```go
func work(ctx context.Context) (<-chan struct{}, <-chan Result) {
    heartbeat := make(chan struct{}, 1)
    results := make(chan Result)
    go func() {
        defer close(heartbeat)
        defer close(results)
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done(): return
            case <-ticker.C:
                select { case heartbeat <- struct{}{}: default: }
            case results <- doStep():
            }
        }
    }()
    return heartbeat, results
}
```

监控方：

```go
hb, res := work(ctx)
for {
    select {
    case <-ctx.Done(): return
    case <-time.After(5 * time.Second):
        log.Println("worker stalled")
        cancel()
    case <-hb: /* alive */
    case r := <-res: handle(r)
    }
}
```

---

## 第十二章：Future / Promise

### 12.1 实现

```go
type Future[T any] struct {
    done chan struct{}
    val  T
    err  error
}

func Run[T any](fn func() (T, error)) *Future[T] {
    f := &Future[T]{done: make(chan struct{})}
    go func() {
        defer close(f.done)
        f.val, f.err = fn()
    }()
    return f
}

func (f *Future[T]) Wait() (T, error) {
    <-f.done
    return f.val, f.err
}
```

### 12.2 用法

```go
fut := Run(func() (User, error) { return fetchUser(ctx, id) })
// 做其他事...
user, err := fut.Wait()
```

---

## 第十三章：Rate Limiter

### 13.1 token bucket

```go
import "golang.org/x/time/rate"

lim := rate.NewLimiter(10, 50)   // 每秒 10 个，burst 50
for req := range reqs {
    if err := lim.Wait(ctx); err != nil { return err }
    handle(req)
}
```

### 13.2 自实现 ticker 版

```go
ticker := time.NewTicker(time.Second / 10)
defer ticker.Stop()
for req := range reqs {
    <-ticker.C
    handle(req)
}
```

简单但不支持 burst、不支持 ctx。

---

## 第十三章半：iter.Seq + range over func —— Go 1.23 重塑 pipeline 写法

### 13.5 新语法：函数即可迭代

**Go 1.23 (2024-08)** 把 `for ... := range f`（其中 `f` 是 push-iterator 函数）正式纳入语言，并新增 `iter` 包。pipeline 不再非得用 channel：

```go
import "iter"

// 一个返回 Seq[T] 的 stage，本身就是个 push iterator
func Squares(n int) iter.Seq[int] {
    return func(yield func(int) bool) {
        for i := 1; i <= n; i++ {
            if !yield(i * i) { return }
        }
    }
}

// 消费端：直接 range
for v := range Squares(10) {
    fmt.Println(v)
}
```

`iter.Seq[V]` 等价于 `func(yield func(V) bool)`；`iter.Seq2[K,V]` 是双值版本（map 风格）。`yield` 返回 `false` 表示消费者不要了——iterator 函数必须立刻 return（否则会 panic）。

### 13.6 与 channel pipeline 的对比

| 维度 | channel pipeline | iter.Seq pipeline |
|---|---|---|
| 调度 | 跨 goroutine（M:N 调度成本） | 同步、同 goroutine |
| 资源泄漏 | 必须正确 close + ctx 取消，不然 goroutine 泄漏 | break 自动通知 yield 结束，无 goroutine |
| 背压 | 由 channel buffer 控制 | 天然——消费端不调用 `next` 上游就停 |
| 适用 | 真正需要并行（CPU 密集 / 阻塞 I/O） | 流式纯计算、组合多个 stage |

**结论**：纯 CPU/纯转换的 pipeline 用 `iter.Seq` 写更轻、更不易漏 goroutine；需要并行执行的 stage 仍然用 channel + worker pool。两者完全可以混用。

### 13.7 Pull 模式 —— 把 push 转成手动驱动

```go
next, stop := iter.Pull(Squares(10))
defer stop()                  // 一定要 stop，否则上游 goroutine 泄漏
for {
    v, ok := next()
    if !ok { break }
    handle(v)
}
```

`iter.Pull` 在内部启了一个 goroutine 把 push 转 pull，**所以 `stop()` 必须 defer**。

### 13.8 标准库迭代器化

`slices`、`maps` 包（Go 1.23）新增了大量迭代器返回值——`slices.All`/`slices.Values`/`slices.Backward`/`slices.Chunk`、`maps.Keys`/`maps.Values`/`maps.All`/`maps.Insert`/`maps.Collect`。组合后非常顺手：

```go
top3 := slices.Collect(iter.Take(slices.Sorted(maps.Values(scores)), 3))
```

> 参考：[Range Over Function Types（Go 官方博客）](https://go.dev/blog/range-functions)、[Go 1.23 release notes — iter](https://go.dev/doc/go1.23#iter)。

---

## 第十四章：生产级最佳实践

1. **每个长 goroutine 加 recover**：worker 不应让进程崩。
2. **每个长 goroutine 接 ctx**：能优雅退出。
3. **生产者 close channel**：never reader-close。
4. **Pipeline 每个阶段隔离**：每段一个 goroutine，可独立替换。
5. **Worker pool 数量随资源调**：CPU 任务 = NumCPU；I/O 任务可远超。
6. **背压用 unbuffered 或小 buffer**：让快者等慢者。
7. **errgroup 替代手动 WaitGroup + channel**：减少模板代码。
8. **测试用 goleak**：避免合并隐藏的泄漏。
9. **超时是必备**：永远给 channel 操作配 timeout 兜底。
10. **复杂模式画图**：在 PR 描述里画数据流，方便 review。

---

## 第十五章：常见陷阱清单

### ❌ 陷阱 1：fan-out 后顺序丢失
分发到多个 worker → 完成顺序≠输入顺序。如需保持顺序，每个任务带 index 排序后输出。

### ❌ 陷阱 2：pipeline 中途 close
中间 stage close 上游会让上游 send panic。close 责任永远在发送端。

### ❌ 陷阱 3：worker 没限并发耗尽资源
`for { go work() }` 无限创建 goroutine。

### ❌ 陷阱 4：semaphore 漏 release
忘了 `defer sem.Release()` → 永久占用槽位。

### ❌ 陷阱 5：tee 阻塞
一个消费者卡住 → 另一个永远收不到。需要监控+timeout。

### ❌ 陷阱 6：心跳频率比任务还高
逻辑反而被心跳干扰；心跳 = 任务周期的 2-10 倍。

### ❌ 陷阱 7：errgroup goroutine 内不 select ctx
fn 不响应 ctx → cancel 没用，得等 fn 自己跑完。fn 内部要 `<-ctx.Done()` 兜底。

---

## 第十六章：练习题

**练习 1**：用 pipeline 实现"从文件读取 URL → 抓取 → 计算 SHA256 → 写到结果文件"，每段一个 goroutine。

**练习 2**：以下代码哪里漏了 ctx？补全。
```go
func gen(out chan<- int) {
    for i := 0; ; i++ { out <- i }
}
```

**练习 3**：实现 `LimitedParallel(ctx, items, work, n int)`：对 items 并发执行 work，最多 n 个同时。

**练习 4**：使用 errgroup 实现"并发请求 3 个 API，任一成功立即返回结果"（取消另外两个）。

**练习 5**：分析下面的 pipeline 是否泄漏 goroutine。
```go
out := gen(ctx)
for v := range out {
    if v > 100 { break }
    fmt.Println(v)
}
```

---

## 参考答案

**练习 1**：
```go
urls := readURLs(ctx, "file")    // chan string
docs := fetch(ctx, urls)          // chan []byte
hashes := sha256s(ctx, docs)      // chan string
write(ctx, hashes, "out")         // 消费者
```
每个函数返回 chan，内部 goroutine 关闭 chan。

**练习 2**：
```go
func gen(ctx context.Context, out chan<- int) {
    for i := 0; ; i++ {
        select {
        case <-ctx.Done(): return
        case out <- i:
        }
    }
}
```

**练习 3**：
```go
func LimitedParallel[T any](ctx context.Context, items []T, work func(context.Context, T) error, n int) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(n)
    for _, it := range items {
        it := it
        g.Go(func() error { return work(ctx, it) })
    }
    return g.Wait()
}
```

**练习 4**：
```go
g, ctx := errgroup.WithContext(ctx)
result := make(chan T, 1)
for _, api := range apis {
    api := api
    g.Go(func() error {
        v, err := api.Call(ctx)
        if err != nil { return err }
        select {
        case result <- v:
        case <-ctx.Done():
        }
        return errors.New("done")   // 返回非 nil 触发 cancel
    })
}
_ = g.Wait()
return <-result
```

**练习 5**：泄漏。`break` 后没人读 out 了，gen goroutine 阻塞在 `out <- i`。修正：使用 `ctx, cancel := context.WithCancel(...)`，break 前 `cancel()`，gen 内部 select Done。

---

## 小结

| 模式 | 用途 |
|---|---|
| Pipeline | 顺序阶段化处理 |
| Fan-out | 并发处理独立任务 |
| Fan-in | 合并多源 |
| Worker Pool | 固定 worker + 任务队列 |
| Semaphore | 限并发 |
| Generator | 按需序列 |
| Broadcast | close 一次唤醒全部 |
| Tee | 一进二出 |
| Or-channel | 多取消源汇成一个 |
| Errgroup | 失败联动取消 |
| Heartbeat | 心跳监控 |
| Future | 异步结果 |
| Rate Limiter | 频率控制 |

下一篇 **G16 — 精通 Race Detection 与数据竞争调试** 会讲清 `-race` 是怎么工作的（Thread Sanitizer）、什么样的代码会触发警告、Happens-before 关系。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G15-精通-Go-并发模式.md`
