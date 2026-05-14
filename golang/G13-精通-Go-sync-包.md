# 精通 sync 包：Mutex、RWMutex、WaitGroup、Once、Pool

> 课程编号：G13
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — sync 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：先抓个 bug

```go
package main

import (
    "fmt"
    "sync"
)

type Counter struct {
    sync.Mutex
    n int
}

func (c Counter) Inc() {   // ← 注意：值接收者
    c.Lock(); defer c.Unlock()
    c.n++
}

func main() {
    c := Counter{}
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); c.Inc() }()
    }
    wg.Wait()
    fmt.Println(c.n)
}
```

这段代码有两个 bug：值接收者复制了 Mutex（go vet copylocks 警告），并且修改的是副本的 n。每个调用都"看似加锁"，但不影响外部 c。结果：c.n 永远是 0。这一章把 sync 包的每个组件拆开讲清楚——状态机、性能特征、坑。

---

## 第一章：sync.Mutex

### 1.1 基础

```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// 临界区
```

互斥锁——同一时间只有一个 goroutine 能持有。其他调用 `Lock()` 的会阻塞。

### 1.2 内部状态机

简化版（`runtime/sync_mutex.go`）：

```go
type Mutex struct {
    state int32   // 含 locked / woken / starving 位 + waiter count
    sema  uint32  // 用于 park/wakeup 的 semaphore
}
```

`state` 一个 int32 编码多种信息：

- bit 0：是否已锁
- bit 1：是否有"清醒中"的等待者（woken）
- bit 2：是否进入饥饿模式（starving）
- bit 3+：等待者数量（waiter count）

`sema` 用 runtime 的 semaphore 把阻塞 goroutine park 起来。

### 1.3 自旋（spin）

Go Mutex 有自旋优化：等待者会先在用户态忙等几次（用 `runtime.procyield`），如果在自旋期间锁释放就立刻拿到，避免 park/wakeup 开销。

自旋条件：
- 多核 CPU
- GOMAXPROCS > 1
- 当前 P 上还有别的 G 可跑

### 1.4 饥饿模式

Go 1.9 引入：如果一个等待者超过 1ms 还没拿到锁，进入饥饿模式。饥饿模式下：

- 解锁时直接把锁交给队首等待者（不再让新到的"插队"）
- 这避免了"长尾"延迟（个别请求等几秒）

### 1.5 Lock 不可重入

```go
mu.Lock()
mu.Lock()   // ❌ 死锁
```

Go 没有递归锁。要在已加锁的代码里再加锁，要么：

- 把临界区拆开
- 用单独的非锁版本函数（`fooLocked()`）

### 1.6 TryLock（Go 1.18+）

```go
if mu.TryLock() {
    defer mu.Unlock()
    // ...
} else {
    // 锁忙，跳过
}
```

非阻塞获取，失败立即返回。**很少用**（Go 设计者建议优先用 channel 或重新设计）。

### 1.7 性能数据

| 操作 | 无竞争 | 有竞争 |
|---|---|---|
| Lock + Unlock | ~20ns | ~100-500ns（park） |
| 自旋成功命中 | ~30ns | - |

---

## 第二章：sync.RWMutex

### 2.1 用途

允许**多读单写**：

```go
var mu sync.RWMutex
mu.RLock(); defer mu.RUnlock()    // 读锁
mu.Lock();  defer mu.Unlock()     // 写锁
```

多个 RLock 可以并发持有；Lock 与任何其他 Lock/RLock 互斥。

### 2.2 何时用 RWMutex

读 >> 写（如 90:10）且临界区有显著工作（> ~100ns）时，RWMutex 快于 Mutex。但 RWMutex 本身比 Mutex 重——开销约 2-3 倍。短临界区下 Mutex 反而快。

实测临界区 < 50ns 时 Mutex 几乎总是赢。

### 2.3 写者饥饿与设计

Go RWMutex 实现保证**写者优先**：一旦有 Lock() 排队，新来的 RLock() 也要等。这避免持续读流量饿死写者。

### 2.4 不可重入

```go
mu.RLock()
mu.RLock()   // 第二个调用合法（多读允许）
mu.RUnlock(); mu.RUnlock()
```

但**不能升级**：

```go
mu.RLock()
mu.Lock()   // ❌ 死锁
```

需要"先读后判断要不要写"的模式：

```go
mu.RLock()
if needsWrite {
    mu.RUnlock()
    mu.Lock()
    defer mu.Unlock()
    // 重新检查条件（其他人可能已经写过）
    if needsWrite { ... }
} else {
    defer mu.RUnlock()
}
```

---

## 第三章：sync.WaitGroup

### 3.1 基础

```go
var wg sync.WaitGroup
for i := 0; i < N; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); work() }()
}
wg.Wait()
```

WaitGroup = 计数器：Add(n) 增加 n；Done() 等价 Add(-1)；Wait() 等到 0。

### 3.2 正确用法

- `Add` **必须**在 goroutine 启动之前调用（否则 Wait 可能在 Add 之前看到 0 而早退）
- `Done` 用 defer 包裹防遗漏
- WaitGroup 不可值复制（go vet copylocks）

```go
// ❌ 错误：Add 在 goroutine 内
go func() {
    wg.Add(1)   // race: Wait 可能已经返回
    defer wg.Done()
    work()
}()
```

### 3.3 复用 WaitGroup

```go
wg.Wait()
// 现在 wg 计数为 0，可以再次 Add 复用
wg.Add(3)
// ...
```

复用合法但要小心：不能 `Wait()` 仍在执行的同时新 Add。

### 3.4 Go 1.25+ 的 WaitGroup.Go

```go
var wg sync.WaitGroup
for _, item := range items {
    wg.Go(func() { process(item) })   // 内部自动 Add(1) + defer Done
}
wg.Wait()
```

`WaitGroup.Go(f)` 是 Go 1.25 新增的便捷方法，等价于 `wg.Add(1); go func() { defer wg.Done(); f() }()`。三个好处：

1. **不会忘记 Add/Done 配对**——少一类 bug
2. **不会忘记 `defer wg.Done()`**——goroutine panic 时也能正确 Done
3. **配合 Go 1.22+ 的循环变量新语义**，连 `item := item` 都不需要写了

参考：[Go 1.25 release notes — sync 包](https://go.dev/doc/go1.25#sync)。

---

## 第四章：sync.Once

### 4.1 单次执行

```go
var (
    once sync.Once
    val  *Config
)

func GetConfig() *Config {
    once.Do(func() {
        val = loadConfig()
    })
    return val
}
```

`Do` 内的函数保证在所有调用中**只执行一次**——即使有 100 个 goroutine 同时调用 `GetConfig`。

### 4.2 实现

```go
type Once struct {
    done atomic.Uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if o.done.Load() == 0 {
        o.doSlow(f)
    }
}
func (o *Once) doSlow(f func()) {
    o.m.Lock(); defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1)
        f()
    }
}
```

经典 "double-checked locking"：先 atomic 看一眼，没初始化才加锁再检查（避免重复）。

### 4.3 panic 与 Once

如果 `f` panic，Once **仍然标记为 done**——下次 `Do` 不会重试。这通常是好的（崩溃的初始化不该掩盖），但记得 panic 内做好日志/recover。

### 4.4 Go 1.21+ OnceFunc/OnceValue/OnceValues

```go
loadConfig := sync.OnceValue(func() *Config { return load() })
cfg := loadConfig()   // 第一次：调用 load；之后：直接返回缓存
```

更优雅的"懒加载值"模式。

---

## 第五章：sync.Pool

### 5.1 用途

复用对象，减少 GC 压力。典型场景：临时 buffer、JSON encoder/decoder、bytes.Buffer。

```go
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(s string) string {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()
    buf.WriteString(s)
    // ...
    return buf.String()
}
```

### 5.2 关键特性

- **每个 P 独立的 pool**（无锁快路径）+ 全局共享的备份
- **GC 时清空**——pool 中的对象在每次 GC 都被回收
- `New` 函数在 Get 找不到对象时调用

### 5.3 为什么 GC 清空

Pool 设计目标是"减少分配压力"，不是"长期持有"。GC 已经被触发说明内存压力存在，此时清空 Pool 释放内存是合理的。

### 5.4 何时用 / 不用

**用**：
- 对象**初始化成本高**（如 bytes.Buffer 内部已分配大 slice）
- 对象**生命周期短**（一个请求内用完丢回）
- 高并发场景

**不用**：
- 简单值类型（int、struct）—— 直接栈分配更快
- 长生命周期对象 —— pool 周期性清空，没意义
- 对象大小**差异大** —— 拿到的可能是 1KB buffer 或 1MB buffer，难以预测

### 5.5 重置对象状态

`Put` 之前必须重置对象内部状态（如 `buf.Reset()`），否则下次 Get 拿到的是脏数据。

### 5.6 性能数据

```
BenchmarkPool-8        300_000_000      ~20 ns/op     0 allocs/op
BenchmarkNoPool-8       30_000_000      ~300 ns/op    1 alloc/op
```

热点路径 10 倍提升常见。

---

## 第六章：sync.Map

### 6.1 适用场景

```go
var m sync.Map
m.Store("k", 1)
v, ok := m.Load("k")
m.LoadOrStore("k", 2)
m.Delete("k")
m.Range(func(k, v any) bool { return true })
```

**仅适合**：
- 写一次、读多次（key 集合稳定）
- 多 goroutine 操作**不相交的 key 子集**

否则用 `map + RWMutex` 通常更快。

### 6.2 内部结构

```go
type Map struct {
    mu     Mutex
    read   atomic.Pointer[readOnly]   // 无锁路径
    dirty  map[any]*entry              // 加锁路径
    misses int
}
```

`read` 是 atomic 指针——读操作无锁原子读。`dirty` 包含 read 之外的新增 key。
miss 太多时 dirty 升级为 read。

### 6.3 缺点

- 接口 `any` 用：每次 Load 类型断言
- 不支持 len（只能 Range 数）
- Range 不是快照（迭代中其他 goroutine 修改可见）

---

## 第七章：sync.Cond

### 7.1 用途

"等待某个条件成立"。比 channel 更轻量，但**更难用对**。

```go
var (
    mu   sync.Mutex
    cond = sync.NewCond(&mu)
    queue []int
)

// consumer
mu.Lock()
for len(queue) == 0 {
    cond.Wait()   // 释放 mu，park；被 Signal 唤醒后重新拿 mu
}
v := queue[0]; queue = queue[1:]
mu.Unlock()

// producer
mu.Lock()
queue = append(queue, x)
cond.Signal()   // 唤醒一个等待者
mu.Unlock()
```

### 7.2 用 channel 替代

```go
ch := make(chan int)
// producer: ch <- x
// consumer: v := <-ch
```

绝大多数 Cond 场景可用 channel 改写。Go 团队建议优先 channel。

---

## 第八章：sync.Atomic（一并讲）

### 8.1 基础原子

```go
import "sync/atomic"

var n int64
atomic.AddInt64(&n, 1)
v := atomic.LoadInt64(&n)
atomic.StoreInt64(&n, 100)
atomic.CompareAndSwapInt64(&n, 100, 200)
```

### 8.2 Go 1.19+ 的类型化 API

```go
var n atomic.Int64
n.Add(1)
v := n.Load()
n.Store(100)
n.CompareAndSwap(100, 200)
```

类型安全，避免传错地址。优先用类型版。

### 8.3 atomic.Value

```go
var cfg atomic.Value
cfg.Store(loadConfig())

// 读
c := cfg.Load().(*Config)
```

无锁广播配置更新。注意**类型必须一致**（第一次 Store 决定类型）。

### 8.4 何时用 atomic vs mutex

- 单个数值的读写 → atomic
- 多字段一起更新或临界区 → mutex
- 频繁读、罕见写的"配置热更新" → atomic.Value

---

## 第九章：生产级最佳实践

1. **永远 `defer mu.Unlock()`**：异常路径不漏锁。
2. **接收者用指针**：值接收者复制 mutex。
3. **不要在 struct 中嵌入 Mutex 并导出**：调用方可能误用。
4. **RWMutex 只在读 >> 写 + 临界区显著时用**。
5. **WaitGroup.Add 在 goroutine 外**：race-free。
6. **sync.Once 用于全局懒初始化**：避免双层判断 + atomic 实现。
7. **sync.Pool 一定 Reset**：防数据污染。
8. **sync.Map 谨慎用**：先 benchmark map+RWMutex。
9. **atomic 加 .Value 是无锁广播配置的标准模式**。
10. **`go vet -copylocks`** + `-race` 是开发必备。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：值接收者复制 Mutex
```go
func (c Counter) Inc() { c.Lock(); ... }   // 副本上加锁
```

### ❌ 陷阱 2：Mutex 复制
```go
m1 := sync.Mutex{}
m2 := m1   // 两把独立锁，原意是共享
```

### ❌ 陷阱 3：忘记 Unlock
```go
mu.Lock()
if err != nil { return err }   // 没 Unlock → 死锁
mu.Unlock()
```
永远 `defer mu.Unlock()`。

### ❌ 陷阱 4：Pool 不 Reset
```go
buf := pool.Get().(*bytes.Buffer)
buf.WriteString(s)   // 之前的内容残留
```

### ❌ 陷阱 5：Pool 存大对象
```go
pool.Put(huge)   // GC 后清空也释放不及时
```
大对象不适合 Pool。

### ❌ 陷阱 6：WaitGroup 错位
```go
go func() { wg.Add(1); ... }()   // race
```

### ❌ 陷阱 7：RWMutex 升级
```go
mu.RLock()
mu.Lock()   // 死锁
```

---

## 第十一章：练习题

**练习 1**：以下代码有何问题？
```go
type Cache struct {
    mu sync.Mutex
    data map[string]int
}
func (c Cache) Get(k string) int {
    c.mu.Lock(); defer c.mu.Unlock()
    return c.data[k]
}
```

**练习 2**：实现 `MemoizedGet`：高频读、低频写、要求读路径无锁。

**练习 3**：以下两段哪个更快？为什么？
```go
// A
var mu sync.Mutex
var n int
go increment(&mu, &n, 1000000)
// B
var n int64
go atomicIncrement(&n, 1000000)
```

**练习 4**：写一个 `Group`，类似 `errgroup`：并发执行多个函数，等所有完成，任一错则取消其他。

**练习 5**：解释为什么 sync.Pool 不能用来做"对象缓存"（如 LRU）。

---

## 参考答案

**练习 1**：值接收者复制了 `sync.Mutex`，每次调用锁的是副本。修正：`func (c *Cache) Get`。

**练习 2**：用 `atomic.Value`：
```go
type Cache struct{ v atomic.Value }
func (c *Cache) Get() *Snapshot { return c.v.Load().(*Snapshot) }
func (c *Cache) Set(s *Snapshot) { c.v.Store(s) }
```
读路径完全无锁。写时整体替换。

**练习 3**：B 快。原子操作 ~5-10ns，无锁；Mutex 单线程 ~20ns，争用更高。但若临界区有更复杂操作，Mutex 必须用。

**练习 4**：参见 `golang.org/x/sync/errgroup`：
```go
type Group struct {
    ctx    context.Context
    cancel context.CancelFunc
    wg     sync.WaitGroup
    err    atomic.Pointer[error]
}
func (g *Group) Go(fn func() error) {
    g.wg.Add(1)
    go func() {
        defer g.wg.Done()
        if err := fn(); err != nil {
            g.err.CompareAndSwap(nil, &err)
            g.cancel()
        }
    }()
}
func (g *Group) Wait() error {
    g.wg.Wait()
    if p := g.err.Load(); p != nil { return *p }
    return nil
}
```

**练习 5**：sync.Pool **每次 GC 都被清空**——没法持有任何"有意义"的缓存条目。LRU 需要确定性容量和生命周期，与 Pool 设计相悖。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Mutex | 自旋 + 饥饿模式；不可重入 |
| RWMutex | 多读单写；写者优先；短临界区不如 Mutex |
| WaitGroup | Add 必须在 goroutine 外；1.20+ Go() |
| Once | double-checked locking；panic 也算 done |
| Pool | per-P 缓存；GC 清空；Reset 后再 Put |
| Map | 读多写少 + key 稳定；不然用 RWMutex |
| Cond | 大多场景应用 channel 替代 |
| atomic | 数值操作首选；1.19+ 类型 API；Value 做广播 |

下一篇 **G14 — 精通 context 包** 会讲清 context 树形结构、Cancel/Deadline/Value 三大语义、传播规则、不应放进 context 的东西。

---

