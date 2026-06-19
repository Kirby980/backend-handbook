# 精通 Python 继承与 MRO

> Python 支持多继承,随之而来的是"方法到底调哪个"的问题——靠 **MRO(方法解析顺序)**和 **C3 线性化**解决。面试常问:"super() 是调父类吗?""菱形继承怎么办?"本篇讲清继承、MRO 与协作式 `super`,依赖 [P09 属性查找](./P09-精通-Python-类与属性查找.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、单继承与方法重写

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        return "..."

class Dog(Animal):
    def speak(self):                 # 重写父类方法
        return "汪"
    def fetch(self):                 # 新增方法
        return f"{self.name} 捡球"
```

子类继承父类的属性和方法,可**重写(override)**同名方法。所有类隐式继承自 **`object`**(Python 3)。查找方法时沿继承链向上找(见 [P09](./P09-精通-Python-类与属性查找.md))。

---

## 二、多继承与菱形问题

Python 允许多继承 `class C(A, B)`。但多继承会遇到**菱形继承(diamond)**:

```
      A
     / \
    B   C
     \ /
      D          # D(B, C),B 和 C 都继承 A
```

问题:`D` 调用某方法时,如果 B、C 都重写了它,应该用谁?A 的 `__init__` 会不会被调用两次?——这需要一个明确的、一致的"方法解析顺序"。

---

## 三、C3 线性化与 `__mro__`

Python 用 **C3 线性化算法**为每个类计算一个唯一的、确定的 **MRO(Method Resolution Order,方法解析顺序)**——把所有祖先类排成一个线性序列。属性/方法查找就沿这个序列从前往后找,**找到即停**。

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass
print([c.__name__ for c in D.__mro__])
# ['D', 'B', 'C', 'A', 'object']
```

C3 保证:①**子类排在父类前**;②**保留每个基类声明的相对顺序**(B 在 C 前);③**单调性**(子类的 MRO 与父类 MRO 一致,不矛盾)。菱形里 **A 只出现一次、且排在 B/C 之后**——这就解决了"A 被处理两次"的问题。若继承关系无法满足 C3 约束,定义类时直接报 `TypeError`。

---

## 四、super() 的真正含义

最大的误解:**`super()` 不是"调用父类",而是"调用 MRO 中的下一个类"**。

```python
class B(A):
    def m(self):
        super().m()      # 不是"调 A.m",而是"调 MRO 里 B 之后的下一个的 m"
```

在单继承里二者恰好一致(下一个就是父类),所以容易误解。但在多继承里,`super()` 调的是**当前实例的 MRO 中、当前类的下一个**——这个"下一个"取决于**最终实例的类型的 MRO**,而非静态的父类关系。正是这个特性让多继承能**协作**起来(见下)。

`super()`(3 中无参)等价 `super(当前类, self)`,自动用实例的 MRO。

---

## 五、协作式多继承

要让多继承正确工作(每个类的逻辑都执行、`object` 只初始化一次),所有相关类都应在方法里用 `super()` 链式调用,形成**协作式继承**:

```python
class Base:
    def __init__(self, **kw):
        super().__init__(**kw)       # 顶到 object

class LoggingMixin(Base):
    def __init__(self, **kw):
        print("log init")
        super().__init__(**kw)       # 沿 MRO 传递,不写死父类名

class Service(LoggingMixin, Base):
    def __init__(self, name, **kw):
        self.name = name
        super().__init__(**kw)
# Service(...) 会沿 MRO 依次执行每个类的 __init__,各调一次
```

要点:①每个类都 `super().__init__(...)` 把调用沿 MRO 传下去,而不是 `Base.__init__(self)` 写死(写死会破坏协作、可能重复或漏调);②多继承 `__init__` 参数难协调,常用 `**kwargs` 透传;③这套约定让 mixin 可自由组合。

---

## 六、Mixin 模式

**Mixin** 是一种只提供**某方面功能**、不单独实例化、被"混入"其他类的小类。多继承的主要正当用途就是组合 mixin:

```python
class JsonMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class TimestampMixin:
    def touch(self):
        self.updated_at = time.time()

class User(JsonMixin, TimestampMixin):   # 组合多个能力
    def __init__(self, name):
        self.name = name
```

约定:mixin 名字常带 `Mixin` 后缀、放在基类列表**靠前**(让其方法优先)、不自己 `__init__` 大量状态、聚焦单一职责。Django/DRF 的视图大量用 mixin。

---

## 陷阱清单

- **以为 `super()` 是"调父类"**:它是"调 MRO 下一个";多继承下"下一个"由实例 MRO 决定,可能不是直接父类。
- **多继承里写死 `Base.__init__(self)`**:破坏协作链,导致某些类 `__init__` 重复调用或被跳过。统一用 `super()`。
- **多继承 `__init__` 参数不一致**:协作链上参数对不上会报错;用 `**kwargs` 透传。
- **菱形继承不理解 MRO**:以为基类被初始化多次;实际 C3 保证公共祖先只在 MRO 出现一次。
- **基类顺序写反**:`class D(C, B)` 与 `class D(B, C)` 的 MRO 不同,方法优先级不同。
- **无法满足 C3 的继承结构**:定义时直接 `TypeError`(MRO 冲突)。
- **滥用多继承**:继承层次复杂难懂;优先组合/委托,多继承只用于清晰的 mixin。

---

## 2026 现状

- **mixin + 协作式 super** 是多继承的主流正当用法;复杂能力组合更推荐**组合(composition)**而非深继承。
- **`__init_subclass__` 和 `Protocol`**(见 [P12](./P12-精通-Python-元类与类创建.md)/[P13](./P13-精通-Python-鸭子类型与协议.md))在很多场景替代了多继承/元类来做"约束子类""结构化类型"。
- **`collections.abc`** 提供大量可继承/混入的抽象基类(如 `Mapping`、`Sequence`),实现少量方法即获得一整套(见 [P13](./P13-精通-Python-鸭子类型与协议.md))。
- 与 Java 对照([J10](../java/INDEX.md)):Java 单继承类 + 多实现接口(默认方法),Python 是真多继承靠 MRO;Python 的 mixin ≈ Java 带默认方法的接口,但 Python 更灵活也更易乱。

---

## 练习题

1. 什么是 MRO?C3 线性化解决了什么问题?

<details><summary>参考答案</summary>

**MRO(Method Resolution Order,方法解析顺序)**是 Python 为每个类计算出的、用于属性/方法查找的**线性化的祖先类序列**——当访问 `obj.attr` 或调用方法时,Python 沿着这个序列从前往后查找,**找到第一个即停**。可以用 `ClassName.__mro__` 或 `ClassName.mro()` 查看。在**多继承**(尤其菱形继承)下,一个类有多条到达祖先的路径,"该用哪个基类的方法""公共祖先要不要被处理多次"会产生歧义,MRO 就是用来消除这种歧义、给出唯一确定的查找顺序。Python 用 **C3 线性化算法**计算 MRO,它保证三条性质:①**子类总是排在其父类之前**(局部优先);②**保留基类在声明列表中的相对顺序**(如 `class D(B, C)` 中 B 排在 C 前);③**单调性**——子类的 MRO 是与其各父类的 MRO 相容(不矛盾)的,即父类中确定的先后关系在子类 MRO 中依然成立。C3 解决的核心问题是**菱形继承**:对于 `A` 被 `B`、`C` 共同继承、`D(B, C)` 的结构,C3 保证 **公共祖先 A 在 MRO 中只出现一次、且排在 B 和 C 的后面**(MRO 为 `[D, B, C, A, object]`),从而既不会重复处理 A、又能让 B 和 C 各自的逻辑都在 A 之前执行。如果某个继承结构无法满足 C3 的约束(产生顺序矛盾),Python 会在**定义类时直接抛 `TypeError`**(MRO 冲突),而不是留到运行时出错。

</details>

2. `super()` 真的是"调用父类方法"吗?在多继承里它调的是谁?

<details><summary>参考答案</summary>

**不准确。`super()` 调用的不是"父类",而是"当前实例的 MRO 中,当前类之后的下一个类"**。在**单继承**里,MRO 中当前类的下一个恰好就是它的直接父类,所以 `super().method()` 看起来就是"调父类的 method",这造成了普遍的误解。但在**多继承**里,这个"下一个"是由**最终实例的类型的 MRO** 动态决定的,可能并不是当前类在源码里写的某个直接基类。举例:`class A`、`class B(A)`、`class C(A)`、`class D(B, C)`,其 MRO 是 `[D, B, C, A, object]`。当你创建 `D()` 的实例并在 `B` 的方法里调用 `super().m()` 时,它调用的是 **MRO 中 B 之后的 C**(而不是 B 的"父类" A!)——因为对 D 的实例而言,B 的下一个是 C。正是这个"沿 MRO 走下一个"的语义,使得多继承可以**协作(cooperative)**:每个类用 `super()` 把调用接力传给 MRO 的下一个类,从而让所有相关类的逻辑都按 MRO 顺序各执行一次、公共祖先(如 A、object)只执行一次。`super()`(Python 3 无参形式)等价于 `super(__class__, self)`,它绑定了当前类和实例,从而能查实例的 MRO 并定位"下一个"。理解这点很关键:在协作式多继承中**绝不能**把 `super().__init__()` 替换成写死的 `BaseClass.__init__(self)`,否则会破坏 MRO 接力链,导致某些类被跳过或被调用多次。

</details>

3. 在多继承中正确写 `__init__` 应该注意什么?

<details><summary>参考答案</summary>

核心原则是**协作式继承(cooperative multiple inheritance)**:让继承体系中**每一个**类的 `__init__` 都通过 **`super().__init__(...)`**(而非写死某个基类名)把初始化调用沿 **MRO** 接力传递下去,这样所有相关类的 `__init__` 都会按 MRO 顺序**各被调用恰好一次**,公共祖先(最终是 `object`)也只初始化一次。注意要点:①**统一用 `super().__init__(...)`,不要用 `BaseClass.__init__(self, ...)` 写死**——后者会绕过 MRO 接力,在菱形结构下可能导致某个类被跳过(漏初始化)或被调用两次(重复初始化),还会破坏其他类的协作;②**参数协调困难**——多继承时不同类的 `__init__` 需要不同参数,而 `super().__init__` 是沿 MRO 串行传递的,参数难以静态对齐;常用的解决办法是**让各类接受并透传 `**kwargs`**(每个类取走自己需要的关键字参数、把其余 `**kwargs` 继续 `super().__init__(**kwargs)` 传下去),最终 `object.__init__()` 不接收多余参数(所以要确保传到 object 时 kwargs 已为空);③**mixin 通常放在基类列表靠前**,且 mixin 应尽量少持有/初始化状态;④理解此时 `super()` 走的是**实例的 MRO**,所以同一个类在不同的最终子类中,`super()` 的"下一个"可能不同——代码要写得对 MRO 透明(不假设下一个具体是谁)。如果继承关系复杂到 `__init__` 协调很痛苦,往往说明应该改用**组合**而非多继承。

</details>

4. 什么是 Mixin?它和普通多继承有什么不同?

<details><summary>参考答案</summary>

**Mixin(混入类)**是一种特殊的、轻量的类,它**只提供某一方面的、可复用的功能(一组方法)**,本身**不打算被单独实例化**、通常也不维护(大量)自己的实例状态,而是被"混入"(通过多继承)到其他类中,给那些类**添加能力**。例如一个 `JsonMixin` 提供 `to_json()` 方法、一个 `LoggingMixin` 提供日志能力,任何类继承它们就获得对应功能。它与"普通多继承"的区别更多是**意图和约定上的**而非语法上的(Python 没有专门的 mixin 关键字,它就是用多继承实现):①**职责单一**——mixin 聚焦于一个横切功能,而不是表示一个"是什么"的实体;②**不独立存在**——mixin 一般依赖宿主类提供某些属性/方法(如 `JsonMixin.to_json` 依赖宿主的 `__dict__`),自己不能独立成对象;③**用于"提供能力(has-a-capability)"而非"是一种(is-a)"**——这区别于经典的"子类是父类的一种"的继承语义;④**组合性**——可以把多个 mixin 自由组合到一个类上(`class User(JsonMixin, TimestampMixin, Base)`)。约定俗成:mixin 名字常以 `Mixin` 结尾、在基类列表中**放在靠前位置**(使其方法在 MRO 中优先、能覆盖/增强后面的基类)、尽量不写复杂的 `__init__`。框架里大量使用(如 Django 的类视图、DRF)。相比之下,把多继承用于组合多个"实体"父类(都带状态和 `__init__`)往往会带来 MRO/初始化的复杂性,容易出错——所以 Python 社区的共识是:**多继承主要用于混入 mixin 这种清晰的能力组合,其他情况优先用组合(composition)/委托**。

</details>

5. `class D(B, C)` 和 `class D(C, B)` 有区别吗?

<details><summary>参考答案</summary>

**有区别**——基类的书写顺序会直接影响 D 的 **MRO**,进而影响**方法解析的优先级**。C3 线性化会保留基类在声明列表中的相对顺序:`class D(B, C)` 的 MRO 大致是 `[D, B, C, ..., object]`(B 在 C 前),而 `class D(C, B)` 的 MRO 是 `[D, C, B, ..., object]`(C 在 B 前)。由于属性/方法查找沿 MRO 从前往后、找到即停,所以:如果 B 和 C 都定义了同名方法 `m`,那么 `class D(B, C)` 的实例调用 `d.m()` 会用 **B 的 `m`**(B 在 MRO 中更靠前),而 `class D(C, B)` 会用 **C 的 `m`**。同理,在协作式继承里 `super()` 接力的"下一个"也随 MRO 顺序改变,初始化/方法的执行顺序都会不同。这正是为什么 **mixin 通常放在基类列表的前面**——让 mixin 的方法在 MRO 中优先,从而能覆盖或增强后面"主基类"的行为。所以基类顺序不是随意的:它表达了"当有冲突时,优先用谁"的意图,写反可能导致调用了非预期的实现。如果两种顺序会导致 C3 无法构造出一致的线性化(顺序矛盾),Python 还会在定义类时直接报 `TypeError`。实践中应有意识地安排基类顺序(更具体/更高优先的放前面),并可用 `D.__mro__` 验证解析顺序是否符合预期。

</details>

6. Python 的多继承和 Java 的单继承+接口相比,有什么取舍?

<details><summary>参考答案</summary>

**Java**:类只能**单继承**(一个直接父类),但可以**实现多个接口**;接口主要声明方法契约(Java 8 起接口可有 `default` 默认方法,带来了有限的"多继承行为"能力,但接口仍不能有实例状态/字段)。这种设计**避免了状态的菱形继承问题**(因为接口无实例字段),类型关系清晰,代价是复用具体实现要靠组合或默认方法,灵活性较低。**Python**:支持**真正的多继承**——一个类可以继承多个父类,且父类都可以带实现和状态;靠 **MRO(C3 线性化)** 来确定方法解析顺序、靠**协作式 super** 来让多个父类的逻辑有序协作、公共祖先只处理一次。**取舍对比**:①**灵活性**:Python 多继承更灵活,能直接复用多个类的实现(mixin 组合能力强),无需像 Java 那样靠组合/委托绕路;②**复杂性/风险**:Python 的灵活也带来风险——菱形继承下的状态初始化、`super` 接力、MRO 顺序都需要正确理解,写不好会出现重复/遗漏初始化、难以追踪方法来源等问题;Java 的限制反而强制了更简单清晰的类型层次;③**状态 vs 行为**:Java 用"单继承传状态 + 多接口传契约/默认行为"区分得很干净;Python 不强制区分,mixin 可以带状态(但最佳实践是 mixin 少带状态);④**对应关系**:Python 的 mixin ≈ Java 带 default 方法的接口,但 Python mixin 能持有状态、组合更自由。**共识**:两者都建议**优先用组合而非深继承**;Python 中多继承应主要用于清晰的 mixin 能力组合,避免用多继承拼装多个"实体"父类。理解差异能避免带着 Java 的单继承惯性去用(或误用)Python 多继承。

</details>
