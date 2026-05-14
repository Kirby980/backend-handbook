# 精通 Go 切片：底层结构与扩容机制

> 课程编号：G03
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Slices 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：一段不该工作的代码

```go
package main

import "fmt"

func main() {
    a := []int{1, 2, 3, 4, 5}
    b := a[1:3]      // [2, 3]
    b = append(b, 99)
    fmt.Println(a)   // ① 输出什么？
    fmt.Println(b)   // ② 输出什么？

    b = append(b, 100, 101, 102)
    fmt.Println(a)   // ③ 这次又是什么？
}
```

第一个 `append` 不会扩容，会偷偷改写 `a`；第二个 `append` 触发扩容，从此 `b` 和 `a` 分道扬镳。如果你不熟悉这背后的"切片别名"和"扩容触发"，至少有一类生产 bug 在等你。本章把 `[]T` 拆开揉碎讲清楚。

---

## 第一章：SliceHeader 三元组

### 1.1 结构

```go
type SliceHeader struct {
    Data uintptr   // 指向底层数组首元素
    Len  int       // 当前长度
    Cap  int       // 容量（底层数组从 Data 起的可用元素数）
}
```

在 64 位机上每个切片占 **24 字节**（3 × 8）。

```go
import "unsafe"
fmt.Println(unsafe.Sizeof([]int(nil)))  // 24
```

### 1.2 三种切片状态

```go
var a []int       // nil slice: Data=nil, Len=0, Cap=0
b := []int{}      // empty slice: Data!=nil, Len=0, Cap=0
c := make([]int, 0)  // empty slice: 同上
```

`a == nil` 是 true，`b == nil` 是 false。但 `len(a) == len(b) == 0`，大多数操作行为一致——**`append`、`range`、`len`、`cap` 对 nil slice 完全合法**。所以"用 nil 还是 empty"几乎是审美问题，除非：

- JSON 序列化：nil → `null`；empty → `[]`
- 反射 `reflect.ValueOf(a).IsNil()` 区分

### 1.3 make 的三种姿势

```go
s := make([]int, 5)        // len=5, cap=5；元素初始化为 0
s := make([]int, 0, 10)    // len=0, cap=10；预分配避免后续扩容
s := make([]int, 5, 10)    // len=5, cap=10
```

**预分配是性能优化的第一条**。如果你知道最终大概要 N 个元素，写 `make([]T, 0, N)`，能省下 log₂(N) 次扩容（每次扩容都是 alloc + copy）。

---

## 第二章：append 与扩容规则

### 2.1 append 的基本语义

```go
s := []int{1, 2, 3}
s = append(s, 4)
```

如果 `len < cap`：在原底层数组的下一个位置写入，`len++`。**无新分配**。

如果 `len == cap`：分配一个**新的、更大的**底层数组，把原数据复制过去，写入新元素，返回新 slice（其 `Data` 已指向新数组）。**原底层数组不变**（旁路的别名 slice 看不到新元素）。

这就是为什么 **必须** 写 `s = append(s, x)` 而不是 `append(s, x)`——返回值可能完全是另一个 slice header。

### 2.2 Go 1.18 之前的扩容规则

- 如果旧 cap < 1024：新 cap = 旧 cap × 2
- 否则：新 cap = 旧 cap × 1.25

### 2.3 Go 1.18+ 的新规则（更平滑）

新规则从 256 开始引入"温和过渡"，公式大致是：

```
newcap = oldcap + (oldcap+3*256)/4   // 当 oldcap >= 256
```

新规则的曲线在 256 到 1024 之间介于 2x 和 1.25x 之间，更平滑。具体实现在 `runtime/slice.go` 的 `growslice`，建议读一遍。

### 2.4 用代码观察扩容轨迹

```go
package main

import "fmt"

func main() {
    var s []int
    prev := cap(s)
    for i := 0; i < 2000; i++ {
        s = append(s, i)
        if cap(s) != prev {
            fmt.Printf("len=%d cap=%d (jumped from %d)\n", len(s), cap(s), prev)
            prev = cap(s)
        }
    }
}
```

典型输出（Go 1.22 amd64）：

```
len=1 cap=1 (jumped from 0)
len=2 cap=2 (jumped from 1)
len=3 cap=4 (jumped from 2)
len=5 cap=8 (jumped from 4)
len=9 cap=16
len=17 cap=32
len=33 cap=64
len=65 cap=128
len=129 cap=256
len=257 cap=512
len=513 cap=848    ← 进入新公式
len=849 cap=1280
len=1281 cap=1792
...
```

### 2.5 内存对齐导致的"上取整"

要求的 cap 还会经过 `roundupsize` 上取整到内存分配器友好的尺寸（mspan 的 size class），所以实际 cap 可能比预期略大。

---

## 第三章：别名——切片最隐蔽的陷阱

### 3.1 共享底层数组

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:4]   // b = [2, 3, 4]
b[0] = 99
fmt.Println(a)   // [1, 99, 3, 4, 5]   ← a 也被改了
```

`a` 和 `b` 的 `Data` 指针指向同一块内存，只是起点和长度不同。

### 3.2 append 让别名不可预测

```go
a := make([]int, 3, 5)   // [0,0,0], cap=5
copy(a, []int{1, 2, 3})
b := a[:2]               // [1, 2], len=2, cap=5

b = append(b, 99)        // b=[1,2,99], len=3, cap=5
                          // 没有扩容，写入了 a 的第三位
fmt.Println(a)            // [1, 2, 99]   ← a 被改！

b = append(b, 100, 101, 102, 103)  // 触发扩容
b[0] = 0                  // 修改 b 不再影响 a
fmt.Println(a)            // [1, 2, 99]   不变
fmt.Println(b)            // [0, 2, 99, 100, 101, 102, 103]
```

这是真实生产代码里最难追的一类 bug。函数接收 `[]T` 并对其 append，**调用方** 可能看到、也可能看不到新元素，取决于运行时 cap。

### 3.3 三索引切片：cap 的护栏

```go
b := a[1:3:3]   // a[low:high:max]，cap = max - low
```

第三个参数显式限定 `cap`。`b` 的 cap 是 `3-1=2`，已等于 len，任何 append 立刻扩容、得到新底层数组。**这是阻断别名传染的标准手法**。

适用场景：
- 库函数返回 sub-slice 时，避免调用方 append 偷偷修改你内部数据
- 把数据"封装"传给不可信代码前

```go
// 安全：返回的子切片 cap = len，无法 append 不扩容
func PublicView(internal []int) []int {
    return internal[:5:5]
}
```

### 3.4 函数参数的隐式共享

```go
func reset(s []int) {
    for i := range s {
        s[i] = 0
    }
}
reset(a)   // a 被清零
```

切片传值时只是复制 24 字节的 header，`Data` 还是同一块内存。这是 Go 切片"既像引用又像值"的根本原因。

---

## 第四章：copy 与 slices 包

### 4.1 copy

```go
n := copy(dst, src)   // 返回实际复制的元素数 = min(len(dst), len(src))
```

`copy` 处理**重叠区间**是安全的（内部用 memmove）：

```go
s := []int{1, 2, 3, 4, 5}
copy(s[1:], s[:4])
fmt.Println(s)   // [1, 1, 2, 3, 4]
```

### 4.2 slices 包（Go 1.21+）

```go
import "slices"

slices.Clone(s)                 // 完整复制底层数组
slices.Concat(a, b, c)          // 多 slice 拼接
slices.Equal(a, b)
slices.Contains(s, x)
slices.Index(s, x)
slices.Delete(s, i, j)          // 删除区间，返回新 slice
slices.Insert(s, i, x...)       // 在 i 插入
slices.Sort(s)                  // in-place 排序（需要 Ordered 约束）
slices.SortFunc(s, less)        // 自定义比较
slices.Reverse(s)               // in-place 反转
slices.Compact(s)               // 去除连续重复
slices.Grow(s, n)               // 确保 cap 至少 len+n
slices.Clip(s)                  // 让 cap = len，释放多余内存
```

`slices.Clone` 是最常用的："给我一个完全独立的副本，断开别名"。

---

## 第五章：内存泄漏与切片

### 5.1 经典场景：子切片引用大数组

```go
func loadConfig() string {
    raw, _ := os.ReadFile("config.yaml")  // 假设 raw 是 10MB []byte
    return string(raw[100:200])   // 看似返回 100 字节
}
```

这里 `string(raw[100:200])` 在 Go 1.18 之前会**继续引用整个 10MB**（因为底层共享）。Go 1.18+ 对 `string(...)` 字面转换做了优化，会单独拷贝。但同样的模式如果是 slice，则**始终共享**：

```go
func first100(big []byte) []byte {
    return big[:100]   // 仍引用 big 的整个底层数组
}
```

如果 `big` 是 1GB 而 `first100` 的结果被长期持有，1GB 永远不被 GC。

**修正**：显式 clone

```go
func first100(big []byte) []byte {
    out := make([]byte, 100)
    copy(out, big)
    return out
}
// 或 Go 1.21+:
return slices.Clone(big[:100])
```

### 5.2 切片中存指针被遗忘

```go
type Task struct{ Data [1024]byte }

var tasks []*Task

func consume() *Task {
    t := tasks[0]
    tasks = tasks[1:]   // 头部元素的指针仍在底层数组里
    return t
}
```

`tasks[1:]` 之后，底层数组的第 0 个位置仍持有那个 `*Task`，GC 看到引用就不会回收。**修正**：显式置零

```go
tasks[0] = nil
tasks = tasks[1:]
```

`container/list` 等队列实现都有这条注意事项。

---

## 第六章：切片与 GC

### 6.1 GC 不扫描已经"超出 len"的部分？

**会扫描**。GC 按 `cap` 扫描底层数组（因为这部分仍可能被引用）。所以即使你截短了 `len`，存储在 `[len:cap]` 区间的指针仍会被认为活跃（如上例）。

### 6.2 Clip 释放尾部空间

```go
s = slices.Clip(s)   // 等价 s = s[:len(s):len(s)]
```

`Clip` 把 cap 降到 len。但**这本身不释放任何内存**——底层数组还是那个数组。要真正释放尾部，需要 clone 后丢弃旧 slice。

### 6.3 大批量删除元素

```go
// ❌ 仍引用所有旧元素
s = s[:0]

// ✅ 显式置零再清空
for i := range s {
    s[i] = nil   // 或 Zero 值
}
s = s[:0]
```

如果 T 不含指针（如 `int`、`[32]byte`），不需要置零——GC 不会"误判"基础数据为指针。

---

## 第七章：高频小技巧

### 7.1 删除中间元素

```go
// 删除索引 i
s = append(s[:i], s[i+1:]...)
// 或 Go 1.21+
s = slices.Delete(s, i, i+1)
```

注意：这种"前移"删除是 O(n)，对大 slice 频繁删除会很慢。如果不在乎顺序，用"末位交换"O(1)：

```go
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

### 7.2 reverse

```go
for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
    s[i], s[j] = s[j], s[i]
}
// 或 Go 1.21+
slices.Reverse(s)
```

### 7.3 去重（保持顺序）

```go
func dedup[T comparable](s []T) []T {
    seen := make(map[T]struct{}, len(s))
    out := s[:0]   // 复用底层数组
    for _, v := range s {
        if _, ok := seen[v]; !ok {
            seen[v] = struct{}{}
            out = append(out, v)
        }
    }
    return out
}
```

`out := s[:0]` 复用原数组——零分配。但要注意：调用方再使用原 `s` 时会看到被覆盖。

### 7.4 用 slice 当栈

```go
// push
stack = append(stack, x)
// pop
x, stack = stack[len(stack)-1], stack[:len(stack)-1]
```

---

## 第八章：生产级最佳实践

1. **能预估容量就 `make([]T, 0, N)`**——免费的性能提升。
2. **函数返回 sub-slice 要警惕别名**：必要时用三索引切片或 `slices.Clone`。
3. **大数组的子切片要 clone**：避免无意中持有 GB 级别数据。
4. **删除指针元素要置 nil**：让 GC 回收被指向的对象。
5. **`append` 返回值必须接收**：`append(s, x)` 不接收等于浪费。
6. **批量插入用 `append(s, batch...)`**，比循环 append 快。
7. **不要在多个 goroutine 间共享并写 slice**：用 mutex 或 channel。
8. **接口签名优先用 `[]T`**：调用方传 array、slice 都可（`a[:]`）。
9. **避免 `[]interface{}`**：大多场景泛型更好（Go 1.18+）。
10. **`s = s[:0]` 复用 vs `s = nil` 释放**：根据是否需要保留底层数组。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：忘记接收 append 返回值
```go
append(s, x)   // 警告但不报错；s 没变
```

### ❌ 陷阱 2：迭代中 append 自身
```go
for _, v := range s {
    s = append(s, v)   // 经典死循环？
}
```
实际不会无限循环——`range` 在循环开始时就**快照了 len**。但行为反直觉，避免。

### ❌ 陷阱 3：循环里取地址放 slice
```go
type T struct{ V int }
var ptrs []*T
for _, t := range items {
    ptrs = append(ptrs, &t)   // 所有指针都指向同一个 t！(Go 1.21 及之前)
}
```
Go 1.22+ 修复了循环变量语义，但 1.21 及之前是经典坑。修正：`t := t` 或 `ptrs = append(ptrs, &items[i])`。

### ❌ 陷阱 4：用 `s == nil` 判断空
```go
s := []int{}
if s == nil { ... }   // false，但 len(s) == 0
```
判断空用 `len(s) == 0`。

### ❌ 陷阱 5：sub-slice 修改影响 JSON 反序列化
```go
buf := []byte(jsonStr)
var v Foo
json.Unmarshal(buf, &v)   // v.StringField 可能引用 buf 的子切片
buf[0] = 'X'              // v.StringField 内容改变！
```
解决：用 `json.RawMessage` 时尤其注意；或 `string(buf[i:j])` 强制拷贝。

### ❌ 陷阱 6：Clip 不会释放底层数组
```go
huge := make([]byte, 1<<30)
small := slices.Clip(huge[:10])   // small 仍指向 1GB 数组
```
要真正释放：`small := append([]byte(nil), huge[:10]...)` 或 `slices.Clone(huge[:10])`。

### ❌ 陷阱 7：跨 goroutine 不加锁
```go
go func() { s = append(s, 1) }()
go func() { s = append(s, 2) }()
```
data race，结果不可预测。用 mutex 或 channel。

---

## 第十章：练习题

**练习 1**：以下输出？
```go
s := make([]int, 2, 4)
s[0], s[1] = 1, 2
t := append(s, 3)
t[0] = 99
fmt.Println(s, t)
```

**练习 2**：以下输出？
```go
s := []int{1, 2, 3}
t := append(s, 4)
u := append(s, 5)
fmt.Println(t)
fmt.Println(u)
```

**练习 3**：写一个 `Pop(s []int) (int, []int)` 函数，返回末元素和剩余 slice。考虑内存释放（如果 T 含指针）。

**练习 4**：以下为何泄漏？写出修正版。
```go
func windows(big []byte) [][]byte {
    var out [][]byte
    for i := 0; i < len(big); i += 100 {
        end := i + 100
        if end > len(big) { end = len(big) }
        out = append(out, big[i:end])
    }
    return out
}
```

**练习 5**：分析 `slices.Clone` 与 `append([]T(nil), s...)` 的差异。

---

## 参考答案

**练习 1**：`s=[99 2] t=[99 2 3]`。append 没扩容，t 与 s 共享底层；t[0] = 99 改了同一块内存。

**练习 2**：`t=[1 2 3 4] u=[1 2 3 5]`。第一次 append 触发扩容，t 是新数组；第二次又触发扩容，u 也是新数组。两者底层不同。**但**如果 s 初始 cap > 3（如 `make([]int, 3, 10)`），t 写 4 到 s 后面，u 又写 5 到同一位置覆盖 4，结果是 `t=[1 2 3 5] u=[1 2 3 5]`——这是经典面试题。

**练习 3**：
```go
func Pop[T any](s []T) (T, []T) {
    var zero T
    last := s[len(s)-1]
    s[len(s)-1] = zero   // 让 GC 回收（若 T 是指针/含指针）
    return last, s[:len(s)-1]
}
```

**练习 4**：每个 100 字节子切片仍引用整个 `big`，无法释放。修正：
```go
out = append(out, slices.Clone(big[i:end]))
```

**练习 5**：行为等价（都复制 cap=len 的新底层数组）。`slices.Clone` 更可读。底层实现也几乎相同；早期版本 `append(nil, s...)` 还多走了一次扩容分支，现已优化。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| SliceHeader | (Data, Len, Cap) = 24 字节 |
| nil vs empty | len 都是 0；JSON 序列化与 IsNil 有别 |
| append 扩容 | 小切片倍增，大切片 ~1.25 倍（Go 1.18+ 平滑） |
| 别名 | 共享底层是 slice 既灵活又危险的根源 |
| 三索引切片 | `s[a:b:c]` 限定 cap，阻断别名传染 |
| copy & slices | `slices.Clone` 是断开别名的标准动作 |
| GC 与 cap | GC 扫到 cap；删除指针元素要置 nil |

下一篇 **G04 — 精通 Go Map：哈希内幕与并发** 会拆开 `hmap`、讲清 bucket/overflow 链、扩容触发、迭代顺序为何随机、为什么并发读写直接 fatal error。

---

