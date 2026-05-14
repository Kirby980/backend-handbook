# 精通 Go Benchmarking 与 benchstat

> 课程编号：G19
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Benchmarks 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：一次"假"加速

```go
func BenchmarkSum(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s = 0
        for j := 0; j < 1000; j++ {
            s += j
        }
    }
}
```

跑出 0.3ns/op。"哇好快！"——其实是编译器发现 `s` 没人用，把整个循环优化掉了。这章讲清楚 `testing.B` 的工作机制、如何写出真实可靠的 benchmark、用 benchstat 做对比、规避编译器优化。

---

## 第一章：testing.B 基础

### 1.1 函数签名

```go
func BenchmarkXxx(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // 被测代码
    }
}
```

`b.N` 是 runtime 决定的迭代数：从 1 开始，每次乘 ~10 直到测得稳定耗时。最终输出 `ns/op`、`B/op`、`allocs/op`。

### 1.2 运行

```bash
go test -bench=.                # 所有 Benchmark
go test -bench=BenchmarkX       # 名字过滤
go test -bench=. -benchmem      # 显示内存/分配
go test -bench=. -benchtime=3s  # 每个 benchmark 至少跑 3s
go test -bench=. -count=10      # 跑 10 次取统计
go test -bench=. -cpu=1,2,4,8   # 不同 GOMAXPROCS
```

### 1.3 输出格式

```
BenchmarkSum-8         3000000      450 ns/op       0 B/op       0 allocs/op
```

含义：
- `-8`：GOMAXPROCS
- `3000000`：b.N 实际值
- `450 ns/op`：每次操作平均耗时
- `0 B/op`：每次操作分配字节数
- `0 allocs/op`：每次操作分配对象数

---

## 第二章：编写正确的 benchmark

### 2.1 防止编译器优化

```go
var result int   // 包级变量

func BenchmarkSum(b *testing.B) {
    var s int
    for i := 0; i < b.N; i++ {
        s = 0
        for j := 0; j < 1000; j++ { s += j }
    }
    result = s   // 防止编译器认为 s 没用
}
```

把结果赋给**包级变量**（带 sink 模式），编译器不敢删。

### 2.2 ResetTimer

```go
func BenchmarkProcess(b *testing.B) {
    data := loadHugeData()   // setup
    b.ResetTimer()           // 之前的时间不计入
    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

setup 阶段（加载数据、初始化）通常不算 benchmark 一部分。

### 2.3 StopTimer / StartTimer

```go
for i := 0; i < b.N; i++ {
    b.StopTimer()
    setup()                  // 每次迭代的 setup 不算
    b.StartTimer()
    work()
}
```

慎用——StopTimer/StartTimer 本身有 ~100ns 开销。如果 work 也只有几十 ns，结果失真。

### 2.4 b.ReportAllocs

```go
func BenchmarkX(b *testing.B) {
    b.ReportAllocs()   // 始终报告 B/op 和 allocs/op
    // ...
}
```

或全局加 `-benchmem`。

### 2.5 b.SetBytes

```go
func BenchmarkCopy(b *testing.B) {
    src := make([]byte, 4096)
    dst := make([]byte, 4096)
    b.SetBytes(4096)
    for i := 0; i < b.N; i++ {
        copy(dst, src)
    }
}
```

`SetBytes(n)` 让输出额外包含 MB/s（吞吐量）：

```
BenchmarkCopy-8   200000000     7.5 ns/op   546.5 MB/s
```

适合 I/O、序列化、压缩等以"吞吐"衡量的场景。

---

## 第三章：sub-benchmarks

### 3.1 b.Run

```go
func BenchmarkSort(b *testing.B) {
    for _, n := range []int{100, 1000, 10000, 100000} {
        b.Run(fmt.Sprintf("n=%d", n), func(b *testing.B) {
            data := rand.Perm(n)
            b.ResetTimer()
            for i := 0; i < b.N; i++ {
                sort.Ints(append([]int(nil), data...))
            }
        })
    }
}
```

输出：

```
BenchmarkSort/n=100-8       ...
BenchmarkSort/n=1000-8      ...
BenchmarkSort/n=10000-8     ...
BenchmarkSort/n=100000-8    ...
```

### 3.2 用途

- 不同输入规模
- 不同算法分支
- 不同参数（buffer 大小、并发数）

---

## 第四章：b.RunParallel

### 4.1 并发 benchmark

```go
func BenchmarkPool(b *testing.B) {
    var pool sync.Pool
    pool.New = func() any { return &bytes.Buffer{} }
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            buf := pool.Get().(*bytes.Buffer)
            buf.Reset()
            buf.WriteString("hi")
            pool.Put(buf)
        }
    })
}
```

`b.RunParallel` 启动多个 goroutine 并发跑——总迭代数仍是 b.N，但分布在 goroutine 间。

### 4.2 适用

- 测试并发安全数据结构（sync.Map、sync.Pool）
- 锁的竞争情况
- 多核扩展性

### 4.3 SetParallelism

```go
b.SetParallelism(4)   // 每个 P 跑 4 个 goroutine
```

---

## 第五章：benchstat —— 对比工具

### 5.1 安装

```bash
go install golang.org/x/perf/cmd/benchstat@latest
```

### 5.2 工作流

```bash
git checkout main
go test -bench=. -count=10 -benchmem > old.txt

git checkout my-optimization
go test -bench=. -count=10 -benchmem > new.txt

benchstat old.txt new.txt
```

输出示例：

```
                    │   old.txt    │              new.txt              │
                    │    sec/op    │   sec/op     vs base              │
Sum-8                 450.0n ± 1%   220.0n ± 2%  -51.11% (p=0.000 n=10)

                    │   old.txt    │              new.txt              │
                    │     B/op     │    B/op      vs base              │
Sum-8                 24.00 ± 0%    0.00 ± 0%   -100.00% (p=0.000 n=10)
```

`-51.11%` + `p=0.000` 表示：新版快了 51%，t-test p-value < 0.05 高度显著。

### 5.3 为什么必须 -count=N

单次 benchmark 受系统噪音影响（GC、其他进程、temperature throttling）。`-count=10` 跑 10 次让 benchstat 做 t-test。

### 5.4 平稳的测试环境

- 关闭浏览器、IDE 等占 CPU 的进程
- 笔记本：电源接好、关闭省电模式
- 服务器：避开高峰
- 同一台机器对比

---

## 第六章：测量内存

### 6.1 内存分配

`-benchmem` 显示 `B/op` 和 `allocs/op`。Go 内置 runtime 追踪：

- B/op：每次操作 heap 分配字节数（不算栈）
- allocs/op：每次操作分配对象数

### 6.2 减少分配的常见手段

| 反模式 | 优化 |
|---|---|
| `s += "x"` 循环 | strings.Builder |
| `bytes.Buffer` 默认 | Pool 复用 |
| `[]byte(s)` 拷贝 | unsafe.SliceData（谨慎） |
| `fmt.Sprintf` 热点 | strconv |
| `interface{}` 装箱 | 具体类型 |

### 6.3 内存吞吐 vs 速度

有时减少 alloc 反而慢（如复杂的 pool 管理开销超过省的 GC）。**benchmark 验证再决定**。

---

## 第七章：runtime 影响

### 7.1 GC 暂停

如果 benchmark 内部 alloc 多，GC 周期性触发，对耗时影响大。可以：

```go
runtime.GC()           // 跑前手动 GC，让基线干净
b.ResetTimer()
```

或环境变量：
```bash
GOGC=off go test -bench=.   // 关 GC（仅适合短测试，否则 OOM）
```

### 7.2 CPU 亲和性

不同 CPU 频率、温度会让结果抖动。

### 7.3 输入大小敏感

`len(data) = 100` vs `len(data) = 10_000_000` 行为可能完全不同：
- 小数据：缓存命中、栈分配主导
- 大数据：内存带宽、TLB miss 主导

写多个 size 的 sub-benchmark。

---

## 第八章：典型 benchmark 例子

### 8.1 字符串拼接对比

```go
func BenchmarkConcatPlus(b *testing.B) {
    parts := []string{"a", "b", "c", "d", "e"}
    var s string
    for i := 0; i < b.N; i++ {
        s = ""
        for _, p := range parts { s += p }
    }
    _ = s
}

func BenchmarkConcatBuilder(b *testing.B) {
    parts := []string{"a", "b", "c", "d", "e"}
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for _, p := range parts { sb.WriteString(p) }
        _ = sb.String()
    }
}
```

典型输出：Builder 比 + 快 3-10 倍。

### 8.2 map vs sync.Map

```go
func BenchmarkMapMutex(b *testing.B) {
    var mu sync.RWMutex
    m := map[int]int{}
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock(); m[1] = 1; mu.Unlock()
            mu.RLock(); _ = m[1]; mu.RUnlock()
        }
    })
}

func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Store(1, 1)
            m.Load(1)
        }
    })
}
```

读多写少时 sync.Map 通常胜出；写多则反过来。

### 8.3 JSON 序列化

```go
type User struct { Name string; Age int }

func BenchmarkJSONEncode(b *testing.B) {
    u := User{Name: "Alice", Age: 30}
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        _, _ = json.Marshal(u)
    }
}
```

加 `b.SetBytes(int64(estimatedSize))` 看吞吐。

---

## 第九章：常见反例

### 9.1 编译器消除了被测代码

```go
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = Compute()   // _ 让编译器认为没用
    }
}
```

修：sink 到包级变量。

### 9.2 setup 在循环内

```go
for i := 0; i < b.N; i++ {
    data := load()   // 每次都重新 load
    process(data)
}
```

修：移到循环外 + ResetTimer。

### 9.3 测了 wall clock 不是 CPU

如果代码包含 `time.Sleep`、`syscall` 阻塞，b.N 自适应仍可工作，但每秒"操作数"含义不同。

### 9.4 没 -count
单次 benchmark 不可信。永远 `-count=10` 起步。

### 9.5 在不同机器对比

不同 CPU 不同结果。同一台机器跑 old / new。

---

## 第十章：生产级最佳实践

1. **永远 ResetTimer 跳过 setup**。
2. **结果赋给包级 sink 变量防优化**。
3. **`-count=10 -benchmem` 是默认**。
4. **新优化必须用 benchstat 验证 + p<0.05**：避免幻觉。
5. **不同输入规模写 sub-benchmark**。
6. **并发用 b.RunParallel**：测扩展性。
7. **吞吐量场景用 b.SetBytes**：直接读 MB/s。
8. **关闭后台干扰**：浏览器、IDE。
9. **CI 跑 benchmark 监控趋势**：性能回退预警。
10. **优化前先 profile（G22）**：避免无谓优化。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：编译器消除
```go
for i := 0; i < b.N; i++ { Compute() }   // 结果未使用
```

### ❌ 陷阱 2：忘 ResetTimer
setup 把首批 b.N 拉低，后续 b.N 增大稀释，结果偏差。

### ❌ 陷阱 3：循环外创建变量但循环内修改
```go
buf := make([]byte, 1024)
for i := 0; i < b.N; i++ { fill(buf) }
```
有时是对的（复用），有时该新建（测分配）—— 看你**测什么**。

### ❌ 陷阱 4：用绝对值比较
`450ns` 在不同机器差异巨大。永远 benchstat 对比 vs base。

### ❌ 陷阱 5：忽略 GC 影响
高分配 benchmark 在 GOGC=off 下表现完全不同。

### ❌ 陷阱 6：在 GOMAXPROCS=1 下测并发
测不出锁竞争。`-cpu=1,4,8` 跑多次。

### ❌ 陷阱 7：以为 ns/op 低就是好
延迟低 ≠ 吞吐高。看具体目标。

---

## 第十二章：练习题

**练习 1**：以下 benchmark 有何问题？修复。
```go
func BenchmarkStrings(b *testing.B) {
    s := ""
    for i := 0; i < b.N; i++ {
        s = "a" + "b"
    }
}
```

**练习 2**：写一个 benchmark 对比三种"set 实现"：map[T]bool、map[T]struct{}、slice + linear scan，输入规模 10、100、10000。

**练习 3**：以下 benchmark 输出 0 ns/op，为什么？
```go
func fib(n int) int { if n<2 {return n}; return fib(n-1)+fib(n-2) }
func BenchmarkFib(b *testing.B) {
    for i := 0; i < b.N; i++ { fib(20) }
}
```

**练习 4**：写一段验证 `bytes.Buffer` 用 Pool 复用比每次新建快多少。

**练习 5**：解释 `b.RunParallel` 的 pb.Next() 如何在 goroutine 间分配迭代。

---

## 参考答案

**练习 1**：① `"a"+"b"` 是编译期常量，可能被折叠；② `s` 没被消费。修：
```go
parts := []string{"a", "b", "c"}
var sink string
for i := 0; i < b.N; i++ {
    s := ""
    for _, p := range parts { s += p }
    sink = s
}
_ = sink
```

**练习 2**：
```go
func BenchmarkSet(b *testing.B) {
    sizes := []int{10, 100, 10000}
    for _, n := range sizes {
        b.Run(fmt.Sprintf("map-bool-%d", n), func(b *testing.B) {
            // ...
        })
        b.Run(fmt.Sprintf("map-struct-%d", n), ...)
        b.Run(fmt.Sprintf("slice-%d", n), ...)
    }
}
```

**练习 3**：调用结果未使用，编译器可能消除整个调用（fib 是纯函数）。修：sink 到包级变量。

**练习 4**：
```go
var sink string
func BenchmarkBuffer_Fresh(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var buf bytes.Buffer
        buf.WriteString("hello world")
        sink = buf.String()
    }
}

var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}
func BenchmarkBuffer_Pool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := bufPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.WriteString("hello world")
        sink = buf.String()
        bufPool.Put(buf)
    }
}
```

**练习 5**：内部用 atomic 计数器分配迭代。每个 pb.Next() 原子取一段，跑完后再取。直到总 b.N 跑完。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| b.N | runtime 自适应 |
| ResetTimer | setup 不计时 |
| ReportAllocs | 报 B/op、allocs/op |
| SetBytes | 输出 MB/s 吞吐 |
| sub-bench | b.Run 多输入规模 |
| RunParallel | 并发 benchmark |
| benchstat | -count=10 + t-test |
| 防优化 | sink 到包级变量 |

下一篇 **G20 — 精通 Go 内存管理** 会拆开 GC 算法、GOGC / GOMEMLIMIT、tri-color marking、write barrier、stop-the-world 等核心。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G19-精通-Go-Benchmarking.md`
