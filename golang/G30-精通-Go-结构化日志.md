# 精通 Go 结构化日志：slog、Zap 与 Zerolog

> 课程编号：G30
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Logging 章节
> 难度：⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：为什么不能 `fmt.Println`

```go
log.Printf("user %d failed to login: %v", uid, err)
```

看起来工作。但这条日志在 ELK / Loki / Datadog 里：

- 解析靠 regex（脆）
- 字段名"user"、错误内容混在文本
- 没有 traceID 关联
- 时间戳格式不统一

**结构化日志**（structured logging）把每条日志输出为 JSON 等机器友好格式：

```json
{"time":"2026-05-12T10:30:00Z","level":"ERROR","msg":"login failed","user_id":42,"error":"bad password","trace_id":"abc123"}
```

Go 生态有三个主流方案：标准库 `slog`（Go 1.21+）、`uber-go/zap`、`rs/zerolog`。本章拆开它们。

---

## 第一章：slog —— 标准库（Go 1.21+）

### 1.1 基础

```go
import "log/slog"

slog.Info("hello", "user", "alice", "age", 30)
// {"time":"...","level":"INFO","msg":"hello","user":"alice","age":30}
```

key-value 对作为可变参数。或用 `slog.String`、`slog.Int` 等强类型 helper：

```go
slog.Info("hello",
    slog.String("user", "alice"),
    slog.Int("age", 30),
)
```

### 1.2 配置 handler

```go
// JSON 输出
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    AddSource: true,
}))

// 文本格式（人类可读）
logger := slog.New(slog.NewTextHandler(os.Stderr, nil))

slog.SetDefault(logger)
```

### 1.3 子 logger

```go
logger := slog.With("trace_id", traceID, "user_id", uid)
logger.Info("processing")
logger.Error("failed", "error", err)
```

`With` 返回带固定 attrs 的子 logger——后续每条日志都带上。

### 1.4 Group

```go
slog.Info("request",
    slog.Group("user",
        slog.String("name", "alice"),
        slog.Int("age", 30),
    ),
    slog.Group("request",
        slog.String("method", "GET"),
        slog.String("path", "/api"),
    ),
)
// {"msg":"request","user":{"name":"alice","age":30},"request":{"method":"GET","path":"/api"}}
```

嵌套字段。

### 1.5 自定义 handler

```go
type myHandler struct{}
func (h *myHandler) Enabled(ctx context.Context, level slog.Level) bool { return true }
func (h *myHandler) Handle(ctx context.Context, r slog.Record) error { /* ... */ }
func (h *myHandler) WithAttrs(attrs []slog.Attr) slog.Handler { /* ... */ }
func (h *myHandler) WithGroup(name string) slog.Handler { /* ... */ }
```

实现 4 方法接口可以做任意输出（推到 Kafka、ClickHouse、syslog 等）。

### 1.5 Go 1.25 / 1.26 新增 API

`slog` 在 Go 1.21 落地后一路在迭代。两个最实用的新增：

- **`slog.GroupAttrs(key, attrs...)`**（Go 1.25+）—— 一次性把若干 `Attr` 合成一个 group。比之前一个个写 `slog.Attr{Key:..., Value:...}` 短得多：

  ```go
  attrs := []slog.Attr{slog.String("ip", ip), slog.Int("port", port)}
  logger.LogAttrs(ctx, slog.LevelInfo, "conn", slog.GroupAttrs("peer", attrs...))
  ```

- **`slog.NewMultiHandler(h1, h2, ...)`**（Go 1.26+）—— 把一条记录扇出给多个 handler。常见用法：JSON handler 进 ELK/Loki，同时 text handler 打到 stderr 给本地开发看：

  ```go
  jsonH := slog.NewJSONHandler(logFile, nil)
  textH := slog.NewTextHandler(os.Stderr, nil)
  logger := slog.New(slog.NewMultiHandler(jsonH, textH))
  ```

  老办法是手写一个分发 handler；现在标准库自带。

> 参考：[Go 1.25 release notes — log/slog](https://go.dev/doc/go1.25#logslog)、[Go 1.26 release notes — log/slog](https://go.dev/doc/go1.26#logslog)。

---

## 第二章：Zap

### 2.1 两种 API

**SugaredLogger**（方便）：

```go
logger, _ := zap.NewProduction()
defer logger.Sync()
sugar := logger.Sugar()

sugar.Infow("user logged in", "uid", 42, "ip", "1.2.3.4")
sugar.Infof("user %d logged in", 42)   // 类似 Printf
```

**Logger**（最快，强类型）：

```go
logger.Info("user logged in",
    zap.Int("uid", 42),
    zap.String("ip", "1.2.3.4"),
)
```

Sugared 比 Logger 慢约 50%（仍比标准库 log 快 10x），可读性好。

### 2.2 配置

```go
cfg := zap.NewProductionConfig()
cfg.Level = zap.NewAtomicLevelAt(zap.DebugLevel)
cfg.OutputPaths = []string{"stdout", "/var/log/app.log"}
logger, _ := cfg.Build()
```

### 2.3 性能

每条日志约 **100ns**（不含 IO），是性能最强的库。原理：
- 零反射（强类型 API）
- 预分配 buffer
- 字段直接序列化为字节，不构造 map

### 2.4 何时选 Zap

- 性能极致（金融、广告、游戏后端）
- 需要 zap.Object 等高级序列化

---

## 第三章：Zerolog

### 3.1 API

```go
import "github.com/rs/zerolog/log"

log.Info().
    Str("user", "alice").
    Int("age", 30).
    Msg("hello")
```

链式调用——每次返回 *Event。

### 3.2 性能

与 Zap 接近，甚至某些场景更快。**Allocation 极少**——通常零分配。

### 3.3 配置

```go
import "github.com/rs/zerolog"

zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
log.Logger = log.With().Caller().Logger()
```

### 3.4 优点

- 链式 API 可读
- 零分配
- console writer 美化输出（dev）

### 3.5 缺点

- 链式 API 与 slog / zap 不同，迁移成本
- 不是标准库，要依赖

---

## 第四章：三者对比

| 维度 | slog | Zap | Zerolog |
|---|---|---|---|
| 标准库 | ✅ | 第三方 | 第三方 |
| 性能 | 中 | 最快 | 最快 |
| 分配 | 少 | 极少 | 零 |
| API 风格 | k-v 可变参 | 函数式（zap.Int）/ Sugared | 链式 |
| Handler 抽象 | ✅ 灵活 | ❌ Core 较复杂 | 中 |
| 生态 | 新但增长 | 成熟 | 成熟 |

**默认选 slog**——除非性能极致需求选 Zap/Zerolog。slog 的 handler 抽象允许将来切换实现（如把 slog 桥接到 zap）。

### 4.1 slog → zap 桥接

```go
import "go.uber.org/zap/exp/zapslog"

zapLogger, _ := zap.NewProduction()
slogLogger := slog.New(zapslog.NewHandler(zapLogger.Core()))
slog.SetDefault(slogLogger)
```

应用代码统一用 slog API，底层用 zap 性能。

---

## 第五章：生产配置

### 5.1 JSON 输出 + stdout

容器化部署的标准做法——日志写到 stdout，由 docker / k8s 收集再投递到 ELK / Loki。

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))
```

### 5.2 必备字段

每条日志至少包含：

- `time`（ISO 8601 / RFC3339）
- `level`
- `msg`
- `trace_id`（关联分布式追踪）
- `request_id`（关联单次请求）
- `service` / `version`（来源标识）

### 5.3 与 context 关联

```go
// middleware 注入
ctx = context.WithValue(ctx, loggerKey, logger.With("trace_id", traceID))

// handler 取出
func loggerFrom(ctx context.Context) *slog.Logger {
    if l, ok := ctx.Value(loggerKey).(*slog.Logger); ok { return l }
    return slog.Default()
}
```

slog 没内置 context 集成 helper，但 handler 可以从 ctx 提取字段。

### 5.4 与 OpenTelemetry 集成

```go
// 在 handler 的 Handle 中读 ctx 内的 span
import "go.opentelemetry.io/otel/trace"

func (h *otelHandler) Handle(ctx context.Context, r slog.Record) error {
    if span := trace.SpanFromContext(ctx); span.IsRecording() {
        sc := span.SpanContext()
        r.AddAttrs(
            slog.String("trace_id", sc.TraceID().String()),
            slog.String("span_id",  sc.SpanID().String()),
        )
    }
    return h.next.Handle(ctx, r)
}
```

---

## 第六章：日志级别选择

### 6.1 五级

```
DEBUG  详细调试信息（开发用）
INFO   一般运行信息
WARN   异常情况但可恢复
ERROR  应该告警
FATAL  无法继续，进程退出（slog 没有 FATAL，用 log.Fatal）
```

### 6.2 何时用 ERROR

- 系统级失败（DB 连不上、磁盘满）
- 应该让 oncall 看到
- **不**用于业务逻辑预期的失败（如 user not found—WARN 或 INFO 即可）

过度使用 ERROR → 告警疲劳。

### 6.3 何时用 DEBUG

```go
slog.Debug("cache miss", "key", k)
```

生产关闭（性能 + 噪音），dev/staging 开。slog 通过 `Level: slog.LevelDebug` 配置。

---

## 第七章：采样（sampling）

### 7.1 问题

每秒 10 万次日志压垮 ELK + 浪费成本。

### 7.2 zap 内置 sampling

```go
core := zapcore.NewSamplerWithOptions(core,
    time.Second,   // 周期
    100,           // 周期前 100 条全输出
    100,           // 之后每 100 条输出 1 条
)
```

### 7.3 slog 自定义

```go
type samplingHandler struct {
    next slog.Handler
    rate uint64
    count atomic.Uint64
}
func (h *samplingHandler) Handle(ctx context.Context, r slog.Record) error {
    if r.Level >= slog.LevelError {
        return h.next.Handle(ctx, r)   // ERROR 不采样
    }
    if h.count.Add(1) % h.rate != 0 { return nil }
    return h.next.Handle(ctx, r)
}
```

### 7.4 何时采样

- 高频成功日志（每个请求 INFO）
- 调试日志（DEBUG）

**ERROR 永不采样**——丢失就发现不了问题。

---

## 第八章：log/syslog 与日志轮转

### 8.1 容器内不轮转

K8s / Docker 接管 stdout，自己轮转。app 不要写本地文件。

### 8.2 传统部署

```go
import "gopkg.in/natefinch/lumberjack.v2"

w := &lumberjack.Logger{
    Filename:   "/var/log/app.log",
    MaxSize:    100, // MB
    MaxBackups: 10,
    MaxAge:     30, // days
}
logger := slog.New(slog.NewJSONHandler(w, nil))
```

`lumberjack` 提供 Write 接口的自动轮转。

### 8.3 多 sink

```go
out := io.MultiWriter(os.Stdout, fileWriter, kafkaWriter)
```

慎用——多 sink 失败处理复杂。容器化环境单 sink（stdout）+ 外部分发更优。

---

## 第九章：生产级最佳实践

1. **结构化 JSON 输出**：可机读、易聚合。
2. **stdout 单一 sink**：让 docker/k8s 处理。
3. **trace_id / request_id 贯穿**：跨服务关联。
4. **ERROR 慎用**：告警纪律。
5. **DEBUG 生产关**：性能 + 安全（避免 PII 泄漏）。
6. **slog 是新默认**：标准库 + handler 灵活。
7. **性能极致需求选 Zap**：100ns/log。
8. **日志中不要 PII / 密码**：遵 GDPR / SOC2。
9. **采样高频 INFO**：保护成本。
10. **集成 OpenTelemetry**：trace 与 log 关联。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：日志带 PII
```go
slog.Info("user", "email", user.Email, "ssn", user.SSN)
```
脱敏或不记。

### ❌ 陷阱 2：fmt.Sprintf 拼字符串
```go
slog.Info(fmt.Sprintf("user %d", uid))   // 失去结构化
```
用 k-v 形式。

### ❌ 陷阱 3：error 直接 log
```go
log.Println(err)
```
用 Error 级别 + 上下文：
```go
slog.Error("failed", "op", "save", "user_id", uid, "error", err)
```

### ❌ 陷阱 4：调用 .Sync() 忘记
zap 缓冲日志，进程退出前 defer Sync。

### ❌ 陷阱 5：日志库阻塞主流程
大 batch 写 disk / kafka 时阻塞 → 用 async writer。

### ❌ 陷阱 6：日志级别 wronged
所有都 INFO 或所有都 ERROR——告警失效。

### ❌ 陷阱 7：忘了关闭 log 文件
程序退出时缓冲未刷 → 部分日志丢失。

---

## 第十一章：练习题

**练习 1**：用 slog 写一个 middleware，给每个 HTTP 请求自动添加 trace_id 和 method+path。

**练习 2**：实现 slog 的 sampling handler（如：每 100 条 INFO 输出 1 条，所有 ERROR 全输出）。

**练习 3**：以下日志哪里不专业？修改。
```go
log.Printf("err: %v", err)
```

**练习 4**：解释 zap.SugaredLogger 与 zap.Logger 的性能差异原因。

**练习 5**：写一个 handler，把 slog 输出转发到 OpenTelemetry log API。

---

## 参考答案

**练习 1**：
```go
func LogMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-Id")
        if traceID == "" { traceID = uuid.New().String() }
        logger := slog.Default().With(
            "trace_id", traceID,
            "method", r.Method,
            "path", r.URL.Path,
        )
        ctx := context.WithValue(r.Context(), loggerKey, logger)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

**练习 2**：见第七章 7.3。

**练习 3**：
- 没字段名
- 没级别（log.Printf 默认 INFO 但 err 应该 WARN/ERROR）
- 没上下文

修：
```go
slog.Error("operation failed", "op", "save_user", "user_id", uid, "error", err)
```

**练习 4**：Sugared 用反射处理 `interface{}` 参数；Logger 接受类型化的 `zap.Field`，直接 marshal 无反射。

**练习 5**：
```go
import "go.opentelemetry.io/otel/log"

type otelHandler struct{ logger log.Logger }
func (h *otelHandler) Handle(ctx context.Context, r slog.Record) error {
    rec := log.Record{}
    rec.SetTimestamp(r.Time)
    rec.SetBody(log.StringValue(r.Message))
    rec.SetSeverity(toOtelSev(r.Level))
    r.Attrs(func(a slog.Attr) bool {
        rec.AddAttributes(log.KeyValue{Key: a.Key, Value: toLogValue(a.Value)})
        return true
    })
    h.logger.Emit(ctx, rec)
    return nil
}
// 其他三个方法略
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| slog | Go 1.21+ 标准库；默认推荐 |
| Zap | 性能最强；100ns/log |
| Zerolog | 链式 API；零分配 |
| 输出 | JSON 到 stdout（容器） |
| 必备字段 | time / level / msg / trace_id / request_id |
| 级别 | ERROR 慎用；DEBUG 生产关 |
| 采样 | INFO 可采样；ERROR 不能 |
| 与 OTel | handler 集成 trace |
| 安全 | 不 log PII / 密码 |

---

## 课程总结

这是 **G30 / 30**——roadmap.sh Go 路线图深度课程系列的最后一篇。

回顾整个系列覆盖：

| 章节 | 主题 |
|---|---|
| G01-G10 | 语言基础：变量、类型、切片、map、struct、函数、指针、接口、泛型、错误 |
| G11-G15 | 并发：goroutine、channel、sync、context、模式 |
| G16-G19 | 工程化：race、modules、测试、benchmark |
| G20-G26 | 性能与底层：GC、逃逸、pprof、trace、反射、unsafe、CGO |
| G27-G30 | 生态：net/http、gRPC、数据库、日志 |

每篇都以"引言悬念 → 章节 → 生产实践 → 陷阱 → 练习 → 小结表 → 下一篇预告"为骨架，配有源码示例、性能数据、工具命令。

**下一步建议**：

1. **不要光读**：每篇配套代码自己跑、验证。
2. **挑 2-3 个深入**：例如 GC（G20）+ 逃逸（G21）+ pprof（G22）能让你成为 Go 性能专家。
3. **做项目实战**：拿一个真实业务（个人博客、微服务、CLI 工具）用到的技术覆盖至少 10 个课程。
4. **看源码**：`runtime/` 目录是宝藏。看 G11、G13、G20 提到的文件。
5. **加入社区**：Gopher Slack、Reddit r/golang、roadmap.sh Discord。

祝你成为优秀的 Go 工程师 🎉

---

