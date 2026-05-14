# 精通 CGO：Go 调用 C 的代价与陷阱

> 课程编号：G26
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — CGO 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：CGO 是把双刃剑

```go
package main

/*
#include <stdio.h>
void hello() { printf("Hello from C\n"); }
*/
import "C"

func main() { C.hello() }
```

10 行代码桥接 Go 与 C。CGO 让你能用 libssl、libsqlite3、libcuda、glibc syscall——大量"基础设施"。但每次 CGO 调用约 200ns 开销，且会失去 Go 的栈伸缩、调度协作、GC 协助。本章拆开 CGO 的代价、内存模型、build 流程、何时该用。

---

## 第一章：CGO 基础

### 1.1 import "C"

```go
/*
#include <stdio.h>
#include <stdlib.h>
*/
import "C"
```

注释里写 C 代码（#include、function、struct）。**注释必须紧邻 `import "C"`**——之间不能有空行。

### 1.2 调用 C 函数

```go
/*
int add(int a, int b) { return a + b; }
*/
import "C"

func main() {
    r := C.add(C.int(3), C.int(5))
    fmt.Println(int(r))   // 8
}
```

`C.add` 是 Go 中的函数。参数和返回值要做类型转换（`C.int(3)`）。

### 1.3 调用 C 库

```go
/*
#cgo LDFLAGS: -lcrypto
#include <openssl/sha.h>
*/
import "C"
```

`#cgo` 指令配置编译/链接：
- `CFLAGS`：C 编译选项
- `LDFLAGS`：链接选项
- `pkg-config`：使用 pkg-config

### 1.4 string 传递

```go
/*
#include <string.h>
*/
import "C"

func length(s string) int {
    cs := C.CString(s)        // 分配 + 复制（C heap）
    defer C.free(unsafe.Pointer(cs))
    return int(C.strlen(cs))
}
```

`C.CString` **在 C 堆分配并复制** Go 字符串到 C。**必须** `C.free`，否则内存泄漏。

反向：`C.GoString(*C.char)` 把 C 字符串拷贝到 Go string。

### 1.5 字节切片

```go
b := []byte{0xde, 0xad, 0xbe, 0xef}
C.write_bytes((*C.uchar)(unsafe.Pointer(&b[0])), C.size_t(len(b)))
```

直接传 Go slice 的底层指针——但要保证 C 函数**返回前不再让 Go 调度移动这块内存**。这一点 CGO runtime 会处理（pinning）。

---

## 第二章：调用开销

### 2.1 一次 cgo call 约 200ns（Go 1.25 及之前）

包括：
- 切换栈（Go 栈 → C 栈）
- 调度器通知（这个 M 即将进入 C 代码）
- 释放当前 P 给其他 G（M-P 解绑）
- C 函数实际执行
- 返回时重新拿 P / 复用 M

> **Go 1.26 把基线 cgo 调用开销降低了约 30%**。官方实测从 ~200ns 降到 ~140ns 量级（具体值随平台和工作负载浮动）。这对每秒成千上万次小调用的场景（如 SQLite 包装、`crypto/openssl` 桥）效果明显。底层做法：精简了 syscall.cgocall 路径上的栈管理与 P 解绑/再绑；细节见 [Go 1.26 release notes — Runtime](https://go.dev/doc/go1.26#runtime)。

### 2.2 对比直接调用

```
直接 Go 调用:          1-2 ns
CGO 调用:              ~200 ns
完整 syscall (Go):     ~300 ns
完整 syscall (CGO):    ~500 ns
```

热路径每秒 1000 万次调用 → CGO 直接增加 2 秒/秒 → 不可承受。

### 2.3 优化思路：批处理

```go
// ❌ 每次循环 cgo
for _, x := range items { C.process(x) }

// ✅ 一次调用处理整个 slice
C.process_batch((*C.int)(unsafe.Pointer(&items[0])), C.size_t(len(items)))
```

把 N 次调用合并为 1 次，开销摊平到 N 个元素上。

---

## 第三章：内存所有权

### 3.1 三类内存

- **Go 内存**：Go runtime 管理，GC 自动回收
- **C 内存**：malloc 分配，必须显式 free
- **栈内存**：Go 栈或 C 栈

### 3.2 Go → C 传指针的规则

Go 1.x cgo pointer passing rules（简化）：

1. **Go 内存可以传给 C**——只要 C 函数返回前不存储该指针超出调用范围
2. **C 内存任意持有**——Go GC 不管 C 堆
3. **不能传"包含 Go 指针的 Go 内存"给 C**（嵌套指针约束）

```go
// ❌ 违反规则
type T struct { ptr *int }
ptr := &T{ptr: new(int)}
C.func(unsafe.Pointer(ptr))   // 嵌套 Go 指针
```

### 3.3 runtime.Pinner（Go 1.21+）

```go
var pinner runtime.Pinner
defer pinner.Unpin()

x := &MyStruct{}
pinner.Pin(x)   // 在解锁前，x 不被 GC 移动
C.processAsync(unsafe.Pointer(x))   // 长时间持有 OK
```

之前 cgo 长期持有 Go 指针只能用 `cgo.Handle`（不透明 ID）。Pinner 提供更直接的 pinning。

### 3.4 cgo.Handle

```go
import "runtime/cgo"

h := cgo.NewHandle(myGoObject)
defer h.Delete()
C.passHandle(C.uintptr_t(h))

// C 侧通过 handle 调回 Go：
// extern void cgo_callback(uintptr_t h);
//export goCallback
func goCallback(h C.uintptr_t) {
    obj := cgo.Handle(h).Value().(*MyType)
}
```

适合需要在 C 侧长期保存"指向 Go 对象的标识"。

---

## 第四章：goroutine 与线程

### 4.1 M-P 解绑

调用 C 时该 M 可能阻塞（看 C 函数是否长），runtime 解绑 P 给别的 M。**结果**：CGO 大量调用时实际 OS 线程数 >> GOMAXPROCS。

### 4.2 C 调 Go callback

```c
// C 侧
extern void GoCallback(int n);
void some_lib_func(void (*cb)(int)) { cb(42); }
```

```go
//export GoCallback
func GoCallback(n C.int) {
    fmt.Println("got", n)
}

C.some_lib_func((*[0]byte)(C.GoCallback))
```

C 调 Go 时 runtime 切换回 Go 栈、可能创建新 goroutine（如果 callback 在新线程）。

### 4.3 信号处理

C 库可能注册 SIGSEGV / SIGFPE handler。Go runtime 也用这些信号（GC、preemption）。冲突 → 各种诡异 bug。

CGO 启动时 Go runtime 会"接管"信号，但 C 库注册的 handler 不可控。要避免在 Go 代码里依赖 C 信号机制。

### 4.4 runtime.LockOSThread

部分 C 库要求"必须在同一线程调用"（OpenGL、OS X UI）。

```go
runtime.LockOSThread()
defer runtime.UnlockOSThread()
C.gl_create_context()
// ... 后续 GL 调用都在这个线程
```

---

## 第五章：build 配置

### 5.1 启用 / 禁用 CGO

```bash
CGO_ENABLED=1 go build   # 默认，开
CGO_ENABLED=0 go build   # 关，纯 Go
```

`CGO_ENABLED=0` + 静态链接 → 适合 docker `FROM scratch` 镜像。

### 5.2 交叉编译

```bash
CGO_ENABLED=1 GOOS=linux GOARCH=arm64 go build
```

要求**目标平台的 C 工具链**（如 `aarch64-linux-gnu-gcc`）。这是为什么交叉编译加 CGO 总是头疼。

无 CGO 的交叉编译只需 `GOOS=...GOARCH=...`，零 toolchain 依赖——这是 Go 标志性特性。

### 5.3 pkg-config

```go
/*
#cgo pkg-config: openssl
#include <openssl/ssl.h>
*/
import "C"
```

让 cgo 调用 `pkg-config --cflags --libs openssl` 自动获取。

### 5.4 静态链接 C 库

```go
/*
#cgo LDFLAGS: -static -lsomething
*/
import "C"
```

生成的二进制不依赖 `.so`。但兼容性问题：glibc 静态链接 fragile。

---

## 第六章：典型应用场景

### 6.1 调用现有 C 库

```go
/*
#cgo pkg-config: libcurl
#include <curl/curl.h>
*/
import "C"
```

复用 libcurl 的全部能力（HTTP、HTTPS、FTP……）。

### 6.2 SQLite

`mattn/go-sqlite3` 是 CGO 封装。性能好、单文件数据库。但失去交叉编译便利。

### 6.3 GPU / CUDA

`gocuda` 等库通过 CGO 调用 CUDA runtime。深度学习推理服务常用。

### 6.4 系统调用扩展

Linux 一些 syscall（io_uring、eBPF）目前 Go 没有原生支持，要 CGO 调 glibc 包装。

### 6.5 嵌入 C 实现高性能算法

复杂算法（图像编解码、压缩、加密）已有高度优化的 C 实现，cgo 直接复用。

---

## 第七章：cgo 的代价清单

### 7.1 编译时

- 需要 C 工具链
- 编译时间大幅增加（5-10 倍）
- 二进制体积增大
- 交叉编译困难

### 7.2 运行时

- 每次调用 ~200ns
- 多余线程（M 增多）
- GC 不管 C 内存
- 调试复杂（栈跨语言）

### 7.3 维护

- 内存泄漏难追
- 平台特定问题（Linux/macOS/Windows C 库不同）
- 升级 C 库 API 变化

### 7.4 安全

- C 代码漏洞（buffer overflow、format string、UAF）
- AddressSanitizer 配置麻烦

---

## 第八章：替代方案

### 8.1 纯 Go 重写

很多 C 库有 Go 等价：
- libsqlite3 → `modernc.org/sqlite`（纯 Go）
- libcrypto 大多 → 标准库 crypto
- libcurl → 标准库 net/http
- libz → compress/gzip

性能可能略差，但避免 CGO 代价。

### 8.2 子进程

```go
cmd := exec.Command("./c_program", "arg")
out, _ := cmd.Output()
```

启动子进程慢（毫秒级），但完全隔离。适合"偶尔调用"的重量级任务。

### 8.3 gRPC / IPC

把 C 代码做成独立服务，Go 通过 gRPC 调用。开销大但松耦合。

### 8.4 WebAssembly

某些 C 库可编译为 wasm，Go 通过 wasmtime 运行。新兴方向。

---

## 第九章：生产级最佳实践

1. **能纯 Go 就纯 Go**：交叉编译、镜像大小、维护成本都更优。
2. **必须 CGO 时把调用集中化**：批处理 + 限制调用频率。
3. **C 内存严格 free**：每个 CString 配 free。
4. **`runtime.Pinner` 替代 cgo.Handle**（Go 1.21+）。
5. **CGO_ENABLED=0 构建容器镜像**：减小镜像 + 静态二进制。
6. **不要在热点路径用 CGO**：200ns × 百万次 = 灾难。
7. **valgrind / ASAN 测试 C 部分**：捕获泄漏与越界。
8. **C 代码独立目录 + cgo wrapper 文件**：分离关注。
9. **避免 C 调 Go 的回调**：复杂、易出 bug。
10. **生产监控 OS 线程数**：CGO 多 → 线程多 → 注意线程上限。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：忘记 free CString
```go
cs := C.CString(s)
C.func(cs)
// 没 C.free → 泄漏
```

### ❌ 陷阱 2：跨语言指针生命周期
```go
go_obj := &T{}
C.save_pointer(unsafe.Pointer(go_obj))
// C 之后用：可能已被 GC
```

### ❌ 陷阱 3：cgo 调用在循环
每次 200ns × N 次。

### ❌ 陷阱 4：C 库有全局状态
全局 mutex、连接池——多 goroutine 调用未必线程安全。

### ❌ 陷阱 5：交叉编译挂了
没装目标平台 C 工具链。

### ❌ 陷阱 6：信号冲突
C 库改了 SIGSEGV handler → Go runtime 失常。

### ❌ 陷阱 7：嵌套 Go 指针
struct 含 Go 指针字段，整体传给 C → cgo runtime 检测到会 panic（默认 `GOEXPERIMENT=cgocheck`）。

---

## 第十一章：练习题

**练习 1**：用 CGO 调 C 标准库 `qsort` 排序 `[]int`，要求最小化 cgo 调用次数。

**练习 2**：解释为什么 `CGO_ENABLED=0` 后 `net` 包某些功能受限。

**练习 3**：以下代码有何问题？
```go
func tryC(s string) {
    cs := C.CString(s)
    C.use(cs)
}
```

**练习 4**：写一个 wrapper：把 C 函数 `int sum(int* arr, size_t n)` 暴露给 Go 用户。

**练习 5**：解释 cgo 调用为什么会"解绑 P"。

---

## 参考答案

**练习 1**：
```go
/*
#include <stdlib.h>
static int cmp(const void* a, const void* b) {
    return *(int*)a - *(int*)b;
}
static void sort_ints(int* arr, size_t n) { qsort(arr, n, sizeof(int), cmp); }
*/
import "C"

func Sort(s []int) {
    C.sort_ints((*C.int)(unsafe.Pointer(&s[0])), C.size_t(len(s)))
}
```
仅 1 次 cgo 调用，C 内部完成所有比较。

**练习 2**：`net` 默认用 glibc 的 DNS 解析（CGO）；CGO_ENABLED=0 强制走纯 Go resolver——某些场景（nsswitch.conf 配置）行为不同。

**练习 3**：cs 没 free → 内存泄漏。修：
```go
cs := C.CString(s)
defer C.free(unsafe.Pointer(cs))
C.use(cs)
```

**练习 4**：
```go
/*
int sum(int* arr, size_t n);
*/
import "C"

func Sum(xs []int) int {
    if len(xs) == 0 { return 0 }
    return int(C.sum((*C.int)(unsafe.Pointer(&xs[0])), C.size_t(len(xs))))
}
```

**练习 5**：C 函数可能任意长地阻塞。runtime 不能让 P 等它，所以解绑 P 给别的 M-G 跑。等 C 返回后该 M 重新拿 P（或入闲置池）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 启用 | `import "C"` + 注释 + CGO_ENABLED=1 |
| 开销 | ~200ns/调用；批处理摊平 |
| 内存 | Go 内存可短期传 C；不能嵌套 Go 指针 |
| 字符串 | CString/GoString 都复制 |
| Pinner | Go 1.21+ 长期持有 |
| 线程 | C 调用可能解绑 P；LockOSThread |
| build | 失去交叉编译便利；体积大 |
| 替代 | 纯 Go / 子进程 / gRPC |

下一篇 **G27 — 精通 Go net/http** 会拆开 ServeMux、Handler/Middleware 模式、超时配置、graceful shutdown、生产 server 经验。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G26-精通-Go-CGO.md`
