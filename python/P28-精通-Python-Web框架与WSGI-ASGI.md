# 精通 Python Web 框架与 WSGI/ASGI

> Python Web 生态有三巨头:Flask(微框架)、Django(全栈)、FastAPI(异步 + 类型驱动)。理解它们的前提是 **WSGI(同步)vs ASGI(异步)**两套网关接口。面试问:"WSGI 和 ASGI 区别?""FastAPI 为什么快?"本篇讲清,串起 [P14 类型注解](./P14-精通-Python-类型注解与typing.md)、[P22 asyncio](./P22-精通-Python-asyncio与协程.md),对照 [Go net/http G27](../golang/INDEX.md)、[Spring MVC J25](../java/INDEX.md)。
>
> **📅 基准:2026 年 6 月,FastAPI/ASGI 主流,Pydantic v2。**

---

## 一、WSGI vs ASGI

Web 框架与服务器之间靠**标准网关接口**对接,Python 有两套:

- **WSGI(PEP 3333)**:**同步**接口——一个 `app(environ, start_response)` 可调用。一请求一(线程/进程),处理期间阻塞。Flask、Django(传统)是 WSGI 应用,跑在 Gunicorn/uWSGI 上。
- **ASGI**:**异步**接口——支持 `async`、WebSocket、HTTP/2、长连接。FastAPI、Starlette 是 ASGI 应用,跑在 Uvicorn/Hypercorn 上。

```
浏览器 → Web服务器(Gunicorn/Uvicorn) → [WSGI/ASGI 接口] → 框架app → 你的视图
```

**根本区别**:WSGI 同步、模型简单但高并发 IO 受线程数限制;ASGI 异步、能用单线程协程([P22](./P22-精通-Python-asyncio与协程.md))扛海量并发连接、支持 WebSocket 等。

---

## 二、Flask:微框架(WSGI)

Flask 是**轻量微框架**——核心小、按需加扩展,灵活:

```python
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.route("/users/<int:uid>")        # 路由(装饰器,见 P06)
def get_user(uid):
    return jsonify({"id": uid})
```

- **同步、WSGI**:视图是普通同步函数;高并发 IO 靠多进程/多线程 worker。
- **微核 + 扩展**:路由、请求/响应是核心;ORM、表单、认证靠 Flask-SQLAlchemy 等扩展自由组合。
- 适合中小项目、API、需要高度自定义的场景。

---

## 三、Django:全栈框架

Django 是**全功能、约定优于配置**的全栈框架——"自带电池":

- **内置 ORM**(模型即数据库,见 [P29](./P29-精通-Python-数据库访问与ORM.md))、**Admin 后台**(自动生成管理界面)、表单、认证、模板、迁移、中间件——开箱即用。
- 传统**同步(WSGI)**,但 3.0+ 起逐步支持 **async 视图/ORM**(ASGI)。
- 适合内容驱动、需要后台管理、快速搭建完整应用的项目(电商、CMS、SaaS)。
- 哲学:大而全、约定多;开发快但相对"重"、灵活性不如 Flask。

---

## 四、FastAPI:异步 + 类型驱动

**FastAPI**(基于 Starlette + Pydantic)是 2026 的明星——**异步(ASGI)+ 类型注解驱动**:

```python
from fastapi import FastAPI
from pydantic import BaseModel
app = FastAPI()

class User(BaseModel):                  # Pydantic 模型(见 P14)
    id: int
    name: str

@app.post("/users")
async def create_user(user: User):      # 类型注解 → 自动校验请求体
    return user                          # 返回自动序列化为 JSON
```

杀手锏(都来自**类型注解 + Pydantic**,见 [P14](./P14-精通-Python-类型注解与typing.md)):
- **自动请求校验**:按类型注解校验/转换请求参数、body,不合法自动返回 422 + 详细错误。
- **自动 API 文档**:据类型生成 OpenAPI/Swagger UI,免写文档。
- **异步原生**:`async def` 视图,配合 asyncio 生态(asyncpg/httpx)扛高并发 IO。
- **高性能**:Starlette(ASGI)+ Pydantic v2(Rust 核心)。

适合现代 API 服务、微服务、AI 后端(LLM 应用首选,见 [ai-backend](../ai-backend/INDEX.md))。

---

## 五、部署:多进程 + 异步

生产部署的典型形态(对照 [P24 并发选型](./P24-精通-Python-并发选型与free-threading.md)):

- **WSGI 应用(Flask/Django)**:用 **Gunicorn** 起**多个 worker 进程**(吃满多核,绕 GIL),每 worker 同步处理;worker 数 ≈ 核数×2+1(IO 密集可更多),或用 gthread/gevent worker。
- **ASGI 应用(FastAPI)**:用 **Uvicorn**(单进程异步事件循环);生产用 **Gunicorn 管理多个 Uvicorn worker**(`gunicorn -k uvicorn.workers.UvicornWorker`)——**多进程吃多核 + 每进程内 asyncio 扛高并发 IO**,是高并发 Python 服务的黄金组合。
- 前面再放 Nginx 反向代理(静态资源、TLS、负载),见 [B25](../backend/INDEX.md)。

```bash
gunicorn app:app -w 4 -k uvicorn.workers.UvicornWorker   # 4 进程 × 异步
```

---

## 六、选型

| 框架 | 接口 | 定位 | 适合 |
|---|---|---|---|
| **Flask** | WSGI(同步) | 微框架、灵活 | 中小项目、自定义 API |
| **Django** | WSGI(+async) | 全栈、自带电池 | 内容/后台驱动的完整应用 |
| **FastAPI** | ASGI(异步) | 类型驱动、高性能 | 现代 API、微服务、AI 后端 |

- **新建 API / 高并发 / AI 后端** → FastAPI(异步 + 自动校验/文档)。
- **完整 Web 应用 + 后台管理 + 快速** → Django。
- **轻量、灵活、可控** → Flask。
- 别在 WSGI 同步视图里写 async 期待高并发(无益);别在 FastAPI 异步视图里做阻塞调用(毁事件循环,见 [P23](./P23-精通-Python-异步实战与陷阱.md))。

---

## 陷阱清单

- **在 FastAPI/异步视图里做同步阻塞调用**:`requests`、同步 DB、`time.sleep` 卡死事件循环(见 [P22](./P22-精通-Python-asyncio与协程.md)/[P23](./P23-精通-Python-异步实战与陷阱.md));用异步库或 `run_in_executor`。
- **以为换了 ASGI 就自动高并发**:还要全链路异步 + 不阻塞,否则等于白换。
- **WSGI 单进程跑生产**:GIL 下单进程吃不满多核;用 Gunicorn 多 worker。
- **Django/Flask 同步视图写 async**:WSGI 下 async 视图无并发收益。
- **N+1 查询**:ORM 懒加载在循环里逐条查(见 [P29](./P29-精通-Python-数据库访问与ORM.md));用 eager loading。
- **不加超时/限流**:下游慢拖垮服务;加超时、限并发(见 [P23](./P23-精通-Python-异步实战与陷阱.md)、[B21](../backend/INDEX.md))。
- **同步框架做 WebSocket/长连接**:WSGI 不适合;用 ASGI。
- **worker 数瞎配**:CPU/IO 密集 worker 数策略不同;压测定。

---

## 2026 现状

- **FastAPI + ASGI 是新 API 服务的主流**,尤其 **AI/LLM 后端**(FastAPI + Pydantic 校验 + SSE 流式,见 [ai-backend](../ai-backend/INDEX.md))几乎是标配。
- **Pydantic v2(Rust 核心)** 大幅提速校验/序列化;FastAPI 性能进一步提升。
- **Django** 持续增强 async(异步 ORM/视图)以应对现代需求,仍是全栈/后台首选。
- **部署**:Gunicorn + UvicornWorker 多进程异步是黄金组合;容器化 + K8s(见 [cloud-native](../cloud-native/INDEX.md))。
- 与 Go/Java 对照:Go 的 net/http([G27](../golang/INDEX.md))原生并发(goroutine,无需区分 sync/async);Java Spring MVC([J25](../java/INDEX.md))线程池 + 虚拟线程([J30](../java/INDEX.md))。Python 要靠 ASGI/asyncio + 多进程来达到类似并发,模型更分裂。

---

## 练习题

1. WSGI 和 ASGI 有什么区别?为什么会出现 ASGI?

<details><summary>参考答案</summary>

**WSGI(Web Server Gateway Interface,PEP 3333)**和 **ASGI(Asynchronous Server Gateway Interface)**都是定义**Web 服务器与 Python Web 应用/框架之间如何对接**的标准接口,使得框架(Flask/Django/FastAPI)和服务器(Gunicorn/Uvicorn)可以解耦、自由组合。**WSGI 是同步的**:它定义应用为一个同步可调用对象 `app(environ, start_response)`,服务器为每个请求调用它、应用同步处理并返回响应,处理期间是**阻塞**的;并发靠服务器开多个进程/线程(一请求一线程/进程)。WSGI 简单成熟,但①**受同步阻塞模型限制**——高并发 IO 时要靠大量线程/进程,扛不住超高并发(线程开销、GIL,见 P17/P20);②**不支持长连接/WebSocket/HTTP2/SSE 等需要异步、双向、持久连接的场景**(WSGI 的请求-响应模型是一次性的)。**ASGI 是异步的**:它定义应用为一个 **`async` 可调用对象**(`async def app(scope, receive, send)`),原生支持 `async/await`,能用单线程协程(asyncio,见 P22)**高效处理海量并发连接**,并且支持 **WebSocket、HTTP/2、Server-Sent Events、长连接、后台任务**等。**为什么出现 ASGI**:随着实时通信(WebSocket、推送)、流式响应(SSE,如 LLM 流式输出)、以及超高并发 IO(海量连接的 API/网关)的需求增长,WSGI 的同步、一次性请求-响应模型不够用了——它无法优雅支持这些异步/长连接场景,也难以用少量资源扛高并发。ASGI 作为 WSGI 的"异步继任者"应运而生,既兼容传统 HTTP 请求-响应,又支持异步和各种现代协议。**生态**:WSGI 应用(Flask、传统 Django)跑在 Gunicorn/uWSGI 上;ASGI 应用(FastAPI、Starlette、现代 Django async)跑在 Uvicorn/Hypercorn 上;ASGI 还能通过适配器运行 WSGI 应用以平滑过渡。简言之:**WSGI=同步、简单、成熟但受限;ASGI=异步、支持高并发与现代协议,是面向未来的选择。**

</details>

2. Flask、Django、FastAPI 各自的定位和适用场景是什么?

<details><summary>参考答案</summary>

①**Flask**——**轻量级"微框架"(microframework)**,WSGI(同步)。它的核心非常小,只提供路由、请求/响应处理、模板等最基础的能力,其余功能(ORM、表单、认证、迁移等)都通过**扩展(Flask-SQLAlchemy、Flask-Login 等)按需自由组合**。哲学是"最小核心 + 高度灵活/可控"。**适合**:中小型项目、需要高度自定义架构的应用、轻量 API、原型、以及你想自己掌控技术选型的场景。代价是大项目要自己组装和维护较多组件。②**Django**——**全功能、"自带电池(batteries-included)"的全栈框架**,WSGI 为主(3.0+ 逐步支持 async/ASGI)。它内置了**强大的 ORM、自动生成的 Admin 后台管理界面、表单系统、用户认证、模板引擎、数据库迁移、中间件、安全防护**等几乎一切常用功能,遵循"约定优于配置"。**适合**:内容/数据驱动、需要后台管理界面、追求快速搭建完整功能应用的项目(CMS、电商、SaaS、企业后台)。优点是开发快、规范统一、生态成熟;代价是相对"重"、约定多、灵活性不如 Flask。③**FastAPI**——**现代的、异步(ASGI)、类型注解驱动的高性能 API 框架**(基于 Starlette + Pydantic)。它用 Python 类型注解(见 P14)自动完成**请求/响应数据校验与序列化、自动生成 OpenAPI/Swagger 交互文档**,原生支持 `async/await` 高并发 IO,性能优秀(Pydantic v2 用 Rust)。**适合**:现代 RESTful API、微服务、高并发 IO 服务、以及 **AI/LLM 后端**(配合 SSE 流式、Pydantic 校验,几乎是该领域标配,见 ai-backend)。**选型小结**:要**完整 Web 应用 + 后台 + 快速** → Django;要**现代 API / 高并发 / AI 后端 / 自动文档** → FastAPI;要**轻量、灵活、可控的中小项目/API** → Flask。三者也可按需混用(如用 FastAPI 写 API、Django 做后台)。

</details>

3. FastAPI 为什么能自动做请求校验和生成文档?

<details><summary>参考答案</summary>

因为 FastAPI 深度利用了 **Python 的类型注解(type hints,见 P14)+ Pydantic** 来"读懂"你的接口契约,并据此自动完成校验、转换、序列化和文档生成。机制:①你在路由处理函数里用**类型注解**声明参数和请求体的类型——路径参数、查询参数用普通类型注解(`uid: int`),请求体用 **Pydantic 模型**(继承 `BaseModel`,字段带类型注解,如 `class User(BaseModel): id: int; name: str`)。②FastAPI 在启动和处理请求时**读取这些注解**:对请求进来的数据,它用 **Pydantic 在运行时按注解进行校验和类型转换**——把 JSON body 解析成对应的模型实例、把字符串路径参数转成 int 等;如果数据缺字段、类型不符或不满足约束,Pydantic 会抛出校验错误,FastAPI **自动返回 422 状态码和结构化的、详细的错误信息**(指出哪个字段错了、为什么),你完全不用手写校验代码。③同样基于这些类型信息,FastAPI **自动推导出每个接口的输入/输出 schema**,生成符合 **OpenAPI** 规范的 API 描述,并据此提供**交互式文档**(Swagger UI / ReDoc,访问 `/docs`),前端/调用方可以直接看文档、试接口——文档与代码**始终同步**(因为都来自同一份类型注解),免去了手写和维护文档的负担。④返回值也按注解(`response_model`)自动**序列化成 JSON** 并校验/过滤字段。这就是 FastAPI 的核心理念——"**类型注解即契约**":一份类型声明同时驱动了运行时校验、数据转换、序列化和文档生成,既减少样板代码、又保证一致性和正确性。底层 Pydantic v2 用 Rust 实现,使这些校验/序列化非常快。这也是为什么 FastAPI 在需要严谨数据校验的 API 和 AI 后端(请求/响应结构复杂)场景特别受欢迎。

</details>

4. 生产部署 FastAPI/Flask 通常怎么做?为什么这样?

<details><summary>参考答案</summary>

核心思路是**用多进程吃满多核(绕过 GIL)+ 适配各自的同步/异步模型 + 前置反向代理**。①**WSGI 应用(Flask、传统 Django)**:用 **Gunicorn**(或 uWSGI)作为 WSGI 服务器,启动**多个 worker 进程**(`gunicorn app:app -w N`)。因为每个视图是同步阻塞处理、且 GIL 使单进程多线程无法并行(见 P17),所以靠**多进程**来利用多核并发处理请求;worker 数经验上 ≈ `CPU核数 × 2 + 1`(IO 密集可更多),也可用线程型 worker(gthread)或协程型(gevent)提升单 worker 的并发。②**ASGI 应用(FastAPI、Starlette、Django async)**:用 **Uvicorn**(基于 uvloop 的高性能 ASGI 服务器)运行异步事件循环。单个 Uvicorn 进程是单事件循环(单核),为了**同时利用多核**,生产通常用 **Gunicorn 来管理多个 Uvicorn worker 进程**:`gunicorn app:app -k uvicorn.workers.UvicornWorker -w N`。这就形成了高并发 Python 服务的**黄金组合——"多进程(每进程一个事件循环,吃满多核)+ 每个进程内用 asyncio 协程扛海量并发 IO"**:多进程解决多核并行(绕 GIL),asyncio 解决单核内的高并发 IO(见 P24)。③**前置反向代理**:在应用服务器前面再放 **Nginx**(或云负载均衡)处理 TLS 终止、静态文件、压缩、负载均衡、限流等(见 B25)。④**容器化部署**:打包成 Docker 镜像,用 Kubernetes 编排、扩缩容、健康检查(见 cloud-native)。**为什么这样**:因为 Python 的 GIL 限制了单进程的 CPU 并行,必须靠多进程才能用满多核;而 IO 并发则用线程(WSGI)或 asyncio(ASGI)在每个进程内提升。worker 数要根据是 CPU 密集还是 IO 密集、以及实际压测结果来定(别瞎配)。还要配置超时、优雅重启、并发限制等保证稳定性。

</details>

5. 在 FastAPI 的 async 视图里能不能用同步的数据库/HTTP 库?为什么?

<details><summary>参考答案</summary>

**不能直接用(会出大问题)**。FastAPI 的 `async def` 视图运行在 **asyncio 事件循环**所在的单个线程上(见 P22),它依赖"协程在等待 IO 时主动让出控制权"来实现高并发。如果你在 async 视图里**直接调用同步阻塞的库**——如同步的 `requests.get()`、同步数据库驱动(同步 psycopg、同步 SQLAlchemy 查询)、`time.sleep()`、同步文件 IO——这些调用**不会让出控制权,而是直接阻塞事件循环所在的线程**,在它阻塞的整段时间里,**事件循环无法处理任何其他请求/协程**,导致整个服务(该进程)被这一个慢调用卡住、并发能力崩塌(见 P22/P23 的阻塞陷阱)。一个同步阻塞调用就能毁掉异步服务的高并发优势。**正确做法**:①**使用异步库**——用 `httpx`(异步模式)或 `aiohttp` 替代 `requests`;用 `asyncpg` 或 **SQLAlchemy 2.0 的 async 引擎/会话**(配异步驱动)替代同步 DB 访问;用 `redis.asyncio` 替代同步 redis(见 P29、P23)。这样这些 IO 操作是 awaitable 的,会在等待时正确让出。②**实在必须用同步/阻塞代码**(没有异步版本的库、或 CPU 密集计算),用 **`await asyncio.to_thread(blocking_func, ...)` 或 `run_in_executor`** 把它**丢到线程池**(阻塞 IO)或**进程池**(CPU 密集)执行,从而阻塞发生在别的线程/进程、不堵事件循环(见 P23)。③**或者**,如果你的整个应用就是同步阻塞风格、又不追求 asyncio 的极致并发,FastAPI 也允许你把视图写成**普通的 `def`(同步函数)**——这种情况下 FastAPI 会**自动把该同步视图放到线程池里执行**(而不是在事件循环线程里),从而同步阻塞库可以安全使用(代价是回到线程并发模型、受线程数限制)。所以要点是:**事件循环线程上绝不能有阻塞调用**——要么 async 视图 + 全异步库,要么用 executor 隔离阻塞,要么干脆用同步 def 视图让框架放线程池跑;最忌讳的是"async 视图里直接调同步阻塞库"。

</details>

6. Python Web 的并发模型和 Go、Java 相比如何?

<details><summary>参考答案</summary>

差异主要源于**各语言的并发能力和 GIL**。**Python**:因为有 GIL(见 P17),单进程无法用多线程并行处理 CPU,所以 Web 并发要靠两条线拼:①**多进程**(Gunicorn 多 worker)来利用多核;②每个进程内用**多线程(WSGI,受线程数限制)或 asyncio(ASGI/FastAPI,单线程协程扛高并发 IO)**。而且 asyncio 模型有"同步/异步分裂"和"染色"问题——你得明确选 WSGI(同步)还是 ASGI(异步),异步要求全链路 async、不能有阻塞调用(见 P23),心智负担较重。所以典型高并发 Python 服务是"多进程 + 每进程 asyncio"的组合。**Go**:并发是语言核心——`net/http`(见 G27)默认就为**每个请求开一个 goroutine**,goroutine 极轻量(可几十万个)、由运行时调度、**能真正多核并行**(无 GIL),且写法是**同步阻塞风格**(代码简单直观,不需要 async/await 染色),运行时自动处理 IO 多路复用。所以 Go 用一个进程、同步写法就能高效利用多核 + 扛高并发,模型最统一简洁。**Java**:Spring MVC(见 J25)传统上用**线程池**处理请求(一请求一线程),线程能真正并行(无 GIL),靠调大线程池和异步支持应对并发;Java 21 引入**虚拟线程**(见 J30)后,更是能用"同步阻塞写法 + 海量轻量虚拟线程"达到类似 Go 的效果(一请求一虚拟线程,阻塞时自动让出),大幅简化高并发 IO 编程。**对比小结**:Go 和带虚拟线程的 Java 可以**用同步写法 + 真并行 + 单进程**优雅地同时解决"多核并行"和"高并发 IO";而 Python 受 GIL 制约,必须**多进程(为多核)+ asyncio 或多线程(为 IO 并发)**组合,且 asyncio 有显式 async/await 的复杂度。这也是 Python 在纯高并发/高性能服务上的相对短板(但其开发效率、生态尤其 AI/数据生态是优势)。未来 free-threading(去 GIL,见 P17/P24)若成熟,有望让 Python 的并发模型向 Go/Java 靠拢。

</details>
