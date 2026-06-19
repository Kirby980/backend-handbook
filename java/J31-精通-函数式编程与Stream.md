# 精通函数式编程与 Stream

> Lambda / Stream 是 Java 8 的招牌，也是日常开发与面试的高频区。但很多人只会用、不懂原理：Lambda 凭什么不是匿名内部类的语法糖？Stream 的惰性和短路是怎么回事？并行流为什么可能更慢、还可能算错？Collector 怎么自定义？本篇把函数式这套讲透，补齐 [J29 版本演进](./J29-精通-Java版本特性演进.md) 中略讲的部分。
>
> **📅 基准：2026 年 6 月，Java 17/21 主流。**

---

## 一、函数式接口：Lambda 的"类型"

Lambda 不是凭空存在的，它必须有一个**目标类型**——**函数式接口（Functional Interface）**：**有且仅有一个抽象方法**的接口（可以有 default/static 方法），用 `@FunctionalInterface` 标注（编译器校验"只有一个抽象方法"）。

`java.util.function` 提供了一套标准函数式接口：

| 接口 | 抽象方法 | 语义 | 例子 |
|---|---|---|---|
| `Function<T,R>` | `R apply(T)` | 转换：T→R | `s -> s.length()` |
| `Predicate<T>` | `boolean test(T)` | 判断 | `s -> s.isEmpty()` |
| `Consumer<T>` | `void accept(T)` | 消费（副作用） | `System.out::println` |
| `Supplier<T>` | `T get()` | 提供/工厂 | `() -> new ArrayList<>()` |
| `UnaryOperator<T>` | `T apply(T)` | T→T | `s -> s.trim()` |
| `BiFunction<T,U,R>` | `R apply(T,U)` | (T,U)→R | `(a,b) -> a+b` |

> 为避免**自动装箱**开销，还有 `IntFunction`、`ToIntFunction`、`IntPredicate`、`IntUnaryOperator` 等基本类型特化版——大数据量处理时优先用它们，避免 `Integer` 装箱拆箱（呼应 [J01](./J01-精通-集合框架与List源码.md) 的装箱开销）。

**四种 Lambda / 方法引用写法**：

```java
Function<String,Integer> f1 = s -> s.length();          // Lambda
Function<String,Integer> f2 = String::length;           // 实例方法引用（首参作接收者）
Supplier<List<String>>   f3 = ArrayList::new;           // 构造方法引用
Function<String,Integer> f4 = Integer::parseInt;        // 静态方法引用
```

---

## 二、Lambda 为什么不是匿名内部类的语法糖

直觉上 `() -> doX()` 像匿名内部类 `new Runnable(){ public void run(){ doX(); } }` 的简写，但**编译产物完全不同**。

- **匿名内部类**：编译期就生成一个独立的 `Outer$1.class` 文件，运行时 `new` 一个实例。类越多，启动越慢、占用越多。
- **Lambda**：编译成一条 **`invokedynamic`** 字节码指令（[J21 字节码](./J21-精通-字节码与执行引擎.md)），把"生成实现类"的动作**延迟到运行期第一次执行时**，由 `LambdaMetafactory` 动态生成（通常用 `MethodHandle` 直接绑定方法，对无捕获的 Lambda 还能复用同一个实例）。

好处：不为每个 Lambda 生成 class 文件（class 数量爆炸会拖慢类加载、占元空间），运行期可优化，无捕获 Lambda 可单例复用。

**捕获规则**：Lambda 只能捕获 **final 或 effectively final（事实不变）**的局部变量。

```java
int base = 10;          // 事实不变 → 可捕获
Function<Integer,Integer> add = x -> x + base;
// base = 20;           // 取消注释则编译报错：base 不再 effectively final
```

为什么？局部变量在栈上、方法返回即销毁，而 Lambda 可能在方法返回后才执行，所以 Lambda 捕获的是变量的**值拷贝**；若允许变量再变，拷贝就和原值不一致、语义混乱，于是干脆要求"事实不变"。（对比 [J14](./J14-精通-ThreadLocal.md) 中匿名类捕获 `this` 引起的内存问题——Lambda 不持有外部类隐式 `this`，除非真的用到外部成员。）

---

## 三、Stream：声明式的数据流水线

Stream 把"对集合做什么"声明出来，而非写"怎么循环"：

```java
List<String> names = users.stream()          // 1. 创建流（源）
    .filter(u -> u.getAge() >= 18)            // 2. 中间操作：过滤
    .map(User::getName)                       // 3. 中间操作：转换
    .distinct().sorted()                      // 4. 中间操作：去重排序
    .collect(Collectors.toList());            // 5. 终端操作：收集（触发执行）
```

三段式：**源（Source）→ 中间操作（Intermediate）→ 终端操作（Terminal）**。

- **中间操作惰性**：`filter`/`map`/`sorted`/`distinct`/`limit`/`peek` 返回新 Stream，**不触发任何计算**，只是搭流水线。
- **终端操作触发执行**：`collect`/`forEach`/`reduce`/`count`/`findFirst`/`anyMatch` 才真正跑，且**消费后流即失效**（不能重用）。

---

## 四、惰性、流水线融合与短路

Stream 不是"每个中间操作完整遍历一遍集合再传给下一个"——而是**惰性 + 流水线融合**：终端操作触发后，**每个元素一次性走完整条流水线**（filter→map→...），不产生中间集合。

```java
Stream.of("a","bb","ccc","dddd")
    .filter(s -> { System.out.println("filter " + s); return s.length() % 2 == 1; })
    .map(s -> { System.out.println("map " + s); return s.toUpperCase(); })
    .findFirst();
// 输出（注意逐元素、且短路）：
// filter a / map a   → 找到第一个就停，bb/ccc/dddd 根本没被处理
```

- **逐元素流水线化**：减少遍历次数、不产生中间集合。
- **短路（short-circuit）**：`findFirst`/`findAny`/`anyMatch`/`allMatch`/`limit` 等满足条件即提前结束，甚至能处理**无限流**：
  ```java
  Stream.iterate(1, n -> n + 1).filter(n -> n % 7 == 0).limit(3).toList(); // [7,14,21]
  ```
- **常见错误**：只写中间操作、没有终端操作 → **什么都不会发生**（流没被触发）。`peek` 仅用于调试，别用它做业务副作用。

---

## 五、Collectors：归约的瑞士军刀

`collect(Collector)` 是最强大的终端操作。常用：

```java
// 分组：Map<部门, List<员工>>
Map<Dept, List<Emp>> byDept = emps.stream()
    .collect(Collectors.groupingBy(Emp::getDept));

// 分组 + 下游归约：每个部门的人数 / 平均薪资
Map<Dept, Long>   count = emps.stream().collect(groupingBy(Emp::getDept, counting()));
Map<Dept, Double> avg   = emps.stream().collect(groupingBy(Emp::getDept, averagingDouble(Emp::getSalary)));

// 拼接 / 分区 / 转 Map
String csv = names.stream().collect(Collectors.joining(", ", "[", "]"));
Map<Boolean, List<Emp>> parts = emps.stream().collect(partitioningBy(e -> e.getSalary() > 10000));
Map<Long, String> idToName = emps.stream().collect(toMap(Emp::getId, Emp::getName)); // key 重复会抛异常！
```

> `toMap` 在 **key 冲突时默认抛 `IllegalStateException`**——有重复风险时要传第三个合并函数：`toMap(k, v, (a,b) -> a)`。

**Collector 的本质**是四件套：`supplier`（建容器）+ `accumulator`（加元素）+ `combiner`（合并两个容器，并行用）+ `finisher`（收尾）。理解了就能自定义：

```java
// 等价于 Collectors.toList 的手写版
Collector<String, ?, List<String>> toList = Collector.of(
    ArrayList::new,                 // supplier
    List::add,                      // accumulator
    (a, b) -> { a.addAll(b); return a; }); // combiner（并行合并）
```

**`reduce` vs `collect`**：`reduce` 适合**不可变**归约（求和、求最值，`reduce(0, Integer::sum)`）；`collect` 适合往**可变容器**累加（list/map/StringBuilder）。可变累加别用 reduce（每步建新对象，浪费）。

---

## 六、并行流：免费的午餐有代价

`parallelStream()` / `stream().parallel()` 把流水线丢到 **`ForkJoinPool.commonPool()`** 上并行跑（[J11](./J11-精通-线程池.md) 的池思想）。看着是"加一个词就并行"，但坑很多：

1. **共用 common pool**：默认所有并行流共享一个公共池（大小≈核数-1），一个慢任务（尤其含阻塞 IO）会拖垮全局所有并行流。**并行流只适合 CPU 密集纯计算，绝不要在里面做阻塞 IO。**
2. **数据量小反而更慢**：拆分、调度、合并（combiner）有开销，小数据量得不偿失。
3. **线程安全**：lambda 里不能有共享可变状态。下面是**错误**示范：
   ```java
   List<Integer> result = new ArrayList<>();              // 非线程安全！
   list.parallelStream().forEach(result::add);            // 数据竞争 → 丢数据/异常
   ```
   正确做法是用 `collect`（其 combiner 保证安全合并）或 `forEachOrdered`。
4. **拆分代价**：数据源要可高效拆分（`ArrayList`/数组好拆，`LinkedList` 难拆，[J01](./J01-精通-集合框架与List源码.md)）。
5. **顺序**：`findFirst`、`forEachOrdered` 保序但牺牲并行收益；`findAny`/`forEach` 不保序。

**结论**：并行流是"CPU 密集 + 大数据量 + 无共享状态 + 易拆分数据源"时的优化，不是默认选项。绝大多数业务的瓶颈是 IO，并行流帮不上、还添乱。需要并行 IO 用[虚拟线程](./J30-精通-虚拟线程与结构化并发.md)或专用线程池。

---

## 七、Optional：优雅地表达"可能没有"

`Optional<T>` 是个容器，明确表达"值可能不存在"，减少 NPE 与满屏 null 判断：

```java
Optional<User> u = repo.findById(id);
String name = u.map(User::getName).orElse("匿名");        // 链式，没值给默认
repo.findById(id).ifPresentOrElse(this::send, this::warn); // 有值/无值分支
User user = repo.findById(id).orElseThrow(() -> new NotFound(id)); // 没值抛异常
```

**用法约束**（高频考点）：

- **正确定位**：作**方法返回值**表达"可能查不到"。**不要**用作字段、方法参数、集合元素。
- **别 `if (opt.isPresent()) opt.get()`**：这只是把 null 判断换了写法。要用 `map/filter/orElse/ifPresent` 的函数式风格。
- **`orElse` vs `orElseGet`**：`orElse(expensive())` 的参数**无论有没有值都会求值**；`orElseGet(() -> expensive())` 只在没值时才求值——默认值代价大时用 `orElseGet`。
- 集合返回**空集合**而非 `Optional<List>`；基本类型用 `OptionalInt/Long/Double`。

---

## 陷阱清单

- **写了中间操作没有终端操作**：流是惰性的，没有终端操作什么都不会执行。
- **复用已消费的 Stream**：终端操作后流即失效，再用抛 `IllegalStateException`。
- **`toMap` 不处理 key 冲突**：默认抛异常，有重复 key 要传合并函数。
- **`Collectors.toList()` 当作可变/特定实现**：返回的 List 实现不保证可变（Java 16+ 的 `Stream.toList()` 返回**不可变** List），要可变用 `collect(toCollection(ArrayList::new))`。
- **并行流里做阻塞 IO**：拖垮共用的 common pool，影响全局。并行流只用于 CPU 密集纯计算。
- **并行流里改共享可变状态**：数据竞争。用 collect 安全归约。
- **小数据量上并行流**：拆分/合并开销大于收益，更慢。
- **`orElse` 里放重操作**：无论有无值都会执行，应改 `orElseGet`。
- **滥用 Optional 做字段/参数**：它是为"返回值可能为空"设计的，序列化、性能、语义都不适合当字段。
- **热点循环硬换 Stream**：Stream 有一定开销且难调试，性能极敏感的热点未必比 for 快——按需选择。

---

## 2026 现状

- **Lambda/Stream 是日常标配**，但团队规范通常要求"热点路径慎用、保持可读、避免炫技长链"。
- **`Stream.toList()`（Java 16+）** 取代 `collect(toList())` 成为取不可变列表的简洁写法（注意它**不可变**）。
- **`teeing`（Java 12+）** 收集器支持"一次遍历同时做两种归约再合并"（如同时求 min 和 max）。
- **Record + Stream** 组合常见：流处理产出 record DTO（[J29](./J29-精通-Java版本特性演进.md)）。
- **并行 IO 用虚拟线程而非并行流**：并行流定位回归"CPU 密集计算"，并行阻塞 IO 交给[虚拟线程](./J30-精通-虚拟线程与结构化并发.md)。
- **`Gatherer`（Java 24 稳定）**：可自定义的中间操作（类似 Collector 之于终端操作），补齐了 Stream 自定义中间步骤的能力，关注其在新版本的应用。

---

## 练习题

1. 什么是函数式接口？Lambda 和它什么关系？列举几个常用的标准函数式接口。

<details><summary>参考答案</summary>

**函数式接口（Functional Interface）**是**有且仅有一个抽象方法**的接口（可以额外有 default 方法和 static 方法，它们不算抽象方法），通常用 `@FunctionalInterface` 注解标注（该注解让编译器校验"确实只有一个抽象方法"，不满足就报错）。**Lambda 与它的关系**：Lambda 表达式本身没有独立类型，它必须有一个**目标类型**，而这个目标类型就是某个函数式接口——Lambda 就是该函数式接口唯一抽象方法的一个"实例/实现"。编译器根据上下文（赋值的变量类型、方法参数类型）推断 Lambda 对应哪个函数式接口，并把 Lambda 体作为那个抽象方法的实现。因为只有一个抽象方法，编译器才能无歧义地知道 Lambda 实现的是哪个方法。**常用标准函数式接口**（在 `java.util.function` 包）：`Function<T,R>`（apply，T→R 转换）、`Predicate<T>`（test，返回 boolean 做判断）、`Consumer<T>`（accept，消费一个值、无返回，做副作用如打印）、`Supplier<T>`（get，无参提供一个值，如工厂）、`UnaryOperator<T>`（T→T）、`BinaryOperator<T>`/`BiFunction<T,U,R>`（两个入参）等；此外还有 `IntFunction`、`ToIntFunction`、`IntPredicate` 等基本类型特化版，用来避免自动装箱开销。`Runnable`、`Comparator`、`Callable` 也都是函数式接口。

</details>

2. Lambda 是匿名内部类的语法糖吗？底层是怎么实现的？

<details><summary>参考答案</summary>

**不是**。虽然写法上 Lambda 看起来像匿名内部类的简写，但二者的编译产物和实现机制完全不同。**匿名内部类**在编译期就会生成一个独立的 class 文件（如 `Outer$1.class`），运行时通过 `new` 创建其实例；类一多就会产生大量 class 文件，拖慢类加载、增加元空间占用。**Lambda** 则被编译成一条 **`invokedynamic`** 字节码指令（Java 7 引入、为动态语言/Lambda 准备的指令）。它把"生成实现该函数式接口的类/对象"这一步**推迟到运行期第一次执行该 Lambda 时**，由 JDK 的 **`LambdaMetafactory`** 在运行时动态生成（通常借助 `MethodHandle` 直接绑定到目标方法）。好处有几点：①不在编译期为每个 Lambda 生成 class 文件，避免 class 数量爆炸；②运行期生成给了 JVM 优化空间，并可对**没有捕获任何外部变量**的 Lambda **复用同一个实例**（无状态、可单例），减少对象创建；③把实现策略交给运行时，未来可优化而不影响字节码。所以"Lambda 是匿名内部类语法糖"是常见误解——它是基于 invokedynamic 的运行期动态实现。补充：Lambda 不像匿名内部类那样隐式持有外部类的 `this`（除非真的访问了外部实例成员），这也减少了意外的强引用。

</details>

3. Stream 的中间操作和终端操作有什么区别？什么是惰性求值和短路？

<details><summary>参考答案</summary>

**中间操作（Intermediate）**：如 filter、map、sorted、distinct、limit、peek，返回值仍是 Stream，用于串联流水线；它们是**惰性的**——调用时不执行任何计算，只记录"要做什么"。**终端操作（Terminal）**：如 collect、forEach、reduce、count、findFirst、anyMatch，会触发整条流水线真正执行并产生结果（集合/值/副作用），终端操作执行后该 Stream 即被消费、不能再用。**惰性求值（lazy evaluation）**：中间操作不立即执行，只有遇到终端操作时才从源开始按流水线处理元素。它带来两个关键优化：①**流水线融合**——不是每个中间操作完整遍历一遍集合再传给下一个，而是**每个元素一次性走完整条流水线**（filter→map→…），减少遍历次数、不产生中间集合；②支持**短路**。**短路（short-circuit）**：findFirst、findAny、anyMatch、allMatch、limit 等操作可以在满足条件后**提前结束**，不必处理全部元素——例如 `stream.filter(...).map(...).findFirst()` 找到第一个满足的元素就停，后面的元素根本不会被 filter/map 处理；正因如此，Stream 甚至能处理**无限流**（如 `Stream.iterate(...).limit(n)`）。一个常见错误：只写了中间操作而**没有终端操作**，由于惰性，整条流水线根本不会执行、什么都不会发生。

</details>

4. 并行流（parallelStream）用要注意什么？为什么说它不是免费的午餐？

<details><summary>参考答案</summary>

`parallelStream()` 把流水线提交到 **`ForkJoinPool.commonPool()`** 上并行处理，看似"加一个词就并行加速"，但有诸多陷阱：①**共用公共池**——默认所有并行流共享同一个 commonPool（大小约为 CPU 核数-1），如果某个并行流任务很慢、尤其在里面做了**阻塞 IO**，会占满公共池、拖垮 JVM 内所有其他并行流，所以**并行流只能用于 CPU 密集的纯计算，绝不能在其中执行阻塞 IO**（并行 IO 应交给虚拟线程或专用线程池）；②**小数据量反而更慢**——任务拆分、线程调度、结果合并（combiner）都有开销，数据量小时收益抵不过开销；③**线程安全**——并行流的 lambda 中绝不能修改共享可变状态（如往一个普通 ArrayList 里 add），否则发生数据竞争导致结果错误或异常，应该用 `collect` 这种带安全 combiner 的归约方式；④**数据源要易拆分**——ArrayList/数组能按下标高效二分，LinkedList、IO 流等难以高效拆分，并行收益差；⑤**有序性开销**——findFirst、forEachOrdered 等保序操作会限制并行度。所以并行流是"CPU 密集 + 大数据量 + 无共享可变状态 + 数据源易拆分"这些条件都满足时的优化手段，而非默认选项；多数业务瓶颈在 IO，盲目用并行流既帮不上忙又可能引入全局性能问题和正确性 bug。

</details>

5. Optional 应该怎么用？orElse 和 orElseGet 有什么区别？

<details><summary>参考答案</summary>

`Optional<T>` 是一个可能包含值、也可能为空的容器，用来**显式表达"值可能不存在"**，从而减少 NullPointerException 和满屏的 null 判断。**正确用法**：①主要用作**方法的返回值**，表达"可能查不到结果"（如 `findById` 返回 `Optional<User>`）；**不要**把它用作类的字段、方法参数或集合元素（会带来序列化、内存和语义问题，集合应直接返回空集合而非 Optional）；②应使用函数式风格 `map`/`filter`/`flatMap`/`ifPresent`/`ifPresentOrElse`/`orElse`/`orElseThrow` 进行链式处理，而**不要**写 `if (opt.isPresent()) { opt.get(); }`——那只是把 null 判断换了个写法、没有发挥 Optional 的价值，且 `get()` 在空时会抛异常；③基本类型用 `OptionalInt/OptionalLong/OptionalDouble` 避免装箱。**orElse 和 orElseGet 的区别**（高频考点）：`orElse(T other)` 接收的是一个**已经计算好的值**，无论 Optional 是否有值，`other` 表达式**都会被求值**；而 `orElseGet(Supplier)` 接收的是一个 Supplier，**只有在 Optional 为空时才会调用** Supplier 求值。所以当默认值的获取代价较大（如要查数据库、new 重对象、有副作用）时，必须用 `orElseGet(() -> expensive())`，否则用 `orElse(expensive())` 会导致即使有值也白白执行了昂贵的默认值计算。例如 `opt.orElse(queryDefaultFromDB())` 每次都会查库，而 `opt.orElseGet(() -> queryDefaultFromDB())` 只在没值时才查。

</details>

6. reduce 和 collect 有什么区别？什么时候用哪个？Collector 由哪几部分组成？

<details><summary>参考答案</summary>

**reduce 和 collect 都是终端归约操作，但适用场景不同**。`reduce` 是**不可变归约**——它从一个初始值开始，用一个二元函数把元素两两合并成一个结果，适合求和、求积、求最大/最小值、字符串拼接成单值等"把流归约成一个（通常不可变的）值"的场景，如 `stream.reduce(0, Integer::sum)`。它的每一步通常产生新值（对不可变类型无所谓）。`collect` 是**可变归约（mutable reduction）**——它把元素不断累加进一个**可变容器**中，适合收集成 List/Set/Map、拼接进 StringBuilder、分组、分区等场景，如 `collect(Collectors.toList())`、`collect(groupingBy(...))`。**选择原则**：归约成单个不可变值（数值、布尔、单一对象）用 reduce；要把元素装进可变容器（集合、Map、StringBuilder）用 collect——如果用 reduce 往集合里"累加"会在每一步创建新集合，性能极差，所以可变容器累加一定用 collect。**Collector 的组成**（四件套）：①`supplier`——创建一个新的结果容器（如 `ArrayList::new`）；②`accumulator`——把一个元素加入容器（如 `List::add`）；③`combiner`——把两个部分结果容器合并成一个（并行流时把各线程的局部结果合并，如 `(a,b)->{a.addAll(b);return a;}`）；④`finisher`——对累加完的容器做最后转换（如转成不可变视图，不需要时是恒等函数）；外加一组 characteristics（如 CONCURRENT、UNORDERED、IDENTITY_FINISH）描述其特性以便优化。理解这四件套就能用 `Collector.of(...)` 自定义收集器。

</details>
