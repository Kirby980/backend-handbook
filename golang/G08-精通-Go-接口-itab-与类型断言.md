# 精通 Go 接口：itab 与类型断言

> 课程编号：G08
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Interfaces 章节
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：接口到底是什么

```go
var x int = 42
var i any = x
fmt.Println(unsafe.Sizeof(i))   // 16

type Greeter interface{ Hello() }
type Foo struct{}
func (Foo) Hello() {}
var g Greeter = Foo{}
fmt.Println(unsafe.Sizeof(g))   // 16
```

无论是空接口还是非空接口，大小都是 16 字节（64 位机）。但内部结构不同。理解接口的物理结构、`itab` 缓存、装箱时机和成本，是写出零开销 / 低开销 Go 代码的关键。

---

## 第一章：接口的内部结构

### 1.1 两种接口类型

Go 内部对应两种结构：

```go
// 非空接口（含方法）
type iface struct {
    tab  *itab            // 类型 + 方法表
    data unsafe.Pointer   // 指向具体值
}

// 空接口 interface{}（即 any）
type eface struct {
    _type *_type          // 类型信息
    data  unsafe.Pointer  // 指向具体值
}
```

两者都是 16 字节（2 × 8）。

### 1.2 itab：接口与具体类型的"桥"

```go
type itab struct {
    inter *interfacetype  // 接口本身的类型信息
    _type *_type          // 具体类型信息
    hash  uint32          // _type.hash 的副本（用于 type switch 快速比较）
    _     [4]byte
    fun   [1]uintptr      // 方法函数指针表（实际长度 = 接口方法数）
}
```

`itab.fun` 是一个**方法函数指针数组**——按接口方法的声明顺序，存对应的具体类型方法地址。调用 `i.Method()` 时，runtime 通过 `itab.fun[methodIndex]` 找到具体函数。

### 1.3 itab 缓存

每次把 `T` 装入 `Interface I` 时，runtime 不会重新构建 itab——它**全局缓存**在 `itabTable`（key 是 (interface, type) 对）。第一次会做 method-lookup，之后 O(1) 命中。

```go
for i := 0; i < 1000000; i++ {
    var iface Stringer = MyType{}   // 只在第一次构建 itab
    _ = iface.String()
}
```

### 1.4 用 unsafe 窥探

```go
type Stringer interface{ String() string }
type T struct{ v int }
func (t T) String() string { return strconv.Itoa(t.v) }

var s Stringer = T{42}
header := (*[2]uintptr)(unsafe.Pointer(&s))
fmt.Printf("itab=%x data=%x\n", header[0], header[1])
```

`header[0]` 是 itab 指针，`header[1]` 是数据指针。

---

## 第二章：装箱与逃逸

### 2.1 装箱（boxing）

把具体类型放入接口槽位时，data 字段需要一个 8 字节的"通道"。如果具体值本身 ≤ 8 字节且不含指针，可能直接内联（Go 早期版本支持）；否则**值会被拷贝到堆**，data 指向它。

```go
func boxed(v interface{}) {}

func main() {
    x := 42
    boxed(x)   // x 装箱：分配 8 字节堆，写入 42
}
```

```bash
go build -gcflags="-m" main.go
# 输出：x escapes to heap
```

### 2.2 大值装箱代价

```go
type Big struct { data [128]byte }

func main() {
    b := Big{}
    var i interface{} = b   // 整个 128 字节复制到堆
}
```

每次装箱大值都是 alloc + memcpy。如果热点路径里出现 `interface{}` 参数和大 struct，性能可见可感。

### 2.3 指针类型不"再"装箱

```go
type T struct{ x int }
var p *T = &T{}
var i any = p   // p 本身 8 字节，data 直接存 p；不分配
```

把 `*T` 装入接口零分配（仅复制 16 字节 header）。这是为什么"返回 *T 给 interface{} 槽位"性能更好。

### 2.4 fmt.Println 的隐藏成本

```go
fmt.Println(1, 2, 3)   // 等价 fmt.Println([]any{1,2,3}...)
```

三个 int 都装箱到堆，slice 也分配。所以 `fmt.Println` 在热点路径上很贵。生产代码改用 `strconv` + `bytes.Buffer` 或 `strings.Builder`。

---

## 第三章：类型断言

### 3.1 两种形式

```go
var i interface{} = "hello"

s := i.(string)      // 失败 panic
s, ok := i.(string)  // 双返回，安全
```

### 3.2 实现机制

```go
v, ok := i.(T)
```

runtime 大致做：
1. 取 `i` 的 type（eface 是 `_type`，iface 是 `tab._type`）
2. 与目标 type 比较（指针相等就是同一类型）
3. 命中返回 data 解引用；不命中返回零值 + false（或 panic）

`(T)` 是具体类型时是 O(1)；`(I)` 是接口时要在 itabTable 查 (I, _type)，命中后用其 itab.fun。这一查找通常也很快。

### 3.3 多种断言对比

```go
var i any = 42

// ① 类型断言（具体类型）
n, ok := i.(int)

// ② 类型断言（接口类型）
s, ok := i.(fmt.Stringer)

// ③ 类型转换（编译期）
var x int = 5
y := int64(x)
```

类型转换 `T(v)` 是编译期，不涉及运行时；类型断言 `v.(T)` 是运行时。

### 3.4 type switch

```go
func describe(i any) {
    switch v := i.(type) {
    case int:
        fmt.Println("int", v)
    case string:
        fmt.Println("string", v)
    case []byte:
        fmt.Println("bytes len", len(v))
    case fmt.Stringer:
        fmt.Println("stringer", v.String())
    case nil:
        fmt.Println("nil")
    default:
        fmt.Println("unknown")
    }
}
```

实现细节：
- runtime 对每个 case 做类型比较，按声明顺序短路
- 编译器对常见类型（int、string）生成快速比较
- 接口类型 case 走 itab 查找

### 3.5 性能数量级

| 操作 | 大致开销 |
|---|---|
| 直接调用 `T.M()` | ~1ns（甚至内联到 0） |
| 接口调用 `I.M()`（已缓存 itab） | ~2-3ns |
| 类型断言 `v.(T)` 命中 | ~1ns |
| type switch 多个 case | ~每 case 1ns |
| 装箱 int 到 interface{} | ~5-10ns + 1 alloc |

不要为了"通用"无脑用 `any`——热点路径生效。

---

## 第四章：interface 与 nil

### 4.1 nil interface

```go
var i interface{}
fmt.Println(i == nil)   // true
```

`i` 的 _type 和 data 都是 nil。

### 4.2 装了 nil 的接口

```go
var p *int = nil
var i interface{} = p
fmt.Println(i == nil)   // false ！
```

`i._type = *int`, `i.data = nil`。type 非 nil，整个接口就非 nil。

### 4.3 真实场景

```go
type CustomErr struct{}
func (*CustomErr) Error() string { return "bad" }

func badFunc() error {
    var e *CustomErr
    if condition { e = &CustomErr{} }
    return e   // 错误！condition 为 false 时仍是非 nil error
}

if err := badFunc(); err != nil {
    log.Fatal(err)   // 永远进入这里
}
```

修正：
```go
func goodFunc() error {
    if !condition { return nil }
    return &CustomErr{}
}
```

或者用 named return + 显式判断。

### 4.4 用 reflect 区分

```go
func IsNil(i interface{}) bool {
    if i == nil { return true }
    v := reflect.ValueOf(i)
    switch v.Kind() {
    case reflect.Ptr, reflect.Map, reflect.Slice, reflect.Chan, reflect.Func:
        return v.IsNil()
    }
    return false
}
```

热点路径慎用——reflect 慢。

---

## 第五章：接口设计原则

### 5.1 接受接口，返回结构体

```go
// ❌ 反模式：返回接口
func NewLogger() Logger { return &fileLogger{...} }

// ✅ 返回具体类型
func NewLogger() *FileLogger { return &FileLogger{...} }
```

返回具体类型让调用方有更丰富的 API；接口由调用方在需要抽象处定义。

### 5.2 接口越小越好

```go
// ✅ Go 标准库风格
type Reader interface { Read([]byte) (int, error) }
type Writer interface { Write([]byte) (int, error) }
type Closer interface { Close() error }

// 组合
type ReadWriteCloser interface {
    Reader; Writer; Closer
}
```

`io.Reader` 一个方法，被无数类型实现。**接口的方法数与可重用性成反比**。

### 5.3 隐式实现

Go 没有 `implements` 关键字。类型实现接口完全靠"方法集匹配"。这种"鸭子类型"+"编译期检查"的组合非常灵活：

- 第三方库类型可以实现你的接口（无需修改库）
- 测试时用 mock 类型即可，无需依赖注入框架

### 5.4 编译期断言

```go
var _ io.Reader = (*MyReader)(nil)
```

放源文件顶部，确保 `*MyReader` 永远满足 `io.Reader`。重构时立即暴露问题。

### 5.5 不要用接口做 enum

```go
// ❌ 反模式
type Color interface { color() }
type Red struct{}
func (Red) color() {}
type Blue struct{}
func (Blue) color() {}

// ✅ 用 iota 常量
type Color int
const (
    Red Color = iota
    Blue
)
```

接口适合"行为"抽象，不适合"枚举值"。

---

## 第六章：接口的高级用法

### 6.1 sealed interface 模式

想限制实现者必须在自己的包内：

```go
type Result interface {
    isResult()
}

type Ok struct{ Value int }
func (Ok) isResult() {}

type Err struct{ Err error }
func (Err) isResult() {}
```

`isResult()` 是导出但**不可调用**（小写）。外部包无法实现这个方法，所以 Result 是 sealed。可以做"标记接口"（marker interface）。

### 6.2 类型 set 与泛型约束

Go 1.18+ 泛型把接口扩展为"类型集"：

```go
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](xs []T) T {
    var s T
    for _, x := range xs { s += x }
    return s
}
```

约束接口在泛型上下文有效；普通接口仍是运行时多态。

### 6.3 通过接口暴露 reflect 信息

```go
type Marshaler interface { MarshalJSON() ([]byte, error) }
```

`encoding/json` 用类型断言检查传入对象是否实现 `Marshaler`，是的话走自定义路径，否则反射默认编码。这是标准库扩展点的常用模式。

### 6.4 接口的多重实现

```go
type Stringer interface { String() string }
type Errorer interface { Error() string }

type Foo struct{}
func (Foo) String() string { return "foo" }
func (Foo) Error() string  { return "foo error" }

// Foo 同时是 Stringer 和 Errorer
```

一个类型可以同时满足多个接口。

---

## 第七章：生产级最佳实践

1. **接受接口，返回结构体**：内部实现保留扩展空间。
2. **小接口、组合优先**：单方法接口最容易复用。
3. **类型断言永远用双返回值**：避免 panic。
4. **不在热点路径用 `interface{}`**：装箱开销显著。
5. **不要 return 具体类型 nil 给 error**：用 `return nil`。
6. **编译期断言每个公开接口实现**：`var _ I = (*T)(nil)`。
7. **不要为了 mock 而过度抽象**：Go 的 mock 多用 wrapper 类型。
8. **大 struct 通过 interface 传时考虑 \*T**：避免装箱拷贝。
9. **type switch 注意短路顺序**：常见 case 放前面。
10. **`any` 是 Go 1.18 的 `interface{}` 别名**：新代码用 `any`。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：interface nil 不等
```go
var p *T = nil
var i error = p
// i != nil
```

### ❌ 陷阱 2：值接收者实现接口，但只有 *T 进入接口
```go
type Foo struct{}
func (f *Foo) Hello() {}
var g Greeter = Foo{}   // ❌ Foo 没有 Hello
```

### ❌ 陷阱 3：interface 装箱使 int 逃逸
```go
func print(v any) {}
print(42)   // 42 逃逸
```

### ❌ 陷阱 4：在循环里类型断言
```go
for _, v := range items {
    if s, ok := v.(string); ok { ... }   // 每次都查 _type
}
```
能预判类型就在循环外断言一次。

### ❌ 陷阱 5：sealed 接口被混淆
```go
type Sealed interface { isSealed() }
```
"sealed" 在 Go 中仅靠 unexported 方法实现，不是语言特性。

### ❌ 陷阱 6：用 interface{} 做 map value 损失性能
```go
m := map[string]interface{}{}
```
所有 value 装箱，频繁 GC。能用具体类型就用具体类型。

### ❌ 陷阱 7：fmt 格式参数装箱
```go
fmt.Sprintf("%d", i)   // i 装箱
```
热点用 `strconv.Itoa(i)`。

---

## 第九章：练习题

**练习 1**：以下输出？
```go
type MyErr struct{}
func (*MyErr) Error() string { return "x" }
func f() error {
    var e *MyErr
    return e
}
fmt.Println(f() == nil)
```

**练习 2**：为何把 `int` 放入 `interface{}` 会导致逃逸？

**练习 3**：写一个 `Cast[T any](v any) (T, bool)` 泛型函数（不能用反射）。

**练习 4**：以下两段哪个性能更好？为什么？
```go
// A
func sum(xs []any) any {
    var s int
    for _, x := range xs { s += x.(int) }
    return s
}

// B
func sum(xs []int) int {
    var s int
    for _, x := range xs { s += x }
    return s
}
```

**练习 5**：解释 `var x = (*Foo)(nil)` 与 `var x Foo = Foo{}` 在赋值给接口时的差异。

---

## 参考答案

**练习 1**：`false`。`return e` 把具体类型 nil 装入 error 接口，type 非 nil。

**练习 2**：interface{} 的 data 字段是 unsafe.Pointer（8 字节，存指针）。int 本身 8 字节但被当作"任意类型"管理时，编译器要确保 data 可指向它——必须放到堆上。Go 团队曾考虑"小值内联到 data 槽"，但当前实现仍 escape。

**练习 3**：
```go
func Cast[T any](v any) (T, bool) {
    t, ok := v.(T)
    return t, ok
}
```

**练习 4**：B 快得多。A 每次循环类型断言（~1ns）+ 取出 int；如果 xs 是 `[]any`，每个元素是 16 字节 interface header；遍历开销大。B 走 SIMD 友好的密集 int 数组。

**练习 5**：
- `(*Foo)(nil)` 装入接口：type=*Foo, value=nil → 非 nil 接口
- `Foo{}` 装入接口：type=Foo, value=Foo{}副本 → 非 nil 接口，且 value 占 stash
两者都非 nil，但 type 不同——`i.(*Foo)` 对第一个命中，对第二个失败。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| iface / eface | 16 字节；(tab, data) 或 (_type, data) |
| itab | 含方法函数指针表；全局缓存 itabTable |
| 装箱 | 大多触发堆分配；指针装入零分配 |
| 类型断言 | `v.(T)` 运行时；双返回值安全 |
| type switch | 实现 ~ 连续断言；常见类型有快路径 |
| nil interface | type 也必须 nil；最常见的 error 陷阱 |
| 设计原则 | 接收接口、返回 struct；小接口；编译期断言 |
| 性能 | 接口调用 ~2-3ns；装箱 alloc 是隐藏成本 |

下一篇 **G09 — 精通 Go 泛型：类型参数、约束与性能** 会拆开 1.18 引入的泛型，讲清类型集语法、GC shape stenciling 实现策略、泛型 vs 接口的性能权衡。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G08-精通-Go-接口-itab-与类型断言.md`
