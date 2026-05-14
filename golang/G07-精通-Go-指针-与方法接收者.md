# 精通 Go 指针与方法接收者

> 课程编号：G07
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Pointers & Methods 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：方法集的两道刁题

```go
type Counter struct{ n int }
func (c Counter) Get() int { return c.n }
func (c *Counter) Inc()    { c.n++ }

type Adder interface { Inc() }

func main() {
    var a Adder = Counter{}    // ① 编译错误，为什么？
    c := Counter{}
    c.Inc()                    // ② 这行能编译，为什么？
    fmt.Println(c.n)           // ③ 输出什么？

    m := map[string]Counter{"x": {}}
    m["x"].Inc()               // ④ 编译错误，为什么？
}
```

四个问题，每一个背后都是 Go 的"方法集 + 可寻址性"规则。这一章把指针、可寻址性、接收者选择讲透。

---

## 第一章：指针基础

### 1.1 取地址与解引用

```go
var x int = 10
p := &x         // *int 类型
fmt.Println(*p) // 10
*p = 20
fmt.Println(x)  // 20
```

`&` 取地址，`*` 解引用。Go 指针**不能做算术**（`p++` 编译错误），这是 Go 安全性的重要组成。如果非要算术，得借助 `unsafe`（见 G25）。

### 1.2 new(T) vs &T{}

```go
p1 := new(int)        // *int, *p1 = 0
p2 := &int{}          // ❌ 编译错误，int 不能用字面量

type Point struct{ X, Y int }
p3 := new(Point)      // *Point, *p3 = Point{0, 0}
p4 := &Point{}        // 等价于 new(Point)
p5 := &Point{1, 2}    // 还可以初始化字段

s := new([]int)       // *[]int, *s == nil（不是 [0]int！）
m := new(map[K]V)     // *map, *m == nil
```

**经验**：99% 场景用 `&T{}` 或 `&T{field: ...}`。`new(T)` 主要用于 `new(int)` 这种基础类型无字面量的情形。

### 1.3 nil 指针

```go
var p *int = nil
fmt.Println(*p)   // panic: runtime error: invalid memory address
```

但**有些方法**可以在 nil 接收者上调用（见 4.5）。

### 1.4 不可寻址（unaddressable）

不是所有"值"都能取地址：

```go
// ✅ 可取地址
var x int
&x

// ✅ slice 元素可寻址
s := []int{1, 2, 3}
&s[0]

// ✅ 结构体字段（如果结构体本身可寻址）
var p Point
&p.X

// ❌ map 元素不可寻址（map 扩容会移动 value）
m := map[string]int{"a": 1}
// &m["a"]   编译错误

// ❌ 字面量不可寻址
// &42       编译错误
// &Point{}  实际上：是可以的，会自动分配（&CompositeLit 是合法语法）

// ❌ 函数返回值不可寻址
// &foo()    编译错误

// ❌ 接口动态值不可寻址
var i interface{} = 5
// v := i.(int); &v 可以，但 &i.(int) 不行
```

可寻址性还决定**方法集**和**赋值合法性**：

```go
// map[K]V 的 V 不可寻址，所以
m["a"]++          // ❌ 编译错误：cannot assign to m["a"]
m["a"] = m["a"]+1 // ✅ OK，读出来加再写回
```

---

## 第二章：方法接收者——值 vs 指针

### 2.1 两种声明

```go
type Counter struct{ n int }

func (c Counter) Get() int { return c.n }    // 值接收者
func (c *Counter) Inc()    { c.n++ }         // 指针接收者
```

### 2.2 何时选哪个

**用指针接收者**：
- 方法要修改接收者状态（`Inc`）
- 接收者是大 struct（>~64 字节），值传递成本高
- 类型本身用作"实例"语义（一个 user 不该被复制成两个不同 user）

**用值接收者**：
- 接收者是小值类型（int、string、小 struct）
- 类型是"不可变值"语义（time.Time、几何点）
- map / slice / chan 本身就是引用语义，方法值接收者也只是复制 header

**保持一致**：同一个类型的所有方法**接收者类型应统一**——要么全值，要么全指针。混用会导致方法集和接口实现的隐患。

### 2.3 调用语法是糖

```go
c := Counter{}
c.Inc()      // 等价于 (&c).Inc()，编译器自动取址
(&c).Inc()   // 显式形式
```

这只在 `c` **可寻址**时生效。看引言谜题第四问：

```go
m := map[string]Counter{"x": {}}
m["x"].Inc()   // ❌ cannot call pointer method on m["x"]
```

`m["x"]` 不可寻址，编译器无法 `&m["x"]`，调用失败。修正：取出来改完写回，或用 `map[string]*Counter`。

---

## 第三章：方法集与接口实现

### 3.1 方法集规则

对于类型 `T`：

- `T` 的方法集 = 所有 `func (T) M()`
- `*T` 的方法集 = 所有 `func (T) M()` ∪ 所有 `func (*T) M()`

也就是说，**指针类型拥有所有方法（值方法 + 指针方法）；值类型只拥有值方法**。

### 3.2 接口实现的连锁后果

```go
type Adder interface { Inc() }

var a Adder = Counter{}     // ❌ Counter 没有 Inc（Inc 是 *Counter 的方法）
var a Adder = &Counter{}    // ✅
```

回到引言谜题：第①问就是这个原因。`Counter{}` 的方法集不含 `Inc()`，所以不满足 `Adder`。

### 3.3 接口赋值时的隐式约束

```go
type Stringer interface { String() string }

type Foo struct{}
func (f Foo) String() string  { return "value" }
func (f *Foo) String() string { return "pointer" }   // 不能同时存在
```

一个类型同名方法只能有一个接收者形式（值或指针），不能两个都有。

### 3.4 强制实现检查

```go
// 编译期断言：*Counter 必须实现 Adder
var _ Adder = (*Counter)(nil)
```

把这一行放在源文件顶部。任何时候 `*Counter` 不满足 `Adder`，编译失败。是写库时的标准防御。

---

## 第四章：指针的进阶使用

### 4.1 指针与逃逸

```go
func makeCounter() *Counter {
    c := Counter{}
    return &c   // c 逃逸到堆
}
```

返回局部变量的地址会让该变量**逃逸到堆**——编译器自动延长生命周期。这是 Go 与 C 的关键区别（C 中这是 use-after-free）。

```bash
go build -gcflags="-m" main.go
# 输出：moved to heap: c
```

### 4.2 不要无意义传 *T

```go
// ❌ 反模式
func setAge(u *User, age int) {
    u.Age = age
}

// 实际很多场景应该：
func WithAge(u User, age int) User {
    u.Age = age
    return u
}
```

不可变值更新风格（"with" 模式）在并发代码中更安全。但要权衡——大 struct 还是用指针。

### 4.3 *bool *int 反模式

Java/C# 用 `Integer` 区分"未设置"和"零"。Go 里有人用 `*int`：

```go
type Config struct {
    Timeout *int   // nil = 未设置；非 nil 用值
}
```

**陷阱**：
- nil 检查 + 解引用是双重判空
- JSON 反序列化时区分"字段缺失"和"显式 0"有用，但其他场景过度设计
- 调用者难免忘了 nil 检查 → panic

替代方案：
- 用包装类型 `type Opt[T any] struct{ V T; Set bool }`
- 用 `Has` 字段：`type T struct { Timeout int; HasTimeout bool }`
- Go 1.24+ 的 `database/sql.Null[T]`

### 4.4 函数参数 *T 是否复制

不复制——传的是 8 字节指针。但**函数内修改 \*p 影响调用方的对象**。

```go
func growSlice(s *[]int) {
    *s = append(*s, 1)
}
```

`*[]int` 是常见但稍古怪的写法——slice header 本身不大，正常做法是返回新 slice：

```go
func growSlice(s []int) []int {
    return append(s, 1)
}
```

### 4.5 nil 接收者上调用方法

```go
type List struct {
    next *List
    val  int
}

func (l *List) Len() int {
    if l == nil { return 0 }   // 显式处理 nil
    return 1 + l.next.Len()
}

var l *List
fmt.Println(l.Len())   // 0，不 panic
```

`l.Len()` 等于 `(*List).Len(l)`，传 nil 是合法的（不解引用就不会 panic）。这是链表、树结构常用的简化技巧。

### 4.6 空结构体指针

```go
a := &struct{}{}
b := &struct{}{}
fmt.Println(a == b)   // 多数情况下 true
```

空 struct 大小为 0，runtime 让所有 `&struct{}{}` 指向同一个特殊地址 `runtime.zerobase`。**不要依赖这个**——某些场景可能不同（如逃逸到不同位置）。

---

## 第五章：常见模式

### 5.1 Constructor 函数

```go
func NewCounter(start int) *Counter {
    return &Counter{n: start}
}
```

返回指针让调用方共享同一实例。这是大多数 Go 类型的标准构造函数模式。

### 5.2 Builder 模式

```go
type Req struct {
    url, method string
    headers map[string]string
}

func (r *Req) Method(m string) *Req { r.method = m; return r }
func (r *Req) Header(k, v string) *Req {
    if r.headers == nil { r.headers = map[string]string{} }
    r.headers[k] = v
    return r
}

req := (&Req{url: "..."}).Method("POST").Header("X", "1")
```

链式调用要求每个方法都返回 `*Req`。指针接收者必不可少。

### 5.3 Options 模式（Functional Options）

```go
type Server struct { port int; tls bool }

type Option func(*Server)

func WithPort(p int) Option { return func(s *Server) { s.port = p } }
func WithTLS() Option       { return func(s *Server) { s.tls = true } }

func NewServer(opts ...Option) *Server {
    s := &Server{port: 80}
    for _, o := range opts { o(s) }
    return s
}

srv := NewServer(WithPort(8443), WithTLS())
```

Options 必须接 `*Server`，否则改不动状态。Go 生态最流行的可扩展构造模式。

---

## 第六章：与接口的微妙交互

### 6.1 接口存的是指针还是值

```go
var i interface{}

i = 42         // 接口存 (type=int, value=42 的副本)
i = &Counter{} // 接口存 (type=*Counter, value=指针)
```

interface 内部是 `(type, value)`——value 通常是指针大小。小值（如 int）可能直接内联到 value 槽位（取决于实现），大值则装箱到堆。

### 6.2 装箱触发逃逸

```go
func print(v interface{}) { fmt.Println(v) }

func main() {
    n := 42
    print(n)   // n 装箱到堆
}
```

`go build -gcflags=-m` 会看到 `n escapes to heap`。这是 `fmt.Println(...interface{})` 等"通用接口"函数有少量开销的原因。

### 6.3 nil interface 经典陷阱

```go
type MyErr struct{}
func (*MyErr) Error() string { return "bad" }

func find() error {
    var e *MyErr = nil
    return e             // 返回 interface(type=*MyErr, value=nil)，不是 nil interface
}

err := find()
fmt.Println(err == nil)  // false ！
```

interface 等于 nil 当且仅当 type 也是 nil。返回具体类型 nil 会产生"装了 nil 的非 nil interface"。

修正：
```go
func find() error {
    if condition {
        return &MyErr{}
    }
    return nil   // 直接 return nil
}
```

---

## 第七章：生产级最佳实践

1. **同类型方法接收者统一**：要么全值，要么全指针。
2. **大 struct（>64 字节）用指针接收者**：避免每次方法调用复制。
3. **小不可变类型用值接收者**：time.Time、Point、Color。
4. **map 中存 struct 要改字段就改用 map[K]\*V**：避免取出—改—写回。
5. **接口实现编译期断言**：`var _ I = (*T)(nil)`。
6. **不要用 *bool *int 做"可选字段"**：用 Has 标志或 Option 类型。
7. **链表/树结构方法对 nil receiver 健壮**：减少调用方判空。
8. **导出类型的字段是否导出取决于 API 设计**：不让外部修改就不导出。
9. **构造函数返回 *T**：让多次方法调用共享状态。
10. **`go vet` 启用 `assign`、`copylocks` 检查器**：捕获取址错误和锁复制。

---

## 第八章：常见陷阱清单

### ❌ 陷阱 1：map 值不可寻址
```go
m["x"].Inc()    // 编译错误
&m["x"]         // 编译错误
m["x"].Field = 1 // 编译错误
```
修正：`v := m["x"]; v.X = 1; m["x"] = v` 或 `map[string]*T`。

### ❌ 陷阱 2：值接收者修改无效
```go
func (c Counter) Inc() { c.n++ }   // 修改副本，调用方看不到
```

### ❌ 陷阱 3：用接口期望值类型却传指针
```go
var s fmt.Stringer = &Foo{}   // 没问题
var s fmt.Stringer = Foo{}    // 仅当 (Foo).String 存在时
```

### ❌ 陷阱 4：返回具体类型 nil 装箱
```go
return (*MyErr)(nil)   // 非 nil interface
```

### ❌ 陷阱 5：循环里取地址
```go
for _, v := range items {
    save(&v)   // go.mod 声明 < 1.22 时所有 &v 是同一个地址；Go 1.22+ 已修复但模式仍易踩
}
```
修正：`v := v` 或 `save(&items[i])`。

### ❌ 陷阱 6：函数返回栈指针的"老 C 错觉"
```go
func bad() *int {
    x := 42
    return &x   // 在 C 是 use-after-free；在 Go 是合法（自动逃逸）
}
```
Go 中合法，但要意识到这是堆分配。

### ❌ 陷阱 7：值复制锁
```go
type Cache struct { mu sync.Mutex; data map[K]V }
c1 := Cache{}
c2 := c1   // copylocks 警告：c2 持有 c1.mu 的副本
```

---

## 第九章：练习题

**练习 1**：以下代码是否合法？如果合法，输出是什么？
```go
type T struct{ n int }
func (t T) Inc()  { t.n++ }
func (t *T) Inc2(){ t.n++ }

t := T{}
t.Inc(); t.Inc()
t.Inc2(); t.Inc2()
fmt.Println(t.n)
```

**练习 2**：能否让 `var i fmt.Stringer = MySlice{}` 编译通过？给出方案。

**练习 3**：分析此函数有何问题：
```go
type Vec struct{ data []float64 }
func (v Vec) Add(other Vec) {
    for i := range v.data { v.data[i] += other.data[i] }
}
```

**练习 4**：为何 `m[k]++` 合法但 `m[k].Field = x` 编译错误？

**练习 5**：实现一个 `*BinaryTree` 类型，支持 `Insert` 和 `InOrder` 方法。要求 `(*BinaryTree)(nil).InOrder()` 不 panic。

---

## 参考答案

**练习 1**：合法。`t.Inc()` 值接收者，修改副本不影响；`t.Inc2()` 指针接收者，Go 自动取址 → 修改原 t。最终输出 `2`。

**练习 2**：MySlice 是值类型 slice。让它实现 Stringer：
```go
type MySlice []int
func (s MySlice) String() string { return fmt.Sprint([]int(s)) }
```
**值接收者**，因为 MySlice 本身是 slice header（引用语义），且不需要修改自身。

**练习 3**：值接收者，`v` 和 `v.data` 是底层数组共享的——所以 `v.data[i] += ...` **真的能改**原数组。但代码可读性差且容易让读者误以为副本无效。建议改成 `func (v *Vec) Add(other Vec)`，更清晰。

**练习 4**：`m[k]++` 是 Go 的特殊语法糖——展开成"读出值、加 1、写回"，等价于 `m[k] = m[k] + 1`。但 `m[k].Field = x` 没有这种糖（编译器不展开），且 `m[k]` 不可寻址，无法直接修改字段。

**练习 5**：
```go
type BinaryTree struct {
    val         int
    left, right *BinaryTree
}

func (t *BinaryTree) Insert(v int) *BinaryTree {
    if t == nil { return &BinaryTree{val: v} }
    if v < t.val { t.left = t.left.Insert(v) }
    else { t.right = t.right.Insert(v) }
    return t
}

func (t *BinaryTree) InOrder() []int {
    if t == nil { return nil }
    out := t.left.InOrder()
    out = append(out, t.val)
    out = append(out, t.right.InOrder()...)
    return out
}
```

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| `&` / `*` | 取地址 / 解引用；不支持指针算术 |
| `new(T)` | 分配并返回 *T 指向零值；用 `&T{}` 替代多数场景 |
| 可寻址性 | 变量、slice 元素、可寻址 struct 字段可；map value、字面量、返回值不可 |
| 值 vs 指针接收者 | 修改状态、大 struct 用指针；小不可变用值；同类型保持一致 |
| 方法集 | T 只含值方法；*T 含全部 |
| 调用糖 | `c.Inc()` 自动 `(&c).Inc()`，前提是 `c` 可寻址 |
| nil 接收者 | 合法只要方法不解引用；链表树常用技巧 |
| nil interface 陷阱 | 装了 nil 的接口 != nil interface |

下一篇 **G08 — 精通 Go 接口：itab 与类型断言** 会拆开接口的 16 字节结构、讲清 `itab` 缓存、类型断言的运行时实现、type switch 的优化。

---

