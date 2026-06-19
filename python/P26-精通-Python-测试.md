# 精通 Python 测试

> 测试是工程质量的底线,Python 的 **pytest** 以简洁(裸 `assert`)、强大(fixture/参数化/插件)成为事实标准。面试问:"pytest fixture 是什么?""mock 怎么用?"本篇讲清现代 Python 测试,对照 [Go 测试 G18](../golang/INDEX.md)、[Java JUnit J32](../java/INDEX.md)。
>
> **📅 基准:2026 年 6 月,pytest 主流。**

---

## 一、pytest vs unittest

- **`unittest`**(标准库):仿 JUnit 的 xUnit 风格,要写 `class TestX(unittest.TestCase)`、用 `self.assertEqual` 等方法——样板多。
- **`pytest`**(第三方,**事实标准**):用**裸 `assert`** 语句、**普通函数**即可,自动发现测试,fixture/参数化/丰富插件生态。更简洁、更强大。

```python
# pytest:一个函数 + 裸 assert
def test_add():
    assert add(2, 3) == 5          # 失败时 pytest 自动显示两边的值(断言自省)
```

pytest 能运行 unittest 风格的用例(兼容),新项目基本都用 pytest。

---

## 二、断言自省与测试发现

- **断言自省(assertion introspection)**:pytest 重写 `assert`,失败时**自动打印表达式两边的实际值**(`assert x == y` 失败会显示 x、y 各是多少),无需像 unittest 那样记一堆 `assertEqual/assertTrue/assertIn`。
- **测试发现**:pytest 自动收集 `test_*.py`/`*_test.py` 文件里的 `test_*` 函数和 `Test*` 类——约定优于配置。
- 运行:`pytest`(全部)、`pytest tests/test_x.py::test_add`(单个)、`pytest -k "add"`(按名筛选)、`pytest -m slow`(按标记)。

---

## 三、fixture:依赖注入式的准备/清理

**fixture** 是 pytest 的核心——用依赖注入的方式提供测试所需的资源(数据、连接、临时目录),并自动清理:

```python
import pytest

@pytest.fixture
def db():                          # 定义 fixture
    conn = create_connection()
    yield conn                     # yield 前=准备,yield 后=清理(类似 with)
    conn.close()

def test_query(db):                # 参数名 = fixture 名,自动注入
    assert db.query("SELECT 1") == 1
```

- **声明依赖即注入**:测试函数把 fixture 名作为参数,pytest 自动调用 fixture 并注入返回值。
- **`yield` 分准备/清理**:yield 前是 setup、后是 teardown(像 [P06](./P06-精通-Python-装饰器与上下文管理器.md) 的上下文管理器)。
- **`scope`**:`function`(默认,每个测试一次)/`class`/`module`/`session`(整个会话一次,适合昂贵资源)。
- **`conftest.py`**:放共享 fixture,自动被同目录及子目录的测试发现,无需 import。

---

## 四、参数化测试

`@pytest.mark.parametrize` 一份逻辑、多组数据——避免复制粘贴(对照 [J32](../java/INDEX.md)):

```python
@pytest.mark.parametrize("a, b, expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected   # 3 组数据 = 3 个独立测试用例
```

每组参数生成一个独立用例(单独报告通过/失败)。用来高效覆盖**边界值、等价类、异常输入**,比写一堆相似 `test_*` 干净得多。

---

## 五、mock:隔离外部依赖

单元测试要隔离外部协作者(DB、网络、时间),用 `unittest.mock`:

```python
from unittest.mock import patch, MagicMock

def test_fetch_user():
    with patch("myapp.service.http_get") as mock_get:   # 替换掉真实的 http_get
        mock_get.return_value = {"id": 1, "name": "a"}   # 打桩:规定返回值
        user = get_user(1)
        assert user.name == "a"
        mock_get.assert_called_once_with("/users/1")     # 验证调用
```

- **`patch`**:临时替换某个对象/函数为 mock;**关键:patch 的是"被使用处"而非"定义处"**(`patch("调用方模块.名字")`,这是最常见的坑)。
- **`MagicMock`**:万能替身,`return_value` 打桩返回值、`side_effect` 模拟异常/多次不同返回、`assert_called_*` 验证调用。
- `pytest-mock` 提供更方便的 `mocker` fixture。
- **mock 什么**:外部的、慢的、有副作用的(网络/DB/时间);**不要 mock 一切**(否则测的是 mock),也别 mock 你不拥有的第三方内部。

---

## 六、覆盖率、异步与集成测试

```bash
pytest --cov=myapp --cov-report=term   # coverage:测试覆盖率
```

- **覆盖率(`pytest-cov`)**:看哪些代码被测到。**但高覆盖 ≠ 测得对**——重点测业务分支、边界、异常,别只追数字(见 [J32](../java/INDEX.md))。
- **`pytest-asyncio`**:测异步代码(`async def test_*` + `@pytest.mark.asyncio`),见 [P22](./P22-精通-Python-asyncio与协程.md)。
- **集成测试用真实依赖**:**Testcontainers**(Docker 拉起真实 DB/Redis,见 [J32](../java/INDEX.md))比 mock/SQLite 更真实;归到集成测试层(慢、需 Docker)。
- 测试金字塔:单元测试多而快(mock 隔离)、集成测试中等、E2E 少。

---

## 陷阱清单

- **mock patch 错位置**:patch 定义处而非使用处——`patch("被测模块.导入进来的名字")`,不是 `patch("原始模块.名字")`。最常见的 mock 坑。
- **过度 mock**:把被测逻辑也 mock 掉,测的是 mock 不是代码;只 mock 外部协作者。
- **测试间状态共享/顺序依赖**:用模块级可变全局、或依赖执行顺序——测试应独立;fixture 隔离。
- **只看覆盖率数字**:100% 覆盖可能全是无意义断言;重点测分支/边界/异常。
- **用 SQLite 替代生产 DB**:方言差异致"测过线上炸";集成测试用 Testcontainers 跑真实 DB。
- **不测异常路径**:只测 happy path,错误处理裸奔。用参数化覆盖。
- **fixture scope 用错**:昂贵资源用 function scope 每次重建,套件巨慢;用 session/module scope。
- **依赖时钟/随机**:导致偶发失败(flaky);mock 时间、固定随机种子。

---

## 2026 现状

- **pytest + pytest-cov + pytest-mock + pytest-asyncio** 是标准组合;unittest 仅遗留代码用。
- **Testcontainers-python** 在集成测试中流行(真实 DB/中间件),取代 SQLite/嵌入式假货。
- **Hypothesis(基于属性的测试)** 自动生成大量边界输入找反例,补充手写用例,渐受欢迎。
- **AI 辅助生成测试** 普及,但**断言正确性需人把关**(AI 易生成"覆盖了但没真正验证"的测试)。
- 与 Go/Java 对照:Go 测试([G18](../golang/INDEX.md))内建 `testing` + table-driven;Java 用 JUnit5 + Mockito([J32](../java/INDEX.md));pytest 的 fixture/参数化/裸 assert 更轻量灵活,理念相通(隔离、参数化、覆盖分支)。

---

## 练习题

1. pytest 相比标准库 unittest 有什么优势?

<details><summary>参考答案</summary>

①**更简洁的写法**:pytest 用**普通函数 + 裸 `assert` 语句**写测试(`def test_add(): assert add(2,3)==5`),不需要像 unittest 那样必须定义继承 `unittest.TestCase` 的类、用 `self.assertEqual/assertTrue/assertIn/...` 一大堆断言方法,样板代码大大减少。②**断言自省(assertion introspection)**:pytest 重写了 `assert`,当断言失败时会**自动、详细地打印出表达式两边的实际值和中间过程**(如 `assert x == y` 失败会显示 x 和 y 分别是什么、哪里不等),无需你手动选择特定断言方法或写失败消息,排错信息比 unittest 丰富得多。③**强大的 fixture 机制**:pytest 用**依赖注入式的 fixture**(测试函数声明需要的 fixture 作为参数即自动注入)来管理测试的准备和清理,支持不同作用域(function/module/session)、可组合、可放在 `conftest.py` 共享,比 unittest 的 `setUp/tearDown` 更灵活、复用性更好。④**参数化测试**:`@pytest.mark.parametrize` 让一份测试逻辑跑多组数据、每组生成独立用例,优雅覆盖边界/等价类。⑤**丰富的插件生态**:pytest-cov(覆盖率)、pytest-asyncio(异步)、pytest-mock、pytest-xdist(并行)、Hypothesis(属性测试)等,扩展能力强。⑥**自动测试发现**:按约定自动收集 `test_*.py` 里的 `test_*` 函数/`Test*` 类,运行灵活(`-k` 按名筛选、`-m` 按标记、指定单个用例)。⑦**兼容 unittest**:能直接运行 unittest 风格的用例,便于迁移。综上,pytest 以"约定优于配置 + 裸 assert + fixture + 参数化 + 插件"提供了远比 unittest 简洁强大的体验,是现代 Python 测试的**事实标准**;unittest 主要用于不想引入第三方依赖的场景或遗留代码。

</details>

2. pytest 的 fixture 是什么?yield、scope、conftest.py 各起什么作用?

<details><summary>参考答案</summary>

**fixture** 是 pytest 用来为测试**提供所需资源/前置条件并负责清理**的机制,采用**依赖注入**风格:你用 `@pytest.fixture` 装饰一个函数来定义 fixture(它准备并返回某个资源,如数据库连接、测试数据、临时目录、配置好的客户端等),然后测试函数**只要把该 fixture 的名字作为参数**,pytest 就会自动调用对应 fixture 并把其结果**注入**进来。这比 unittest 的 setUp/tearDown 更灵活:fixture 可被多个测试复用、可相互依赖组合、可按需选择。**`yield`**:在 fixture 里用 `yield` 可以把它写成"准备 + 清理"两段——**`yield` 之前的代码是 setup(准备资源)**,**`yield` 产出的值就是注入给测试的资源**,**`yield` 之后的代码是 teardown(清理,如关闭连接、删临时文件)**,且**无论测试是否失败,清理都会执行**(类似上下文管理器 `with` 的进入/退出,见 P06)。没有清理需求时直接 `return` 也行。**`scope`(作用域)**:控制 fixture 被创建/销毁的频率——`function`(默认,**每个测试函数**都重新创建一次,隔离性最好)、`class`(每个测试类一次)、`module`(每个测试模块一次)、`package`、`session`(**整个测试会话只创建一次**)。对**昂贵资源**(如启动一个数据库容器、建立连接池)应使用更大的 scope(如 session/module)以避免每个测试都重建导致套件极慢;对需要严格隔离、每次都要干净状态的用 function scope。**`conftest.py`**:一个特殊文件,放在测试目录(或其上层),用于定义**共享的 fixture(以及钩子、插件配置)**——它里面的 fixture 会被**同目录及所有子目录下的测试自动发现和使用,无需 import**。这让多个测试文件共用 fixture(如数据库连接、app 实例)变得很方便,也便于分层组织(不同目录的 conftest.py 提供不同层级的共享设施)。

</details>

3. 参数化测试 `@pytest.mark.parametrize` 解决什么问题?

<details><summary>参考答案</summary>

它解决的是**"同一份测试逻辑需要用多组不同的输入/期望来验证"时,避免复制粘贴大量几乎相同的测试函数**的问题。如果不用参数化,你可能要为每组数据写一个单独的 `test_xxx` 函数(`test_add_positive`、`test_add_zero`、`test_add_negative`…),它们结构雷同、只是数据不同,既冗余又难维护。`@pytest.mark.parametrize("参数名列表", [一组组数据])` 装饰一个测试函数后,pytest 会**用每一组数据各运行一次该测试,并把它们当作独立的测试用例**(各自单独报告通过/失败、单独显示是哪组数据失败)。例如 `@pytest.mark.parametrize("a,b,expected", [(2,3,5),(0,0,0),(-1,1,0)])` 配合 `def test_add(a,b,expected): assert add(a,b)==expected`,就用三组数据生成了三个独立用例。**价值**:①**消除重复**——一份逻辑覆盖多种情况,代码 DRY、易维护(改逻辑只改一处);②**高效覆盖边界值和等价类**——可以方便地把"正常值、零、负数、最大值、空、非法输入"等各种情形列成数据表,系统性地覆盖边界条件和异常路径(这正是测试该重点覆盖的地方);③**清晰的失败定位**——某组数据失败时,pytest 会明确指出是哪组参数导致的(可用 `ids` 给用例起名),而不是混在一个大测试里;④**可组合**——多个 parametrize 叠加会做笛卡尔积,也能参数化 fixture。它和 Go 的 table-driven 测试、JUnit5 的 `@ParameterizedTest`(见 J32)是同一理念:用数据驱动覆盖多种情况。建议:优先用参数化来覆盖边界/等价类,而不是写一堆相似的测试函数。

</details>

4. mock 是用来做什么的?`patch` 最常见的坑是什么?

<details><summary>参考答案</summary>

**mock(模拟对象)用于在单元测试中"替换掉被测代码所依赖的外部协作者"**,从而**隔离**被测单元、让测试快速、确定、可控。典型要 mock 的是:网络请求/HTTP 客户端、数据库访问、第三方服务、文件系统、当前时间/随机数、消息队列等——这些东西**慢、有副作用、不确定、或在测试环境不可用**。通过 mock,你可以:①**打桩(stub)**——用 `mock.return_value` 规定"当被调用时返回什么"(如让 `http_get` 返回预设的假数据),从而**控制被测代码走到你想测的分支**;②用 `side_effect` 模拟抛异常或多次调用返回不同值;③**验证交互**——用 `mock.assert_called_once_with(...)`、`assert_called_with`、`call_count` 等检查"被测代码是否、以什么参数、调用了该依赖几次"(用于验证副作用型行为,如"下单后应调用支付一次")。Python 用标准库 `unittest.mock`(`MagicMock`、`patch`),`pytest-mock` 提供更方便的 `mocker` fixture。**`patch` 最常见的坑是"patch 错了位置"——你应该 patch 的是"名字被使用(查找)的地方",而不是它"被定义的地方"。** 原因:`patch("a.b")` 替换的是 `a` 模块命名空间里的 `b` 这个名字。当被测模块用 `from othermod import func` 把 `func` 导入到自己的命名空间后,被测代码里调用的 `func` 实际是**被测模块命名空间中的那个引用**;所以要 `patch("被测模块.func")`(使用处),而不是 `patch("othermod.func")`(定义处)——patch 定义处不会影响被测模块里已经绑定的那个名字,导致 mock "不生效"、测试仍调用了真实函数。记忆口诀:**"patch where it's looked up / used, not where it's defined"**。其他注意:别**过度 mock**(把被测逻辑本身也 mock 了,就变成"测试 mock"而非测代码);只 mock 真正的外部边界;mock 你拥有/控制的接口,谨慎 mock 第三方库的内部实现(脆弱)。

</details>

5. 代码覆盖率高就说明测试好吗?应该重点测什么?

<details><summary>参考答案</summary>

**不是**。代码覆盖率(用 `pytest-cov`/coverage 统计的行覆盖、分支覆盖)只能说明"测试运行时**这些代码被执行过**",但**不能保证测试真正验证了行为的正确性**。常见的"高覆盖但低质量":①测试调用了代码却**几乎没有有意义的断言**(或只 `assert not None`),代码被执行、覆盖率上去了,但任何逻辑错误都发现不了;②**只覆盖了正常路径(happy path)**,没覆盖**边界值、异常分支、错误输入**——而 bug 往往恰恰藏在这些地方,所以即使行覆盖率很高,关键分支可能从未被真正"考验"过;③为凑覆盖率写一堆无价值的测试,增加维护负担却没保护作用。因此覆盖率是**有用但有局限的参考指标**:太低(如低于某阈值)肯定说明测试不足、可作为 CI 的底线门槛;但**不能把"100% 覆盖"当作目标或质量证明**。**应该重点测**:①**核心业务逻辑与各个分支**(if/else、循环边界、状态转移);②**边界条件和等价类**(空、None、0、负数、最大值、越界、首尾元素)——用参数化测试高效覆盖;③**异常/错误路径**(非法输入抛什么异常、依赖失败时如何降级/重试);④**关键的副作用交互**(该调用的调用了、不该调用的没调用,用 mock 的 assert 验证);⑤**回归测试**(每修复一个 bug 就补一个能复现它的测试)。衡量测试**有效性**还可以用**变异测试(mutation testing,如 mutmut/cosmic-ray)**——故意把代码改坏(制造变异体)看测试能否检测到失败,比覆盖率更能反映"测试是否真的在验证逻辑"。一句话:追求**有意义的断言去覆盖关键路径和边界/异常**,而非单纯的覆盖率数字(理念同 J32)。

</details>

6. 集成测试为什么推荐用 Testcontainers 而不是 SQLite/mock?

<details><summary>参考答案</summary>

核心是**真实性(环境一致性)**。在集成测试(验证你的代码与数据库/中间件等真实协作)中,用 **mock** 或用**轻量替身(如用 SQLite 冒充生产的 MySQL/PostgreSQL)**有明显问题:①**mock 不验证真实交互**——mock 掉数据库,你测的只是"在我假设的返回值下逻辑对不对",**完全测不到真实 SQL 是否正确、迁移脚本、约束、索引、事务、并发行为**等,而这些恰恰是集成测试该覆盖的;过度 mock 会让"集成测试"名存实亡。②**SQLite 与生产数据库行为不一致**——SQLite 与 MySQL/PostgreSQL 在 **SQL 方言、数据类型、函数、约束处理、并发/锁、特定语法(如 `ON CONFLICT`/`ON DUPLICATE KEY`、JSON 类型、窗口函数、序列)** 等方面存在差异,用 SQLite "测过"很可能**上线连真实库就出错**(典型的"在 SQLite 上绿、在生产 MySQL 上炸"),反之有些生产库能用的写法 SQLite 不支持,导致测试覆盖不到真实问题。**Testcontainers** 用 **Docker 在测试运行时拉起一个与生产同款的真实数据库/中间件容器**(如 `mysql:8.0`、`postgres:16`、Redis、Kafka 等),让集成测试**连真实容器跑**,测完自动销毁。这样:①测试运行在**和生产几乎一致的真实环境**上,能发现方言、迁移、真实约束、事务等问题,可信度高;②每次用**全新、隔离的容器**,可重复、互不干扰;③适用于几乎所有有官方 Docker 镜像的依赖。**代价**:需要 Docker 环境、首次拉镜像和启动容器比内嵌库慢——所以它属于**集成测试层**(用更大的 fixture scope 复用容器、在 CI 的合并前/夜间阶段跑),不放进每次都飞快跑的纯单元测试里。**分工**:纯单元测试用 mock 隔离外部依赖(快、聚焦逻辑);集成测试用 Testcontainers 跑真实依赖(真实、可信);二者互补,构成测试金字塔(单元多、集成中、E2E 少)。这与 Java 生态(JUnit + Testcontainers,见 J32)的最佳实践一致。

</details>
