# 精通 Go 测试：table-driven、subtests、httptest 与 mocks

> 课程编号：G18
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Testing 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：Go 测试的极简哲学

```go
func TestAdd(t *testing.T) {
    if Add(1, 2) != 3 {
        t.Fatal("expected 3")
    }
}
```

不需要 framework、不需要 annotations、不需要 setUp/tearDown 类。`go test` 内置一切。但简单不等于贫瘠——table-driven 模式、subtests、parallel、httptest、t.Cleanup、t.Helper、benchmark 共同构成 Go 测试生态。本章讲怎么用对。

---

## 第一章：测试文件约定

### 1.1 文件命名

```
foo.go         → 实现
foo_test.go    → 测试
```

`_test.go` 后缀的文件：
- 仅在 `go test` 时编译
- 不进发布二进制
- 可以访问同包未导出符号

### 1.2 package 选择

```go
// 同包测试（白盒）
package foo

import "testing"
func TestInternal(t *testing.T) { /* 能调用 foo.unexported */ }
```

```go
// 外部测试包（黑盒）
package foo_test

import (
    "testing"
    "github.com/me/foo"
)
func TestPublicAPI(t *testing.T) { /* 只能访问 foo 的导出符号 */ }
```

经验：默认同包；当测试**只验证公共 API + 想避免循环依赖**时用 `foo_test`。

### 1.3 函数签名

```go
func TestXxx(t *testing.T)    // 测试
func BenchmarkXxx(b *testing.B)   // benchmark
func ExampleXxx()              // 文档示例
func FuzzXxx(f *testing.F)     // fuzz (Go 1.18+)
func TestMain(m *testing.M)    // 全局 setUp/tearDown
```

---

## 第二章：table-driven 测试

### 2.1 标准模式

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 1, 2, 3},
        {"negative", -1, -2, -3},
        {"mixed", 1, -2, -1},
        {"zero", 0, 0, 0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### 2.2 优点

- 添加 case 只需加一行
- `t.Run` 让每个 case 独立运行
- 失败一个不影响其他
- 跑指定 case：`go test -run TestAdd/positive`

### 2.3 错误信息

```go
t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.expected)
```

失败时打印**实际输入和期望/实际值**。让人不需要重看代码就能定位。

### 2.4 复杂结构

```go
tests := []struct {
    name    string
    input   *Foo
    wantErr error
    wantOut *Bar
}{ ... }

for _, tt := range tests {
    // Go 1.22+：循环变量每次迭代是新变量，下面这行不再必要。维护 go.mod 声明 < 1.22 的老项目时仍需 `tt := tt`
    t.Run(tt.name, func(t *testing.T) {
        out, err := Process(tt.input)
        if !errors.Is(err, tt.wantErr) {
            t.Errorf("err = %v, want %v", err, tt.wantErr)
        }
        if !reflect.DeepEqual(out, tt.wantOut) {
            t.Errorf("out = %v, want %v", out, tt.wantOut)
        }
    })
}
```

---

## 第三章：subtests 与 parallel

### 3.1 t.Run

```go
func TestUser(t *testing.T) {
    t.Run("create", func(t *testing.T) { ... })
    t.Run("update", func(t *testing.T) { ... })
    t.Run("delete", func(t *testing.T) { ... })
}
```

`go test -run TestUser/create` 只跑 create。

### 3.2 t.Parallel

```go
func TestSlow(t *testing.T) {
    t.Parallel()   // 标记这个测试可并行
    // ...
}
```

调用后该测试与其他 `t.Parallel()` 测试**并发执行**。`go test -parallel N` 控制并发度（默认 GOMAXPROCS）。

### 3.3 subtests 中的 parallel

```go
for _, tt := range tests {
    // Go 1.22+ 不再需要 `tt := tt`（循环变量已每轮新建），可直接进入 t.Run
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // ...
    })
}
```

每个 subtest 都并行——大幅加速 table-driven。

如果 `go.mod` 仍声明 `go 1.21` 或更早，则保留旧写法 `tt := tt` 防循环变量捕获。

### 3.4 何时不 parallel

- 测试有共享状态（全局变量、文件、网络端口）
- 测试改环境变量
- 测试访问数据库且没事务隔离

---

## 第四章：t.Helper 与 t.Cleanup

### 4.1 t.Helper

```go
func mustParseTime(t *testing.T, s string) time.Time {
    t.Helper()   // 标记这是辅助函数
    tm, err := time.Parse(time.RFC3339, s)
    if err != nil { t.Fatal(err) }
    return tm
}
```

调用 `t.Helper()` 后，测试失败时的报告会**跳过此函数**，指向调用方代码——失败信息更有意义。

### 4.2 t.Cleanup

```go
func setupDB(t *testing.T) *sql.DB {
    db, _ := sql.Open("sqlite3", ":memory:")
    t.Cleanup(func() { db.Close() })
    return db
}
```

类似 defer，但 LIFO 顺序、跨 subtest 自动继承、即使 subtest fail 也执行。比手动 defer 更安全。

### 4.3 t.TempDir

```go
func TestWriteFile(t *testing.T) {
    dir := t.TempDir()   // 自动 cleanup
    path := filepath.Join(dir, "file.txt")
    // ...
}
```

每个测试一个临时目录，结束自动删除。

### 4.4 t.Setenv (Go 1.17+)

```go
func TestConfig(t *testing.T) {
    t.Setenv("API_KEY", "test123")
    // 测试运行中环境变量是 test123
    // 测试结束自动还原
}
```

不能与 t.Parallel 同用（环境变量是进程级共享）。

---

## 第五章：httptest

### 5.1 测试 HTTP handler

```go
func TestHelloHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/hello?name=Alice", nil)
    rec := httptest.NewRecorder()

    helloHandler(rec, req)

    resp := rec.Result()
    if resp.StatusCode != 200 { t.Errorf("status = %d", resp.StatusCode) }
    body, _ := io.ReadAll(resp.Body)
    if !strings.Contains(string(body), "Alice") {
        t.Errorf("body = %s", body)
    }
}
```

`httptest.NewRequest` + `httptest.NewRecorder` 不启动真实 server，直接调用 handler。

### 5.2 模拟整个 server

```go
func TestClient(t *testing.T) {
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/expected" {
            http.Error(w, "bad path", 400); return
        }
        w.Write([]byte(`{"ok":true}`))
    }))
    defer srv.Close()

    client := NewClient(srv.URL)
    result, err := client.Fetch()
    // 断言
}
```

`httptest.NewServer` 启动真实 HTTP server，返回真实 URL。测试 client 端代码用。

### 5.3 TLS server

```go
srv := httptest.NewTLSServer(handler)
// srv.Client() 返回内置信任该证书的 http.Client
```

---

## 第六章：mocks vs fakes vs stubs

### 6.1 三种 test double

| 名字 | 用途 |
|---|---|
| **stub** | 返回固定数据 |
| **fake** | 实现逻辑但用内存或简化版（如 fake DB） |
| **mock** | 记录调用，验证调用次数/参数 |

### 6.2 Go 风格：直接写 fake

```go
type UserStore interface {
    Get(id int) (*User, error)
}

type FakeStore struct {
    users map[int]*User
}
func (s *FakeStore) Get(id int) (*User, error) {
    if u, ok := s.users[id]; ok { return u, nil }
    return nil, ErrNotFound
}

// 测试
store := &FakeStore{users: map[int]*User{1: {Name: "A"}}}
svc := NewService(store)
```

简单、不需要框架。Go 团队倾向 fake > mock。

### 6.3 用 mockgen / testify/mock

复杂依赖（如 gRPC 客户端 50 个方法）时，手写 fake 太繁琐。可以用 [mockgen](https://github.com/uber-go/mock) 或 testify/mock 自动生成。

```go
//go:generate mockgen -source=store.go -destination=mock_store.go

mock := NewMockStore(ctrl)
mock.EXPECT().Get(1).Return(&User{Name: "A"}, nil)
```

### 6.4 何时不用 mock

如果一个测试 90% 是 setup mocks + assert calls，往往设计有问题——把业务逻辑提取为纯函数，让 mock 集中在 IO 边界。

---

## 第七章：TestMain

### 7.1 全局 setup / teardown

```go
func TestMain(m *testing.M) {
    // 全局 setup
    db = setupDB()
    code := m.Run()
    // 全局 teardown
    db.Close()
    os.Exit(code)
}
```

每个 `_test.go` package 最多一个 `TestMain`。所有 `TestXxx` 在 `m.Run()` 内执行。

### 7.2 何时用

- 启动测试数据库
- 注册全局 mock
- 解析命令行参数（自定义 flag）

```go
var dbDSN = flag.String("dsn", "", "test database DSN")

func TestMain(m *testing.M) {
    flag.Parse()
    // ...
}
```

---

## 第八章：Fuzz testing（Go 1.18+）

### 8.1 基础

```go
func FuzzParseURL(f *testing.F) {
    f.Add("http://example.com")
    f.Add("https://golang.org/doc")

    f.Fuzz(func(t *testing.T, raw string) {
        u, err := ParseURL(raw)
        if err != nil { return }
        round := u.String()
        if _, err := ParseURL(round); err != nil {
            t.Errorf("round-trip failed: %q -> %q -> error %v", raw, round, err)
        }
    })
}
```

`f.Add` 提供 seed corpus；`f.Fuzz` 接收随机变异输入。

### 8.2 运行

```bash
go test -fuzz=FuzzParseURL -fuzztime=30s
```

发现失败时，输入会保存到 `testdata/fuzz/` —— 之后普通 `go test` 把它作为回归测试。

### 8.3 用途

- 解析器（URL、JSON、proto）
- 加密/签名验证
- 状态机 invariant

---

## 第八章半：testing/synctest —— 测并发代码不再等真实时钟（Go 1.25+）

测带定时器的并发逻辑过去要么 sleep（慢、不稳）、要么注入 fake clock（侵入业务代码）。`testing/synctest`（**Go 1.24 实验，Go 1.25 稳定**）把整套测试装进一个"气泡"里，**虚拟化时间 + 自动等待 goroutine 全部阻塞**。

### 9.1 Test 与 Wait 双 API

```go
import "testing/synctest"

func TestRefreshEvery5Min(t *testing.T) {
    synctest.Test(t, func(t *testing.T) {
        cache := NewCache(5 * time.Minute)

        cache.Refresh()
        old := cache.LastRefresh()

        time.Sleep(4 * time.Minute)
        synctest.Wait()                // 等气泡内所有 goroutine 阻塞
        if cache.LastRefresh() != old {
            t.Fatal("不该提前 refresh")
        }

        time.Sleep(2 * time.Minute)    // 累计 6 分钟
        synctest.Wait()
        if cache.LastRefresh() == old {
            t.Fatal("应已自动 refresh")
        }
    })
}
```

行为要点：

- `synctest.Test` 把闭包跑在一个隔离的 **bubble** 里。
- `time.Now()` / `time.Sleep` / `time.Timer` / `time.Ticker` 在 bubble 内走**假时钟**。
- 假时钟只在"bubble 内全部 goroutine 都阻塞"时才向前跳——这等价于"逻辑时钟"。
- `synctest.Wait()` 同步点：等 bubble 内全部 goroutine 阻塞，确保后续断言不会与异步逻辑赛跑。
- 实际墙钟时间近乎 0——一个"等 24 小时"的逻辑测试，跑起来还是毫秒级。

### 9.2 替代了什么

| 旧做法 | synctest 怎么取代 |
|---|---|
| `time.Sleep(50*time.Millisecond)` 等异步完成 | `synctest.Wait()` |
| 注入 `Clock` interface + 业务代码用 `clock.Now()` | 业务代码继续用 `time.Now()` |
| 测试里手动推进 mock clock | 用真实 `time.Sleep` 推进；假时钟自动跳 |
| `goleak.VerifyNone` | bubble 退出时未清理的 goroutine 会被报告 |

### 9.3 注意事项

- 仅 `time`、`context.WithTimeout/Deadline`、`sync` 与一些 channel 阻塞参与"全部阻塞"判定。涉及真实 I/O（网络、磁盘）的 goroutine 永远不会"阻塞"——这类测试不适合 synctest。
- 旧实验 API `synctest.Run`（Go 1.24 `GOEXPERIMENT=synctest`）在 Go 1.26 移除。新代码用 `synctest.Test` / `synctest.Wait`。

> 参考：[Go 1.25 release notes — testing/synctest](https://go.dev/doc/go1.25#testingsynctest)、[Go 1.24 release notes — testing/synctest](https://go.dev/doc/go1.24#testingsynctest)。

---

## 第九章：覆盖率

### 9.1 跑覆盖

```bash
go test -coverprofile=cover.out ./...
go tool cover -html=cover.out                    # 浏览器查看
go tool cover -func=cover.out                    # 函数级摘要
```

### 9.2 阈值

```bash
go test -coverprofile=cover.out ./...
go tool cover -func=cover.out | grep total | awk '{print $3}'
# "85.2%"
```

CI 检查阈值（如 < 80% fail）。

### 9.3 不是终极指标

100% 覆盖不等于"测了所有行为"——只是行被执行过。要看：
- 边界条件
- 错误路径
- 并发场景

---

## 第十章：生产级最佳实践

1. **test 与实现同包**：除非有循环依赖。
2. **table-driven + subtests**：标准、可扩展。
3. **每个 subtest `t.Parallel()`**：CI 提速。
4. **t.Helper 标记测试辅助函数**：失败定位更准。
5. **t.Cleanup 取代 defer**：跨 subtest 安全。
6. **t.TempDir + t.Setenv**：自动隔离环境。
7. **fake 优先 mock**：避免过度耦合实现细节。
8. **httptest 测 HTTP**：不要启真实端口（除非 e2e）。
9. **fuzz 覆盖解析器**：30 秒 fuzz 比 10000 个手写 case 强。
10. **CI 跑 `-race -coverprofile`**：发现 race + 监控覆盖。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：循环变量在 subtest 中（仅当 go.mod < 1.22）
```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        use(tt)   // ⚠️ go.mod 声明 < 1.22 时所有 subtest 看到的是最后一个 tt
    })
}
```
修：`tt := tt`，或把 `go.mod` 升到 `go 1.22+`（推荐——所有维护中的 Go 版本都已 ≥ 1.22）。

### ❌ 陷阱 2：t.Fatal in goroutine
```go
go func() { if x != y { t.Fatal(...) } }()   // ❌ runtime.Goexit 在错误 goroutine
```
用 `t.Errorf` + 关闭 channel + 主 goroutine `t.FailNow()`。

### ❌ 陷阱 3：测试相互依赖顺序
依赖前一个测试留下的状态。Go 测试顺序不保证（尤其加了 -parallel）。每个测试自给自足。

### ❌ 陷阱 4：mock 验证调用次数耦合实现
`mock.EXPECT().Get(1).Times(2)` —— 如果重构改成只调 1 次仍正确，测试坏。验证**结果**，不是**调用细节**。

### ❌ 陷阱 5：测试输出全 OK 但没断言
```go
func TestX(t *testing.T) {
    result := Compute()
    // 忘了断言 result
}
```
启用 `staticcheck` SA1019 等。

### ❌ 陷阱 6：依赖外部网络
集成测试用 httptest；单元测试不该访问真实网络。

### ❌ 陷阱 7：长 sleep 在测试里
`time.Sleep(time.Second)` 让测试变慢且 flaky。换成 channel 通知或 `assert.Eventually`。

---

## 第十二章：练习题

**练习 1**：把以下测试改成 table-driven + subtests：
```go
func TestAdd(t *testing.T) {
    if Add(1, 2) != 3 { t.Fatal() }
    if Add(0, 0) != 0 { t.Fatal() }
    if Add(-1, 1) != 0 { t.Fatal() }
}
```

**练习 2**：写一个 `httptest` 测试，验证 `client.GetUser(42)` 发起的请求 URL 包含 `/users/42`。

**练习 3**：写一个 Fake 实现 `Mailer` 接口，记录所有发送的邮件供测试断言。

**练习 4**：以下测试如何 flaky？修复。
```go
func TestAsync(t *testing.T) {
    go Process()
    time.Sleep(100 * time.Millisecond)
    if !Done { t.Fail() }
}
```

**练习 5**：用 fuzz 测试一个 `Reverse(s string) string` 函数（要求 Reverse(Reverse(s)) == s 对所有合法 UTF-8 字符串成立）。

---

## 参考答案

**练习 1**：
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name string; a, b, want int
    }{
        {"pos", 1, 2, 3},
        {"zero", 0, 0, 0},
        {"neg+pos", -1, 1, 0},
    }
    for _, tt := range tests {
        tt := tt
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            if got := Add(tt.a, tt.b); got != tt.want {
                t.Errorf("Add(%d,%d)=%d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

**练习 2**：
```go
func TestGetUser(t *testing.T) {
    var captured string
    srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        captured = r.URL.Path
        w.Write([]byte(`{"id":42}`))
    }))
    defer srv.Close()
    c := NewClient(srv.URL)
    _, _ = c.GetUser(42)
    if captured != "/users/42" {
        t.Errorf("URL = %s", captured)
    }
}
```

**练习 3**：
```go
type FakeMailer struct {
    mu   sync.Mutex
    Sent []Email
}
func (m *FakeMailer) Send(e Email) error {
    m.mu.Lock(); defer m.mu.Unlock()
    m.Sent = append(m.Sent, e)
    return nil
}
```

**练习 4**：sleep 100ms 不一定够（CI 慢机器、GC 暂停）。修：
```go
done := make(chan struct{})
go func() { defer close(done); Process() }()
select {
case <-done:
case <-time.After(5 * time.Second):
    t.Fatal("timeout")
}
```

**练习 5**：
```go
func FuzzReverse(f *testing.F) {
    f.Add("hello")
    f.Add("中文")
    f.Fuzz(func(t *testing.T, s string) {
        if !utf8.ValidString(s) { return }
        r := Reverse(Reverse(s))
        if r != s { t.Errorf("round-trip: %q -> %q", s, r) }
    })
}
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 文件命名 | `_test.go`；可同包或 _test 包 |
| table-driven | + subtests + parallel = 标准模式 |
| t.Helper | 跳过辅助函数调用栈 |
| t.Cleanup | 自动 LIFO；跨 subtest |
| httptest | NewRequest+Recorder（handler 测试）/ NewServer（client 测试） |
| Fake vs Mock | Go 偏好 fake |
| TestMain | 全局 setUp/tearDown |
| Fuzz | 1.18+；解析器、加密、状态机 |
| 覆盖率 | -coverprofile；不是终极指标 |

下一篇 **G19 — 精通 Go Benchmarking 与 benchstat** 会讲清 `testing.B`、b.ResetTimer、b.RunParallel、benchstat 对比、防优化掉的技巧。

---

