# 精通 Race Detection 与数据竞争调试

> 课程编号：G16
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Race Detection 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：40 分钟

---

## 引言：测试通过的代码也可能有 race

```go
var count int

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); count++ }()
    }
    wg.Wait()
    fmt.Println(count)
}
```

跑一次输出 998。再跑输出 1000。再跑 992。**没有"错误信号"——程序"工作"了**。这就是 data race 的可怕之处：它不一定 panic，不一定崩溃，但你的状态从此被腐蚀。Go 内置 `-race` 标志，能在测试时把这类问题揪出来。

---

## 第一章：什么是 data race

### 1.1 定义

Go memory model 规定：**两个或更多 goroutine 访问同一内存位置，至少一个是写**，且**没有 happens-before 关系**——就是 data race。

四个要素必须全部满足才构成 race：

1. 多个 goroutine
2. 同一内存位置
3. 至少一个是写
4. 无同步保证顺序

### 1.2 不是 race 的例子

```go
// 只读 → 不是 race
var data []int = readOnce()
go fmt.Println(data)
go fmt.Println(data)
```

```go
// 有同步 → 不是 race
var mu sync.Mutex
var x int
go func() { mu.Lock(); x = 1; mu.Unlock() }()
go func() { mu.Lock(); _ = x; mu.Unlock() }()
```

```go
// channel 同步 → 不是 race
ch := make(chan int)
go func() { x = 1; ch <- 1 }()
<-ch
fmt.Println(x)
```

---

## 第二章：-race 标志

### 2.1 用法

```bash
go run -race main.go
go build -race -o app
go test -race ./...
```

### 2.2 工作原理

底层基于 LLVM 的 **ThreadSanitizer (TSan)**。编译器对每个内存访问注入"shadow memory"记录，运行时跟踪：

- 每个 goroutine 的逻辑时钟
- 每个内存位置的最后访问者及其时钟

如果发现两次访问没有 happens-before 关系且至少一个是写——报 race。

### 2.3 开销

- CPU：5-10 倍慢
- 内存：5-10 倍
- 二进制：~2 倍大

**不要在生产部署 -race**。开发、CI 跑测试足够。

### 2.4 报告形式

```
WARNING: DATA RACE
Read at 0x00c000018088 by goroutine 7:
  main.main.func1()
      /path/main.go:10 +0x35
Previous write at 0x00c000018088 by goroutine 6:
  main.main.func1()
      /path/main.go:10 +0x49
Goroutine 7 (running) created at:
  main.main() /path/main.go:8 +0x73
Goroutine 6 (finished) created at:
  main.main() /path/main.go:8 +0x73
```

包含：访问类型、地址、调用栈、goroutine 来源。

---

## 第三章：Happens-before 关系

### 3.1 含义

A happens-before B 意味着 A 的所有内存写入对 B 可见。Go 程序中**只有以下原语建立 happens-before**：

- channel send → 对应的 channel receive
- channel close → 后续 receive 返回 zero
- mutex Unlock → 之后的 Lock
- WaitGroup Done → Wait 返回
- Once Do 返回 → 之后的 Do 调用
- atomic load/store 的明确顺序（从 Go 1.19 起规范化）

### 3.2 关键陷阱：仅靠 print 顺序判断同步是错的

```go
var x int
go func() { x = 1; fmt.Println("set") }()
fmt.Println("read", x)
```

即使你看到 `set` 后再 `read` —— **不代表** `x = 1` 对 read 可见。CPU 可能乱序，内存可能没同步。要建立 happens-before 必须用前述原语。

---

## 第四章：典型 race 模式

### 4.1 共享变量计数

```go
var n int
for i := 0; i < 100; i++ {
    go func() { n++ }()
}
```

修正：`atomic.Int64` 或 `mutex`。

### 4.2 map 并发读写

```go
m := map[int]int{}
go func() { m[1] = 1 }()
go func() { _ = m[1] }()
```

不仅是 race，runtime 还会 fatal error。修正：sync.Map 或 mutex。

### 4.3 slice 并发 append

```go
var s []int
for i := 0; i < 10; i++ {
    go func(i int) { s = append(s, i) }(i)
}
```

append 是"读 cap → 可能分配 → 写 len" 多步操作，并发会写丢、写覆盖。修正：mutex 或预分配 + 各写各下标。

### 4.4 写完即关闭，读方期待
```go
var ready bool
go func() {
    data = fetch()
    ready = true   // ❌ 没同步，读方看不到
}()
for !ready {}    // 可能永远循环
fmt.Println(data)
```
修正：channel close 表示完成。

### 4.5 循环变量 + goroutine（Go 1.21 及之前的历史问题）
```go
for i := 0; i < 5; i++ {
    go func() { fmt.Println(i) }()   // race 读 i
}
```
**Go 1.22 起循环变量语义已修复**（每次迭代新变量），所以这段在新代码里不再 race。但维护 `go.mod` 声明 < 1.22 的老仓库时仍要注意，模式上"goroutine 闭包捕获循环变量"也仍是常见 review 检查项。

### 4.6 复制结构体含 Mutex
```go
type S struct{ mu sync.Mutex }
s := S{}
s2 := s   // copylocks 警告：s2 是独立 mutex
```

### 4.7 测试用 -race 跑

```bash
go test -race ./...
```

CI 必备。

---

## 第五章：debug race 的方法论

### 5.1 复现

race 多半概率触发。增加触发概率：
- 跑更多次：`go test -race -count=100`
- 跑更高并发：临时改高 goroutine 数
- 限制 CPU：`GOMAXPROCS=1`（虽然降低，但有时反而能触发不同顺序的 race）

### 5.2 看报告找定位

报告的两个堆栈是 race 双方。优先看：
- 是不是同一变量？（地址相同）
- 两边是否经过同步原语？

### 5.3 添加缩小范围

如果报告堆栈跨越多个函数：用 `runtime.GOMAXPROCS(1)` 临时限制并行度。这能减少干扰，专注一对竞争。

### 5.4 用 sync 工具修

| 场景 | 工具 |
|---|---|
| 单变量计数 | atomic |
| 多字段更新 | mutex |
| 读多写少 | RWMutex / atomic.Value |
| 一次性初始化 | sync.Once |
| 等所有完成 | WaitGroup |
| pub/sub 事件 | channel |

---

## 第六章：原子操作与内存模型

### 6.1 atomic 提供 happens-before 吗

**Go 1.19 之前**：模糊。社区按"应该"理解，但规范没明说。
**Go 1.19+**：规范明确：atomic 操作之间有 sequential consistency。

```go
// Go 1.19+ 这段没有 race
var done atomic.Bool
go func() { data = compute(); done.Store(true) }()
for !done.Load() { runtime.Gosched() }
fmt.Println(data)
```

### 6.2 何时用 atomic 替代 mutex

- 单个数值的递增、设置、比较
- 高频读、零开销路径
- 实现 lock-free 数据结构

不能用 atomic 的：
- 多字段一起更新
- 复杂条件判断
- 不可分割逻辑

---

## 第六章半：goroutineleak profile —— 让 GC 帮你抓泄漏（Go 1.26+ 实验）

`-race` 抓数据竞争，不抓 goroutine 泄漏；`pprof` 的 `goroutine` profile 能列出所有活着的 goroutine，但要人工对比快照才发现"应该消失却没消失的"。

**Go 1.26 在 `runtime/pprof` 加了一个新 profile：`goroutineleak`**（实验，由 GOEXPERIMENT 控制；预计 1.27 默认开）。它的判定方式很优雅：

> 一个 goroutine 阻塞在某个并发原语上（chan、mutex、cond...），而该原语已**从所有可运行 goroutine 的可达图里失联**——那它就是泄漏的。

判定借助 GC 的可达性分析，不需要人写规则。访问方式：

```go
import _ "net/http/pprof"
// 然后浏览器访问：
// http://localhost:6060/debug/pprof/goroutineleak
```

返回的 profile 与普通 goroutine profile 同结构（含每个泄漏 goroutine 的栈），可直接 `go tool pprof` 看。**生产服务定期抓一次** + 报警，能持续清理累积泄漏。

> 参考：[Go 1.26 release notes — runtime/pprof](https://go.dev/doc/go1.26#runtimepprof)。

---

## 第七章：常见误以为是 race 的情况

### 7.1 不同 channel 收发

```go
ch1 := make(chan int)
ch2 := make(chan int)
var x int
go func() { x = 1; ch1 <- 1 }()
go func() { x = 2; ch2 <- 1 }()
```

这**是** race—两个 goroutine 写 x，没有同步关系。即使分别走不同 channel。

### 7.2 单 goroutine

单 goroutine 内永远没 race（只有一个执行流）。所以"main goroutine 启动后改值，goroutine 内读"——若启动前的 `go` 在 happens-before 那次写之前，则安全。

```go
x := 1
go func() { fmt.Println(x) }()   // ✅ go 语句之后写 x 才不安全
x = 2                              // ❌ race
```

`go` 语句**本身**建立 happens-before（goroutine 的开始）。

---

## 第八章：生产级最佳实践

1. **CI 永远跑 `-race`**：每次 PR 都过一遍。
2. **不要在生产用 -race**：开销巨大。
3. **go vet -copylocks**：发现 mutex/WaitGroup 复制。
4. **共享状态走 mutex / channel / atomic**：不要假设"仅读"。
5. **map 永远不要"无保护"并发访问**：fatal error 比 race 更糟。
6. **atomic 操作建立 happens-before（Go 1.19+）**。
7. **测试关键并发路径**：用 -race + 高并发反复跑。
8. **每次重构后再跑 -race**：refactor 容易引入 race。
9. **依赖第三方库时检查它们是否 thread-safe**：默认否。
10. **Race 报告必须修复**：忽视 = 定时炸弹。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：以为加 print 同步了
print 内部有锁，但**不为你的变量建立 happens-before**。

### ❌ 陷阱 2：以为 atomic.Bool flag 够了
单 flag OK，但 flag + 多字段共同更新 = 仍 race。

### ❌ 陷阱 3：misaligned atomic
32 位平台上 atomic.Int64 要求 8 字节对齐；放在 struct 中间可能不对齐 → panic。Go 1.19 类型化 API（atomic.Int64）自动对齐。

### ❌ 陷阱 4：在 close 之后还 send
race 之外，还会 panic。

### ❌ 陷阱 5：以为 sync.Mutex 防止"逻辑"问题
mutex 只保护数据访问，不保护"业务一致性"——竞态条件可能产生不一致状态。

### ❌ 陷阱 6：在 goroutine 内修改循环变量（Go 1.21 前）
经典坑。

### ❌ 陷阱 7：在 -race 下运行才有 race
Race 永远在；-race 只是检测。生产没 -race 不代表没 race。

---

## 第十章：练习题

**练习 1**：以下代码有几个 race？修复。
```go
var counts = map[string]int{}
func bump(k string) { counts[k]++ }
go bump("a")
go bump("b")
go bump("a")
```

**练习 2**：以下代码是否有 race？解释。
```go
var x int
done := make(chan struct{})
go func() { x = 1; close(done) }()
<-done
fmt.Println(x)
```

**练习 3**：编写一个 unit test 验证 `func F()` 在并发调用下不 race。

**练习 4**：以下代码为何 race，即使有 mutex？
```go
var mu sync.Mutex
var m = map[int]int{}
func get(k int) int { return m[k] }                  // 无锁读
func set(k, v int)  { mu.Lock(); m[k]=v; mu.Unlock() }
```

**练习 5**：解释为什么 `var b atomic.Bool` 比 `var b bool` 加 mutex 在某些场景更慢。

---

## 参考答案

**练习 1**：3 个 goroutine 并发写 map → 多对 race + fatal error。修正：
```go
var mu sync.Mutex
func bump(k string) {
    mu.Lock(); defer mu.Unlock()
    counts[k]++
}
```

**练习 2**：无 race。`close(done)` happens-before `<-done`，所以 `x = 1` 对 main 可见。

**练习 3**：
```go
func TestF_NoRace(t *testing.T) {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); F() }()
    }
    wg.Wait()
}
```
配合 `go test -race`。

**练习 4**：写有锁、读无锁——任何"读时另一 goroutine 在写"都是 race。修正：读也加 RLock。

**练习 5**：atomic.Bool 每次操作有 memory barrier 开销，~5-10ns。mutex 无竞争时 ~20ns 但临界区可包含多次访问；如果只 read 一次 bool，atomic 更快；如果要做"读 bool → 改 5 个字段 → 写 bool"，mutex 更便宜（一次进入临界区）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| race 定义 | 多 g + 同地址 + 至少一写 + 无 happens-before |
| -race | TSan 实现；CPU/内存 5-10x；不要上生产 |
| happens-before | 由 channel/mutex/wg/once/atomic 等建立 |
| 修复工具 | atomic（单值）/ mutex（多字段）/ channel（事件） |
| go vet -copylocks | 检查锁复制 |
| Go 1.19+ atomic | 规范确立顺序保证 |
| CI | 跑 -race 是标配 |

下一篇 **G17 — 精通 Go Modules** 会讲清 `go.mod` 文件、版本号语义、replace/exclude 指令、vendoring、private repo 配置、modules proxy。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G16-精通-Go-Race-Detection.md`
