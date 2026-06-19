# 精通 Python 鸭子类型、ABC 与 Protocol

> "如果它走起来像鸭子、叫起来像鸭子,那它就是鸭子"——鸭子类型是 Python 多态的灵魂:不看类型,看能力。但有时需要"显式约定接口",于是有了抽象基类(ABC)和协议(Protocol)。面试问:"什么是鸭子类型?""ABC 和 Protocol 区别?"本篇讲清,串起 [P12 元类](./P12-精通-Python-元类与类创建.md)、[P14 类型注解](./P14-精通-Python-类型注解与typing.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、鸭子类型

Python 不关心对象"是什么类型",只关心它"**能做什么**"——只要实现了所需的方法/协议,就能用。

```python
def total_length(items):
    return sum(len(x) for x in items)   # 只要每个 x 支持 len() 就行

total_length(["ab", "cde"])             # 字符串
total_length([[1,2], [3,4,5]])          # 列表
total_length([{1,2}, {3}])              # 集合
# 这些类型毫无继承关系,但都"支持 len()",所以都能用
```

不需要共同基类、不需要声明"我实现了 Sizeable 接口"——能 `len()` 就行。这就是鸭子类型:**多态靠"能力"而非"类型/继承"**。`for`(可迭代协议)、`with`(上下文协议)、`[]`(`__getitem__`)等都是鸭子类型的体现(见 [P05](./P05-精通-Python-迭代器与生成器.md)/[P06](./P06-精通-Python-装饰器与上下文管理器.md))。

---

## 二、鸭子类型与 EAFP

鸭子类型天然配合 **EAFP**(见 [P08](./P08-精通-Python-异常处理.md)):直接用,不行再处理,而不是先 `isinstance` 检查类型。

```python
# ❌ 反鸭子:硬查类型,限制了可用类型
if isinstance(x, list):
    x.append(y)
# ✅ 鸭子 + EAFP:谁支持 append 都能用
try:
    x.append(y)
except AttributeError:
    ...
```

过度 `isinstance` 检查会**破坏鸭子类型的灵活性**(把函数锁死在具体类型上)。但完全不约束有时也有问题——接口不明确、IDE/类型检查器无从帮忙。于是有了 ABC 和 Protocol 来"显式表达接口"。

---

## 三、抽象基类 ABC

`abc` 模块用于定义**抽象基类**——声明"子类必须实现哪些方法",否则不能实例化:

```python
from abc import ABC, abstractmethod

class Storage(ABC):                       # 继承 ABC(其元类是 ABCMeta)
    @abstractmethod
    def save(self, key, data): ...        # 抽象方法,子类必须实现
    @abstractmethod
    def load(self, key): ...

class FileStorage(Storage):
    def save(self, key, data): ...        # 实现了
    def load(self, key): ...

# s = Storage()        # TypeError:含抽象方法,不能实例化
FileStorage()          # OK:实现了全部抽象方法
```

- 含未实现抽象方法的类**不能实例化**——把"忘了实现接口"的错误从运行时提前到实例化时。
- ABC 是**名义子类型(nominal)**:子类要么显式继承 ABC,要么用 `Storage.register(SomeClass)` 注册为"虚拟子类"(让 `isinstance` 通过,但不强制实现)。
- ABC 的元类是 `ABCMeta`(见 [P12](./P12-精通-Python-元类与类创建.md))。

---

## 四、collections.abc:现成的抽象基类

标准库 `collections.abc` 提供一组容器抽象基类——**实现少量核心方法,就免费获得一整套**:

```python
from collections.abc import Sequence

class MyList(Sequence):
    def __init__(self, data): self._d = list(data)
    def __getitem__(self, i): return self._d[i]   # 只需实现这两个
    def __len__(self): return len(self._d)
    # 自动获得 __contains__/__iter__/__reversed__/index/count(由 ABC 提供)

ml = MyList([1, 2, 3])
print(3 in ml, list(reversed(ml)), ml.index(2))   # 都能用!
```

常用:`Iterable`/`Iterator`、`Sequence`/`MutableSequence`、`Mapping`/`MutableMapping`、`Set`、`Callable`、`Hashable`。`isinstance(x, Iterable)` 可判断对象是否可迭代(基于是否有 `__iter__`)。

---

## 五、Protocol:结构化子类型(静态鸭子类型)

`typing.Protocol`(PEP 544,3.8)把鸭子类型带进**静态类型检查**——它是**结构化子类型(structural)**:一个类**无需显式继承** Protocol,只要"长得像"(实现了协议要求的方法/属性),类型检查器(mypy/Pyright)就认为它符合该协议。

```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...          # 只要有 close() 方法就算符合

def shutdown(resource: SupportsClose) -> None:
    resource.close()

class Conn:
    def close(self) -> None: print("closed")   # 没继承 SupportsClose

shutdown(Conn())     # 类型检查通过!Conn "长得像" SupportsClose
```

- **静态鸭子类型**:既保留鸭子类型的灵活(不强制继承),又让类型检查器能验证接口、IDE 能补全。
- **`@runtime_checkable`**:加上它后可用 `isinstance(x, SupportsClose)` 在运行时检查(只查方法名存在性,不查签名)。
- 对比:**ABC 是名义的(要继承/注册),Protocol 是结构的(长得像就行)**——后者更贴合鸭子类型精神。

---

## 六、三者怎么选

| 需求 | 用 |
|---|---|
| 纯动态、不在乎静态检查 | **鸭子类型**(直接用 + EAFP) |
| 要强制子类实现某些方法、共享部分实现 | **ABC**(继承 + `@abstractmethod`) |
| 要静态类型检查/IDE 支持,但不想强制继承 | **Protocol**(结构化) |
| 自定义容器想白嫖一套方法 | **collections.abc**(继承 Sequence/Mapping 等) |

现代趋势:**库的公共接口用 Protocol 表达**(灵活 + 可类型检查);需要复用实现或强约束时用 ABC;应用内部仍大量靠鸭子类型。

---

## 陷阱清单

- **过度 `isinstance` 检查**:破坏鸭子类型灵活性,把函数锁死到具体类型;优先靠协议/EAFP。
- **ABC 抽象方法没全实现就实例化**:`TypeError`——这是特性(提前报错),但要理解。
- **以为 Protocol 要继承**:Protocol 是结构化的,不继承也算符合(继承它只是为了显式声明或共享默认实现)。
- **`isinstance` 对 Protocol**:必须 `@runtime_checkable` 才能用,且只查方法**存在性**、不查签名/类型,可能误判。
- **ABC 的 `register` 误用**:注册虚拟子类只让 `isinstance` 通过,不会强制/检查实现,可能"假装符合"。
- **混淆名义 vs 结构子类型**:ABC=名义(要继承/注册),Protocol=结构(长得像);选错增加耦合或失去检查。

---

## 2026 现状

- **Protocol 成为表达接口的现代首选**:配合类型注解(见 [P14](./P14-精通-Python-类型注解与typing.md)),库 API 用 Protocol 既灵活又可静态检查,广泛用于依赖注入、可替换组件。
- **`collections.abc`** 仍是实现自定义容器的标准方式;`typing` 里对应的泛型别名(`Iterable[int]` 等)用于注解。
- **ABC** 在需要"强制契约 + 共享实现"的框架基类中常见(如 `Storage`/`BaseHandler`)。
- 与 Go/Java 对照:**Go 的隐式接口几乎等于 Protocol**(结构化、不用声明实现);**Java 的 interface 是名义的**(要 `implements`),更像 ABC。Python 三种都有,按需选择。

---

## 练习题

1. 什么是鸭子类型?它和基于继承的多态有什么不同?

<details><summary>参考答案</summary>

**鸭子类型(duck typing)**是 Python 的多态哲学,名字来自"如果一个东西走起来像鸭子、叫起来像鸭子,那它就是鸭子"——意思是:**判断一个对象能否被某段代码使用,不取决于它的类型(是哪个类、继承自谁),而只取决于它是否具备所需的行为(方法/属性)**。例如一个函数 `sum(len(x) for x in items)` 能处理任何"元素支持 `len()`"的可迭代对象,无论里面是字符串、列表、集合还是自定义类,它们之间没有任何继承关系,但都"支持 len()",所以都能用。`for`(需要可迭代协议)、`with`(需要上下文协议)、`obj[i]`(需要 `__getitem__`)等也都是鸭子类型——只看对象实现了对应的协议方法。**与基于继承的多态的不同**:传统(如 Java)的多态是**名义的(nominal)**——一个对象要被当作某接口/基类使用,必须**显式声明继承或实现**那个类型(`implements Comparable`),编译器据此检查;而鸭子类型是**结构的/隐式的**——不需要任何继承声明或类型标注,只要运行时对象碰巧有那些方法就能工作,完全靠"能力"而非"血统"。优点:极其灵活、解耦(函数不绑定具体类型,任何"长得对"的对象都能传入,便于扩展和测试 mock);缺点:接口是隐式的、不明确(光看签名不知道参数需要哪些方法),错误要到运行时才暴露,IDE/静态检查难帮忙——这正是后来引入 ABC 和 Protocol(尤其 Protocol 是"静态鸭子类型")来弥补的:在保留灵活性的同时,让接口可被显式表达和(静态)检查。

</details>

2. 抽象基类(ABC)是做什么的?含抽象方法的类为什么不能实例化?

<details><summary>参考答案</summary>

**抽象基类(ABC,Abstract Base Class)**用 `abc` 模块定义,用来**声明一个"接口契约"——规定子类必须实现哪些方法**,从而强制一组相关类遵守统一的接口。做法:让类继承 `abc.ABC`(其元类是 `ABCMeta`),并用 `@abstractmethod` 装饰那些"子类必须实现"的方法(可以只有声明、没有实现体)。例如 `class Storage(ABC): @abstractmethod def save(self, ...): ...`。**含(未被实现的)抽象方法的类不能实例化的原因**:`ABCMeta` 会跟踪一个类里还有哪些抽象方法没有被实现;当你试图实例化一个仍存在未实现抽象方法的类时,在 `__call__`/`__new__` 层面会**直接抛 `TypeError: Can't instantiate abstract class ... with abstract methods ...`**。这是**有意的设计**:它把"忘了实现接口要求的方法"这种错误**从"调用到该方法时才在运行时崩溃"提前到了"创建对象时就报错"**,更早、更明确地暴露问题。只有当一个子类**实现了基类所有的抽象方法**后,它才"具体化"、可以被实例化。这样 ABC 既定义了契约(必须实现 save/load 等),又能在实例化时强制校验。补充:①ABC 还可以包含**非抽象的具体方法**(提供默认实现,子类可直接复用或重写),所以它兼具"接口约束 + 实现复用"的能力;②ABC 是**名义子类型**——一个类要被认作某 ABC 的子类,要么显式继承它,要么用 `ABC.register(cls)` 注册为"虚拟子类"(但 register 只让 `isinstance` 通过、不强制也不检查实现,需谨慎);③`collections.abc` 提供了一批现成 ABC(Sequence、Mapping 等),实现少量核心方法即可获得一整套混入方法。

</details>

3. `collections.abc` 有什么用?为什么继承 `Sequence` 只实现两个方法就能用一堆功能?

<details><summary>参考答案</summary>

`collections.abc` 提供了一组**描述容器/可调用等"能力"的抽象基类**,如 `Iterable`、`Iterator`、`Container`、`Sized`、`Sequence`/`MutableSequence`、`Mapping`/`MutableMapping`、`Set`、`Hashable`、`Callable` 等。它有两大用途:①**作为类型判断的依据**——`isinstance(x, Iterable)`、`isinstance(x, Mapping)` 可以基于"是否具备相应协议方法"判断对象属于哪类容器(这些 ABC 重写了 `__subclasshook__`,会检查结构);②**作为实现自定义容器的基类(混入)**——这是最实用的:这些 ABC 采用了"**少量抽象方法 + 大量基于它们的具体混入方法**"的设计。以 `Sequence` 为例,它只要求子类实现两个**抽象方法**:`__getitem__`(按索引取元素)和 `__len__`(长度);一旦你实现了这两个,`Sequence` 这个 ABC 就用它们**自动为你提供了一整套派生方法的默认实现**——`__contains__`(`in` 判断,基于遍历 `__getitem__`)、`__iter__`(迭代,基于 `__getitem__` 从 0 取到越界)、`__reversed__`、`index()`、`count()` 等。也就是说,这些通用逻辑都可以**仅依赖 `__getitem__` 和 `__len__`** 推导出来,ABC 已经替你写好了,你继承它就"免费"获得,无需自己重复实现。同理 `MutableMapping` 只需实现 `__getitem__/__setitem__/__delitem__/__iter__/__len__` 五个,就能获得 `get/pop/update/items/keys/values/setdefault` 等一大堆方法。好处是:大幅减少自定义容器的样板代码、保证行为与内置容器一致、并让你的类被 `isinstance(x, Sequence)` 正确识别。这是"模板方法模式 + ABC"的优雅应用。

</details>

4. `typing.Protocol` 是什么?它和 ABC 有什么本质区别?

<details><summary>参考答案</summary>

`typing.Protocol`(PEP 544,Python 3.8)用于定义**协议**——一种基于"结构"的接口,把鸭子类型带入**静态类型检查**。你定义一个继承 `Protocol` 的类,在里面声明一些方法/属性签名(只签名、无实现),它就表示"任何具备这些方法/属性的对象"。关键在于它是**结构化子类型(structural subtyping)**:**一个类无需显式继承这个 Protocol,只要它"长得像"(实现了协议要求的全部方法/属性,且签名兼容),静态类型检查器(mypy/Pyright)就认为它满足该协议**,可以传给标注为该协议类型的参数。例如定义 `class Closable(Protocol): def close(self)->None: ...`,任何有 `close()` 方法的类(哪怕完全没继承 Closable)都能传给 `def f(x: Closable)`,类型检查通过。**与 ABC 的本质区别**在于"名义 vs 结构":①**ABC 是名义子类型(nominal)**——一个类要被认作某 ABC 的实例,**必须显式继承它或用 `register` 注册**,是"声明出来的"关系;②**Protocol 是结构子类型(structural)**——只看对象**实际有没有**那些方法,"长得像就算",是"碰巧具备"的关系,不需要任何继承或声明(更贴合鸭子类型的精神)。其他差异:Protocol 主要服务于**静态检查时刻**(运行时默认不能用 `isinstance` 判断,除非给 Protocol 加 `@runtime_checkable`,且那样也只检查方法名存在、不检查签名);ABC 在**运行时**就强制约束(不实现抽象方法不能实例化)、还能提供共享的具体实现。**选择**:想要"灵活、解耦、可被任意'长得对'的类满足、又能享受类型检查/IDE 补全"的接口(尤其库的公共 API、依赖注入)用 **Protocol**;想要"强制子类实现 + 复用部分默认实现 + 运行时就拦截"用 **ABC**。值得一提:Go 的接口就是结构化的(类似 Protocol,无需声明实现),而 Java 的 interface 是名义的(类似 ABC,要 implements)。

</details>

5. 为什么过度使用 `isinstance` 检查被认为不够 Pythonic?有更好的方式吗?

<details><summary>参考答案</summary>

因为**过度的 `isinstance` 类型检查违背了鸭子类型的精神,削弱了灵活性和可扩展性**。鸭子类型主张"只要对象具备所需能力就能用,不关心它具体是什么类型";而到处写 `if isinstance(x, list): ...` 实际上是把函数**硬绑定到具体的类型**上——这样一来,任何"行为正确但类型不在白名单里"的对象(比如另一个支持 append 的自定义序列、或 tuple、或某个鸭子类型的 mock 对象)都会被拒之门外,函数失去了对未来新类型的开放性,也更难测试(无法用简单的替身对象替换)。这本质上是把"是不是某类型"当成了"能不能做某事"的前提,过度耦合。**更好的方式**:①**直接尝试 + EAFP**(见 P08)——直接调用所需的方法,出错(`AttributeError`/`TypeError`)再处理,让任何"支持该操作"的对象都能工作,这是最 Pythonic 的;②**用协议/抽象基类表达"能力"而非"具体类型"**——如果确实需要判断,检查 `isinstance(x, collections.abc.Iterable)`(判断"是否可迭代"这种能力)比 `isinstance(x, list)`(判断具体类型)好得多,因为前者对所有可迭代对象开放;③**用 `typing.Protocol` + 静态类型检查**——在类型注解层面表达"需要具备 close()/read() 等方法",既保留运行时的鸭子灵活性、又能让 mypy/IDE 静态验证接口(必要时配 `@runtime_checkable` 做基于能力的 isinstance);④用多态/重载(`functools.singledispatch` 按类型分派)来代替一长串 isinstance 分支。**合理使用 isinstance 的场景**也存在:如处理 `int` vs `str` 这种需要真正区分基本类型的逻辑、解析异构输入、或对外部不可信数据做防御性检查时,isinstance 是恰当的。原则是:**优先按"能力"编程(鸭子类型/协议/EAFP),把 isinstance 留给确实需要按具体类型分流的场合,且尽量检查抽象能力(abc)而非具体类。**

</details>

6. 鸭子类型、ABC、Protocol 三者在实际项目中怎么选?

<details><summary>参考答案</summary>

按"是否需要显式约束/静态检查、是否需要复用实现"来选:①**纯鸭子类型(直接用 + EAFP)**——当代码是内部的、动态的,不需要对外暴露明确接口、也不追求静态类型检查时,直接依赖对象具备所需方法即可(`for`、`len()`、`x.read()`),最灵活、最简洁,适合脚本、内部辅助函数、以及大量"反正能用就行"的场景。②**抽象基类 ABC(继承 + `@abstractmethod`)**——当你在设计一个**框架基类**,需要**强制子类实现某些方法(契约)**,并且希望**同时提供一部分默认/共享实现**给子类复用,以及希望在**运行时**就阻止"没实现完整接口"的类被实例化时,用 ABC。典型如 `BaseHandler`、`Storage`、插件基类;`collections.abc` 也属此类(自定义容器继承 `Sequence`/`Mapping` 白嫖方法)。它是名义的(子类要继承/注册)。③**Protocol(结构化协议)**——当你想为**库的公共 API / 可替换组件 / 依赖注入**定义接口,**既要保留鸭子类型的灵活(调用方传任何'长得像'的对象,无需继承你的类型,降低耦合)、又要享受静态类型检查和 IDE 支持**时,用 Protocol。这是现代 Python 表达接口的首选,尤其适合"我需要一个有 read() 方法的东西"这类描述能力的参数标注。需要运行时 isinstance 检查时加 `@runtime_checkable`。④**组合使用**也常见:用 Protocol 标注函数参数(对外灵活)、用 ABC 实现内部的具体基类层次、用 `collections.abc` 做容器。一句话指引:**内部动态逻辑→鸭子类型;要强约束+共享实现的基类→ABC;要灵活又可静态检查的公共接口→Protocol;自定义容器→collections.abc。** 总体现代趋势是多用 Protocol 表达接口(配合类型注解),减少不必要的继承耦合。

</details>
