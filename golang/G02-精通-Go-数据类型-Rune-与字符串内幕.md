# 精通 Go 的数据类型、Rune 与字符串内幕

> 课程编号：G02
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Data Types & Strings 章节
> 难度：⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：三道开胃题

```go
package main

import "fmt"

func main() {
    s := "中文"
    fmt.Println(len(s))          // ① 输出什么？
    fmt.Println(len([]rune(s)))  // ② 输出什么？

    var a int32 = 1
    var b int = 1
    fmt.Println(a + b)           // ③ 编译错误，为什么？
}
```

如果你能立刻说出三个答案以及背后原因，可以跳到第七章看实践要点；否则，这篇值得花一杯咖啡的时间。Go 的数据类型表面"和 C 一样简单"，但藏着 UTF-8、IEEE 754、平台尺寸差异、不可变字符串、rune 与 byte 哲学等多个细节，它们是日后排查"乱码""精度丢失""跨平台编译失败"等问题的钥匙。

---

## 第一章：整数类型——别再以为 int 就是 int32

### 1.1 类型清单

| 类型 | 大小（字节） | 范围 |
|---|---|---|
| `int8` / `uint8` | 1 | -128~127 / 0~255 |
| `int16` / `uint16` | 2 | -32768~32767 / 0~65535 |
| `int32` / `uint32` | 4 | ±2.1e9 / 0~4.3e9 |
| `int64` / `uint64` | 8 | ±9.2e18 / 0~1.8e19 |
| `int` / `uint` | **平台相关**（32 位机 4 字节，64 位机 8 字节） | 等同 int32 或 int64 |
| `uintptr` | 与指针等宽 | 用于 unsafe |
| `byte` | 1 | `uint8` 别名 |
| `rune` | 4 | `int32` 别名（Unicode code point） |

记住一条：**`int` 不是固定 32 位**。在 amd64/arm64 上 `int` 是 8 字节，在 386 / armv7 / wasm 上是 4 字节。这导致看似无害的代码在 32 位平台上溢出。

```go
var n int = 3_000_000_000  // ❌ 32 位平台编译失败：constant overflows int
```

`unsafe.Sizeof` 可以验证：

```go
fmt.Println(unsafe.Sizeof(int(0)))  // 64 位机：8；32 位机：4
```

### 1.2 不同整数类型之间不会隐式转换

```go
var a int32 = 1
var b int   = 1
c := a + b   // ❌ invalid operation: mismatched types int32 and int
```

即使 `int == int32` 在物理上相等，Go 也认为它们是不同类型。这是 Go 严格类型的一面，能避免 C 语言里隐式提升带来的 bug，但写起来略啰嗦。**修正**：

```go
c := int(a) + b
// 或
c := a + int32(b)
```

### 1.3 溢出语义：环绕（wrapping），不报错

```go
var x int8 = 127
x++
fmt.Println(x)  // -128
```

Go 整数溢出是**静默环绕**（C/C++ 是未定义行为，Rust 是 debug 模式 panic / release 模式环绕）。这意味着：

- 加减乘除溢出**不会** panic
- 当**具型化为有界整型**时，常量算术溢出**编译期就会报错**（无类型常量本身精度无限）
- 运行时溢出**无法自动检测**，要靠 `math/bits.Add64`、`math/bits.Mul64` 等显式 API

```go
const N = 1 << 64            // ✅ 合法：无类型整数常量精度无限，未具型化就不溢出
const M int = 1 << 64        // ❌ 编译错误：constant overflows int（强行钉到 int）
var n2 int = 1 << 64         // ❌ 编译错误：同上，赋值时具型化为 int 即溢出
// fmt.Println(N)            // ❌ 若解开本行，N 隐式具型化为默认类型 int，仍溢出

var n int64 = 1
n <<= 63
n <<= 1   // 运行时静默溢出，n 变成 0
```

### 1.4 `uint` 的潜在危险

```go
for i := uint(len(arr)) - 1; i >= 0; i-- {  // ❌ 死循环！
    process(arr[i])
}
```

当 `arr` 为空时，`len(arr) - 1` 在 `uint` 域里是一个巨大的正数（最大 uint），永远 >= 0。**经验法则**：循环变量、计数器、索引一律用 `int`，除非有协议位运算等明确理由用 `uint`。

`uint` 真正适合的场景：

- 位掩码（不需要负数概念）
- 与底层协议字段（IP 包、TLS 记录）对齐
- 字节大小、缓冲区长度（但 Go 标准库仍偏好 `int`）

---

## 第二章：浮点数——IEEE 754 的甜蜜与陷阱

### 2.1 类型

- `float32` —— 4 字节，约 7 位有效十进制位
- `float64` —— 8 字节，约 15-17 位有效十进制位（默认）

Go 中没有 `float` 简写——必须明确写出位数。`1.5` 这样的字面量默认是 `float64`。

### 2.2 == 几乎永远是错的

```go
fmt.Println(0.1 + 0.2 == 0.3)  // false
```

原因：0.1、0.2、0.3 在二进制 IEEE 754 中都无法精确表示，加法引入舍入误差。**正确做法**：

```go
const eps = 1e-9
func almostEqual(a, b float64) bool {
    return math.Abs(a-b) < eps
}
```

或针对相对误差：

```go
func relEqual(a, b, eps float64) bool {
    if a == b { return true }
    return math.Abs(a-b)/math.Max(math.Abs(a), math.Abs(b)) < eps
}
```

### 2.3 NaN、+Inf、-Inf

```go
nan := math.NaN()
fmt.Println(nan == nan)  // false ！NaN 不等于自身
fmt.Println(math.IsNaN(nan))  // true（正确判定方式）

inf := math.Inf(1)
fmt.Println(1.0 / 0.0)  // ❌ 编译错误：除数为常量 0
var zero float64 = 0
fmt.Println(1.0 / zero)  // +Inf
fmt.Println(0.0 / zero)  // NaN
```

### 2.4 不同精度的常量算术

回顾 G01：**无类型常量是任意精度**，所以 `const x = 0.1 + 0.2` 等于 `0.3`。一旦赋给 `float64`，才引入舍入。

```go
const a = 0.1 + 0.2
const b = 0.3
fmt.Println(a == b)  // true（编译期任意精度比较）

var c = 0.1 + 0.2
var d = 0.3
fmt.Println(c == d)  // false（运行时 float64 比较）
```

### 2.5 float32 vs float64

科学计算和性能敏感场景偶尔用 `float32`（如机器学习、图形）。但要注意：

- 标准库 `math` 包几乎全部用 `float64`，混用要显式转换
- `float32 → float64` 是无损的；反之有舍入
- GPU 友好但 CPU 上没有明显速度差

---

## 第三章：rune 与 byte——两个常被混淆的别名

```go
type byte = uint8     // 二进制数据的一个字节
type rune = int32     // 一个 Unicode code point
```

`byte` 和 `rune` 都是**别名**（用 `type X = Y` 而不是 `type X Y` 定义的），意味着它们和底层类型完全互换。但**语义不同**：

- 当你写 `byte`，告诉读者"我在处理二进制数据"
- 当你写 `rune`，告诉读者"我在处理字符"

```go
s := "Hello, 世界"
for i, c := range s {
    fmt.Printf("%d: %c (U+%04X)\n", i, c, c)
}
```

输出（注意索引跳跃）：

```
0: H (U+0048)
1: e (U+0065)
2: l (U+006C)
3: l (U+006C)
4: o (U+006F)
5: , (U+002C)
6:   (U+0020)
7: 世 (U+4E16)
10: 界 (U+754C)
```

`range string` 按 rune 解码 UTF-8，索引是字节位置；中文每个字 3 字节，所以索引从 7 跳到 10。

---

## 第四章：字符串的内部结构

### 4.1 StringHeader

```go
type StringHeader struct {
    Data uintptr  // 指向字节数组的指针
    Len  int      // 字节长度
}
```

在 64 位机上一个字符串 16 字节。**`len(s)` 是字节数，不是字符数**。

```go
fmt.Println(unsafe.Sizeof(""))   // 16

s := "中"
fmt.Println(len(s))                       // 3
fmt.Println(utf8.RuneCountInString(s))    // 1
```

### 4.2 字符串不可变

Go 字符串底层数据通常存放在**只读段**（`.rodata`），无法修改：

```go
s := "hello"
s[0] = 'H'  // ❌ 编译错误：cannot assign to s[0]
```

要修改，先转 `[]byte` 或 `[]rune`：

```go
b := []byte(s)
b[0] = 'H'
s2 := string(b)
```

这两次转换**都涉及内存拷贝**。

### 4.3 字符串 vs []byte 互转的代价

```go
s := "hello world"
b := []byte(s)   // 分配 11 字节并拷贝
s2 := string(b)  // 再分配 11 字节并拷贝
```

每次转换都是 O(n) 时间 + O(n) 分配。但编译器对**几个特殊场景**做了优化（避免分配）：

- `string(byteSlice)` 当 byteSlice 没有其他引用且字符串只用于比较：编译器内联，零拷贝
- `m[string(byteSlice)]` 当 byteSlice 用作 map 查找的 key：零拷贝（直接哈希）
- `for i, c := range string(byteSlice)`：零拷贝迭代

```go
buf := []byte("hello")
v, ok := cache[string(buf)]  // ✅ 编译器零分配优化
```

但只要你把转换结果赋给一个变量并保留，就一定会复制：

```go
s := string(buf)   // ❌ 复制
v := cache[s]
```

### 4.4 Go 1.20+ 的零拷贝 API

```go
// 把 []byte 转 string，无拷贝。前提：之后不再修改 b。
s := unsafe.String(unsafe.SliceData(b), len(b))

// 把 string 转 []byte，无拷贝。前提：永远不修改返回的 slice。
b := unsafe.Slice(unsafe.StringData(s), len(s))
```

这两个 API 是 `unsafe` 提供的官方"零拷贝"通道，比之前用 `reflect.StringHeader` 的小技巧更安全。但**仍然是 unsafe**：违反不变式会导致字符串底层数据被改、其他持有该字符串的代码看到诡异行为。

---

## 第五章：UTF-8 与字符串迭代

### 5.1 Go 源码强制 UTF-8

Go 规范规定**源码文件本身必须是 UTF-8**，字符串字面量也按 UTF-8 编码。`"中文"` 在源码里和字节流中都是 6 个字节（`0xE4 0xB8 0xAD 0xE6 0x96 0x87`）。

### 5.2 两种迭代方式

```go
s := "Hello,世界"

// 方式 ① 按字节
for i := 0; i < len(s); i++ {
    fmt.Printf("%d:%x ", i, s[i])
}
// 输出：0:48 1:65 2:6c 3:6c 4:6f 5:2c 6:e4 7:b8 8:96 9:e7 10:95 11:8c

// 方式 ② 按 rune
for i, r := range s {
    fmt.Printf("%d:%c ", i, r)
}
// 输出：0:H 1:e 2:l 3:l 4:o 5:, 6:世 9:界
```

**反例：截断中文字符串**

```go
s := "中文测试"
fmt.Println(s[:5])  // 截到第 5 字节，正好在某个中文字的中间，输出乱码
```

修正：

```go
runes := []rune(s)
fmt.Println(string(runes[:2]))  // 中文
```

或用 `utf8.DecodeRuneInString` 边解码边截：

```go
func truncateRunes(s string, n int) string {
    if n <= 0 { return "" }
    cnt := 0
    for i := range s {
        if cnt == n { return s[:i] }
        cnt++
    }
    return s
}
```

### 5.3 unicode/utf8 包关键函数

```go
utf8.RuneCountInString(s)              // 字符数
utf8.ValidString(s)                    // 是否为合法 UTF-8
utf8.DecodeRuneInString(s)             // 解码第一个 rune
utf8.RuneLen(r)                        // 该 rune 编码后占几字节
```

---

## 第六章：字符串拼接性能

### 6.1 五种方式

```go
// ① + 拼接
s := s1 + s2 + s3

// ② fmt.Sprintf
s := fmt.Sprintf("%s%s%s", s1, s2, s3)

// ③ strings.Join
s := strings.Join([]string{s1, s2, s3}, "")

// ④ strings.Builder（推荐）
var b strings.Builder
b.WriteString(s1)
b.WriteString(s2)
b.WriteString(s3)
s := b.String()

// ⑤ bytes.Buffer
var buf bytes.Buffer
buf.WriteString(s1); buf.WriteString(s2); buf.WriteString(s3)
s := buf.String()
```

### 6.2 性能对照（典型量级，1000 次拼接小字符串）

| 方式 | 时间 | 分配次数 | 备注 |
|---|---|---|---|
| `s += x`（循环里） | 极慢 | O(n) 次 | 每次都 alloc & copy |
| `fmt.Sprintf` | 慢 | 多次 | 反射 + 解析格式串 |
| `strings.Join` | 中等 | 1 次 | 提前算总长 |
| `strings.Builder` | 快 | 1-2 次 | 推荐 |
| `bytes.Buffer` | 快 | 1-2 次 | 旧代码常见，多了 string ↔ []byte 转换 |

`strings.Builder` 比 `bytes.Buffer` 略快，原因：`Builder.String()` 用 `unsafe` 把内部 `[]byte` 直接当 string 返回，零拷贝。而 `Buffer.String()` 必须复制（因为 Buffer 仍可被修改）。

### 6.3 Builder 的 Grow

如果能估算最终长度，先 `b.Grow(n)` 避免多次扩容：

```go
var b strings.Builder
b.Grow(1024)
for _, s := range parts {
    b.WriteString(s)
}
return b.String()
```

### 6.4 编译器优化：常量字符串拼接

```go
const prefix = "user:"
const id = "123"
const key = prefix + id  // 编译期就是 "user:123"，零运行时开销
```

### 6.5 unique 包 —— 字符串内化（Go 1.23+）

如果你的服务里有大量**重复的字符串**（HTTP header 名、tag、metric label、JSON 字段名……），每来一份请求都新分配一份相同内容的 `string`，浪费内存又给 GC 添麻烦。**Go 1.23 把字符串内化（interning）做进了标准库** `unique` 包：

```go
import "unique"

type LabelSet struct {
    Name unique.Handle[string]   // 不再持 string，持一个 8 字节 handle
    Env  unique.Handle[string]
}

func newLabel(name, env string) LabelSet {
    return LabelSet{
        Name: unique.Make(name),   // 同 name → 同 handle，全局唯一
        Env:  unique.Make(env),
    }
}

// 取回值
fmt.Println(label.Name.Value())
```

**原理**：`unique.Make` 维护一个 weak-ref 全局池。相同内容的值返回相同的 `Handle[T]`。Handle 比较 = 指针比较，O(1)；比较两个 `LabelSet` 不再扫字节。值真正不再有任何引用时（含 weak），后台异步从池里清掉。

**适用**：高频出现的有限集合（label、enum 字符串、URL path 段）。**不适用**：低重复率字符串——会因为额外哈希/锁开销得不偿失。

> 参考：[Go 1.23 release notes — unique 包](https://go.dev/doc/go1.23#unique)。

---

## 第七章：类型转换 vs 类型断言

容易混淆的两个概念：

| 概念 | 语法 | 时机 | 用途 |
|---|---|---|---|
| **类型转换** | `T(x)` | 编译期 | 同底层类型之间显式转换（int → int64、[]byte → string） |
| **类型断言** | `x.(T)` | 运行时 | 从接口取出具体类型 |

```go
var i int = 5
var f float64 = float64(i)    // 转换

var x interface{} = "hi"
s := x.(string)               // 断言；不是 string 会 panic
s, ok := x.(string)           // 安全断言
```

类型转换不允许跨"类别"，例如不能直接 `string(intVal)`（除非是单个 rune → 单字符字符串）：

```go
s := string(65)        // "A"（合法但常被滥用）
s2 := string(65)+ "X"  // "AX"
n := 12345
s3 := string(n)        // "ሙ"（U+3039 字符）——通常你想要的是 strconv.Itoa(n)
```

`go vet` 会对 `string(intVal)` 这种可疑用法警告。

---

## 第八章：生产级最佳实践

1. **循环计数、索引、长度统一用 `int`**，除非有协议约束。
2. **跨平台编译要小心 `int`**：写 `int64` 显式声明字段长度（如序列化）。
3. **浮点比较永远用 epsilon**：`math.Abs(a-b) < eps`。
4. **NaN 判断用 `math.IsNaN`**，永远不要 `== NaN`。
5. **处理字符内容用 rune，处理字节用 byte**：选 type 即文档。
6. **截断字符串先转 `[]rune` 或用 `utf8.DecodeRune`**，不要按字节切。
7. **拼接超过 3 段用 `strings.Builder`**，并 `Grow` 估容量。
8. **map key 用 byte slice 时直接 `m[string(b)]`**，编译器零拷贝。
9. **大字符串切片要警惕引用整个底层**：`s[:10]` 仍可能引用整个 1MB 字符串；用 `strings.Clone`（Go 1.18+）显式拷贝。
10. **类型转换 `T(x)` 与断言 `x.(T)` 区分清楚**，启用 `go vet`。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：跨平台 int 溢出
```go
var n int = 3 << 30   // 32 位机溢出
```
修正：明确写 `int64`。

### ❌ 陷阱 2：字符串按字节截断
```go
s := "你好"
fmt.Println(s[:2])  // 乱码
```
修正：转 `[]rune` 再切。

### ❌ 陷阱 3：浮点 ==
```go
if speed == 0.1 { ... }
```
修正：用 epsilon。

### ❌ 陷阱 4：用 string(int) 当 Itoa
```go
s := string(65)  // "A" 不是 "65"
```
修正：`strconv.Itoa(65)`。

### ❌ 陷阱 5：子字符串引用大字符串内存

```go
func first10(big string) string {
    return big[:10]  // 底层 Data 仍指向 big 整个数据
}
```
如果 `big` 是 1MB 字符串而你保留了 `first10` 的结果几小时，1MB 不会释放。修正：`strings.Clone(big[:10])`。

### ❌ 陷阱 6：unsafe.String 用错
```go
b := []byte("hello")
s := unsafe.String(&b[0], len(b))
b[0] = 'H'                // 修改 b 后 s 也被改了！
fmt.Println(s)            // "Hello"
```
`unsafe.String` 假设 b 不再被修改。一旦违反这条不变式，行为是未定义的。

### ❌ 陷阱 7：== 比较 NaN
```go
if math.NaN() == math.NaN() {  // 永远 false
}
```

---

## 第十章：练习题

**练习 1**：以下输出？
```go
s := "Go语言"
fmt.Println(len(s), utf8.RuneCountInString(s))
```

**练习 2**：以下代码在 32 位 Linux 与 64 位 Linux 上行为是否相同？
```go
var x int = 1 << 31
fmt.Println(x)
```

**练习 3**：哪段更快？为什么？
```go
// A
m := map[string]int{}
buf := []byte("key")
v := m[string(buf)]

// B
m := map[string]int{}
buf := []byte("key")
s := string(buf)
v := m[s]
```

**练习 4**：以下代码有何问题？
```go
func getSuffix(s string) string {
    return s[len(s)-10:]
}
```

**练习 5**：写一个函数 `truncate(s string, maxRunes int) string`，保证不在中间截断 UTF-8 字符。

---

## 参考答案

**练习 1**：`8 4`。"Go" 是 2 字节，"语" 3 字节，"言" 3 字节，共 8 字节，4 个 rune。

**练习 2**：32 位机上**编译错误**（常量溢出 int）。64 位机上正常输出 `2147483648`。要跨平台运行，写 `var x int64 = 1 << 31`。

**练习 3**：A 更快。编译器对 `m[string(byteSlice)]` 这种模式做了特殊优化，零分配；B 先把 string 赋给变量 `s`，编译器无法证明 `s` 之后不再使用，只能分配。

**练习 4**：当 `len(s) < 10` 时 panic（slice bounds out of range）。修正：先判长度。另外即使长度够，按字节切可能切到 UTF-8 字符中间。

**练习 5**：
```go
func truncate(s string, maxRunes int) string {
    if maxRunes <= 0 { return "" }
    cnt := 0
    for i := range s {
        if cnt == maxRunes {
            return s[:i]
        }
        cnt++
    }
    return s
}
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 整数 | `int` 平台相关；不同整数类型不隐式转换；溢出环绕不报错 |
| 浮点 | 永远用 epsilon 比较；NaN 用 IsNaN |
| rune / byte | 别名但语义不同；rune = Unicode code point，byte = uint8 |
| 字符串 | (Data, Len)，16 字节；不可变；UTF-8 编码 |
| 迭代 | `range` 按 rune；`s[i]` 按 byte |
| 转换 | string ↔ []byte 默认复制；特殊场景编译器零分配 |
| Go 1.20+ | `unsafe.String` / `unsafe.Slice` 提供零拷贝通道 |
| 拼接 | `strings.Builder` + `Grow` 是首选 |

下一篇 **G03 — 精通 Go 切片：底层结构与扩容机制** 会拆开 `SliceHeader` 三元组、讲清 `append` 何时分配新底层数组、为什么子切片会让一个 1MB 数组永远不被回收。

---

