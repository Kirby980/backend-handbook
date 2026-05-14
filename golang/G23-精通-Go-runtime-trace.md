# 精通 Go runtime/trace 与 go tool trace

> 课程编号：G23
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Tracing 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：pprof 看不到的"时间维度"

pprof 告诉你**哪些函数占 CPU**，但回答不了：

- 为什么这个请求耗了 200ms？只有 30ms 在 CPU 上做事，其余在等什么？
- GC 暂停发生在哪个时间点？影响了哪些 goroutine？
- goroutine 启动后到第一次跑过去了多久？

这些"**时序 + 调度**"问题要靠 `runtime/trace`。本章拆开它的采集、可视化、典型用例。

---

## 第一章：trace 的本质

### 1.1 与 pprof 的区别

| | pprof | trace |
|---|---|---|
| 数据形式 | 采样统计 | 事件流（每事件带时间戳） |
| 看什么 | 哪些函数热 | 程序在时间线上的行为 |
| 开销 | 低（采样） | 中-高（每个 syscall/调度都记） |
| 文件大小 | 小（< MB） | 大（几秒可能几十 MB） |
| 工具 | go tool pprof | go tool trace |

### 1.2 trace 记录的事件

- goroutine 创建、退出、阻塞、解除阻塞
- syscall 进入/退出
- GC 标记/扫描各阶段
- 网络轮询事件
- 用户自定义 region / task / log

### 1.3 时间精度

事件时间戳来自 CPU TSC，纳秒级精度。可以精确观察"goroutine 在 200µs 之内做了几件事"。

---

## 第二章：采集 trace

### 2.1 程序内嵌

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()

    // 你的工作
}
```

### 2.2 HTTP endpoint

```go
import _ "net/http/pprof"
go http.ListenAndServe("localhost:6060", nil)
```

```bash
curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"
```

### 2.3 测试 + benchmark

```bash
go test -trace=trace.out -bench=.
```

### 2.4 时长

通常 5-10 秒。文件大小约 1-10MB/s（视活动量）。

---

## 第三章：trace UI

### 3.1 启动

```bash
go tool trace trace.out
# 自动开浏览器，默认 localhost:port
```

### 3.2 主页

```
View trace
Goroutine analysis
Network blocking profile
Synchronization blocking profile
Syscall blocking profile
Scheduler latency profile
User-defined tasks
User-defined regions
Minimum mutator utilization
```

每个链接对应一种分析视角。

### 3.3 主时间线（View trace）

最强大的视图。看到：

- 时间轴（横向）
- 每个 P 的活动（纵向多行）
- 每个 goroutine 在每个时刻的状态（运行 / 阻塞 / 等待 / GC）
- syscall、GC 事件标记

**怎么读**：

- 黄色短条：单个 goroutine 在 P 上执行
- 绿色长条：GC mark
- 蓝色：网络 wait
- 灰色：调度间隙

放大某个时间段可以看到具体 goroutine ID + 函数。

---

## 第四章：典型 trace 模式

### 4.1 看尾延迟

某请求 P99 200ms 但平均 20ms：

1. 在 view trace 找到那个慢请求的时段
2. 看 goroutine 的状态分布
3. 发现：80% 时间在 channel wait
4. 找发送方为什么慢

### 4.2 看 GC 影响

```
GC mark assist
GC pause
```

在时间轴上 GC 事件清晰可见。如果某 GC 持续 50ms 且包含 STW 5ms → 找到对应的 P 看哪些 goroutine 被挂起。

### 4.3 看调度延迟

"Scheduler latency profile" 显示 goroutine 从 _Grunnable 状态到真正 _Grunning 的延迟分布。

延迟高（>1ms）意味着：
- GOMAXPROCS 太少
- 持续被抢占
- 频繁 syscall 解绑 P 后重新拿 P

### 4.4 看 syscall

"Syscall blocking profile" 列出哪些函数花最多时间在 syscall。

典型：read/write 系统调用比例过高 → 改用 buffered I/O、批量。

---

## 第五章：用户自定义事件

### 5.1 Region

```go
ctx, task := trace.NewTask(context.Background(), "processRequest")
defer task.End()

func process(ctx context.Context) {
    defer trace.StartRegion(ctx, "decode").End()
    decode(...)
    
    defer trace.StartRegion(ctx, "compute").End()
    compute(...)
}
```

trace UI 的 "User-defined regions" 显示每个 region 的延迟分布。把"业务阶段"暴露给 trace，比纯调用栈更易懂。

### 5.2 Task

```go
ctx, task := trace.NewTask(ctx, "fetchUser")
defer task.End()
```

跨 goroutine 的逻辑任务。task 的子事件（region、log）都关联到这个 task。

### 5.3 Log

```go
trace.Log(ctx, "category", "message")
```

打点。在 trace UI 中看到。

---

## 第六章：go tool trace 子视图

### 6.1 Goroutine analysis

按 goroutine 创建栈分组，显示每组的：
- 总时间
- 各状态时间分布（运行 / 网络 / 同步 / syscall / GC / runnable）

找哪类 goroutine 占资源最多。

### 6.2 Synchronization blocking profile

类似 pprof 的 block profile，但带时间信息。

### 6.3 Network blocking profile

哪些 goroutine 在网络 I/O 上阻塞多。

### 6.4 Minimum mutator utilization (MMU)

最复杂但最强大的视图：给定窗口大小（如 1ms），程序最低有多少时间在跑业务而非 GC。

理想：100%。实际：GC 期间会下降。这告诉你"GC 在最坏时段对 mutator 的影响"。

---

## 第七章：trace 的实战工作流

### 7.1 案例：慢请求

1. 复现：构造能稳定触发慢请求的负载
2. 采 5s trace 包含至少一次慢请求
3. 在 trace UI 找到慢请求的 goroutine（按时间）
4. 看该 goroutine 时间分布：
   - 80% running CPU → 算法慢，转 CPU profile
   - 80% network → 下游 API 慢
   - 80% syscall → I/O 模式问题
   - 80% blocked on sync → 锁争用
5. 针对性优化

### 7.2 案例：QPS 上不去

1. 跑压测 + 采 trace
2. View trace 看 P 利用率
3. 如果 P 经常空闲 → 看是否被 syscall 阻塞 / 调度未跟上
4. 如果 P 全满但 QPS 低 → 看 CPU profile 找热点

### 7.3 案例：GC 抖动

1. trace + GODEBUG=gctrace=1
2. 看 GC 频率、duration、stop-the-world
3. heap profile 找 alloc 源
4. 改善：sync.Pool / GOMEMLIMIT / 减少分配

---

## 第八章：trace 的开销

### 8.1 采集开销

- CPU：~10-30%（开 trace 时）
- 文件 I/O：写 trace 文件
- 内存：runtime buffer

**生产慎用**。建议：仅在排查时短时开启（5-10 秒）。

### 8.2 trace 与 pprof 同时开

可以同时，但相互影响。常见：先 pprof 找 hot function，再 trace 看具体时序。

---

## 第九章：生产级最佳实践

1. **trace 短期采集**：5-10 秒够看清问题。
2. **pprof 先，trace 后**：pprof 找方向，trace 看细节。
3. **用户 region / task 标记业务阶段**：让 trace 更易读。
4. **关注尾延迟用 trace**：平均/中位数用 pprof 够。
5. **看 MMU**：判断 GC 对延迟的真实影响。
6. **trace 文件大要压缩**：跨网络传输前 gzip。
7. **trace UI 在本地跑**：避免敏感数据上云。
8. **结合 CPU profile 一起看**：互补。
9. **优化前后双采**：trace 也用 -base 对比？trace 没有这功能但人工对比。
10. **trace 不要长期开**：开销大 + 文件巨大。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：trace 太长文件巨大
20 秒 trace 可能 1GB。先短时间试。

### ❌ 陷阱 2：在 trace UI 里 lost
不知道怎么看时间线。先看 Goroutine analysis 找重点 goroutine。

### ❌ 陷阱 3：把 trace 当 pprof
trace 不是 sample，是事件流。统计性指标用 pprof。

### ❌ 陷阱 4：忽视 user-defined region
没埋点 → trace UI 看不到业务语义。

### ❌ 陷阱 5：生产开 trace 不关
持续开 trace 让程序慢、文件爆。

### ❌ 陷阱 6：在开发机看不到生产现象
trace 显示的是采集时的状况；生产负载不同。

### ❌ 陷阱 7：试图看几百万 goroutine
trace UI 处理大 goroutine 数 lagging。先用 goroutine analysis 收敛。

---

## 第十一章：练习题

**练习 1**：写一段代码，使用 trace.NewTask + StartRegion 包裹 HTTP handler，让 trace UI 显示每个请求和各阶段。

**练习 2**：trace 显示一个 goroutine 80% 时间在 "Sync block"，可能是什么问题？怎么定位？

**练习 3**：如何用 trace 区分"GC stop-the-world 卡 1ms" vs "调度延迟 1ms"？

**练习 4**：MMU 0 在某段窗口意味着什么？

**练习 5**：解释为什么 trace 不能完全替代 pprof。

---

## 参考答案

**练习 1**：
```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, task := trace.NewTask(r.Context(), "httpRequest")
    defer task.End()

    func() {
        defer trace.StartRegion(ctx, "parse").End()
        // parse
    }()
    func() {
        defer trace.StartRegion(ctx, "db").End()
        // db query
    }()
    func() {
        defer trace.StartRegion(ctx, "render").End()
        // render
    }()
}
```

**练习 2**：等 mutex 或 sync.Cond。定位：mutex profile 找哪些锁争用最重；或直接 trace 点开该 goroutine 看其栈。

**练习 3**：trace UI 中 GC 期间有专门的 "GC mark / GC pause" 标记；调度延迟在 "Scheduler latency profile" 中独立分类。看时间标签即可分。

**练习 4**：在那个窗口期 GC 完全占用 CPU，应用 mutator 几乎得不到时间。意味着 GC 工作过载。

**练习 5**：trace 高开销不适合长期；pprof 累计统计才能给出"top 10 函数"。trace 看时序、pprof 看分布——两者互补。

---

## 小结

| 工具 | 看什么 |
|---|---|
| go tool trace | 时序 / 调度 / GC / 网络 / syscall |
| View trace | 时间线 + P 通道 + goroutine 状态 |
| Goroutine analysis | 按创建栈分组 |
| MMU | GC 对 mutator 的最坏影响 |
| trace.NewTask + Region | 业务阶段标注 |

开销：CPU 10-30%；只在调优时短时开。

下一篇 **G24 — 精通 Go 反射（reflect）** 会拆开 reflect.Type、reflect.Value 的内部结构、性能成本、典型用例与避免反射的方法。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G23-精通-Go-runtime-trace.md`
