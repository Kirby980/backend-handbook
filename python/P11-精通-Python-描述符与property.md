# 精通 Python 描述符与 property

> 描述符是 Python 对象模型里最底层、也最被低估的机制——`property`、`@staticmethod`/`@classmethod`、**方法绑定 self**、`cached_property`、ORM 字段,全都建立在它之上。理解它,很多"魔法"就不再神秘。这是面试区分高级与中级的硬核题。依赖 [P09 属性查找](./P09-精通-Python-类与属性查找.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、描述符协议

**描述符**:一个**定义在类上**、实现了以下任一方法的对象,它能拦截对某属性的访问:

- **`__get__(self, obj, objtype)`**:读取属性时调用
- **`__set__(self, obj, value)`**:赋值属性时调用
- **`__delete__(self, obj)`**:删除属性时调用

```python
class Descriptor:
    def __get__(self, obj, objtype=None):
        print("get"); return obj.__dict__.get("_x")
    def __set__(self, obj, value):
        print("set"); obj.__dict__["_x"] = value

class C:
    x = Descriptor()          # 描述符必须作为【类属性】

c = C()
c.x = 10                      # 触发 Descriptor.__set__
print(c.x)                    # 触发 Descriptor.__get__
```

关键:**描述符必须放在类上(作类属性)才生效**——属性查找机制(见 [P09](./P09-精通-Python-类与属性查找.md))在类里发现属性是描述符时,会去调它的 `__get__/__set__`,而不是直接返回它。

---

## 二、数据描述符 vs 非数据描述符

这是理解优先级的关键:

- **数据描述符(data descriptor)**:定义了 `__set__` **或** `__delete__`(通常也有 `__get__`)。如 `property`。
- **非数据描述符(non-data descriptor)**:**只**定义了 `__get__`。如普通函数/方法。

**优先级(见 [P09](./P09-精通-Python-类与属性查找.md) 的查找顺序)**:

```
数据描述符  >  实例 __dict__  >  非数据描述符 / 类属性
```

- **数据描述符压过实例属性**:即使实例 `__dict__` 里有同名 key,`property` 也会被优先调用(这就是 property 能"拦截"赋值的原因)。
- **非数据描述符让位于实例属性**:实例 `__dict__` 里有同名就用实例的(这让 `cached_property` 能"第一次算、之后走实例缓存")。

这条优先级规则解释了 property、方法、cached_property 的全部行为差异。

---

## 三、property 是数据描述符

`@property` 把方法变成"看起来像属性"的访问——它本质是一个**数据描述符**:

```python
class Circle:
    def __init__(self, r): self._r = r
    @property
    def radius(self):              # 读 c.radius 时调用
        return self._r
    @radius.setter
    def radius(self, value):       # 写 c.radius = v 时调用
        if value < 0: raise ValueError("半径不能为负")
        self._r = value
    @property
    def area(self):                # 只读计算属性
        return 3.14159 * self._r ** 2
```

`property` 同时定义了 `__get__`/`__set__`/`__delete__`,所以是数据描述符——它能拦截读写,即使实例 `__dict__` 有同名也压过。用途:**给属性加校验/计算/只读**,而对外接口仍是简单的 `c.radius`(无需改调用方代码,符合"统一访问原则")。

---

## 四、函数是非数据描述符——方法绑定的本质

最深刻的一点:**普通函数对象实现了 `__get__`(只有 `__get__`,所以是非数据描述符)**,这正是"方法自动绑定 self"的底层机制(见 [P09](./P09-精通-Python-类与属性查找.md))。

```python
class C:
    def greet(self): return "hi"
c = C()
# c.greet 触发 函数.__get__(c, C),返回一个【绑定方法】(把 c 记为 self)
c.greet()         # 等价 C.greet(c) —— self 已被 __get__ 绑好
```

- 通过实例访问函数属性 → 函数的 `__get__(实例, 类)` 返回**绑定方法**(自动带 self)。
- 通过类访问 → `__get__(None, 类)` 返回函数本身(不绑定)。

所以"方法"不是什么特殊东西,就是"类里的函数 + 描述符协议带来的绑定"。因为函数是**非数据**描述符,实例 `__dict__` 里若有同名属性会遮蔽它(优先级)。

---

## 五、classmethod / staticmethod 也是描述符

- **`@staticmethod`**:其描述符 `__get__` 返回**原函数**(不绑定任何东西)——所以静态方法不接收 self/cls。
- **`@classmethod`**:其 `__get__` 返回绑定到**类**的方法(第一个参数是 `cls` 而非实例),无论通过实例还是类访问,绑的都是类。
- **`functools.cached_property`(3.8)**:一个**非数据描述符**——`__get__` 首次计算结果后,把它写入**实例 `__dict__`**(同名);之后再访问,因为"实例 `__dict__` 优先于非数据描述符",直接命中实例缓存、不再调用描述符。这就是它"只算一次"的原理。

```python
class Dataset:
    @cached_property
    def stats(self):           # 首次访问才计算
        return expensive_compute()   # 之后存进 self.__dict__['stats'],不再算
```

⚠️ 因为 `cached_property` 是非数据描述符、靠写实例 `__dict__` 缓存,所以**用 `__slots__`(无 `__dict__`)的类不能用它**。

---

## 六、自己写描述符:校验与复用

描述符的实用价值:把"属性的访问逻辑"封装成**可复用**的组件,跨多个属性/类共享(比给每个属性写 property 更省)。`__set_name__`(3.6)自动拿到属性名:

```python
class Positive:                     # 一个可复用的"必须为正数"描述符
    def __set_name__(self, owner, name):
        self.name = name            # 自动得到被赋值的属性名
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return obj.__dict__[self.name]
    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} 必须为正")
        obj.__dict__[self.name] = value

class Account:
    balance = Positive()            # 复用同一个描述符类
    rate = Positive()
```

这正是 **ORM(SQLAlchemy/Django 的字段)、Pydantic、dataclass** 等框架的底层手段——用描述符把字段的校验、类型转换、懒加载、SQL 映射封装起来。

---

## 陷阱清单

- **把描述符放在实例上**:描述符**必须是类属性**才生效;`self.x = Descriptor()` 不会触发协议。
- **混淆数据/非数据描述符优先级**:数据描述符压过实例属性,非数据描述符让位实例属性——记不清就解释不了 property 和 cached_property 的行为差异。
- **`cached_property` 配 `__slots__`**:无 `__dict__` 缓存不了,报错。
- **描述符里直接 `obj.attr` 存值**:可能递归触发自己;应存到 `obj.__dict__[name]`(用不同的 key)。
- **`property` 里访问自己**:`@property def x: return self.x` 无限递归;要存到 `self._x`。
- **以为 `@staticmethod` 必要**:很多静态方法其实可以是模块级函数;classmethod 才常用于替代构造器。
- **过度用描述符**:简单场景用 property/dataclass 即可;自定义描述符用于跨多属性复用的逻辑。

---

## 2026 现状

- **描述符是框架的底层基石**:SQLAlchemy 2.0 的 `Mapped` 字段、Django ORM 字段、Pydantic、`dataclasses` 都靠描述符实现字段语义。
- **`property` + 类型注解** 是封装计算/校验属性的日常手段;`cached_property` 用于惰性昂贵计算。
- **`__set_name__`(3.6)** 让自定义描述符无需手动传属性名,写起来更干净。
- 与 Java 对照:Python 的 `@property` ≈ Java 的 getter/setter,但 Python 能在**不改调用方**(仍写 `obj.x`)的前提下后加校验/计算(统一访问原则),而 Java 一开始就得写 getX()。描述符则比 Java 字段机制更灵活、可复用。

---

## 练习题

1. 什么是描述符?它的协议方法有哪些?描述符必须放在哪里才生效?

<details><summary>参考答案</summary>

**描述符(descriptor)**是一个实现了"描述符协议"中至少一个特殊方法的对象,用来**自定义/拦截对某个属性的访问(读、写、删)**。协议方法有三个:①**`__get__(self, obj, objtype=None)`**——当读取该属性时被调用(obj 是访问它的实例,通过类访问时 obj 为 None);②**`__set__(self, obj, value)`**——当对该属性赋值时被调用;③**`__delete__(self, obj)`**——当删除该属性时被调用。一个对象只要实现了其中任意一个,就是描述符。**描述符必须作为【类属性】定义(放在类上)才会生效**——也就是说,要把描述符实例赋给类的某个属性(`class C: x = MyDescriptor()`),这样当通过该类的实例或类访问 `c.x` 时,属性查找机制(见 P09)在类的 `__dict__` 中发现 `x` 是一个描述符,就会去调用它的 `__get__`/`__set__`/`__delete__`,而不是直接把描述符对象本身返回。**如果把描述符放在实例上**(如在 `__init__` 里 `self.x = MyDescriptor()`),它**不会触发**描述符协议——因为协议只在"通过类查找到的属性是描述符"时才激活,实例 `__dict__` 里的对象只会被当作普通值返回。描述符是 Python 对象模型中非常底层的机制,`property`、`staticmethod`、`classmethod`、`functools.cached_property`、以及方法的"自动绑定 self",还有各种 ORM 的字段,全都建立在它之上。

</details>

2. 数据描述符和非数据描述符有什么区别?它们和实例属性的优先级是怎样的?

<details><summary>参考答案</summary>

区别在于实现了哪些协议方法:**数据描述符(data descriptor)**实现了 **`__set__` 或 `__delete__`**(通常也有 `__get__`)——它能拦截赋值/删除,典型如 `property`;**非数据描述符(non-data descriptor)**只实现了 **`__get__`**(没有 `__set__`/`__delete__`)——典型如普通函数(方法)、`staticmethod`、`classmethod`、`cached_property`。二者最重要的差异体现在**与实例 `__dict__` 的优先级**上,完整的属性查找优先级是:**数据描述符 > 实例 `__dict__`(实例属性)> 非数据描述符 / 普通类属性**(都没有再走 `__getattr__`,见 P09)。这意味着:①**数据描述符"压过"实例属性**——即使实例的 `__dict__` 里存在同名 key,访问该属性时仍然会优先调用类上数据描述符的 `__get__`/`__set__`(这正是 `property` 能始终拦截读写、做校验的原因,实例无法"绕过"它);②**非数据描述符"让位于"实例属性**——如果实例 `__dict__` 里有同名 key,就直接返回实例属性、不再调用非数据描述符的 `__get__`。这条规则解释了两个经典现象:**普通方法是非数据描述符**,所以理论上实例属性能遮蔽同名方法;而 **`functools.cached_property` 是非数据描述符**,它第一次被访问时计算结果并把结果**写入实例 `__dict__`**(同名),由于"实例属性优先于非数据描述符",之后再访问就直接命中实例 `__dict__` 里的缓存值、不再触发计算——这就是"只计算一次"的实现原理。理解这条优先级是看懂 property、方法、cached_property 行为差异的钥匙。

</details>

3. `@property` 的本质是什么?它解决了什么问题?

<details><summary>参考答案</summary>

`@property` 的本质是一个**数据描述符**:被它装饰后,该名字在类上成为一个 property 对象,它同时实现了 `__get__`(读时调用被装饰的 getter 方法)、`__set__`(写时调用 setter,没定义 setter 则赋值报 AttributeError 实现只读)、`__delete__`。因为是数据描述符,它在属性查找中**优先级高于实例 `__dict__`**,所以总能拦截对该属性的读写。**它解决的问题**:让一个属性在**对外接口保持"简单属性访问"语法(`obj.x` / `obj.x = v`)的同时,背后能执行方法逻辑**——比如:①**计算属性**——`area` 由 `radius` 实时算出,不需要存储,访问 `c.area` 像普通属性但其实在运行函数;②**只读属性**——只定义 getter 不定义 setter,外部不能赋值;③**赋值校验/副作用**——在 setter 里校验值的合法性(如半径不能为负)、做日志、触发更新,而调用方仍然写 `c.radius = v`;④**封装/重构友好**——这是"统一访问原则(Uniform Access Principle)"的体现:一开始可以把某字段写成普通属性,以后需要加校验/计算时,可以**无缝改成 property 而不需要修改任何调用方代码**(调用方一直写 `obj.x`)。这正是 Python 相比 Java 的优势之一:Java 为了将来可能加逻辑,往往一上来就写 `getX()/setX()` 样板;而 Python 可以先用朴素属性,真有需要时再用 property 升级,接口不变。用法:`@property` 定义 getter,`@x.setter` 定义 setter,`@x.deleter` 定义删除逻辑;注意 getter/setter 内部要访问真正的存储字段(如 `self._x`),不要访问 property 自身(否则无限递归)。

</details>

4. 为什么说"方法自动绑定 self"是描述符机制的体现?

<details><summary>参考答案</summary>

因为**普通函数对象本身就是一个(非数据)描述符**——函数类型实现了 `__get__` 方法(且没有 `__set__`,所以是非数据描述符)。当你在类里定义一个方法时,它其实就是类 `__dict__` 里的一个普通函数对象。当通过**实例**访问这个方法属性 `obj.method` 时,属性查找机制发现类上的 `method` 是个描述符,于是调用它的 **`__get__(obj, type(obj))`**——函数的 `__get__` 会返回一个**绑定方法(bound method)对象**,这个对象把 `obj` 记录下来作为将来调用时的第一个参数 `self`(通过 `__self__` 指向实例、`__func__` 指向原函数)。所以 `obj.method(args)` 实际等价于 `原函数(obj, args)`——self 是被 `__get__` "自动绑定"上去的,而不需要你手动传。相对地,通过**类**访问 `Cls.method` 时,`__get__(None, Cls)` 返回的是**原函数本身**(没有绑定实例),所以你必须显式传实例:`Cls.method(obj, args)`。这就完整解释了"为什么实例调方法不用传 self、类调方法要传"。延伸:`@staticmethod` 的描述符 `__get__` 返回原函数(不绑定任何东西,因此静态方法签名里没有 self/cls);`@classmethod` 的描述符 `__get__` 返回绑定到**类**的方法(第一个参数 cls 是类对象,无论通过实例还是类访问都绑类)。由于函数是**非数据**描述符,理论上实例 `__dict__` 里的同名属性会遮蔽方法(优先级:实例属性 > 非数据描述符)。一句话:方法 = 类里的函数 + 描述符协议带来的自动绑定,没有任何"魔法"。

</details>

5. `functools.cached_property` 是怎么实现"只计算一次"的?它有什么限制?

<details><summary>参考答案</summary>

`functools.cached_property`(Python 3.8 引入)是一个**非数据描述符**(只实现了 `__get__`,没有 `__set__`)。它的"只计算一次"原理正好利用了属性查找的优先级规则(数据描述符 > 实例 `__dict__` > 非数据描述符):**第一次**访问 `obj.prop` 时,由于实例 `__dict__` 里还没有同名 key,查找落到类上的 cached_property 这个非数据描述符,触发它的 `__get__`——`__get__` 调用被装饰的方法计算出结果,然后把结果**直接写入实例的 `__dict__`**(用同样的属性名作 key),并返回该结果。**之后再次**访问 `obj.prop` 时,因为"实例 `__dict__` 的优先级高于非数据描述符",查找会**直接命中实例 `__dict__` 里缓存的那个值**、根本不会再触发描述符的 `__get__`,于是计算只发生一次、后续都是直接读缓存。它适合"计算昂贵、但实例生命周期内结果不变"的惰性属性。**限制/注意**:①**类必须有 `__dict__`**——因为它靠写实例 `__dict__` 来缓存,所以**定义了 `__slots__`(没有 `__dict__`)的类不能用它**(会报错);②**缓存随实例存活**——缓存的值跟随实例,实例被回收时缓存一起释放(不像 lru_cache 那样可能因函数级缓存持有 self 导致泄漏);③**若底层数据会变,缓存不会自动更新**——需要手动 `del obj.prop`(删除实例 `__dict__` 里的缓存,下次访问重新计算);④它不是线程安全地"恰好计算一次"(并发首次访问可能多算几次,但结果一致);⑤与普通 `property`(数据描述符、每次都调用、不缓存)区别明显——要缓存用 cached_property,要每次重算/要 setter 用 property。

</details>

6. 自定义描述符有什么实际用途?举一个例子。

<details><summary>参考答案</summary>

自定义描述符的核心价值是:把"某类属性的访问逻辑(校验、类型转换、计算、懒加载、存取映射等)"**封装成一个可复用的组件**,从而能**跨多个属性、多个类共享同一套逻辑**,避免为每个属性重复写 property 或在每个 `__init__`/setter 里重复校验。它是众多框架的底层机制。**例子:一个可复用的"值必须为正数"校验描述符**:
```python
class Positive:
    def __set_name__(self, owner, name):   # 3.6+:自动获得被赋值的属性名
        self.name = name
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__[self.name]
    def __set__(self, obj, value):         # 有 __set__ → 数据描述符
        if value <= 0:
            raise ValueError(f"{self.name} 必须为正数")
        obj.__dict__[self.name] = value     # 存到实例字典(用属性名作 key)
```
然后任意类都能复用:`class Account: balance = Positive(); rate = Positive()`——给 `balance`、`rate` 赋负值会自动抛错,而校验逻辑只写了一份。要点:①用 `__set_name__` 自动拿到属性名,无需手动传;②把实际值存到 `obj.__dict__[self.name]`(而不是描述符自身,否则所有实例会共享同一份值);③它是数据描述符(有 `__set__`),所以能稳定拦截赋值。**实际框架中的用途**:ORM(SQLAlchemy 的 `Mapped`/Django 的 Field)用描述符把模型类属性映射到数据库列、并在存取时做类型转换/SQL 生成;Pydantic/dataclasses 用类似机制做字段校验与序列化;还可用于类型强制、单位换算、惰性加载(首次访问才从磁盘/网络取)、记录访问日志、属性别名等。当你发现"很多属性都要做同一种 getter/setter 逻辑"时,描述符比堆叠 property 更 DRY、更优雅。

</details>
