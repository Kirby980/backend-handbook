# 精通 Go 泛型：类型参数、约束与性能

> 课程编号：G09
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Generics 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：为什么 Go 等了 10 年才有泛型

```go
func Map[T, U any](in []T, f func(T) U) []U {
    out := make([]U, len(in))
    for i, v := range in { out[i] = f(v) }
    return out
}

nums := []int{1, 2, 3}
doubled := Map(nums, func(n int) int { return n * 2 })   // [2 4 6]
strs := Map(nums, strconv.Itoa)                          // ["1" "2" "3"]
```

Go 1.0 发布于 2012 年；泛型在 Go 1.18 才推出（2022 年）。十年的争论与设计迭代，最终拿出的方案叫 **类型参数（type parameters）+ 类型集（type sets）**。本章讲清楚语法、约束系统、实现策略（GC shape stenciling）、以及"什么时候**不**该用泛型"。

---

## 第一章：类型参数语法

### 1.1 泛型函数

```go
func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}

Min(3, 5)           // 类型推断: T = int
Min[int](3, 5)      // 显式指定
Min(3.14, 2.71)     // T = float64
Min("a", "b")       // T = string
```

`[T constraints.Ordered]` 是类型参数声明：`T` 是名字，`constraints.Ordered` 是约束。

### 1.2 多个类型参数

```go
func ToMap[K comparable, V any](keys []K, vals []V) map[K]V {
    m := make(map[K]V, len(keys))
    for i, k := range keys { m[k] = vals[i] }
    return m
}
```

### 1.3 泛型类型

```go
type Stack[T any] struct {
    data []T
}

func (s *Stack[T]) Push(v T) { s.data = append(s.data, v) }
func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.data) == 0 { return zero, false }
    n := len(s.data) - 1
    v := s.data[n]
    s.data = s.data[:n]
    return v, true
}

s := &Stack[int]{}
s.Push(1); s.Push(2)
```

注意方法签名 `(s *Stack[T]) Push(v T)`——必须把类型参数 T 也加上。

### 1.4 实例化

```go
type IntStack = Stack[int]   // 类型别名
var s IntStack
```

`Stack[int]` 是一个完整的、独立的类型。`Stack[int]` 和 `Stack[string]` 是完全不同的类型，互相不能赋值。

### 1.5 泛型类型别名（Go 1.24+）

上面 `type IntStack = Stack[int]` 是给"具象类型"起别名。**Go 1.24 (2025-02) 把类型别名本身也允许带类型参数**，即 generic type aliases GA：

```go
// 一个保留参数的泛型别名
type ComparableStack[T comparable] = Stack[T]

// 用作约束/桥接两套包名时非常方便
type Set[T comparable] = map[T]struct{}
```

讨论了三年多才落地。**典型用途**：库重命名/迁移时保持旧包名兼容、为长签名的泛型类型起短别名、跨模块桥接两套类型参数命名风格。

> Go 1.24 release notes: *"a type alias may be parameterized like a defined type"*。短期内可用 `GOEXPERIMENT=noaliastypeparams` 关掉，但 **Go 1.25 已移除该开关**——现在是默认行为。

---

## 第二章：约束（constraints）

### 2.1 约束本身是接口

```go
type Number interface {
    int | int64 | float32 | float64
}

func Sum[T Number](xs []T) T {
    var s T
    for _, x := range xs { s += x }
    return s
}
```

`Number` 接口里用 `|` 列出允许的类型集合。这种"类型集语法"是 Go 1.18 对接口的扩展——以前接口只描述方法，现在还可以描述允许的底层类型。

### 2.2 预声明约束

| 名字 | 含义 |
|---|---|
| `any` | `interface{}` 别名（无约束） |
| `comparable` | 支持 == 和 != |

### 2.3 标准库 constraints 包（已升级）

Go 1.21 之前用 `golang.org/x/exp/constraints`：

```go
type Ordered interface {
    Integer | Float | ~string
}
```

Go 1.21+ 把数值约束迁移到 `cmp` 包：`cmp.Ordered` 内置。`slices`、`maps` 包接口签名也直接使用 `cmp.Ordered`。

### 2.4 `~` 操作符（底层类型）

```go
type MyInt int

func Add[T ~int](a, b T) T { return a + b }

Add(MyInt(1), MyInt(2))   // OK，因为 ~int 包含底层类型是 int 的所有类型
Add(1, 2)                  // OK
```

不带 `~`：只接受 `int` 自身；带 `~`：接受任何 `type X int` 派生类型。

### 2.5 方法约束

```go
type Stringer interface { String() string }

func Print[T Stringer](v T) { fmt.Println(v.String()) }
```

约束接口可以混合类型集和方法集（但有些组合限制，需查规范）。

### 2.6 嵌入约束

```go
type Float interface{ ~float32 | ~float64 }
type Int interface{ ~int | ~int64 }
type Number interface{ Float; Int }   // 嵌入两个，等价于并集
```

---

## 第三章：类型推断

### 3.1 函数实参推断

```go
func Max[T constraints.Ordered](a, b T) T { ... }

Max(3, 5)         // 推断 T=int
Max(3, 5.0)       // ❌ 不能从 (int, float) 推断
Max[float64](3, 5.0)   // 显式
```

### 3.2 推断不出来时

```go
type S[T any] struct{ Val T }

func New[T any]() S[T] { return S[T]{} }

x := New()           // ❌ 无法推断 T
x := New[int]()      // OK
```

返回值不参与推断（只看参数）。需要显式实例化。

### 3.3 部分推断

```go
func Map[T, U any](in []T, f func(T) U) []U { ... }

Map[int, string]([]int{1,2}, strconv.Itoa)   // 显式
Map([]int{1,2}, strconv.Itoa)                 // 完全推断
Map[int]([]int{1,2}, strconv.Itoa)            // 部分（指定 T，推断 U）
```

---

## 第四章：实现策略——GC Shape Stenciling

### 4.1 三种主流策略

- **Monomorphization**（C++、Rust）：每个具体类型生成一份代码副本——零开销但 binary 膨胀
- **Type erasure**（Java、JS）：编译期擦除类型，运行时全部装箱——binary 小但有装箱开销
- **GC Shape Stenciling**（Go）：按"GC 形状"分组生成——折中

### 4.2 什么是 GC shape

GC 关心的类型属性：大小、对齐、哪些字段是指针。**所有具有相同 GC shape 的类型共享一份机器码**。例如：

- `*int`、`*string`、`*Foo` 都是"8 字节单指针"——共享一份代码
- `int`、`int64`、`float64` 都是"8 字节非指针"——共享一份
- 任意 `T any` 实际上对应大约一个 dictionary + 共享代码

### 4.3 Dictionary（字典）

调用泛型函数时，编译器隐式传入一个"字典"，里头包含类型信息（具体类型大小、关键方法地址等）。runtime 根据 dictionary 完成正确行为。

```go
func F[T any](v T) { ... }
// 实际等价
func F_shape(dict *dictionary, v Tshape) { ... }
```

### 4.4 性能含义

- 泛型函数调用比直接调用稍慢（额外的 dictionary 间接），但通常持平
- 比 `interface{}` 版**不需要装箱**（值类型直接传 shape 槽位）
- 比 monomorphization 版略慢（多一次 dict 索引），但 binary 不膨胀

### 4.5 实测对照

| 实现 | 1M Sum int 调用 | 1M Sum int 字符串拼接 |
|---|---|---|
| 具体类型版 | ~0.5ms | ~1ms |
| 泛型版 | ~0.6ms | ~1.2ms |
| interface{} 版 | ~3ms（装箱） | ~5ms |

热点路径泛型 vs 具体类型差距在 10-30%；interface{} 因装箱可能慢 5-10 倍。

---

## 第五章：泛型方法的限制

### 5.1 方法不能引入新类型参数

```go
type Stack[T any] struct { ... }

// ❌ 不允许：方法不能有自己的类型参数
func (s *Stack[T]) MapTo[U any](f func(T) U) Stack[U] { ... }
```

设计决策：Go 团队认为方法上加类型参数会引入显著的类型检查复杂度，暂时不开放。**替代方案**：

```go
func MapStack[T, U any](s *Stack[T], f func(T) U) *Stack[U] { ... }
```

用顶层泛型函数代替。

### 5.2 泛型类型只能用类型参数声明的 T

```go
func (s *Stack[T]) Push(v T)   // T 必须是 Stack 的类型参数
```

### 5.3 不能在接口里写泛型方法

```go
type Container interface {
    Put[T any](v T)   // ❌
}
```

仍然是 Go 团队保留的设计空间。

---

## 第六章：实战模式

### 6.1 容器

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(v T)         { s[v] = struct{}{} }
func (s Set[T]) Has(v T) bool    { _, ok := s[v]; return ok }
func (s Set[T]) Remove(v T)      { delete(s, v) }
func (s Set[T]) Len() int        { return len(s) }
```

### 6.2 Map/Filter/Reduce

```go
func Map[T, U any](s []T, f func(T) U) []U {
    out := make([]U, len(s))
    for i, v := range s { out[i] = f(v) }
    return out
}

func Filter[T any](s []T, pred func(T) bool) []T {
    out := s[:0]
    for _, v := range s {
        if pred(v) { out = append(out, v) }
    }
    return out
}

func Reduce[T, U any](s []T, init U, f func(U, T) U) U {
    acc := init
    for _, v := range s { acc = f(acc, v) }
    return acc
}
```

### 6.3 Optional / Result

```go
type Optional[T any] struct {
    value T
    set   bool
}

func Some[T any](v T) Optional[T] { return Optional[T]{value: v, set: true} }
func None[T any]() Optional[T]    { return Optional[T]{} }

func (o Optional[T]) Get() (T, bool) { return o.value, o.set }
```

### 6.4 Generic Pool

```go
type Pool[T any] struct {
    pool sync.Pool
    new  func() T
}

func NewPool[T any](newFn func() T) *Pool[T] {
    return &Pool[T]{
        pool: sync.Pool{New: func() any { return newFn() }},
        new:  newFn,
    }
}

func (p *Pool[T]) Get() T  { return p.pool.Get().(T) }
func (p *Pool[T]) Put(v T) { p.pool.Put(v) }
```

### 6.5 Functional Options 泛型版

```go
type Option[T any] func(*T)

func With[T any](opts ...Option[T]) T {
    var t T
    for _, o := range opts { o(&t) }
    return t
}
```

---

## 第七章：泛型的局限与"何时不该用"

### 7.1 标准库不会大改

`io.Reader`、`net/http.Handler` 等都不会重写为泛型，因为：
- 兼容性
- 接口已经足够好
- 泛型不会带来显著优势

### 7.2 不要为了"看起来高级"用泛型

```go
// ❌ 过度泛型
func Print[T any](v T) { fmt.Println(v) }
// 等价于 fmt.Println，没收益
```

### 7.3 不要泛型化没有共性的类型

```go
// ❌ 牵强
func Process[T A | B | C](v T) { ... }
// A B C 没有共同行为，函数内 type switch 反而难看
```

### 7.4 反思 checklist

- 是否有 2+ 类型在用同样算法？
- 是否避免了 interface{} 装箱？
- 是否让 API 更清晰？

三个都"是"才值得引入。

---

## 第八章：生产级最佳实践

1. **小心 `any` 滥用**：约束越具体，可读性和安全性越高。
2. **优先 `~` 包含派生类型**：除非真有理由排除。
3. **泛型与接口能解决同样问题时优先泛型**：避免装箱。
4. **不要给方法加自己的类型参数**：现在不支持，未来语义可能变。
5. **泛型类型的方法签名要写完整**：`(s *Stack[T]) Push(v T)`。
6. **测试覆盖多种实例化**：T=int, T=string, T=自定义类型。
7. **不要把泛型当 macro**：抽象要值得。
8. **类型推断不通过就显式实例化**：`F[int](...)`。
9. **constraints 包是 1.21 后的 cmp + slices + maps**：用标准库，避免 `x/exp`。
10. **泛型不是性能银弹**：benchmark 验证再决定。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：约束方法 + 类型集冲突
```go
type Foo interface {
    int | string
    Hello()   // int 没有 Hello 方法，编译期约束失败
}
```

### ❌ 陷阱 2：泛型类型方法忘记带 T
```go
func (s *Stack) Push(v T) {}   // ❌ 应该是 (s *Stack[T])
```

### ❌ 陷阱 3：方法引入新类型参数
```go
func (s *Stack[T]) Map[U any](f func(T) U) {}   // ❌
```

### ❌ 陷阱 4：误以为不同实例化的类型相同
```go
var a Stack[int]
var b Stack[int64]
a = b   // ❌ 类型不同
```

### ❌ 陷阱 5：把 any 当 interface{} 方法的替代
```go
func F[T any](v T) {
    v.String()   // ❌ T 没有 String 方法
}
```
要约束：`[T Stringer]`。

### ❌ 陷阱 6：性能假设
"泛型一定比 interface{} 快"——多数情况是的，但不是所有，特别是涉及大对象传值时仍可能复制。

### ❌ 陷阱 7：循环引用约束
```go
type A[T any] interface { B() T }
type B[T any] interface { A() T }
// 互相引用过深时编译器报错
```

---

## 第十章：练习题

**练习 1**：写一个泛型 `Contains[T comparable](s []T, v T) bool`。

**练习 2**：以下能否编译？为什么？
```go
type Container interface {
    Get() any
}
func F[T any](c Container) T { return c.Get().(T) }
```

**练习 3**：实现 `OrderedMap[K comparable, V any]`：按插入顺序遍历的 map。

**练习 4**：用泛型实现一个 LRU cache。

**练习 5**：解释 `[T ~[]int]` 和 `[T []int]` 的差异。

---

## 参考答案

**练习 1**：
```go
func Contains[T comparable](s []T, v T) bool {
    for _, x := range s {
        if x == v { return true }
    }
    return false
}
// 或 Go 1.21+: slices.Contains
```

**练习 2**：能编译。F 接受任何 Container，从其 Get() 取出 any，断言为 T 返回。但调用时要保证 Get 返回的实际类型与 T 匹配，否则 panic。

**练习 3**：
```go
type OrderedMap[K comparable, V any] struct {
    keys []K
    data map[K]V
}
func NewOrderedMap[K comparable, V any]() *OrderedMap[K, V] {
    return &OrderedMap[K, V]{data: map[K]V{}}
}
func (m *OrderedMap[K, V]) Set(k K, v V) {
    if _, ok := m.data[k]; !ok { m.keys = append(m.keys, k) }
    m.data[k] = v
}
func (m *OrderedMap[K, V]) Range(f func(K, V) bool) {
    for _, k := range m.keys {
        if !f(k, m.data[k]) { return }
    }
}
```

**练习 4**：思路：双向链表 + map。略，可基于 container/list 包装。

**练习 5**：`[T ~[]int]` 接受 `[]int` 以及 `type X []int`；`[T []int]` 只接受 `[]int` 本身。日常使用应加 `~`。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 类型参数 | `[T any]`；可多个；可有约束 |
| 约束 | 接口扩展为"类型集"；`~T` 含派生类型 |
| 预声明 | `any` / `comparable`；其他在 cmp/slices/maps |
| 类型推断 | 从参数推断；返回值不参与 |
| 实现策略 | GC shape stenciling + dictionary |
| 性能 | 比 interface{} 快（无装箱）；比 monomorph 略慢 |
| 限制 | 方法不能引入新类型参数；接口不能写泛型方法 |
| 适用场景 | 容器、算法、Option 类型；避免过度抽象 |

下一篇 **G10 — 精通 Go 错误处理与 panic/recover** 会讲清 Go 1.13+ wrapping、errors.Is/As、Join、何时该 panic、HTTP middleware 的恢复模式。

---

