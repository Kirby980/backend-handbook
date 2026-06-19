# 精通 Python 函数、参数与作用域

> 作用域和参数是 Python 最容易踩坑的地方:默认可变参数、闭包延迟绑定、`global`/`nonlocal`,几乎每个进阶面试都问。本篇讲透 LEGB、闭包、参数机制与传参语义,串起 [P01 名字绑定](./P01-精通-Python-数据模型与对象.md)、[P06 装饰器](./P06-精通-Python-装饰器与上下文管理器.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、LEGB 作用域查找

Python 解析一个名字时,按 **LEGB** 顺序查找,找到即停:

- **L**ocal:当前函数内部
- **E**nclosing:外层(嵌套)函数的作用域
- **G**lobal:模块级
- **B**uiltin:内置(`len`、`print`、`range`…)

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        # 这里读 x:Local 无 → Enclosing 有 → 用 "enclosing"
        print(x)
    inner()
outer()
```

注意:**Python 没有块级作用域**——`if`/`for`/`while`/`with` 不创建新作用域,里面定义的变量泄漏到所在函数(这点和 C/Java 不同)。作用域的单位是**函数、模块、类**。

---

## 二、global 与 nonlocal

默认情况下,在函数里**赋值**一个名字会把它当作**局部变量**。要修改外层变量,需声明:

- **`global x`**:声明 `x` 是模块级全局变量,函数内对它的赋值作用于全局。
- **`nonlocal x`**:声明 `x` 来自**最近的外层函数**作用域(用于闭包修改外层局部变量)。

```python
count = 0
def inc():
    global count
    count += 1            # 不写 global 会报 UnboundLocalError

def make_counter():
    n = 0
    def step():
        nonlocal n        # 修改外层函数的 n
        n += 1
        return n
    return step
```

⚠️ **`UnboundLocalError` 经典坑**:函数里只要有**对某名字的赋值**(哪怕在后面),该名字整个函数内就被当作局部变量,在赋值前读取它会报错——即使全局有同名变量:

```python
x = 1
def f():
    print(x)   # UnboundLocalError!因为下面有 x = 2,x 被判定为局部
    x = 2
```

---

## 三、闭包

**闭包**:内层函数引用了外层函数的变量,即使外层已返回,这些变量仍被"记住"。

```python
def multiplier(factor):
    def mul(x):
        return x * factor      # 捕获自由变量 factor
    return mul
double = multiplier(2)
print(double(5))               # 10
```

闭包捕获的是**变量(名字),不是值**——这导致**延迟绑定(late binding)陷阱**:

```python
# ❌ 经典坑:循环里建闭包
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])    # [2, 2, 2] —— 都引用同一个 i,循环结束时 i=2
# ✅ 修复:用默认参数在定义时"快照"当前值
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])    # [0, 1, 2]
```

原因:lambda 捕获的是变量 `i` 这个名字,而非每次迭代的值;等到调用时才去读 `i`,此时循环早已结束、`i` 停在最后一个值。用 `i=i` 默认参数,把当前值在定义时绑定到参数上,即可快照。

---

## 四、参数:位置、关键字、可变

```python
def f(pos1, pos2, /, normal, *args, kw_only, **kwargs):
    ...
#       ↑ positional-only  ↑普通  ↑收集多余位置  ↑keyword-only  ↑收集多余关键字
```

- **位置参数 / 关键字参数**:调用时可按位置或 `name=value` 传。
- **`*args`**:收集多余的**位置**参数为 tuple;**`**kwargs`**:收集多余的**关键字**参数为 dict。
- **`/`(PEP 570,3.8)**:其左侧为**仅限位置**参数(调用方不能用关键字传)。
- **`*`**:其右侧为**仅限关键字(keyword-only)**参数(必须 `name=value` 传),用于强制调用方写清参数名、提升可读性。

```python
def connect(host, port, *, timeout=30, retries=3):   # timeout/retries 必须关键字传
    ...
connect("localhost", 5432, timeout=10)   # OK
connect("localhost", 5432, 10)           # TypeError:不能位置传 timeout
```

参数顺序固定:位置-only → 普通 → `*args` → keyword-only → `**kwargs`。

---

## 五、默认可变参数陷阱

**最著名的 Python 坑**:默认参数值在**函数定义时只创建一次**,被所有调用**共享**。如果默认值是可变对象(list/dict),多次调用会累积:

```python
# ❌ 坑
def add_item(item, bucket=[]):
    bucket.append(item)
    return bucket
print(add_item(1))   # [1]
print(add_item(2))   # [1, 2] —— 不是 [2]!共享了同一个默认 list
# ✅ 正确:用 None 哨兵,每次新建
def add_item(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

原因呼应 [P01](./P01-精通-Python-数据模型与对象.md):默认值对象在 `def` 执行时创建一次、存在函数对象的 `__defaults__` 里,之后每次调用不传该参数都复用同一对象;可变对象被原地修改后,下次调用就带着上次的状态。**规则:默认值永远用不可变对象;需要可变默认值时用 `None` 哨兵 + 函数内新建。**

---

## 六、传参语义:传对象引用

Python 的传参既不是"传值"也不是"传引用",而是 **"传对象引用(call by object reference / call by sharing)"**——把实参对象的引用绑定到形参名字上(等价于一次赋值,见 [P01](./P01-精通-Python-数据模型与对象.md))。

```python
def modify(lst, x):
    lst.append(x)        # 原地改可变对象 → 影响外部(同一对象)
    lst = [99]           # 重新绑定形参 → 不影响外部(只是本地改名)
data = [1, 2]
modify(data, 3)
print(data)              # [1, 2, 3] —— append 生效;lst=[99] 不影响
```

- 传入**可变对象**且函数**原地修改**它 → 外部可见(因为是同一对象)。
- 函数内对形参**重新赋值** → 只改本地名字绑定,不影响外部。
- 传入**不可变对象**,函数内"修改"必然是重新绑定 → 永不影响外部。

所以"函数会不会改到我的数据"取决于:对象是否可变 + 函数是原地改还是重新绑定。要保护实参可传副本(`func(data[:])`)。

---

## 陷阱清单

- **默认可变参数**:`def f(x, lst=[])` 跨调用共享累积。用 `None` 哨兵。
- **闭包延迟绑定**:循环里建 lambda/闭包都引用同一变量,结果全是最后一个值。用 `x=x` 默认参数快照,或用工厂函数。
- **`UnboundLocalError`**:函数里给某名字赋值,它整个函数就是局部的,赋值前读取报错。要改全局/外层用 `global`/`nonlocal`。
- **以为有块级作用域**:`for`/`if` 里的变量泄漏到函数级;循环变量在循环后仍存在。
- **滥用 `global`**:可变全局状态难测难维护;优先用返回值/参数/类封装。
- **`*args/**kwargs` 滥用**:掩盖真实签名、丢失类型提示;明确参数更好。
- **把函数调用 `f()` 当默认值期望每次执行**:默认值只在定义时求值一次(`def g(t=time.time())` 不会每次更新)。

---

## 2026 现状

- **keyword-only 参数(`*`)** 是现代 API 设计的推荐:布尔开关、配置项强制写参数名,提升可读与可维护(类似很多库的 `def f(*, verbose=False)`)。
- **类型注解([P14](./P14-精通-Python-类型注解与typing.md))** 配合参数让签名自文档化;`/` 和 `*` 与注解共同表达精确 API 契约。
- **`functools.partial`** 固定部分参数生成新函数,是闭包之外常用的"携带状态的可调用"手段(见 [P06](./P06-精通-Python-装饰器与上下文管理器.md))。
- 与 Go/Java 对照:Python 无块级作用域、传对象引用;Go 是传值(指针显式)、Java 是传值(对象传引用的值)——理解差异避免跨语言惯性出错。

---

## 练习题

1. 什么是 LEGB 规则?Python 有块级作用域吗?

<details><summary>参考答案</summary>

**LEGB** 是 Python 解析(查找)一个名字时遵循的作用域顺序,从内到外依次查找,找到即停:**L**ocal(当前函数的局部作用域)→ **E**nclosing(外层嵌套函数的作用域,即闭包环境)→ **G**lobal(当前模块的全局作用域)→ **B**uiltin(内置作用域,如 `len`、`print`、`range` 等)。例如在嵌套函数里读一个变量,先看本函数局部有没有,没有就看外层函数,再看模块全局,最后看内置;都没有则 `NameError`。**Python 没有块级作用域**:`if`、`for`、`while`、`with`、`try` 这些语句块**不创建新的作用域**——在它们内部定义的变量会"泄漏"到所在的**函数(或模块)作用域**,在块外依然可见。例如 `for i in range(3): pass` 之后 `i` 仍然存在且等于 2;`if True: x = 1` 之后 `x` 在函数其余部分都可用。这与 C/C++/Java 的 `{}` 块级作用域不同,是 Python 初学者常困惑的点。Python 中创建新作用域的单位是**函数、模块、类**(以及推导式有自己的作用域,见 P07),而不是任意代码块。理解这点对避免变量意外泄漏/覆盖、理解循环变量在循环后仍存在等行为很重要。

</details>

2. `global` 和 `nonlocal` 分别用来做什么?什么是 UnboundLocalError?

<details><summary>参考答案</summary>

默认情况下,在函数内部对一个名字进行**赋值**,Python 就把该名字视为这个函数的**局部变量**。如果想在函数里修改外层的变量,需要显式声明:**`global x`** 声明 `x` 指向**模块级全局变量**,此后函数内对 `x` 的赋值会作用到全局(否则赋值只会创建一个同名局部变量);**`nonlocal x`** 声明 `x` 指向**最近的外层(enclosing)函数**作用域中的变量,用于在嵌套函数/闭包中修改外层函数的局部变量(典型如计数器闭包 `nonlocal n; n += 1`)。**`UnboundLocalError`** 是一个经典陷阱:由于"函数内只要存在对某名字的赋值语句,该名字在整个函数体内就被静态判定为局部变量"(这是在编译期决定的,与赋值语句的位置无关),如果在赋值之前就**读取**这个名字,就会报 `UnboundLocalError: local variable referenced before assignment`——即使外层/全局存在同名变量也不会去用它。例如 `x=1; def f(): print(x); x=2` 中,因为 `f` 里有 `x=2`,`x` 被判定为 `f` 的局部变量,`print(x)` 在它被赋值前执行,于是报错(而不是打印全局的 1)。解决:若本意是读全局/外层,就用 `global`/`nonlocal` 声明;若本意是用局部,确保先赋值再使用。

</details>

3. 解释默认可变参数的陷阱,以及正确的写法。

<details><summary>参考答案</summary>

陷阱:函数的**默认参数值在函数定义(`def` 执行)时就被创建,且只创建这一次**,之后存储在函数对象上(`func.__defaults__`),被所有"未传该参数"的调用**共享同一个对象**。如果默认值是**可变对象**(如 `[]`、`{}`),那么对它的原地修改会在多次调用之间累积保留,造成出乎意料的结果。例如 `def add(item, bucket=[]): bucket.append(item); return bucket`,第一次 `add(1)` 返回 `[1]`,第二次 `add(2)` 返回的不是 `[2]` 而是 `[1, 2]`——因为两次调用用的是定义时创建的**同一个** list,第一次 append 进去的 1 还在。**正确写法**是用 **`None` 作为哨兵默认值,在函数体内判断并新建对象**:`def add(item, bucket=None): if bucket is None: bucket = []; bucket.append(item); return bucket`——这样每次未传 `bucket` 时都新建一个独立的空 list,互不影响。原则:**默认参数值应永远使用不可变对象**(数字、字符串、None、tuple);需要"默认是空列表/空字典/新对象"的语义时,一律用 `None` 哨兵 + 函数内新建。这个坑的根源与"赋值不复制、默认值在定义时求值一次"的语义一致(见 P01)。同理,`def g(t=time.time())` 的默认值也只在定义时求值一次、不会每次调用更新。

</details>

4. 什么是闭包?循环里创建闭包的"延迟绑定"陷阱是怎么回事?

<details><summary>参考答案</summary>

**闭包(closure)**指一个内层(嵌套)函数引用了其外层函数作用域中的变量(自由变量),并且即使外层函数已经返回、其局部环境本应销毁,这些被引用的变量依然被内层函数"记住"并可访问。例如 `def multiplier(f): def mul(x): return x*f; return mul`,`multiplier(2)` 返回的 `mul` 记住了 `f=2`,之后调用 `mul(5)` 得 10——`f` 被闭包捕获了。**延迟绑定(late binding)陷阱**的关键在于:**闭包捕获的是变量(名字、引用),而不是创建闭包那一刻该变量的值**;自由变量的值是在闭包**被调用时**才去查找的。所以在循环里批量创建闭包时会出问题:`funcs = [lambda: i for i in range(3)]` 创建了 3 个 lambda,它们都捕获了**同一个变量 `i`**;等到后面调用 `f()` 时才去读 `i`,而此时循环早已结束、`i` 停在最后的值 2,于是三个函数都返回 2(`[2, 2, 2]`),而不是期望的 `[0, 1, 2]`。**修复**:把"当前迭代的值"在定义时就绑定下来——最常用是利用默认参数在定义时求值的特性:`[lambda i=i: i for i in range(3)]`(每个 lambda 有自己的默认参数 `i`,在创建时被赋为当前循环值),或用工厂函数 `def make(i): return lambda: i` 为每次迭代创建独立作用域,或用 `functools.partial`。理解这点也解释了为什么很多"循环里注册回调"的 bug 都表现为"所有回调都用了最后一个值"。

</details>

5. `*args`、`**kwargs`、`/`、`*` 在函数参数里分别是什么含义?

<details><summary>参考答案</summary>

①**`*args`**:在函数定义中收集所有**多余的位置参数**,打包成一个 **tuple**。例如 `def f(a, *args)`,调用 `f(1, 2, 3)` 时 `a=1`、`args=(2, 3)`。在调用处 `f(*iterable)` 则是**解包**,把可迭代对象展开为位置实参。②**`**kwargs`**:收集所有**多余的关键字参数**,打包成一个 **dict**。例如 `def f(a, **kwargs)`,`f(1, x=2, y=3)` 时 `kwargs={'x':2,'y':3}`;调用处 `f(**dict)` 把字典解包为关键字实参。二者常用于编写通用包装器/装饰器(见 P06)透传任意参数。③**`/`(PEP 570,3.8+)**:出现在参数列表中,表示**它左侧的参数是"仅限位置(positional-only)"**——调用方只能按位置传、不能用关键字名传。例如 `def f(x, /, y)`,可以 `f(1, 2)` 或 `f(1, y=2)`,但不能 `f(x=1, y=2)`。用于隐藏参数名(便于以后改名)或模仿内置函数行为。④**`*`(单独一个星号)**:出现在参数列表中,表示**它右侧的参数是"仅限关键字(keyword-only)"**——调用方必须用 `name=value` 形式传、不能按位置传。例如 `def connect(host, port, *, timeout=30)`,`timeout` 只能 `connect("h", 5432, timeout=10)`,不能 `connect("h", 5432, 10)`。keyword-only 参数能强制调用方写清参数名,提升可读性、避免位置传参出错,是现代 API(尤其布尔开关、可选配置)的推荐做法。完整参数顺序:positional-only(`/` 前)→ 普通 → `*args`(或单独 `*`)→ keyword-only → `**kwargs`。

</details>

6. Python 的参数传递是"传值"还是"传引用"?

<details><summary>参考答案</summary>

都不完全是。Python 的参数传递机制准确说是**"传对象引用"(call by object reference)**,也叫 **call by sharing(传共享)**:调用函数时,把**实参对象的引用**绑定到函数的形参名字上——这本质上等同于做了一次赋值(形参 = 实参对象),形参和实参**指向同一个对象**(见 P01 的"赋值即绑定")。它的行为取决于两点:对象是否可变、以及函数内是"原地修改"还是"重新绑定"。①如果传入的是**可变对象**(list/dict 等),并且函数内**原地修改**它(`lst.append(...)`、`d[k]=v`),由于形参和实参是同一个对象,修改对**外部可见**;②如果函数内对形参**重新赋值**(`lst = [99]`),只是把形参这个**本地名字**改绑到新对象上,**不影响**外部实参原来指向的对象;③如果传入的是**不可变对象**(int/str/tuple),函数内任何"修改"本质都是重新绑定,所以**永远不会影响**外部。所以"传值/传引用"的二分法不适用:它既不像 C 那样拷贝整个值(大对象不会被复制),也不像 C++ 引用那样能让 `形参=新值` 影响实参。要点记忆:**传的是引用的副本(指向同一对象),原地改可变对象会影响外部,重新绑定不会**。若想保证函数不改动调用方的可变数据,应传入副本(如 `func(data[:])` 或 `copy.deepcopy`)或在函数内部先拷贝。这与 Go(传值,需显式指针)、Java(传值,但对象变量持有的是引用的副本)的语义各有不同。

</details>
