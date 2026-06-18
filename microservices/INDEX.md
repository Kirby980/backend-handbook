# 微服务架构深度课程 · 总目录

> 22 篇中文深度课程，聚焦微服务从设计到落地的完整知识体系
> 每篇约 10000-15000 字，含原理、模式、代码示例、生产实践、陷阱清单与练习题
> 适合从"懂 RPC"到"能架构 50+ 服务平台"的中高级工程师 / 架构师 / SRE
>
> **📅 内容基准：2026 年 6 月** —— Istio Ambient / Cilium / gRPC / GraphQL Federation / OpenTelemetry / Kafka 4 / Saga / Outbox / Argo Rollouts / Feature Flags / Zero Trust + mTLS

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| M01 | [精通微服务架构总览](./01-精通-微服务架构总览.md) | ⭐⭐⭐ | 单体 vs 微服务 / 何时拆 / 康威定律 / SOA 与微服务史 |
| M02 | [精通服务拆分与 DDD](./02-精通-服务拆分与-DDD.md) | ⭐⭐⭐⭐⭐ | 限界上下文 / Aggregate / 战略设计 / 上下文映射 |
| M03 | [精通单体到微服务演进策略](./03-精通-单体到微服务演进.md) | ⭐⭐⭐⭐ | 绞杀者模式 / 并行运行 / 数据库拆分 / 影子流量 |
| M04 | [精通同步通信（REST / gRPC / GraphQL）](./04-精通-同步通信.md) | ⭐⭐⭐⭐ | REST / gRPC / GraphQL Federation / Protobuf / OpenAPI |
| M05 | [精通服务发现与注册](./05-精通-服务发现与注册.md) | ⭐⭐⭐⭐ | Consul / etcd / K8s Service / Eureka / DNS / 健康检查 |
| M06 | [精通 API 网关与 BFF](./06-精通-API-网关与-BFF.md) | ⭐⭐⭐⭐ | Kong / Envoy / KrakenD / Apollo Router / BFF 模式 |
| M07 | [精通异步消息与事件驱动](./07-精通-异步消息与事件驱动.md) | ⭐⭐⭐⭐ | Kafka / RabbitMQ / NATS / 事件 vs 消息 / Pub-Sub |
| M08 | [精通分布式事务](./08-精通-分布式事务.md) | ⭐⭐⭐⭐⭐ | 2PC / TCC / Saga / Outbox / 事件驱动一致性 |
| M09 | [精通幂等与去重](./09-精通-幂等与去重.md) | ⭐⭐⭐⭐ | 幂等 key / Token / 状态机 / 去重窗口 |
| M10 | [精通 Event Sourcing 与 CQRS](./10-精通-EventSourcing-与-CQRS.md) | ⭐⭐⭐⭐⭐ | 事件溯源 / CQRS / 物化视图 / 快照 |
| M11 | [精通限流熔断降级](./11-精通-限流熔断降级.md) | ⭐⭐⭐⭐ | Token Bucket / Sentinel / Resilience4j / 自适应限流 |
| M12 | [精通韧性设计模式](./12-精通-韧性设计模式.md) | ⭐⭐⭐⭐ | Bulkhead / Circuit Breaker / Retry / Fallback / Timeout |
| M13 | [精通分布式锁与协调](./13-精通-分布式锁与协调.md) | ⭐⭐⭐⭐ | Redis / etcd / ZK / Redlock / Lease |
| M14 | [精通配置中心与动态配置](./14-精通-配置中心.md) | ⭐⭐⭐ | Apollo / Nacos / Consul KV / 热更新 / 灰度配置 |
| M15 | [精通服务网格（应用层视角）](./15-精通-服务网格.md) | ⭐⭐⭐⭐⭐ | Istio Ambient / Linkerd / Cilium / mTLS / 流量治理 |
| M16 | [精通分布式 ID 与序列](./16-精通-分布式-ID.md) | ⭐⭐⭐ | Snowflake / UUIDv7 / Leaf / 序列号 / 时钟回拨 |
| M17 | [精通分布式链路追踪](./17-精通-分布式链路追踪.md) | ⭐⭐⭐⭐ | OpenTelemetry / Jaeger / Tempo / W3C TraceContext / 采样 |
| M18 | [精通微服务可观测三支柱](./18-精通-可观测三支柱.md) | ⭐⭐⭐⭐ | Metrics / Logs / Traces / Profiling / SLO |
| M19 | [精通灰度发布与渐进式交付](./19-精通-灰度发布.md) | ⭐⭐⭐⭐ | Canary / Blue-Green / A/B / Feature Flags / Argo Rollouts |
| M20 | [精通契约测试与混沌工程](./20-精通-契约测试与混沌工程.md) | ⭐⭐⭐⭐ | Pact / Schema 兼容 / Chaos Mesh / Gremlin |
| M21 | [精通微服务安全](./21-精通-微服务安全.md) | ⭐⭐⭐⭐⭐ | mTLS / Zero Trust / 服务身份 / JWT / RBAC / SPIFFE |
| M22 | [精通微服务反模式与重构](./22-精通-反模式与重构.md) | ⭐⭐⭐⭐ | 分布式单体 / 共享数据库 / 过细拆分 / 重新合并 |

---

## 🗺️ 按模块组织

### 🟢 模块一：基础与拆分（M01-M03）

> 微服务不是"必然演进"，而是"组织 + 业务 + 技术"三者达到某个阈值后的工程选择。先把"什么时候用"想清楚，再讨论"怎么用"。

- **M01 架构总览**：单体何时不够、康威定律的两个方向（组织 → 架构、架构 → 组织）、SOA 与微服务史、Microservices vs Macroservices、何时不要做微服务
- **M02 服务拆分（DDD）**：限界上下文（Bounded Context）、聚合根（Aggregate）、领域事件（Domain Event）、战略设计、上下文映射（Anti-Corruption Layer / Conformist / Open Host Service）
- **M03 演进策略**：绞杀者模式（Strangler Fig）、并行运行（Parallel Run）、影子流量、数据库拆分（Database per Service）、渐进式数据迁移

### 🔵 模块二：通信与契约（M04-M06）

- **M04 同步通信**：REST（Richardson 成熟度）、gRPC（HTTP/2 + Protobuf + streaming）、GraphQL Federation（Apollo Router 2026）、超时与背压、协议选型矩阵
- **M05 服务发现**：客户端 vs 服务端发现、Consul / etcd / Eureka / K8s Service、健康检查、DNS-based vs API-based、CAP 选择
- **M06 API 网关与 BFF**：Kong / Envoy / KrakenD / 自研、BFF（Backend for Frontend）、聚合层、协议转换、限流认证

### 🟡 模块三：异步与一致性（M07-M10）

> 微服务"最难"的部分。同步调用容易、异步通信难、跨服务事务最难。这一模块的 4 篇构成微服务的"内核"。

- **M07 异步消息**：消息 vs 事件、Pub-Sub vs Point-to-Point、Kafka / RabbitMQ / NATS / Pulsar 对比、Exactly-Once 语义、消费幂等、死信队列
- **M08 分布式事务**：2PC 与它的死亡、TCC（Try-Confirm-Cancel）、Saga（Orchestration vs Choreography）、Outbox 模式、CDC + Debezium、最终一致性
- **M09 幂等设计**：幂等 key、Token 模式、状态机、去重窗口、幂等存储、PUT vs POST、Idempotency-Key header（Stripe 标准）
- **M10 Event Sourcing 与 CQRS**：事件溯源原理、Append-only log、物化视图（Read Model）、快照（Snapshot）、CQRS、与 Saga 的协同

### 🔴 模块四：韧性（M11-M13）

- **M11 限流熔断降级**：Token Bucket / Leaky Bucket / 滑动窗口、本地限流 vs 分布式限流、Sentinel / Resilience4j / Envoy Rate Limit、自适应限流（基于 RT/CPU/Load）、降级策略
- **M12 韧性设计模式**：Bulkhead（舱壁隔离）、Circuit Breaker（断路器三态）、Retry（指数退避 + 抖动）、Fallback（降级返回）、Timeout（必须有）、Hedged Requests
- **M13 分布式锁与协调**：Redis SETNX / Redlock（争议）、etcd Lease、ZooKeeper、Fencing Token、租约更新与死锁、Quorum 选主

### 🟣 模块五：治理与基础设施（M14-M16）

- **M14 配置中心**：Apollo / Nacos / Consul KV / Spring Cloud Config、热更新机制、灰度配置、多环境、机密管理（与 Vault 集成）
- **M15 服务网格**：sidecar vs Ambient、Istio Ambient（ztunnel + waypoint）、Linkerd、Cilium Service Mesh（eBPF）、mTLS、流量切分、熔断、与 K8s/Gateway API 整合（链到 cloud-native/C10）
- **M16 分布式 ID**：Snowflake 原理与时钟回拨、UUIDv7（PG 18 内置）、美团 Leaf、号段 + 双 Buffer、序列号集中分配、ULID / NanoID

### 🟠 模块六：可观测（M17-M18）

- **M17 分布式链路追踪**：OpenTelemetry（CNCF Graduated）、W3C TraceContext、Span 模型、采样策略（Head vs Tail）、Jaeger / Tempo / Zipkin、链路上下文跨进程传播
- **M18 可观测三支柱**：Metrics（Prometheus）+ Logs（Loki / ELK）+ Traces（Tempo）+ Profiling（continuous profiling、Pyroscope）、关联三者、SLO / SLI / 错误预算、eBPF 可观测

### 🟤 模块七：发布与测试（M19-M20）

- **M19 灰度发布**：Canary / Blue-Green / A/B / Shadow、Argo Rollouts / Flagger 渐进式交付、Feature Flags（LaunchDarkly / GrowthBook / Unleash）、SLO 驱动回滚
- **M20 契约测试与混沌工程**：Pact 契约测试、Schema Registry（Avro/Protobuf）、向后/向前兼容规则、Chaos Mesh / Gremlin / Litmus、生产混沌的伦理

### 🔘 模块八：安全与反模式（M21-M22）

- **M21 微服务安全**：mTLS（服务间）、Zero Trust 架构、服务身份（SPIFFE / SPIRE）、JWT / OAuth 2.1 / OIDC、RBAC / ABAC、密钥轮转、东-西流量加密、网络策略
- **M22 反模式与重构**：分布式单体（最常见反模式）、共享数据库、过细拆分、Chatty Services、Dual Write、Big Ball of Mud 微服务版本、何时合并服务、反向演进

---

## 🎯 学习路径

### 路径 A：完整通学（3-4 个月）

按编号顺序，每周 1-2 篇。每篇配套：
1. 在本地或测试集群跑 demo
2. 阅读官方文档对应章节
3. 做练习题
4. 拿一个真实场景把知识点串起来

### 路径 B：应用开发者切入微服务（1-2 个月）

- **M01 架构总览**（先想清楚要不要做）
- **M04 同步通信**（必备）
- **M07 异步消息**（必备）
- **M09 幂等**（防雷）
- **M11-M12 限流韧性**（生产基本功）
- **M17-M18 可观测**（眼睛）

### 路径 C：架构师 / 技术 Owner（2-3 个月）

- **M02 服务拆分 + M03 演进**（核心责任）
- **M08 分布式事务 + M10 ES/CQRS**（数据一致性）
- **M15 服务网格**（治理基础设施）
- **M19 灰度发布 + M20 契约测试**（交付与质量）
- **M22 反模式**（避坑指南）

### 路径 D：SRE / 平台工程（2 个月）

- **M05 服务发现 + M06 API 网关**（流量入口）
- **M11-M13 韧性三件套**（稳定性）
- **M14 配置 + M15 网格**（治理基础设施）
- **M17-M18 可观测**（生产眼睛）
- **M21 安全**（合规）

### 路径 E：数据一致性专精（1 个月）

- **M07 异步消息**（基础）
- **M08 分布式事务**（核心）
- **M09 幂等**（必备）
- **M10 Event Sourcing / CQRS**（高阶）
- **配合 kafka/ + postgresql/P16 逻辑复制**

### 路径 F：从单体到微服务实战（1-2 个月）

- **M01 架构总览**
- **M02 服务拆分**
- **M03 演进策略**（绞杀者）
- **M22 反模式**（先知道坑在哪儿）
- **M07-M09 异步/事务/幂等**
- **M19 灰度**（小步快跑）

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **必读书**：
  - 《微服务架构设计模式》（Chris Richardson）—— 圣经，几乎每一章都对应一篇
  - 《领域驱动设计》（Eric Evans）—— DDD 原典
  - 《实现领域驱动设计》（Vaughn Vernon）—— 落地版 DDD
  - 《Building Microservices》（Sam Newman）—— 演进与拆分
  - 《Release It!》（Michael Nygard）—— 韧性模式
- **博客**：
  - [microservices.io](https://microservices.io/)（Chris Richardson）
  - [Martin Fowler microservices](https://martinfowler.com/microservices/)
- **关联章节**：
  - cloud-native/ —— 基础设施视角（K8s / Istio / GitOps）
  - backend/ B20-B21 —— 韧性与限流
  - kafka/ —— 消息中间件深度
  - postgresql/ P16 —— 逻辑复制做 CDC

---

## 🛠️ 工具速查

| 任务 | 工具 |
|---|---|
| 服务发现 | Consul / etcd / K8s Service / Eureka |
| API 网关 | Kong / Envoy Gateway / KrakenD / Apollo Router / Apisix |
| 配置中心 | Apollo / Nacos / Consul KV / etcd / Spring Cloud Config |
| 消息 / 事件 | Kafka / RabbitMQ / NATS / Pulsar / Redpanda |
| 服务网格 | Istio Ambient / Linkerd / Cilium Service Mesh / Consul Connect |
| 链路追踪 | OpenTelemetry SDK + Jaeger / Tempo / Zipkin / SkyWalking |
| 指标 | Prometheus + Grafana / VictoriaMetrics |
| 日志 | Loki / ELK / Vector / Fluentbit |
| Profiling | Pyroscope / Parca |
| 限流熔断 | Sentinel / Resilience4j / Envoy Rate Limit / nginx limit_req |
| 分布式锁 | Redis / etcd / ZooKeeper |
| 灰度发布 | Argo Rollouts / Flagger / Istio Traffic Split |
| Feature Flags | LaunchDarkly / GrowthBook / Unleash / FF4j |
| 契约测试 | Pact / Spring Cloud Contract / Schemathesis |
| 混沌工程 | Chaos Mesh / Litmus / Gremlin / Pumba |
| 服务身份 | SPIFFE / SPIRE / Istio Citadel |
| Schema | Confluent Schema Registry / Apicurio |
| 测试 | Testcontainers / Pact / k6 / locust |
| CDC | Debezium + Kafka Connect / Maxwell / Canal |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 说出 5 个"不要做微服务"的信号
- [ ] 用 DDD 战略设计把一个复杂业务切成 5-10 个限界上下文
- [ ] 设计一个从单体到微服务的绞杀者迁移计划（含数据库拆分）
- [ ] 在 REST / gRPC / GraphQL 之间为给定场景做选型并说明理由
- [ ] 解释客户端服务发现 vs 服务端服务发现的取舍
- [ ] 用 Saga（Orchestration）实现一个跨 3 服务的订单流程，处理补偿
- [ ] 用 Outbox + Debezium 实现"业务表 → 事件"的零丢失发布
- [ ] 设计一个支持上亿请求/天的幂等服务（含存储选型与过期策略）
- [ ] 解释 Event Sourcing 何时是正确的选择，何时是过度设计
- [ ] 配置一个自适应限流器（基于 RT/CPU），并解释为什么固定阈值在突发流量下失效
- [ ] 画出 Circuit Breaker 三态转换 + 关键参数（threshold / timeout / half-open trial）
- [ ] 用 Redis Redlock 实现分布式锁，并指出它的争议在哪里
- [ ] 给出 Snowflake 时钟回拨的 3 种解决方案
- [ ] 用 OpenTelemetry SDK 在 Go 应用里打全链路追踪，正确传递 TraceContext
- [ ] 解释 SLO + 错误预算如何驱动发布节奏
- [ ] 设计一个 Canary 发布流程，含失败时的自动回滚
- [ ] 用 Pact 实现 Consumer-Driven Contract 测试
- [ ] 解释 mTLS 在服务网格里是如何自动注入的（Istio Ambient ztunnel）
- [ ] 列出"分布式单体"的 5 个典型症状以及如何重构
- [ ] 给出"微服务过细"何时应该合并服务的判断标准

---

## 🆕 2026 关键变化速查

| 主题 | 2026 必知 |
|---|---|
| **API 通信** | gRPC + Protobuf 已成内部通信主流；REST 主要做外部 API；GraphQL Federation（Apollo Router 2.x）在 BFF 层兴起 |
| **服务发现** | K8s Service 几乎成默认；Consul 在多集群跨云仍主流 |
| **API 网关** | Envoy Gateway / Apisix / Kong 三分天下；Apollo Router（GraphQL Federation）独立赛道 |
| **消息** | Kafka 4（KRaft，无 ZK）成主流；NATS 在轻量场景增长；Pulsar 仍小众但优势明显 |
| **分布式事务** | Saga + Outbox 是事实标准；2PC 几乎绝迹；TCC 在金融领域仍用 |
| **服务网格** | **Istio Ambient Mesh GA（2024-11）** 让 sidecarless 成主流；Cilium Service Mesh（eBPF）继续推进 |
| **可观测** | OpenTelemetry CNCF Graduated；Prometheus 3.0；Pyroscope/Parca 持续 profiling 普及 |
| **灰度发布** | Argo Rollouts / Flagger 主导；Feature Flags 与 GrowthBook（开源）流行 |
| **混沌工程** | Chaos Mesh / Litmus 在 K8s 上主导；生产混沌实践仍在扩散 |
| **服务身份** | SPIFFE / SPIRE 标准化；Istio 默认集成 |
| **零信任** | "Service to Service Zero Trust" 成默认架构；mTLS 全网络 |
| **平台工程** | "Internal Developer Platform"（IDP）兴起；Backstage 流行 |

---

> 准备好你的 Go / Java 项目和一个 K8s 集群，开始学习吧。
