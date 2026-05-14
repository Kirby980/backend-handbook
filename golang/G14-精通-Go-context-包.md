# 精通 Go context 包

> 课程编号：G14
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — context 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：为什么 context 无处不在

```go
func (s *Service) GetUser(ctx context.Context, id int) (*User, error) {
    return s.repo.GetUser(ctx, id)
}

func (r *Repo) GetUser(ctx context.Context, id int) (*User, error) {
    row := r.db.QueryRowContext(ctx, "SELECT ... WHERE id=?", id)
    // ...
}
```

每一个公开函数都带 `ctx context.Context` 做首参——这是现代 Go 的"约定"。但 context 究竟解决什么问题？它的内部结构如何？什么不该放进 context？这一章把 context 包从语义到实现拆开看。

---

## 第一章：context 解决的核心问题

### 1.1 三个职责

1. **取消传播**（cancellation）：上游请求取消时，下游所有衍生操作（DB 查询、HTTP 调用、worker goroutine）一并停止。
2. **超时控制**（deadline）：整条请求链共享一个截止时间，避免下游单独计时。
3. **请求级值传递**（value）：traceID、userID 等跨函数边界传递。

### 1.2 为什么不用全局变量 / channel

- 全局变量：无法区分不同请求
- channel：每个调用层手动 `select` 监听，模板代码爆炸
- context：标准化、有树形语义、与标准库深度集成

### 1.3 接口签名

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

四个方法对应四种语义。

---

## 第二章：context 的树形结构

### 2.1 根节点

```go
ctx := context.Background()   // 空上下文，永不取消
ctx := context.TODO()         // 等同 Background，但表示"暂未决定"
```

Background 用于 main、初始化、tests；TODO 用于"我知道这里需要 context 但还没设计好"。

### 2.2 衍生节点

```go
ctx, cancel := context.WithCancel(parent)
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
ctx        := context.WithValue(parent, key, value)
```

每个衍生 context **持有 parent 引用**，构成一棵树（实际是有向无环图）。

### 2.3 取消传播

```go
parent
  ├─ child1
  │    ├─ grandchild1.1
  │    └─ grandchild1.2
  └─ child2
```

`parent.cancel()` → 沿树向下广播，所有子节点的 `Done()` channel 同时关闭，`Err()` 返回 `context.Canceled`。

### 2.4 实现机制

每个 cancelCtx 有：

```go
type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     atomic.Value   // chan struct{}
    children map[canceler]struct{}
    err      error
}
```

`cancel()` 关闭 `done` channel，遍历 children 递归 cancel。

---

## 第三章：WithCancel

### 3.1 基础

```go
ctx, cancel := context.WithCancel(parent)
defer cancel()   // 必须！否则 child 永远不释放

go work(ctx)

if condition {
    cancel()   // 通知 work 停止
}
```

`cancel` 函数关闭 ctx 的 done channel。

### 3.2 必须调用 cancel

即使函数已正常返回，也要 `defer cancel()`。原因：父节点维护 children 集合，child 不被 cancel 就一直留在 map 中——泄漏。

Go vet 会警告：

```
the cancel function returned by context.WithCancel should be called, not discarded, to avoid a context leak
```

### 3.3 work 函数的标准模式

```go
func work(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case x := <-jobs:
            process(ctx, x)
        }
    }
}
```

每个长循环都要 `select` 监听 `Done`。

---

## 第四章：WithTimeout & WithDeadline

### 4.1 区别

```go
WithTimeout(parent, 5*time.Second)
// 等价
WithDeadline(parent, time.Now().Add(5*time.Second))
```

Timeout 是"从现在起 N 秒后"；Deadline 是"在某绝对时刻"。

### 4.2 整链共享 deadline

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

callA(ctx)   // 内部又 WithTimeout(ctx, 10s) 也只 5s（取较短）
```

衍生子节点的 deadline 不能晚于父节点。

### 4.3 检查 deadline 剩余

```go
if d, ok := ctx.Deadline(); ok {
    remaining := time.Until(d)
    // 根据 remaining 决定要不要继续
}
```

### 4.4 触发后的 Err

```go
<-ctx.Done()
err := ctx.Err()
// errors.Is(err, context.Canceled)         // WithCancel 触发
// errors.Is(err, context.DeadlineExceeded) // 超时触发
```

---

## 第五章：WithValue

### 5.1 基础

```go
type key int
const userIDKey key = 0

ctx = context.WithValue(ctx, userIDKey, 42)

uid, _ := ctx.Value(userIDKey).(int)
```

### 5.2 key 必须是导出的私有类型

**反模式**：

```go
ctx.Value("user_id")   // ❌ 字符串 key 冲突风险
```

**正确**：

```go
type ctxKey int   // 包私有类型
const userIDKey ctxKey = iota

// 别的包想读？提供 helper
func UserIDFrom(ctx context.Context) (int, bool) {
    uid, ok := ctx.Value(userIDKey).(int)
    return uid, ok
}
```

不同包的 ctxKey 类型不同 → 哪怕值相同也是不同 key。安全。

### 5.3 查找路径

`ctx.Value(k)` 沿父链向上查找，第一个匹配返回。线性时间 O(depth)。所以**不要把热点字段塞 context**——每次访问遍历整条链。

### 5.4 什么应该 / 不应该放进 context

**应该**：
- traceID、requestID
- 认证身份（userID、tenantID）
- 客户端语言、地区
- 日志字段（按惯例）

**不应该**：
- 数据库连接、客户端实例 → 用依赖注入
- 配置 → 用 struct 字段
- 业务参数（如 user 对象、订单数据） → 显式函数参数
- 大对象 → 性能、内存

**判断准则**：能放进**函数签名**的就别放 context。

---

## 第六章：传播规则

### 6.1 接受 + 传递

```go
func foo(ctx context.Context) error {
    // ...
    return bar(ctx)   // 透传
}
```

每一层都接 ctx 做首参（不是命名 c、context 等，**统一叫 ctx**）。

### 6.2 不要存储 context

```go
type Service struct {
    ctx context.Context   // ❌ 反模式
}
```

context 的生命周期 = 请求；存到长生命周期对象会导致：
- 误用过期的 ctx
- 难以推理
- 容易内存泄漏

例外：测试 helper 暂存 ctx 用于 cleanup（一般也避免）。

### 6.3 ctx 第一参数

```go
// ✅
func F(ctx context.Context, a, b int) error

// ❌
func F(a int, ctx context.Context, b int) error
```

Go 团队约定：ctx **永远是第一个参数**。

### 6.4 不要传 nil context

```go
foo(nil)   // ❌ 会 panic
foo(context.TODO())   // ✅
```

context.TODO() 是"我还没想好"的占位符——比 nil 安全。

---

## 第七章：标准库集成

### 7.1 database/sql

```go
db.QueryContext(ctx, "SELECT ...")
db.ExecContext(ctx, "UPDATE ...")
```

ctx 取消时，正在执行的查询被尝试 abort（驱动支持）。

### 7.2 net/http

```go
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
client.Do(req)
```

服务端：`req.Context()` 返回连接级 context，客户端断开时自动 cancel。

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    // ...
    select {
    case <-ctx.Done():
        return   // 客户端断开
    case v := <-work:
        w.Write(v)
    }
}
```

### 7.3 grpc-go

每个 RPC 自动带 ctx，timeout 通过 metadata 传播到服务端。

### 7.4 工具库

`golang.org/x/sync/errgroup`：

```go
g, ctx := errgroup.WithContext(parent)
for _, url := range urls {
    url := url
    g.Go(func() error { return fetch(ctx, url) })
}
if err := g.Wait(); err != nil { ... }
```

任一 goroutine 返回错则 ctx 自动 cancel，其他 goroutine 收到 Done。

---

## 第八章：常见模式

### 8.1 HTTP 中间件链

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        user, err := authenticate(r)
        if err != nil { http.Error(w, "...", 401); return }
        ctx := context.WithValue(r.Context(), userKey, user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 8.2 数据库事务超时

```go
func withTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }

    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    return tx.Commit()
}
```

### 8.3 worker pool 优雅退出

```go
ctx, cancel := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer cancel()

go worker(ctx)
go worker(ctx)

<-ctx.Done()   // SIGINT/SIGTERM 触发
// 等 worker 完成 cleanup
```

`signal.NotifyContext`（Go 1.16+）是优雅退出的标准入口。

### 8.4 trace 信息透传

```go
type traceKey int
const traceIDKey traceKey = 0

func WithTraceID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, traceIDKey, id)
}
func TraceID(ctx context.Context) string {
    if id, ok := ctx.Value(traceIDKey).(string); ok { return id }
    return ""
}
```

---

## 第九章：性能考量

### 9.1 context 创建成本

```go
ctx, cancel := context.WithCancel(parent)
```

约 200ns（含一次小 alloc）。在热点路径每次创建可见——但通常 context 在请求边界创建，热点路径只是接收。

### 9.2 Value 查找成本

每次 `ctx.Value(k)` 沿链 O(depth)。如果 context 链有 10 层、热点函数每次调用查 3 次值，每次访问 ~30ns × 3 = 90ns。

**优化**：在请求入口提取值到局部变量，避免重复查找。

### 9.3 Done() channel

`ctx.Done()` 返回的是同一个 channel（缓存），多次调用零成本。

---

## 第十章：生产级最佳实践

1. **每个公开函数首参 `ctx context.Context`**：从入口直贯到底。
2. **永远 `defer cancel()`**：哪怕看起来没必要。
3. **不要把 context 存入 struct**：用参数传递。
4. **WithValue 的 key 用包私有类型**：避免冲突。
5. **不要传 nil**：用 `context.TODO()` 或 `context.Background()`。
6. **服务端首先把 r.Context() 拿到手**：客户端断开自动通知。
7. **每个长循环 select Done**：避免泄漏。
8. **errgroup 用于"并发 + 任一失败取消"**：减少手动协调。
9. **`signal.NotifyContext`**：信号驱动优雅退出。
10. **不要把请求数据塞 context**：用参数或 struct。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：cancel 未调用
```go
ctx, _ := context.WithCancel(parent)   // _ 丢了 cancel
// parent 在树中持有 ctx，永不释放
```

### ❌ 陷阱 2：把 context 存入结构体
```go
type Worker struct{ ctx context.Context }
```

### ❌ 陷阱 3：字符串 key
```go
ctx = context.WithValue(ctx, "uid", 42)
```

### ❌ 陷阱 4：忽视 ctx.Err
```go
<-ctx.Done()
// 不知道是 timeout 还是 cancel
```
应该检查 `ctx.Err()` 区分。

### ❌ 陷阱 5：把 *http.Request 放 context
```go
ctx = context.WithValue(ctx, reqKey, r)
```
request 已经持有 context，循环引用风险。

### ❌ 陷阱 6：传 nil
```go
F(nil)   // panic
```

### ❌ 陷阱 7：在 Goroutine 内修改父 context
```go
go func() {
    ctx = context.WithValue(ctx, k, v)   // 修改的是局部副本
}()
```
context 是不可变的——WithValue 返回**新** context，老的不变。

---

## 第十二章：练习题

**练习 1**：以下代码是否泄漏？为什么？
```go
func work() {
    ctx, _ := context.WithCancel(context.Background())
    go func() { <-ctx.Done(); fmt.Println("done") }()
}
```

**练习 2**：写一个带 timeout 的 `Fetch(ctx context.Context, url string) ([]byte, error)`，使用 http.Client。

**练习 3**：以下为什么不好？给出修正：
```go
func saveUser(ctx context.Context, db *sql.DB) error {
    user := ctx.Value("user").(*User)
    return db.Exec("INSERT ...", user.Name).Err()
}
```

**练习 4**：实现一个 "WithDeadlineOrDefault"：如果 parent 没 deadline，用 default；否则取较短。

**练习 5**：解释为什么 `context.Background().Done()` 返回 nil。

---

## 参考答案

**练习 1**：泄漏。`_, cancel` 丢了；ctx 永不取消，goroutine 永远等。父节点（Background）虽然不维护 children，但你的 ctx 自己持有 children map，goroutine 引用 ctx → ctx 不会被 GC → goroutine 永远阻塞。修正：`defer cancel()`。

**练习 2**：
```go
func Fetch(ctx context.Context, url string) ([]byte, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil { return nil, err }
    resp, err := http.DefaultClient.Do(req)
    if err != nil { return nil, err }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}
```

**练习 3**：① 用字符串 key 容易冲突；② 把 user 这种业务数据塞 context；③ 类型断言不检查 ok 可能 panic。修正：
```go
func saveUser(ctx context.Context, db *sql.DB, user *User) error {
    _, err := db.ExecContext(ctx, "INSERT ...", user.Name)
    return err
}
```

**练习 4**：
```go
func WithDeadlineOrDefault(parent context.Context, dflt time.Duration) (context.Context, context.CancelFunc) {
    if _, ok := parent.Deadline(); ok {
        return context.WithCancel(parent)   // 沿用 parent
    }
    return context.WithTimeout(parent, dflt)
}
```

**练习 5**：`Background` 永不取消，所以 `Done()` 设计为返回 nil（永不关闭的语义可以用 nil chan 表达——recv nil chan 永久阻塞，效果一致）。这样在 `select` 里 case `<-ctx.Done()` 行为正确（永不触发）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| context 接口 | Deadline / Done / Err / Value |
| 根节点 | Background, TODO |
| WithCancel | 必须 defer cancel |
| WithTimeout | 整链共享 deadline |
| WithValue | key 用包私有类型；O(depth) 查找 |
| 传播规则 | ctx 是首参；不存 struct；不传 nil |
| 标准库集成 | sql/http/grpc 都接 ctx |
| signal.NotifyContext | 优雅退出标准入口 |

下一篇 **G15 — 精通 Go 并发模式** 将基于前 4 章基础，把 fan-in、fan-out、pipeline、worker pool、pub/sub、限流、广播等模式整理成 cookbook。

---

