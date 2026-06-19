# 精通 String 与不可变设计

> `String` 是 Java 用得最多的类，也是面试高频点：为什么不可变？常量池怎么回事？`new String("a")` 创建几个对象？字符串拼接为什么不能在循环里用 `+`？这一篇讲清楚。
>
> **📅 基准：Java 17/21。** 注意 Java 9 起 String 底层从 `char[]` 改为 `byte[]`（紧凑字符串）。

---

## 一、String 的不可变性

`String` 是**不可变（immutable）**的：

```java
public final class String          // ① final 类，不可被继承
    implements ... {
    private final byte[] value;    // ② final 数组引用，JDK9+ 是 byte[]（JDK8 是 char[]）
    private int hash;              // 缓存的 hashCode
}
```

不可变的三重保证：
1. `String` 类是 `final`，不能被继承去破坏不可变性。
2. 内部 `value` 数组是 `private final`，引用不可变。
3. String 没有任何修改 `value` 内容的方法；所有"修改"操作（`substring`、`replace`、`concat`）都**返回新的 String 对象**，原对象不变。

所以 `s.concat("x")` 不改变 s，要 `s = s.concat("x")` 才有效——这是新手常见误区。

---

## 二、不可变带来的好处

| 好处 | 说明 |
|---|---|
| **线程安全** | 不可变对象天然线程安全，可在多线程间自由共享，无需同步 |
| **可缓存 hashCode** | hashCode 算一次就缓存（`hash` 字段），适合做 HashMap 的 key（高频） |
| **支持常量池** | 因为不可变，相同字面量可安全共享同一个对象，省内存 |
| **安全性** | 类加载路径、网络地址、数据库连接串等用 String，不可变防止被恶意篡改 |

正因不可变 + 缓存 hashCode，**String 是 HashMap key 的理想类型**（呼应 [J02](./J02-精通-HashMap与ConcurrentHashMap.md) 里"key 应用不可变对象"）。

---

## 三、字符串常量池

**字符串常量池（String Pool）** 是 JVM 维护的一块特殊存储，用来复用字符串字面量。Java 7 起常量池从永久代移到了**堆**中。

```java
String a = "hello";   // 字面量 → 进常量池
String b = "hello";   // 复用常量池里同一个对象
System.out.println(a == b); // true（同一引用）
```

- 字面量 `"hello"` 在编译期进入 class 常量池，运行时驻留到字符串常量池，相同字面量共享同一对象。
- `intern()`：手动把字符串放入/查找常量池——`s.intern()` 返回常量池中等于 s 的引用（已存在则返回池中的，否则放入并返回）。

---

## 四、`new String("x")` 创建几个对象

经典面试题：

```java
String s = new String("hello");
```

- 若 `"hello"` 此前**未在常量池**：创建 **2 个**对象——一个常量池里的 `"hello"`（编译期字面量），一个堆上 `new` 出来的 String 对象。
- 若 `"hello"` **已在常量池**：创建 **1 个**——只 new 堆对象，复用池中字面量。

```java
String s1 = new String("hello");
String s2 = "hello";
System.out.println(s1 == s2);          // false（s1 是 new 的堆对象，s2 是池中对象）
System.out.println(s1.intern() == s2); // true（intern 返回池中引用）
System.out.println(s1.equals(s2));     // true（值相等）
```

记住：`==` 比引用，`equals` 比值。**比较字符串内容永远用 equals**。

---

## 五、String / StringBuilder / StringBuffer

| 类 | 可变性 | 线程安全 | 场景 |
|---|---|---|---|
| String | 不可变 | 安全（因不可变） | 少量、固定字符串 |
| StringBuilder | 可变 | **不安全** | 单线程拼接（绝大多数场景） |
| StringBuffer | 可变 | 安全（方法 synchronized） | 多线程共享拼接（很少见） |

- `StringBuilder`/`StringBuffer` 底层是**可变的 char[]/byte[]**，append 在原数组上追加（不够则扩容），不产生大量中间对象。
- `StringBuffer` 的方法都加了 `synchronized`，有锁开销；单线程用 `StringBuilder` 更快。
- 多线程拼接其实极少真的共享一个 buffer，所以 StringBuffer 实际很少用——通常各线程用各自的 StringBuilder。

---

## 六、字符串拼接的编译优化

```java
String s = "a" + "b" + "c";   // 编译期常量折叠 → 直接是 "abc"
```

对于**字面量/常量**拼接，编译器在编译期就折叠成一个字符串，无运行时开销。

但对**变量**拼接：

```java
String r = a + b + c;  // 编译为 new StringBuilder().append(a).append(b).append(c).toString()
```

Java 编译器把 `+` 转成 StringBuilder（Java 9+ 用 `invokedynamic` + `StringConcatFactory` 优化）。问题出在**循环里用 `+`**：

```java
// ❌ 每次循环都 new 一个 StringBuilder，O(n²) 级别
String s = "";
for (int i = 0; i < n; i++) s += i;

// ✅ 复用一个 StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(i);
```

循环里 `+=` 每轮都新建 StringBuilder 并 toString，产生大量临时对象、性能差。**循环拼接必须用 StringBuilder**。

---

## 七、JDK 9 紧凑字符串

Java 9 引入 **Compact Strings（紧凑字符串）**：String 底层从 `char[]`（每字符 2 字节 UTF-16）改为 `byte[]` + 一个 `coder` 标志。

- 如果字符串只含 Latin-1（ISO-8859-1，单字节能表示，多数英文/数字），用 **1 字节/字符**存储，省一半内存。
- 含非 Latin-1 字符（如中文）才用 UTF-16（2 字节/字符）。
- 对开发者透明，但显著降低了以英文为主的应用的内存占用——这是 JDK 9 重要优化。

---

## 八、常见考点

- **`==` vs `equals`**：`==` 比引用地址，`equals` 比字符值。比较内容用 equals。
- **switch 支持 String**（Java 7+）：底层用 hashCode + equals 实现。
- **`split` 的坑**：`split` 参数是正则，特殊字符（`.`、`|`）要转义；末尾空字符串默认被去掉（`"a,,".split(",")` 长度为 1，不是 3）。
- **常量池与 GC**：常量池中的字符串可被回收（Java 7 后在堆），但长期持有大量 intern 字符串要小心内存。

---

## 陷阱清单

- **以为 `s.concat("x")` 改变了 s**：String 不可变，所有"修改"返回新对象，要赋值回去。
- **循环里用 `+` 拼接**：每轮 new StringBuilder，O(n²)。用一个 StringBuilder.append。
- **比较字符串用 `==`**：比的是引用，应该用 equals。
- **`new String("x") == "x"`**：false，new 出来是新堆对象。
- **split 不转义特殊字符**：`"a.b".split(".")` 得到空数组（`.` 是正则任意字符）。
- **大量 intern() 字符串**：可能占用内存，且 intern 有查找开销，谨慎使用。
- **多线程无脑用 StringBuffer**：单线程下白白承担锁开销，用 StringBuilder。

---

## 2026 现状

- **紧凑字符串已成默认**（Java 9+），英文为主的应用内存占用显著下降。
- **文本块（Text Blocks）**：Java 15+ 的 `"""..."""` 多行字符串，写 JSON/SQL/HTML 更方便（见 [J29](./J29-精通-Java版本特性演进.md)）。
- **字符串模板（String Templates）**：作为预览特性演进中（用于安全插值），关注后续 LTS 的稳定化。
- **`StringConcatFactory`**：Java 9 起用 `invokedynamic` 做字符串拼接，运行时可选最优策略，比老的固定 StringBuilder 方案更灵活高效。

---

## 练习题

1. String 为什么设计成不可变？不可变带来哪些好处？

<details><summary>参考答案</summary>

不可变通过三点保证：String 类是 final（不能被继承破坏）、内部存储数组 `value` 是 private final（引用不可变）、且没有任何修改内容的方法（substring/replace 等都返回新对象）。好处：①**线程安全**——不可变对象天然线程安全，可在多线程间自由共享无需同步；②**可缓存 hashCode**——hashCode 算一次缓存起来，使 String 成为 HashMap key 的理想类型；③**支持字符串常量池**——因为内容不会变，相同字面量可安全复用同一对象，节省内存；④**安全性**——用作类路径、网络地址、参数等时，不可变防止被恶意或意外篡改。代价是每次"修改"都产生新对象，所以频繁拼接要用 StringBuilder。

</details>

2. `String s = new String("hello")` 创建了几个对象？`s == "hello"` 的结果是什么？

<details><summary>参考答案</summary>

取决于 "hello" 是否已在常量池：若之前常量池中没有 "hello"，则创建 **2 个**对象——一个是编译期字面量 "hello" 驻留到字符串常量池的对象，一个是 `new` 在堆上创建的 String 对象；若 "hello" 已在常量池，则只创建 **1 个**（堆上 new 的那个，复用池中字面量）。`s == "hello"` 结果是 **false**：因为 s 指向 new 出来的堆对象，而字面量 "hello" 指向常量池对象，两者引用地址不同。`s.equals("hello")` 是 true（值相等），`s.intern() == "hello"` 是 true（intern 返回常量池中的引用）。结论：比较字符串内容要用 equals，== 比的是引用。

</details>

3. 为什么不能在循环里用 `+` 拼接字符串？正确做法是什么？

<details><summary>参考答案</summary>

因为 String 不可变，编译器把变量的 `+` 拼接转换成 `new StringBuilder().append(...).append(...).toString()`。在循环里写 `s += i` 时，**每一轮迭代都会新建一个 StringBuilder、append、再 toString 成新 String**，旧的中间字符串变成垃圾。n 次循环就创建 n 个 StringBuilder 和 n 个中间 String，时间复杂度退化到约 O(n²)，并产生大量临时对象加重 GC。正确做法是在循环外创建**一个** StringBuilder，循环内只调用 `sb.append(i)`，最后一次性 `toString()`，全程复用同一个可变缓冲区，复杂度 O(n)。注意：纯字面量/常量的 `+`（如 `"a"+"b"`）会被编译期常量折叠，没有这个问题；问题只出在循环中对变量反复 `+`。

</details>

4. StringBuilder 和 StringBuffer 有什么区别？什么时候用哪个？

<details><summary>参考答案</summary>

两者都是可变字符序列，底层是可变的字符数组，append 时在原数组追加（不够则扩容），避免像 String 那样产生大量中间对象。区别在线程安全：**StringBuffer** 的方法都加了 `synchronized`，是线程安全的，但有锁开销；**StringBuilder** 没有同步，非线程安全，但单线程下更快。选择：**绝大多数场景用 StringBuilder**（字符串拼接通常在单个方法内、单线程完成，不存在共享）；只有当确实有多个线程共享同一个可变字符串缓冲区时才用 StringBuffer，但这种情况实际很少见（通常各线程用各自的 StringBuilder）。所以面试结论是：默认用 StringBuilder，StringBuffer 几乎是历史遗留。

</details>

5. 字符串常量池是什么？`intern()` 方法有什么作用？

<details><summary>参考答案</summary>

字符串常量池（String Pool）是 JVM 维护的一块用于复用字符串的存储区（Java 7 起位于堆中）。编译期出现的字符串字面量会进入 class 常量池，运行时驻留到字符串常量池；相同的字面量共享同一个 String 对象，从而节省内存（如 `String a="x", b="x"` 时 a==b 为 true）。`intern()` 的作用是手动与常量池交互：`s.intern()` 会在常量池中查找是否有值等于 s 的字符串，有则返回池中那个引用，没有则把 s 放入池中并返回。典型用途是把运行时动态生成的（如拼接、new 出来的）字符串规约到常量池，使内容相同的字符串能用 == 比较或共享内存。但要谨慎大量 intern——会占用常量池内存且 intern 本身有查找开销。

</details>

6. Java 9 的紧凑字符串（Compact Strings）做了什么优化？

<details><summary>参考答案</summary>

Java 9 把 String 的底层存储从 `char[]`（每个字符固定占 2 字节，UTF-16）改为 `byte[]` 加一个 `coder` 编码标志位。优化点：如果一个字符串的所有字符都能用 Latin-1（ISO-8859-1，单字节，覆盖常见英文、数字、符号）表示，就用 **1 字节/字符**存储，内存占用减半；只有当字符串包含非 Latin-1 字符（如中文、emoji）时才用 UTF-16 的 2 字节/字符。coder 标志记录当前用的是哪种编码。这个改动对开发者完全透明（API 不变），但对以英文/ASCII 为主的应用（绝大多数日志、配置、键名等），能显著降低字符串内存占用和 GC 压力，是 JDK 9 一项重要的内存优化。

</details>
