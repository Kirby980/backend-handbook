# 精通 Go pprof 性能剖析

> 课程编号：G22
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Profiling 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：直觉是不可靠的

```
"我觉得这个 JSON 序列化慢"     → profile 显示是 GC 占 40%
"日志拖累了 QPS"               → profile 显示是某个 regex compile
"数据库查询是瓶颈"             → profile 显示是 Goroutine 在 mutex 等待
```

把猜测交给 pprof——它是 Go 内置的"性能 X 光"。本章拆开五种 profile（CPU、heap、goroutine、block、mutex），讲清采集、查看、解读、典型坑。

---

## 第一章：pprof 概览

### 1.1 五种 profile

| 类型 | 测量 |
|---|---|
| **CPU** | 哪些函数占 CPU 时间 |
| **Heap** | 哪些代码分配/持有内存 |
| **Goroutine** | 当前 goroutine 数量、各自的状态/堆栈 |
| **Block** | 哪些代码阻塞在 channel/sync |
| **Mutex** | 哪些 mutex 争用严重 |

### 1.2 两种采集方式

**A. 一次性采集**（CLI / benchmark）：

```bash
go test -cpuprofile=cpu.out -bench=.
go test -memprofile=mem.out -bench=.
```

**B. 持续服务**（HTTP endpoint）：

```go
import _ "net/http/pprof"

go http.ListenAndServe("localhost:6060", nil)
```

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### 1.3 工具

```bash
go tool pprof <file_or_url>
```

进入交互模式：

```
(pprof) top              # 前 10
(pprof) top20            # 前 20
(pprof) list FuncName    # 按函数查看带行号代码
(pprof) web              # 浏览器 SVG
(pprof) tree             # 树形
(pprof) png > out.png    # 导出 PNG
```

或浏览器：

```bash
go tool pprof -http=:8080 cpu.out
```

---

## 第二章：CPU profile

### 2.1 原理

按一定频率（默认 100Hz）采样当前所有 goroutine 的栈。每个栈帧"出现一次" = 一次采样。累计采样数 ~ CPU 占用比。

### 2.2 采集

```go
import "runtime/pprof"

f, _ := os.Create("cpu.out")
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
// 你的工作负载
```

或在 HTTP server：

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### 2.3 查看

```
(pprof) top
Showing nodes accounting for 4500ms, 90% of 5000ms total
      flat  flat%   sum%        cum   cum%
    1200ms 24.00% 24.00%     1200ms 24.00%  runtime.mapaccess1_faststr
     800ms 16.00% 40.00%     2400ms 48.00%  myapp/handler.parse
     ...
```

- `flat`：函数本体消耗
- `cum`：本体 + 调用的所有子函数

### 2.4 `list` 命令

```
(pprof) list parse
ROUTINE ======================== myapp/handler.parse
   800ms      2400ms (flat, cum) 48.00% of Total
         .         .   12:func parse(s string) Result {
     200ms     200ms   13:    parts := strings.Split(s, ",")
         .     400ms   14:    n := strconv.Atoi(parts[0])
     ...
```

按行级看哪行最慢——通常一眼能定位优化点。

### 2.5 火焰图

```bash
go tool pprof -http=:8080 cpu.out
# 浏览器 → VIEW → Flame Graph
```

最直观——横向是采样累积时间，纵向是调用栈。宽的"平台"就是热点。

---

## 第三章：Heap profile

### 3.1 含义

记录**采样的堆分配**，默认每 512KB 分配采样一次（`runtime.MemProfileRate`）。

### 3.2 两种 metric

- `inuse_space`：**当前**还活着的对象内存
- `alloc_space`：从程序启动累计**分配过**的内存

```bash
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
go tool pprof -inuse_objects http://localhost:6060/debug/pprof/heap
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap
```

### 3.3 区别使用

- 怀疑**内存泄漏** → inuse_space
- 怀疑**GC 压力**（频繁分配） → alloc_space / alloc_objects

### 3.4 采集精度

```go
runtime.MemProfileRate = 1   // 每次分配都采样（开销大，仅 debug 用）
```

默认 512KB 已经够发现大头分配源。

### 3.5 典型解读

```
flat  flat%       cum   cum%
 1GB    50%      1GB    50%   bytes.Buffer.grow
500MB   25%   500MB    25%   json.encodeState.string
```

看到 bytes.Buffer.grow 50% → 优化方向：sync.Pool 复用 buffer。

---

## 第四章：Goroutine profile

### 4.1 用途

**找 goroutine 泄漏**——按调用栈分组列出当前所有 goroutine。

### 4.2 采集

```bash
curl http://localhost:6060/debug/pprof/goroutine?debug=2 > goroutine.txt
```

`debug=2` 给完整可读的 stack（含 goroutine id、状态、阻塞原因）。

### 4.3 典型解读

```
1024 @ 0x40123e 0x4012ef ...
#       0x40123e   runtime.gopark+0xee
#       0x4012ef   runtime.chanrecv+0x...
#       0x...      myapp.worker+0x4f
```

`1024` 个 goroutine 阻塞在同一 channel recv。

如果业务上不应该有这么多 worker 在等 → 泄漏。

### 4.4 单次快照不够

泄漏需要观察**变化趋势**：
- 启动后 1 分钟拍一次
- 10 分钟后再拍一次
- 对比，新增的 goroutine 是泄漏候选

或写测试用 `runtime.NumGoroutine()` 监控。

---

## 第五章：Block profile

### 5.1 用途

记录 goroutine 在**同步操作上阻塞的时间**——channel、mutex、wait group、select。

### 5.2 默认关闭

```go
runtime.SetBlockProfileRate(1)   // 采样所有阻塞 >= 1ns 的事件（开销大）
runtime.SetBlockProfileRate(1_000_000)   // 每 1ms 采一次
```

### 5.3 采集

```bash
go tool pprof http://localhost:6060/debug/pprof/block
```

### 5.4 典型解读

发现某 channel 阻塞累计 30 秒——说明生产/消费速率不匹配；可能要加缓冲或加 worker。

### 5.5 不要默认开

开启后每次阻塞事件都有少量开销。生产用要 sample（`rate=100_000_000` = 100ms 一次）。

---

## 第六章：Mutex profile

### 6.1 用途

哪些 mutex 争用最严重——`Lock` 等待时间长的栈。

### 6.2 启用

```go
runtime.SetMutexProfileFraction(100)   // 采样 1/100
```

`0` 关闭；`N` 每 N 次锁竞争采一次。

### 6.3 采集

```bash
go tool pprof http://localhost:6060/debug/pprof/mutex
```

### 6.4 典型场景

```
flat  flat%   cum   cum%
   2s    50%    2s    50%  myapp/cache.(*Cache).Get
```

`cache.Get` 累计等锁 2s → 也许换成 RWMutex / sync.Map / 分片。

---

## 第七章：trace（runtime/trace）

虽然不是 pprof 而是另一个工具，常一起用：

### 7.1 启用

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()
// work
```

或 HTTP：

```bash
go tool trace http://localhost:6060/debug/pprof/trace?seconds=5
```

### 7.2 查看

```bash
go tool trace trace.out
# 自动打开浏览器
```

显示：
- 每个 goroutine 的时间线（运行 / 阻塞 / 等待）
- syscall、GC 事件
- channel 事件
- 网络阻塞

**适合**：分析延迟尾部、调度时间、看清 goroutine 互动。

详见 G23 专题。

---

## 第七章半：FlightRecorder —— 只在出问题时拿轨迹（Go 1.25+）

### 7.4 旧痛：always-on tracing 太贵

`runtime/trace` 拿到的轨迹信息丰富，但持续开启会**显著增加 CPU 与内存开销**。生产里只能"出事再开"——但 99% 的"出事"是过去时，等开了什么都没了。

### 7.5 FlightRecorder 的做法

**Go 1.25 在 `runtime/trace` 加了 FlightRecorder**：常驻一个**有限大小的环形缓冲区**，持续写入最近一段时间的 trace 事件。出问题时立即 `WriteTo` 把缓冲区落盘——拿到的就是出事**前**的现场。

```go
import (
    "os"
    "runtime/trace"
)

fr := trace.NewFlightRecorder(trace.FlightRecorderConfig{
    MinAge:   5 * time.Second,
    MaxBytes: 10 << 20,    // 10 MiB 环形 buffer
})
fr.Start()
defer fr.Stop()

// ……长跑业务……
// 当指标系统监测到 p99 飙升时：
if p99 > threshold {
    f, _ := os.Create("flight.trace")
    fr.WriteTo(f)         // 只把最近 5-30 秒的 trace 写出来
    f.Close()
}
```

之后用 `go tool trace flight.trace` 看出事前的调度时间线、syscall、GC、network poll 等等。**和"在线监测"+ "事后取证"的 SRE 流程契合度极高**。

### 7.6 Go 1.26：pprof Web UI 默认火焰图

`go tool pprof -http=:8080` 在 **Go 1.26** 起 web UI 默认展示**火焰图**而不是老的"Top + Graph" 视图。Graph 仍可在 *View → Graph* 切回。这与上面 2.5 节火焰图的推荐一致。

> 参考：[Go 1.25 release notes — runtime/trace](https://go.dev/doc/go1.25#runtimetrace)、[Go 1.26 release notes — pprof](https://go.dev/doc/go1.26#pprof)。

---

## 第八章：实战工作流

### 8.1 发现问题

```
- CPU 使用率高 → CPU profile
- 内存增长 → Heap profile (inuse_space)
- GC 频繁 → Heap profile (alloc_space)
- 服务慢 → CPU + block profile
- goroutine 数飙升 → goroutine profile
- 锁争用怀疑 → mutex profile
- 复杂时序 → runtime/trace
```

### 8.2 采集时长

- CPU profile：30 秒典型（够大数定律平均）
- Heap：瞬间快照
- Goroutine：瞬间快照
- Trace：5-10 秒（文件大）

### 8.3 对比 profile

```bash
go tool pprof -base=baseline.out new.out
```

显示差异。优化前后对比时关键。

### 8.4 生产环境注意

- pprof endpoint **不要暴露公网**：会泄漏 stack trace
- 绑定 localhost：`http.ListenAndServe("127.0.0.1:6060", nil)`
- 或加身份认证

```go
mux := http.NewServeMux()
mux.HandleFunc("/debug/pprof/", basicAuth(pprof.Index))
// ...
```

---

## 第九章：常见瓶颈与解法

### 9.1 GC 占 30%

heap profile → 找最大 alloc 源 → 用 sync.Pool 或减少分配。

### 9.2 JSON 序列化慢

CPU profile → reflect.Type 大量出现 → 用 easyjson / sonic / sonnet 生成代码 marshal。

### 9.3 大量字符串分配

alloc_objects → strings.Builder Grow 预分配；避免 fmt.Sprintf 在热路径。

### 9.4 锁争用

mutex profile → 分片或 RWMutex；read-mostly 用 atomic.Value。

### 9.5 channel 阻塞

block profile → 加 buffer、加 worker、改 select non-blocking。

### 9.6 syscall 过多

CPU profile → 看 runtime.syscall 比例 → 批量化、cache、改 mmap 等。

### 9.7 反射开销

reflect.Value 调用多 → 提前用 reflect 取出 method 缓存。

---

## 第十章：生产级最佳实践

1. **永远引入 `_ "net/http/pprof"`**：零成本（除非 endpoint 被命中）。
2. **绑定 localhost 或加认证**：防泄漏。
3. **CI 跑 benchmark + cpuprofile**：定期对比。
4. **生产周期采 30s CPU profile**：归档供事后分析。
5. **goroutine endpoint 监控**：突增 = 报警。
6. **block / mutex profile 谨慎开**：高 rate 有 overhead。
7. **优化前后用 benchstat + pprof 对比**：避免幻觉。
8. **火焰图分享给团队**：可视化沟通。
9. **不要怀疑直觉，profile 后才决定**。
10. **结合 trace 看尾延迟**：单纯 profile 看不到调度。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：profile 时长太短
3 秒 CPU profile 噪声大。30 秒起步。

### ❌ 陷阱 2：开发机调优、生产无效
开发机内存、CPU、负载与生产不同。profile 要在**接近生产**的环境跑。

### ❌ 陷阱 3：optimize 测试代码
profile 看到 testing 内部 → 测试基础设施，不是被测代码。filter 掉。

### ❌ 陷阱 4：忽略 cum vs flat
flat 高 = 本函数慢；cum 高 = 函数+子函数慢。区分清楚。

### ❌ 陷阱 5：alloc_space 等于 inuse_space
alloc_space 是累计分配（包括已 GC 的）；inuse_space 是当前活着的。两个看不同问题。

### ❌ 陷阱 6：mutex profile 不准
sample rate 太高会 missing；太低看不到。试验调整。

### ❌ 陷阱 7：pprof endpoint 暴露生产
攻击者可看堆栈、命令名、heap 内容（可能含敏感数据）。

---

## 第十二章：练习题

**练习 1**：以下程序如何用 pprof 找到瓶颈？给出步骤。
```go
func handler(w http.ResponseWriter, r *http.Request) {
    data := []byte{}
    for i := 0; i < 100000; i++ {
        data = append(data, fmt.Sprintf("%d", i)...)
    }
    w.Write(data)
}
```

**练习 2**：解释 `flat=0 cum=80%` 的函数意味着什么。

**练习 3**：写出查看一个程序 heap inuse_objects 前 20 的命令。

**练习 4**：goroutine profile 显示 5000 个 goroutine 都阻塞在 `cache.Get`，分析可能原因。

**练习 5**：trace UI 中看到大量 "GC mark assist" 时间，意味着什么？怎么改进？

---

## 参考答案

**练习 1**：
1. 添加 `_ "net/http/pprof"` 引入 + 启动 6060 server。
2. 持续打 handler： `wrk -t 4 -c 50 -d 30s http://localhost:8080/`
3. 同时 `go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30`
4. `top` 会看到 `fmt.Sprintf` 和 `growslice` 占大头
5. 修：strings.Builder + Grow + strconv.Itoa

**练习 2**：本函数自己不耗 CPU，但**调用的子函数**累计耗 80%。它是"调用栈中间节点"。看 cum 链向下到底是哪个子函数 flat 高，那才是真热点。

**练习 3**：
```bash
go tool pprof -inuse_objects http://localhost:6060/debug/pprof/heap
(pprof) top20
```

**练习 4**：
- cache 实现锁太粗（单 mutex 整个 map）
- 一个慢调用堆积请求
- LRU 驱逐时持锁太久
排查：mutex profile + 看 `Cache.Get` 实现。

**练习 5**：mark assist 是 GC 给 app goroutine 派的"标记任务"——堆增长太快、GC 跟不上。改进：减少分配、调高 GOGC、增加 GOMEMLIMIT 控制峰值。

---

## 小结

| profile | 用途 |
|---|---|
| CPU | 哪些函数占 CPU |
| Heap (inuse) | 当前哪些代码持有内存 |
| Heap (alloc) | 哪些代码分配压力大 |
| Goroutine | g 数量与阻塞栈；查泄漏 |
| Block | channel/sync 阻塞 |
| Mutex | 锁争用 |
| Trace | 时序、调度、GC、网络 |

工具：

- `go tool pprof <file_or_url>`
- `go tool pprof -http=:8080`
- `top` `list` `web` `tree`
- `-base=` 做对比

下一篇 **G23 — 精通 runtime/trace 与 go tool trace** 会深入 trace UI：goroutine 时间线、syscall、GC、网络阻塞、调度延迟。

---

