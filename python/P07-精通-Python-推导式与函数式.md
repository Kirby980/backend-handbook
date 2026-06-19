# 精通 Python 推导式与函数式

> 推导式是 Python 最具辨识度的语法,写得好简洁优雅、写得过头则晦涩难读。配合 `map`/`filter`/`lambda` 等函数式工具,能写出声明式的数据处理。本篇讲清推导式、生成器表达式与函数式风格的取舍,依赖 [P05 生成器](./P05-精通-Python-迭代器与生成器.md)、[P04 lambda 作用域](./P04-精通-Python-函数参数与作用域.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、三种推导式

推导式用一行声明式地从可迭代对象构造容器:

```python
# 列表推导
squares = [x*x for x in range(10)]
# 字典推导
name_len = {name: len(name) for name in names}
# 集合推导(自动去重)
unique_lens = {len(name) for name in names}
```

通用形式:`[表达式 for 变量 in 可迭代 if 条件]`。比等价的 `for` 循环 + `append` 更短、更快(底层优化)、意图更清晰。

---

## 二、生成器表达式:惰性版

把推导的 `[]` 换成 `()` 得到**生成器表达式**——惰性、省内存(见 [P05](./P05-精通-Python-迭代器与生成器.md)):

```python
total = sum(x*x for x in range(10**7))   # 不建千万元素的 list,逐个产出
# 作为唯一实参时可省略外层括号 ↑
```

**只遍历一次 + 喂给消费函数(sum/any/max/join)用生成器表达式;要复用/索引用列表推导。**

---

## 三、过滤、转换与嵌套

```python
# 带条件过滤
evens = [x for x in nums if x % 2 == 0]
# 表达式里用三元(转换,不是过滤)
labels = ["偶" if x % 2 == 0 else "奇" for x in nums]
# 多重 for(展平)——注意顺序与嵌套循环一致
flat = [x for row in matrix for x in row]
# 嵌套推导(构造二维)
grid = [[0 for _ in range(cols)] for _ in range(rows)]
```

注意区分:**`if` 在 `for` 后面**是过滤;**三元 `a if cond else b` 在表达式位置**是按条件取值(不能省略 else)。多重 for 的书写顺序和等价嵌套循环相同。

---

## 四、map / filter / reduce

```python
list(map(str.upper, words))              # 对每个元素应用函数
list(filter(lambda x: x > 0, nums))      # 保留满足条件的
from functools import reduce
reduce(lambda a, b: a + b, nums, 0)      # 累积归约成单值
```

- `map`/`filter` 返回**惰性迭代器**(3 中),要 list 化才看到结果。
- Pythonic 风格里**推导式通常优于 `map`/`filter` + lambda**(更易读):`[f(x) for x in xs]` 比 `map(f, xs)` 更清晰,`[x for x in xs if p(x)]` 比 `filter(p, xs)` 更直观。
- `map(已有函数, xs)`(如 `map(int, strs)`)仍简洁好用;`reduce` 多数场景可被 `sum`/`math.prod`/循环替代,慎用。

---

## 五、lambda

`lambda` 是**单表达式匿名函数**,常作回调/key:

```python
data.sort(key=lambda p: p.age)           # 排序 key
sorted(items, key=lambda kv: kv[1], reverse=True)
```

- 只能是**一个表达式**(无语句、无多行);复杂逻辑请用 `def` 命名函数(更可读、可测试、可加文档)。
- 排序/分组的 `key=` 是 lambda 最佳用武之地;也可用 `operator.itemgetter/attrgetter` 替代(更快更清晰):`key=operator.attrgetter("age")`。

---

## 六、何时用推导、何时用循环

- **用推导式**:从一个可迭代**构造一个新容器**、逻辑简单(一两个 for + 一个 if)。
- **用普通循环**:有副作用(写文件、打印、修改外部状态)、逻辑复杂(多分支、try)、嵌套太深影响可读时。**别为了"一行"硬塞**——三层嵌套推导没人看得懂。
- **海象运算符 `:=`(3.8)**:在表达式里赋值,能减少重复计算:
  ```python
  results = [y for x in data if (y := f(x)) > 0]   # f(x) 只算一次,复用为 y
  ```

可读性优先于"炫一行"。

---

## 陷阱清单

- **推导式过度嵌套**:三层以上推导晦涩难懂,改用循环或拆分。
- **推导式里塞副作用**:`[print(x) for x in xs]` 是反模式(为副作用建了个没用的 list);要副作用就用 `for` 循环。
- **`map`/`filter` 忘了 list 化**:3 中它们返回惰性迭代器,直接打印看到的是对象;且一次性耗尽。
- **生成器表达式当 list 用**:不能下标、只能遍历一次。
- **lambda 写复杂逻辑**:可读性差、不可加名字/文档;复杂就 `def`。
- **闭包/lambda 在循环里延迟绑定**:`[lambda: i for i in range(3)]` 全返回 2(见 [P04](./P04-精通-Python-函数参数与作用域.md));用 `lambda i=i: i`。
- **推导式作用域误解**:推导式有**独立作用域**(3 中),循环变量不泄漏到外部(与普通 for 不同)。

---

## 2026 现状

- **推导式是 Pythonic 的标志**,数据转换/过滤的默认写法;`dict`/`set` 推导广泛用于建索引、去重。
- **生成器表达式 + 聚合函数**(`sum`/`any`/`all`/`max`)是处理大数据流、保持惰性的标准组合。
- **海象运算符 `:=`** 在推导式里避免重复计算的场景越来越常见。
- **`operator` 模块**(`itemgetter`/`attrgetter`)在排序/分组中比 lambda 更快更清晰,生产代码常用。
- 与 Java Stream([J31](../java/INDEX.md))对照:推导式 ≈ Stream 的 `map/filter/collect`,但推导式是语言级语法、更轻;Java Stream 更适合长链式与并行。

---

## 练习题

1. 列表推导相比等价的 for 循环 + append 有什么优势?

<details><summary>参考答案</summary>

①**更简洁、更声明式**:一行表达"从某可迭代对象,(按条件筛选并)转换出一个新列表"的意图,如 `[x*x for x in nums if x>0]`,比 `result=[]; for x in nums: if x>0: result.append(x*x)` 三四行更紧凑、更易读(在逻辑简单时);②**通常更快**:CPython 对列表推导有专门优化——它在字节码层面避免了反复查找并调用 `list.append` 方法(普通循环每次 append 都要做属性查找和方法调用),推导式内部用更高效的指令构建列表,所以同样逻辑下推导式一般比手写循环 append 略快;③**意图聚焦**:推导式表达的是"构造一个新容器",读者一眼知道结果是个列表;④**有独立作用域**:推导式的循环变量不会泄漏到外部作用域(Python 3 中),避免污染。**但要注意适用边界**:推导式只适合"从可迭代构造新容器且逻辑简单"的场景;如果有**副作用**(打印、写文件、改外部状态)、**复杂逻辑**(多重分支、try/except、需要中间变量)、或**嵌套过深**(三层以上),则应该用普通 for 循环,否则会牺牲可读性、甚至产生"为了副作用而建一个没用的列表"的反模式(如 `[print(x) for x in xs]`)。一句话:简单转换/过滤用推导式(简洁高效),复杂或有副作用用循环。

</details>

2. 列表推导和生成器表达式怎么选?(可结合 P05)

<details><summary>参考答案</summary>

核心区别:**列表推导 `[...]` 立即构建出完整列表**(占 O(n) 内存,可下标、可多次遍历、有 len);**生成器表达式 `(...)` 返回惰性迭代器**(几乎不占额外内存,只能遍历一次,不支持下标/len)。选择:①当结果**只需消费一次**、尤其直接传给聚合/消费函数(`sum`、`any`、`all`、`max`、`min`、`"".join`、`for`)时,用**生成器表达式**——省内存,且 `any`/`all` 等能短路提前结束,数据量大时优势明显,如 `sum(x*x for x in range(10**7))` 不会先建千万元素的列表;②当需要**多次遍历、随机下标访问、求 len、或传给会迭代它多次的代码**时,用**列表推导**得到实体列表;③数据可能很大或无限时**必须**用生成器表达式;④生成器表达式作为函数**唯一实参**时可省略外层括号(`sum(x for x in xs)`)。一个常见坑:生成器只能消费一次,遍历完再用会得到空。总之:一次性流式消费、省内存 → 生成器表达式;要复用/索引 → 列表推导。

</details>

3. 推导式里的 `if` 和三元表达式 `a if cond else b` 有什么区别?

<details><summary>参考答案</summary>

它们在推导式里位置不同、作用不同。①**`if` 放在 `for` 子句之后**(末尾),是**过滤(filter)**——只有满足条件的元素才会被纳入结果,不满足的被跳过、不出现在结果里。形式:`[expr for x in it if cond]`,如 `[x for x in nums if x>0]` 只保留正数。这种 `if` **没有 else**(它是筛选,不是取值)。②**三元表达式 `a if cond else b` 放在推导式开头的"表达式"位置**,是**按条件取值/转换(map)**——对**每一个**元素都产生一个结果,只是结果根据条件在 a 和 b 之间二选一,元素个数不变。形式:`[(a if cond else b) for x in it]`,如 `["偶" if x%2==0 else "奇" for x in nums]` 把每个数映射成标签,结果长度等于 nums 长度。这种三元**必须有 else**(它是表达式,要有确定的值)。区分要点:**末尾的 `if` 决定"留不留这个元素"(改变结果数量),开头的三元决定"这个元素变成什么"(不改变数量)**。两者可同时用:`[(a if c2 else b) for x in it if c1]`——先用末尾 `if c1` 过滤,再对保留下来的元素用三元 `c2` 取值。初学者常把"想过滤却写了 else"或"想转换却漏了 else"搞混导致语法错误或逻辑错误。

</details>

4. `map`/`filter` 和推导式相比,什么时候用哪个?

<details><summary>参考答案</summary>

`map(func, it)` 对每个元素应用函数做转换,`filter(pred, it)` 保留满足谓词的元素;在 Python 3 中它们都返回**惰性迭代器**(需 list 化或遍历才出结果,且一次性)。**Pythonic 风格通常更推荐推导式**,因为更易读、更直观:`[f(x) for x in xs]` 比 `map(f, xs)` 清晰,`[x for x in xs if p(x)]` 比 `filter(p, xs)` 直观,尤其当需要配合 `lambda` 时——`map(lambda x: x*2, xs)` 不如 `[x*2 for x in xs]` 简洁。**`map`/`filter` 仍有合适场景**:①当转换函数是**现成的具名函数/内置函数**时,`map` 很简洁,如 `map(int, str_list)`、`map(str.strip, lines)`——这里没有 lambda,比推导式 `[int(s) for s in str_list]` 差不多甚至更短;②需要**惰性**且喂给其他迭代器工具时(不过生成器表达式也能做到且更灵活);③函数式风格代码库或与 `itertools` 组合时。**`reduce`**(在 `functools` 里)做累积归约成单值,但可读性较差,多数场景能被 `sum`/`math.prod`/`max`/`"".join`/显式循环替代,应慎用、仅在确实是自定义二元累积时考虑。总结:**默认用推导式**(可读);转换是现成函数时 `map(func, it)` 也好;避免 `map/filter + lambda` 的组合(不如推导式),`reduce` 尽量用更具体的聚合替代。

</details>

5. lambda 适合用在哪里?什么时候不该用?

<details><summary>参考答案</summary>

`lambda` 是**只含单个表达式的匿名函数**,语法 `lambda 参数: 表达式`,返回该表达式的值。**适合的场景**是"需要一个简短的、用完即弃的小函数作为参数"——最典型的是作为 `key` 或回调:排序 `data.sort(key=lambda p: p.age)`、`sorted(items, key=lambda kv: kv[1])`、`max(words, key=lambda w: len(w))`、`min`/`heapq`/`groupby` 的 key,以及 GUI/异步里的简单回调等。它的好处是就地定义、不必为一个一次性的小逻辑单独起名。**不该用 lambda 的情况**:①**逻辑复杂或需要多条语句**——lambda 只能是单个表达式,不能有赋值、循环、try、多行;硬塞会很难读甚至写不出,应改用 `def` 具名函数;②**需要复用、需要文档字符串、需要单元测试、需要好的 traceback 名字**时——具名函数更清晰(lambda 的 `__name__` 都是 `'<lambda>'`,出错时栈信息不友好);③**把 lambda 赋值给变量**(如 `f = lambda x: x+1`)是反模式——PEP 8 明确建议这种情况直接 `def f(x): return x+1`,既有名字又可读;④**能用更专门的工具时**——如排序 key 取属性/下标,用 `operator.attrgetter("age")` / `operator.itemgetter(1)` 比 lambda 更快、更明确。总结:lambda 用于作参数的简短一次性逻辑(尤其 `key=`),逻辑一旦复杂或需要复用/测试/命名,就用 `def`。

</details>

6. 海象运算符 `:=` 是什么?在推导式里有什么用?

<details><summary>参考答案</summary>

海象运算符 `:=`(PEP 572,Python 3.8 引入,因形似海象的眼睛和獠牙得名)是**赋值表达式**——它在**表达式内部**完成赋值并**同时返回所赋的值**,从而能在只允许表达式、不允许赋值语句的地方"顺便记住一个值"。普通赋值 `x = 5` 是语句、没有值;而 `(x := 5)` 是表达式,既把 5 绑定给 x、整个表达式的值也是 5。**在推导式里的主要用途是避免重复计算**:当推导式的过滤条件和结果表达式都需要用到同一个"昂贵计算"的结果时,用 `:=` 计算一次并复用。例如 `[y for x in data if (y := f(x)) > 0]`——这里 `f(x)` 通过 `(y := f(x))` 只调用一次,把结果存进 `y`,既用于条件判断 `> 0`,又直接作为输出值 `y`;若不用海象,要么写 `[f(x) for x in data if f(x) > 0]`(`f(x)` 被算了两次,浪费且若有副作用会出错),要么改写成多行循环。其他常见用途:`while (line := f.readline()):` 边读边判断;`if (m := re.match(...)) :` 匹配并保留结果;`if (n := len(a)) > 10:` 同时拿到长度。注意事项:①不要滥用,过度使用会降低可读性;②有作用域细节(在推导式中用 `:=` 赋的变量会泄漏到包含推导式的外层作用域,这是个有意设计的例外);③别和普通 `=` 混淆,`:=` 用于表达式语境。总之它让"计算一次、即用即存"在表达式中成为可能,推导式里主要用来消除重复计算或重复调用。

</details>
