# 后端工程师深度课程 · 中文知识库

> 🚀 **在线体验**：[OfferPilot 后端知识库](https://offerpilot.yzenghe.top/) —— 在线浏览课程内容、按专题学习，并通过练习题自测。

> 一套面向**中级到高级**后端工程师的系统进阶课程，共 **17 大专题、317 篇万字长文**，每篇含底层原理、代码示例、生产实践、陷阱清单与练习题。
>
> **📅 内容基准：2026 年 6 月** —— HTTP/3 主流、TLS 1.3 + post-quantum、PostgreSQL 18、Redis 8 / Valkey、Kafka 4 (KRaft)、Go 1.26、Passkeys、OAuth 2.1 + DPoP、OpenTelemetry、Prometheus 3、Claude 4.x / GPT-5 / MCP 主流化、Kubernetes 1.36、Gateway API GA、Istio Ambient GA、Cilium eBPF；Linux 6.12 LTS（EEVDF / io_uring / cgroup v2 / eBPF / PSI）。

---

## 📚 课程矩阵

| 专题 | 篇数 | 难度 | 适合谁 | 入口 |
|---|---|---|---|---|
| 🟦 **Backend 通用** | 25 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 所有后端工程师 | [backend/](./backend/INDEX.md) |
| 🐹 **Go 语言** | 30 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | Go 中高级工程师 | [golang/](./golang/INDEX.md) |
| ☕ **Java 后端** | 32 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | Java 中高级 / 面试备战 | [java/](./java/INDEX.md) |
| 🐍 **Python 后端** | 30 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | Python 中高级 / AI 后端 / 面试 | [python/](./python/INDEX.md) |
| 🐬 **MySQL** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | DBA / 应用开发 | [mysql/](./mysql/INDEX.md) |
| 🐘 **PostgreSQL** | 22 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | DBA / 应用 / 平台 / AI | [postgresql/](./postgresql/INDEX.md) |
| 🟥 **Redis** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐ | 缓存 / 高并发场景 | [redis/](./redis/INDEX.md) |
| 🍃 **MongoDB** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐ | 文档库使用者 | [mongodb/](./mongodb/INDEX.md) |
| 📨 **Kafka** | 12 | ⭐⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 流式 / 消息系统 | [kafka/](./kafka/INDEX.md) |
| 🔎 **Elasticsearch** | 11 | ⭐⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 搜索 / 日志分析 | [elasticsearch/](./elasticsearch/INDEX.md) |
| 🤖 **AI / LLM 后端** | 16 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | LLM 应用工程化 | [ai-backend/](./ai-backend/INDEX.md) |
| ☸️ **云原生 / K8s** | 16 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 平台 / SRE / 应用上云 | [cloud-native/](./cloud-native/INDEX.md) |
| 🕸️ **微服务架构** | 22 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 架构师 / 后端 / SRE | [microservices/](./microservices/INDEX.md) |
| 🐧 **Linux / 操作系统** | 20 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 后端 / SRE / 平台 | [linux/](./linux/INDEX.md) |
| 🏛️ **系统设计** | 15 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 面试备战 / 后端 / 架构师 | [system-design/](./system-design/INDEX.md) |
| 🧮 **算法面试** | 20 | ⭐⭐ — ⭐⭐⭐⭐⭐ | 面试备战 / 编码面试 | [algorithm/](./algorithm/INDEX.md) |
| 🗣️ **求职与面试软技能** | 10 | ⭐⭐⭐ — ⭐⭐⭐⭐ | 求职全流程 / 简历 / 模拟面试 | [interview-skills/](./interview-skills/INDEX.md) |

- `INDEX.md` —— 总目录、模块划分、学习路径
- `ROADMAP.md` —— Mermaid 可视化路线图
- `QUIZ.md` —— 配套测验与答案

---

## ✨ 这套课程的特点

- **深度优先**：每篇约 1.0–1.5 万字，讲清楚底层原理而不是 API 罗列
- **2026 时效**：跟进 Go 1.26、PostgreSQL 18、Kafka 4.0、Redis 8、TLS 后量子、Claude 4.x / GPT-5 / MCP、Kubernetes 1.36、Istio Ambient、Gateway API 等最新变化
- **生产视角**：每章都有「生产实践」「陷阱清单」「2026 现状」小节
- **可练可考**：每篇附练习题，每个专题附 QUIZ
- **路线图驱动**：Backend 与 Golang 专题基于 [roadmap.sh](https://roadmap.sh) 系统组织

---

## 🎯 推荐学习路径

### 路径 A：Go 后端工程师全栈进阶（6 个月）

```
backend/ B01–B25  →  golang/ G01–G30  →  按业务选数据系统专题
```

### 路径 B：数据库专精（3–4 个月）

```
backend/ B08–B14（索引/事务/分片/复制/NoSQL）
   ↓
mysql/ 全 12 篇  →  postgresql/ 全 22 篇  →  redis/ 全 12 篇  →  mongodb/ 全 12 篇
```

### 路径 C：高并发 / 流式系统（2 个月）

```
backend/ B15 缓存策略  →  redis/ 全套
   ↓
backend/ B17 消息队列  →  kafka/ 全套
   ↓
backend/ B20–B21 韧性 + 限流
```

### 路径 D：搜索 / 检索工程师（1 个月）

```
elasticsearch/ 11 篇全套  →  backend/ B08 索引、B14 NoSQL、B15 缓存
```

### 路径 E：AI / LLM 应用工程师（2 个月）

```
ai-backend/ A01–A03（API 基础）  →  A04–A05（Prompt 与上下文）
   ↓
ai-backend/ A06–A07（Embedding + RAG）+ postgresql/ P06 pgvector + elasticsearch/ 向量检索
   ↓
ai-backend/ A08–A10（Tool Use + Agent + MCP）
   ↓
ai-backend/ A11–A14（Gateway / SSE / 可观测 / 安全）+ backend/ B21 限流、B24 可观测
```

### 路径 G：PostgreSQL 专家（2-3 个月）

```
postgresql/ P01–P03（架构 / 堆表 / MVCC）—— PG vs MySQL 根本差异
   ↓
postgresql/ P04–P06（B-tree / GIN-GiST-BRIN / pgvector）+ P07-P08（隔离锁 / VACUUM）
   ↓
postgresql/ P09–P11（Planner / EXPLAIN / 调优实战）
   ↓
postgresql/ P12–P14（JSONB-全文 / 高级 SQL / 分区） + P18-P19（PostGIS-Timescale-Citus / FDW）
   ↓
postgresql/ P15–P17（WAL / 逻辑复制 / Patroni-PgBouncer） + P20-P22（参数 / 监控 / PG 18 新特性）
```

### 路径 F：云原生 / 平台 SRE 工程师（3-4 个月）

```
cloud-native/ C01–C03（容器与 K8s 基础：Docker / 工作负载 / 网络）
   ↓
cloud-native/ C04–C07（流量入口 / 配置 / 调度 / 存储）
   ↓
cloud-native/ C08–C09（Helm / Kustomize / Operator） + golang/ G14 context、G27 net/http
   ↓
cloud-native/ C10–C12（Service Mesh / 可观测 / 安全） + backend/ B24 可观测
   ↓
cloud-native/ C13–C16（生产调优 / GitOps / Serverless / 多集群）
```

### 路径 H：微服务架构师（3-4 个月）

```
microservices/ M01–M03（架构总览 / DDD 拆分 / 演进策略）
   ↓
microservices/ M04–M06（同步通信 / 服务发现 / 网关 BFF）
   ↓
microservices/ M07–M10（异步消息 / 分布式事务 / 幂等 / ES-CQRS）+ kafka/ 全套
   ↓
microservices/ M11–M16（限流熔断 / 韧性 / 锁 / 配置 / 网格 / 分布式 ID）
   ↓
microservices/ M17–M22（可观测 / 灰度 / 契约测试 / 安全 / 反模式）+ cloud-native/ C10 Service Mesh
```

### 路径 I：Linux 系统 / 内核 / SRE 工程师（3-4 个月）

```
linux/ L01–L03（架构 / 进程 / 调度）
   ↓
linux/ L04–L10（内存 / 文件 / I/O：epoll / io_uring / 块设备）
   ↓
linux/ L11–L17（网络栈 / TCP / Socket / IPC / 同步 / 容器：namespace / cgroup）
   ↓
linux/ L18–L20（性能诊断 / eBPF / systemd）+ cloud-native/ C03 网络、C06 资源、C11 可观测
```

### 路径 J：算法 + 系统设计面试突击（3-4 周）

```
algorithm/ 01 方法论与复杂度  →  04-06 双指针/滑窗/二分（高频模板）
   ↓
algorithm/ 08/12/13 回溯/二叉树/堆  →  15-16 动态规划  →  20 高频题清单
   ↓
system-design/ S01 方法论  →  按高频场景题（秒杀/Feed/IM/排行榜）查漏补缺
```

### 路径 K：Python 后端 / AI 工程师（2-3 个月）

```
python/ P01-P08（语言核心：对象/容器/作用域/生成器/装饰器）
   ↓
python/ P09-P14（OOP 与类型）→ P15-P19（CPython 内幕：执行/GC/GIL/性能）
   ↓
python/ P20-P24（并发与异步：线程/进程/asyncio/选型）+ P28 FastAPI + P29 数据库
   ↓
ai-backend/ 全套（LLM 应用工程化：RAG / Agent / MCP）
```

> 每个专题的 `INDEX.md` 里都有更细致的路径建议（API 工程师特化、性能特化、架构师视角、RAG / Agent 特化、Operator 开发者特化等）。

---

## 📂 目录结构

```
.
├── backend/         # 通用后端 25 篇（网络/API/数据库/架构/安全/运维）
├── golang/          # Go 30 篇（语言/并发/工程化/性能/生态）
├── java/            # Java 后端 32 篇（集合/并发/JVM/Spring/MyBatis/IO/版本演进/虚拟线程/函数式/测试）
├── python/          # Python 后端 30 篇（语言核心/OOP与类型/CPython内幕/GIL/并发异步/工程化/Web/数据/版本演进）
├── mysql/           # MySQL 12 篇（InnoDB/MVCC/复制/调优/9.x 新特性）
├── postgresql/      # PostgreSQL 22 篇（架构/MVCC/索引/pgvector/复制/PostGIS/调优/PG 18）
├── redis/           # Redis 12 篇（数据结构/集群/Streams/Redis 8 / Valkey）
├── mongodb/         # MongoDB 12 篇（BSON/WiredTiger/副本集/分片）
├── kafka/           # Kafka 12 篇（KRaft/KIP-848/事务/Streams/Connect）
├── elasticsearch/   # ES 11 篇（Lucene/分片/Query DSL/BM25/向量检索）
├── ai-backend/      # AI/LLM 16 篇（Claude/OpenAI/RAG/Agent/MCP/Gateway/安全）
├── cloud-native/    # 云原生 16 篇（Docker/K8s/Gateway API/Service Mesh/Operator/GitOps/Serverless/多集群）
├── microservices/   # 微服务 22 篇（DDD/通信/服务发现/分布式事务/韧性/网格/可观测/安全/反模式）
├── linux/           # Linux/OS 20 篇（架构/进程/调度/内存/I/O/网络/IPC/同步/容器/eBPF/systemd）
├── system-design/   # 系统设计 15 篇（方法论/估算/接入/数据层/缓存/秒杀/Feed/IM/网盘/搜索/附近/延迟队列/支付）
├── algorithm/       # 算法面试 20 篇（复杂度/双指针/滑窗/二分/回溯/树图/DP/贪心/字符串/位运算）
└── interview-skills/ # 求职与面试软技能 10 篇（简历/JD对标/自我介绍/项目深挖/行为面试/HR/反问/谈薪/表达复盘）
```

---

## 🛠️ 使用建议

1. **先看 INDEX，再选起点**：每个专题的 INDEX.md 有难度标记和模块说明，按需切入
2. **配合 ROADMAP 建立全景**：先看路线图建立心智模型，再读细节章节
3. **做 QUIZ 验证理解**：读完一个模块用 QUIZ 自检
4. **动手跑示例**：所有代码示例都可直接运行，配合修改加深理解

---

## 🔄 内容版本

- **最近一次内容基准对齐**：2026-06
- **更新原则**：跟随主流技术栈每 6–12 个月迭代一次，重大变更（如 Kafka 5、Go 1.27、PostgreSQL 19）随版本号同步

---

## 🤝 贡献

发现错误或有改进建议，欢迎提 Issue 或 PR：

- 内容勘误：直接 PR 改 markdown
- 新增专题：先开 Issue 讨论范围与基准
- 翻译 / 二创：请保留出处链接

---

## 📜 License

内容采用 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 授权 —— 可自由分享与改编，需署名并以相同协议发布。
