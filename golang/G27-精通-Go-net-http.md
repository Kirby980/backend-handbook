# 精通 Go net/http：ServeMux、Middleware 与 Graceful Shutdown

> 课程编号：G27
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Web Development 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：标准库够用吗

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World!")
})
http.ListenAndServe(":8080", nil)
```

四行代码做出 web server——这是 Go 的招牌。但生产环境的 net/http 远不止于此：超时配置、graceful shutdown、middleware 链、ServeMux 路由限制、连接池、HTTP/2 配置。这一章把生产级 `net/http` 拆开。

---

## 第一章：Server 与 Handler

### 1.1 Handler 接口

```go
type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}
```

任何实现 `ServeHTTP(w, r)` 的类型都是 handler。`http.HandlerFunc(fn)` 把函数包装为 Handler。

### 1.2 ServeMux

```go
mux := http.NewServeMux()
mux.HandleFunc("/api/users", usersHandler)
mux.HandleFunc("/api/orders", ordersHandler)

server := &http.Server{Addr: ":8080", Handler: mux}
server.ListenAndServe()
```

`ServeMux` 是 Go 内置的简单路由器：

- 最长前缀匹配：`/api/` 匹配 `/api/foo`、`/api/bar`
- 末尾 `/` 表示子树（subtree）
- 末尾无 `/` 表示精确匹配

### 1.3 增强 ServeMux（Go 1.22 引入，已是标配）

自 Go 1.22 起 ServeMux 支持：

- 方法匹配：`mux.HandleFunc("POST /users", ...)`
- 路径参数：`mux.HandleFunc("/users/{id}", func(w, r) { id := r.PathValue("id") })`
- wildcard：`/files/{path...}`

```go
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
})
```

Go 1.22 之前要用 chi、gorilla/mux、gin 等第三方路由。**今天（Go 1.26 时代）维护中的项目几乎都已升级**，新项目直接用标准库就够；只有需要 middleware 链组合、路由组、参数注入等更重业务时再考虑第三方。

### 1.4 默认 ServeMux 的陷阱

```go
http.HandleFunc("/", ...)
http.ListenAndServe(":8080", nil)
```

`http.HandleFunc` 注册到**全局**默认 mux。问题：

- 导入的库可能也注册（如 `_ "net/http/pprof"`）
- 测试相互污染
- 失去 explicit 控制

**生产代码永远 `http.NewServeMux()`**。

---

## 第二章：超时——最重要的生产配置

### 2.1 默认超时是 0（永不超时）

```go
http.ListenAndServe(":8080", mux)   // 无超时
```

这是无数生产事故的源头。慢客户端能用 4 字节请求让 server 耗尽连接（Slowloris 攻击）。

### 2.2 必须设的四个超时

```go
server := &http.Server{
    Addr:              ":8080",
    Handler:           mux,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
}
server.ListenAndServe()
```

| 超时 | 含义 |
|---|---|
| ReadHeaderTimeout | 读完请求头的最大时间 |
| ReadTimeout | 读完完整请求（含 body）最大时间 |
| WriteTimeout | 从读完请求到写完响应最大时间 |
| IdleTimeout | keep-alive 空闲连接保留时间 |

### 2.3 超时应对策略

- 客户端慢：ReadHeaderTimeout / ReadTimeout 截断
- 处理慢：WriteTimeout（不严谨——它从读完 request 算起）
- 业务超时：用 `context.WithTimeout` 在 handler 内控制

### 2.4 MaxHeaderBytes

```go
server.MaxHeaderBytes = 1 << 20   // 1MB
```

防止巨大头部攻击。默认 1MB，够用。

---

## 第三章：Middleware 模式

### 3.1 中间件签名

```go
type Middleware func(http.Handler) http.Handler

func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

接受一个 handler、返回一个包装的 handler。

### 3.2 组合

```go
handler := Logger(Auth(RateLimit(mux)))
```

或 builder 风格：

```go
func Chain(h http.Handler, mws ...Middleware) http.Handler {
    for i := len(mws) - 1; i >= 0; i-- {
        h = mws[i](h)
    }
    return h
}
handler := Chain(mux, Logger, Auth, RateLimit)
```

### 3.3 必备中间件

```go
// recover：捕获 panic
func Recover(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if rec := recover(); rec != nil {
                log.Printf("panic: %v\n%s", rec, debug.Stack())
                http.Error(w, "internal error", 500)
            }
        }()
        next.ServeHTTP(w, r)
    })
}

// requestID
func RequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get("X-Request-ID")
        if id == "" { id = uuid.New().String() }
        ctx := context.WithValue(r.Context(), reqIDKey, id)
        w.Header().Set("X-Request-ID", id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// CORS
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        if r.Method == "OPTIONS" { return }
        next.ServeHTTP(w, r)
    })
}
```

### 3.4 ResponseWriter 包装

记录响应 status code 等：

```go
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (sw *statusWriter) WriteHeader(s int) {
    sw.status = s
    sw.ResponseWriter.WriteHeader(s)
}

func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        sw := &statusWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(sw, r)
        log.Printf("%d %s", sw.status, r.URL.Path)
    })
}
```

---

## 第四章：Graceful Shutdown

### 4.1 信号触发

```go
import (
    "os/signal"
    "syscall"
)

ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

server := &http.Server{...}
go server.ListenAndServe()

<-ctx.Done()
shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := server.Shutdown(shutdownCtx); err != nil {
    log.Printf("shutdown error: %v", err)
}
```

`server.Shutdown(ctx)`：
1. 关闭监听 socket，不再接新连接
2. 已建立连接继续处理直到完成
3. 等待 ctx 超时；超时后强制关连接

### 4.2 实际部署

K8s pod 终止流程：

1. K8s 发送 SIGTERM
2. server 进入 graceful shutdown
3. service mesh 不再发流量（取决于 readiness probe）
4. 等连接处理完
5. 进程退出

要点：grace period 要长于服务的 P99 响应时间。

### 4.3 readiness probe 配合

```go
var ready atomic.Bool
ready.Store(true)

// readiness handler
mux.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    if !ready.Load() { http.Error(w, "shutting down", 503); return }
    w.WriteHeader(200)
})

// shutdown
ready.Store(false)              // 立即停接收新流量
time.Sleep(5 * time.Second)     // 等 ELB/k8s 检测
server.Shutdown(ctx)            // 优雅关闭
```

---

## 第五章：客户端

### 5.1 永远不要用 http.DefaultClient

```go
resp, _ := http.Get(url)   // 无超时！
```

默认 client 没超时——慢服务器能让 goroutine 永久挂起。

### 5.2 自定义 client

```go
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 10 * time.Second,
    },
}
```

`Timeout` 是**整个请求**（含 DNS + connect + read body）的最大时间。

### 5.3 连接复用

```go
resp, _ := client.Get(url)
defer resp.Body.Close()
io.ReadAll(resp.Body)   // 必须读完，否则连接不能复用
```

**永远 Close body**，**永远读完**——不然 keep-alive 失败，每个请求都新 connect。

### 5.4 Context

```go
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```

ctx 超时/取消 → 请求被中断。

---

## 第六章：路由库选择

### 6.1 标准库 ServeMux（Go 1.22+）

- 零依赖
- 方法 + path 参数
- 通配符
- 性能足够

**默认选它**。

### 6.2 chi

- 中间件友好（链式）
- 类似 Express
- 性能好

适合复杂中间件层级。

### 6.3 gin

- 全功能框架
- 自带 binding、validation、render
- 性能强

适合快速构建 REST API；但脱离标准库更多。

### 6.4 echo / fiber

类似 gin。fiber 基于 fasthttp（非标准 net/http），生态不兼容标准 middleware。

### 6.5 gorilla/mux

历史悠久；已 sunset（维护者放弃）。新项目别用。

---

## 第七章：HTTP/2 与 HTTP/3

### 7.1 HTTP/2

`net/http` 默认在 TLS 启用时自动用 HTTP/2：

```go
server.ListenAndServeTLS("cert.pem", "key.pem")   // 自动 HTTP/2
```

特性：多路复用、头部压缩、server push（已逐步废弃）。

### 7.2 h2c（HTTP/2 cleartext）

不用 TLS 的 HTTP/2：

```go
import "golang.org/x/net/http2/h2c"

h2s := &http2.Server{}
handler := h2c.NewHandler(mux, h2s)
server := &http.Server{Handler: handler}
```

主要用于 service mesh 内部通信（已 TLS 由 sidecar 处理）。

### 7.3 HTTP/3 / QUIC

`net/http` 还未正式支持。用 `quic-go/http3` 第三方库。

---

## 第七章半：内置 CSRF 防护 —— `http.CrossOriginProtection`（Go 1.25+）

### 7.5 长久以来的痛点

CSRF 过去要么靠 SameSite cookie + 自己分发同步 token，要么引第三方库（`gorilla/csrf` 等）。**Go 1.25 把 OWASP 推荐的"基于 Fetch metadata 的 CSRF 防护"抽进了标准库**：[`net/http.CrossOriginProtection`](https://pkg.go.dev/net/http#CrossOriginProtection)。

### 7.6 工作原理

不依赖 token、不发新 cookie。中间件检查每个非安全方法（POST/PUT/PATCH/DELETE）的两类信号：

1. **`Sec-Fetch-Site` 请求头**——浏览器自动标注请求来源（`same-origin` / `same-site` / `cross-site` / `none`）。`cross-site` 直接拒。
2. 如果浏览器没发 `Sec-Fetch-Site`（旧浏览器或非浏览器 client），**回退比对 `Origin` 与 `Host`**——不一致就拒。

### 7.7 用法

```go
mux := http.NewServeMux()
mux.HandleFunc("POST /transfer", handleTransfer)

protection := http.NewCrossOriginProtection()
protection.AddTrustedOrigin("https://admin.example.com")  // 允许来自这个域名的 cross-origin
protection.AddInsecureBypassPattern("/webhook/")          // 第三方 webhook 走 HMAC 验证，不走 CSRF

srv := &http.Server{
    Addr:    ":8080",
    Handler: protection.Handler(mux),
}
log.Fatal(srv.ListenAndServe())
```

GET / HEAD / OPTIONS 不会被检查（语义上应该是幂等的）。

### 7.8 配合现代浏览器

`Sec-Fetch-Site` 在 Chrome/Edge/Firefox/Safari 主流版本都已支持多年。**这套防护对真实浏览器流量足够安全**——攻击者无法在普通跨站环境下让浏览器自己发出标着 `same-origin` 的请求。

> 参考：[Go 1.25 release notes — net/http](https://go.dev/doc/go1.25#nethttp)、[OWASP CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)。

---

## 第八章：常见生产配置

### 8.1 完整模板

```go
mux := http.NewServeMux()
// 注册路由...

handler := Chain(mux,
    Recover,
    RequestID,
    Logger,
    Auth,
)

server := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 5 * time.Second,
    ReadTimeout:       30 * time.Second,
    WriteTimeout:      30 * time.Second,
    IdleTimeout:       120 * time.Second,
    MaxHeaderBytes:    1 << 20,
    ErrorLog:          log.New(os.Stderr, "http: ", log.LstdFlags),
}

ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

go func() {
    log.Println("listening on :8080")
    if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatalf("listen: %v", err)
    }
}()

<-ctx.Done()
log.Println("shutting down")
shutCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
if err := server.Shutdown(shutCtx); err != nil {
    log.Printf("shutdown: %v", err)
}
log.Println("bye")
```

---

## 第九章：生产级最佳实践

1. **永远 `http.NewServeMux()`**：不要用全局 DefaultMux。
2. **所有超时必须设**：ReadHeaderTimeout、ReadTimeout、WriteTimeout、IdleTimeout。
3. **client.Timeout 必须设**：调外部 API 永远不能"无限等"。
4. **Body 永远 Close + 读完**：保证 keep-alive。
5. **Graceful shutdown**：signal.NotifyContext + server.Shutdown。
6. **Recover middleware** + structured logging。
7. **Request ID** 贯穿日志、tracing、metrics。
8. **Readiness probe** 配合 shutdown：先 503 再 close。
9. **MaxHeaderBytes 限制**：防 DoS。
10. **Go 1.22+ 用增强 ServeMux**：减少第三方依赖。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：DefaultClient/Server 无超时
```go
http.Get(url)   // 可能挂几小时
```

### ❌ 陷阱 2：忘了 close body
```go
resp, _ := http.Get(url)
return resp.StatusCode   // body 泄漏 → 连接不能复用
```

### ❌ 陷阱 3：WriteTimeout 误解
WriteTimeout 从读完 request 算起；对 streaming response 长写入会被截断。

### ❌ 陷阱 4：handler 内启动后台 goroutine 不挂 ctx
handler 返回后 ctx cancel；后台 goroutine 应该 detach 或自己管理生命周期。

### ❌ 陷阱 5：CORS middleware 顺序错误
Auth 在 CORS 前 → preflight 请求被认证拒绝。CORS 必须在 Auth 之前。

### ❌ 陷阱 6：panic 不 recover
默认 net/http server 有内置 recover（不会杀进程，但断连接）。自己加 middleware 给响应可控的 500。

### ❌ 陷阱 7：用 fasthttp 但中间件生态不兼容
fasthttp 不是 net/http API，要重写所有 middleware。除非性能极致需求，否则不值。

---

## 第十一章：练习题

**练习 1**：写一个完整生产 server 启动模板，含信号处理、超时、middleware。

**练习 2**：以下代码哪里有问题？
```go
for _, url := range urls {
    resp, _ := http.Get(url)
    process(resp.Body)
}
```

**练习 3**：实现一个 rate limit middleware（基于 token bucket）。

**练习 4**：解释 keep-alive 不工作时性能下降多少？

**练习 5**：用 Go 1.22 ServeMux 实现 REST 路由 `GET /users/{id}`、`POST /users`。

---

## 参考答案

**练习 1**：参考第八章模板。

**练习 2**：
- 错误未处理
- resp.Body 没 close → 连接泄漏 + goroutine 泄漏
- 无 timeout

修：
```go
client := &http.Client{Timeout: 30*time.Second}
for _, url := range urls {
    resp, err := client.Get(url)
    if err != nil { log.Println(err); continue }
    process(resp.Body)
    resp.Body.Close()
}
```

**练习 3**：
```go
func RateLimit(rate int) Middleware {
    lim := rate.NewLimiter(rate.Limit(rate), rate*2)
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !lim.Allow() {
                http.Error(w, "too many requests", 429)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

**练习 4**：keep-alive 复用 TCP 连接和 TLS 会话。失败时每个请求要 TCP handshake（1 RTT）+ TLS handshake（1-2 RTT）。100ms 网络下每请求增 200-300ms。

**练习 5**：
```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
})
mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
    // ...
})
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Handler | ServeHTTP(w, r) 接口 |
| ServeMux | Go 1.22+ 自带方法 + path 参数 |
| 超时 | 4 个必设 |
| Middleware | func(Handler) Handler |
| Graceful shutdown | signal + Shutdown |
| Client | 自定义 Transport + Timeout |
| Body | 必读完 + Close |
| HTTP/2 | TLS 自动启用 |

下一篇 **G28 — 精通 gRPC 与 Protobuf** 将拆开 HTTP/2 流、protoc 代码生成、interceptor、双向流、可靠生产配置。

---

