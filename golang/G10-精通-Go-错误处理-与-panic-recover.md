# 精通 Go 错误处理与 panic/recover

> 课程编号：G10
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Error Handling 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：err != nil 真的够吗

```go
file, err := os.Open(path)
if err != nil { return err }
```

这是无数 Go 教程的开场白。但当 `err` 经过 10 层调用、5 个不同的库、嵌套在批量任务中时，"err != nil"远远不够：你需要知道**是什么**错（io.EOF？某种业务错？）、**在哪一层**错（数据库还是缓存？）、**带什么上下文**（哪个 user_id？哪次重试？）。本章从 `error` 接口拆到 `errors.Join`，再到 panic/recover 的边界。

---

## 第一章：error 接口的本质

### 1.1 一个方法

```go
type error interface {
    Error() string
}
```

仅此而已。任何实现 `Error() string` 的类型都是 error。这种极简设计带来巨大灵活性：

```go
type MyError struct {
    Code int
    Msg  string
}
func (e *MyError) Error() string {
    return fmt.Sprintf("[%d] %s", e.Code, e.Msg)
}
```

### 1.2 errors.New 与 fmt.Errorf

```go
err1 := errors.New("not found")
err2 := fmt.Errorf("user %d not found", uid)
```

底层都是 `errorString{s string}`，仅在 `Error()` 时返回 s。轻量、几乎零开销。

### 1.3 自定义错误类型的两种风格

**Sentinel（哨兵）值**：

```go
var ErrNotFound = errors.New("not found")
var ErrInvalid  = errors.New("invalid")

if errors.Is(err, ErrNotFound) { ... }
```

适合**全局通用错误**，调用方需要精确判等。

**自定义类型**：

```go
type ValidationError struct {
    Field string
    Msg   string
}
func (e *ValidationError) Error() string { return e.Field + ": " + e.Msg }

var ve *ValidationError
if errors.As(err, &ve) {
    fmt.Println(ve.Field)   // 提取字段
}
```

适合**携带额外信息**的错误。

---

## 第二章：Go 1.13+ 的 wrapping

### 2.1 %w 包装

```go
if err := readConfig(); err != nil {
    return fmt.Errorf("init: %w", err)
}
```

`%w` 把 `err` 包成新错误，形成**错误链**。调用方可以：

- `errors.Is(err, target)` ：链中是否有等于 target
- `errors.As(err, &x)` ：链中是否有可赋值给 x 的类型
- `errors.Unwrap(err)` ：剥一层

`%v` 与 `%w` 的关键差别：`%v` 仅作字符串拼接，丢失链；`%w` 保留链。

### 2.2 检查链

```go
var ErrNotFound = errors.New("not found")

func loadUser(id int) error {
    return fmt.Errorf("load user %d: %w", id, ErrNotFound)
}

err := loadUser(42)
fmt.Println(errors.Is(err, ErrNotFound))   // true
```

`errors.Is` 沿链上溯比较，命中即返回 true。

### 2.3 提取自定义类型

```go
type DBError struct{ Query string }
func (e *DBError) Error() string { return "db error: " + e.Query }

func saveUser(u User) error {
    return fmt.Errorf("save: %w", &DBError{Query: "INSERT ..."})
}

err := saveUser(u)
var dbe *DBError
if errors.As(err, &dbe) {
    fmt.Println(dbe.Query)   // 提取查询语句
}
```

`errors.As` 沿链找第一个能赋值给 `*dbe` 的错误。注意 `&dbe`——传指针的指针。

### 2.4 实现 Is/As 接口

如果你希望两个不同的错误值在 `Is` 下被视为相等，可以实现：

```go
func (e *MyError) Is(target error) bool {
    t, ok := target.(*MyError)
    if !ok { return false }
    return e.Code == t.Code   // 只比较 code，不比较 Msg
}
```

`errors.Is` 沿链调用每个 error 的 `Is(target)` 方法。

---

## 第三章：sentinel errors 的使用

### 3.1 标准库内置

| Sentinel | 含义 |
|---|---|
| `io.EOF` | 读到流末尾 |
| `io.ErrUnexpectedEOF` | 读到不完整数据 |
| `sql.ErrNoRows` | 查询无结果 |
| `os.ErrNotExist` | 文件不存在 |
| `context.Canceled` / `context.DeadlineExceeded` | 上下文取消/超时 |

### 3.2 比较方式

```go
// ❌ 旧风格
if err == io.EOF { ... }

// ✅ 新风格（兼容 wrapping）
if errors.Is(err, io.EOF) { ... }
```

旧风格在 `err` 被 wrap 后会失败。新风格一律安全。

### 3.3 何时设计 sentinel

- 错误状态**有限且稳定**（如 EOF）
- 调用方需要**精确判等**
- 不携带额外信息

如果错误"种类"多于 5 个，考虑自定义 type 而非堆砌 sentinel。

---

## 第四章：errors.Join（Go 1.20+）

### 4.1 合并多个错误

```go
func validate(u User) error {
    var errs []error
    if u.Name == ""  { errs = append(errs, errors.New("name required")) }
    if u.Age < 0     { errs = append(errs, errors.New("age invalid")) }
    if u.Email == "" { errs = append(errs, errors.New("email required")) }
    return errors.Join(errs...)
}

err := validate(u)
fmt.Println(err)
// name required
// age invalid
// email required
```

`errors.Join(e1, e2)` 返回一个 multiError；nil 错误自动过滤。

### 4.2 与 Is/As 兼容

```go
err := errors.Join(io.EOF, errors.New("other"))
errors.Is(err, io.EOF)   // true，沿所有子错误检查
```

### 4.3 何时用

- 批量验证
- 并发 goroutine 错误汇总
- 多步骤事务回滚

```go
var wg sync.WaitGroup
errCh := make(chan error, n)
for i := 0; i < n; i++ {
    wg.Add(1)
    go func(i int) {
        defer wg.Done()
        if err := work(i); err != nil { errCh <- err }
    }(i)
}
wg.Wait(); close(errCh)

var all []error
for e := range errCh { all = append(all, e) }
return errors.Join(all...)
```

---

## 第五章：panic 的语义

### 5.1 触发方式

```go
panic("something terrible")
panic(errors.New("with error"))
panic(&MyPanic{Code: 42})
```

panic 可以接受任意值（不限于 error）。但 recover 返回 `any`，多数情况下你想要的是 error。

### 5.2 runtime panic

```go
var p *int
*p = 1   // panic: runtime error: invalid memory address

a := []int{1, 2, 3}
_ = a[10]   // panic: index out of range

var m map[int]int
m[1] = 1    // panic: assignment to entry in nil map
```

这些 panic **可以** recover。但下面这些**不可** recover：

- `concurrent map writes`
- `out of memory`
- 栈溢出
- C 代码 segfault（CGO）

### 5.3 panic 与 defer

panic 后**立刻停止**当前函数，按 LIFO 顺序执行 defer。每个 defer 都会执行，即使中间还有 panic（后一个 panic 替代前一个）。

```go
func f() {
    defer fmt.Println("d1")
    defer fmt.Println("d2")
    panic("oops")
}
// 输出:
// d2
// d1
// panic: oops
// goroutine ... [running]:
// ...
```

### 5.4 recover 的范围

```go
func main() {
    defer recover()   // ❌ 没用，因为 recover 不在 defer 函数里
    panic("oops")
}
```

必须：

```go
defer func() {
    if r := recover(); r != nil {
        log.Println("recovered:", r)
    }
}()
```

或封装成工具：

```go
func Recover(fn func() error) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    return fn()
}
```

---

## 第六章：goroutine 中的 panic

### 6.1 不会被父 goroutine 捕获

```go
defer func() { recover() }()   // 当前 goroutine

go func() {
    panic("oops")   // 整个程序崩溃，父的 recover 无效
}()
time.Sleep(time.Second)
```

每个 goroutine 自己负责 recover。

### 6.2 worker pool 必备 recover

```go
func runWorker(jobs <-chan Job) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker recovered: %v\n%s", r, debug.Stack())
        }
    }()
    for j := range jobs {
        j.Do()
    }
}

for i := 0; i < 4; i++ {
    go runWorker(jobs)
}
```

否则一个 bug 就可能让整个服务崩溃。

### 6.3 HTTP server

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    panic("oops")
})
```

标准 `net/http` 内置每请求 recover，不会因为单个 handler panic 杀掉 server。但**仍会**断开当前连接（除非 handler 自己处理）。

```go
func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v\n%s", rec, debug.Stack())
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

显式中间件可以保证 500 响应、记录 stack trace。

---

## 第七章：错误处理设计模式

### 7.1 三层错误

```
具体 error  →  业务 error  →  HTTP 响应
   |              |              |
sql.ErrNoRows  →  ErrNotFound  →  404
*MyDBError     →  ErrInternal →  500
```

底层只返回技术错误；中间层翻译为业务错误；顶层映射到协议响应。

### 7.2 错误附加上下文

```go
// ❌ 丢失信息
if err != nil { return errors.New("failed") }

// ✅ 包装
if err != nil { return fmt.Errorf("query user %d: %w", id, err) }
```

包装时加**最有价值的上下文**——通常是参数和操作名。

### 7.3 在 API 边界翻译

```go
// 内部
type Service struct{}

var ErrUserNotFound = errors.New("user not found")

func (s *Service) GetUser(id int) (*User, error) {
    row := s.db.QueryRow("SELECT ...")
    var u User
    if err := row.Scan(...); err != nil {
        if errors.Is(err, sql.ErrNoRows) { return nil, ErrUserNotFound }
        return nil, fmt.Errorf("scan: %w", err)
    }
    return &u, nil
}

// HTTP handler
u, err := svc.GetUser(id)
if err != nil {
    if errors.Is(err, ErrUserNotFound) {
        http.Error(w, "Not Found", 404)
        return
    }
    log.Println(err)
    http.Error(w, "Internal", 500)
    return
}
```

### 7.4 不要忽视 error

```go
// ❌
_, _ = f.Write(data)

// ✅ 至少 log
if _, err := f.Write(data); err != nil {
    log.Printf("write failed: %v", err)
}
```

`errcheck` lint 工具会标出所有被忽略的 error。

---

## 第八章：生产级最佳实践

1. **error 永远是最后一个返回值**：Go 约定。
2. **包装错误用 `%w`**，**渲染用 `%v`**。
3. **检查 sentinel 用 `errors.Is`**，**提取类型用 `errors.As`**。
4. **每一层包装都附加上下文**：参数、阶段名、ID。
5. **不要返回 `nil, nil`**：成功带 result，失败带 error，二选一。
6. **不在库内 panic**：只用 error。
7. **每个 goroutine 加 recover**：尤其是长生命周期 worker。
8. **HTTP 中间件 recover + log stack**：保护服务可用性。
9. **避免 sentinel 滥用**：超过 5 个考虑 type。
10. **使用 `errcheck` 静态检查**：消灭被忽略的 error。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：%v 替代 %w
```go
return fmt.Errorf("failed: %v", err)   // 链断了
```

### ❌ 陷阱 2：== 比较 wrapped error
```go
err := fmt.Errorf("wrap: %w", io.EOF)
if err == io.EOF { ... }   // false ！
```
用 `errors.Is`。

### ❌ 陷阱 3：返回具体类型 nil
```go
func find() error {
    var e *MyErr
    return e   // 非 nil interface
}
```

### ❌ 陷阱 4：recover 后假装没事
```go
defer func() { recover() }()
// 程序状态可能已被破坏，应至少 log
```

### ❌ 陷阱 5：在 init 用 panic 拒绝启动是合理的
但要确保信息足够定位（带配置路径、缺失字段等）。

### ❌ 陷阱 6：goroutine 内 panic 不 recover
直接杀进程。永远在 goroutine 顶部 defer recover。

### ❌ 陷阱 7：把所有错误都包装一遍
```go
if err != nil { return fmt.Errorf("%w", err) }   // 等价 return err
```
没意义。仅在加上下文时包装。

---

## 第十章：练习题

**练习 1**：以下输出？
```go
var ErrA = errors.New("A")
err := fmt.Errorf("wrap: %w", ErrA)
fmt.Println(err == ErrA)
fmt.Println(errors.Is(err, ErrA))
```

**练习 2**：写一个 `Retry(fn func() error, attempts int) error`，捕获 fn 中的 panic 并转为错误。

**练习 3**：实现 `errors.Is` 的简化版（不考虑 wrapping 内部用 Join 的情况）。

**练习 4**：以下代码在 goroutine 中 panic，主程序如何安全继续？
```go
go doWork()
```

**练习 5**：解释为什么 `errors.As(err, &target)` 要传 `&target` 而不是 `target`。

---

## 参考答案

**练习 1**：
```
false   // 直接 == 比较，wrap 后的 err 是新对象
true    // errors.Is 沿链上溯
```

**练习 2**：
```go
func Retry(fn func() error, attempts int) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    for i := 0; i < attempts; i++ {
        if err = fn(); err == nil { return nil }
    }
    return err
}
```

**练习 3**：
```go
func Is(err, target error) bool {
    for err != nil {
        if err == target { return true }
        if x, ok := err.(interface{ Is(error) bool }); ok && x.Is(target) {
            return true
        }
        err = errors.Unwrap(err)
    }
    return false
}
```

**练习 4**：在 `doWork` 内部 defer recover，或包一层：
```go
func safe(fn func()) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("recovered: %v", r)
        }
    }()
    fn()
}
go safe(doWork)
```

**练习 5**：`errors.As(err, &target)` 通过反射要把找到的具体错误**赋值给** target 变量，要写入 target 需要它的地址。直接传 target 只能读，无法写。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| error 接口 | 单方法 `Error() string`；任何类型可实现 |
| sentinel | 全局变量；errors.Is 比较 |
| 自定义类型 | 携带上下文；errors.As 提取 |
| %w | 包装错误；保留链 |
| errors.Join | Go 1.20+；合并多个错误 |
| panic | 编程错误用；库不要 panic |
| recover | 必须 defer 内；只对当前 goroutine |
| 设计模式 | 翻译错误、附加上下文、API 边界归一 |

下一篇 **G11 — 精通 Goroutines 与 GMP 调度** 会拆开 Go 调度器的 G/M/P 模型、抢占机制、栈伸缩、syscall 与 P 解绑等运行时核心。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G10-精通-Go-错误处理-与-panic-recover.md`
