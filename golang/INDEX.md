# Go 路线图深度课程 · 总目录

> 基于 [roadmap.sh/golang](https://roadmap.sh/golang) 生成的 30 篇中文深度课程
> 每篇约 10000-15000 字，含底层原理、代码示例、生产实践、陷阱清单、练习题
> 适合从中级到高级 Go 工程师的系统进阶
>
> **📅 内容基准：Go 1.26**（2026-02-10 发布，最新补丁 1.26.3 / 2026-05-07）
> 涵盖 Go 1.21–1.26 五个版本的关键特性。每章末尾的"📅 2026 更新"小节标注了相对原版本课程的增量。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| G01 | [精通 Go 变量、常量与 iota 习语](./G01-精通-Go-变量-常量与-iota-习语.md) | ⭐⭐⭐ | var / const / iota / 无类型常量 / 作用域 |
| G02 | [精通 Go 数据类型、Rune 与字符串内幕](./G02-精通-Go-数据类型-Rune-与字符串内幕.md) | ⭐⭐⭐ | int 平台差异 / IEEE 754 / UTF-8 / StringHeader |
| G03 | [精通 Go 切片：底层结构与扩容机制](./G03-精通-Go-切片-底层结构与扩容机制.md) | ⭐⭐⭐⭐ | SliceHeader / append 扩容 / 别名陷阱 / 三索引切片 |
| G04 | [精通 Go Map：哈希内幕、迭代顺序与并发](./G04-精通-Go-Map-哈希内幕与并发.md) | ⭐⭐⭐⭐ | hmap / bucket / sync.Map / delete 不缩容 |
| G05 | [精通 Go Struct：内存布局、字段对齐与嵌入](./G05-精通-Go-Struct-内存布局与嵌入.md) | ⭐⭐⭐⭐ | padding / fieldalignment / 嵌入 / noCopy |
| G06 | [精通 Go 函数、闭包与 defer 机制](./G06-精通-Go-函数-闭包与-defer-机制.md) | ⭐⭐⭐⭐ | 闭包 / 开放编码 defer / 命名返回值 |
| G07 | [精通 Go 指针与方法接收者](./G07-精通-Go-指针-与方法接收者.md) | ⭐⭐⭐⭐ | 值/指针接收者 / 方法集 / 可寻址性 |
| G08 | [精通 Go 接口：itab 与类型断言](./G08-精通-Go-接口-itab-与类型断言.md) | ⭐⭐⭐⭐⭐ | iface / eface / itab / nil interface 陷阱 |
| G09 | [精通 Go 泛型：类型参数、约束与性能](./G09-精通-Go-泛型-类型参数与约束.md) | ⭐⭐⭐⭐ | 类型集 / `~T` / GC Shape Stenciling |
| G10 | [精通 Go 错误处理与 panic/recover](./G10-精通-Go-错误处理-与-panic-recover.md) | ⭐⭐⭐⭐ | %w wrapping / errors.Is/As / Join |
| G11 | [精通 Goroutines 与 GMP 调度](./G11-精通-Goroutines-与-GMP-调度.md) | ⭐⭐⭐⭐⭐ | G/M/P / work stealing / 异步抢占 |
| G12 | [精通 Go Channels 与 select](./G12-精通-Go-Channels-与-select.md) | ⭐⭐⭐⭐⭐ | hchan / 状态机 / nil channel / broadcast |
| G13 | [精通 sync 包：Mutex、RWMutex、WaitGroup、Once、Pool](./G13-精通-Go-sync-包.md) | ⭐⭐⭐⭐⭐ | 自旋 / 饥饿模式 / atomic.Value |
| G14 | [精通 Go context 包](./G14-精通-Go-context-包.md) | ⭐⭐⭐⭐ | 树形结构 / cancel / WithValue / 传播 |
| G15 | [精通 Go 并发模式](./G15-精通-Go-并发模式.md) | ⭐⭐⭐⭐⭐ | pipeline / fan-out/in / worker pool / errgroup |
| G16 | [精通 Race Detection 与数据竞争调试](./G16-精通-Go-Race-Detection.md) | ⭐⭐⭐⭐ | TSan / happens-before / `-race` |
| G17 | [精通 Go Modules](./G17-精通-Go-Modules.md) | ⭐⭐⭐ | go.mod / MVS / replace / GOPRIVATE |
| G18 | [精通 Go 测试：table-driven、subtests、httptest 与 mocks](./G18-精通-Go-测试.md) | ⭐⭐⭐⭐ | t.Parallel / t.Cleanup / fuzz / fake vs mock |
| G19 | [精通 Go Benchmarking 与 benchstat](./G19-精通-Go-Benchmarking.md) | ⭐⭐⭐⭐ | b.N / ResetTimer / benchstat / 防编译器消除 |
| G20 | [精通 Go 内存管理：GC、GOGC 与 GOMEMLIMIT](./G20-精通-Go-内存管理.md) | ⭐⭐⭐⭐⭐ | 三色标记 / 写屏障 / size class / GOMEMLIMIT |
| G21 | [精通 Go 逃逸分析：栈 vs 堆与 -gcflags=-m](./G21-精通-Go-逃逸分析.md) | ⭐⭐⭐⭐⭐ | escape analysis / leaking param / 内联 |
| G22 | [精通 Go pprof 性能剖析](./G22-精通-Go-pprof-性能剖析.md) | ⭐⭐⭐⭐⭐ | CPU / heap / goroutine / block / mutex profile |
| G23 | [精通 Go runtime/trace 与 go tool trace](./G23-精通-Go-runtime-trace.md) | ⭐⭐⭐⭐ | 时间线 / MMU / region / task |
| G24 | [精通 Go 反射（reflect）](./G24-精通-Go-反射-reflect.md) | ⭐⭐⭐⭐ | Type / Value / 三定律 / 50-100x 开销 |
| G25 | [精通 Go unsafe 包与 //go:linkname](./G25-精通-Go-unsafe-与-linkname.md) | ⭐⭐⭐⭐⭐ | unsafe.Pointer / 六条合法转换 / Pinner |
| G26 | [精通 CGO：Go 调用 C 的代价与陷阱](./G26-精通-Go-CGO.md) | ⭐⭐⭐⭐⭐ | ~200ns/调用 / 内存所有权 / 信号 / 交叉编译 |
| G27 | [精通 Go net/http](./G27-精通-Go-net-http.md) | ⭐⭐⭐⭐ | ServeMux / middleware / 超时 / graceful shutdown |
| G28 | [精通 gRPC 与 Protobuf](./G28-精通-Go-gRPC-与-Protobuf.md) | ⭐⭐⭐⭐ | 四种 RPC / interceptor / status code / metadata |
| G29 | [精通 Go 数据库访问](./G29-精通-Go-数据库访问.md) | ⭐⭐⭐⭐ | database/sql / pgx / sqlc / N+1 / 连接池 |
| G30 | [精通 Go 结构化日志](./G30-精通-Go-结构化日志.md) | ⭐⭐⭐ | slog / zap / zerolog / 采样 / OTel |

---

## 🗺️ 按模块组织

### 🟢 模块一：语言基础（G01-G10）

> 从语法到类型系统，覆盖每一个 Go 程序员都该掌握的核心概念。

- **G01-G02 标量与字符串**：变量声明的语义、零值、无类型常量、UTF-8 字符串底层
- **G03-G05 复合类型**：切片三元组、Map 哈希表、Struct 对齐与嵌入
- **G06-G07 函数与方法**：闭包捕获、defer 三种实现、值/指针接收者权衡
- **G08-G09 抽象**：接口的 itab 结构、泛型的类型集与 GC shape
- **G10 错误**：Go 1.13+ wrapping、Sentinel vs 自定义类型、panic 边界

### 🔵 模块二：并发（G11-G15）

> Go 招牌特性，从原子积木到生产级模式。

- **G11 Goroutines**：GMP 调度器、栈伸缩、syscall 解绑
- **G12 Channels**：hchan 结构、select 随机化、nil channel 妙用
- **G13 sync**：Mutex 自旋 + 饥饿模式、RWMutex、WaitGroup、Once、Pool、atomic
- **G14 context**：树形 cancel/deadline/value 传播
- **G15 模式 cookbook**：pipeline、fan-in/out、worker pool、semaphore、errgroup、heartbeat、future、rate limiter

### 🟡 模块三：工程化（G16-G19）

> 让代码可信赖：检测、依赖、测试、benchmark。

- **G16 Race Detection**：TSan 原理、Happens-before 关系
- **G17 Modules**：go.mod 全指令、MVS 算法、replace 与 vendor、私有仓库
- **G18 测试**：table-driven、subtests、parallel、httptest、fake vs mock、fuzz
- **G19 Benchmarking**：testing.B、防编译器消除、benchstat 统计对比

### 🔴 模块四：性能与底层（G20-G26）

> 把 Go 性能榨干所需的全部知识。

- **G20 内存管理**：分配器三层结构、三色标记、写屏障、GOGC、GOMEMLIMIT
- **G21 逃逸分析**：六条规则、读 `-gcflags=-m` 输出、内联影响
- **G22 pprof**：五种 profile（CPU/heap/goroutine/block/mutex）的采集与解读
- **G23 runtime/trace**：时序、调度延迟、MMU、用户 region/task
- **G24 反射**：Type/Value 内部结构、三定律、50-100x 性能成本
- **G25 unsafe**：六条合法转换、//go:linkname、Pinner
- **G26 CGO**：~200ns 开销、内存所有权、信号交互、交叉编译影响

### 🟠 模块五：生态（G27-G30）

> 生产级 Go 服务的最后一公里。

- **G27 net/http**：ServeMux 增强（Go 1.22+）、middleware 链、四个超时、graceful shutdown
- **G28 gRPC**：protoc 流程、四种 RPC 模式、interceptor、status code、metadata
- **G29 数据库**：连接池、事务模式、pgx、sqlc、GORM、N+1
- **G30 日志**：slog（标准库）/ zap / zerolog 对比、采样、OTel 集成

---

## 🎯 学习路径建议

### 路径 A：从零到生产（半年-1 年）

按编号顺序通读，每周 2-3 篇。每篇配套：
1. 在脑中过一遍引言代码，预测输出
2. 通读章节，关键点动手敲一遍
3. 做完练习题
4. 看小结表自检
5. 一周后用闪卡复习关键概念

### 路径 B：性能特化（2-3 个月）

如果已经会写 Go 但要进阶到"高性能":
- 先扫 **G05**（Struct 对齐）、**G07**（指针）、**G08**（接口装箱）
- 重点 **G20**（GC）→ **G21**（逃逸）→ **G19**（benchmark）→ **G22**（pprof）→ **G23**（trace）
- 配合 **G13**（sync）、**G24**（reflect）、**G25**（unsafe）
- 实战项目用 pprof + benchstat 优化一个真实瓶颈

### 路径 C：并发特化（1-2 个月）

并发是 Go 招牌:
- **G11**（GMP）→ **G12**（channel）→ **G13**（sync）→ **G14**（context）
- **G15**（模式 cookbook）→ **G16**（race detection）
- 实战：写一个 worker pool 库、一个 rate limiter、一个 pub/sub

### 路径 D：后端工程师（1 个月急训）

直接面试 / 入职用:
- **G01-G02**（语法），**G03-G06**（核心类型）一周
- **G08-G10**（接口、泛型、错误）三天
- **G11-G14**（goroutine、channel、sync、context）一周
- **G18**（测试）、**G27**（net/http）、**G28-G29**（gRPC/DB）一周

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)
- **生态库选型地图**：见 [Go 生态库选型地图](./libraries/Go-生态库选型地图.md)（按场景精选第三方库 + 2026 选型建议）
- **官方 roadmap**：[roadmap.sh/golang](https://roadmap.sh/golang)
- **官方文档**：[go.dev/doc](https://go.dev/doc/)
- **源码**：[github.com/golang/go](https://github.com/golang/go)（重点看 `runtime/` 和 `sync/`）

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 看逃逸决策 | `go build -gcflags="-m"` |
| 看汇编 | `go tool compile -S file.go` |
| race 检测 | `go test -race ./...` |
| 覆盖率 | `go test -coverprofile=cover.out` |
| CPU profile | `go test -cpuprofile=cpu.out -bench=.` |
| Heap profile | `curl :6060/debug/pprof/heap` |
| Goroutine profile | `curl :6060/debug/pprof/goroutine?debug=2` |
| Benchmark + memory | `go test -bench=. -benchmem -count=10` |
| 模块整理 | `go mod tidy` |
| 模块图 | `go mod graph` |
| 模块来源 | `go mod why <pkg>` |
| 二进制大小 | `go tool nm app \| sort -k 1 -h` |
| 看内联 | `go build -gcflags="-m=2"` |
| 字段对齐 | `fieldalignment -fix ./...` |
| benchmark 对比 | `benchstat old.txt new.txt` |
| GC trace | `GODEBUG=gctrace=1 ./app` |
| trace 查看 | `go tool trace trace.out` |
| pprof 浏览器 | `go tool pprof -http=:8080 cpu.out` |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 解释 nil slice 和 empty slice 在 JSON 序列化上的差异
- [ ] 说明为什么 `(*MyErr)(nil)` 装入 error 接口后 `!= nil`
- [ ] 阐述 GMP 模型 + work stealing + 异步抢占
- [ ] 设计一个可优雅退出的 worker pool（含 recover + ctx）
- [ ] 解读 `go build -gcflags=-m` 的输出
- [ ] 用 pprof 找到一个 CPU 热点 + 一个内存泄漏
- [ ] 解释 GOGC=100 与 GOMEMLIMIT=8GiB 的关系
- [ ] 写一个零分配的 string ↔ []byte 转换并说明风险
- [ ] 给一个真实业务设计 gRPC + interceptor + 错误码
- [ ] 配置生产级 net/http server（四超时 + middleware + graceful）

---

## 🆕 2026 新特性快速索引

| Go 版本 | 关键特性 | 出现在 |
|---|---|---|
| **1.23** (2024-08) | `iter.Seq` / `range over func` | G15 §13.5–13.8 |
| 1.23 | `unique` 包（字符串内化） | G02 §6.5 |
| 1.23 | `slices` / `maps` 迭代器化 | G15 §13.8 |
| 1.23 | `time.Timer/Ticker` 不再需 Stop | G14（context 章节相关） |
| **1.24** (2025-02) | **Swiss Tables map 实现**（默认） | G04 §5.5–5.7 |
| 1.24 | **泛型类型别名 GA** | G09 §1.5 |
| 1.24 | `weak` 包（弱指针） | G20（GC 相关） |
| 1.24 | `runtime.AddCleanup`（取代 SetFinalizer） | G20 |
| 1.24 | **`tool` 指令**（go.mod 登记工具） | G17 §9.5–9.7 |
| 1.24 | `os.Root`（防路径穿越） | G27（http） |
| 1.24 | `omitzero` 字段标签 | G29（DB 序列化） |
| 1.24 | FIPS 140-3 验证 | G27（TLS） |
| **1.25** (2025-08) | **container-aware GOMAXPROCS** | G11 §7.3 |
| 1.25 | **`testing/synctest` 稳定** | G18 §8.5 |
| 1.25 | **`runtime/trace.FlightRecorder`** | G22 §7.4–7.5 |
| 1.25 | **`http.CrossOriginProtection`**（CSRF） | G27 §7.5–7.7 |
| 1.25 | **`WaitGroup.Go`** | G13 §3.4 |
| 1.25 | **`reflect.TypeAssert[T]`** | G24 §6.4 |
| 1.25 | `slog.GroupAttrs` | G30 §1.5 |
| 1.25 | `encoding/json/v2`（实验） | G29 |
| 1.25 | DWARF 5 调试信息 | 工具链全局 |
| 1.25 | 移除规范中的"core types"概念 | G09 |
| **1.26** (2026-02) | **Green Tea GC 默认启用**（10–40% GC 改善） | G20 §3.5–3.8 |
| 1.26 | **CGO 调用开销 -30%** | G26 §2.1 |
| 1.26 | `new(int64(300))` 表达式 | G01 |
| 1.26 | 自引用泛型类型 | G09 |
| 1.26 | `crypto/hpke`（含后量子 KEM） | G27 / B03 |
| 1.26 | **goroutineleak profile** | G16 §6.5 |
| 1.26 | `pprof` 火焰图默认 UI | G22 §7.6 |
| 1.26 | `slog.NewMultiHandler` | G30 §1.5 |
| 1.26 | `go fix` 重做（代码现代化器） | G17 / G18 |
| 1.26 | `os.File.End` | 标准库 |
| 1.26 | 实验 `simd/archsimd`、`runtime/secret` | G25 |
| 1.26 | `cmd/doc` 移除（用 `go doc`） | G17 |

权威来源：[Go 1.23](https://go.dev/doc/go1.23) · [Go 1.24](https://go.dev/doc/go1.24) · [Go 1.25](https://go.dev/doc/go1.25) · [Go 1.26](https://go.dev/doc/go1.26)。

---

> 🔁 反馈：发现错误或建议 → 改进，写完每篇都建议跑一遍代码验证
