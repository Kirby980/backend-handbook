# 精通 Go 的变量、常量与 iota 习语

> 课程编号：G01
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Variables & Constants 章节
> 难度：⭐⭐⭐（看似入门，但有大量编译器层面的细节常被忽略）
> 预计阅读时间：45 分钟

---

## 引言：为什么"变量与常量"值得花一整章

很多开发者把 Go 的变量声明和常量当成"扫一眼语法表就够了"的入门内容。这是一个昂贵的误解。

下面这段代码，能立刻说出输出吗？

```go
package main

import "fmt"

func main() {
    const x = 1 << 62
    var y float64 = x / 1e10
    fmt.Println(y)
}
```

（顺带一个相关陷阱：把上面 `float64` 改成 `int32`，会**编译失败**——`x / 1e10` 是无类型浮点常量 `461168601.8427387904`，赋值给整型时必须"可表示为整数"，小数部分非零就报 `truncated to integer`。这条规则也常被用来面试考察对"无类型常量晋升"的理解。）

或者这段——它会触发什么编译器优化？运行时会分配内存吗？

```go
package main

const Pi = 3.14159265358979323846264338327950288419716939937510582097494459

func main() {
    println(Pi * 2)
}
```

如果你对这两个问题中的任何一个犹豫了，那么这一章就值得你认真读完。Go 的变量与常量背后藏着**无类型常量（untyped constants）**、**任意精度算术（arbitrary precision arithmetic）**、**零值哲学**、**变量遮蔽**、**作用域规则**、**编译期常量折叠**等一整套机制。理解它们，是写出零分配热点路径代码、写出可维护枚举、避免一类难调的并发 bug 的前提。

本文按"语法 → 语义 → 编译器/运行时行为 → 工程实践 → 陷阱"的脉络展开。

---

## 第一章：变量声明的五种姿势

Go 提供了 5 种声明变量的方式，每种背后的语义都不同。

### 1.1 五种写法对照

```go
// ① 完整 var 声明，带类型和初始值
var a int = 10

// ② var 推导类型（省略类型）
var b = 10        // b 是 int

// ③ var 只声明（用零值初始化）
var c int         // c == 0

// ④ 短变量声明（短语句，:= 仅函数内）
d := 10

// ⑤ 多变量同时声明
var (
    e int     = 1
    f string  = "hello"
    g, h      = 2, 3.14   // g 是 int, h 是 float64
)
```

### 1.2 `var` 与 `:=` 的关键差异

下面这张表把开发者经常混淆的点拉清楚：

| 维度 | `var x = v` | `x := v` |
|---|---|---|
| 可用位置 | 包级 + 函数内 | **仅函数内（含方法体、闭包体）** |
| 是否允许重复声明 | 否 | 至少要有 1 个新变量时允许 |
| 与零值的关系 | 可只声明不赋值 | 必须有右值表达式 |
| 与类型注解的关系 | 可显式标注 `var x T = v` | 必须由右值推导 |

最隐蔽的一条是"重复声明":

```go
func parse() error {
    f, err := os.Open("a.txt")      // 声明 f 和 err
    // ...
    g, err := os.Open("b.txt")      // ✅ 合法: g 是新的, err 被重新赋值
    _, _ = f, g
    return err
}
```

这条规则非常实用——它让 `if err := ...; err != nil` 链式写法能不断复用 `err`。但它也是**变量遮蔽（shadowing）**的源头，后面单独一节会展开。

### 1.3 包级与函数级的初始化顺序

```go
var x = computeX()
var y = x + 1

func computeX() int { return 42 }
```

包级 `var` 的初始化顺序由**依赖图的拓扑序**决定，而不是源码顺序。`y` 依赖 `x`，所以 `x` 先求值。这条规则在多文件、多变量交叉依赖时会变得复杂。当编译器检测到循环依赖，会直接报错：`initialization cycle`。

包级初始化的完整顺序是：

1. 所有导入包的初始化（递归）
2. 包内所有 `var` 按依赖拓扑序求值
3. 包内所有 `init()` 函数按源文件字典序、再按文件内出现顺序依次执行

记住这条顺序，能解释很多"为什么我把变量放在文件 A 而 init 在文件 B 行为不同"的怪事。

---

## 第二章：零值哲学

### 2.1 Go 没有"未初始化的变量"

C/C++ 允许 `int x;` 后 `x` 是栈上的随机内存（未定义行为）。Go 不允许——**任何变量声明出来就是它类型对应的零值**。这是 Go 安全性的一个支柱。

完整零值表：

| 类型类别 | 零值 |
|---|---|
| 数值（int / uint / float / complex） | `0` / `0.0` / `(0+0i)` |
| 布尔 | `false` |
| 字符串 | `""`（长度 0，**底层指针为 nil**） |
| 指针、函数、接口、map、slice、channel | `nil` |
| 数组 | 每个元素的零值组成 |
| 结构体 | 每个字段的零值组成 |

### 2.2 零值并非"无用值"

Go 标准库刻意设计了大量类型，使其**零值即可用**（zero value is useful）。典型例子：

```go
var mu sync.Mutex   // 零值就是一个未锁的、可用的互斥锁
mu.Lock()

var b bytes.Buffer  // 零值就是一个空缓冲区
b.WriteString("hi")

var wg sync.WaitGroup  // 零值就是计数器为 0 的 WaitGroup
```

这套设计的好处：**你不需要写构造函数，不需要 `New()`，不需要担心忘记初始化**。如果你设计自己的类型，应当尽可能让零值有意义。

### 2.3 `nil` 不只是一个值

`nil` 在 Go 中**不是单一类型的值**，它是 6 种类型的零值表示：指针、interface、map、slice、channel、function。每种 nil 的行为不一样：

```go
var s []int
s = append(s, 1)   // ✅ append 到 nil slice 是合法的

var m map[string]int
m["a"] = 1         // ❌ panic: assignment to entry in nil map（必须 make）

var ch chan int
ch <- 1            // ❌ 阻塞永远（向 nil channel 发送）
<-ch               // ❌ 阻塞永远（从 nil channel 接收）

var f func()
f()                // ❌ panic: invalid memory address
```

特别地，**`nil` interface 不等于"包了 nil 的 interface"**：

```go
var p *int = nil
var i interface{} = p
fmt.Println(i == nil)   // false ！
```

理由：interface 在运行时是一对 `(type, value)`，只有当 type 也是 nil 时整个 interface 才等于 nil。这是 Go 最经典的面试题之一，也是真实生产代码里**最常见的隐式 bug**——返回 `error` 类型时把已经赋了 nil 的具体类型变量返回，调用方 `if err != nil` 永远为真。

---

## 第三章：作用域与变量遮蔽

### 3.1 词法作用域

Go 是**静态词法作用域**，作用域以 `{ }` 为边界。但有几个常被忽略的细节：

```go
if x := compute(); x > 0 {
    // x 在 if 体和 else 体内都可见
} else {
    fmt.Println(x)   // ✅ 可见
}
// x 在这里不可见
```

`for`、`switch`、`if` 的"短语句"在**整个语句结构内可见**，不只是第一个块。

### 3.2 遮蔽（shadowing）的真实陷阱

```go
func update() (err error) {
    defer func() {
        if err != nil {
            log.Println(err)
        }
    }()

    if cond {
        err := doSomething()      // ❌ 新声明了一个局部 err
        if err != nil {
            return err            // 返回值赋给了外层 err，但内层 err 已经被丢
        }
    }
    return nil
}
```

这段代码看起来"工作正常"，但实际上**外层命名返回值 `err` 从未被赋值**——内层 `err :=` 创建了一个新变量。如果有人后续重构 `if` 内的 `return err` 改成 `return`，bug 立刻出现：defer 里 `err` 永远是 nil。

**防御工具**：

1. 启用 `go vet -shadow`（旧版）或 `golangci-lint` 的 `govet` 检查器配合 `shadow` 启用。
2. 个人原则：**命名返回值 + 短变量声明同名变量** 是高危组合，要么换名，要么用 `=`。

### 3.3 循环变量的"经典陷阱"——历史与现状（Go 1.22 分水岭）

```go
funcs := []func(){}
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs {
    f()
}
```

- **Go 1.21 及之前**：输出 `3 3 3`。原因：所有闭包捕获的是**同一个** `i`，循环结束后 `i == 3`。
- **Go 1.22 起**（已是当前默认；最新稳定版 Go 1.26）：输出 `0 1 2`。语言规范修改为：`for` 的循环变量在**每次迭代都是一个新变量**。生效条件是 `go.mod` 声明 `go 1.22` 或更高——现在维护中的项目几乎都已升级。

这是 Go 历史上罕见的、破坏向后兼容的语义修改（受 `go.mod` 中 `go 1.22` 指令控制）。在写跨版本兼容代码、或维护老项目时，务必明确意识到这条规则取决于模块声明的 Go 版本。

---

## 第四章：常量的本质

常量是 Go 中最被低估、也最有特色的部分。

### 4.1 编译期 vs 运行时

```go
const Pi = 3.14159
var Pi2 = 3.14159
```

`const Pi` **没有运行时存在**——它不占用任何内存，编译器在编译期把所有引用替换为字面值（或经过常量折叠后的字面值）。`var Pi2` 是一个真实的全局变量，编译后在二进制的 `.data` 或 `.rodata` 段中占 8 字节（一个 float64）。

```bash
# 编译后用 nm 查看
$ go build -o app main.go
$ go tool nm app | grep Pi
# 你会看到 Pi2 但看不到 Pi
```

### 4.2 无类型常量：Go 的精度魔法

```go
const Big = 1 << 100
```

`int` 在 64 位机上最大是 `1 << 63 - 1`，远小于 `1 << 100`。但这段代码**完全合法**，因为 `Big` 是一个**无类型常量（untyped constant）**。

无类型常量的关键性质：

1. **任意精度**：编译器内部用 `math/big.Int` / `big.Float` 表示，精度至少 256 位（数值类常量）或 272 位（浮点）。
2. **没有具体类型，直到被使用**：一旦在表达式中需要类型（赋值给变量、传给函数），编译器才会做隐式类型转换。
3. **若上下文也没指定类型，则使用其"默认类型"**：整数 → `int`，浮点 → `float64`，字符 → `rune`(`int32`)，字符串 → `string`，复数 → `complex128`。

来看个会让人困惑的例子：

```go
const x = 1 << 62          // 合法：无类型整数常量 = 4611686018427387904
var y int = x              // ✅ 64 位平台：int=int64 (max ~9.2e18)，仍在范围内
                           // ❌ 32 位平台：int=int32 (max ~2.1e9)，编译错误溢出
var z int = x * 2          // ❌ 编译错误：x*2 = 2^63 = 9223372036854775808，刚好溢出 int64 max (2^63-1)

const a = 1 << 62
const b = a * 2            // ✅ 合法：常量算术仍在无类型范畴中，可保 2^63
fmt.Println(b)             // ❌ 编译错误：b 在传给 Println 时需具型化为默认类型 int → 溢出
```

记住这条规则：**常量算术不溢出，只有"赋给具体类型变量"时才会做范围检查**。

### 4.3 类型化常量

```go
const Pi float32 = 3.14159
```

一旦标注类型，常量就**失去任意精度特性**。在表达式中混用类型化常量与无类型常量时，无类型常量会被隐式转为对方类型。两个类型化常量必须类型相同才能直接运算。

```go
const a float32 = 1.0
const b float64 = 2.0
var c = a + b              // ❌ 编译错误：mismatched types
var d = a + 2.0            // ✅ 合法：2.0 是无类型，被转为 float32
```

### 4.4 常量表达式的能力边界

常量表达式中**不能**调用：

- 用户定义的函数（除了少数内置函数）
- 接口方法

**能**调用的内置函数（结果仍是常量）：`len`、`cap`（只对常量字符串和数组）、`unsafe.Sizeof`、`unsafe.Alignof`、`unsafe.Offsetof`、`real`、`imag`、`complex`、`min`、`max`（Go 1.21+）。

```go
const s = "hello"
const n = len(s)           // ✅ 5
const arr = [3]int{1,2,3}
const m = len(arr)         // ❌ 数组本身不是常量，无法这样写

var arr2 = [3]int{1,2,3}
const m2 = len(arr2)       // ❌ 同样不行
// 但若是数组字面量：
const m3 = len([3]int{1,2,3})  // ❌ 仍然不行 —— 数组字面量不是常量
```

---

## 第五章：iota 完全指南

`iota` 是 Go 的**编译期自增计数器**，是常量章节最值得花时间的部分。

### 5.1 iota 的精确语义

`iota` 在 `const` 块中：
- 每个 `const` 块开始时，`iota = 0`
- **每出现一行新的 ConstSpec**（不管是否提到 iota），`iota` 自增 1
- 同一行内 `iota` 值相同

```go
const (
    A = iota    // iota = 0, A = 0
    B           // iota = 1, B = 1（隐式继承上行表达式）
    C           // iota = 2, C = 2
)

const (
    X = iota    // X = 0
    _           // 跳过 1
    Y           // Y = 2
    Z           // Z = 3
)

const (
    a, b = iota, iota * 2   // a=0, b=0
    c, d                    // c=1, d=2（继承表达式: iota, iota*2）
    e, f                    // e=2, f=4
)
```

### 5.2 经典模式 ①：枚举

```go
type Weekday int

const (
    Sunday Weekday = iota
    Monday
    Tuesday
    Wednesday
    Thursday
    Friday
    Saturday
)
```

注意第一行 `Sunday Weekday = iota` —— 它做了两件事：① 给 `Sunday` 一个具体类型 `Weekday`，② 后续 `Monday...Saturday` 隐式继承"类型 + 表达式"，全都是 `Weekday`。

配合 `go generate` + `stringer` 工具可以自动生成 `String()` 方法：

```go
//go:generate stringer -type=Weekday
```

### 5.3 经典模式 ②：位掩码（bit flags）

```go
type Permission uint8

const (
    Read    Permission = 1 << iota    // 1 << 0 = 1
    Write                              // 1 << 1 = 2
    Execute                            // 1 << 2 = 4
)

p := Read | Write
fmt.Println(p & Read != 0)   // true，是否有读权限
```

`<<` 表达式被继承，每行 `iota` 不同，结果就是 1、2、4、8……的位掩码。这是 Unix 文件权限、Linux open flag 的经典风格。

### 5.4 经典模式 ③：跳过零值

零值常被用作"未指定"占位，所以枚举时常希望真正的取值从 1 开始：

```go
type Status int

const (
    _ Status = iota   // 0 留作 unknown / 未指定
    Active            // 1
    Inactive          // 2
    Pending           // 3
)
```

或者用一个显式的 `Unknown`：

```go
const (
    StatusUnknown Status = iota
    StatusActive
    StatusInactive
)
```

第二种写法更推荐——`StatusUnknown` 显式参与了 String()、JSON 序列化等。

### 5.5 经典模式 ④：单位与量纲

`time` 包是教科书级范例：

```go
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```

注意这里**没有用 iota**——因为单位不是等差，而是阶梯倍数。这告诉我们：**iota 不是"枚举的银弹"**，它只在等差/可由表达式推出的场景下闪光。

类似地可以定义存储单位：

```go
const (
    KB = 1 << (10 * (iota + 1))   // 1 << 10
    MB                             // 1 << 20
    GB                             // 1 << 30
    TB                             // 1 << 40
    PB                             // 1 << 50
)
```

### 5.6 iota 的坑：跨 const 块不连续

```go
const A = iota   // A = 0
const B = iota   // B = 0（新的 const 块，iota 重置）
```

`iota` 在**每个 const 块**重置，不是全局自增。新手常把 5 个常量分散在 5 个独立 `const` 写法里，结果它们全部等于 0。

---

## 第六章：性能与编译器视角

### 6.1 常量折叠（Constant Folding）

编译器会在编译期对常量表达式求值，结果直接嵌入指令。

```go
const Width = 1920
const Height = 1080

func area() int {
    return Width * Height
}
```

`go tool compile -S main.go` 反汇编你会看到 `area` 直接返回立即数 `2073600`，根本没有乘法指令。

**实战意义**：把循环边界、数组容量、缓冲区大小等用 `const` 表达，编译器能做更激进的优化（循环展开、向量化）。用 `var` 时编译器不敢假设值不变。

### 6.2 变量在栈还是堆？逃逸分析

`var x int = 5` 在函数内部并不一定意味着"栈分配"。Go 的逃逸分析决定变量去哪儿：

```go
func foo() *int {
    x := 5
    return &x   // x 逃逸到堆
}

func bar() int {
    x := 5
    return x    // x 留在栈
}
```

用 `go build -gcflags="-m" main.go` 可以看到编译器的逃逸决策。这是 G21（逃逸分析专题）的核心内容，这里只提示：**变量声明语法不决定内存位置，使用方式才决定**。

### 6.3 包级 var 一定在堆吗？

不。包级 `var` 存放在**二进制的数据段（`.data` 或 `.bss`）**，不在堆也不在栈。它们的生命周期等于整个进程，地址固定，GC 不扫描它们本身（但会扫描它们指向的堆对象）。

```go
var globalBuf [1 << 20]byte   // 1MB，在 .bss 段
```

这意味着用一个大数组做全局缓冲池是**零分配**的——但要小心初始化静态大小会让二进制变胖。

---

## 第七章：生产级最佳实践

1. **包级常量优先**：所有"会被多处引用的字面量"（超时、buffer 大小、协议字段名、错误码……）都应当是 `const`。
2. **导出常量带类型注解**：`const MaxRetries int = 3` 比 `const MaxRetries = 3` 更稳。后者在与外部 API 比较时可能引发意外的隐式转换。
3. **零值即可用**：自己设计类型时，让 `var x MyType` 直接可用，避免强制要求 `NewXxx()`。
4. **避免 `var x = "..."` 这种废话**：能用 `const` 就别用 `var`，能省类型就省类型。
5. **iota 配合 stringer**：所有枚举搭配 `//go:generate stringer -type=...`，避免手写 `String()`。
6. **位标志用 `1 << iota`**，不要硬编码 `1, 2, 4, 8, 16`。
7. **不要把魔法数硬编码在循环里**：`for i := 0; i < 86400; i++` 远不如 `for i := 0; i < SecondsPerDay; i++` 易读。
8. **包级 `var` 慎用大对象**：超过几 MB 的全局缓冲会让 binary 显著膨胀；考虑 lazy `sync.Once` 初始化。
9. **接口 nil 陷阱**：返回 error 时，如果用 `var err *MyError` 然后 `return err`（即使 nil），调用方看到的就不是 nil interface。要么直接 `return nil`，要么 `if err == nil { return nil }`。
10. **命名返回值 + shadowing**：开启 `golangci-lint` 的 `shadow` 检查，或在团队规范里禁止 `if err := ...; err != nil { return err }` 与命名返回值 `err` 同时存在。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：在循环里把"零值即可用"当成"每次重新清空"

```go
var m map[string]int
for _, k := range keys {
    m[k] = 1   // panic: nil map
}
```

修正：`m := make(map[string]int)`。

### ❌ 陷阱 2：以为 `:=` 永远是声明新变量

```go
func f() {
    x := 1
    x := 2   // ❌ 编译错误：no new variables on left side of :=
}
```

`:=` 必须**至少有一个左值是新变量**。同一个函数体里第二次给同名变量赋值要用 `=`。

### ❌ 陷阱 3：常量浮点等于性

```go
const a = 0.1 + 0.2
const b = 0.3
fmt.Println(a == b)   // true（常量是任意精度，0.1+0.2 精确等于 0.3）

var c = 0.1 + 0.2
var d = 0.3
fmt.Println(c == d)   // false（运行时 float64 误差）
```

常量算术没有 IEEE 754 误差——这是隐式陷阱也是隐式福利。

### ❌ 陷阱 4：iota 的多变量行内继承

```go
const (
    a, b = iota + 1, iota + 2   // a=1, b=2
    c, d                         // c=2, d=3
    e, f                         // e=3, f=4
)
```

不少人以为 `c, d` 会重新执行 `iota + 1, iota + 2`，确实——但 iota 已经是 1，结果是 2 和 3。新手常算成 (1,3) 或 (3,4)。

### ❌ 陷阱 5：用 var 当 const

```go
var MaxConnections = 100   // ❌ 全大写但用了 var
```

任何"启动后不应变"的值都该是 `const`。用 `var` 给了别人改它的权利，破坏假设。

### ❌ 陷阱 6：包级 var 求值顺序依赖

```go
// file_a.go
var X = Y + 1
// file_b.go
var Y = 10
```

虽然能通过编译，但**跨文件依赖让代码可读性骤降**。同一个变量族尽量放同一文件，必要时用 `init()` 显式表达顺序。

### ❌ 陷阱 7：返回带具体类型的 nil

```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func find() error {
    var e *MyError = nil
    return e   // ❌ 返回的不是 nil interface
}

if find() == nil {
    // 永远不会进来
}
```

修正：直接 `return nil`。

---

## 第九章：练习题

> 试着不运行代码，先在脑子里算出答案，再用 `go run` 验证。

**练习 1**：以下输出？

```go
const (
    A = iota * 2     // ?
    B                // ?
    _                // skip
    C                // ?
)
```

**练习 2**：以下代码能否编译？为什么？

```go
const N = 1 << 100
var x = N / (1 << 99)
var y int = N / (1 << 99)
```

**练习 3**：找出 bug：

```go
func process() (result int, err error) {
    defer func() {
        if err != nil {
            log.Println("failed:", err)
        }
    }()
    if v, err := compute(); err == nil {
        result = v
    }
    return
}
```

**练习 4**：以下两段哪个性能更好？为什么？

```go
// A
const BufSize = 4096
func read() {
    var buf [BufSize]byte
    // ...
}

// B
func read() {
    buf := make([]byte, 4096)
    // ...
}
```

**练习 5**：写出一个用 iota 表达"HTTP 状态码大类（1xx/2xx/3xx/4xx/5xx）"的简洁定义。

---

## 参考答案与解析

**练习 1**：`A=0, B=2, C=6`。`A`：iota=0, 0*2=0；`B`：iota=1, 1*2=2；`_`：iota=2 但被丢弃；`C`：iota=3, 3*2=6。

**练习 2**：`var x = N / (1<<99)` 合法（常量算术：2，默认 int），`var y int = N / (1<<99)` 也合法（结果 2 在 int 范围内）。陷阱在于：常量算术中 `1<<100` 不溢出，但若直接 `var y int = N` 会编译错误。

**练习 3**：典型 shadowing。`if v, err := compute(); ...` 中的 `err` 是新局部变量，defer 看到的命名返回值 `err` 永远是 nil。修正：拆开成 `v, err2 := compute(); if err2 == nil { ... }`，或外层先声明再用 `=`。

**练习 4**：A 更好。A 是栈上固定数组（前提是 buf 不逃逸），零分配；B 是堆分配 + GC 压力。但若 `buf` 被返回或传给闭包，A 也会逃逸到堆，结果近似。用 `-gcflags="-m"` 验证。

**练习 5**：
```go
const (
    HTTPInformational = (iota + 1) * 100   // 100
    HTTPSuccess                             // 200
    HTTPRedirection                         // 300
    HTTPClientError                         // 400
    HTTPServerError                         // 500
)
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 变量声明 | 5 种姿势；`:=` 仅函数内；至少 1 个新变量 |
| 零值 | 任何变量都有零值；nil 有 6 种含义；零值应可用 |
| 作用域 | 词法 + 块；`if`/`for`/`switch` 短语句作用域跨整个结构 |
| 遮蔽 | 命名返回值 + `:=` 是高危；用 lint 工具防御 |
| 常量本质 | 编译期、无运行时存在、可任意精度 |
| 无类型常量 | 直到被使用前没有类型；默认类型在上下文缺失时启用 |
| iota | 每个 const 块重置；按行自增；表达式可继承 |
| 编译器优化 | 常量折叠是 `const` 的核心红利；`var` 拿不到 |

下一篇 **G02 — 精通 Go 数据类型：数值、Rune 与字符串内幕** 会继续从字节层面挖：`int` 在 32 位与 64 位平台为什么不同？为什么 `len("中")` 等于 3？UTF-8 与 rune 如何共生？字符串底层结构和切片有何关系？

---

> 🔁 反馈：写完一节就动手敲一遍，不要只看。Go 的教程价值 = 阅读 × 编译运行。
