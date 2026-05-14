# 精通 Go 反射（reflect）

> 课程编号：G24
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Reflection 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：反射的代价与价值

```go
type User struct{ Name string; Age int }

func main() {
    u := User{Name: "Alice", Age: 30}
    t := reflect.TypeOf(u)
    v := reflect.ValueOf(u)
    for i := 0; i < t.NumField(); i++ {
        f := t.Field(i)
        fmt.Printf("%s = %v (%s)\n", f.Name, v.Field(i), f.Type)
    }
}
// Name = Alice (string)
// Age = 30 (int)
```

`encoding/json`、`encoding/xml`、ORM、依赖注入框架——都靠 reflect 工作。但反射有代价：方法调用 ~100ns（直接调用 1ns）、增加 GC 压力、丧失编译期类型安全。本章拆开反射的内部结构与正确用法。

---

## 第一章：Type 与 Value

### 1.1 入口

```go
t := reflect.TypeOf(x)    // 取类型信息
v := reflect.ValueOf(x)   // 取值（与 x 同步的 wrapper）
```

`Type` 是接口（描述类型）；`Value` 是 struct（包装值）。两者都是 reflect 的核心。

### 1.2 三定律（Go 官方）

1. 反射从接口值映射到反射对象
2. 反射对象可映射回接口值
3. 要修改反射对象，原值必须可寻址（addressable）

### 1.3 实际类型

```go
v := reflect.ValueOf(u)
fmt.Println(v.Kind())   // struct
fmt.Println(v.Type())   // main.User
```

`Kind()` 是底层种类（struct、int、string、ptr、slice、map、func 等）；`Type()` 是具体类型。

---

## 第二章：常见操作

### 2.1 读字段

```go
type User struct{ Name string; Age int }
u := User{Name: "A", Age: 30}
v := reflect.ValueOf(u)
fmt.Println(v.FieldByName("Name").String())
fmt.Println(v.Field(0).Interface())
```

### 2.2 写字段

```go
u := &User{}
v := reflect.ValueOf(u).Elem()   // 解引用到 User
v.FieldByName("Name").SetString("Alice")
```

注意：必须传**指针** + 调用 `Elem()`。否则 Value 不可寻址，Set 会 panic。

### 2.3 调用方法

```go
v := reflect.ValueOf(obj)
m := v.MethodByName("Save")
out := m.Call([]reflect.Value{reflect.ValueOf(ctx)})
```

`Call` 接收 `[]Value` 参数列表，返回 `[]Value`。

### 2.4 类型断言

```go
t := reflect.TypeOf(x)
if t.Kind() == reflect.Slice && t.Elem().Kind() == reflect.Int {
    // x is []int
}
```

### 2.5 创建零值

```go
t := reflect.TypeOf(User{})
v := reflect.New(t)   // *User，零值
```

---

## 第三章：内部结构（简化）

### 3.1 reflect.Type 是接口

```go
type Type interface {
    Name() string
    Kind() Kind
    NumField() int
    Field(i int) StructField
    Implements(u Type) bool
    // ... 几十个方法
}
```

底层是 runtime 的 `*runtime._type` 包装。

### 3.2 reflect.Value 是 struct

```go
type Value struct {
    typ_ *abi.Type      // 类型指针
    ptr  unsafe.Pointer // 数据指针
    flag                // 内部标志（kind、addressable、readonly）
}
```

Value 包含三段：类型、数据指针、flag。

### 3.3 性能影响

- 每次 `ValueOf` 创建一个 Value 对象（~24 字节）—— 频繁分配 → GC 压力
- 字段访问通过类型表查找
- 方法调用通过函数指针 + 参数装箱

---

## 第四章：反射的性能数据

| 操作 | 直接 | 反射 |
|---|---|---|
| 读 struct 字段 | 0.5 ns | 30-50 ns |
| 调用方法 | 1 ns | 200-300 ns |
| 创建 struct | 5 ns | 50-100 ns |
| Slice append | 5 ns | 100-200 ns |

每次反射操作慢 50-100 倍是常态。

---

## 第五章：典型用例

### 5.1 JSON marshal

```go
v := reflect.ValueOf(obj)
for i := 0; i < v.NumField(); i++ {
    f := v.Field(i)
    tag := v.Type().Field(i).Tag.Get("json")
    // 输出 tag: f.Interface()
}
```

`encoding/json` 的核心。

### 5.2 ORM struct → SQL

```go
t := reflect.TypeOf(u)
fields := []string{}
for i := 0; i < t.NumField(); i++ {
    if col := t.Field(i).Tag.Get("db"); col != "" {
        fields = append(fields, col)
    }
}
// 生成 SELECT field1, field2 FROM ...
```

### 5.3 依赖注入

```go
type Container struct {
    instances map[reflect.Type]reflect.Value
}
func (c *Container) Get(t reflect.Type) any {
    return c.instances[t].Interface()
}
```

### 5.4 通用 deepcopy

```go
func DeepCopy(src any) any {
    v := reflect.ValueOf(src)
    dst := reflect.New(v.Type()).Elem()
    copyValue(dst, v)
    return dst.Interface()
}
```

### 5.5 验证标签

```go
type Form struct{ Email string `validate:"required,email"` }

t := reflect.TypeOf(form)
for i := 0; i < t.NumField(); i++ {
    rule := t.Field(i).Tag.Get("validate")
    validate(rule, v.Field(i).Interface())
}
```

---

## 第六章：减少反射开销的技巧

### 6.1 缓存 reflect 结果

```go
var typeCache sync.Map   // map[reflect.Type]*Schema

func getSchema(t reflect.Type) *Schema {
    if s, ok := typeCache.Load(t); ok { return s.(*Schema) }
    s := buildSchema(t)
    typeCache.Store(t, s)
    return s
}
```

每个类型只反射一次，之后直接走缓存数据结构（如预提取的 fieldIndex / setter 函数）。

### 6.2 代码生成

`easyjson`、`mockgen`、`sqlc` 等工具读取 struct → 生成专用代码：

```go
// 生成
func (u *User) MarshalJSON() ([]byte, error) {
    b := make([]byte, 0, 64)
    b = append(b, `{"name":"`...)
    b = append(b, u.Name...)
    b = append(b, `","age":`...)
    b = strconv.AppendInt(b, int64(u.Age), 10)
    b = append(b, '}')
    return b, nil
}
```

完全去反射，性能与 hand-written 一致。

### 6.3 unsafe.Pointer

reflect 内部为了类型安全做了大量检查。如果你能保证类型一致，用 unsafe 直接读写：

```go
*(*string)(unsafe.Pointer(uintptr(ptr) + offset)) = "value"
```

慎用——但 fast JSON 库都这么做。

### 6.4 类型 switch 优先

```go
switch v := x.(type) {
case int:    return strconv.Itoa(v)
case string: return v
case Stringer: return v.String()
default:     return fmt.Sprintf("%v", x)
}
```

比反射快很多——多数场景几个常见类型就够。

### 6.4 reflect.TypeAssert —— 泛型 + 反射的桥（Go 1.25+）

旧痛点：`reflect.Value.Interface().(T)` 这套类型断言**会装箱**——每次断言都新分配一个 `interface{}`。**Go 1.25** 加了泛型版本 `reflect.TypeAssert`：

```go
func TypeAssert[T any](v Value) (T, bool)
```

零额外装箱，且类型推断让代码更短：

```go
v := reflect.ValueOf(getSomething())

// 旧：v.Interface().(string) — 装箱一次
// 新：直接返回 (string, bool)
if s, ok := reflect.TypeAssert[string](v); ok {
    use(s)
}
```

`reflect.TypeFor[T]()`（Go 1.22+）拿 `reflect.Type` 不用先 `reflect.New`，配合 TypeAssert 就拼出了完整的"泛型 ↔ 反射"桥。

> 参考：[Go 1.25 release notes — reflect](https://go.dev/doc/go1.25#reflect)。

---

## 第七章：反射陷阱

### 7.1 不可寻址

```go
u := User{}
v := reflect.ValueOf(u)
v.FieldByName("Name").SetString("X")   // panic
```

修：传指针。

```go
v := reflect.ValueOf(&u).Elem()
v.FieldByName("Name").SetString("X")   // OK
```

### 7.2 私有字段不可设

```go
type T struct{ name string }   // 小写
v.FieldByName("name").SetString("x")   // panic
```

reflect 默认遵守可见性。要绕过：`reflect.NewAt(t, unsafe.Pointer(&v.field))`（不推荐）。

### 7.3 nil interface

```go
var x any
v := reflect.ValueOf(x)
fmt.Println(v.Kind())   // Invalid
```

nil interface 的 Value 是 zero Value——任何操作 panic。先检查 `v.IsValid()`。

### 7.4 调用方法时参数数量错

```go
m.Call([]reflect.Value{}) 缺参数 → panic
```

### 7.5 nil pointer 调方法 panic

```go
var p *T
v := reflect.ValueOf(p)
v.MethodByName("Foo").Call(...)   // 看方法实现
```

如果方法不解引用 receiver 不 panic；否则 panic。

---

## 第八章：alternatives—— 优先用

### 8.1 类型 switch

```go
switch v := x.(type) { ... }
```

90% 的"需要反射"场景其实是几个具体类型 → switch 更快更清晰。

### 8.2 interface 方法

```go
type Stringer interface { String() string }
```

让类型实现接口，调用时不走反射。

### 8.3 泛型

```go
func F[T any](v T) { ... }
```

Go 1.18+，零开销 + 类型安全。

### 8.4 代码生成

复杂场景（JSON、ORM）用 `go generate` 一次生成、无运行时反射。

---

## 第九章：生产级最佳实践

1. **能用 switch / 接口 / 泛型解决就不要反射**。
2. **必要的反射缓存 reflect.Type 的元数据**。
3. **代码生成（easyjson、sqlc 等）替代框架反射**。
4. **Tag 解析放缓存**：每次解析 tag 字符串成本高。
5. **修改值要 reflect.ValueOf(&x).Elem()**。
6. **检查 IsValid / CanSet 防 panic**。
7. **私有字段不要尝试改**：违反可见性 + 易碎。
8. **`reflect.DeepEqual` 慎用热路径**：比 == 慢 50-100 倍。
9. **设计 API 接受具体类型 + 泛型**：减少调用方走 reflect。
10. **profile 看反射开销**：CPU profile 中 reflect.* 占比高 = 优化方向。

---

## 第十章：常见陷阱清单

### ❌ 陷阱 1：reflect 修改值类型副本无效
```go
v := reflect.ValueOf(u)
v.Field(0).SetString("X")   // panic
```

### ❌ 陷阱 2：访问私有字段
```go
v.FieldByName("private").Set(...)   // panic
```

### ❌ 陷阱 3：以为 reflect 是免费的
单次还好，热点路径 50-100x 慢。

### ❌ 陷阱 4：反复 ValueOf 同一对象
```go
for i := 0; i < N; i++ {
    reflect.ValueOf(x).Field(0).Interface()
}
```
缓存 reflect.Value 或预提取字段 index。

### ❌ 陷阱 5：忘了 IsValid
nil interface 的 Value 任何操作 panic。

### ❌ 陷阱 6：自以为快的 reflect tricks
某些"优化"反而慢。benchmark 验证。

### ❌ 陷阱 7：用反射做泛型
Go 1.18+ 有真正的泛型，反射只在动态类型不确定时用。

---

## 第十一章：练习题

**练习 1**：写一个 `SetField(obj any, name string, value any) error`，用反射设置 struct 字段。

**练习 2**：分析以下函数的开销：
```go
func Process(items []any) {
    for _, x := range items {
        v := reflect.ValueOf(x)
        for i := 0; i < v.NumField(); i++ {
            handle(v.Field(i).Interface())
        }
    }
}
```

**练习 3**：用反射实现一个简单的"struct → map[string]any" 函数。

**练习 4**：以下代码哪里出错？
```go
type T struct{ name string }
t := T{}
reflect.ValueOf(&t).Elem().FieldByName("name").SetString("x")
```

**练习 5**：何时该用 reflect.DeepEqual，何时该用自定义 Equal？

---

## 参考答案

**练习 1**：
```go
func SetField(obj any, name string, value any) error {
    v := reflect.ValueOf(obj)
    if v.Kind() != reflect.Ptr { return errors.New("not ptr") }
    v = v.Elem()
    f := v.FieldByName(name)
    if !f.IsValid() { return errors.New("no field") }
    if !f.CanSet() { return errors.New("cannot set") }
    f.Set(reflect.ValueOf(value).Convert(f.Type()))
    return nil
}
```

**练习 2**：
- `items` 每个 any 装箱 + 反射 ValueOf 创建对象 + 每字段 Interface() 装箱
- 整体可能比直接代码慢 100 倍。建议泛型或类型 switch。

**练习 3**：
```go
func Struct2Map(obj any) map[string]any {
    v := reflect.ValueOf(obj)
    if v.Kind() == reflect.Ptr { v = v.Elem() }
    t := v.Type()
    m := make(map[string]any, t.NumField())
    for i := 0; i < t.NumField(); i++ {
        f := t.Field(i)
        if !f.IsExported() { continue }
        m[f.Name] = v.Field(i).Interface()
    }
    return m
}
```

**练习 4**：私有字段 `name` 即使通过 reflect 拿到也不可 Set，CanSet 返回 false。SetString panic。

**练习 5**：
- DeepEqual：泛用、慢、可比较任意结构（含 slice/map）
- 自定义 Equal：快、易控（如忽略某字段、float 容差）
- 测试代码常用 DeepEqual；热点逻辑用自定义

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 入口 | TypeOf / ValueOf |
| 三定律 | interface ↔ reflect；修改需 addressable |
| Type vs Value | 接口 vs struct |
| Kind | 底层种类（struct/int/ptr/...） |
| 性能 | 50-100x 慢；缓存元数据 |
| 修改值 | 必须 ValueOf(&x).Elem() |
| 替代 | 类型 switch / 接口 / 泛型 / 代码生成 |
| DeepEqual | 慢；热路径不用 |

下一篇 **G25 — 精通 Go unsafe 包** 会讲清 unsafe.Pointer、//go:linkname 的用法、合法的指针变换规则、和何时 unsafe 是必须的（以及通常不该用）。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G24-精通-Go-反射-reflect.md`
