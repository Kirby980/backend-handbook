# 精通 Java 测试

> Go 专题把测试（[G18](../golang/G18-精通-Go-测试.md)）和 benchmark 单列一章，Java 这边却常被忽视——很多人只会写几个 `@Test`，不懂 JUnit 5 的扩展模型、Mockito 的 mock/stub/verify 边界、Spring Boot 的测试切片，更不知道 Testcontainers 怎么用真数据库做集成测试。本篇补齐这块工程化短板，是 Java track 对照 Go track 的关键缺口。
>
> **📅 基准：2026 年 6 月。JUnit 5（Jupiter）、Mockito 5、Spring Boot 3.x Test、Testcontainers 主流。**

---

## 一、测试金字塔与分层

- **单元测试（Unit）**：测一个类/方法，**不依赖外部**（DB/网络/文件），用 mock 隔离协作者。快、多、稳——金字塔底座。
- **集成测试（Integration）**：测多个组件协作（如 Service + Repository + 真实 DB），慢一些、数量中等。
- **端到端（E2E）**：跑完整链路，最慢最脆，数量最少——金字塔顶。

原则：**底层多、顶层少**。单测要快（毫秒级、可并行），慢的依赖（DB/中间件）留给集成测试，并用 Testcontainers 保证真实可靠。

---

## 二、JUnit 5 架构与生命周期

JUnit 5 = **Platform + Jupiter + Vintage** 三部分：

- **JUnit Platform**：测试运行的基础平台（IDE/Maven/Gradle 通过它发现并运行测试）。
- **Jupiter**：JUnit 5 的新编程模型与注解（`@Test` 等）。
- **Vintage**：兼容运行 JUnit 3/4 老用例的引擎。

核心注解与生命周期：

```java
class OrderServiceTest {
    @BeforeAll  static void initAll() {}   // 全类一次（默认静态）
    @BeforeEach void init() {}             // 每个测试前
    @Test       void shouldCreateOrder() {}
    @AfterEach  void tearDown() {}         // 每个测试后
    @AfterAll   static void cleanup() {}   // 全类一次
}
```

| JUnit 4 | JUnit 5（Jupiter） |
|---|---|
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| `@BeforeClass` / `@AfterClass` | `@BeforeAll` / `@AfterAll` |
| `@Ignore` | `@Disabled` |
| `@RunWith` / `@Rule` | **`@ExtendWith`（统一扩展模型）** |
| `expected=`、`timeout=` 属性 | `assertThrows` / `assertTimeout` |

默认每个 `@Test` 方法都**新建一次测试类实例**（保证测试间隔离、无状态串扰）；这也是 `@BeforeAll`/`@AfterAll` 默认要 static 的原因。

---

## 三、断言与组织

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(expected, actual, "可选失败消息");
assertThrows(IllegalArgumentException.class, () -> svc.create(null)); // 断言抛异常
assertAll("user",                                  // 分组断言：全部执行再汇总失败
    () -> assertEquals("Alice", user.name()),
    () -> assertTrue(user.age() > 0));
assertTimeout(Duration.ofMillis(100), () -> svc.fast()); // 超时
```

- **`assertThrows`** 返回异常对象，可继续断言其 message。
- **`assertAll`** 把多个断言**全部执行**再统一报告（不像普通断言遇错即停），适合校验一个对象的多个字段。
- 复杂对象/集合断言推荐 **AssertJ**（`assertThat(list).hasSize(3).contains(x)`），流式、可读性远好于原生。
- 其他实用注解：`@DisplayName("中文用例名")`、`@Nested`（嵌套组织场景）、`@Tag`（分类筛选，如 `@Tag("slow")`）、`@Disabled`、条件注解 `@EnabledOnOs`/`@EnabledIf`。

---

## 四、参数化测试

一份逻辑、多组数据——避免复制粘贴：

```java
@ParameterizedTest
@ValueSource(ints = {2, 3, 5, 7})
void isPrime(int n) { assertTrue(PrimeUtil.isPrime(n)); }

@ParameterizedTest
@CsvSource({"1,1,2", "2,3,5", "10,-4,6"})        // 多列：a,b,期望
void add(int a, int b, int expected) { assertEquals(expected, Calc.add(a, b)); }

@ParameterizedTest
@MethodSource("provideCases")                     // 复杂对象用方法提供
void check(User u, boolean valid) { assertEquals(valid, validator.test(u)); }
static Stream<Arguments> provideCases() {
    return Stream.of(arguments(new User("a", 20), true), arguments(new User("", -1), false));
}
```

数据源：`@ValueSource`（单列简单值）、`@CsvSource`/`@CsvFileSource`（多列）、`@EnumSource`（枚举全覆盖）、`@MethodSource`（任意复杂对象，最灵活）。**优先用参数化覆盖边界/等价类**，比写一堆相似 `@Test` 干净得多。

---

## 五、Mockito：隔离协作者

单测要隔离外部协作者（DB、远程服务），用 **Mockito** 造**测试替身**：

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock  OrderRepository repo;          // 造一个假的依赖
    @InjectMocks OrderService service;    // 把 mock 注入被测对象

    @Test
    void shouldReject_whenStockEmpty() {
        // given：打桩（stub）——规定 mock 被调用时返回什么
        when(repo.stockOf(1L)).thenReturn(0);

        // when & then
        assertThrows(OutOfStock.class, () -> service.place(1L));

        // verify：验证交互（行为）
        verify(repo).stockOf(1L);
        verify(repo, never()).deduct(anyLong(), anyInt()); // 不该扣减
    }
}
```

关键概念：

- **Stub（打桩）**：`when(...).thenReturn(...)` / `thenThrow(...)` —— 规定 mock 在特定输入下的返回，用于**控制被测代码走哪条路径**。
- **Verify（验证）**：`verify(mock, times(n)/never()/atLeast(n)).method(...)` —— 验证某交互**是否/被调用几次**，用于测"副作用型"行为（发消息、扣库存）。
- **参数匹配器**：`any()`、`eq()`、`argThat(...)`；**一旦用了匹配器，所有参数都要用匹配器**（`eq(1L)` 不能和裸 `2` 混用）。
- **`@Spy`**：部分 mock——默认走真实方法，可选择性打桩。
- **`mockStatic`**（Mockito 5）：mock 静态方法（少用，往往是设计耦合的信号）。

> **mock 什么、不 mock 什么**：mock **你拥有的、有副作用的、慢的**协作者（Repository、远程 client）。**不要 mock** 值对象、不可变数据、第三方类型本身（mock 你不拥有的类型很脆），也不要在单测里 mock 一切以致"测的是 mock 而非逻辑"。

---

## 六、Spring Boot 测试切片

Spring 应用的测试分两档，关键是**别动不动启动整个容器**：

```java
// ① 切片测试：只加载 Web 层，Service 用 @MockBean 顶替——快
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockBean OrderService service;       // 容器里用 mock 替换真实 bean

    @Test
    void getOrder() throws Exception {
        when(service.find(1L)).thenReturn(new Order(1L));
        mvc.perform(get("/orders/1"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.id").value(1));
    }
}

// ② 全量集成：启动完整上下文 + 随机端口真实 HTTP——慢，少用
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderApiIT { @Autowired TestRestTemplate rest; /* ... */ }
```

切片注解（只加载相关那层 bean，启动快）：

| 注解 | 加载范围 |
|---|---|
| `@WebMvcTest` | 只 Controller + MVC 相关（[J25](./J25-精通-Spring-MVC.md)），配 `MockMvc` |
| `@DataJpaTest` | 只 JPA/Repository + 内嵌或 Testcontainers DB，默认事务回滚 |
| `@JsonTest` | 只 Jackson 序列化 |
| `@SpringBootTest` | **整个**应用上下文（最重，留给真正的集成测试） |

要点：用 **`@MockBean`** 把切片外的依赖换成 mock；`@SpringBootTest` 会因不同配置缓存并复用 `ApplicationContext`（[J22](./J22-精通-Spring-IOC容器.md)）——尽量统一配置以命中缓存、别让每个测试类都新建上下文（否则测试套件极慢）。

---

## 七、Testcontainers：用真实依赖做集成测试

内嵌 H2 替代 MySQL 做测试有个老问题：**H2 的 SQL 方言/行为和生产 MySQL 不一致**，测过了线上照样炸。**Testcontainers** 用 Docker 在测试时**拉起一个真实的 MySQL/Redis/Kafka 容器**，测完销毁——既真实又隔离：

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIT {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @DynamicPropertySource                       // 把容器的随机端口注入 Spring
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", mysql::getJdbcUrl);
        r.add("spring.datasource.username", mysql::getUsername);
        r.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired OrderRepository repo;
    @Test void persistsAndReads() { /* 对真实 MySQL 验证 */ }
}
```

- `static` 容器 + `@Container` → 全类共享一个容器（省启动时间）；非 static 则每个测试一个（更隔离但更慢）。
- 适用 MySQL/PostgreSQL/Redis/Kafka/ES 等几乎所有带 Docker 镜像的中间件。
- 代价：需要 Docker 环境、首次拉镜像慢——所以归到**集成测试**层，不放在每次都跑的单测里。

---

## 八、覆盖率与 CI

- **JaCoCo** 统计行/分支覆盖率，Maven/Gradle 插件可设阈值（如低于 70% 构建失败）。
- **别迷信覆盖率数字**：100% 覆盖不等于测得对——覆盖率只说明"代码被执行过"，不保证断言有意义。重点测**业务分支、边界、异常路径**。
- CI 里分层跑：单测每次提交都跑（快）；集成测试（Testcontainers）可在合并前/夜间跑。用 `@Tag` + Maven Surefire/Failirsafe 区分 `*Test`（单元）与 `*IT`（集成）。

---

## 陷阱清单

- **测试之间有状态依赖/顺序依赖**：JUnit 5 默认每个测试新建实例正是为隔离；别用静态可变状态让测试互相影响，别依赖执行顺序。
- **动不动 `@SpringBootTest` 启全容器**：套件巨慢。优先用切片（`@WebMvcTest`/`@DataJpaTest`）+ `@MockBean`。
- **每个测试类用不同配置**：导致 Spring 上下文缓存失效、反复重建，套件极慢。统一测试配置命中缓存。
- **用 H2 替代生产 DB**：方言/行为不一致，测过仍线上炸。集成测试用 Testcontainers 跑真实 DB。
- **mock 一切**：把被测逻辑也 mock 掉，测的是 mock 不是代码。只 mock 外部/有副作用的协作者。
- **mock 你不拥有的第三方类型**：脆弱、随库升级失效。在自己的接口边界上 mock。
- **`orElse`/参数匹配器混用**：用了 matcher 后所有参数都要 matcher，否则报错。
- **只看覆盖率不看断言质量**：高覆盖率可能全是没断言或断言无意义的"伪测试"。
- **测试不可重复/依赖时钟与随机**：用固定 `Clock`、固定种子，避免偶发失败（flaky）。
- **慢的集成测试和单测混跑**：拖慢反馈。用 `@Tag`/命名区分，CI 分层执行。

---

## 2026 现状

- **JUnit 5 + AssertJ + Mockito** 是事实标准组合；新项目基本不再用 JUnit 4。
- **Testcontainers 成为集成测试主流**，取代内嵌 H2/嵌入式中间件；Spring Boot 3.1+ 提供 `@ServiceConnection` 进一步简化容器与配置的对接。
- **虚拟线程下的测试**：注意被测代码若用[虚拟线程](./J30-精通-虚拟线程与结构化并发.md)/结构化并发，并发测试要确保确定性（控制并发、避免依赖时序）。
- **契约测试**（Spring Cloud Contract / Pact）在微服务间流行，保证服务间接口兼容（属微服务专题范畴）。
- **AI 辅助生成测试**：用 Claude 等生成单测脚手架/边界用例已普及，但**断言的正确性仍需人把关**——AI 易生成"覆盖了但没真正验证"的测试。
- **变异测试（PIT）**：用"故意改坏代码看测试能否发现"来评估测试**有效性**，弥补覆盖率只看"是否执行"的不足，在重视质量的团队中升温。

---

## 练习题

1. JUnit 5 由哪几部分组成？相比 JUnit 4 有哪些主要变化？

<details><summary>参考答案</summary>

**JUnit 5 由三部分组成**：①**JUnit Platform**——测试运行的基础平台，定义了 TestEngine API，IDE、Maven（Surefire）、Gradle 等通过它来发现和启动测试；②**JUnit Jupiter**——JUnit 5 全新的编程模型和扩展模型，提供 `@Test`、`@BeforeEach` 等新注解和断言，跑在一个 Jupiter TestEngine 上；③**JUnit Vintage**——提供一个兼容引擎，让旧的 JUnit 3/4 测试用例也能在 JUnit 5 平台上运行，便于渐进迁移。**相比 JUnit 4 的主要变化**：①生命周期注解改名——`@Before/@After`→`@BeforeEach/@AfterEach`，`@BeforeClass/@AfterClass`→`@BeforeAll/@AfterAll`，`@Ignore`→`@Disabled`；②**用统一的扩展模型 `@ExtendWith` 取代了 JUnit 4 的 `@RunWith`（Runner）和 `@Rule/@ClassRule`**——一个测试可以注册多个扩展，更灵活可组合（如 `@ExtendWith(MockitoExtension.class)`、`@ExtendWith(SpringExtension.class)`）；③异常和超时断言改为编程式的 `assertThrows`、`assertTimeout`，不再用 `@Test(expected=…, timeout=…)` 属性；④新增 `assertAll`（分组断言，全部执行再汇总）、`@DisplayName`（自定义用例名、支持中文）、`@Nested`（嵌套组织）、`@Tag`（分类筛选）、强大的参数化测试（`@ParameterizedTest` + 各种数据源）；⑤大量使用 Java 8 特性（Lambda、Stream），如断言接收 Supplier 延迟构造消息。此外 JUnit 5 默认每个测试方法都新建一个测试类实例以保证隔离。

</details>

2. Mockito 中 stub（打桩）和 verify（验证）有什么区别？分别用于什么场景？

<details><summary>参考答案</summary>

二者针对测试替身的两种不同关注点。**Stub（打桩）**：用 `when(mock.method(args)).thenReturn(value)` / `thenThrow(ex)` 等，**规定 mock 在被特定输入调用时返回什么或抛什么**。它的目的是**控制被测代码的执行路径**——通过让协作者返回预设的数据，驱动被测方法走到我们想测的分支（如让 repository 返回库存为 0，以测试"库存不足"的逻辑）。它关注的是"**状态/输入**"——给被测代码喂好数据。**Verify（验证）**：用 `verify(mock, times(n)/never()/atLeastOnce()).method(args)`，**验证被测代码在执行过程中是否、以及以何种参数、调用了 mock 的某个方法多少次**。它的目的是**断言"交互行为"是否符合预期**，用于测试那些**没有返回值、靠副作用体现**的逻辑——例如"下单成功后应该调用库存服务扣减一次"、"参数非法时不应该调用支付"。它关注的是"**行为/输出交互**"。**场景区分**：当被测方法的正确性体现在它**返回了什么/走了哪条分支**时，重点用 stub 准备数据 + 对返回值断言；当正确性体现在它**对协作者做了什么副作用调用**（发消息、写库、扣减）时，用 verify 验证交互。实际测试中两者常配合：先 stub 设定环境，执行被测方法，再对结果 assert + 对关键副作用 verify。注意不要过度 verify（验证每一个无关紧要的调用会让测试僵硬、一改实现就崩），只验证真正重要的交互。

</details>

3. Spring Boot 中 `@WebMvcTest`、`@DataJpaTest`、`@SpringBootTest` 有什么区别？为什么不应该都用 @SpringBootTest？

<details><summary>参考答案</summary>

这三者加载的 Spring 上下文范围不同，对应不同的测试粒度。**`@WebMvcTest`**：**切片测试**，只加载 Web 层相关的 bean（Controller、`@ControllerAdvice`、Filter、MVC 基础设施，见 J25），**不加载** Service、Repository 等，通常配合 `MockMvc` 测试 HTTP 请求/响应、状态码、JSON、参数绑定，Service 层用 `@MockBean` 顶替。启动快。**`@DataJpaTest`**：切片测试，只加载 JPA/Repository 相关组件（EntityManager、Repository、DataSource），用于测持久层；默认每个测试在事务中执行并在结束时回滚，默认会用内嵌库（也可配 Testcontainers 用真实库）。**`@SpringBootTest`**：**加载整个应用的 ApplicationContext**（所有 bean、自动配置），是真正的集成测试，可配合 `webEnvironment=RANDOM_PORT` 启动真实内嵌服务器 + `TestRestTemplate`/`WebTestClient` 做端到端 HTTP 测试。**为什么不应该都用 @SpringBootTest**：①**慢**——它启动完整上下文（可能上百上千个 bean、连接池、各种自动配置），每个这样的测试类启动成本很高，整个测试套件会变得极慢，破坏"测试要快速反馈"的原则；②**不够聚焦**——测一个 Controller 却把整个应用拉起来，引入大量无关依赖，出错时难定位；③切片测试只装载需要的那层、用 `@MockBean` 隔离其余，既快又聚焦，符合测试金字塔"底层单元/切片测试要多而快"的理念。所以正确做法是：优先用 `@WebMvcTest`/`@DataJpaTest`/`@JsonTest` 等切片做窄范围测试，只在确实需要验证"整个应用端到端协作"时才用 `@SpringBootTest`，并尽量统一测试配置以命中 Spring 的 ApplicationContext 缓存（不同配置会导致上下文反复重建，拖慢套件）。

</details>

4. 为什么集成测试推荐用 Testcontainers 而不是内嵌 H2？

<details><summary>参考答案</summary>

核心原因是**真实性（环境一致性）**。内嵌数据库（如 H2）虽然启动快、零依赖，但它**和生产实际使用的数据库（如 MySQL/PostgreSQL）在 SQL 方言、函数、数据类型、索引行为、隔离级别、约束处理、特定语法等方面存在差异**。用 H2 测试可能出现"测试全过、上线就炸"的情况：比如用到了 MySQL 特有的语法/函数、`ON DUPLICATE KEY UPDATE`、特定的 JSON 类型、或依赖某种锁/隔离行为，H2 要么不支持要么行为不同，测试根本覆盖不到真实问题；反之有些 H2 兼容模式下能跑的写法在真实库上又有差异。**Testcontainers** 用 Docker 在测试运行时**拉起一个与生产同款的真实数据库容器**（如 `mysql:8.0`），测试连真容器跑，结束后自动销毁。这样集成测试运行在**和生产几乎一致的真实环境**上，能发现方言、迁移脚本、真实约束等问题，可信度高得多；同时每次用全新容器、互相隔离、可重复。代价是**需要 Docker 环境、首次拉镜像和启动容器比内嵌库慢**，所以它属于"集成测试"层面（用 static 容器全类共享、或在 CI 的合并前/夜间阶段跑），不适合放进每次提交都要飞快跑完的纯单元测试里。它适用于几乎所有有官方 Docker 镜像的中间件（MySQL、PostgreSQL、Redis、Kafka、Elasticsearch 等）。Spring Boot 3.1+ 还提供 `@ServiceConnection` 让容器与数据源配置自动对接，进一步降低样板。

</details>

5. 代码覆盖率高就代表测试质量好吗？应该重点测什么？

<details><summary>参考答案</summary>

**不是**。代码覆盖率（如 JaCoCo 统计的行覆盖率、分支覆盖率）只能说明"**测试运行时这些代码被执行过**"，并**不能保证测试真正验证了行为的正确性**。常见的"伪高覆盖"：①测试调用了方法但**几乎没有断言**，或断言无意义（如只 `assertNotNull`），代码被执行了、覆盖率上去了，但任何逻辑错误都发现不了；②只覆盖了正常路径（happy path），没覆盖**边界值、异常分支、错误输入**——而 bug 往往就藏在这些地方；③为了凑覆盖率写一堆无价值的测试，维护成本高却没有保护作用。所以覆盖率是一个**有用但有局限的参考指标**（太低肯定有问题，比如低于某阈值让 CI 失败是合理的底线），但不能把"100% 覆盖"当作目标或质量证明。**应该重点测**：①**业务核心逻辑和各个分支**（if/else、switch、循环边界）；②**边界条件与等价类**（空、null、0、最大值、越界、首尾元素）——用参数化测试高效覆盖；③**异常/错误路径**（非法输入抛什么异常、依赖失败如何降级）；④**关键的副作用交互**（该调用的调用了、不该调用的没调用，用 Mockito verify）；⑤回归测试（每修一个 bug 补一个能复现它的测试）。衡量测试"有效性"更进一步可以用**变异测试（如 PIT）**——它故意把代码改坏（制造变异体）看测试能否检测到失败，能更真实地反映测试是否真的在验证逻辑，弥补覆盖率只看"是否执行"的不足。总之：追求**有意义的断言覆盖关键路径**，而非单纯的数字。

</details>
