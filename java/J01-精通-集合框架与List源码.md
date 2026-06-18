# 精通集合框架与 List 源码

> 集合框架（Java Collections Framework）是 Java 面试的"开胃菜"也是"必答题"。从 `ArrayList` 怎么扩容，到 `fail-fast` 怎么触发，几乎每场后端面试都会问。这一篇把 Collection 体系和 List 的源码讲透，后面的 [J02 HashMap](./J02-精通-HashMap与ConcurrentHashMap.md) 专攻 Map。
>
> **📅 基准：Java 21 / 17 LTS。** 源码以 OpenJDK 为准。

---

## 一、集合框架全景

Java 集合分两大体系：**Collection**（单列）和 **Map**（双列），都在 `java.util` 包下。

```
Collection（接口）
├── List    有序、可重复       —— ArrayList / LinkedList / Vector
├── Set     无序、不可重复     —— HashSet / LinkedHashSet / TreeSet
└── Queue   队列              —— ArrayDeque / LinkedList / PriorityQueue

Map（接口）  键值对、键不可重复  —— HashMap / LinkedHashMap / TreeMap / ConcurrentHashMap
```

- **List**：关注"顺序 + 索引访问"。
- **Set**：关注"去重"，底层常借助 Map（`HashSet` 内部就是一个 `HashMap`）。
- **Queue/Deque**：关注"两端进出"。
- **Map**：关注"键值映射"，是面试最深的部分（见 J02）。

本篇聚焦 List；Set/Map 的去重与哈希原理在 [J02](./J02-精通-HashMap与ConcurrentHashMap.md) 讲。

---

## 二、ArrayList 源码

`ArrayList` 底层是**一个 `Object[]` 数组**，支持随机访问。

### 2.1 关键字段与初始化

```java
transient Object[] elementData; // 实际存储（transient：不走默认序列化）
private int size;               // 元素个数（不是数组长度）
private static final int DEFAULT_CAPACITY = 10;
```

`new ArrayList()` **不会立即分配 10 的数组**，而是先指向一个空数组 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，**第一次 add 时才扩容到 10**（懒初始化，省内存）。

### 2.2 add 与扩容（高频考点）

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1); // 确保容量够
    elementData[size++] = e;
    return true;
}
```

扩容核心 `grow()`：**新容量 = 旧容量 + (旧容量 >> 1)，即 1.5 倍**：

```java
int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5x
elementData = Arrays.copyOf(elementData, newCapacity); // 数组拷贝
```

要点：
- 扩容是 **1.5 倍**（不是 2 倍，HashMap 才是 2 倍）。
- 扩容要 `Arrays.copyOf` 拷贝整个数组，代价 O(n)——**已知大小时用 `new ArrayList<>(capacity)` 预设容量**，避免反复扩容拷贝。

### 2.3 随机访问与增删

- `get(i)`/`set(i)`：直接索引数组，**O(1)**。
- `add(i, e)`/`remove(i)`：要 `System.arraycopy` 移动后续元素，**O(n)**。
- 尾部 `add(e)`：摊还 O(1)（偶尔触发扩容）。

---

## 三、LinkedList 源码

`LinkedList` 是**双向链表**，同时实现 `List` 和 `Deque`。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
transient Node<E> first; // 头
transient Node<E> last;  // 尾
```

- **头尾增删 O(1)**（改指针即可），适合做队列/栈（`Deque`）。
- **按索引访问 O(n)**：`get(i)` 要从头或尾遍历到第 i 个（源码会判断 i 靠近哪头，从近的一端遍历）。
- 没有扩容概念（链表节点按需 new），但每个节点多两个指针引用，**内存开销比 ArrayList 大**。

---

## 四、ArrayList vs LinkedList

| 维度 | ArrayList | LinkedList |
|---|---|---|
| 底层 | 动态数组 | 双向链表 |
| 随机访问 get(i) | O(1) | O(n) |
| 头部插入/删除 | O(n)（要搬移） | O(1) |
| 尾部插入 | 摊还 O(1) | O(1) |
| 中间插入 | O(n) | O(n)（找位置也要 O(n)） |
| 内存 | 紧凑（可能有空余容量） | 每节点额外两个指针 |
| 缓存友好 | 好（连续内存） | 差（指针跳转） |

**实战结论**：绝大多数场景用 **ArrayList**。即使是"频繁中间插入"，LinkedList 也要先 O(n) 定位插入点，优势不明显；而它的指针开销和缓存不友好往往让它整体更慢。LinkedList 真正的价值是当 `Deque`（双端队列）用。**面试别再背"频繁增删用 LinkedList"这种过时结论**——要看是否需要随机访问、是否是两端操作。

---

## 五、fail-fast 与 modCount

**fail-fast（快速失败）**：在用迭代器遍历集合时，如果集合**结构被修改**（增删），会立刻抛 `ConcurrentModificationException`，而不是让你拿到错乱数据。

原理是 `modCount`（修改次数）：

```java
// 迭代器创建时记录 expectedModCount = modCount
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

每次 add/remove 都会 `modCount++`，迭代器每次 next 都校验 `modCount == expectedModCount`，不等就抛异常。

- **触发场景**：`for-each` 遍历时直接调 `list.remove()`（for-each 本质是迭代器，但你绕过迭代器改了集合）。
- **正确删除**：用迭代器自己的 `iterator.remove()`（它会同步更新 expectedModCount）。
- **fail-safe**：`CopyOnWriteArrayList` 等并发容器遍历的是快照副本，不抛异常（见 [J12 并发容器](./J12-精通-并发容器.md)）。

注意：fail-fast 是**尽力而为的检测机制**（不保证一定触发），不能当作并发正确性的保证。

---

## 六、Iterator 与 ListIterator

迭代器模式让遍历与集合实现解耦。

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (cond) it.remove(); // ✅ 正确：用迭代器删除
}
```

- `Iterator`：单向 `hasNext/next/remove`。
- `ListIterator`：List 专属，支持**双向**遍历（`hasPrevious/previous`）和遍历中 `add/set`。
- **遍历中删除只能用 `it.remove()`**，否则 fail-fast。

---

## 七、Vector / Stack 与线程安全

- `Vector`：古老的线程安全 List，所有方法 `synchronized`，性能差，**已不推荐**。
- `Stack`：继承 `Vector`，更不推荐——要用栈用 `ArrayDeque`。
- 需要线程安全 List：用 `CopyOnWriteArrayList`（读多写少）或 `Collections.synchronizedList()`，见 [J12](./J12-精通-并发容器.md)。

---

## 八、Arrays.asList 与常见坑

```java
List<Integer> list = Arrays.asList(1, 2, 3);
list.add(4);    // ❌ UnsupportedOperationException：固定大小
```

- `Arrays.asList` 返回的是 `Arrays` 内部类，**固定大小**，不能 add/remove（底层就是传入的数组）。
- 它和原数组**共享引用**，改 list 会改原数组。
- 要可变集合：`new ArrayList<>(Arrays.asList(...))`。
- **基本类型坑**：`Arrays.asList(new int[]{1,2,3})` 会得到 `List<int[]>` 只有一个元素（int[] 被当成一个对象）。用 `Integer[]` 或 Java 8 `Arrays.stream(...).boxed()`。

---

## 陷阱清单

- **以为 `new ArrayList()` 就分配了容量 10**：实际是懒初始化，首次 add 才分配。
- **ArrayList 扩容当成 2 倍**：是 1.5 倍；HashMap 才是 2 倍。
- **已知大小不预设容量**：反复扩容 + 数组拷贝，性能损失。
- **for-each 里直接 `list.remove()`**：fail-fast 抛 ConcurrentModificationException。用迭代器 remove。
- **无脑"增删多用 LinkedList"**：LinkedList 定位也要 O(n)、缓存不友好，实战多数仍用 ArrayList。
- **`Arrays.asList` 后 add/remove**：固定大小，抛 UnsupportedOperationException。
- **用 Vector/Stack**：过时，分别用 ArrayList / ArrayDeque 替代。
- **subList 当独立 list 用**：`subList` 是视图，改它会影响原 list，且原 list 结构变化会让 subList 失效。

---

## 2026 现状

- **集合框架本身稳定**：核心实现多年未大改，仍是面试必考的源码题。
- **不可变集合**：Java 9+ 的 `List.of()`/`Map.of()` 创建不可变集合，比 `Arrays.asList` 更推荐用于常量集合（真不可变、null 不友好需注意）。
- **Stream 与集合**：Java 8 Stream（见 [J28](./J28-精通-Java版本特性演进.md)）改变了集合的遍历/转换写法，但底层数据结构不变。
- **Sequenced Collections**：Java 21 引入 `SequencedCollection`/`SequencedMap` 接口，统一了"有顺序集合"的首尾访问 API（`getFirst/getLast/reversed`）。

---

## 练习题

1. `ArrayList` 的扩容机制是怎样的？默认容量多少？为什么说"已知大小要预设容量"？

<details><summary>参考答案</summary>

`ArrayList` 底层是 Object 数组。`new ArrayList()` 时是懒初始化（指向空数组），**第一次 add 才分配默认容量 10**。当元素个数超过当前容量时触发扩容 `grow()`：新容量 = 旧容量 + (旧容量 >> 1)，即 **1.5 倍**，然后用 `Arrays.copyOf` 把旧数组元素拷贝到新数组。预设容量的原因：每次扩容都要分配新数组并拷贝全部元素（O(n)），如果从 10 开始反复扩容到很大，会发生多次拷贝、产生大量临时数组和 GC 压力；已知大致大小时用 `new ArrayList<>(expectedSize)` 一次性分配足够容量，避免反复扩容拷贝，性能更好。

</details>

2. ArrayList 和 LinkedList 如何选择？"频繁增删就用 LinkedList"这个说法对吗？

<details><summary>参考答案</summary>

ArrayList 底层数组：随机访问 get(i) O(1)、尾部添加摊还 O(1)、中间/头部增删 O(n)（要搬移元素）、内存紧凑且缓存友好。LinkedList 双向链表：头尾增删 O(1)、随机访问 O(n)（要遍历）、每个节点有额外指针开销、缓存不友好。"频繁增删就用 LinkedList"是**过时/不准确**的说法：因为 LinkedList 在中间增删前，定位到那个位置本身就要 O(n) 遍历，优势被抵消；加上指针开销和缓存不友好，实际整体性能常常不如 ArrayList。结论：绝大多数场景用 ArrayList；LinkedList 真正有价值的是当作 Deque（双端队列/栈）用、只在两端操作的场景。选择应看"是否需要随机访问、是否只在两端操作"，而非笼统的"增删多少"。

</details>

3. 什么是 fail-fast？它是如何实现的？在 for-each 循环里直接调用 `list.remove()` 会发生什么？正确做法是什么？

<details><summary>参考答案</summary>

fail-fast（快速失败）是指用迭代器遍历集合时，若检测到集合结构被修改（增删），立即抛出 `ConcurrentModificationException`，避免基于错乱数据继续运行。实现靠 `modCount`（结构修改计数器）：迭代器创建时记录 `expectedModCount = modCount`，每次 add/remove 会 `modCount++`，迭代器每次 `next()` 都校验 `modCount == expectedModCount`，不相等就抛异常。for-each 本质是用迭代器遍历，但你绕过迭代器直接调 `list.remove()` 改了 modCount，下次 next 校验失败就抛 ConcurrentModificationException。正确做法：用迭代器自身的 `iterator.remove()`（它会同步更新 expectedModCount），或用 `removeIf()`、或倒序索引遍历删除、或收集后批量删。注意 fail-fast 是尽力检测、不保证一定触发，不能当并发安全保证。

</details>

4. `Arrays.asList(1, 2, 3)` 返回的 List 调用 add 会怎样？为什么？

<details><summary>参考答案</summary>

会抛 `UnsupportedOperationException`。因为 `Arrays.asList` 返回的不是 `java.util.ArrayList`，而是 `Arrays` 的一个内部类（`Arrays$ArrayList`），它是对传入数组的固定大小视图：底层直接引用那个数组，所以**大小固定**，不支持 add/remove（这些方法没有重写、用了 AbstractList 默认实现，直接抛 UnsupportedOperationException），但支持 set（修改会同步反映到原数组，因为共享引用）。如果需要可变列表，应 `new ArrayList<>(Arrays.asList(...))` 拷贝一份。另外注意基本类型数组的坑：`Arrays.asList(new int[]{1,2,3})` 会得到 `List<int[]>` 只含一个元素，应使用 `Integer[]` 或 `Arrays.stream(arr).boxed().toList()`。

</details>

5. `HashSet` 是如何保证元素不重复的？它和 `HashMap` 是什么关系？

<details><summary>参考答案</summary>

`HashSet` 内部就是一个 `HashMap`：它把要存的元素作为 HashMap 的 **key**，value 用一个固定的虚拟对象（`PRESENT`，一个 `static final Object`）。`set.add(e)` 实际是 `map.put(e, PRESENT)`，利用 HashMap key 不可重复的特性来去重——是否重复由 key 的 `hashCode()` 和 `equals()` 决定（先比 hash 定位桶，再用 equals 比较）。所以放入 HashSet 的对象必须正确重写 `hashCode` 和 `equals`，否则去重失效（两个"相等"的对象因 hashCode 不同被当作不同元素）。`HashSet` 的无序性、O(1) 增删查、扩容等特性都直接来自底层 HashMap（见 J02）。同理 `TreeSet` 底层是 `TreeMap`。

</details>

6. `List.of()`（Java 9+）和 `Arrays.asList()` 有什么区别？

<details><summary>参考答案</summary>

①**可变性**：`Arrays.asList()` 返回固定大小但元素可 set（共享原数组）的视图；`List.of()` 返回**完全不可变**集合，add/remove/set 全部抛 UnsupportedOperationException。②**null**：`Arrays.asList()` 允许 null 元素；`List.of()` **不允许 null**（传 null 抛 NullPointerException）。③**与数组的关系**：`Arrays.asList()` 底层引用传入数组、二者联动；`List.of()` 是独立的不可变实现，不与外部数组共享。④**用途**：`List.of()` 适合定义真正的常量/不可变集合（更安全、语义清晰），是 Java 9+ 推荐方式；`Arrays.asList()` 多用于快速把数组转成 List 视图或作为可变列表的构造参数。选择上，需要不可变常量用 `List.of()`，需要可变列表用 `new ArrayList<>(...)`。

</details>
