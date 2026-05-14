# 精通 Go Channels 与 select

> 课程编号：G12
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Channels 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：channel 谜题三连

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 1)
    ch <- 1
    close(ch)
    v, ok := <-ch
    fmt.Println(v, ok)   // ① 输出？
    v, ok = <-ch
    fmt.Println(v, ok)   // ②

    var nilCh chan int
    select {              // ③
    case <-nilCh:
        fmt.Println("nil channel")
    default:
        fmt.Println("default")
    }
}
```

Channel 是 Go 并发的"语义层"基石。但它远不止"管道"那么简单——hchan 结构、send/recv 状态机、nil channel、关闭语义、select 随机化，每一项都有"看似常识但实际反直觉"的细节。本章把它们拆开看。

---

## 第一章：hchan 结构

### 1.1 内部字段

简化版（`runtime/chan.go`）：

```go
type hchan struct {
    qcount   uint           // 当前队列元素数
    dataqsiz uint           // buffer 容量
    buf      unsafe.Pointer // 环形 buffer 指针
    elemsize uint16         // 元素大小
    closed   uint32
    sendx    uint           // buffer 中下次 send 的位置
    recvx    uint           // buffer 中下次 recv 的位置
    recvq    waitq          // 等待 recv 的 G 队列
    sendq    waitq          // 等待 send 的 G 队列
    lock     mutex
}
```

每个 channel 都有一个**互斥锁** + **环形 buffer**（有缓冲时）+ **两个等待队列**（recvq 和 sendq）。

### 1.2 buffer 是环形数组

```
buf:  [ 1, 2, 3, _, _ ]      (容量 5，qcount=3)
        ↑           ↑
       recvx       sendx
```

`sendx` 和 `recvx` 是写入/读出位置，超过容量回绕到 0。

---

## 第二章：发送与接收的状态机

### 2.1 send (`ch <- v`)

伪代码（简化）：

```
lock(ch.lock)
if ch.closed:
    panic("send on closed channel")
if ch.recvq not empty:
    g = dequeue(ch.recvq)
    直接把 v 拷到 g 的接收槽位，唤醒 g
    unlock; return
if ch.qcount < ch.dataqsiz:
    buf[sendx] = v
    sendx = (sendx+1) % dataqsiz
    qcount++
    unlock; return
// 当前 goroutine 阻塞
enqueue(ch.sendq, current_g, v)
unlock; gopark()
```

### 2.2 recv (`v, ok := <-ch`)

```
lock(ch.lock)
if ch.sendq not empty AND buffer 空(unbuffered case):
    g = dequeue(ch.sendq)
    v = g 的待发值
    唤醒 g
    return v, true
if ch.qcount > 0:
    v = buf[recvx]
    recvx = (recvx+1) % dataqsiz
    qcount--
    // 若 sendq 有等待者，把它的值放进 buf 并唤醒
    return v, true
if ch.closed:
    return zero, false
enqueue(ch.recvq, current_g)
gopark()
```

### 2.3 关键性质

- 永远是**复制**：send 把发送方的 v 复制到 buf 或 receiver
- recv 时如果有发送方在等，**接收方直接从发送方拷贝**（绕过 buffer）
- close 后**再 send 会 panic**；再 recv 会拿到零值 + ok=false

---

## 第三章：无缓冲 vs 有缓冲

### 3.1 unbuffered

```go
ch := make(chan int)   // cap 0
```

**同步**：send 必须等到有 recv（或反之）。可以视为"会合"（rendezvous）。

```go
go func() { ch <- 1 }()    // 阻塞，直到 main 准备 recv
v := <-ch                  // 触发 send 完成
```

### 3.2 buffered

```go
ch := make(chan int, 3)
ch <- 1; ch <- 2; ch <- 3   // 不阻塞
ch <- 4                      // 阻塞，buffer 满
```

**异步**：buffer 未满时 send 不阻塞，未空时 recv 不阻塞。

### 3.3 选哪个

| 用途 | 建议 |
|---|---|
| 同步信号（等待完成） | unbuffered |
| 多生产者批量传递 | buffered |
| 限制并发数（semaphore） | buffered |
| 简单的 done 通知 | `chan struct{}` unbuffered |

**经验**：默认 unbuffered。需要"缓冲削峰"才用 buffered，并谨慎选 capacity（太小没用，太大掩盖问题）。

### 3.4 容量调优

`cap` 决定 buffer 大小，但不是性能旋钮——它改变的是**生产消费速率不匹配时的容忍度**：

- cap=0：每个 send 严格等一个 recv（背压最强）
- cap=N：允许 N 个 send 提前完成
- cap=∞（不可能）：所有 send 立即返回，背压消失

如果 cap 设大让程序"工作"，通常是消费速率不够——更多 worker、或限速。

---

## 第四章：close 的语义

### 4.1 close 后的行为

```go
close(ch)
ch <- 1     // panic: send on closed channel
v, ok := <-ch
// 如果还有未消费的数据，返回数据 + true
// 否则返回零值 + false
```

`for v := range ch` 会在 close 且 buffer 排空后自动退出循环。

### 4.2 close 的责任

**永远由发送方 close**。如果有多个发送方：

```go
var wg sync.WaitGroup
for _, x := range items {
    wg.Add(1)
    go func(x int) {
        defer wg.Done()
        ch <- x
    }(x)
}
go func() {
    wg.Wait()
    close(ch)   // 所有发送方完成后关
}()
```

接收方关闭可能让其他发送方 panic。

### 4.3 重复 close panic

```go
close(ch)
close(ch)   // panic: close of closed channel
```

防御：用 `sync.Once`：

```go
var once sync.Once
once.Do(func() { close(ch) })
```

### 4.4 broadcast 信号

```go
done := make(chan struct{})

// 多个 goroutine 接收
go func() { <-done; /* cleanup */ }()
go func() { <-done; /* cleanup */ }()

close(done)   // 所有等待者同时唤醒
```

close 是**广播**——所有阻塞在 recv 的 goroutine 都被唤醒。这是 done channel pattern 的核心。

---

## 第五章：select

### 5.1 多路复用

```go
select {
case v := <-ch1:
    fmt.Println("ch1:", v)
case ch2 <- 42:
    fmt.Println("sent to ch2")
case <-time.After(time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("no op")
}
```

- 多个 case 就绪：**随机**选一个（避免某个 case 始终饥饿）
- 没 case 就绪 + 有 default：执行 default
- 没 case 就绪 + 没 default：阻塞，直到某个就绪

### 5.2 随机化的重要性

```go
for {
    select {
    case v := <-fast:
        process(v)
    case v := <-slow:
        process(v)
    }
}
```

若按顺序选，`fast` 持续供应时 `slow` 永远饥饿。随机化保证公平。

### 5.3 select 的实现

简化：

1. lock 所有涉及的 channel（按地址排序避免死锁）
2. 检查每个 case 是否能立即执行
3. 若有就绪 case：随机选一个，unlock，执行
4. 若没有：把当前 G 注册到每个 channel 的 wait queue，unlock，gopark
5. 任一 channel 就绪唤醒 G，重新走 1（其他 case 的等待状态被清除）

### 5.4 nil channel 在 select 中的妙用

对 **nil channel** 的 send/recv **永远阻塞**——但在 select 中表现为"这个 case 永远不会被选"。

```go
var ch1, ch2 chan int = ..., nil   // 一开始 ch2 是 nil

for {
    select {
    case v := <-ch1: handle1(v)
    case v := <-ch2: handle2(v)   // 等同于不存在
    }
}
```

动态启用某个 case：把 nil 替换成真实 channel。这种"动态屏蔽 case"的技巧在状态机中很常用。

### 5.5 default 实现非阻塞

```go
select {
case v := <-ch:
    handle(v)
default:
    // 没有数据，不阻塞
}
```

实现"试探性接收"。

---

## 第六章：常见模式

### 6.1 done channel

```go
done := make(chan struct{})

go func() {
    for {
        select {
        case <-done: return
        case work := <-jobs: process(work)
        }
    }
}()

// 停止
close(done)
```

### 6.2 generator (range)

```go
func gen(n int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for i := 0; i < n; i++ { out <- i }
    }()
    return out
}

for v := range gen(10) { fmt.Println(v) }
```

返回**只读 channel**：`<-chan int`。

### 6.3 fan-out

```go
func split(in <-chan int, n int) []<-chan int {
    outs := make([]chan int, n)
    for i := range outs { outs[i] = make(chan int) }
    go func() {
        defer func() {
            for _, o := range outs { close(o) }
        }()
        i := 0
        for v := range in {
            outs[i] <- v
            i = (i + 1) % n
        }
    }()
    return castToReadOnly(outs)
}
```

### 6.4 fan-in

```go
func merge(ins ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, in := range ins {
        wg.Add(1)
        go func(in <-chan int) {
            defer wg.Done()
            for v := range in { out <- v }
        }(in)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

### 6.5 超时

```go
select {
case v := <-ch: handle(v)
case <-time.After(time.Second):
    return errors.New("timeout")
}
```

**注意**：`time.After` 每次创建新 timer，频繁调用会有 GC 压力。长期等待用 `time.NewTimer` + `defer t.Stop()`。

### 6.6 限流（semaphore）

```go
sem := make(chan struct{}, 10)   // 同时最多 10 个

for _, job := range jobs {
    sem <- struct{}{}   // 占一个
    go func(j Job) {
        defer func() { <-sem }()
        process(j)
    }(job)
}
```

---

## 第七章：方向性 channel 类型

### 7.1 三种类型

```go
chan int      // 双向
<-chan int    // 只读
chan<- int    // 只写
```

### 7.2 用法

```go
func produce(out chan<- int) { out <- 1; close(out) }
func consume(in <-chan int)  { for v := range in { use(v) } }

ch := make(chan int)
go produce(ch)
go consume(ch)
```

把方向限制写入签名，**让编译器帮你检查**："这个函数不应该 close 它，那个不应该读它"。

### 7.3 自动转换

双向 → 单向是自动的（隐式转换）；反过来不行。

```go
ch := make(chan int)
var r <-chan int = ch   // OK
var s chan<- int = ch   // OK
var b chan int = r      // ❌ 编译错误
```

---

## 第八章：channel 性能数据

| 操作 | 大致开销 |
|---|---|
| unbuffered send/recv（已配对） | ~50ns |
| unbuffered send/recv（需 park） | ~200ns + 调度延迟 |
| buffered send（buf 未满） | ~30ns |
| close + 唤醒 N 等待者 | ~N × 100ns |
| select 2 case 已就绪 | ~50ns |
| select N case 全 park | ~N × 50ns 注册 + 唤醒开销 |

不是真"快"——比 mutex 的 ~20ns 慢。但 channel 是**语义工具**，不是性能优化。如果纯粹是共享状态保护，直接 mutex 更快。

---

## 第九章：生产级最佳实践

1. **发送方 close**：从不接收方关。
2. **多发送方用 WaitGroup 协调 close**：所有完成后再 close。
3. **default 慎用**：除非真要非阻塞，不然引入忙等。
4. **`time.After` 用完即弃 OK，长 loop 用 Timer + Stop**。
5. **chan struct{} 表事件**：零负载、语义清晰。
6. **签名用单向 channel**：减少误用。
7. **不要让 channel 替代锁**：保护共享数据用 mutex 更高效。
8. **nil channel 是 select 的开关**：动态屏蔽 case。
9. **使用 `for range ch` 而非反复 `<-ch`**：自动检测 close。
10. **`go.uber.org/goleak` 测试 channel 泄漏**：常见的 worker 没退出。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：向已关闭 channel 发送
```go
close(ch)
ch <- 1   // panic
```

### ❌ 陷阱 2：重复 close
```go
close(ch); close(ch)   // panic
```

### ❌ 陷阱 3：忘记 close 导致 range 死锁
```go
out := gen()
for v := range out { ... }   // 生产者没 close → 这里永远不退出
```

### ❌ 陷阱 4：接收方 close
```go
v := <-ch
close(ch)   // 如果有别的 goroutine 在 send，下次 send panic
```

### ❌ 陷阱 5：buffered channel 大就以为安全
buffer 满了照样阻塞；缓冲只是延迟问题暴露。

### ❌ 陷阱 6：select 没 default + 没就绪 = 永远阻塞
忘了写 timeout case → goroutine 泄漏。

### ❌ 陷阱 7：time.After 在长循环里
```go
for {
    select {
    case <-ch: ...
    case <-time.After(time.Second):   // 每次循环新 timer
    }
}
```
改成 `t := time.NewTimer(time.Second); defer t.Stop()` + `t.Reset()`。

---

## 第十一章：练习题

**练习 1**：回到引言：
```go
ch := make(chan int, 1)
ch <- 1
close(ch)
v, ok := <-ch       // ① ?
v, ok = <-ch        // ② ?

var nilCh chan int
select {            // ③ ?
case <-nilCh: ...
default: ...
}
```

**练习 2**：写 `Mux[T any](ins ...<-chan T) <-chan T` 合并多个 channel。

**练习 3**：以下程序是否泄漏？为什么？
```go
func work(timeout time.Duration) {
    ch := make(chan struct{})
    go func() {
        heavyWork()
        ch <- struct{}{}
    }()
    select {
    case <-ch:
    case <-time.After(timeout):
    }
}
```

**练习 4**：实现一个限速器：最多每秒 N 个请求通过。

**练习 5**：解释 `select{}`（空 select）的语义和典型用途。

---

## 参考答案

**练习 1**：
- ① `1 true`（buffer 中还有 1）
- ② `0 false`（buffer 空 + closed）
- ③ `"default"`（nil channel 的 recv 永远阻塞，但 default 让 select 立即返回）

**练习 2**：
```go
func Mux[T any](ins ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    for _, in := range ins {
        wg.Add(1)
        go func(in <-chan T) {
            defer wg.Done()
            for v := range in { out <- v }
        }(in)
    }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

**练习 3**：泄漏。timeout 触发后函数返回，但 goroutine 内的 `ch <- struct{}{}` 阻塞（unbuffered + 接收方走了）→ 永久泄漏。修正：`make(chan struct{}, 1)`，让 send 不阻塞。

**练习 4**：
```go
type Limiter struct{ ticker *time.Ticker }
func New(n int) *Limiter {
    return &Limiter{ticker: time.NewTicker(time.Second / time.Duration(n))}
}
func (l *Limiter) Wait() { <-l.ticker.C }
```

**练习 5**：`select{}` 没有 case，永远阻塞。典型用途：main 函数最后等其他 goroutine 运行（如 server 启动后），或在测试 deadlock 探测。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| hchan | mutex + 环形 buf + recvq + sendq |
| unbuffered | 同步会合 |
| buffered | 异步缓冲；满阻塞、空阻塞 |
| close | 发送方负责；广播给所有 recv；重复 close panic |
| select | 多路复用；随机选 ready 的 case |
| nil channel | 在 select 中永久阻塞 = 屏蔽该 case |
| 方向性 | `<-chan T` 只读，`chan<- T` 只写；签名表达意图 |
| 性能 | 比 mutex 慢；用语义不用性能 |

下一篇 **G13 — 精通 sync 包：Mutex、RWMutex、WaitGroup、Once、Pool** 会讲底层 futex、Mutex 的状态机、RWMutex 的 starvation、Pool 在 GC 时的清理策略。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G12-精通-Go-Channels-与-select.md`
