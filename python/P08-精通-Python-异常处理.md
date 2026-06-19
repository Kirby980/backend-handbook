# 精通 Python 异常处理与最佳实践

> Python 推崇 EAFP("先做,错了再处理")而非 LBYL("先检查再做"),异常是它的核心控制流之一。面试常问:"EAFP 是什么?""异常体系结构?""异常组(3.11)?"本篇讲清异常处理与最佳实践,串起 [P06 上下文管理](./P06-精通-Python-装饰器与上下文管理器.md)、[P05 StopIteration](./P05-精通-Python-迭代器与生成器.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、异常体系

所有异常继承自 `BaseException`,业务异常继承自 `Exception`:

```
BaseException
 ├─ SystemExit / KeyboardInterrupt / GeneratorExit   ← 不该被笼统捕获
 └─ Exception                                        ← 业务异常的基类
     ├─ ValueError / TypeError / KeyError / IndexError
     ├─ AttributeError / FileNotFoundError(OSError)
     ├─ RuntimeError / StopIteration / ...
     └─ 你的自定义异常
```

关键:**`SystemExit`(`sys.exit()`)、`KeyboardInterrupt`(Ctrl-C)、`GeneratorExit` 直接继承 `BaseException`,不在 `Exception` 下**——所以 `except Exception` **不会**误吞它们(这是设计:别拦住程序退出/中断)。捕获业务异常用 `except Exception`,绝不用 `except BaseException`。

---

## 二、try / except / else / finally

```python
try:
    data = parse(raw)            # 可能抛异常的代码
except ValueError as e:          # 精确捕获特定异常
    log.warning("parse failed: %s", e)
    data = default
except (KeyError, TypeError):    # 一个 except 捕多种
    ...
else:
    use(data)                    # try 未抛异常才执行(把"成功后"逻辑与 try 分离)
finally:
    cleanup()                    # 无论是否异常都执行(释放资源)
```

- **`except` 顺序从具体到一般**:子类异常写前面,否则会被父类先拦截。
- **`else`**:`try` 块**没抛异常**时执行——把"可能出错的代码"和"成功后才做的代码"分开,缩小 try 范围。
- **`finally`**:**一定执行**(即使 try/except 里 `return`/`raise`)——做清理。⚠️ 别在 finally 里 `return`(会吞掉异常和原返回值)。

---

## 三、EAFP vs LBYL

- **LBYL**(Look Before You Leap,先检查后做):先 `if` 判断条件再操作。
- **EAFP**(Easier to Ask Forgiveness than Permission,先做错了再处理):直接做,出错再 `except`。**Python 推崇 EAFP**。

```python
# LBYL:有竞态(检查后、使用前文件可能被删),且多次查找
if key in d and d[key] is not None:
    use(d[key])
# EAFP:更 Pythonic,无竞态,一次查找
try:
    use(d[key])
except KeyError:
    handle()
```

EAFP 的好处:①避免**检查与使用之间的竞态**(TOCTOU,尤其文件/并发);②少做重复查找(`in` + `[]` 查两次);③更符合 Python "异常廉价"的设计。但**异常用于"异常/罕见"情况**——若失败是常态(高频),频繁抛异常反而慢,这时 LBYL 或 `dict.get(k, default)` 更好。

---

## 四、自定义异常与异常链

```python
class OrderError(Exception):              # 自定义异常基类
    """订单领域异常"""

class InsufficientStock(OrderError):      # 细分
    def __init__(self, sku, need, have):
        super().__init__(f"{sku} 缺货:需 {need} 有 {have}")
        self.sku, self.need, self.have = sku, need, have

# raise ... from:保留异常链(根因)
try:
    row = db.query(sku)
except DBError as e:
    raise OrderError("查询库存失败") from e   # __cause__ = e,traceback 显示两段
```

- **定义领域异常类**:让调用方能精确 `except` 你的异常,而非笼统捕获。
- **`raise NewError from original`**:建立**异常链**(`__cause__`),保留根因,traceback 会打印"在处理 X 时发生了 Y"。**别裸 `raise NewError`** 丢掉原始栈;若想隐藏链用 `from None`。

---

## 五、异常组 except*(3.11)

`ExceptionGroup` + `except*`(PEP 654,3.11)用于**同时处理多个并发产生的异常**(典型:`asyncio.TaskGroup` 里多个任务各自失败,见 [P23](../python/P23-精通-Python-异步实战与陷阱.md)):

```python
try:
    raise ExceptionGroup("多个失败", [ValueError("a"), TypeError("b")])
except* ValueError as eg:        # 处理组里所有 ValueError
    print("值错误:", eg.exceptions)
except* TypeError as eg:         # 处理组里所有 TypeError
    print("类型错误:", eg.exceptions)
```

`except*` 能从一个异常组里**按类型分别拣选并处理**多个异常(普通 `except` 只能处理单个)。这是结构化并发([P23](../python/P23-精通-Python-异步实战与陷阱.md))的配套设施。

---

## 六、最佳实践

- **精确捕获**:`except ValueError` 而非 `except Exception`(更不要裸 `except:`)。只捕获你能处理的。
- **不要吞异常**:`except: pass` 是大忌——错误被无声吞掉,排障无门。至少 `log.exception(...)`(自动带 traceback)。
- **`log.exception` 记录栈**:在 except 块里用它(等价 `log.error(..., exc_info=True)`),保留完整 traceback(见 [P27](./P27-精通-Python-日志与可观测.md))。
- **资源清理用 `with`/`finally`**:见 [P06](./P06-精通-Python-装饰器与上下文管理器.md)。
- **缩小 try 范围**:只包住可能抛异常的那几行,配合 `else` 放成功逻辑。
- **异常携带上下文**:自定义异常带上出错的关键信息(id、值),便于排查。
- **不要用异常做正常控制流的高频路径**:异常应表达"异常情况";高频失败用返回值/`get`。

---

## 陷阱清单

- **裸 `except:` 或 `except BaseException`**:会吞掉 `KeyboardInterrupt`/`SystemExit`,Ctrl-C 杀不掉、程序无法正常退出。
- **`except: pass` 吞异常**:错误消失、排障无门。至少记录日志。
- **`except` 顺序错(父类在前)**:子类 except 永远不被命中。
- **`finally` 里 return/break**:会吞掉异常或覆盖返回值,极隐蔽。
- **裸 `raise NewError` 丢根因**:用 `raise NewError from e` 保留链。
- **捕获太宽**:`except Exception` 把本不该处理的也吞了,掩盖真实 bug。
- **在生成器里 `raise StopIteration`**:被转成 `RuntimeError`(PEP 479);用 `return`(见 [P05](./P05-精通-Python-迭代器与生成器.md))。
- **异常做高频控制流**:失败是常态时频繁抛异常有性能与可读代价。

---

## 2026 现状

- **EAFP 仍是 Python 主流风格**;`dict.get`/`contextlib.suppress`/`try` 各有适用。
- **异常组 `except*` + `asyncio.TaskGroup`(3.11)** 成为结构化并发的错误处理标配(见 [P23](../python/P23-精通-Python-异步实战与陷阱.md))。
- **`add_note()`(3.11)** 可给异常追加上下文说明(`e.add_note("处理订单 123 时")`),不改异常类型也能补充信息。
- **结构化日志 + `log.exception`** 是生产排障标准;配合 Sentry/OTel 上报异常(见 [P27](./P27-精通-Python-日志与可观测.md))。
- 与 Java 对照([J05](../java/INDEX.md)):Java 区分受检/非受检异常并强制声明,Python 全是"非受检"(不强制 try),更灵活但也更依赖文档与约定;两者都推荐异常链与精确捕获。

---

## 练习题

1. Python 的异常体系是怎样的?为什么 `except Exception` 不会捕获 `KeyboardInterrupt`?

<details><summary>参考答案</summary>

Python 所有异常都继承自顶层的 **`BaseException`**。其下分两大支:①一类**直接继承 `BaseException`**(不在 `Exception` 之下)的特殊异常——`SystemExit`(由 `sys.exit()` 触发,用于正常退出程序)、`KeyboardInterrupt`(用户按 Ctrl-C 触发的中断)、`GeneratorExit`(生成器关闭);②**`Exception`**——它是所有"常规/业务"异常的基类,常见的 `ValueError`、`TypeError`、`KeyError`、`IndexError`、`AttributeError`、`FileNotFoundError`(属 `OSError`)、`RuntimeError`、`StopIteration` 以及用户自定义异常都在它之下。**`except Exception` 不会捕获 `KeyboardInterrupt`/`SystemExit` 的原因**:这是**有意的设计**——把这三个"控制程序生命周期/中断"的异常放在 `Exception` 之外、直接挂在 `BaseException` 下,正是为了让常见的 `except Exception`(用来兜底处理业务错误)**不会误吞**它们。设想如果 `except Exception` 把 `KeyboardInterrupt` 也捕获了,那么在一个 `while True: try: ... except Exception: pass` 的循环里,用户按 Ctrl-C 都无法终止程序、`sys.exit()` 也会被拦下,非常危险。所以:**捕获业务异常用 `except Exception`**(它不会拦住退出/中断);**几乎永远不要写 `except BaseException` 或裸 `except:`**(它们会吞掉 KeyboardInterrupt/SystemExit,导致程序无法被 Ctrl-C 或正常退出)。如果确实需要在退出前做清理,应针对性地捕获或用 `finally`/`atexit`/信号处理。

</details>

2. try/except/else/finally 四个块分别在什么时候执行?finally 有什么坑?

<details><summary>参考答案</summary>

①**`try`**:放置可能抛出异常的代码,总是先执行。②**`except`**:当 try 块中抛出了**匹配**的异常时执行对应的 except 分支(可以有多个,按从上到下匹配,应把更具体的子类异常写在更一般的父类之前,否则子类分支永远命中不到);如果没有异常或异常不匹配则跳过。③**`else`**:仅当 try 块**正常执行完、没有抛任何异常**时才执行——它的价值是把"可能出错的操作"(放 try)和"只有成功后才做的后续逻辑"(放 else)清晰分离,从而能**缩小 try 的范围**(只包真正可能抛异常的语句),避免把不该被这个 except 捕获的代码也塞进 try。④**`finally`**:**无论 try 块是正常结束、抛了异常、还是执行了 return/break/continue,都一定会执行**,通常用于必须进行的清理(关闭文件/连接、释放锁)。**finally 的坑**:如果在 finally 里使用 `return`(或 `break`/`continue`),它会**覆盖/吞掉** try 或 except 中的返回值,更危险的是**会吞掉正在传播的异常**——即 try 里抛了异常本应向外传播,但 finally 里的 return 会让函数正常返回、异常被无声丢弃,导致 bug 极难发现。所以**不要在 finally 里写 return**,finally 应只做清理。另外清理资源更推荐用 `with`(上下文管理器,见 P06),比手写 try/finally 更简洁可靠。

</details>

3. 什么是 EAFP 和 LBYL?Python 为什么推崇 EAFP?有没有反例?

<details><summary>参考答案</summary>

**LBYL**(Look Before You Leap,"三思而后行"):在执行某操作前,先用条件判断检查前提是否满足,再操作。如 `if key in d: use(d[key])`、`if os.path.exists(p): open(p)`。**EAFP**(Easier to Ask Forgiveness than Permission,"先斩后奏、错了再道歉"):直接尝试操作,假设前提成立,如果出错(抛异常)再用 except 处理。如 `try: use(d[key]) except KeyError: handle()`。**Python 推崇 EAFP** 的原因:①**避免竞态条件(TOCTOU)**——LBYL 在"检查"和"使用"之间存在时间窗口,期间状态可能改变(典型如多线程/多进程下文件先 `exists` 检查通过、紧接着 `open` 时却被别的进程删了),EAFP 直接尝试 + 捕获就没有这个窗口;②**减少重复工作**——LBYL 往往要查两次(先 `key in d` 判断、再 `d[key]` 取值),EAFP 一次操作搞定;③**更符合 Python 哲学**——Python 的异常机制设计得相对廉价、且鸭子类型下"是否支持某操作"本就难以预先穷举检查,直接尝试更自然、代码也更简洁可读。**反例/不适用情况**:当"失败是高频的正常情况"而非罕见异常时,频繁抛出和捕获异常会有性能开销且语义上不妥(异常应表达"异常情况"),此时 LBYL 或更专门的手段更好——比如用 `dict.get(key, default)` 取带默认值的取值、用 `dict.setdefault`、先校验用户输入合法性等。所以原则是:**罕见错误用 EAFP(try/except),高频可预期的"缺失/不满足"用 LBYL 或带默认的 API**。

</details>

4. 自定义异常有什么意义?`raise ... from ...` 是做什么的?

<details><summary>参考答案</summary>

**自定义异常的意义**:①**让调用方能精确捕获**——定义领域相关的异常类(如 `OrderError`、`InsufficientStock`)后,调用方可以 `except InsufficientStock` 只处理这种具体错误,而不必笼统 `except Exception`(那样会把无关错误也吞了、无法区分错误种类);②**建立异常层次**——让相关异常继承一个共同基类(如所有订单异常继承 `OrderError`),调用方既能精确捕获子类、也能用基类一次捕获整类错误,结构清晰;③**携带上下文信息**——在异常对象上保存出错的关键数据(如缺货的 sku、需要量、库存量),便于上层处理和排障;④**表达业务语义**——异常名字本身就是文档。**`raise NewError from original`**(异常链)的作用:当你在处理一个异常(original)的过程中需要抛出另一个更上层/更语义化的异常(NewError)时,用 `from` 把原始异常关联为新异常的**根因(`__cause__`)**。这样 traceback 会**同时显示两段**:"在处理 original 异常时,发生了 NewError",完整保留了错误的因果链和原始栈,极大方便排障。例如底层 `DBError` 被转换成领域层 `OrderError("查询库存失败") from e`。如果不写 `from e` 而直接 `raise OrderError(...)`,Python 虽然也会隐式记录"在处理上一个异常时发生"(`__context__`),但语义不如显式 `from` 清晰;而如果想**故意隐藏/切断**原始异常链(不暴露底层细节),可以用 `raise NewError from None`。要点:转换异常时用 `raise ... from e` 保留根因,别让原始栈信息丢失。

</details>

5. 处理异常时有哪些常见的反模式?正确做法是什么?

<details><summary>参考答案</summary>

常见反模式及纠正:①**裸 `except:` 或 `except BaseException`**——会捕获包括 `KeyboardInterrupt`、`SystemExit` 在内的一切,导致 Ctrl-C 无法中断、程序无法正常退出;正确做法是捕获 `except Exception`(业务兜底)或更具体的异常类型。②**`except ...: pass` 吞掉异常**——错误被无声吞没,出问题时毫无线索、排障极难;正确做法是至少记录日志(在 except 块里用 `logging.exception("...")`,它会自动附上完整 traceback),只有在确实"可以安全忽略"且明确知道原因时才忽略(可用 `contextlib.suppress(SpecificError)` 表达"有意忽略某特定异常")。③**捕获过宽**(动辄 `except Exception`)——把本不该由这里处理、本该向上抛的错误也吞了,掩盖真实 bug;正确做法是**只捕获你确实能处理的、具体的异常类型**(EAFP 也要精确)。④**`except` 分支顺序把父类写在子类前**——子类分支永远命中不到;应从具体到一般排列。⑤**转换异常时裸 `raise NewError` 丢失根因**——应 `raise NewError from e` 保留异常链。⑥**在 `finally` 里 `return`**——吞掉异常/覆盖返回值;finally 只做清理。⑦**用异常做高频正常控制流**——有性能和语义问题;高频可预期的缺失用 `dict.get`/LBYL。⑧**手动管理资源忘记释放**——用 `with`(上下文管理器)替代手写 try/finally。⑨**异常信息无上下文**——自定义异常应携带关键数据。总原则:**精确捕获、绝不静默吞、记录带栈的日志、保留异常链、用 with 清理资源、缩小 try 范围(配合 else)**。

</details>

6. 什么是异常组(ExceptionGroup)和 `except*`?为什么 3.11 要引入它?

<details><summary>参考答案</summary>

**`ExceptionGroup`**(PEP 654,Python 3.11 引入)是一种特殊异常,它可以**把多个异常打包成一个组**一起抛出/传播;配套的新语法 **`except*`** 用于**从一个异常组里按类型分别拣选并处理其中的多个异常**(普通 `except` 一次只能匹配并处理一个异常,无法优雅地处理"同时发生了好几个不同异常"的情况)。用法:`try: ... except* ValueError as eg: ... except* TypeError as eg: ...`——每个 `except*` 分支会从异常组中提取出所有匹配该类型的异常(`eg.exceptions` 是一个子组),不同分支可以**各自处理一部分**,未被任何分支处理的异常会重新组成一个组继续向外传播。**3.11 引入它的主要动机是结构化并发**:在 `asyncio.TaskGroup`(同样 3.11 引入)中,一个任务组里可能**并发运行多个子任务,它们可能各自独立地失败、同时抛出不同的异常**——传统的单异常模型无法表达"5 个任务里有 3 个失败了,分别是 2 个 ValueError 和 1 个 TimeoutError"。`TaskGroup` 会把这些并发异常收集成一个 `ExceptionGroup` 抛出,调用方就能用 `except*` 分门别类地处理。除了 asyncio,任何需要"汇总多个独立错误"的场景(如并行校验多个字段、批量操作部分失败、重试中累积多次失败)都可用它。配套还有 `BaseExceptionGroup`(含 BaseException 类异常)。一句话:异常组解决了"多个异常需要同时表达和分别处理"的问题,是并发/批处理时代的异常处理基础设施。

</details>
