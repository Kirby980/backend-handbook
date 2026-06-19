# 精通 Python 元类与类创建

> 元类是 Python "类也是对象"哲学的终极体现——类由元类创建,正如对象由类创建。它强大但少用,常被神化。面试问:"type 的两个用法?""元类干嘛的?"本篇讲清类的创建机制与元类,并说明现代多数场景用更轻的 `__init_subclass__` 替代。依赖 [P09](./P09-精通-Python-类与属性查找.md)、[P11](./P11-精通-Python-描述符与property.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、type 的双重身份

`type` 有两个用法:

```python
type(42)                          # ① 查询对象的类型 → <class 'int'>
MyClass = type("MyClass", (Base,), {"x": 1, "m": lambda self: 42})  # ② 动态创建类!
```

第二种 `type(name, bases, namespace)` **动态创建一个类**:名字、基类元组、属性/方法字典。这揭示了一个事实:**`class` 语句本质就是调用 `type` 来创建类对象**。

```python
class Foo:        # 等价于 ↓
    x = 1
Foo = type("Foo", (), {"x": 1})
```

---

## 二、类也是对象

Python 里**类本身也是对象**——它是 **`type` 的实例**:

```python
class Dog: pass
print(type(Dog))         # <class 'type'> —— Dog 是 type 的实例
print(type(type))        # <class 'type'> —— type 是自己的实例(终点)
```

- 对象 ← 由 → 类创建(对象是类的实例)
- 类 ← 由 → **元类(metaclass)**创建(类是元类的实例)
- 默认元类就是 **`type`**

既然类是对象,就能赋值、传参、动态创建、动态加属性——这是装饰器、注册表、ORM 等的基础。

---

## 三、自定义元类

**元类是"类的类"**——控制类的创建过程。继承 `type` 自定义:

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace, **kw):
        # 在类被创建【之前】介入:可检查/修改 namespace
        print(f"创建类 {name}")
        namespace["created_by"] = "Meta"
        return super().__new__(mcs, name, bases, namespace)

class MyClass(metaclass=Meta):        # 指定元类
    pass
# 输出:创建类 MyClass
print(MyClass.created_by)             # "Meta"
```

- `metaclass=Meta` 让 `MyClass` 由 `Meta`(而非默认 `type`)创建。
- `Meta.__new__` 在**类创建时**调用(注意是创建"类"这个对象,不是实例),能审查/改写类的成员、基类、名字。
- `Meta.__init__` 在类创建后初始化它;`Meta.__call__` 控制"用这个类创建实例"的过程(如单例)。

---

## 四、元类的用途

元类适合"对一类(及其所有子类)统一施加规则":

- **注册表**:类定义时自动注册到某处(插件系统、序列化器、命令分发)。
- **校验**:强制子类实现某些方法/遵守某约定,定义时就报错(而非运行时)。
- **改写类**:自动给类加方法/属性、包装方法、注入元数据。
- **ORM/框架**:Django Model、SQLAlchemy(部分)、旧版库用元类把类属性(字段)转成数据库映射。

```python
class RegistryMeta(type):
    registry = {}
    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        if bases:                       # 跳过基类自身
            RegistryMeta.registry[name] = cls   # 子类定义即注册
        return cls
```

---

## 五、`__init_subclass__`:轻量替代

Python 3.6 引入 **`__init_subclass__`**,在**定义子类时**自动调用父类的这个钩子——**很多原来要用元类的场景,现在用它更简单**(无需理解元类):

```python
class Plugin:
    registry = []
    def __init_subclass__(cls, **kwargs):   # 每个子类定义时触发
        super().__init_subclass__(**kwargs)
        Plugin.registry.append(cls)         # 自动注册子类

class A(Plugin): pass     # 定义即被注册
class B(Plugin): pass
print(Plugin.registry)    # [A, B]
```

配套的 **`__set_name__`**(见 [P11](./P11-精通-Python-描述符与property.md))让描述符自动获知属性名。**子类注册、子类校验、描述符命名**这三类常见需求,优先用 `__init_subclass__`/`__set_name__`,而非元类。

---

## 六、何时(不)用元类

**Tim Peters 名言**:"元类是 99% 用户都不需要操心的深魔法。如果你拿不准要不要用,那就不用。"

- **绝大多数情况不需要元类**——能用类装饰器、`__init_subclass__`、描述符、普通继承解决的,就别上元类。
- 元类的代价:难理解、难调试、**多继承时元类必须兼容**(否则 metaclass 冲突 `TypeError`)、和其他元类(如 `ABCMeta`)组合复杂。
- 真正需要元类的:框架级"对类创建做深度定制且子类钩子不够用"的场景(且通常是库作者,不是应用开发者)。

---

## 陷阱清单

- **为了炫技用元类**:增加理解成本;能用 `__init_subclass__`/类装饰器/描述符就别用元类。
- **元类冲突**:多继承的父类元类不兼容会 `TypeError: metaclass conflict`;混用 `ABCMeta` 等要让自定义元类继承它。
- **`__new__` vs `__init__` 混淆**:元类的 `__new__` 创建并返回类对象(可改 namespace),`__init__` 在类创建后初始化;实例层面的 `__new__`/`__init__` 是另一回事。
- **以为 `type(x)` 只能查类型**:它还能三参数动态建类。
- **元类影响所有子类**:元类会被子类继承,作用范围大,慎改。
- **调试困难**:元类逻辑在"类定义时"运行,错误栈不直观。

---

## 2026 现状

- **`__init_subclass__` + `__set_name__` + 类装饰器** 已覆盖过去大量元类用例,现代代码很少手写元类。
- **`ABCMeta`**(抽象基类,见 [P13](./P13-精通-Python-鸭子类型与协议.md))是最常"被动接触"到的元类;`Enum`、Django Model 等框架内部用元类,但用户无需自己写。
- **`dataclass`/Pydantic/attrs** 用类装饰器(而非元类)处理类,更易组合。
- 与 Java 对照:元类近似"在类加载时改写类"(类似 Java 的字节码增强/APT 注解处理,见 [J06](../java/INDEX.md)),但 Python 是运行时、纯 Python 可写,更动态也更易滥用。

---

## 练习题

1. `type` 有哪两种用法?这说明了 Python 的什么设计?

<details><summary>参考答案</summary>

`type` 有两种用法:①**单参数 `type(obj)`**——返回对象的**类型(它是哪个类的实例)**,如 `type(42)` 返回 `<class 'int'>`、`type("a")` 返回 `<class 'str'>`,这是日常查类型的用法。②**三参数 `type(name, bases, namespace)`**——**动态创建一个新的类对象**:`name` 是类名(字符串)、`bases` 是基类元组、`namespace` 是包含类属性和方法的字典。例如 `MyClass = type("MyClass", (Base,), {"x": 1, "greet": lambda self: "hi"})` 等价于用 `class MyClass(Base): x = 1; def greet(self): return "hi"` 定义。这两种用法揭示了 Python 的核心设计:**`type` 既是"所有类型的类型",又是默认的"类工厂(元类)"**。更深层的说明是——**`class` 语句本质上就是调用 `type`(或指定的元类)来创建一个类对象**:解释器执行 `class` 块时,先执行类体收集出 namespace 字典,然后调用 `type(类名, 基类, namespace)` 创建类。这体现了 Python "**一切皆对象,类本身也是对象**"的统一设计——类是 `type` 的实例,可以在运行时动态创建、传递、修改。正是这种"类是一等对象"的特性,支撑了装饰器装饰类、动态生成类、注册表、ORM 等元编程能力。

</details>

2. "类也是对象"是什么意思?元类是什么?

<details><summary>参考答案</summary>

"类也是对象"意思是:在 Python 中,用 `class` 定义出来的**类本身也是一个运行时对象**(而不只是编译期的蓝图)——它可以被赋值给变量、作为参数传递、作为返回值、存进容器、动态地添加属性、甚至在运行时被动态创建。既然每个对象都有它的"类型"(由某个类创建),那么类这个对象的类型是什么?答案是:**类是"元类(metaclass)"的实例**。默认情况下,所有类的元类是内置的 **`type`**——即 `type(MyClass)` 返回 `<class 'type'>`,说明 `MyClass` 是 `type` 的实例。于是形成一条链:**普通对象 是 类 的实例 → 类 是 元类(type)的实例 → type 是它自己的实例(终点)**。**元类就是"类的类"**——它控制**类是如何被创建的**,正如类控制实例是如何被创建的。当你写 `class Foo: ...` 时,解释器实际上是调用元类(默认 `type`)来构造出 `Foo` 这个类对象。你可以通过 `class Foo(metaclass=MyMeta)` 指定自定义元类(`MyMeta` 需继承 `type`),从而在**类被创建的过程中介入**:`MyMeta.__new__`/`__init__` 在类创建时被调用,可以检查、修改类的属性/方法/基类,或做注册、校验等。简言之:对象由类造,类由元类造,默认元类是 type;"类是对象"使得元编程(运行时操纵类)成为可能。

</details>

3. 自定义元类能做什么?举一个典型用途。

<details><summary>参考答案</summary>

自定义元类(继承 `type`)能**在"类被创建的那一刻"介入并定制类的构造过程**,因此适合对"一整类(及其所有子类)"统一施加规则或处理。典型能力/用途:①**自动注册(注册表模式)**——在元类的 `__new__`/`__init__` 里,把每个被创建的类自动登记到某个全局注册表中,用于插件系统、命令分发、序列化器查找等(子类一旦定义就自动注册,无需手动);②**强制约束/校验子类**——在类创建时检查它是否实现了必需的方法、是否符合命名/结构约定,不符合就**在定义时直接抛错**(把错误从运行时提前到定义时);③**改写/增强类**——自动给类添加方法、属性、包装已有方法、注入元数据、修改类的 `__dict__`;④**控制实例化**——通过元类的 `__call__` 拦截"用类创建实例"的过程,实现单例、缓存实例等;⑤**框架级映射**——ORM(如 Django Model、旧版库)用元类把类里声明的字段属性转换成数据库列的映射、生成表结构等。典型例子(注册表):
```python
class RegistryMeta(type):
    registry = {}
    def __new__(mcs, name, bases, ns):
        cls = super().__new__(mcs, name, bases, ns)
        if bases:                      # 跳过基类本身
            RegistryMeta.registry[name] = cls
        return cls
class Handler(metaclass=RegistryMeta): ...
class FooHandler(Handler): ...         # 定义即被自动注册到 registry
```
这样所有 Handler 子类一经定义就进了 `registry`,可按名字查找分发。不过要强调:这类需求现在**大多可以用更轻量的 `__init_subclass__`(子类注册/校验)或类装饰器实现**,不必动用元类。

</details>

4. `__init_subclass__` 是什么?为什么说它能替代很多元类用例?

<details><summary>参考答案</summary>

`__init_subclass__` 是 Python 3.6 引入的一个**类方法钩子**:在一个类里定义它(它隐式是 classmethod),那么**每当有子类被定义时,这个钩子就会被自动调用一次**,参数 `cls` 是新定义的那个子类(还可接收通过 `class Sub(Base, key=value)` 传入的关键字参数)。例如在基类 `Plugin` 里写 `def __init_subclass__(cls, **kw): super().__init_subclass__(**kw); Plugin.registry.append(cls)`,那么之后每定义一个 `class A(Plugin)`、`class B(Plugin)`,A、B 都会被自动加入 `Plugin.registry`。**为什么能替代很多元类用例**:过去要实现"在子类定义时自动注册""校验子类是否实现了某些方法""给子类统一加点东西"这类需求,通常需要写一个自定义元类(`class Meta(type): def __new__(...)`),而元类**晦涩难懂、容易出错、且有多继承时元类必须兼容(否则 metaclass 冲突)等麻烦**。`__init_subclass__` 用一个普通的钩子方法就覆盖了这些最常见的需求,**不需要理解元类、不引入元类冲突、可读性好得多**——它本质上是 Python 把"子类创建时介入"这个高频需求从元类里抽出来、做成了一个轻量入口。配套的还有 **`__set_name__`**(描述符自动获知自己被赋给的属性名,见 P11),解决了另一类常见的元编程需求。所以现代 Python 的建议是:**子类注册、子类约束校验等优先用 `__init_subclass__`;描述符命名用 `__set_name__`;改写类用类装饰器;只有这些都不够、需要对类创建做深度定制时才考虑元类**。这也是为什么应用层代码现在几乎不需要手写元类。

</details>

5. 什么时候应该用元类?什么时候不该用?

<details><summary>参考答案</summary>

**结论先行:绝大多数情况都不应该用元类**——正如 Tim Peters 的名言"元类是 99% 的用户都无需操心的深魔法,如果你拿不准是否需要它,那就是不需要"。**不该用的情况(即大多数情况)**:凡是能用更简单的手段达成的,都不要上元类——①需要在子类定义时注册/校验子类 → 用 `__init_subclass__`;②需要给描述符自动命名 → 用 `__set_name__`;③需要在类定义后改写/增强类(加方法、包装、注入)→ 用**类装饰器**(如 `@dataclass` 的方式);④需要约束接口 → 用抽象基类(ABC)或 `Protocol`(P13);⑤需要单例/实例缓存 → 用模块级单例、`functools.cache` 包装工厂函数等。这些都比元类易懂、易调试、少坑。**元类的代价**:难以理解和调试(逻辑在"类创建时"运行,错误栈不直观)、**会被所有子类继承(影响面大)**、**多继承时各父类的元类必须兼容**(否则报 `metaclass conflict`)、和 `ABCMeta`、`Enum` 等已有元类组合时复杂。**真正适合用元类的少数场景**:通常是**框架/库作者**需要对"类的创建过程"做 `__init_subclass__`/类装饰器都无法表达的深度定制时——例如需要在类创建的极早期改写基类列表、需要控制类对象本身的行为(给类加 `__call__` 之外更底层的定制)、或要构建像 ORM/Enum 那样对用户透明的声明式 DSL。即便如此,也应优先评估能否用组合钩子实现。一句话:**应用开发者基本用不到元类;先穷尽 `__init_subclass__`/类装饰器/描述符/ABC,只有库级深度定制才考虑元类,且越少越好。**

</details>

6. 元类的 `__new__` 和 `__init__` 有什么区别?(可对比实例层面)

<details><summary>参考答案</summary>

要分清两个层面。**在元类层面**(元类继承自 `type`):元类的 **`__new__(mcs, name, bases, namespace, **kw)`** 在**类对象被创建之前**调用,负责**实际创建并返回那个类对象**——此时可以检查和**修改 `namespace`(类体里收集到的属性/方法字典)、bases、name**,然后通常调用 `super().__new__(...)`(即 `type.__new__`)真正生成类。因为它能在类对象生成前改写其成员,所以"想增删/改写类的属性、基类"要在 `__new__` 里做。元类的 **`__init__(cls, name, bases, namespace, **kw)`** 在**类对象已经创建之后**调用,`cls` 就是刚造好的类,用于对这个已存在的类做**初始化/登记**(如注册到表、设置一些类级状态),但此时类的结构已定型(改 namespace 已无意义,应改类属性)。简言之:**`__new__` 造类(可改结构,必须返回类对象),`__init__` 配置已造好的类(不返回)**。此外元类还有 **`__call__`**,它在"**用这个类去创建实例**"(即 `MyClass(...)`)时被调用,可用来拦截实例化过程(实现单例等)。**对比实例层面**(普通类):类的 `__new__(cls, ...)` 负责**创建并返回实例对象**(分配内存,极少重写,通常用于不可变类型如 int/str/tuple 的定制或单例),`__init__(self, ...)` 在实例创建后**初始化它**(给 self 设属性,常写)。结构是一致的:`__new__` 负责"造出对象并返回",`__init__` 负责"初始化已造出的对象";只不过在元类里造的是"类对象",在普通类里造的是"实例对象"。一个常见误区是把元类的 `__new__` 和实例的 `__new__` 搞混——前者拦截"类的创建",后者拦截"实例的创建"。

</details>
