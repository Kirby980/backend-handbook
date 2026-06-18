# 精通 Spring Boot 自动配置

> Spring Boot 让"开箱即用"成为可能——引入一个 starter 就自动配好一切，不用写一堆 XML/配置。它的核心魔法是**自动配置（Auto-Configuration）**。理解它的原理（`@EnableAutoConfiguration` + 条件注解 + SPI），是 Spring Boot 面试的重点。
>
> **📅 基准：Spring Boot 3（Spring 6，基线 Java 17）。**

---

## 一、Spring Boot 解决什么

传统 Spring 要写大量 XML/Java 配置（配 DataSource、事务、MVC…），繁琐易错。Spring Boot 用**约定优于配置（Convention over Configuration）**简化：

- **自动配置**：根据 classpath 上有什么依赖，自动配好对应的 Bean（有 spring-web 就配 MVC、有 mysql driver 就配 DataSource）。
- **起步依赖（starter）**：一个 starter 聚合一组相关依赖，引一个搞定。
- **内嵌容器**：内置 Tomcat，打成可执行 jar 直接 `java -jar` 运行，无需部署到外部容器。
- **开箱即用**：合理的默认配置，需要时再覆盖。

一句话：Spring Boot = Spring + 自动配置 + starter + 内嵌容器，让你专注业务。

---

## 二、@SpringBootApplication

启动类上的 `@SpringBootApplication` 是一个组合注解，三大核心：

```java
@SpringBootApplication   // = 下面三个
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

| 组成注解 | 作用 |
|---|---|
| `@SpringBootConfiguration` | 标记这是配置类（本质是 `@Configuration`） |
| `@EnableAutoConfiguration` | **开启自动配置**（核心魔法，见第三节） |
| `@ComponentScan` | 扫描启动类所在包及子包的组件（@Component/@Service 等） |

所以**启动类的位置很重要**——@ComponentScan 默认扫它所在的包，放错位置会扫不到你的 Bean。

---

## 三、自动配置原理

**这是核心考点。** `@EnableAutoConfiguration` 怎么实现"自动配好"？

```mermaid
flowchart TB
    EA["@EnableAutoConfiguration"] --> Import["@Import(AutoConfigurationImportSelector)"]
    Import --> Selector[AutoConfigurationImportSelector<br>selectImports]
    Selector --> Load[读取 META-INF/spring/<br>...AutoConfiguration.imports<br>所有自动配置类全限定名]
    Load --> Filter[按条件注解过滤<br>@ConditionalOnXxx]
    Filter --> Register[满足条件的自动配置类<br>注册成 Bean]
    style Selector fill:#fff9c4
    style Filter fill:#c8e6c9
```

流程：
1. `@EnableAutoConfiguration` 通过 `@Import` 导入 **AutoConfigurationImportSelector**。
2. 它读取所有依赖 jar 里 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（**Spring Boot 2.7 前是 `META-INF/spring.factories`**），拿到一大堆候选的"自动配置类"全限定名。
3. 对每个自动配置类，按其上的**条件注解（@ConditionalOnXxx）**判断要不要生效。
4. 满足条件的自动配置类被加载，里面用 `@Bean` 注册了相应的 Bean（如 DataSourceAutoConfiguration 配 DataSource）。

本质：**基于 SPI（类似 [J16](./J16-精通-类加载与双亲委派.md) 的 SPI 思想）加载一堆候选配置 + 用条件注解按需启用**。

---

## 四、条件注解

自动配置"按需生效"靠 **@Conditional** 系列条件注解——只有满足条件才装配：

| 条件注解 | 含义 |
|---|---|
| `@ConditionalOnClass` | classpath 有某个类才生效（如有 DataSource 类才配数据源） |
| `@ConditionalOnMissingClass` | 没有某类才生效 |
| `@ConditionalOnBean` | 容器有某 Bean 才生效 |
| **`@ConditionalOnMissingBean`** | 容器**没有**某 Bean 才生效（**关键**：用户自定义了就用用户的，没定义才用默认） |
| `@ConditionalOnProperty` | 配置项满足条件才生效（如 `xxx.enabled=true`） |
| `@ConditionalOnWebApplication` | 是 Web 应用才生效 |

`@ConditionalOnMissingBean` 是"**默认配置可被覆盖**"的关键：自动配置类用它声明默认 Bean——如果你自己定义了同类型 Bean，默认的就不生效（你的优先），没定义才用默认。这就是"约定优于配置 + 可覆盖"的实现。

---

## 五、starter 机制

**starter（起步依赖）** 是"依赖聚合 + 自动配置"的打包：

- 一个 starter（如 `spring-boot-starter-web`）的 pom 里聚合了一组相关依赖（Spring MVC、Tomcat、Jackson 等），引一个就把这套都带进来——**省去手动管理一堆依赖和版本冲突**。
- 配合对应的自动配置类（在依赖里），引入 starter 后相关功能自动配好。

常见 starter：`starter-web`（Web/MVC）、`starter-data-jpa`（JPA）、`starter-data-redis`、`starter-test`、`starter-security` 等。

starter 的价值 = **依赖管理（版本统一，靠 BOM）+ 自动配置（开箱即用）**。

---

## 六、配置加载与优先级

Spring Boot 支持**外部化配置**，同一配置可来自多处，按优先级覆盖：

- 配置文件：`application.yml`/`application.properties`。
- **Profile**：`application-{profile}.yml`（如 dev/test/prod），用 `spring.profiles.active` 激活——多环境配置。
- **优先级（高 → 低，高的覆盖低的）**：命令行参数 > 环境变量 > `application-{profile}` > `application.yml` > 自动配置默认值。
- 配置可绑定到 `@ConfigurationProperties` 类（类型安全）或 `@Value` 注入。

外部化配置让同一份 jar 能在不同环境用不同配置（[B19 12-Factor](../backend/B19-精通-12-Factor-App.md) 的"配置与代码分离"原则）。

---

## 七、内嵌容器

Spring Boot **内嵌 Servlet 容器**（默认 Tomcat，也可换 Jetty/Undertow）：

- 应用打成**可执行 fat jar**（含所有依赖 + 内嵌 Tomcat），`java -jar app.jar` 直接启动，**不用再部署到外部 Tomcat**。
- 这契合云原生/容器化——一个 jar 就是一个自包含的服务，方便 Docker 化（见 [cloud-native](../cloud-native/INDEX.md)）。
- DispatcherServlet（[J25](./J25-精通-Spring-MVC.md)）跑在这个内嵌容器里。

---

## 八、自定义 starter

理解了原理就能自定义 starter（如公司内部组件）：

1. 写自动配置类（`@Configuration` + `@Bean` + `@ConditionalOnXxx`）。
2. 在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 里注册它（Boot 2.7+；老版本用 `spring.factories`）。
3. 提供 `@ConfigurationProperties` 让使用方可配置。
4. 打成 starter，别人引入即自动生效（且可用 `@ConditionalOnMissingBean` 允许覆盖）。

命名约定：官方 starter 叫 `spring-boot-starter-xxx`，第三方/自定义叫 `xxx-spring-boot-starter`。

---

## 陷阱清单

- **启动类放错包**：@ComponentScan 默认扫启动类所在包及子包，放错导致 Bean 扫不到。启动类应在最外层包。
- **以为自动配置是"扫描所有类"**：是读 imports 文件（SPI）+ 条件注解过滤，不是无脑扫。
- **不懂 @ConditionalOnMissingBean**：自定义 Bean 覆盖默认配置的关键；不理解会困惑"为什么我的配置没生效/被覆盖"。
- **spring.factories vs AutoConfiguration.imports**：Boot 2.7 起自动配置改用新文件，老资料/老 starter 仍用 spring.factories，升级要注意。
- **配置优先级搞错**：命令行/环境变量优先级高于配置文件，线上排查"配置没生效"要查是否被高优先级覆盖。
- **fat jar 解压当普通 jar 用**：Boot 的可执行 jar 有特殊结构（嵌套依赖），不能当普通库直接被依赖。
- **排除自动配置**：不需要的自动配置可用 `@SpringBootApplication(exclude=...)` 排除，否则可能因缺配置报错（如引了 DataSource 依赖却没配数据库）。

---

## 2026 现状

- **AutoConfiguration.imports 取代 spring.factories**：Spring Boot 2.7+ 自动配置注册改用 `META-INF/spring/...AutoConfiguration.imports`，3.x 已标准化。
- **Spring Boot 3 + GraalVM 原生镜像**：通过**构建期 AOT 处理**，把"运行时反射 + 条件判断"的自动配置提前在构建期计算、生成代码，以适配原生镜像（无运行时类加载/反射受限，见 [J16](./J16-精通-类加载与双亲委派.md)/[J21](./J21-精通-字节码与执行引擎.md)），并大幅加快启动。
- **可观测开箱即用**：Boot 3 内置 Micrometer + Observation API，自动配好 metrics/tracing（见 [microservices 可观测](../microservices/18-精通-可观测三支柱.md)）。
- **配置仍是外部化主流**：application.yml + profile + 配置中心（Nacos/Apollo，见 [microservices 配置中心](../microservices/14-精通-配置中心.md)）。

---

## 练习题

1. @SpringBootApplication 由哪几个注解组成？各自作用是什么？

<details><summary>参考答案</summary>

@SpringBootApplication 是一个组合注解，核心由三个注解构成：①**@SpringBootConfiguration**——本质是 @Configuration 的特化，标记该类是一个 Spring 配置类（可以在里面用 @Bean 定义 Bean），表明这是 Spring Boot 应用的主配置类。②**@EnableAutoConfiguration**——**开启自动配置功能**，这是 Spring Boot 的核心魔法所在：它会根据 classpath 上存在的依赖、以及各种条件注解，自动地装配大量默认的 Bean（如检测到 spring-web 就自动配置 Spring MVC、检测到数据库驱动就自动配置 DataSource），免去手动配置。③**@ComponentScan**——开启组件扫描，默认扫描**启动类所在包及其子包**下的所有组件（@Component、@Service、@Repository、@Controller、@Configuration 等），把它们注册为 Bean。三者合在一起：标识主配置类 + 开启自动配置 + 扫描业务组件，使得一个标了 @SpringBootApplication 的启动类配合 SpringApplication.run 就能拉起整个应用。需要注意 @ComponentScan 默认从启动类所在包扫起，所以**启动类应放在项目的最外层根包**，否则部分业务包里的 Bean 会扫描不到。

</details>

2. Spring Boot 自动配置的原理是什么？

<details><summary>参考答案</summary>

自动配置由 @EnableAutoConfiguration 驱动，核心流程：①@EnableAutoConfiguration 通过 `@Import` 导入了 **AutoConfigurationImportSelector** 这个选择器。②该选择器在容器启动时，会去读取所有依赖 jar 包中 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 2.7 之前是读 `META-INF/spring.factories` 中 EnableAutoConfiguration 对应的配置），从中获得一大批**候选的自动配置类**的全限定名（这是一种 SPI 机制——约定的文件里列出所有可能要加载的自动配置类）。③对这些候选自动配置类，Spring Boot 会结合每个类上标注的**条件注解（@ConditionalOnXxx）**进行**过滤**——只有满足条件的才会真正生效（例如某自动配置类标了 @ConditionalOnClass(DataSource.class)，只有 classpath 里有 DataSource 类时它才生效）。④满足条件的自动配置类被加载、解析，其中用 @Bean 方法定义的 Bean 被注册到容器中（如 DataSourceAutoConfiguration 注册数据源、WebMvcAutoConfiguration 注册 MVC 相关组件）。本质上：**"通过 SPI 加载一堆候选配置类 + 用条件注解按需启用 + 注册对应 Bean"**。再配合 @ConditionalOnMissingBean，使得这些默认配置可以被用户自定义的 Bean 覆盖。所以"引入依赖就自动配好"的背后，是 classpath 检测（条件注解）+ 约定文件（imports/spring.factories）+ 条件化 Bean 注册的组合。

</details>

3. 条件注解（如 @ConditionalOnMissingBean）在自动配置中起什么作用？

<details><summary>参考答案</summary>

条件注解（@Conditional 系列）是自动配置实现"**按需、智能、可覆盖**"装配的关键——它们让一个配置类或 @Bean 方法只有在满足特定条件时才生效，从而避免盲目装配、并允许用户定制。常见的有：@ConditionalOnClass（classpath 存在某个类时才配置，用于"有这个依赖才配相关功能"）、@ConditionalOnMissingClass（没有某类时）、@ConditionalOnBean（容器中已有某 Bean 时）、@ConditionalOnMissingBean（容器中**没有**某 Bean 时）、@ConditionalOnProperty（某配置项满足条件时，如 xxx.enabled=true）、@ConditionalOnWebApplication（是 Web 应用时）等。其中 **@ConditionalOnMissingBean 尤其重要**，它是"约定优于配置、默认可被覆盖"的实现机制：自动配置类在用 @Bean 提供一个**默认实现**时，往往加上 @ConditionalOnMissingBean——意思是"只有当用户自己没有定义这个类型的 Bean 时，我才提供这个默认 Bean"。这样一来，如果开发者自定义了同类型的 Bean，自动配置的默认 Bean 就不会生效（用户的优先）；如果开发者没定义，就用框架提供的合理默认值。这正是 Spring Boot "开箱即用又可灵活覆盖"的精髓——既给你默认配置让你零配置就能跑，又允许你随时用自己的 Bean 覆盖默认行为，而不必关闭整个自动配置。总之条件注解让自动配置做到"该配的才配、用户定制优先、按环境/依赖智能启用"。

</details>

4. 什么是 Spring Boot 的 starter？它解决了什么问题？

<details><summary>参考答案</summary>

starter（起步依赖）是 Spring Boot 提供的一种"**依赖聚合 + 自动配置**"的打包方式。一个 starter（如 spring-boot-starter-web、spring-boot-starter-data-redis）本身几乎不含代码，它的 pom 里**聚合了实现某项功能所需的一组相关依赖**（例如 starter-web 聚合了 Spring MVC、内嵌 Tomcat、Jackson 等），并且这些依赖的版本由 Spring Boot 的依赖管理（BOM/parent）统一管理。同时，相关依赖中带有对应的自动配置类，引入 starter 后该功能就被自动配置好。它解决的问题：①**简化依赖管理**——开发者不用自己一个个去找、添加并搭配某功能所需的多个库，引入一个 starter 就把整套相关依赖都带进来；②**避免版本冲突**——starter 配合 Spring Boot 的 BOM 统一指定了各依赖的兼容版本，免去手动协调版本号、避免"依赖地狱"和版本不兼容问题；③**开箱即用**——配合自动配置，引入 starter 后相应功能（如 Web、数据访问、缓存）几乎零配置就能用。所以 starter = 一组协调好版本的相关依赖 + 对应的自动配置，让"加一个依赖就拥有一项完整能力"成为可能，极大降低了搭建项目的成本。命名约定：官方 starter 为 spring-boot-starter-xxx，自定义/第三方为 xxx-spring-boot-starter。

</details>

5. Spring Boot 的配置加载有哪些来源？优先级如何？

<details><summary>参考答案</summary>

Spring Boot 支持**外部化配置**，同一个配置项可以从多种来源提供，并按优先级覆盖（高优先级覆盖低优先级），实现"同一份程序、不同环境用不同配置"。主要来源（优先级从高到低，常见排序）：①**命令行参数**（如 `--server.port=8081`，启动时传入，优先级最高）；②**操作系统环境变量 / JVM 系统属性**（如 `SERVER_PORT`、`-Dserver.port`，常用于容器化部署）；③**特定 Profile 的配置文件** `application-{profile}.yml/properties`（如 application-prod.yml，通过 `spring.profiles.active=prod` 激活，用于区分 dev/test/prod 多环境）；④**默认配置文件** `application.yml/application.properties`；⑤**@PropertySource 引入的配置**、自动配置提供的默认值等（优先级最低）。核心规则是：**越"外部"、越"具体环境"的配置优先级越高**，会覆盖通用/默认配置。配置可以通过 @Value 注入单个值，或绑定到 @ConfigurationProperties 标注的类做类型安全的批量绑定。这种设计契合 12-Factor App 的"配置与代码分离"原则——把会随环境变化的配置（数据库地址、密钥、端口等）外置，使同一个构建产物（jar/镜像）能部署到不同环境。实践提示：线上排查"配置没生效"时要特别注意是否被更高优先级的来源（如环境变量、命令行参数、配置中心）覆盖了。

</details>

6. 为什么 Spring Boot 应用可以打成一个 jar 直接 java -jar 运行，而不需要部署到外部 Tomcat？

<details><summary>参考答案</summary>

因为 Spring Boot **内嵌了 Servlet 容器**（默认 Tomcat，也可选 Jetty/Undertow）。传统的 Spring Web 应用需要打成 war 包，再部署到外部独立安装的 Tomcat 等 Servlet 容器中由容器来启动和管理。而 Spring Boot 把 Servlet 容器作为一个普通的依赖直接打包进应用内部：当应用启动（SpringApplication.run）时，它会以编程方式启动这个内嵌的 Tomcat，并把 DispatcherServlet 注册进去，应用自己就是一个能监听端口、处理 HTTP 请求的完整服务。配合 Spring Boot 的打包插件，应用被打成一个**可执行的 fat jar（胖 jar）**——里面包含了你的代码、所有第三方依赖、以及内嵌的 Tomcat，是一个自包含的整体，所以用 `java -jar app.jar` 就能直接启动运行，无需预先安装和配置外部容器。好处：①**部署极简**——不依赖目标机器上有特定版本的 Tomcat，避免容器版本/配置不一致问题，一个 jar 走天下；②**契合云原生/容器化**——一个自包含的 jar 非常适合做成 Docker 镜像、在 K8s 中部署（一个进程就是一个服务，符合"一个应用一个进程"的理念，见 cloud-native）；③开发调试方便——本地直接 run main 方法即可启动完整 Web 应用。如果确有需要（如部署到已有的应用服务器），Spring Boot 也支持打成 war 部署到外部容器，但内嵌容器 + 可执行 jar 是默认且主流的方式。

</details>
