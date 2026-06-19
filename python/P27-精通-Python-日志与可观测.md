# 精通 Python 日志与可观测

> `print` 调试是新手习惯,生产要用 `logging`——可分级、可配置、可路由、可结构化。面试问:"logging 的层级结构?""为什么不用 f-string 记日志?"本篇讲清日志体系与可观测,串起 [P03 格式化](./P03-精通-Python-字符串与编码.md)、[P08 异常](./P08-精通-Python-异常处理.md)。
>
> **📅 基准:2026 年 6 月,结构化日志 + OpenTelemetry 主流。**

---

## 一、为什么不用 print

`print` 调试的问题:无法分级(都一样)、无法统一开关、无时间/位置/级别信息、无法路由到文件/远程、无法结构化。**生产用 `logging`**:

```python
import logging
logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)
log.info("user %s logged in", user_id)    # 分级、带元数据、可路由
```

`logging` 提供:**级别**(DEBUG<INFO<WARNING<ERROR<CRITICAL,按级别过滤)、时间/模块/行号、输出目标可配、可结构化。

---

## 二、logging 架构:四件套

`logging` 的核心是四类组件协作:

- **Logger**:你调用的入口(`getLogger(name)`),按名字组织成**树形层级**。
- **Handler**:决定日志**输出到哪**(控制台 `StreamHandler`、文件 `FileHandler`/`RotatingFileHandler`、远程、syslog…)。一个 logger 可挂多个 handler。
- **Formatter**:决定日志的**格式**(时间、级别、名字、消息、traceback…)。
- **Filter**:更细粒度的过滤。

```python
log = logging.getLogger("myapp.service")
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s %(message)s"))
log.addHandler(handler)
log.setLevel(logging.INFO)
```

记录流程:`log.info(...)` → 按级别过滤 → 经 Filter → 传给各 Handler → Handler 用 Formatter 格式化并输出。

---

## 三、层级与传播

Logger 按名字(`a.b.c`)构成**树**,根是 root logger。日志记录会**向上传播(propagate)**到父 logger 的 handler:

```python
logging.getLogger("myapp")          # 父
logging.getLogger("myapp.service")  # 子,记录会传到 myapp 及 root 的 handler
```

- **按模块命名 logger**:`log = logging.getLogger(__name__)`——自动得到模块路径名的层级 logger,便于按模块控制级别/输出。
- **在根/父 logger 配 handler**,子 logger 不必各自配(靠传播)。
- **传播重复**:若子 logger 和父 logger 都加了 handler,日志会被**输出多次**(每层 handler 各打一遍)——常见坑;通常只在顶层配 handler。

---

## 四、配置:dictConfig

生产用 **`logging.config.dictConfig`**(或文件)集中配置,而非散落代码里:

```python
import logging.config
logging.config.dictConfig({
    "version": 1,
    "formatters": {"json": {"()": "pythonjsonlogger.jsonlogger.JsonFormatter"}},
    "handlers": {"console": {"class": "logging.StreamHandler", "formatter": "json"}},
    "root": {"level": "INFO", "handlers": ["console"]},
})
```

- **库代码不要配置 root logger / 不要加 handler**:库只 `getLogger(__name__)` 记录,**把配置权留给应用**(应用决定级别和输出)。库给自己的 logger 加 `NullHandler` 避免"No handler"警告。
- 应用在入口处用 dictConfig 统一配置。

---

## 五、惰性格式化与结构化日志

- **用 `%` 占位参数,不要 f-string**(见 [P03](./P03-精通-Python-字符串与编码.md)):`log.info("x=%s", x)` 是**惰性求值**——若该级别未启用,**不会**做字符串格式化(省开销);而 `log.info(f"x={x}")` 无论是否输出都先拼好了字符串。
  ```python
  log.debug("payload=%s", expensive_repr(obj))   # ✅ debug 关闭时不调用 expensive_repr
  log.debug(f"payload={expensive_repr(obj)}")     # ❌ 总是先求值
  ```
- **结构化日志(JSON)**:用 `structlog` 或 `python-json-logger` 输出 JSON 日志(键值对),便于机器解析、检索、聚合(ELK/Loki)。生产可观测的标配——日志带上 `request_id`、`user_id`、`trace_id` 等上下文字段。

---

## 六、异常日志与可观测

- **记录异常带栈**:在 `except` 块里用 **`log.exception("...")`**(等价 `log.error(..., exc_info=True)`)——自动附上完整 traceback(见 [P08](./P08-精通-Python-异常处理.md)),别只 `log.error(str(e))`(丢了栈)。
- **关联追踪(OpenTelemetry)**:微服务里把日志、指标、链路追踪(trace)关联——日志带 `trace_id`/`span_id`,出问题能从一条日志跳到完整调用链。OTel 是 2026 的可观测标准(对照 [B24 可观测](../backend/INDEX.md))。
- **错误上报**:Sentry 等捕获未处理异常 + 上下文,生产告警。
- 三大支柱:**日志(logs)、指标(metrics)、链路(traces)**——结构化 + 关联是关键。

---

## 陷阱清单

- **用 print 当日志**:无分级/路由/上下文;生产用 logging。
- **日志用 f-string**:丧失惰性求值,无论是否输出都先格式化,浪费(尤其 debug 日志、昂贵参数)。用 `%` 占位。
- **`log.error(str(e))` 丢栈**:看不到 traceback,排障难;用 `log.exception(...)`。
- **库代码配置 root / 加 handler**:抢了应用的配置权、产生重复或意外输出;库只 getLogger + NullHandler。
- **logger 传播导致重复日志**:父子都加 handler → 打多遍;通常只在顶层配。
- **直接用 root logger(`logging.info`)**:难按模块控制;用 `getLogger(__name__)`。
- **日志泄露敏感信息**:密码、token、PII 进日志(还可能被索引);脱敏。
- **同步日志 IO 阻塞**:高频日志写慢介质阻塞主流程(尤其 asyncio,见 [P22](./P22-精通-Python-asyncio与协程.md));用异步/队列 handler、控制级别。

---

## 2026 现状

- **结构化(JSON)日志 + OpenTelemetry** 是云原生/微服务可观测标配:日志带 trace_id 关联链路,统一接入 Grafana/Loki/Tempo、ELK 等。
- **`structlog`** 流行(灵活的结构化日志、上下文绑定);标准 `logging` 仍是基础和兼容层。
- **Sentry** 等做异常聚合告警;日志采样控制成本。
- **`logging` 性能**:高吞吐场景用 `QueueHandler`/异步,避免日志 IO 阻塞业务(异步服务尤其重要)。
- 与 Go/Java 对照:Go 用 `slog`(结构化,[G30](../golang/INDEX.md));Java 用 SLF4J/Logback;Python 的 `logging` 灵活但配置较繁,理念一致——**结构化 + 分级 + 关联追踪**是跨语言的现代日志范式。

---

## 练习题

1. 为什么生产环境要用 logging 而不是 print?

<details><summary>参考答案</summary>

`print` 适合临时调试,但**不适合生产**,因为它缺少日志系统该有的能力,而 `logging` 都提供了:①**日志级别(分级)**——logging 有 DEBUG/INFO/WARNING/ERROR/CRITICAL 五个级别,可以**按级别过滤**(如生产只输出 INFO 及以上、开发时打开 DEBUG),一行配置就能调整详略;print 全都一样、无法分级开关。②**丰富的元数据**——logging 能自动附带**时间戳、日志级别、记录器名字(模块)、行号、线程/进程**等信息,便于定位;print 只有你手写的内容。③**可路由的输出目标(Handler)**——logging 能把日志同时/分别送到控制台、文件(支持轮转 RotatingFileHandler)、syslog、远程收集器等,且可针对不同 handler 设不同级别/格式;print 只能去标准输出。④**可配置的格式(Formatter)与过滤(Filter)**——统一格式、按需过滤;⑤**层级化、按模块控制**——用 `getLogger(__name__)` 得到树形 logger,可对不同模块单独设级别,集中在应用入口用 dictConfig 配置;⑥**结构化日志**——可输出 JSON 便于机器解析、检索、聚合(ELK/Loki);⑦**异常栈**——`log.exception` 自动附完整 traceback;⑧**惰性格式化**——用 `%` 占位时,未启用的级别不会执行字符串拼接,省开销;⑨**生态集成**——可对接 OpenTelemetry(trace 关联)、Sentry(告警)等可观测体系。总之,生产需要的是"可分级、可配置、可路由、带上下文、可结构化、可关联追踪、性能可控"的日志,这些 print 都做不到。所以**库和应用都应使用 logging**(库只记录、把配置权留给应用),把 print 留给一次性的本地调试。

</details>

2. logging 的 Logger、Handler、Formatter、Filter 各是什么?

<details><summary>参考答案</summary>

这是 logging 的四个核心组件,协作完成"产生→过滤→格式化→输出"日志:①**Logger(记录器)**——是**应用代码调用的入口**,通过 `logging.getLogger(name)` 获取(同名返回同一个,按名字 `a.b.c` 组织成**树形层级**)。你用 `logger.info()/.debug()/.error()` 等方法产生日志记录(LogRecord);Logger 有自己的级别,低于其级别的调用被忽略。②**Handler(处理器)**——决定日志**输出到哪里**。常见的有 `StreamHandler`(控制台/stderr)、`FileHandler`/`RotatingFileHandler`/`TimedRotatingFileHandler`(文件,支持按大小/时间轮转)、`SysLogHandler`、`SMTPHandler`、`QueueHandler`(异步)、以及发往远程/网络的 handler。一个 Logger 可以挂**多个** Handler(同时输出到控制台和文件),每个 Handler 也可单独设级别。③**Formatter(格式化器)**——决定日志记录**长什么样**,通过格式字符串(如 `"%(asctime)s %(levelname)s %(name)s %(message)s"`)把时间、级别、记录器名、消息、traceback 等字段渲染成最终文本(或用 JSON formatter 渲染成结构化 JSON)。每个 Handler 关联一个 Formatter。④**Filter(过滤器)**——提供比"级别"更**细粒度的过滤**逻辑,可挂在 Logger 或 Handler 上,根据 LogRecord 的任意属性(如某个字段、模块、自定义条件)决定是否放行,也可用于给记录**注入上下文字段**(如 request_id)。**协作流程**:`logger.info(msg)` 产生一条 LogRecord → Logger 先按自身级别过滤、经其 Filter → 记录沿 Logger 树**向上传播**到各级 Logger 的 Handler → 每个 Handler 再按自己的级别和 Filter 过滤 → 用其 Formatter 格式化 → 输出到目标。理解这套分工,就能灵活配置"哪些日志、以什么格式、送到哪里"。

</details>

3. logger 的层级和传播是怎么回事?会带来什么坑?

<details><summary>参考答案</summary>

**层级**:logging 的 Logger 通过**点分名字**组织成一棵**树**——名字 `a.b.c` 的 logger 是 `a.b` 的子、`a.b` 是 `a` 的子,所有 logger 的共同祖先是**根 logger(root)**。通常约定用 `logging.getLogger(__name__)` 获取 logger,这样每个模块的 logger 名就是它的模块路径(如 `myapp.service.user`),天然形成与代码结构对应的层级。**传播(propagate)**:当一条日志记录在某个 logger 上产生并通过其级别/过滤后,它不仅会交给**该 logger 自己的 handler**,默认还会**向上传播给所有祖先 logger,依次交给它们的 handler 处理**,直到 root(除非中途某 logger 的 `propagate=False` 截断)。这个设计的好处是:你**只需在根(或某个父)logger 上配置一次 handler 和级别**,所有子 logger 的日志都会向上传播、被统一输出和格式化,**子 logger 无需各自配置 handler**;同时还能对不同子树单独调整级别(如把 `myapp.db` 设为 DEBUG、其余 INFO)。**坑**:**重复输出**——如果你**既在子 logger 上加了 handler、又在父/根 logger 上也加了 handler**,那么一条日志会先被子 logger 的 handler 输出一遍,再传播上去被父/根 logger 的 handler 再输出一遍,导致**同一条日志打印多次**。这是最常见的 logging 配置错误。**避免**:通常**只在顶层(root 或应用顶层 logger)配置 handler**,子 logger 只负责产生日志、靠传播输出;若确实要给某 logger 单独配 handler 且不想重复,可把它的 `propagate` 设为 False(但要权衡)。另一个相关坑是**直接用 root logger**(如调用 `logging.info(...)` 或 `basicConfig` 后到处用根)难以按模块精细控制——应坚持 `getLogger(__name__)`。还有:**库代码不应给 root 加 handler 或调用 basicConfig**(会干扰使用该库的应用的日志配置),库只 `getLogger(__name__)` 记录、必要时加 `NullHandler`,把配置权留给应用。

</details>

4. 为什么记录日志要用 `log.info("x=%s", x)` 而不是 `log.info(f"x={x}")`?

<details><summary>参考答案</summary>

因为前者(用 `%` 占位 + 参数)支持 **惰性格式化(lazy formatting)**,而 f-string 是**立即求值**。当你写 `log.info("x=%s", x)` 时,你**只是把格式串和参数 `x` 传给了 logging**——真正把 `%s` 替换成 `x` 的值、拼出最终字符串这一步,会被**推迟到 logging 确认这条日志需要被输出时**才执行。如果当前的日志级别比这条高(例如生产中级别是 INFO,而这是一条 `log.debug(...)`),这条日志**不会被输出**,于是 logging **根本不会做那次字符串格式化**,从而**省掉了拼接(以及对参数调用 `__str__`/`__repr__`)的开销**。这在以下情况尤其重要:①**高频日志**(循环里、热路径上的 debug 日志);②**参数的字符串化很昂贵**(如 `log.debug("data=%s", huge_object)`,`huge_object` 的 repr 可能很慢/很大)——惰性下级别未开就完全不构造它。相反,`log.info(f"x={x}")` 中的 f-string 会在**调用 logging 之前就已经被 Python 求值、拼接成完整字符串**了(f-string 是普通表达式),**无论这条日志最终是否输出,格式化开销都已经发生**;对昂贵参数尤其浪费(`log.debug(f"data={expensive_repr(obj)}")` 即使 debug 关闭也会调用 `expensive_repr`)。所以**最佳实践是用 `%` 占位 + 把变量作为参数传入**(logging 也支持 `{}` 风格但要配置,`%` 是默认),让 logging 决定是否真的格式化。补充:这也是为什么 lint 工具(如 pylint/ruff)会对"logging 里用 f-string/`.format()`"给出告警。另外,把参数分开传还有利于**结构化日志**(可以把参数作为结构化字段记录,而非糊成一个字符串)。当然,对于一定会输出的、参数廉价的日志,f-string 的影响很小,但养成用 `%` 占位的习惯在性能敏感和高频日志场景更稳妥。

</details>

5. 什么是结构化日志?为什么生产环境推荐它?

<details><summary>参考答案</summary>

**结构化日志(structured logging)**指把日志记录成**机器易解析的结构化数据(通常是 JSON 的键值对)**,而不是一行人类可读但格式自由的纯文本。例如不写 `"user 123 placed order 456 amount 99"` 这样的散文,而是输出 `{"event": "order_placed", "user_id": 123, "order_id": 456, "amount": 99, "level": "INFO", "ts": "...", "trace_id": "abc"}` 这样带明确字段的记录。Python 里可用 `structlog` 或 `python-json-logger`(给标准 logging 配 JSON Formatter)实现,并支持把**上下文字段**(request_id、user_id、trace_id 等)绑定到后续日志上。**为什么生产推荐**:①**便于机器解析、检索和聚合**——现代生产日志会被集中收集到日志平台(ELK/Elasticsearch、Grafana Loki、Splunk 等),结构化的字段可以被**索引、精确查询、过滤、聚合统计**(如"查 user_id=123 的所有 ERROR""按 endpoint 统计错误率"),而非结构化的自由文本只能做脆弱的正则/全文搜索,难以可靠提取字段;②**关联与可观测**——结构化字段里带上 `trace_id`/`span_id` 等,能把一条日志与分布式链路追踪(OpenTelemetry)、指标关联起来,出问题时从一条日志跳到完整调用链(可观测三支柱:日志/指标/链路);③**一致性**——字段化避免了每个人写日志格式各异、难以统一处理的问题;④**告警/分析自动化**——基于字段做告警规则、仪表盘、异常检测更容易。所以云原生/微服务时代,**结构化(JSON)日志 + 统一字段(含 trace_id)+ 集中收集 + OTel 关联**是标配。**注意**:仍要兼顾人类可读性(开发环境可用彩色文本格式)、控制日志量与采样以降成本、以及**脱敏**(别把密码/token/PII 记进结构化字段被索引)。这与 Go 的 slog(G30)、Java 的结构化日志理念一致。

</details>

6. 在 except 块里记录异常,为什么要用 `log.exception` 而不是 `log.error(str(e))`?

<details><summary>参考答案</summary>

因为 **`log.exception(...)` 会自动记录完整的异常堆栈跟踪(traceback),而 `log.error(str(e))` 只记录了异常的消息文本、丢失了栈信息**,后者会让排障变得极其困难。具体:①**`log.error(str(e))`**(或 `log.error(f"failed: {e}")`)只把异常对象转成字符串(通常就是异常消息,如 `"division by zero"`),日志里**没有 traceback**——你只知道"出了个 XXError"、却**不知道它是从哪一行、经过怎样的调用链抛出来的**,在复杂代码里几乎无法定位根因。②**`log.exception("...")`** 必须在 **except 块内**调用,它等价于 `log.error("...", exc_info=True)`——会自动捕获**当前正在处理的异常的完整信息**(异常类型、消息、以及**整个调用栈的 traceback**)并附加到日志输出中,这样日志里就有了完整的"错误现场":在哪个文件哪一行、经过哪些函数调用、最终在哪抛出,排障时一目了然。它记录的级别是 ERROR(也可用 `exc_info=True` 配合其他级别)。所以**在捕获并记录异常时应优先用 `log.exception("处理订单失败")`**(再带上下文信息更好,如订单 id),把完整栈留在日志里。补充注意:①只在确实"捕获并要记录"时用它,且通常配合"记录后重新 raise 或做降级处理"——不要捕获后既不处理也不记录(静默吞异常是大忌,见 P08);②也可以用 `exc_info=e` 显式传异常对象;③结构化日志里异常栈应作为字段记录,便于检索;④敏感信息别随异常一起泄露到日志。总之:**记异常用 `log.exception`(带栈),不要用 `log.error(str(e))`(丢栈)**——完整 traceback 是定位线上问题的关键线索。

</details>
