# 精通 Go Map：哈希内幕、迭代顺序与并发安全

> 课程编号：G04
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Maps 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：一段会崩溃的程序

```go
package main

func main() {
    m := map[int]int{}
    for i := 0; i < 4; i++ {
        go func(i int) {
            for j := 0; j < 1000; j++ {
                m[i] = j
            }
        }(i)
    }
    select {}
}
```

运行不几秒，进程崩溃，错误信息是不可 recover 的 `fatal error: concurrent map writes`。为什么这种错误不像普通 panic 而是直接杀死整个进程？Go 的 map 是怎么探测并发写入的？删 1 万 key 之后内存为啥不下降？这一章把 `hmap` 的每一格都拆开看。

---

## 第一章：hmap 与 bucket 的物理结构

### 1.1 hmap

`runtime/map.go` 中的核心结构（简化版）：

```go
type hmap struct {
    count     int             // map 的元素数（== len(m)）
    flags     uint8           // 状态标志（含"正在写"位）
    B         uint8           // bucket 数量 = 2^B
    noverflow uint16          // 溢出桶大约数
    hash0     uint32          // 哈希种子（防 DoS）
    buckets   unsafe.Pointer  // 指向 2^B 个 bmap 的数组
    oldbuckets unsafe.Pointer // 扩容时指向旧数组
    nevacuate uintptr         // 已搬迁进度
    extra     *mapextra       // 溢出桶相关
}
```

### 1.2 bmap（bucket）

每个 bucket 固定能容纳 **8 个 key-value 对**：

```go
type bmap struct {
    tophash [8]uint8      // 每个 slot 的哈希高 8 位
    // 后面紧跟 [8]K，[8]V，overflow *bmap（编译器按 K,V 类型生成）
}
```

字段排列设计巧妙：
- `tophash` 集中存储，加速线性查找（缓存友好）
- key 和 value 分别集中（而非交错），避免内存对齐 padding
- overflow 指针在末尾，链式连接下一个桶

### 1.3 一个 map 的内存形态

```
hmap                  buckets array (2^B 个 bmap)
+------------+        +--------+ +--------+ +--------+
| count: N   |  -->   | bmap 0 | | bmap 1 | | bmap 2 | ...
| B: 3       |        +--------+ +--------+ +--------+
| hash0: ... |             |          |          |
| buckets ---+             v          v          v
+------------+         overflow chains（如果有的话）
```

---

## 第二章：哈希、tophash 与查找路径

### 2.1 哈希函数

Go runtime 内置类型相关的哈希函数（`runtime/alg.go`），不可定制。每个进程启动时 `hash0` 随机化，使**同样的 key 在不同进程的哈希值不同**——防御 hash collision DoS 攻击。

### 2.2 定位 bucket

```
hash := hashfunc(key, hash0)
bucketIndex := hash & (2^B - 1)      // 低 B 位选 bucket
tophashHint := uint8(hash >> 56)     // 高 8 位作为快速比较
```

进入对应 bucket 后，**先扫一遍 8 个 tophash 字节**，命中则再比较完整 key（处理哈希碰撞）。这种"先 8 位筛选、再全量比较"的两段查找让小 key（如 int、short string）查找极快。

### 2.3 overflow 链

当一个 bucket 装满 8 个 slot 仍要插入新 key 时，分配一个 overflow bmap 挂在链尾。哈希分布良好的情况下 overflow 链很短。链太长会触发 same-size 扩容（见下节）。

---

## 第三章：扩容机制

### 3.1 两种扩容

**Double 扩容**（grow）：
- 触发条件：`load factor > 6.5`（count / 2^B > 6.5）
- 行为：bucket 数 ×2，B+=1

**Same-size 扩容**（sameSizeGrow）：
- 触发条件：overflow 桶过多（noverflow ≥ 2^B 或 ≥ 32768）
- 行为：bucket 数不变，但重建一遍以整理碎片

### 3.2 增量搬迁

Go map 扩容**不是一次性完成**——会保留 `oldbuckets`，每次 `insert`/`delete` 时**最多搬两个旧 bucket** 到新数组（`evacuate`）。这把扩容代价均摊，避免单次 `m[k] = v` 卡住很久。

期间查找要同时检查新旧两个数组——这是为什么扩容期间性能轻微下降。

### 3.3 预分配避免反复扩容

```go
m := make(map[string]int, 1000)   // hint=1000
```

`make` 的第二个参数是**预期 key 数量**。runtime 据此选择初始 B（保证 load factor 在合理范围）。对已知规模的 map（如读 CSV、解析配置），预分配能消除多次扩容。

---

## 第四章：迭代顺序为何随机

### 4.1 语言规范

> The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next.

Go 故意让 `range m` **每次顺序不同**。每次开始迭代时，runtime 在 `mapiterinit` 中随机选一个起始 bucket + slot。

```go
m := map[int]int{1: 1, 2: 2, 3: 3, 4: 4}
for k := range m { fmt.Print(k, " ") }
fmt.Println()
for k := range m { fmt.Print(k, " ") }
// 输出（典型）：
// 3 1 4 2
// 2 4 1 3
```

### 4.2 为什么这么设计

历史上一些 Map 实现迭代顺序"恰好稳定"，开发者就**依赖**它（即使文档没承诺），后来实现改进时大量代码 break。Go 团队主动随机化，**强迫**所有开发者不要依赖顺序——"如果你的代码依赖顺序，就让它立刻 bug 给你看"。

### 4.3 真要有序怎么办

```go
keys := make([]int, 0, len(m))
for k := range m { keys = append(keys, k) }
sort.Ints(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
```

或用 Go 1.21+ 的 `slices.Sorted(maps.Keys(m))`（需 `golang.org/x/exp/maps` 或等待标准库迭代器稳定）。

---

## 第五章：并发与 sync.Map

### 5.1 fatal error：concurrent map writes

```go
m := map[int]int{}
go func() { m[1] = 1 }()
go func() { m[1] = 2 }()
// fatal error: concurrent map writes
```

runtime 在每个写操作前后会切换 `hmap.flags` 的 `hashWriting` 位。如果发现进入写状态时该位已置，**直接 fatal error**，不能 recover。

为什么这么严？因为并发写 map 可能损坏 bucket 链结构，导致后续读到非法内存（segfault 甚至安全漏洞）。fatal 比让损坏的 map 继续运行更安全。

### 5.2 读写也算并发

只有读没事；但**只要有一个 goroutine 在写**，其他读取也会触发：

```go
go func() { m[1] = 1 }()
go func() { _ = m[1] }()  // 也可能 fatal
```

### 5.3 三种并发方案

**方案 A：map + RWMutex**

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}
func (s *SafeMap) Get(k string) (int, bool) {
    s.mu.RLock(); defer s.mu.RUnlock()
    v, ok := s.m[k]; return v, ok
}
func (s *SafeMap) Set(k string, v int) {
    s.mu.Lock(); defer s.mu.Unlock()
    s.m[k] = v
}
```

适用：写比例较高、key 集合动态。

**方案 B：sync.Map**

```go
var m sync.Map
m.Store(k, v)
val, ok := m.Load(k)
m.LoadOrStore(k, v)
m.Range(func(k, v any) bool { return true })
```

适用：读多写少（约 95%+ 读）、或写入是"key 集合稳定 + 频繁覆盖更新"。**写多读少时 sync.Map 比 RWMutex 慢**。

sync.Map 内部维护 `read` 和 `dirty` 两个 map：读优先走 read（atomic 无锁），未命中再去 dirty（加锁）。这种"读路径无锁"是它在读多场景快的原因。

**方案 C：分片 map**（shard）

```go
type Shard struct {
    mu sync.RWMutex
    m  map[string]int
}
type ShardMap [256]*Shard

func (s *ShardMap) shard(k string) *Shard {
    h := fnv.New32a(); h.Write([]byte(k))
    return s[h.Sum32() % 256]
}
```

适用：超高并发（数百万 ops/s）。每个分片独立锁，互不阻塞。

### 5.4 基准对照（量级）

| 场景 | 1 写 / 99 读 | 50 写 / 50 读 |
|---|---|---|
| map + RWMutex | 中等 | 中等 |
| sync.Map | **快 2-3 倍** | 慢 |
| 256-shard map + RWMutex | 最快 | 最快 |

---

## 第五章半：Swiss Tables —— Go 1.24 把 map 实现换了一遍

### 5.5 为什么换

上面整章讲的 `hmap` + bmap + overflow 链路是 Go 自 2014 起的经典布局。**Go 1.24 (2025-02) 把内置 map 默认实现整体替换为 Swiss Tables**——一种 Google Abseil 起源的现代哈希表设计：

- **基础结构**：以 group（一组 8 槽位 + 8 字节 control word）为单位组织。control word 每字节存对应槽的 7-bit hash 摘要 + 状态标记（empty / deleted / used）。
- **查找路径**：先在 control word 上跑一次 SIMD/SWAR 比较（一次比 8 个字节），匹配的槽再去取真实 key 比对。**少了"老 bmap 8 个 tophash 一个个比"的循环**，现代 CPU 上更友好。
- **碰撞策略**：开放寻址 + 二次探测（不再有 overflow 桶链）。

### 5.6 性能影响（官方数据）

> Go 1.24 release notes: *"runtime CPU overheads decreased by 2-3% on average across a suite of representative benchmarks"*。社区独立测试在大 map 上更明显：访问与赋值约快 30%、预分配赋值快 35%、迭代视规模快 10–60%。

### 5.7 对你写代码的影响

- **API 完全兼容**——`m[k]`、`m[k] = v`、`delete`、`for k,v := range m` 全不变。
- **迭代顺序仍然随机**（甚至更随机；Go 一直主动打乱以阻止你依赖顺序）。
- **`unsafe` 直接读 `hmap` 内部字段的代码会爆**。如果你或依赖里有这种 hack，升级前先 grep 一遍。
- **`delete` 仍然不缩容**——和老实现一样。Swiss Tables 的"deleted" 标记会让连续删除的桶保留一段时间，再次插入可能复用槽位。
- **想退回老实现**（用于性能对比或排错）：构建时 `GOEXPERIMENT=noswissmap`。该退路在 Go 1.27 之前都保留。

### 5.8 同步：sync.Map 也升级了

Go 1.24 一同把 `sync.Map` 换为基于 **concurrent hash-trie** 的实现（KIP-style infinite-array trie），在大多数 benchmark 上比旧版快。读多写少的旧场景仍然受益，写多场景的回退也大幅减少。

> 参考：[Go 1.24 release notes — Runtime](https://go.dev/doc/go1.24#runtime)、Abseil [Swiss Tables design notes](https://abseil.io/about/design/swisstables)。

---


### 6.1 不能取 map value 的地址

```go
m := map[string]Point{"a": {1, 2}}
p := &m["a"]   // ❌ 编译错误：cannot take the address of m["a"]
```

原因：map 扩容/搬迁可能让 value 位置移动；取地址后让其被指向意义不明。

### 6.2 不能直接改 struct 字段

```go
m["a"].X = 99   // ❌ 编译错误：cannot assign to struct field
```

必须取出整个 struct 改完写回：

```go
p := m["a"]
p.X = 99
m["a"] = p
```

或改用 `map[string]*Point`——value 是指针，指针指向的对象就可以改：

```go
m := map[string]*Point{"a": {1, 2}}
m["a"].X = 99   // OK
```

### 6.3 nil map 不能写但能读

```go
var m map[string]int
_ = m["a"]      // OK，返回 0
m["a"] = 1      // ❌ panic: assignment to entry in nil map
```

修正：先 `m = make(...)` 或 `m = map[string]int{}`。

### 6.4 delete 不缩容

```go
m := make(map[int]int, 1<<20)
for i := 0; i < 1<<20; i++ { m[i] = i }
for k := range m { delete(m, k) }
// m 内部 bucket 数组仍是 ~1MB 容量
```

Go runtime 没有 "shrink" 操作。若需要释放，**重建 map**：

```go
m = make(map[int]int)   // 让 GC 回收旧的
```

`runtime.ReadMemStats` 可以验证内存确实回落。

---

## 第七章：map 与逃逸分析

### 7.1 map 本身在堆上

```go
m := map[int]int{}
```

`hmap` 结构永远在堆上分配（指针传递、有 hash 表的语义）。但**装入 map 的 key/value** 不一定都逃逸——值类型 key/value 是按值拷贝进 bucket。

### 7.2 哪些操作会触发分配

- `make(map, hint)`：分配 hmap + buckets 数组
- `m[k] = v`，若触发扩容：分配新 buckets
- `m[k] = v`，若需 overflow：分配新 bmap

`go build -gcflags="-m"` 可以看到 escape 决策。

---

## 第八章：生产级最佳实践

1. **已知规模就 `make(map, N)`**，免费消除多次扩容。
2. **map value 是 struct 且常更新字段时用 `*struct`**，避免取出—改—写回三步舞。
3. **读多写少考虑 sync.Map**；写多写读用 RWMutex；超高并发用分片。
4. **永远不要"先读后写"不加锁**：哪怕只是"如果不存在就插入"也要 `LoadOrStore` 或锁。
5. **删除大量 key 后想释放内存，必须重建 map**。
6. **不要依赖迭代顺序**，必要时收集到 slice 再排序。
7. **map key 必须 comparable**：`func`、`slice`、`map` 类型不能当 key（编译错误）；包含这些字段的 struct 也不行。
8. **string key 短而稳定时哈希极快**；超长 key 考虑预哈希存 uint64。
9. **不要并发 range + delete 同一个 map**（即使加 mutex 也会让迭代失序）。
10. **JSON map[string]any 反序列化数字会变 float64**：用 `json.Number` 或自定义类型。

---

## 第九章：常见陷阱清单

### ❌ 陷阱 1：取 map value 地址
```go
&m[k]   // 编译错误
```

### ❌ 陷阱 2：修改 map 中的 struct 字段
```go
m["x"].Y = 1  // 编译错误
```

### ❌ 陷阱 3：nil map 写入
```go
var m map[string]int
m["x"] = 1   // panic
```

### ❌ 陷阱 4：并发 read + write
```go
go m[1] = 1
go _ = m[1]
// fatal error
```

### ❌ 陷阱 5：以为 delete 释放内存
```go
for k := range m { delete(m, k) }
// 内存不变；要 m = make(...)
```

### ❌ 陷阱 6：用 float 当 key
```go
m := map[float64]int{}
m[math.NaN()] = 1
m[math.NaN()] = 2
fmt.Println(len(m))   // 2 ！NaN != NaN，每次都是"新 key"
```

### ❌ 陷阱 7：复制 map
```go
m2 := m   // 仅复制 header；m2 和 m 是同一个 map
```
要深拷贝得用循环或 `maps.Clone`（Go 1.21+）。

---

## 第十章：练习题

**练习 1**：以下输出？
```go
m := map[int]int{1:1, 2:2, 3:3}
for k, v := range m {
    if k == 2 { delete(m, 3) }
    fmt.Println(k, v)
}
```

**练习 2**：以下为什么 panic？怎么修？
```go
type T struct{ M map[int]int }
var t T
t.M[1] = 1
```

**练习 3**：写一个并发安全的"key → list of values"map，支持高频读、偶尔写。

**练习 4**：以下两种实现，哪个内存占用低？为什么？
```go
// A
type cacheA struct { m map[string]Big }
// B
type cacheB struct { m map[string]*Big }
```

**练习 5**：用 `runtime.ReadMemStats` 写一个小程序：插入 100 万 key 后全部 delete，对比 `Sys` 和 `HeapAlloc` 变化。

---

## 参考答案

**练习 1**：可能输出 `1 1 / 2 2`（k=3 已删，不再出现）或 `3 3 / 2 2 / 1 1`（顺序随机，删除生效）。规范允许迭代时删除——已遍历过的 key 不重现，未遍历的 key 可能跳过。**绝不要**在迭代时插入。

**练习 2**：`t.M` 是 nil map，赋值 panic。修正：在结构体初始化时 `t := T{M: map[int]int{}}` 或 `t.M = make(map[int]int)` 后再写。

**练习 3**：
```go
type MultiMap struct {
    mu sync.RWMutex
    m  map[string][]int
}
func (s *MultiMap) Get(k string) []int {
    s.mu.RLock(); defer s.mu.RUnlock()
    return append([]int(nil), s.m[k]...)  // 返回副本避免外部修改
}
func (s *MultiMap) Append(k string, v int) {
    s.mu.Lock(); defer s.mu.Unlock()
    s.m[k] = append(s.m[k], v)
}
```

**练习 4**：A 一次性占用较高（每个 value 完整拷贝进 bucket），B 用指针（每个 8 字节）但每个 Big 单独在堆上分配，并增加 GC 压力。**Big 较大时 B 更省内存**，但 GC 扫描更慢。Big 较小（<64 字节）时 A 更好（局部性好、零额外分配）。

**练习 5**：
```go
var m runtime.MemStats
runtime.ReadMemStats(&m)
// 插入 + 测量
// delete + 测量
// 重建 + 测量
```
你会看到：delete 后 HeapAlloc 几乎不变；`m = make(...)` 后大幅下降（旧 hmap 被 GC）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| hmap 结构 | bucket = 8 slot；overflow 链；hash0 随机化 |
| 查找路径 | hash → bucketIndex → tophash 匹配 → key 比较 |
| 扩容 | doubling（负载过高）或 same-size（碎片）；增量搬迁 |
| 迭代 | 顺序随机；rang 中删除合法、插入未定义 |
| 并发 | 写写、读写都 fatal；用 RWMutex / sync.Map / shard |
| value 不可寻址 | 不能 &m[k]；不能 m[k].field = ... |
| delete 不缩容 | 释放内存必须 `m = make(...)` |
| 预分配 | `make(map, hint)` 是免费性能 |

下一篇 **G05 — 精通 Go Struct：内存布局与嵌入** 会拆开结构体字段对齐、padding，讲清"为什么字段顺序影响 sizeof 50%"、unsafe.Offsetof 的用法、嵌入 vs 继承的本质区别。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G04-精通-Go-Map-哈希内幕与并发.md`
