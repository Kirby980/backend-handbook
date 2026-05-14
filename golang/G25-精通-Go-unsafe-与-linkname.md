# 精通 Go unsafe 包与 //go:linkname

> 课程编号：G25
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Unsafe 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：可控的危险

```go
import "unsafe"

s := "hello"
p := (*[5]byte)(unsafe.Pointer(unsafe.StringData(s)))
fmt.Println(p[0])   // 104 ('h')
```

`unsafe` 包让你拿到 Go 类型系统外的能力——指针运算、类型重新解释、绕过逃逸分析。代价是：失去 Go 的内存安全保证，破坏 GC 不变式可能导致段错误甚至安全漏洞。本章拆开它的合法用法、危险边界、何时该用以及通常该回避。

---

## 第一章：unsafe 提供什么

### 1.1 类型与函数

```go
type Pointer *ArbitraryType

func Sizeof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Alignof(x ArbitraryType) uintptr

// Go 1.17+
func Add(ptr Pointer, len IntegerType) Pointer
func Slice(ptr *ArbitraryType, len IntegerType) []ArbitraryType
func SliceData(slice []ArbitraryType) *ArbitraryType
// Go 1.20+
func String(ptr *byte, len IntegerType) string
func StringData(str string) *byte
```

### 1.2 unsafe.Pointer 的能力

普通 Go 中：

- `*int` 和 `*string` 互转 → 编译错误
- 指针不能做算术
- 不能"重新解释"内存

`unsafe.Pointer` 是**任何指针类型可互转的中转站**。

```go
var x int64 = 42
p := unsafe.Pointer(&x)
f := *(*float64)(p)   // 把 int64 字节按 float64 解释
```

### 1.3 与 uintptr 的区别

```go
ptr  unsafe.Pointer   // 仍是指针，GC 追踪
addr uintptr          // 只是个数字，不被 GC 追踪
```

`uintptr` 把指针变成数字——一旦如此，**GC 移动该对象时不会更新这个值**，原指针可能失效。

---

## 第二章：六条合法用法（Go 文档规范）

`unsafe.Pointer` 文档列出 **6 种合法转换模式**。其他用法都是未定义行为。

### 2.1 转换为另一种指针类型 + 立即读写

```go
var x int = 42
y := *(*uint32)(unsafe.Pointer(&x))   // 合法（前提：大小一致）
```

### 2.2 转 uintptr 立即用于地址打印

```go
fmt.Printf("%x\n", uintptr(unsafe.Pointer(&x)))
```

### 2.3 转 uintptr 立即做指针算术再转回 Pointer

```go
p := unsafe.Pointer(uintptr(unsafe.Pointer(&arr[0])) + offset)
```

注意：**整个表达式必须原子**——中间不能有函数调用、不能存到变量。

```go
// ❌ 错误：分两步
a := uintptr(unsafe.Pointer(&arr[0]))
b := a + offset
p := unsafe.Pointer(b)
// GC 可能在中间搬迁 arr，b 失效
```

### 2.4 与 syscall.Syscall 配合

```go
syscall.Syscall(SYS_X, uintptr(unsafe.Pointer(&buf[0])), uintptr(len(buf)), 0)
```

`syscall.Syscall` 的参数声明为 uintptr，但 runtime 特殊处理：在 syscall 期间 pin 对应的指针。

### 2.5 reflect.Value.Pointer / UnsafePointer 转回

```go
v := reflect.ValueOf(&x)
p := unsafe.Pointer(v.Pointer())
```

### 2.6 reflect.SliceHeader / StringHeader（已 deprecated）

旧代码：

```go
sh := (*reflect.SliceHeader)(unsafe.Pointer(&slice))
sh.Data = ...
```

**Go 1.20+ 推荐**用 `unsafe.SliceData` / `unsafe.StringData`，因为 SliceHeader 字段类型不安全（Data 是 uintptr）。

---

## 第三章：典型用例

### 3.1 string ↔ []byte 零拷贝

```go
// Go 1.20+
func b2s(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}

func s2b(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}
```

**前提条件**：
- `b2s`：返回的 string 期间，b 不被修改
- `s2b`：永远不修改返回的 slice（字符串可能在只读段）

违反 → 段错误 / 哈希异常 / 安全漏洞。

### 3.2 访问私有字段

```go
type Foo struct{ private int }   // 包外不可见

// 在别的包
f := &Foo{}
p := unsafe.Pointer(f)
private := (*int)(unsafe.Add(p, 0))   // 通过偏移读
*private = 42
```

非常脆——版本升级字段顺序变就崩。仅用于 fast JSON 库等极致优化。

### 3.3 Slice header 修改

```go
// 把 []byte 的底层数组共享给 []float32（前提大小匹配）
b := []byte{...}
f := unsafe.Slice((*float32)(unsafe.Pointer(unsafe.SliceData(b))), len(b)/4)
```

零拷贝重新解释。

### 3.4 atomic 操作 64 位字段

Go 1.19 之前 `atomic.AddInt64` 要求 8 字节对齐；某些 struct 中可能不齐。用 `unsafe.Alignof` 检查 + 调整字段顺序。

Go 1.19+ `atomic.Int64` 自动对齐，不再需要 unsafe。

---

## 第四章：unsafe.Sizeof / Alignof / Offsetof

### 4.1 编译期常量

```go
const size = unsafe.Sizeof(int(0))      // 8（64 位）
const align = unsafe.Alignof(int(0))    // 8
```

这三个函数返回**编译期常量**，可用于 const 表达式：

```go
type S struct{ A int; B byte }
const offsetB = unsafe.Offsetof(S{}.B)  // 8
```

### 4.2 计算 struct 内存

```go
type User struct {
    Name string
    Age  int
}
fmt.Println(unsafe.Sizeof(User{}))   // 24（16 + 8）
```

用于内存优化（见 G05）。

---

## 第五章：unsafe.Add 与 unsafe.Slice

### 5.1 unsafe.Add（Go 1.17+）

```go
p := unsafe.Pointer(&arr[0])
q := unsafe.Add(p, 16)   // p + 16 字节
```

比 `unsafe.Pointer(uintptr(p) + 16)` 更安全——`Add` 是原子操作，避免 uintptr 漂移期间 GC 的问题。

### 5.2 unsafe.Slice（Go 1.17+）

```go
p := (*int)(C.malloc(C.size_t(n * 8)))
s := unsafe.Slice(p, n)   // 把 C 内存包装成 []int
```

把 unsafe.Pointer 包装成 slice，跟 cgo 配合常用。

---

## 第六章：//go:linkname

### 6.1 用法

```go
package myapp

import _ "unsafe"   // 必须

//go:linkname runtimeNanotime runtime.nanotime
func runtimeNanotime() int64
```

`//go:linkname` 让你"借用"其他包（包括 runtime）的私有函数。

### 6.2 必须 `import _ "unsafe"`

linkname 是 unsafe 范畴。即使代码本身没用 unsafe.Pointer，也要导入。

### 6.3 风险

- 链接到 runtime 私有 API → Go 版本升级可能 break
- 编译时不报错，运行时找不到符号会 panic
- 标准库不保证 linkname 目标 API 稳定

### 6.4 真实用例

- `sync` 包内部用 linkname 访问 runtime semaphore
- 高性能日志库（zap、zerolog）有时用 linkname 拿 runtime 内部 nanotime
- `gjson` / `fastjson` 用 linkname 访问字符串内部

### 6.5 用户代码尽量避免

升级 Go 版本会突然崩溃——除非性能极致 + 你愿意维护。

---

## 第七章：unsafe 的 GC 陷阱

### 7.1 uintptr 不被 GC 跟踪

```go
addr := uintptr(unsafe.Pointer(&obj))
runtime.GC()
// addr 现在可能指向回收后的内存
p := unsafe.Pointer(addr)   // 错误
```

### 7.2 不要"持有 uintptr 一段时间"

uintptr 看起来像 8 字节整数，但实际上**违反 GC 不变式**。规则：uintptr 只在**当行表达式**里短暂用，立刻转回 Pointer。

### 7.3 GC 移动内存吗

Go 当前的 GC **不移动堆对象**——堆地址是稳定的。但栈对象**会移动**（栈伸缩），所以指向栈对象的 unsafe.Pointer 转 uintptr 危险。

Future-proof：永远不要假设地址不变。

---

## 第八章：何时该用 unsafe

### 8.1 应该用

- **CGO 互操作**：与 C 库交换内存
- **极致性能优化**（已 benchmark 证明 + 没其他办法）
- **runtime/syscall 级别接口**
- **零拷贝字符串/字节切片转换**（小心使用）

### 8.2 通常不该用

- 绕过类型系统省事
- "看起来更快"但没 benchmark
- 修改私有字段
- 实现 hack 工具

### 8.3 替代方案

| 想做的事 | 安全替代 |
|---|---|
| 通用容器 | 泛型 |
| 类型分发 | type switch / interface |
| 反射 | 代码生成 |
| 内存复用 | sync.Pool |
| 零拷贝 string/[]byte | Go 1.20+ unsafe.String/Slice（仍 unsafe 但官方） |

---

## 第九章：常见生产示例

### 9.1 字符串拷贝优化（fastjson 风格）

```go
func appendField(buf []byte, val string) []byte {
    sh := (*[2]uintptr)(unsafe.Pointer(&val))
    bh := (*[3]uintptr)(unsafe.Pointer(&buf))
    // ... 直接操作 header
}
```

复杂、易错，但 fastjson 这样做能比标准库快 5 倍。

### 9.2 atomic.Value 内部

```go
type Value struct {
    v any
}
```

atomic.Value 用 unsafe 直接 atomic swap interface header（16 字节），实现原子配置广播。

### 9.3 mmap

```go
data, _ := syscall.Mmap(fd, 0, size, syscall.PROT_READ, syscall.MAP_SHARED)
recs := unsafe.Slice((*Record)(unsafe.Pointer(&data[0])), len(data)/sizeofRecord)
```

把 mmap 区域当成 Record 数组操作。

---

## 第十章：生产级最佳实践

1. **能不用 unsafe 就不用**——80% 的"必要"实际上不必要。
2. **每处 unsafe 必须注释**：解释为什么必须 + 不变式。
3. **写覆盖 unsafe 路径的测试 + race + fuzz**。
4. **Go 版本升级时人工 review 所有 unsafe**。
5. **不要把 uintptr 存为字段或局部变量**：仅做表达式内传递。
6. **`unsafe.Add` / `Slice` / `String` 是 1.17+ 的安全替代**：优先用。
7. **避免 //go:linkname**：除非性能极致。
8. **CGO 是 unsafe 的合法领域**：但仍要小心生命周期。
9. **benchmark 证明收益再引入 unsafe**：否则徒增复杂度。
10. **公开库别在 API 中暴露 unsafe.Pointer**：让调用方头疼。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：uintptr 存到变量
```go
addr := uintptr(unsafe.Pointer(&x))   // x 可能被 GC
```

### ❌ 陷阱 2：unsafe.Pointer 转换大小不匹配
```go
var b byte = 1
p := *(*int)(unsafe.Pointer(&b))   // 读 8 字节但 b 只有 1
```

### ❌ 陷阱 3：修改只读 string 底层
```go
s := "hello"
p := unsafe.StringData(s)
*p = 'H'   // 段错误（.rodata 段）
```

### ❌ 陷阱 4：linkname 到非稳定 API
比如 linkname 到 `runtime.fastrand`——Go 1.22 已改名 `runtime.cheaprand`，1.24 又对外暴露了 `math/rand/v2`。每次升级都可能找不到符号 → 程序 crash。**Go 1.23 起 linker 也开始限制对 std 内部符号的 linkname**，不在白名单的 std 包符号 linkname 直接编译失败（除非 `-linkname-allow-list` 开口）。结论：**只 linkname 公开 API；私有符号视为随时会变**。

### ❌ 陷阱 5：CGO 调用后 Go GC 移动栈
传给 C 的指针生命周期内 Go runtime 保证 pin；但越过 syscall boundary 后规则复杂，遵守 cgo 文档。

### ❌ 陷阱 6：跨 goroutine 共享 unsafe.Pointer 不同步
普通 race 之外，type 重新解释让 race 后果更不可控。

### ❌ 陷阱 7：用 unsafe 修改"看起来 final"的值
比如修改 string 让两个变量内容不一致——所有依赖 string 不可变性的代码（map key、interned）失效。

---

## 第十二章：练习题

**练习 1**：用 unsafe 实现一个 `BytesToString(b []byte) string` 零拷贝版本，并说明使用约束。

**练习 2**：以下代码合法吗？
```go
a := uintptr(unsafe.Pointer(&x))
b := a + 8
runtime.GC()
p := unsafe.Pointer(b)
```

**练习 3**：为什么 `unsafe.Sizeof` 是编译期常量而 `len(slice)` 是运行时？

**练习 4**：用 unsafe.Offsetof 写一个函数，给定 struct 实例和字段偏移，读出该字段的 int 值。

**练习 5**：解释 `//go:linkname` 为何需要 `import _ "unsafe"`。

---

## 参考答案

**练习 1**：
```go
func BytesToString(b []byte) string {
    if len(b) == 0 { return "" }
    return unsafe.String(unsafe.SliceData(b), len(b))
}
```
约束：返回的 string 只在 b 未被修改时合法。修改 b → 字符串内容跟着变（违反不可变性）。

**练习 2**：不合法。GC 在 a → b 之间可能搬迁 x（如果 x 在栈），b 失效。规则：uintptr 只在原子表达式中使用，不能跨函数调用、不能存到变量。

**练习 3**：Sizeof 仅依赖类型（编译期已知）。slice 的 len 是 runtime 信息（slice header）。

**练习 4**：
```go
func FieldAtOffset(p unsafe.Pointer, off uintptr) int {
    return *(*int)(unsafe.Add(p, off))
}
```

**练习 5**：linkname 是 unsafe 范畴的功能。Go 编译器强制要求显式 import 提醒：你在用底层机制，要承担责任。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| unsafe.Pointer | 任意指针互转中转站 |
| uintptr | 数字；GC 不追；只能短暂用 |
| 六种合法转换 | 严格遵守，否则 UB |
| Sizeof/Alignof/Offsetof | 编译期常量 |
| Add/Slice/String | Go 1.17+/1.20+ 的安全替代 |
| //go:linkname | 借用私有 API；Go 版本敏感 |
| 用 vs 不用 | CGO + 极致性能用；其他场景避免 |

下一篇 **G26 — 精通 CGO** 会讲清 Go 调 C 的 ABI、内存所有权、性能成本、信号/线程交互、build 配置。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G25-精通-Go-unsafe-与-linkname.md`
