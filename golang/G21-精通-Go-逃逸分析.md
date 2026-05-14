# 精通 Go 逃逸分析：栈 vs 堆与 -gcflags=-m

> 课程编号：G21
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Escape Analysis 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：一个反直觉的小测试

```go
func a() int {
    n := 42
    return n
}

func b() *int {
    n := 42
    return &n
}
```

`a` 的 `n` 在栈上、随函数返回销毁；`b` 的 `n` 在**堆上**——因为它的地址逃出了函数。在 C 中 `b` 是 use-after-free；在 Go 中编译器自动把 `n` 移到堆。这就是**逃逸分析**。本章拆清楚每条逃逸规则，让你能预测哪一行代码会分配。

---

## 第一章：什么是逃逸分析

### 1.1 编译器的工作

每个函数编译时，编译器分析变量的生命周期和引用关系：

- 如果变量地址**不超出**函数：栈分配
- 如果变量地址**超出**函数（return、存入逃逸的对象、传给逃逸的函数）：堆分配

### 1.2 为什么重要

栈分配：~1ns、无 GC、随返回回收。
堆分配：~20-50ns、参与 GC 扫描、GC 时增加暂停。

**降低分配 = 降低 GC 压力 = 降低延迟**。

### 1.3 工具

```bash
go build -gcflags="-m" main.go
# 单 -m 简洁；-m -m 详细原因
```

输出示例：

```
./main.go:5:2: moved to heap: n
./main.go:10:13: &n escapes to heap
./main.go:15:9: ... argument does not escape
```

---

## 第二章：基础逃逸规则

### 2.1 返回局部变量地址

```go
func newInt() *int {
    n := 5
    return &n   // n 逃逸
}
```

### 2.2 存入 interface

```go
func print(v any) {}

func main() {
    n := 5
    print(n)   // n 装箱（escape）
}
```

interface 内部存 `(type, data)`，data 槽位通常是指针——值要进堆。

### 2.3 存入逃逸的 slice / map

```go
type Item struct{ Val int }

func collect() []*Item {
    var out []*Item
    for i := 0; i < 10; i++ {
        it := Item{Val: i}
        out = append(out, &it)   // it 逃逸
    }
    return out
}
```

`out` 逃逸 → 它持有的每个 `&it` 也逃逸 → it 在堆。

### 2.4 闭包捕获

```go
func counter() func() int {
    n := 0
    return func() int { n++; return n }   // n 逃逸
}
```

返回的闭包持有 n 引用，n 必须在堆。

### 2.5 channel send

```go
ch := make(chan *Item, 1)
it := &Item{}
ch <- it   // it 逃逸（无法证明接收方在当前函数内）
```

### 2.6 太大的栈对象

```go
var huge [10 * 1024 * 1024]byte   // 10MB
```

栈最大 1GB 但分配大栈对象会"逃逸"到堆——避免栈拷贝（goroutine 栈搬迁）成本。

---

## 第三章：fmt.Println 为何让一切逃逸

### 3.1 经典 escape 案例

```go
func main() {
    n := 5
    fmt.Println(n)   // n 逃逸
}
```

```bash
$ go build -gcflags="-m" main.go
./main.go:5:14: ... argument does not escape
./main.go:5:14: n escapes to heap
```

原因：`fmt.Println(args ...any)`——参数装箱为 `[]any`，每个值变 interface{}，触发 escape。

### 3.2 解决

热点路径用 `strconv.Itoa` / `strings.Builder` / `io.Writer + Fprintln`：

```go
buf := []byte(strconv.AppendInt(nil, int64(n), 10))
os.Stdout.Write(buf)
```

或者：在生产代码里 `log` 包内部已有优化（虽然也走 interface）；调试用 fmt 没问题，性能热点改写。

---

## 第四章：函数内联

### 4.1 inlining 改变逃逸

```go
func square(x int) int { return x * x }

func main() {
    n := 5
    _ = square(n)   // square 被内联，n 不逃逸
}
```

编译器把 `square` 的代码"复制"到调用点，原本的"传参"消失，自然不逃逸。

### 4.2 内联的边界

不可内联的函数：
- 包含 `defer`（Go 1.13 后部分可）
- 太复杂（"内联预算"超限）
- 调用 panic / recover

可手动控制：

```go
//go:noinline   // 编译器指令：禁止内联
func bigFunc() { ... }
```

### 4.3 看内联决策

```bash
go build -gcflags="-m=2"
# can inline foo
# inlining call to bar
```

`-m=2` 显示更详细的决策。

---

## 第五章：常见陷阱

### 5.1 大 struct 值传递

```go
type Big struct{ data [1024]byte }
func process(b Big) {}    // 每次复制 1KB
```

```go
func process(b *Big) {}   // 传指针；但 b 是否逃逸还要看 process 内部
```

### 5.2 切片接受者 vs 数组接受者

```go
type buf [4096]byte
func (b *buf) write(...) {}   // 指针接受者
func (b buf) write(...) {}    // 值接受者；每次调用复制 4KB
```

### 5.3 闭包持有大对象

```go
func work() {
    big := make([]byte, 1<<20)
    go func() {
        // 用 big 一小段
        _ = big[:100]
    }()
}
```

闭包捕获 big → big 逃逸 + 1MB 堆。修：拷贝一小段进闭包。

### 5.4 接口参数让值逃逸

```go
func sum(items []any) int {
    var s int
    for _, x := range items {
        s += x.(int)
    }
    return s
}

func main() {
    s := sum([]any{1, 2, 3})   // 1, 2, 3 都装箱逃逸
}
```

修：用 `[]int` 而不是 `[]any`，或用泛型。

### 5.5 map / chan 中的值

```go
m := map[string][1024]byte{}
m["a"] = [1024]byte{}   // value 拷贝进 map
```

map 内部是堆分配的，所以 value 也在堆。但 [1024]byte 值类型——没指针，GC 不扫描。

```go
m := map[string]*Big{}   // 每个 value 是堆对象
```

---

## 第六章：解读 -gcflags=-m

### 6.1 常见消息

| 消息 | 含义 |
|---|---|
| `escapes to heap` | 显式逃逸（取地址、interface） |
| `moved to heap` | 编译器决定移到堆 |
| `... argument does not escape` | 函数内的引用不超出 |
| `leaking param: x` | 参数地址逃出 |
| `leaking param: x to result ...` | 参数地址作为返回值 |
| `can inline` | 函数可内联 |
| `inlining call to f` | 内联了 f |

### 6.2 leaking param

```go
func get(s []int) *int { return &s[0] }
// leaking param: s
```

意思：`s` 的某部分被返回——调用方传入的 slice 底层会逃逸。

### 6.3 详细模式

```bash
go build -gcflags="-m=2" main.go 2>&1 | head -50
```

每行附带原因（哪一行触发的）。

---

## 第七章：手动控制工具

### 7.1 //go:nosplit

```go
//go:nosplit
func leaf() { /* 简短代码 */ }
```

告诉编译器：不要插入栈检查序言。多用于 runtime 内部。一般业务代码不用。

### 7.2 //go:noinline

```go
//go:noinline
func dontInline() { ... }
```

调试 / benchmark 时强制不内联。

### 7.3 unsafe.Pointer

```go
type Header struct{ Data uintptr; Len, Cap int }
```

绕过 escape 分析（不推荐），见 G25。

### 7.4 sync.Pool

参考 G13——复用堆对象，弥补 escape 不可避免的场景。

---

## 第八章：实战优化案例

### 8.1 字符串拼接

```go
// 反例
func concat(parts []string) string {
    s := ""
    for _, p := range parts { s += p }
    return s
}
```

每次 `s += p` 分配新底层数组。改：

```go
func concat(parts []string) string {
    var sb strings.Builder
    sb.Grow(estimateSize(parts))
    for _, p := range parts { sb.WriteString(p) }
    return sb.String()
}
```

### 8.2 JSON marshal 预分配

```go
buf := make([]byte, 0, 4096)
e := json.NewEncoder(bytes.NewBuffer(buf))
```

或用 `easyjson` / `sonic` 等零分配 JSON 库。

### 8.3 转换 []byte ↔ string

```go
s := string(buf)   // 复制
b := []byte(s)     // 复制
```

热点路径用 Go 1.20+ `unsafe.String` / `unsafe.Slice`（见 G02）。

### 8.4 减少接口装箱

```go
// 反例
type Element interface{ ID() int }
items := []Element{User{1}, User{2}}   // 装箱

// 优化
items := []User{User{1}, User{2}}
// 或泛型
func process[T Identifier](items []T) {}
```

### 8.5 字段 vs 指针字段

```go
type Conn struct {
    addr  string   // 直接字段
    timer *Timer   // 指针字段（必要时才创建）
}
```

直接字段不分配，零值即可用；指针字段灵活但分配。

---

## 第九章：生产级最佳实践

1. **热点路径 benchmark + `-gcflags=-m`** 双手段确认零分配。
2. **避免 `interface{}` 在热路径**：用泛型或具体类型。
3. **大值传指针**：单 struct >64 字节用 *T 传递。
4. **可复用 buffer 用 sync.Pool**。
5. **不要为了"灵活"无脑用 *T**：值字段更省 GC。
6. **fmt 在生产路径少用**：日志用 `slog` 或 `zap` 高性能库。
7. **map / slice key 用值类型**：避免连锁逃逸。
8. **预估容量预分配**：`make([]T, 0, n)`、`Grow`。
9. **`-gcflags="-m -l"`** 看不内联时的真实逃逸。
10. **优化基于数据**：benchstat 验证。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：interface 参数装箱
```go
log.Printf("%d", n)   // n 装箱
```

### ❌ 陷阱 2：返回大值类型
```go
func f() [1024]byte { ... }
```
返回 *T 或 []byte。

### ❌ 陷阱 3：闭包持有大变量
```go
big := make([]byte, 1<<20)
go func() { _ = big[:10] }()
```

### ❌ 陷阱 4：忽视内联失败导致逃逸
内联失败 → 参数走"正常 ABI" → 容易逃逸。检查 `-m=2`。

### ❌ 陷阱 5：用 *bool 表示可选
每个 *bool 是一次分配。用 (bool, bool) 或 atomic.Bool。

### ❌ 陷阱 6：误以为 `new(int)` 一定堆
`new(T)` 返回的指针逃逸时才堆；否则可能栈。

### ❌ 陷阱 7：栈太大被强制逃逸
单变量 > 几 MB 时 escape。改 chunked 处理。

---

## 第十一章：练习题

**练习 1**：以下变量哪些逃逸？
```go
func foo() {
    a := 5
    b := &a
    c := make([]int, 10)
    d := make([]*int, 0, 10)
    e := []byte("hello")
    _ = b; _ = c; _ = d; _ = e
}
```

**练习 2**：以下函数为何性能差？修复。
```go
func sum(xs ...any) any {
    var s int
    for _, x := range xs { s += x.(int) }
    return s
}
```

**练习 3**：解释为什么 `fmt.Sprintf("%d", n)` 让 n 逃逸。

**练习 4**：以下代码 -gcflags=-m 输出什么？
```go
type S struct{ data [1024]byte }
func work() {
    var s S
    process(&s)
}
func process(s *S) {}
```

**练习 5**：写一个零分配的 `Hash(b []byte) uint64`（不能取 b 内地址逃逸）。

---

## 参考答案

**练习 1**：
- `a`：取地址但 `b` 没逃出，理论上栈；如果 `b` 进一步逃出则 `a` 也逃。当前 `_ = b`，编译器多半保留栈。
- `c`：栈（编译器看出 len 是常量）
- `d`：栈（同上）
- `e`：栈（`[]byte("...")` 编译期已知大小）

实际用 `-m` 看具体输出。

**练习 2**：每次调用 `s := sum(1, 2, 3)`——三个 int 装箱成 any，slice 也分配；返回的 any 又是装箱。改成 `func sum(xs ...int) int`，或用泛型 `func sum[T Number](xs ...T) T`。

**练习 3**：`fmt.Sprintf(format string, args ...any)` —— args 是 `[]any`，每个值装箱进 interface{}，触发 escape。即使 fmt 内部不长期持有 args，编译器看不到内部逻辑，保守逃逸。

**练习 4**：约：
```
work s does not escape
process s does not escape
```
但仍可能 inline → process 消失。`-m=2` 看详细。

**练习 5**：
```go
func Hash(b []byte) uint64 {
    var h uint64 = 14695981039346656037
    for _, c := range b {
        h ^= uint64(c)
        h *= 1099511628211
    }
    return h
}
```
`b` 是参数 slice，调用方传入的 slice header 在栈；range 不取元素地址，hash 仅累加 uint64。`-m` 应显示 `b does not escape`。

---

## 小结

| 规则 | 触发逃逸 |
|---|---|
| 返回局部地址 | yes |
| 存入 interface | yes |
| 进 slice/map/chan（如果它逃） | yes |
| 闭包捕获 + 闭包逃 | yes |
| 大栈对象 | yes |
| 内联后 | 可能不逃 |
| 简单局部使用 | no |

工具：

- `go build -gcflags="-m"` 看决策
- `-m=2` 看详细原因
- benchmark `-benchmem` 看实际分配
- pprof heap 看运行时分配

下一篇 **G22 — 精通 pprof 性能剖析** 会讲清 CPU profile、heap profile、goroutine profile、block profile、mutex profile 的采集与解读。

---

