# 精通 Python 数据库访问与 ORM

> 从底层 DB-API 到 SQLAlchemy ORM,Python 的数据库访问要点是:防 SQL 注入、用连接池、避开 N+1、按需异步。面试问:"怎么防注入?""N+1 是什么?"本篇讲清,对照 [Go 数据库访问 G29](../golang/INDEX.md)、[Java MyBatis J27](../java/INDEX.md)、[数据库专题](../mysql/INDEX.md)。
>
> **📅 基准:2026 年 6 月,SQLAlchemy 2.0、异步驱动主流。**

---

## 一、DB-API 2.0:统一的底层接口

**DB-API 2.0(PEP 249)** 是 Python 数据库驱动的**统一标准接口**——各数据库的驱动(psycopg/PostgreSQL、PyMySQL/MySQL、sqlite3)都实现它,所以用法一致:

```python
import sqlite3
conn = sqlite3.connect("app.db")    # 连接
cur = conn.cursor()                  # 游标
cur.execute("SELECT * FROM users WHERE id = ?", (uid,))  # 执行(参数化!)
row = cur.fetchone()                 # 取结果
conn.commit()                        # 提交事务
conn.close()
```

核心对象:**Connection**(连接、管理事务)、**Cursor**(执行 SQL、取结果)。`execute`/`executemany`、`fetchone`/`fetchall`、`commit`/`rollback` 是通用方法。

---

## 二、参数化查询:防 SQL 注入

**绝对不要用字符串拼接构造 SQL**——这是 SQL 注入的根源(见 [B23 OWASP](../backend/INDEX.md)):

```python
# ❌ 致命:字符串拼接 → SQL 注入
cur.execute(f"SELECT * FROM users WHERE name = '{name}'")
# 攻击者传 name = "' OR '1'='1" 就能拖库

# ✅ 参数化查询:占位符 + 参数分开传
cur.execute("SELECT * FROM users WHERE name = ?", (name,))      # sqlite/mysql
cur.execute("SELECT * FROM users WHERE name = %s", (name,))     # psycopg
```

**参数化查询**把 SQL 模板和数据分开传给驱动,驱动负责安全转义——**数据永远被当作数据、不会被解释成 SQL**,从根本上杜绝注入。占位符风格随驱动不同(`?`/`%s`/`:name`)。**这是数据库安全的第一铁律。**

---

## 三、连接池

每次查询新建连接代价高(TCP + 认证握手)。**连接池**预先维护一组连接、复用它们:

- 减少连接建立/销毁开销、控制最大连接数(保护数据库,见 [B20 韧性](../backend/INDEX.md))。
- SQLAlchemy 内置连接池;或用 `psycopg_pool`、PgBouncer(外部连接池)。
- 关键参数:池大小、最大溢出、连接超时、回收时间(防陈旧连接);Web 应用必配。

类比 [Java HikariCP](../java/INDEX.md)、[Go database/sql 连接池](../golang/INDEX.md)——都是同一思想:复用连接、限制并发、健康管理。

---

## 四、SQLAlchemy:Core 与 ORM

**SQLAlchemy** 是 Python 事实标准的数据库工具,两层:

- **Core**:SQL 表达式语言——用 Python 对象构造 SQL,比裸字符串安全(自动参数化)、可组合,但仍是"面向 SQL"。
- **ORM**:对象关系映射——把**数据库表映射成 Python 类、行映射成对象**,用对象操作数据,自动生成 SQL:

```python
from sqlalchemy.orm import Session
# 定义模型(类 ↔ 表)
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str]

with Session(engine) as session:
    user = session.get(User, 1)          # 按主键查,返回对象
    user.name = "new"                     # 改对象
    session.add(User(name="alice"))       # 新增
    session.commit()                      # 工作单元:统一 flush + 提交
```

- **Session(工作单元 Unit of Work)**:跟踪对象变更,`commit` 时统一生成 SQL 持久化;管理身份映射(同一行同一对象)。
- ORM 提升开发效率、面向对象;但**隐藏了 SQL**,不留意会产生低效查询(见 N+1)。

---

## 五、N+1 问题

ORM 最经典的性能陷阱:**懒加载(lazy loading)在循环里逐条触发查询**:

```python
# ❌ N+1:1 次查 users + 对每个 user 各查 1 次 orders = 1+N 次查询
users = session.query(User).all()        # 1 次
for u in users:
    print(u.orders)                        # 每次访问触发 1 次查询!N 次

# ✅ 预加载(eager loading):用 JOIN/IN 一次性把关联查出来
from sqlalchemy.orm import selectinload, joinedload
users = session.query(User).options(selectinload(User.orders)).all()  # 1-2 次
```

- **N+1**:查 N 条主记录,又为每条各查一次关联 → 总共 1+N 次查询,数据库被打爆。
- **解决**:**eager loading(预加载)**——`joinedload`(JOIN 一次拉)、`selectinload`(用 IN 批量拉,常更优)一次/两次查询搞定。
- 这是 ORM 跨语言的通病(见 [Java N+1 / MyBatis J27](../java/INDEX.md)、[B13 N+1](../backend/INDEX.md))——用 ORM 必须警惕。

---

## 六、异步数据库与迁移

- **异步驱动**(配合 asyncio/FastAPI,见 [P22](./P22-精通-Python-asyncio与协程.md)/[P28](./P28-精通-Python-Web框架与WSGI-ASGI.md)):**asyncpg**(PostgreSQL,高性能)、**SQLAlchemy 2.0 async**(`AsyncEngine`/`AsyncSession` + `await`)。异步视图里**必须用异步驱动**,同步驱动会阻塞事件循环(见 [P23](./P23-精通-Python-异步实战与陷阱.md))。
  ```python
  async with AsyncSession(async_engine) as session:
      user = await session.get(User, 1)
  ```
- **迁移(schema 版本管理)**:**Alembic**(SQLAlchemy 配套)——把表结构变更写成可版本化、可升级/回滚的迁移脚本(类比 Django migrations、[Java Flyway](../java/INDEX.md))。
- **事务**:用 `with session.begin()`/上下文管理保证提交或回滚;注意事务边界与隔离级别(见 [B09 事务](../backend/INDEX.md)/[MySQL](../mysql/INDEX.md))。

---

## 陷阱清单

- **字符串拼接 SQL**:SQL 注入,致命。永远用**参数化查询**(占位符 + 参数)。
- **N+1 查询**:ORM 懒加载在循环里逐条查;用 `selectinload`/`joinedload` 预加载。
- **不用连接池**:每请求新建连接拖慢、压垮 DB;用连接池并限大小。
- **异步视图用同步驱动**:阻塞事件循环(见 [P23](./P23-精通-Python-异步实战与陷阱.md));用 asyncpg/SQLAlchemy async。
- **Session 生命周期/线程共享乱**:Session 不是线程安全的,别跨线程/请求共享;每请求一个、用完关闭(scoped_session/依赖注入)。
- **连接/游标泄漏**:忘记 close/commit;用 `with` 上下文管理。
- **一次性 fetchall 大结果集**:撑爆内存;用流式/分批(server-side cursor、yield_per)。
- **事务边界不清**:忘 commit/rollback、长事务持锁;明确事务范围。

---

## 2026 现状

- **SQLAlchemy 2.0**(统一 Core/ORM API、原生 async、类型注解 `Mapped[...]`)是事实标准;**Alembic** 做迁移。
- **异步栈**:FastAPI + SQLAlchemy async + asyncpg 是高并发服务典型组合;同步项目仍大量用同步 SQLAlchemy/psycopg。
- **Pydantic + SQLAlchemy**:Pydantic(见 [P14](./P14-精通-Python-类型注解与typing.md))做 API 出入参校验、SQLAlchemy 做持久化,分工明确(SQLModel 尝试合二为一)。
- **连接池**:云数据库前常加 **PgBouncer** 等外部池;Serverless 场景注意连接数。
- 与 Go/Java 对照:Go 用 `database/sql` + sqlx/sqlc/GORM([G29](../golang/INDEX.md));Java 用 JDBC + MyBatis([J27](../java/INDEX.md))/JPA + HikariCP——**参数化防注入、连接池、N+1、事务**是跨语言共通的核心问题。

---

## 练习题

1. 什么是 DB-API?它有什么意义?

<details><summary>参考答案</summary>

**DB-API 2.0(PEP 249)** 是 Python 官方制定的**数据库访问的标准接口规范**——它定义了 Python 程序与关系数据库交互时应有的**统一 API**(对象、方法、行为约定),要求各数据库的驱动库都按这个规范来实现。具体规定了:**Connection 对象**(代表连接,负责 `commit()`/`rollback()` 管理事务、`cursor()` 创建游标、`close()` 关闭)、**Cursor 对象**(负责 `execute(sql, params)`/`executemany()` 执行 SQL、`fetchone()`/`fetchmany()`/`fetchall()` 取结果)、参数占位风格、异常层次、类型映射等。**意义**:带来**可移植性与一致性**——PostgreSQL 的 psycopg、MySQL 的 PyMySQL、内置 sqlite3 等各种驱动都实现同一套 DB-API,所以用基本一致的代码风格(connect→cursor→execute→fetch→commit→close)就能操作不同数据库,切换数据库时上层改动小。它也是**上层抽象(SQLAlchemy/ORM)的基础**——这些工具底层正是通过 DB-API 驱动与数据库通信。注意:DB-API 只统一"接口形状",**SQL 方言本身仍因库而异**(语法、占位符 `?`/`%s`/`:name` 不同),完全的数据库无关还需 ORM/查询构建器封装方言。总之它是 Python 数据库生态的底层契约,保证驱动统一用法和上层工具可插拔。

</details>

2. 为什么必须用参数化查询?字符串拼接 SQL 有什么危险?

<details><summary>参考答案</summary>

必须用参数化查询是为了**防止 SQL 注入**(OWASP Top 10 级别的严重漏洞,见 B23)。**字符串拼接 SQL 的危险**:把用户输入直接拼进 SQL,如 `f"SELECT * FROM users WHERE name = '{name}'"`,用户输入会被**当作 SQL 的一部分解释执行**。攻击者构造特殊输入即可篡改查询:传 `name = "' OR '1'='1"` 使条件恒真、**返回全表(拖库)**;用 `'; DROP TABLE users; --` 之类可**删表、改数据、绕过认证、读取其他表**,造成数据泄露/篡改/破坏。**参数化查询如何防注入**:写带**占位符**的 SQL 模板(`"... WHERE name = ?"`/`%s`/`:name`),把值作为**单独参数**传给 `execute(sql, (value,))`;驱动把 SQL 结构与参数**分开处理**——SQL 先解析/预编译,参数值只作为"数据"被安全绑定、由驱动正确转义,**绝不会被当作 SQL 代码**。因此无论输入什么(引号、OR、分号)都只是普通字符串值,无法改变查询结构,从根本上杜绝注入。**铁律**:**永远不要用拼接/格式化把外部数据放进 SQL**,一律参数化;ORM/查询构建器默认参数化(安全优势之一);确需动态拼表名/列名时用严格白名单。

</details>

3. 什么是 ORM?SQLAlchemy 的 Session(工作单元)是做什么的?

<details><summary>参考答案</summary>

**ORM(对象关系映射)**把**数据库的表/行/列**与**面向对象的类/对象/属性**相互映射:定义 Python 类对应表(实例↔行、属性↔列),用操作对象的方式读写数据库,ORM 底层**自动生成执行 SQL**。好处:面向对象、提升效率、屏蔽部分方言差异、自动参数化(防注入)、便于建模关联;代价:隐藏 SQL,不留意会产生低效查询(N+1)、复杂查询有时不如手写。Python 事实标准是 **SQLAlchemy**。**Session 实现"工作单元(Unit of Work)"模式**,职责:①**跟踪对象变更**——你在 Session 里 add/修改/delete 对象,它记录所有变更(脏对象),**不立即逐条发 SQL**;②**统一持久化**——`commit()`(或 flush)时把累积变更**作为一个工作单元、按正确顺序统一生成执行 SQL**(INSERT/UPDATE/DELETE)并提交事务,批量有序、减少往返、保证一致;③**身份映射(Identity Map)**——同一 Session 内数据库同一行只对应同一个 Python 对象(查两次同主键返回同一实例),避免重复/不一致;④管理**事务边界**和从连接池借连接。用法:Session **非线程安全**,应**每请求/每工作单元一个、用完关闭**(`with Session(engine) as session:` 或依赖注入/scoped_session),不要跨线程/长期共享。

</details>

4. 什么是 N+1 查询问题?如何解决?

<details><summary>参考答案</summary>

**N+1 查询**是 ORM 的经典性能陷阱,根源是**懒加载(lazy loading)**:查出 N 条主记录后,在循环里访问每条的**关联对象**时,ORM 为**每条**主记录**单独再发一次查询**加载其关联,于是总共 **1(查 N 条主记录)+ N(每条各查关联)= N+1 次**查询。例如 `users = query(User).all()`(1 次)后 `for u in users: u.orders`——每次访问 `u.orders` 触发一次 SQL,N 个用户就 N 次。N 大时产生成百上千次小查询,巨大往返开销,拖慢甚至打垮数据库(本可一两条 SQL 解决)。**解决:预加载(eager loading)**——查主记录时就一次性(或少数几次)把关联也加载进来。SQLAlchemy 提供:①**`joinedload`**——用 **JOIN** 在一条 SQL 里连带查出(适合一对一/小数据,一对多 JOIN 有行膨胀);②**`selectinload`**——先查主记录、再用一条 `WHERE id IN (...)` 批量查关联(约 2 次查询,一对多/多对多通常更优、无行膨胀,常用推荐);用法 `query(User).options(selectinload(User.orders))`;还有 `subqueryload` 等,也可手写 JOIN 或调整关联加载策略。**要点**:警惕"循环里访问关联属性",主动用 eager loading 把 N+1 降到 1~2 次;用 ORM 也要懂它生成的 SQL、开 SQL 日志检查查询次数。这是跨语言/跨 ORM 通病(见 J27/B13)。

</details>

5. 在异步(FastAPI)应用里访问数据库要注意什么?

<details><summary>参考答案</summary>

核心:**异步应用里必须用异步数据库驱动/接口,绝不能用同步阻塞调用**,否则阻塞事件循环、毁掉并发(见 P22/P23)。①**用异步驱动 + 异步 ORM**——`async def` 视图跑在 asyncio 事件循环上,DB 访问用**异步驱动**(PostgreSQL 的 **asyncpg**)配 **SQLAlchemy 2.0 异步接口**(`create_async_engine`、`AsyncSession`、`await session.execute/get`),这些查询 awaitable,会在等数据库时让出控制权给事件循环处理其他请求。②**绝不在 async 视图用同步 DB 调用**——同步驱动/查询会卡住事件循环线程,期间所有请求被堵(一个慢查询拖垮全服务);不得已的同步调用要用 `asyncio.to_thread`/`run_in_executor` 丢线程池,或把视图写成同步 `def` 让 FastAPI 放线程池跑(见 P28)。③**异步连接池**——用异步引擎自带的池、合理设大小;注意数据库最大连接数(Serverless/多实例常加 PgBouncer)。④**Session 生命周期**——AsyncSession 不可随意跨任务共享,**每请求一个**,用 FastAPI 依赖注入(`Depends`)创建/关闭(配 `async with`)。⑤**事务**——`async with session.begin():` 明确边界、避免长事务持锁。⑥**仍要防 N+1 和注入**——参数化(ORM 默认)、`selectinload`/`joinedload` 预加载。⑦避免一次性加载超大结果集(流式/分批)。总之:全链路异步 + 每请求一 session + 异步连接池 + 防 N+1/注入 + 明确事务。

</details>

6. Python 的数据库访问方式和 Java、Go 相比有什么异同?

<details><summary>参考答案</summary>

**相同的核心问题与思想(跨语言共通)**:①**统一驱动接口**——Python 有 DB-API、Java 有 **JDBC**、Go 有 **`database/sql`**,都是"标准接口 + 各驱动实现";②**参数化防注入**——都强调占位符 + 参数(JDBC 的 PreparedStatement、Go 的 `?` 占位、Python 的 `execute(sql, params)`),绝不拼接;③**连接池**——都用池复用连接、限并发(Java 的 **HikariCP**、Go 内置池、Python SQLAlchemy 池/PgBouncer);④**ORM 与 N+1**——都有 ORM/数据映射且都有 N+1 陷阱:Java 有 **MyBatis(SQL 映射,见 J27)/Hibernate-JPA**、Go 有 **GORM/sqlx/sqlc**、Python 有 **SQLAlchemy/Django ORM**;⑤**事务与迁移**——都需管理事务边界和 schema 迁移(Flyway/Alembic/Django migrations)。**差异**:①**ORM 取向**——Go 社区偏好**轻量、贴近 SQL**(sqlc 编译期由 SQL 生成类型安全代码、sqlx 薄封装),Java 既有贴近 SQL 的 MyBatis 也有重 ORM 的 Hibernate,Python 的 SQLAlchemy 提供 Core+ORM 两层、成熟全面;②**并发/异步模型(最大不同)**——**Go 的 `database/sql` 配 goroutine 天然高并发**(同步写法、运行时并行、池自动管理,见 G29),不分同步/异步;**Java** 传统同步阻塞 + 线程池,JDBC 是阻塞 API(有 R2DBC 响应式,虚拟线程 J30 让阻塞写法也高并发);**Python 因 GIL/asyncio** 需显式区分**同步驱动**(psycopg)与**异步驱动**(asyncpg + SQLAlchemy async),异步应用必须全链路异步否则阻塞事件循环,模型更分裂、心智更高;③**类型安全**——Go 的 sqlc 编译期类型安全、Java 强类型 JPA、Python 靠运行时 + 类型注解(`Mapped[...]`、Pydantic)。**总结**:三者在"标准接口、参数化、连接池、ORM 与 N+1、事务迁移"上理念一致(核心都要掌握,见 mysql/B 系列);主要差异在**并发模型**(Go 同步即高并发最省心,Java 线程池/虚拟线程,Python 需同步/异步分治)与 **ORM 抽象取向**上。

</details>
