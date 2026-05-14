# Go 路线图 · 知识检测测验

> 每章 5 道题，共 150 题。答案在最末尾"参考答案"章节。
> 题目类型：单选（含真假）、概念辨析、代码输出。
> 建议：先盖住答案，独立完成；> 80% 正确视为掌握该章；< 50% 建议重读。

---

## G01 — 变量、常量与 iota

**1.1** 以下哪一项不是合法的 Go 变量声明？
- A. `var x int = 5`
- B. `var x = 5`
- C. `x := 5`（在函数外）
- D. `var (a int; b string)`

**1.2** 以下代码输出？
```go
const (
    A = iota
    B
    _
    C
)
fmt.Println(A, B, C)
```
- A. `0 1 2`
- B. `0 1 3`
- C. `1 2 4`
- D. `0 1 4`

**1.3** Go 字符串变量的零值是？
- A. `nil`
- B. `""`（长度 0 但底层指针非 nil）
- C. `""`（长度 0 且底层指针为 nil）
- D. 未定义

**1.4** 以下哪个**不是** Go 无类型常量的特性？
- A. 任意精度算术
- B. 没有具体类型直到被使用
- C. 不可参与赋值给具体类型变量
- D. 在数值上下文中有默认类型

**1.5** 以下代码运行结果？
```go
func f() (err error) {
    defer func() { err = nil }()
    return errors.New("bad")
}
fmt.Println(f())
```
- A. `bad`
- B. `nil`
- C. `<nil>`
- D. 编译错误

---

## G02 — 数据类型、Rune 与字符串内幕

**2.1** 在 64 位 Linux 上，`unsafe.Sizeof(int(0))` 等于？
- A. 4
- B. 8
- C. 16
- D. 取决于编译器

**2.2** 以下代码输出？
```go
s := "中文"
fmt.Println(len(s), utf8.RuneCountInString(s))
```
- A. `2 2`
- B. `6 2`
- C. `6 6`
- D. `2 6`

**2.3** `rune` 类型的底层是？
- A. `byte`
- B. `int32`
- C. `int64`
- D. `uint8`

**2.4** Go 字符串可以被修改吗？
- A. 可以，通过下标赋值
- B. 不可以，必须转 `[]byte` 后修改再转回
- C. 仅常量字符串不可修改
- D. 通过 unsafe 可以但破坏不可变性

**2.5** 以下两段，哪个会触发 map 查找时的零拷贝优化（编译器特例）？
```go
// A
buf := []byte("key")
v := m[string(buf)]

// B
buf := []byte("key")
k := string(buf)
v := m[k]
```
- A. A
- B. B
- C. 都会
- D. 都不会

---

## G03 — 切片：底层结构与扩容机制

**3.1** 在 64 位机上一个 `[]int` 占多少字节（slice header 本身，不含底层数组）？
- A. 8
- B. 16
- C. 24
- D. 32

**3.2** 以下代码输出？
```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3]
b = append(b, 99)
fmt.Println(a[3])
```
- A. `4`
- B. `99`
- C. `0`
- D. 不确定

**3.3** `var s []int` 和 `s := []int{}` 的关键差别在？
- A. 长度不同
- B. 容量不同
- C. 前者 `s == nil` 为 true，后者为 false
- D. 没有差别

**3.4** Go 1.18+ 切片扩容规则相比 1.18 之前的主要改变是？
- A. 永远倍增
- B. 永远 1.25x
- C. 在 256+ 范围内更平滑的过渡
- D. 取决于元素大小

**3.5** 三索引切片 `s[a:b:c]` 中 `c` 限定的是？
- A. 长度
- B. 容量
- C. 步长
- D. 元素索引上限

---

## G04 — Map：哈希内幕与并发

**4.1** Go map 的每个 bucket 默认能容纳多少个 key-value 对？
- A. 4
- B. 8
- C. 16
- D. 32

**4.2** 以下代码运行结果？
```go
m := map[int]int{}
go func() { m[1] = 1 }()
go func() { m[2] = 2 }()
time.Sleep(time.Second)
```
- A. 正常运行
- B. 可能 panic 但能 recover
- C. fatal error，不可 recover
- D. 编译错误

**4.3** `delete(m, k)` 后 m 的内存是否会立刻释放？
- A. 是
- B. 否，bucket 数组保留；要 `m = make(...)` 重建才释放
- C. 取决于 GOGC
- D. Go 1.21+ 已修复

**4.4** 以下哪个**不是** sync.Map 的设计场景？
- A. 读远多于写
- B. key 集合稳定
- C. 多 goroutine 操作不同 key
- D. 写多读少

**4.5** 以下代码合法吗？
```go
type Point struct{ X, Y int }
m := map[string]Point{"a": {1, 2}}
m["a"].X = 99
```
- A. 合法，X 改为 99
- B. 合法但 X 不变
- C. 编译错误：cannot assign to struct field
- D. 运行时 panic

---

## G05 — Struct：内存布局与嵌入

**5.1** 以下 struct 的 `unsafe.Sizeof` 是？
```go
type S struct {
    a bool
    b int64
    c bool
}
```
- A. 10
- B. 16
- C. 24
- D. 12

**5.2** 把上题字段重排为 `b, a, c` 后 sizeof 变成？
- A. 10
- B. 16
- C. 24
- D. 12

**5.3** `struct{}` 占用多少字节？
- A. 0
- B. 1
- C. 取决于编译器
- D. 8

**5.4** 以下代码合法吗？
```go
type A struct{}
func (A) Foo() {}

type B struct{}
func (B) Foo() {}

type C struct{ A; B }

var c C
c.Foo()
```
- A. 合法，自动选 A 或 B
- B. 合法，编译器报警
- C. 编译错误：ambiguous selector
- D. 运行时 panic

**5.5** 一个 struct 在内存中的总大小，必须是其字段中**最大对齐**的整数倍。这种说法：
- A. 正确
- B. 错误，无对齐要求
- C. 仅 64 位平台成立
- D. 仅含指针字段时成立

---

## G06 — 函数、闭包与 defer

**6.1** 以下代码输出？
```go
func main() {
    for i := 0; i < 3; i++ {
        defer fmt.Print(i)
    }
}
```
- A. `012`
- B. `210`
- C. `000`
- D. `222`

**6.2** 以下代码输出？
```go
func g() (n int) {
    defer func() { n++ }()
    return 10
}
fmt.Println(g())
```
- A. `10`
- B. `11`
- C. `0`
- D. 编译错误

**6.3** 闭包捕获外部变量是按？
- A. 值
- B. 引用
- C. 取决于变量类型
- D. 默认值，可显式改

**6.4** Go 1.14 引入的"开放编码 defer"主要解决什么？
- A. defer 嵌套限制
- B. defer 的运行时开销
- C. defer 不能修改返回值
- D. defer 不能在 goroutine 中用

**6.5** 以下代码哪一行**立即求值**？
```go
i := 1
defer fmt.Println(i)        // ①
defer func() { fmt.Println(i) }()   // ②
i = 2
```
- A. ① 立即求值 i，② 延迟求值
- B. ② 立即求值，① 延迟
- C. 两者都立即
- D. 两者都延迟

---

## G07 — 指针与方法接收者

**7.1** 以下代码合法吗？
```go
m := map[string]int{"a": 1}
p := &m["a"]
```
- A. 合法
- B. 编译错误：cannot take address of map element
- C. 运行时 panic
- D. 仅 Go 1.22+ 合法

**7.2** 类型 `*Counter` 的方法集**不**包含？
- A. 值接收者方法
- B. 指针接收者方法
- C. 嵌入字段的方法
- D. 同包其他类型的方法

**7.3** `new(int)` 返回的是？
- A. `int`，值为 0
- B. `*int`，指向值为 0 的 int
- C. `*int`，指向 nil
- D. 编译错误

**7.4** 以下代码输出？
```go
type T struct{}
a := &T{}
b := &T{}
fmt.Println(a == b)
```
- A. `true`（多数情况，因为空 struct 共享 zerobase）
- B. `false`
- C. 编译错误
- D. 运行时 panic

**7.5** 在 nil 接收者上调方法会 panic 吗？
- A. 一定 panic
- B. 一定不 panic
- C. 仅当方法解引用 receiver 时 panic
- D. 仅当方法是值接收者时 panic

---

## G08 — 接口、itab 与类型断言

**8.1** Go 中一个非空接口（如 `io.Reader`）占多少字节？
- A. 8
- B. 16
- C. 24
- D. 取决于具体类型

**8.2** 以下代码结果？
```go
type MyErr struct{}
func (*MyErr) Error() string { return "x" }
func f() error {
    var e *MyErr
    return e
}
fmt.Println(f() == nil)
```
- A. `true`
- B. `false`
- C. 运行时 panic
- D. 编译错误

**8.3** itab 是什么？
- A. 接口的实例
- B. 接口类型与具体类型的桥（含方法函数指针表）
- C. 类型反射的别名
- D. GC 内部结构

**8.4** 以下代码哪种最常导致小值"装箱到堆"？
- A. `var x int = 5`
- B. `var i any = 5`
- C. `var p *int = &x`
- D. `var s []int = []int{5}`

**8.5** 接口实现是显式还是隐式？
- A. 显式：必须用 `implements`
- B. 隐式：实现了方法集即满足
- C. 半隐式：在 import 时声明
- D. 取决于编译器开关

---

## G09 — 泛型

**9.1** Go 泛型在哪一版引入？
- A. 1.16
- B. 1.17
- C. 1.18
- D. 1.20

**9.2** 以下约束的含义是什么？
```go
type Number interface {
    ~int | ~float64
}
```
- A. 仅 `int` 与 `float64`
- B. `int`、`float64` 及其衍生类型（如 `type MyInt int`）
- C. 任意数值类型
- D. 编译错误

**9.3** Go 泛型在方法上的限制是？
- A. 方法不能有泛型参数本身
- B. 方法不能引入"自己的"类型参数（只能用类型本身的）
- C. 接口方法不能用泛型
- D. B 和 C 都对

**9.4** Go 泛型的实现策略是？
- A. C++ 风格 monomorphization
- B. Java 风格类型擦除
- C. GC shape stenciling + dictionary
- D. 全运行时反射

**9.5** 以下哪个**不是** Go 内置的预声明约束？
- A. `any`
- B. `comparable`
- C. `ordered`
- D. 以上都是预声明的

---

## G10 — 错误处理、panic 与 recover

**10.1** Go 错误包装的格式符是？
- A. `%v`
- B. `%w`
- C. `%e`
- D. `%err`

**10.2** 以下代码输出？
```go
var ErrA = errors.New("A")
err := fmt.Errorf("wrap: %w", ErrA)
fmt.Println(err == ErrA, errors.Is(err, ErrA))
```
- A. `true true`
- B. `false true`
- C. `false false`
- D. `true false`

**10.3** 以下哪种情况 recover 不能捕获？
- A. nil pointer dereference
- B. index out of range
- C. concurrent map writes
- D. 自己 panic("x")

**10.4** `recover()` 必须在哪里调用才有效？
- A. main 函数中
- B. defer 函数中
- C. goroutine 启动后第一行
- D. 任意位置都行

**10.5** Go 1.20 引入的合并多个错误的函数是？
- A. `errors.Merge`
- B. `errors.Combine`
- C. `errors.Join`
- D. `errors.All`

---

## G11 — Goroutines 与 GMP 调度

**11.1** Goroutine 的初始栈大小约为？
- A. 4 KB
- B. 2 KB
- C. 8 KB
- D. 64 KB

**11.2** GMP 中 P 表示？
- A. Process
- B. Pipe
- C. Processor（逻辑处理器，持有 runqueue 和 mcache）
- D. Pthread

**11.3** Go 1.14 引入的抢占机制基于？
- A. 函数调用检查点
- B. 协程主动 yield
- C. 信号（SIGURG）
- D. CPU 中断

**11.4** 阻塞 syscall 期间发生什么？
- A. 整个进程阻塞
- B. GOMAXPROCS 减 1
- C. M 与 P 解绑，P 可被其他 M 拿去跑别的 G
- D. 当前 P 销毁

**11.5** 容器内 GOMAXPROCS 在 Go 1.22 之前的常见问题是？
- A. 总是 1
- B. 看到宿主机核数，不尊重 cgroup limit
- C. 不能修改
- D. 影响 GC

---

## G12 — Channels 与 select

**12.1** 关闭已关闭的 channel 会？
- A. 静默 no-op
- B. panic
- C. 仅打印 warning
- D. 返回 error

**12.2** 向已关闭的 channel **接收**：
- A. panic
- B. 阻塞
- C. 返回零值 + ok=false（缓冲排空后）
- D. 立即返回 ErrClosed

**12.3** 向 nil channel send 会？
- A. panic
- B. 永久阻塞
- C. 立即返回
- D. 编译错误

**12.4** select 中所有 case 都阻塞且**没**有 default，行为是？
- A. 退出 select
- B. 阻塞，直到某 case 就绪
- C. panic
- D. 走第一个 case

**12.5** select 多个 case 同时就绪时如何选？
- A. 按声明顺序
- B. 按反向顺序
- C. 随机
- D. 按 channel 编号

---

## G13 — sync 包

**13.1** 复制一个 `sync.Mutex` 会发生什么？
- A. 编译错误
- B. 运行时 panic
- C. 合法但通常是 bug（go vet copylocks 警告）
- D. 等同 `new(sync.Mutex)`

**13.2** `sync.RWMutex` 在 Go 实现中倾向于？
- A. 读者优先
- B. 写者优先（避免写饥饿）
- C. FIFO 顺序
- D. 随机

**13.3** `sync.WaitGroup.Add(n)` 应该在？
- A. goroutine 启动前
- B. goroutine 内部首行
- C. goroutine 结束后
- D. 任意位置

**13.4** `sync.Pool` 的对象在什么时候被清空？
- A. 程序退出
- B. 显式调用 Reset
- C. 每次 GC
- D. 超过容量

**13.5** Go 1.19+ 提倡使用以下哪个？
- A. `atomic.AddInt64(&n, 1)`（包级函数）
- B. `n.Add(1)` 其中 n 是 `atomic.Int64`（类型化）
- C. `sync.Mutex` 加锁更新
- D. `interface{}` + 锁

---

## G14 — context 包

**14.1** 以下哪个根 context 表示"还没决定用什么"？
- A. `context.Background()`
- B. `context.TODO()`
- C. `context.Nil()`
- D. `context.New()`

**14.2** `context.WithValue` 的 key 应该用什么类型？
- A. `string`
- B. `int`
- C. 包私有类型（避免冲突）
- D. `any`

**14.3** 以下代码有何问题？
```go
ctx, _ := context.WithCancel(context.Background())
go func() { <-ctx.Done() }()
```
- A. 编译错误
- B. cancel 函数丢失，goroutine 永远阻塞 → 泄漏
- C. 没有问题
- D. ctx 立即取消

**14.4** `context.Background().Done()` 返回什么？
- A. 已关闭的 channel
- B. nil channel（永不触发）
- C. 缓冲为 1 的 channel
- D. panic

**14.5** 以下哪个**不该**放进 context？
- A. trace ID
- B. request ID
- C. 数据库连接
- D. 用户身份

---

## G15 — 并发模式

**15.1** "fan-out" 模式描述？
- A. 一个生产者向多个消费者分发任务
- B. 多个生产者汇总到一个消费者
- C. 长任务定期汇报进度
- D. 限制并发数

**15.2** errgroup 的核心能力是？
- A. 一次启动多个 goroutine
- B. 任一 goroutine 失败时取消其他
- C. 自动 recover
- D. A 和 B 都是

**15.3** "Semaphore" 模式通常用什么实现？
- A. atomic 计数器
- B. buffered channel
- C. mutex
- D. 全局变量

**15.4** 以下泄漏的代码如何修复？
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
- A. `ch := make(chan struct{}, 1)` 让 send 非阻塞
- B. 用 mutex
- C. 用 sync.Pool
- D. 加 defer close(ch)

**15.5** "Tee" 模式将一个输入复制到几个输出？
- A. 一个
- B. 两个
- C. 任意（用反射）
- D. 取决于参数

---

## G16 — Race Detection

**16.1** 构成 data race 的四要素**不**包括？
- A. 多个 goroutine
- B. 同一内存位置
- C. 至少一个写
- D. 同时执行

**16.2** Go 的 `-race` 基于？
- A. Valgrind
- B. ThreadSanitizer (TSan)
- C. AddressSanitizer
- D. Go 自研工具

**16.3** `-race` 的开销大约是？
- A. 10% CPU
- B. 100% CPU
- C. 5-10x CPU 和内存
- D. 几乎无开销

**16.4** 以下哪个**不能**建立 happens-before？
- A. channel send/recv
- B. mutex Lock/Unlock
- C. 函数返回
- D. atomic load/store（Go 1.19+）

**16.5** 以下代码：
```go
var x int
done := make(chan struct{})
go func() { x = 1; close(done) }()
<-done
fmt.Println(x)
```
- A. 有 data race
- B. 无 data race
- C. 编译错误
- D. 偶尔有 race

---

## G17 — Go Modules

**17.1** Go modules 选择依赖版本的算法叫？
- A. SAT solver
- B. MVS (Minimum Version Selection)
- C. SemVer match
- D. Latest available

**17.2** v2+ 的模块路径必须？
- A. 加 `/v2` 后缀
- B. 改名
- C. 升级 Go 版本
- D. 没特殊要求

**17.3** `go mod tidy` 的作用？
- A. 升级所有依赖
- B. 添加缺失、移除未用、更新 go.sum
- C. 删除 go.sum
- D. 重新下载依赖

**17.4** `GOPRIVATE` 环境变量的作用？
- A. 匹配的路径不走 proxy，不查 sumdb
- B. 设置私钥
- C. 加密 go.sum
- D. 仅限内部代码

**17.5** `go.work` 的设计目标是？
- A. 替代 go.mod
- B. 多 module 本地联合开发（不入 git）
- C. 配置 CI
- D. 监控依赖

---

## G18 — 测试

**18.1** 测试文件必须以什么结尾？
- A. `_spec.go`
- B. `_test.go`
- C. `_t.go`
- D. `test_*.go`

**18.2** `t.Run(name, fn)` 的作用？
- A. 异步执行
- B. 创建 subtest
- C. 计时
- D. 重试

**18.3** `t.Parallel()` 后多少个 subtest 并行？
- A. 1
- B. 由 -parallel 控制（默认 GOMAXPROCS）
- C. 100
- D. 无上限

**18.4** `t.Cleanup` 与 `defer` 的关键差别？
- A. 没差别
- B. t.Cleanup 跨 subtest 自动继承，失败也执行
- C. defer 更可靠
- D. t.Cleanup 仅 Go 1.22+ 可用

**18.5** Go 1.18 引入的 fuzz testing 配套字段是？
- A. `*testing.T`
- B. `*testing.B`
- C. `*testing.F`
- D. `*testing.Fuzz`

---

## G19 — Benchmarking

**19.1** `b.N` 是？
- A. 你指定的迭代数
- B. runtime 自适应决定，从小开始
- C. CPU 核数
- D. 默认 1000

**19.2** 以下 benchmark 有什么常见问题？
```go
func BenchmarkX(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Compute()
    }
}
```
- A. 没问题
- B. 编译器可能消除 Compute()（结果没用）
- C. b.N 太大
- D. 缺少 t.Parallel

**19.3** `b.ResetTimer()` 的用途？
- A. 重置 b.N
- B. 跳过 setup 阶段的耗时
- C. 重启 benchmark
- D. 强制 GC

**19.4** 对比新旧版本性能用哪个工具？
- A. go tool pprof
- B. benchstat
- C. go test -count=1
- D. go vet

**19.5** `b.SetBytes(n)` 在输出中额外显示？
- A. ns/op
- B. allocs/op
- C. MB/s 吞吐
- D. 内存峰值

---

## G20 — 内存管理：GC、GOGC、GOMEMLIMIT

**20.1** `GOGC=100`（默认）的含义？
- A. GC 占 100% CPU
- B. 堆涨到上次 GC 后存活对象的 2 倍触发下次 GC
- C. 每 100MB GC 一次
- D. 关闭 GC

**20.2** Go 当前的 GC 算法是？
- A. 分代复制
- B. 引用计数
- C. 并发三色标记 + 写屏障
- D. STW 复制

**20.3** GOMEMLIMIT 与 GOGC 的关系？
- A. GOMEMLIMIT 替代 GOGC
- B. GOMEMLIMIT 是**软限**，GC 时一并参考
- C. 互斥
- D. 必须同时设置

**20.4** Go 的小对象分配走哪一层？
- A. 直接 OS mmap
- B. mheap
- C. mcentral
- D. mcache（per-P 无锁）

**20.5** 关闭 GC 的设置？
- A. `GOGC=off`
- B. `runtime.GCStop()`
- C. `debug.GC(0)`
- D. 不能关闭

---

## G21 — 逃逸分析

**21.1** 以下代码 n 是否逃逸？
```go
func f() *int {
    n := 5
    return &n
}
```
- A. 不逃逸（栈）
- B. 逃逸到堆
- C. 取决于编译器优化
- D. 编译错误（use after free）

**21.2** 查看逃逸分析的命令？
- A. `go vet`
- B. `go build -gcflags="-m"`
- C. `go tool escape`
- D. `go env`

**21.3** 以下哪个**通常**不导致逃逸？
- A. 返回局部变量地址
- B. 装入 interface
- C. 闭包捕获后闭包逃出
- D. 局部 slice 内部使用

**21.4** `fmt.Println(n)` 为什么常让 n 逃逸？
- A. fmt 包内部用堆
- B. Println 签名是 `args ...any`，n 装箱
- C. 编译器 bug
- D. 不会逃逸

**21.5** "leaking param" 的含义？
- A. 参数泄漏内存
- B. 参数地址传给返回值或外部，导致传入对象逃逸
- C. 参数未被使用
- D. 参数类型未指定

---

## G22 — pprof

**22.1** 启用 HTTP pprof 的最简方式？
- A. 调用 `pprof.Start()`
- B. import `_ "net/http/pprof"` + 启动 HTTP server
- C. 设置 `GODEBUG=pprof=1`
- D. 编译时 `-pprof`

**22.2** Heap profile 的两种主要 metric？
- A. `cpu_space` 和 `cpu_objects`
- B. `inuse_space`（当前活）和 `alloc_space`（累计）
- C. `total` 和 `freed`
- D. `mark` 和 `sweep`

**22.3** 找 goroutine 泄漏最直接的 endpoint？
- A. `/debug/pprof/heap`
- B. `/debug/pprof/profile`
- C. `/debug/pprof/goroutine?debug=2`
- D. `/debug/pprof/block`

**22.4** pprof 输出中 `flat` 与 `cum` 的差别？
- A. flat 是堆，cum 是栈
- B. flat 是函数本身耗时，cum 含子调用
- C. flat 是采样次数，cum 是绝对时间
- D. 没差别

**22.5** mutex profile 默认开启吗？
- A. 是
- B. 否；需 `runtime.SetMutexProfileFraction(>0)`
- C. 仅 -race 时开
- D. 仅 Linux 上开

---

## G23 — runtime/trace

**23.1** trace 工具主要看？
- A. 函数热点
- B. 内存分配
- C. 时序、调度、GC、网络事件
- D. SQL 查询

**23.2** 启动 trace 采集？
- A. `go tool trace start`
- B. `trace.Start(w)` + `trace.Stop()`
- C. 设置 `GOTRACE=1`
- D. `runtime.StartTrace`

**23.3** 用户自定义 region 通过哪个 API 创建？
- A. `trace.Region(...)`
- B. `trace.StartRegion(ctx, name)`
- C. `trace.New(...)`
- D. `runtime.Trace`

**23.4** MMU 是？
- A. Memory Management Unit
- B. Minimum Mutator Utilization：给定窗口内 mutator 最低占比
- C. Max Memory Usage
- D. 与 GC 无关

**23.5** trace 的开销大约是？
- A. 几乎零
- B. CPU 10-30%
- C. 与 GOGC 相同
- D. 仅磁盘开销

---

## G24 — 反射 reflect

**24.1** reflect 的三定律**不**包括？
- A. interface 与 reflect 对象互转
- B. reflect 对象可还原为 interface
- C. 修改值需要原值可寻址
- D. reflect 仅适用于 struct

**24.2** 反射调用方法比直接调用慢约？
- A. 1.2 倍
- B. 5 倍
- C. 50-100 倍
- D. 1000 倍

**24.3** 以下代码会怎样？
```go
u := User{}
v := reflect.ValueOf(u)
v.FieldByName("Name").SetString("X")
```
- A. 正常修改
- B. panic（u 是值副本，不可寻址）
- C. 编译错误
- D. 仅写入副本

**24.4** `reflect.DeepEqual` 比 `==` 慢约？
- A. 持平
- B. 50-100 倍
- C. 2 倍
- D. 10000 倍

**24.5** 不可寻址的 reflect.Value 是因为？
- A. 来源是接口装的值（通过值传入 ValueOf）
- B. 来源是字面量
- C. 来源是 map value
- D. 以上都是

---

## G25 — unsafe 与 //go:linkname

**25.1** `unsafe.Pointer` 与 `uintptr` 的关键差别？
- A. 没差别
- B. uintptr 是数字，GC 不追踪 → 长期持有可能失效
- C. unsafe.Pointer 是 8 字节，uintptr 是 4
- D. uintptr 仅 32 位平台用

**25.2** 以下代码合法吗？
```go
a := uintptr(unsafe.Pointer(&x))
runtime.GC()
p := unsafe.Pointer(a)
```
- A. 合法
- B. 不合法：a 不被 GC 追踪，期间 x 可能被搬迁
- C. 取决于 x 大小
- D. 仅 Go 1.22+ 合法

**25.3** Go 1.20+ 提供的零拷贝 byte ↔ string API？
- A. `unsafe.B2S` / `unsafe.S2B`
- B. `unsafe.String` / `unsafe.Slice` (+ StringData / SliceData)
- C. `reflect.StringHeader`
- D. `strings.Clone`

**25.4** `//go:linkname` 需要导入什么包？
- A. `runtime`
- B. `_ "unsafe"`
- C. `syscall`
- D. 不需要导入

**25.5** `unsafe.Sizeof` 返回值是？
- A. 运行时计算
- B. 编译期常量
- C. 包级变量
- D. 取决于 GOOS

---

## G26 — CGO

**26.1** 一次 cgo call 的开销大约？
- A. 1 ns
- B. ~200 ns
- C. ~1 ms
- D. 与直接调用相同

**26.2** `C.CString(s)` 的内存如何回收？
- A. GC 自动回收
- B. 必须 `C.free(unsafe.Pointer(cs))`
- C. runtime 自动 free
- D. 不需要回收

**26.3** `CGO_ENABLED=0` 的常用场景？
- A. 永远应该关
- B. 构建静态二进制 + 容器（FROM scratch）
- C. 仅 macOS 用
- D. 关闭 GC

**26.4** Go 1.21+ 提供的"在 cgo 期间禁止 GC 移动 Go 对象"的 API？
- A. `runtime.GC()`
- B. `runtime.Pinner`
- C. `unsafe.Pin`
- D. `cgo.Pin`

**26.5** 以下哪条**不是** cgo 的代价？
- A. 调用开销 ~200ns
- B. 失去交叉编译便利
- C. 调试更复杂
- D. 失去 GC

---

## G27 — net/http

**27.1** Go 1.22+ ServeMux 新增的能力？
- A. 异步处理
- B. 方法 + 路径参数（`mux.HandleFunc("GET /users/{id}", ...)`）
- C. 自动 TLS
- D. WebSocket

**27.2** 生产 server **必设**的四个超时**不**包括？
- A. ReadTimeout
- B. WriteTimeout
- C. ConnectTimeout
- D. IdleTimeout

**27.3** Graceful shutdown 的 API？
- A. `server.Close()`
- B. `server.Stop()`
- C. `server.Shutdown(ctx)`
- D. `os.Exit(0)`

**27.4** `http.DefaultClient` 默认超时是？
- A. 30 秒
- B. 60 秒
- C. 无超时（容易卡死）
- D. 与 Server 相同

**27.5** Client 端 `resp.Body` 必须？
- A. 立即丢弃
- B. 读完并 Close（否则 keep-alive 失败）
- C. 不能读
- D. 用 atomic 读

---

## G28 — gRPC 与 Protobuf

**28.1** gRPC 默认基于哪种传输协议？
- A. HTTP/1.1
- B. HTTP/2
- C. WebSocket
- D. raw TCP

**28.2** 四种 RPC 模式中支持双向流的是？
- A. Unary
- B. Server streaming
- C. Client streaming
- D. Bidirectional streaming

**28.3** protobuf 字段编号一旦发布建议？
- A. 永远不重用
- B. 偶尔可以改
- C. 仅 v2 之后才重用
- D. 自动跳过

**28.4** gRPC error 应该用哪个返回？
- A. `errors.New`
- B. `fmt.Errorf`
- C. `status.Error(codes.X, msg)`
- D. `panic`

**28.5** Interceptor 中应该最外层放？
- A. Logger
- B. Auth
- C. Recover
- D. Metrics

---

## G29 — 数据库访问

**29.1** `sql.Open` 调用后立即建立连接吗？
- A. 是
- B. 否；首次 Query/Ping 才连
- C. 取决于 driver
- D. 仅 PostgreSQL 否

**29.2** 以下代码主要 bug？
```go
rows, _ := db.Query("SELECT ...")
for rows.Next() {
    var x int
    rows.Scan(&x)
}
```
- A. 没问题
- B. 缺 defer rows.Close()
- C. 缺 rows.Err() 检查
- D. B 和 C 都是

**29.3** PostgreSQL 在 Go 中推荐的驱动？
- A. lib/pq（已停止维护新特性）
- B. pgx
- C. mysql-driver
- D. go-pg

**29.4** N+1 问题的修复方法**不**包括？
- A. JOIN
- B. Preload
- C. IN (...) 批量查
- D. 增加连接池大小

**29.5** 标准库 `database/sql` 中开启事务用？
- A. `db.Begin()` 或 `db.BeginTx(ctx, opts)`
- B. `db.Transaction()`
- C. `db.Exec("BEGIN")`
- D. 没有内置事务

---

## G30 — 结构化日志

**30.1** Go 标准库的结构化日志包（Go 1.21+）？
- A. `log`
- B. `logger`
- C. `log/slog`
- D. `structlog`

**30.2** 以下哪个日志库性能最强？
- A. log（标准）
- B. slog
- C. zap
- D. logrus

**30.3** 容器化部署的日志输出推荐？
- A. 写本地文件
- B. JSON 到 stdout
- C. 直接 send 到 ELK
- D. 写 syslog

**30.4** Zerolog 的 API 风格是？
- A. 函数式 `zap.Int("k", v)`
- B. 链式 `log.Info().Str("k", v).Msg("...")`
- C. printf 风格
- D. struct 字面量

**30.5** 高频 INFO 日志的成本控制方法？
- A. 关闭日志
- B. 日志采样（如每 100 条输出 1 条）
- C. 改 ERROR
- D. 异步写

---

## ✅ 参考答案

### G01
1.1 **C**（包级别不能用 `:=`）
1.2 **B**（A=0, B=1, _=2 跳过, C=3）
1.3 **B**（"" 长度 0；Data 非 nil 但仍是 "" 字面常量）
1.4 **C**（赋给具体类型变量时做隐式转换，合法）
1.5 **C**（defer 改了命名返回值 err，最终返回 nil → 打印 `<nil>`）

### G02
2.1 **B**（64 位机上 int = 8）
2.2 **B**（"中文" = 6 字节，2 个 rune）
2.3 **B**（rune = int32）
2.4 **B**（必须转 []byte 再修改再转回）
2.5 **A**（A 形式编译器零拷贝；B 因 k 长期存在不优化）

### G03
3.1 **C**（24 = 3 × 8）
3.2 **B**（cap=5 时 append 不扩容，写到 a 的位置 3）
3.3 **C**（var 是 nil；[]T{} 不是）
3.4 **C**（1.18+ 在 256+ 范围更平滑）
3.5 **B**（c 限定的是底层 cap 索引）

### G04
4.1 **B**（8 slot 每 bucket）
4.2 **C**（fatal error: concurrent map writes，不可 recover）
4.3 **B**（bucket 数组不缩；要 rebuild）
4.4 **D**（写多读少时 sync.Map 慢）
4.5 **C**（map value 不可寻址，不能直接改字段）

### G05
5.1 **C**（24：bool+pad7+int64+bool+pad7）
5.2 **B**（16：b 8 + a,c 各 1 + pad6）
5.3 **A**（0 字节）
5.4 **C**（嵌入字段同名 Foo 歧义）
5.5 **A**（正确）

### G06
6.1 **B**（LIFO，每次 i 立即求值）
6.2 **B**（defer 修改命名返回值 n）
6.3 **B**（按引用）
6.4 **B**（开放编码 defer 让常见场景接近零开销）
6.5 **A**（① defer 参数立即求值，② 闭包捕获按引用，看最终值）

### G07
7.1 **B**（map value 不可寻址，无法取地址）
7.2 **D**（同包其他类型方法不在 *Counter 方法集）
7.3 **B**（new 返回指向零值的指针）
7.4 **A**（多数情况 true，因 zerobase）
7.5 **C**（不解引用 receiver 就不 panic）

### G08
8.1 **B**（iface 16 字节：tab + data）
8.2 **B**（type=*MyErr 非 nil，整个 interface != nil）
8.3 **B**（interface ↔ 具体类型的桥）
8.4 **B**（int 装 any 触发装箱）
8.5 **B**（隐式实现）

### G09
9.1 **C**（Go 1.18）
9.2 **B**（`~` 包含底层类型为 int/float64 的衍生类型）
9.3 **D**（B 和 C 都对）
9.4 **C**（GC shape stenciling + dictionary）
9.5 **C**（Ordered 在 cmp/slices 包中，不是预声明；any 和 comparable 是预声明）

### G10
10.1 **B**（%w）
10.2 **B**（直接 == false；errors.Is 沿链 true）
10.3 **C**（concurrent map writes 是 fatal error，recover 无效）
10.4 **B**（必须在 defer 函数中）
10.5 **C**（errors.Join，Go 1.20+）

### G11
11.1 **B**（2 KB）
11.2 **C**（Processor，持有 local runq 和 mcache）
11.3 **C**（基于 SIGURG 信号）
11.4 **C**（M-P 解绑）
11.5 **B**（看到宿主机核数，需要 automaxprocs 或 Go 1.22+）

### G12
12.1 **B**（重复 close panic）
12.2 **C**（返回零值 + ok=false）
12.3 **B**（永久阻塞）
12.4 **B**（阻塞）
12.5 **C**（随机选择）

### G13
13.1 **C**（合法但 go vet copylocks 警告）
13.2 **B**（写者优先）
13.3 **A**（必须在 go 启动前）
13.4 **C**（每次 GC 清空）
13.5 **B**（类型化 API：atomic.Int64.Add）

### G14
14.1 **B**（TODO）
14.2 **C**（包私有类型）
14.3 **B**（cancel 丢失 → 泄漏）
14.4 **B**（nil channel 永不触发，符合 Background 永不取消的语义）
14.5 **C**（DB 连接不该塞 context）

### G15
15.1 **A**（一对多分发）
15.2 **D**（A 和 B 都对）
15.3 **B**（buffered channel）
15.4 **A**（buffer 1 让 send 非阻塞，超时后 goroutine 也能完成）
15.5 **B**（两个）

### G16
16.1 **D**（"同时"不是要求；happens-before 才是）
16.2 **B**（ThreadSanitizer）
16.3 **C**（5-10x CPU 和内存）
16.4 **C**（函数返回不建立 happens-before）
16.5 **B**（close happens-before <-done，对 x 的写可见，无 race）

### G17
17.1 **B**（MVS）
17.2 **A**（加 /v2 后缀）
17.3 **B**
17.4 **A**
17.5 **B**（多 module 本地联合开发，不入 git）

### G18
18.1 **B**（_test.go）
18.2 **B**（subtest）
18.3 **B**（由 -parallel 控制）
18.4 **B**
18.5 **C**（*testing.F）

### G19
19.1 **B**（runtime 自适应）
19.2 **B**（结果没用，可能被消除）
19.3 **B**（跳过 setup）
19.4 **B**（benchstat）
19.5 **C**（MB/s）

### G20
20.1 **B**（堆涨 2x 触发）
20.2 **C**（并发三色标记 + 写屏障）
20.3 **B**（软限，与 GOGC 协同；接近 limit 时强制 GC）
20.4 **D**（mcache 无锁）
20.5 **A**（GOGC=off）

### G21
21.1 **B**（逃逸到堆）
21.2 **B**（`-gcflags="-m"`）
21.3 **D**（局部内部使用通常不逃）
21.4 **B**（args ...any 装箱）
21.5 **B**（参数地址逃出函数）

### G22
22.1 **B**（import _ "net/http/pprof" + 启 server）
22.2 **B**（inuse vs alloc）
22.3 **C**（goroutine?debug=2）
22.4 **B**（flat 自身，cum 含子调用）
22.5 **B**（默认关闭）

### G23
23.1 **C**（时序 / 调度 / GC / 网络）
23.2 **B**（trace.Start/Stop）
23.3 **B**（trace.StartRegion）
23.4 **B**（Minimum Mutator Utilization）
23.5 **B**（CPU 10-30%）

### G24
24.1 **D**（适用于所有类型，不只 struct）
24.2 **C**（50-100x）
24.3 **B**（u 不可寻址，panic）
24.4 **B**（50-100 倍）
24.5 **D**（以上都是）

### G25
25.1 **B**（uintptr 是数字，GC 不追）
25.2 **B**（不合法）
25.3 **B**（unsafe.String / unsafe.Slice + Data）
25.4 **B**（`_ "unsafe"`）
25.5 **B**（编译期常量）

### G26
26.1 **B**（~200 ns）
26.2 **B**（必须 C.free）
26.3 **B**（静态二进制 + scratch 镜像）
26.4 **B**（runtime.Pinner）
26.5 **D**（cgo 不"失去 GC"；只是 C 内存不受 GC 管）

### G27
27.1 **B**（方法 + 路径参数）
27.2 **C**（没有 ConnectTimeout；是 ReadHeaderTimeout/ReadTimeout/WriteTimeout/IdleTimeout）
27.3 **C**（Shutdown(ctx)）
27.4 **C**（无超时，危险）
27.5 **B**（读完 + Close）

### G28
28.1 **B**（HTTP/2）
28.2 **D**（双向流）
28.3 **A**（永不重用，删字段用 reserved）
28.4 **C**（status.Error）
28.5 **C**（Recover 最外，保证后续 panic 都被捕获）

### G29
29.1 **B**（Ping 时才连）
29.2 **D**（B 和 C 都是问题）
29.3 **B**（pgx）
29.4 **D**（连接池大不解决 N+1）
29.5 **A**（Begin / BeginTx）

### G30
30.1 **C**（log/slog）
30.2 **C**（Zap，约 100 ns/log）
30.3 **B**（JSON 到 stdout）
30.4 **B**（链式）
30.5 **B**（采样）

---

## 📊 评分标准

| 分数 | 评价 |
|---|---|
| 135+/150（>90%） | 🏆 精通—— Go 高级工程师候选 |
| 105-134（70-89%） | 🥇 熟练—— 能胜任生产开发 |
| 75-104（50-69%） | 🥈 入门—— 重点补足薄弱章节 |
| <75（<50%） | 📖 建议重读对应章节 + 动手实践 |

按模块统计错题：
- **模块 1 错 >5 道**：重做 G01-G10 中错题对应章节的练习
- **模块 2 错 >3 道**：并发是核心，强烈建议重读 G11-G15
- **模块 3-5 错较多**：可针对性查漏，不必整章重读

---

## 🔁 复习建议

1. **错的题目标记**：在 Anki / 备忘录中保存错题，一周后重做
2. **错题对应章节深读**：找到该章的"陷阱清单"和"练习题"重做
3. **代码实战**：把高频错的概念用 5-10 行代码反复实操
4. **每月一刷**：上线一段时间后再做一遍，巩固长期记忆

---

## 🆕 2026 版补充题（Go 1.23–1.26 新特性）

> 检测你是否掌握课程"📅 2026 更新"小节。每题答案附在题目末尾。

**Q1**：`for v := range f` 中 `f` 必须是什么签名？哪个 Go 版本起原生支持？
<details><summary>答</summary>`func(yield func() bool)` / `func(yield func(V) bool)` / `func(yield func(K, V) bool)`，**Go 1.23** 起原生支持。`iter.Seq[V]` 与 `iter.Seq2[K, V]` 是别名。</details>

**Q2**：Go 1.24 内置 map 实现换成了什么？写代码要改吗？
<details><summary>答</summary>**Swiss Tables**。API/语义完全兼容，业务代码不用改。但 unsafe 直接读 hmap 字段的代码会爆。可用 `GOEXPERIMENT=noswissmap` 退回旧实现做对比。</details>

**Q3**：在容器里跑 Go，CPU 限制是 1.5 核。Go 1.25 的默认 GOMAXPROCS 是几？
<details><summary>答</summary>**2**（向上取整）。Go 1.25 默认从 cgroup 读 quota / period，并定期重检。可用 `GODEBUG=containermaxprocs=0` 退回旧行为；`updatemaxprocs=0` 关掉周期性更新。</details>

**Q4**：`sync.WaitGroup.Go(f)` 等价于哪几行？哪个 Go 版本起加入？
<details><summary>答</summary>`wg.Add(1); go func() { defer wg.Done(); f() }()`。**Go 1.25** 加入。注意：很多旧资料把它误标为 1.20。</details>

**Q5**：`testing/synctest` 在测试里替代了什么？什么时候不能用？
<details><summary>答</summary>替代了"sleep + fake clock"两套老做法：业务代码继续用 `time.Now/Sleep`，测试侧用 `synctest.Test` 隔离 + `synctest.Wait` 同步。涉及真实 I/O（网络、磁盘）的 goroutine 永远不会"全部阻塞"——这类场景不适合 synctest。Go 1.24 实验、**Go 1.25 稳定**。</details>

**Q6**：Go 1.26 默认开启的 Green Tea GC 解决了什么核心问题？带来了什么代价？
<details><summary>答</summary>**解决**：经典三色标记按对象做指针追踪 → cache miss 多 + 多 worker 同步开销大。Green Tea 改为按 8 KiB span 批量扫描小对象，用 SIMD 加速。**代价**：基线 RSS 上涨 8-15%（autoscaler 阈值要调）。GC overhead 减 10-40%。</details>

**Q7**：写一段最短代码，用 Go 1.25 内置 CSRF 防护包住一个 mux。
<details><summary>答</summary>

```go
mux := http.NewServeMux()
mux.HandleFunc("POST /transfer", handler)
p := http.NewCrossOriginProtection()
http.ListenAndServe(":8080", p.Handler(mux))
```
基于 `Sec-Fetch-Site` + `Origin`/`Host` 比对。
</details>

**Q8**：`reflect.TypeAssert[T](v)` 比 `v.Interface().(T)` 强在哪？
<details><summary>答</summary>不需要装箱（没有中间 `interface{}` 分配）；直接返回 `(T, bool)`，配合泛型代码更短。**Go 1.25** 加入。</details>

**Q9**：Go 1.24 的 `tool` 指令解决了什么"老 hack"？怎么用？
<details><summary>答</summary>解决了 `tools.go` + `_ "..."` 黑魔法。在 go.mod 里写 `tool github.com/sqlc-dev/sqlc/cmd/sqlc`，CI 用 `go tool sqlc ...` 跑。版本钉在 go.mod，不再 `go install foo@latest` 拉非确定版本。</details>

**Q10**：Go 1.26 CGO 调用开销降了多少？背后做了什么？
<details><summary>答</summary>约 **30%**。精简了 syscall.cgocall 路径上的栈管理与 P 解绑/再绑。每秒上万次小 cgo 调用的场景（SQLite、crypto/openssl 桥）受益最大。</details>

按对答案给自己打分。错 ≥ 4 题：把 INDEX.md 的"2026 新特性快速索引"对照看一遍。

---

> 📁 本测验位于 `/data/workspace/dp4/golang/QUIZ.md`
> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [ROADMAP.md](./ROADMAP.md) 路线图
