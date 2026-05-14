# 精通 Go 函数、闭包与 defer 机制

> 课程编号：G06
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Functions 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：一段 defer 谜题

```go
package main

import "fmt"

func mystery() (result int) {
    defer func() { result *= 2 }()
    defer fmt.Println("A:", result)
    result = 10
    defer fmt.Println("B:", result)
    return 5
}

func main() {
    fmt.Println("ret:", mystery())
}
```

输出是什么？为什么 defer A 打印 0 而 B 打印 10？最终返回 5×2 还是 10×2？这一章把函数值、闭包捕获、defer LIFO、defer 与命名返回值的协作、Go 1.14 的"开放编码 defer"零开销优化，一并讲清。

---

## 第一章：函数是一等公民

### 1.1 函数类型

```go
type BinOp func(int, int) int

var add BinOp = func(a, b int) int { return a + b }

ops := map[string]BinOp{
    "+": func(a, b int) int { return a + b },
    "-": func(a, b int) int { return a - b },
}

result := ops["+"](3, 4)
```

函数类型由参数类型列表和返回值列表决定，命名参数不参与类型识别——`func(a int) int` 与 `func(x int) int` 是同一类型。

### 1.2 多返回值

```go
func divmod(a, b int) (int, int) {
    return a / b, a % b
}
q, r := divmod(17, 5)
```

多返回值是 Go 的标志。最常见的两元组是 `(value, error)`。

### 1.3 命名返回值

```go
func parse(s string) (n int, err error) {
    n, err = strconv.Atoi(s)
    return   // 裸 return：使用当前命名返回值
}
```

命名返回值的好处：
- 可作为文档（参数说明）
- 在函数顶部声明，零值初始化
- 配合 defer 修改返回值

坏处：
- 容易和短变量声明（`:=`）冲突 → shadow bug（参考 G01）
- 裸 `return` 在长函数中可读性差

**建议**：函数 < 10 行可以裸 return；更长的函数显式 `return n, err`。

### 1.4 可变参数 ...T

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)        // 1+2+3
nums := []int{1, 2, 3}
sum(nums...)        // 等价于 sum(1, 2, 3)
```

`nums ...int` 在函数内是 `[]int`。**底层就是 slice**，没有特殊魔法。`sum(slice...)` 是 spread 语法——直接把已有 slice 当作可变参数传入。

注意：spread 时**不会复制底层数组**，函数内修改 `nums[i]` 会改原 slice。

---

## 第二章：闭包

### 2.1 闭包定义

闭包 = 函数 + 它捕获的外部变量。

```go
func counter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

c1 := counter()
fmt.Println(c1(), c1(), c1())   // 1 2 3
c2 := counter()
fmt.Println(c2())               // 1（独立的 n）
```

`n` 是 `counter` 的局部变量，但被返回的闭包"捕获"。闭包持有 `n` 的**引用**（不是值），每次调用都修改同一个 `n`。

### 2.2 闭包让变量逃逸到堆

```go
func counter() func() int {
    n := 0
    return func() int { n++; return n }
}
```

`n` 本应在 `counter` 的栈上。但闭包返回后栈帧销毁，`n` 必须移到堆。这是 Go 编译器**自动逃逸分析**的典型场景。

```bash
go build -gcflags="-m" main.go
# 输出：moved to heap: n
```

### 2.3 按引用捕获，不是按值

```go
x := 1
f := func() { fmt.Println(x) }
x = 2
f()   // 2 ！不是 1
```

闭包永远捕获**变量本身**（引用），即使变量在闭包定义后改变。如果想"冻结"当时的值，显式复制：

```go
x := 1
y := x   // 复制
f := func() { fmt.Println(y) }
x = 2
f()   // 1
```

### 2.4 循环变量陷阱（Go 1.22 分水岭，老项目仍要小心）

```go
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs { f() }
```

- **Go 1.21 及之前**：输出 `3 3 3`。所有闭包捕获同一个 `i`，循环结束 `i==3`。
- **Go 1.22 起**（当前所有维护中的版本默认行为，要求 `go.mod` 声明 `go 1.22+`）：输出 `0 1 2`。每次迭代 `i` 是新变量。

**老代码兼容写法**（无关 Go 版本都正确）：
```go
for i := 0; i < 3; i++ {
    i := i   // 局部复制
    funcs = append(funcs, func() { fmt.Println(i) })
}
```

### 2.5 闭包与 goroutine

```go
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()   // Go 1.21 前: 输出不确定，常 3 3 3
}
```

最经典的并发陷阱。Go 1.22 修复了，但仍建议显式传参：

```go
for i := 0; i < 3; i++ {
    go func(i int) { fmt.Println(i) }(i)
}
```

---

## 第三章：defer 的基础

### 3.1 LIFO 顺序

```go
func f() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
// 输出：3 2 1
```

defer 注册到一个**栈**，函数返回时（正常或 panic）按 LIFO 顺序执行。

### 3.2 参数立即求值

```go
i := 1
defer fmt.Println("deferred:", i)
i = 2
fmt.Println("normal:", i)
// 输出：
// normal: 2
// deferred: 1
```

`defer fmt.Println("...", i)` 中 `i` 是参数，**defer 注册时就求值**。如果想用最新值：

```go
defer func() { fmt.Println("deferred:", i) }()
```

匿名函数没参数，捕获的是 `i` 这个变量本身（按引用），所以会读到最新值。

### 3.3 一图看清

| 语法 | 求值时机 |
|---|---|
| `defer foo(a, b)` | `a`, `b` 立即求值；`foo` 调用延迟 |
| `defer func() { ... a, b ... }()` | 整个匿名函数体延迟；a, b 用最终值 |

### 3.4 典型用法

```go
// ① 资源释放
func readFile(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close()
    // ...
}

// ② 锁释放
mu.Lock()
defer mu.Unlock()

// ③ panic 防护
defer func() {
    if r := recover(); r != nil {
        log.Printf("panic: %v", r)
    }
}()

// ④ 计时器
defer func(t time.Time) {
    log.Printf("took %v", time.Since(t))
}(time.Now())
```

---

## 第四章：defer 与命名返回值

### 4.1 修改返回值

```go
func double() (n int) {
    defer func() { n *= 2 }()
    n = 10
    return    // 实际返回 20
}
```

执行顺序：
1. `n = 10`（设置命名返回值）
2. `return` 触发——把 n 赋给"返回插槽"
3. 执行 defer（在 return 完成前），修改 n
4. 真正退出，返回当前 n

只有**命名返回值**能这样被 defer 修改；匿名返回值无法。

### 4.2 panic → error 转换

```go
func safeRun(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    f()
    return nil
}
```

`f()` 内部 panic 时，defer 中 recover 捕获并设置 `err`。调用方拿到 error 而非崩溃。

### 4.3 回到引言谜题

```go
func mystery() (result int) {
    defer func() { result *= 2 }()
    defer fmt.Println("A:", result)
    result = 10
    defer fmt.Println("B:", result)
    return 5
}
```

执行：
1. 注册 defer1 (匿名函数：result *= 2)
2. 注册 defer2 (Println A，**参数立即求值** → "A: 0"）
3. `result = 10`
4. 注册 defer3 (Println B，参数立即求值 → "B: 10")
5. `return 5` → result = 5
6. LIFO 执行：defer3 (打印"B: 10")、defer2 (打印"A: 0")、defer1 (result = 5 * 2 = 10)
7. 真正返回 10

输出：
```
B: 10
A: 0
ret: 10
```

---

## 第五章：defer 的运行时实现

### 5.1 三种实现机制

Go 1.13 之前：所有 defer 都通过 `runtime.deferproc` 注册到 goroutine 的 \_defer 链表，函数返回时 `runtime.deferreturn` 逐个执行。每次注册成本 ~50ns，加上链表分配。

Go 1.13：**栈上分配 deferred record**。常见场景不再走堆，开销减半。

Go 1.14+：**开放编码 defer**（open-coded defer）。当函数的 defer 满足：
- defer 数量 ≤ 8
- 没有出现在循环里
- 编译器能内联展开

编译器**不生成 deferproc 调用**，而是直接在函数末尾插入 defer 代码（带条件 bitmap 控制每个 defer 是否执行）。开销几乎为零。

### 5.2 benchmark 对比

```go
func BenchmarkNoDefer(b *testing.B) {
    var mu sync.Mutex
    for i := 0; i < b.N; i++ {
        mu.Lock()
        mu.Unlock()
    }
}

func BenchmarkDefer(b *testing.B) {
    var mu sync.Mutex
    for i := 0; i < b.N; i++ {
        mu.Lock()
        defer mu.Unlock()
    }
}
```

Go 1.13 之前：defer 版慢 4-5 倍。
Go 1.14+ 开放编码生效：几乎无差距（10% 以内）。

### 5.3 哪些场景仍走慢路径

```go
for i := 0; i < n; i++ {
    defer cleanup(i)   // 循环里：每次都堆分配
}
```

循环内的 defer 不能 open-coded（因为数量不确定，bitmap 不够）。要么把循环体提到独立函数：

```go
for i := 0; i < n; i++ {
    func(i int) {
        defer cleanup(i)
        // ...
    }(i)
}
```

每次迭代是独立函数调用，defer 在迭代结束时执行。这也是修复"循环里 defer 资源不及时释放"的标准手法。

---

## 第六章：panic 与 recover

### 6.1 panic 的传播

panic 后**当前函数立即停止**，依次执行 defer 栈，再传到调用方继续这个过程，直到：
- 某个 defer 中 recover 截住
- 或一路到 goroutine 顶层 → 整个程序崩溃打印 stack trace

### 6.2 recover 必须在 defer 中

```go
defer func() {
    if r := recover(); r != nil {
        log.Println("recovered:", r)
    }
}()
```

直接调用 `recover()` 返回 nil。规范要求 recover 在 deferred 函数中才生效。

### 6.3 recover 只对当前 goroutine

```go
go func() {
    panic("oops")   // 在新 goroutine
}()

defer func() { recover() }()
time.Sleep(time.Second)
// ❌ 整个程序崩溃
```

新 goroutine 的 panic **不会**被父 goroutine 的 defer 捕获。每个 goroutine 必须自己的 recover。

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Println(r)
        }
    }()
    panic("oops")
}()
```

### 6.4 哪些 panic 不能 recover

以下情况整个进程必定崩溃，recover 也救不了：
- `fatal error: concurrent map writes`
- `fatal error: out of memory`
- 栈溢出
- 显式 `runtime.Goexit()` 后还有 panic
- C 代码（CGO）里的 segfault

### 6.5 何时用 panic

- **库的 API 不应该 panic 给调用方**：返回 error。
- **库内部检测到不变式违反**（例如调用方传入明显非法参数）：panic 可以。
- **真正不可恢复的初始化失败**：包级 `var` 初始化或 `init()` 中 panic 阻止程序启动。
- **业务逻辑用 panic 当 throw**：禁止——Go 不是 Java。

---

## 第七章：生产级最佳实践

1. **顶部 `defer f.Close()`**：紧跟在资源获取后注册，最不容易遗漏。
2. **defer 不要在循环里**：要么提到函数，要么用 explicit cleanup。
3. **每个长生命周期 goroutine 都加 recover**：避免一个 bug 把整个进程拖死。
4. **HTTP handler 一定 recover**：标准 `http.Server` 已经内置每请求 recover；自己写 worker pool 别忘了。
5. **闭包跨 goroutine 时显式传参**：`go func(x int) {...}(i)`。
6. **想冻结闭包捕获的值就显式复制**：`x := x` 或 `y := x`。
7. **defer 与命名返回值组合 → panic 转 error / 资源清理 / 计时**。
8. **复杂函数显式 `return a, b`**：避免裸 return 在长函数里降低可读性。
9. **不要用 panic 控制流**：用 error 显式表达失败。
10. **看汇编验证 open-coded defer 是否生效**：`go tool compile -S` 找 `runtime.deferproc` 调用。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：循环里 defer 资源累积
```go
for _, path := range paths {
    f, _ := os.Open(path)
    defer f.Close()   // 全部累积到函数末尾才关
    // ...
}
```

### ❌ 陷阱 2：defer 参数被冻结
```go
i := 1
defer fmt.Println(i)   // 1
i = 2
```

### ❌ 陷阱 3：闭包捕获循环变量（Go 1.21 之前）
```go
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()   // 常 3 3 3
}
```

### ❌ 陷阱 4：recover 在普通函数
```go
func bad() {
    if r := recover(); r != nil { ... }   // 永远 nil
}
```

### ❌ 陷阱 5：闭包共享状态导致竞态
```go
n := 0
for i := 0; i < 10; i++ {
    go func() { n++ }()   // data race
}
```

### ❌ 陷阱 6：defer 释放 nil 资源 panic
```go
f, err := os.Open(path)
defer f.Close()   // 如果 err != nil，f 是 nil，Close 调用本身没事但容易遗忘判断
if err != nil { return err }
```
修正：
```go
f, err := os.Open(path)
if err != nil { return err }
defer f.Close()
```

### ❌ 陷阱 7：defer 改命名返回值出乎意料
```go
func f() (err error) {
    defer func() { err = nil }()   // 任何错误都被吞了
    return errors.New("bad")
}
```

---

## 第九章：练习题

**练习 1**：以下输出？
```go
func f() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
}
```

**练习 2**：以下输出？
```go
func g() (n int) {
    defer func() { n++ }()
    return 10
}
fmt.Println(g())
```

**练习 3**：写一个 `withTimer(name string) func()` 工具，使用方式：
```go
defer withTimer("doWork")()   // 注意末尾 ()
```
打印从调用到 defer 触发的耗时。

**练习 4**：以下函数有什么问题？修复它。
```go
func process(files []string) []*os.File {
    var out []*os.File
    for _, p := range files {
        f, err := os.Open(p)
        if err != nil { continue }
        defer f.Close()
        out = append(out, f)
    }
    return out
}
```

**练习 5**：解释 Go 1.14 的开放编码 defer 在哪些条件下退化。

---

## 参考答案

**练习 1**：`2 1 0`。LIFO 顺序；每次 defer 注册时 `i` 立即求值。

**练习 2**：`11`。`return 10` 设置 n=10，defer 把 n 改成 11，再真正返回。

**练习 3**：
```go
func withTimer(name string) func() {
    start := time.Now()
    return func() {
        log.Printf("%s took %v", name, time.Since(start))
    }
}
```
注意调用时 `defer withTimer("x")()`——外层 `withTimer("x")` 立即执行（启动计时），返回的闭包延迟执行。

**练习 4**：① defer 在函数结束才执行，所有文件没法立刻读取后关闭——会持有大量打开句柄；② 返回 `*os.File` 给调用方，但已注册 defer Close 会过早关闭。修复：把 Close 责任移交调用方。
```go
func process(files []string) ([]*os.File, error) {
    out := make([]*os.File, 0, len(files))
    for _, p := range files {
        f, err := os.Open(p)
        if err != nil {
            for _, x := range out { x.Close() }
            return nil, err
        }
        out = append(out, f)
    }
    return out, nil
}
```

**练习 5**：开放编码 defer 退化到慢路径的条件：
- 函数中 defer 数量 > 8
- defer 出现在 for/while 循环里
- defer 关键字嵌在动态调度的位置（如 switch case 但编译器无法确定）
- 使用 `runtime.Goexit()` 跨 defer

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 函数 | 一等公民；可赋值、可返回、可传参 |
| 命名返回值 | 顶部声明零值；可被 defer 修改；裸 return |
| 闭包 | 按引用捕获；让变量逃逸；循环变量 1.22 修复 |
| defer 顺序 | LIFO，函数返回时执行 |
| defer 求值 | 参数立即求值；匿名函数体延迟执行 |
| Go 1.14 优化 | 开放编码 defer，几乎零开销 |
| recover | 只能在 defer 函数中；只救当前 goroutine |
| panic | 不要当异常用；不可恢复有 fatal、OOM 等 |

下一篇 **G07 — 精通 Go 指针与方法接收者** 将讨论值接收者 vs 指针接收者、可寻址性、空指针调用方法、接口实现的隐式约束。

---

