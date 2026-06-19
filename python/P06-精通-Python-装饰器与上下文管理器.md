# 精通 Python 装饰器与上下文管理器

> 装饰器和上下文管理器是 Python 最优雅的两个特性——前者给函数"包一层"(日志/缓存/鉴权/重试),后者管理资源生命周期(`with`)。面试必问:"装饰器原理?""`functools.wraps` 为什么要加?""`with` 怎么实现?"本篇讲透,依赖 [P04 闭包](./P04-精通-Python-函数参数与作用域.md)、[P01 一等对象](./P01-精通-Python-数据模型与对象.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、装饰器的本质

装饰器 = **接收一个函数、返回一个(增强后的)函数的高阶函数**。它能成立,是因为 Python 里**函数是一等对象**(可传递、可返回,见 [P01](./P01-精通-Python-数据模型与对象.md)),加上**闭包**(见 [P04](./P04-精通-Python-函数参数与作用域.md))。

```python
def log(func):                       # 接收被装饰函数
    def wrapper(*args, **kwargs):    # 闭包,捕获 func
        print(f"call {func.__name__}")
        result = func(*args, **kwargs)
        print("done")
        return result
    return wrapper                   # 返回增强后的函数

def add(a, b):
    return a + b
add = log(add)        # 手动装饰:add 现在指向 wrapper
```

`@log` 只是 `add = log(add)` 的语法糖。`*args/**kwargs` 让 wrapper 能透传任意参数。

---

## 二、@ 语法糖与 functools.wraps

```python
import functools
def log(func):
    @functools.wraps(func)           # 关键!保留原函数元数据
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

@log                                 # 等价 greet = log(greet)
def greet(name):
    "打招呼"
    return f"hi {name}"
```

**为什么必须 `@functools.wraps(func)`**:不加的话,被装饰后 `greet.__name__` 变成 `"wrapper"`、`greet.__doc__` 丢失、签名错乱——因为 `greet` 现在实际指向 `wrapper`。`wraps` 把原函数的 `__name__`/`__doc__`/`__wrapped__`/签名等拷到 wrapper 上,让装饰透明。**写装饰器永远加 `@wraps`**。

---

## 三、带参数的装饰器(三层)

要让装饰器接收参数(如 `@retry(times=3)`),需再套一层——**装饰器工厂**:

```python
def retry(times):                    # ① 接收参数,返回真正的装饰器
    def decorator(func):             # ② 接收函数
        @functools.wraps(func)
        def wrapper(*args, **kwargs): # ③ 真正的包装
            for i in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if i == times - 1:
                        raise
        return wrapper
    return decorator

@retry(times=3)                      # retry(3) 先返回 decorator,再装饰
def fetch(): ...
```

三层:`retry(times)` → `decorator(func)` → `wrapper(*args)`。`@retry(times=3)` 先调用 `retry(3)` 得到 `decorator`,再用它装饰 `fetch`。

---

## 四、类装饰器与装饰类

- **用类做装饰器**:实现 `__call__`(见 [P01](./P01-精通-Python-数据模型与对象.md))让实例可调用,适合需要保存状态的装饰器:
  ```python
  class Counter:
      def __init__(self, func):
          functools.update_wrapper(self, func)
          self.func, self.count = func, 0
      def __call__(self, *args, **kwargs):
          self.count += 1
          return self.func(*args, **kwargs)
  ```
- **装饰类**:装饰器也能作用于类(返回修改后的类),如 `@dataclass`、注册类、单例。

---

## 五、functools 利器

```python
from functools import wraps, lru_cache, cache, partial, cached_property

@lru_cache(maxsize=128)              # 缓存函数结果(记忆化)
def fib(n):
    return n if n < 2 else fib(n-1) + fib(n-2)

@cache                               # 3.9+,等价 lru_cache(maxsize=None) 无上限
def heavy(x): ...

add5 = partial(lambda a, b: a+b, 5)  # 固定部分参数,返回新可调用

class C:
    @cached_property                 # 首次计算后缓存到实例(见 P11 描述符)
    def data(self): ...
```

- **`lru_cache`/`cache`**:自动记忆化,极大加速纯函数/递归;参数必须**可哈希**(见 [P01](./P01-精通-Python-数据模型与对象.md))。⚠️ 缓存基于参数,**可变参数/长生命周期会内存泄漏**——给方法加 `lru_cache` 还会让 `self` 被缓存持有(内存泄漏)。
- **`partial`**:固定部分参数,是闭包之外携带配置的轻量手段。

---

## 六、上下文管理器与 with

`with` 确保**资源被正确获取与释放**(无论是否异常),靠对象实现上下文管理协议:

- **`__enter__()`**:进入时调用,返回值绑定给 `as` 后的变量。
- **`__exit__(exc_type, exc_val, tb)`**:**退出时一定调用**(正常或异常),做清理。返回 `True` 可吞掉异常。

```python
class Timer:
    def __enter__(self):
        self.t = time.perf_counter(); return self
    def __exit__(self, *exc):
        print(f"耗时 {time.perf_counter()-self.t:.3f}s")
        return False                 # 不吞异常

with open("f.txt") as f:             # 文件:__exit__ 里自动 close
    data = f.read()
```

**`contextlib` 更简单**:

```python
from contextlib import contextmanager, suppress, ExitStack
@contextmanager                      # 用生成器写上下文管理器
def timer():
    t = time.perf_counter()
    try:
        yield                        # yield 之前=__enter__,之后=__exit__
    finally:
        print(time.perf_counter() - t)

with suppress(FileNotFoundError):    # 优雅忽略特定异常
    os.remove("maybe.txt")
with ExitStack() as stack:           # 动态管理可变数量的上下文
    files = [stack.enter_context(open(p)) for p in paths]
```

`@contextmanager` + 生成器是写上下文管理器的最简方式:`yield` 前是进入逻辑,`finally` 里是退出清理。

---

## 陷阱清单

- **装饰器不加 `@functools.wraps`**:丢失原函数名/文档/签名,影响调试、内省、文档工具。
- **带参装饰器层数搞错**:`@deco` vs `@deco()` 用错——无参版直接传函数,有参版要先调用。
- **`lru_cache` 内存泄漏**:无界 `cache`/给实例方法加缓存会持有对象不释放;长生命周期慎用、设 `maxsize`。
- **`lru_cache` 参数不可哈希**:传 list/dict 报错。
- **`__exit__` 误返回 True**:会**吞掉所有异常**,隐藏 bug;不想吞就返回 None/False。
- **手动管理资源不用 with**:忘记 close/release;一律用 `with`。
- **装饰器在导入时执行**:`@app.route(...)` 等在模块导入时就运行,注意副作用与导入顺序。
- **`@contextmanager` 里 yield 不放 try/finally**:异常时清理不执行。

---

## 2026 现状

- **装饰器无处不在**:Web 路由(`@app.get`)、缓存(`@cache`)、鉴权、重试、限流、`@dataclass`、`@property`、`@pytest.fixture`——是框架的核心机制。
- **`functools.cache`(3.9+)** 取代 `lru_cache(maxsize=None)`;`cached_property` 常用于惰性计算属性。
- **`contextlib`** 的 `@contextmanager`、`suppress`、`ExitStack`、`closing` 让资源管理极简;异步版 `@asynccontextmanager` + `async with`(见 [P23](./P23-精通-Python-异步实战与陷阱.md))。
- 与 Java 对照:装饰器类似 Java 注解 + AOP(见 [J23](../java/INDEX.md)),但 Python 装饰器是真实的函数包装、更直接;`with` 类似 Java try-with-resources(见 [J05](../java/INDEX.md))。

---

## 练习题

1. 装饰器的本质是什么?`@deco` 语法糖等价于什么?

<details><summary>参考答案</summary>

装饰器的本质是一个**高阶函数**:它**接收一个函数(或类)作为参数,返回一个(通常是增强/包装过的)新函数(或类)**。它之所以能成立,依赖 Python 的两个特性:①**函数是一等对象**——函数可以像普通值一样作为参数传递、作为返回值返回、赋给变量(见 P01);②**闭包**——内层包装函数能记住并访问外层的被装饰函数(见 P04)。`@deco` 放在 `def func(): ...` 上方,**等价于** `func = deco(func)`——即定义完 `func` 后,立刻把它传给 `deco`,再把 `deco` 的返回值重新绑定回名字 `func`。所以之后调用 `func(...)` 实际调用的是 `deco` 返回的那个包装函数。典型实现:`def log(fn): def wrapper(*args, **kwargs): <前置逻辑>; r = fn(*args, **kwargs); <后置逻辑>; return r; return wrapper`——`wrapper` 用 `*args/**kwargs` 透传任意参数,在调用原函数前后插入额外行为(日志、计时、缓存、鉴权、重试等),从而在**不修改原函数代码**的前提下增强它。多个装饰器叠加 `@a @b def f` 等价于 `f = a(b(f))`(就近的先应用)。

</details>

2. 为什么写装饰器要加 `functools.wraps`?不加会怎样?

<details><summary>参考答案</summary>

因为装饰后,名字实际指向的是装饰器内部定义的**包装函数(wrapper)**,而不是原函数。如果不做处理,wrapper 会带着自己的元数据:`func.__name__` 会变成 `"wrapper"`、`func.__doc__`(文档字符串)会丢失或变成 wrapper 的、`__module__`/`__qualname__`/默认参数/类型注解/签名等也都是 wrapper 的而非原函数的。这会带来一系列问题:①调试时打印函数名、看 traceback 都显示 "wrapper",难以定位;②`help(func)`、文档生成工具、IDE 提示拿到的是错误的名字和文档;③依赖函数元数据/签名的内省代码(如某些框架用 `inspect.signature` 解析参数、或按 `__name__` 注册路由)会出错;④多个被同一装饰器装饰的函数看起来"都叫 wrapper"。`@functools.wraps(func)` 装饰内部的 wrapper,会把**原函数的 `__name__`、`__doc__`、`__module__`、`__qualname__`、`__dict__`、`__wrapped__`(指回原函数)以及签名**等关键属性复制/关联到 wrapper 上,使被装饰后的函数对外**看起来仍然像原函数**,内省、文档、调试都正常。所以最佳实践是:**任何手写装饰器都应在 wrapper 上加 `@functools.wraps(func)`**(类装饰器用 `functools.update_wrapper`)。

</details>

3. 怎么写一个带参数的装饰器(如 `@retry(times=3)`)?

<details><summary>参考答案</summary>

带参数的装饰器需要**三层嵌套函数**(比无参装饰器多一层),因为 `@retry(times=3)` 这个语法会**先调用 `retry(times=3)`、用它的返回值去装饰函数**——也就是说 `retry(times=3)` 必须返回一个"真正的装饰器"。结构是:①最外层 `def retry(times):` 接收**装饰器参数**,返回一个装饰器;②中间层 `def decorator(func):` 是真正的装饰器,接收**被装饰函数**,返回包装函数;③最内层 `def wrapper(*args, **kwargs):` 是包装逻辑,使用到了 `times` 和 `func`(闭包捕获)。完整例子:
```python
import functools
def retry(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for i in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if i == times - 1:
                        raise
        return wrapper
    return decorator
```
调用链:`@retry(times=3)` 先执行 `retry(3)` 返回 `decorator`,然后 `decorator` 装饰目标函数(等价 `fetch = retry(3)(fetch)`)。要点:①别忘了内层 `@functools.wraps(func)`;②`times` 被 `wrapper` 通过闭包记住;③这也解释了无参装饰器 `@deco`(直接 `deco(func)`)和有参装饰器 `@deco()`(先 `deco()` 再装饰)在用法上的区别——用错括号会报错(无参版被当成参数传了函数,有参版漏了调用)。如果想让装饰器**既能带参又能不带参**,可以用 `functools.partial` 或判断第一个参数是否可调用来兼容,但通常明确区分更清晰。

</details>

4. `with` 语句和上下文管理器是怎么工作的?`__exit__` 返回值有什么讲究?

<details><summary>参考答案</summary>

`with` 用于**确定性地管理资源的获取与释放**(无论代码块是否抛异常都能正确清理),它依赖对象实现**上下文管理协议**——两个方法:①**`__enter__(self)`**:在进入 `with` 块时调用,做"获取资源/准备"工作,其**返回值会绑定给 `as` 后的变量**(如 `with open(p) as f`,`f` 是 `__enter__` 的返回值);②**`__exit__(self, exc_type, exc_val, exc_tb)`**:在离开 `with` 块时调用——无论是正常执行完、还是块内抛了异常、还是 return/break,**`__exit__` 都保证被调用**,用于做"释放资源/清理"工作(如关闭文件、释放锁、回滚事务)。当块内发生异常时,异常的类型、值、traceback 会作为三个参数传给 `__exit__`(正常退出时三者均为 None)。**`__exit__` 返回值的讲究**:如果它返回一个**真值(True)**,表示"我已经处理了这个异常",Python 会**吞掉(抑制)该异常**、不再向外传播;如果返回**假值(None/False,默认)**,异常会在 `__exit__` 执行完清理后**继续向外抛出**。所以——除非你确实想吞掉某类异常,否则 `__exit__` 应返回 None/False;**误返回 True 会静默吞掉所有异常、隐藏 bug**,是个危险陷阱。`with` 的价值就是把 try/finally 的清理逻辑封装进对象,既保证清理执行、又让业务代码简洁(类似 Java 的 try-with-resources)。还可以用 `with A() as a, B() as b:` 同时管理多个资源,或用 `contextlib.ExitStack` 管理动态数量的资源。

</details>

5. `contextlib.contextmanager` 怎么用?它和写 `__enter__/__exit__` 类有什么关系?

<details><summary>参考答案</summary>

`@contextlib.contextmanager` 是一个装饰器,让你能**用一个生成器函数**来写上下文管理器,而不必定义一个实现 `__enter__/__exit__` 的完整类,代码更简洁。用法:写一个**只 `yield` 一次**的生成器函数并加上 `@contextmanager`——**`yield` 之前的代码相当于 `__enter__`**(获取资源/准备),**`yield` 产出的值就是 `as` 变量拿到的值**(不需要返回值就 `yield` 空),**`yield` 之后的代码相当于 `__exit__`**(清理);为了保证即使块内抛异常也能清理,通常把 `yield` 放在 `try` 中、把清理放在 `finally` 里。例如:
```python
from contextlib import contextmanager
@contextmanager
def timer():
    t = time.perf_counter()
    try:
        yield
    finally:
        print(time.perf_counter() - t)
```
然后 `with timer():` 即可。**关系**:`@contextmanager` 内部其实是把你的生成器包装成了一个实现了 `__enter__/__exit__` 协议的对象——`__enter__` 驱动生成器执行到 `yield`(并返回 yield 的值),`__exit__` 再驱动生成器从 `yield` 处继续(把块内的异常通过 `gen.throw()` 抛进生成器,所以放在 try/finally 里能捕获/清理)。所以二者等价、可互换:**简单的、主要是"前置+后置清理"逻辑的上下文管理器用 `@contextmanager` + 生成器最省事**;而当上下文管理器需要保存较多状态、被复用为对象、或逻辑复杂时,写成实现 `__enter__/__exit__` 的**类**更清晰。`contextlib` 还提供 `suppress`(忽略指定异常)、`closing`(给有 close 方法的对象加上下文)、`ExitStack`(动态管理多个上下文)、以及异步版 `@asynccontextmanager`(配合 `async with`)。

</details>

6. `functools.lru_cache` 有什么用?用它有哪些注意事项/坑?

<details><summary>参考答案</summary>

`functools.lru_cache`(以及 3.9+ 的无上限版 `functools.cache`)是一个装饰器,为函数提供**记忆化(memoization)缓存**:它按函数的**调用参数**缓存返回值,相同参数再次调用时直接返回缓存结果、不再执行函数体。这能极大加速**纯函数**(无副作用、相同输入总是相同输出)和**重复子问题多的递归**(如斐波那契、动态规划)——例如给 `fib` 加 `@lru_cache` 能把指数级递归降到线性。`maxsize` 参数限制缓存条目数(LRU 淘汰最久未用的),`@cache` 等价于 `lru_cache(maxsize=None)`(无上限);可用 `func.cache_info()` 看命中率、`func.cache_clear()` 清空。**注意事项/坑**:①**参数必须可哈希**——因为缓存以参数为 key 存在 dict 里,传入 list/dict/set 等不可哈希参数会直接报 `TypeError`(可改传 tuple/frozenset);②**内存泄漏风险**——`@cache` 或 `maxsize=None` 缓存无上限,长生命周期进程里参数空间大时会无限增长吃内存;应设合理的 `maxsize`;③**给实例方法加 lru_cache 会泄漏对象**——因为 `self` 会作为缓存 key 被缓存字典**强引用持有**,导致实例无法被回收(内存泄漏),且不同实例共享同一函数级缓存语义混乱;实例级缓存应改用 `functools.cached_property`(缓存到实例自身,实例回收时一起释放)或在实例上自己管理缓存;④**只适合纯函数**——如果函数有副作用、依赖外部可变状态或返回随时间变化的结果(如查数据库、当前时间),缓存会返回过期/错误结果;⑤缓存不区分"等价但不同"的参数(如 `f(1)` 与 `f(1.0)` 可能命中同一条,因为 `1 == 1.0` 且 hash 相同);⑥多线程下 lru_cache 基本线程安全(可能重复计算但不会损坏)。总之:用于可哈希参数的纯函数,设好上限,别用在实例方法上。

</details>
