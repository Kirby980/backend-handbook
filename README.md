# 后端工程师深度课程 · 中文知识库

> 一套面向**中级到高级**后端工程师的系统进阶课程，共 **8 大专题、130 篇万字长文**，每篇含底层原理、代码示例、生产实践、陷阱清单与练习题。
>
> **📅 内容基准：2026 年 5 月** —— HTTP/3 主流、TLS 1.3 + post-quantum、PostgreSQL 18、Redis 8 / Valkey、Kafka 4 (KRaft)、Go 1.26、Passkeys、OAuth 2.1 + DPoP、OpenTelemetry、Prometheus 3、Claude 4.x / GPT-5 / MCP 主流化。

---

## 📚 课程矩阵

| 专题 | 篇数 | 难度 | 适合谁 | 入口 |
|---|---|---|---|---|
| 🟦 **Backend 通用** | 25 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 所有后端工程师 | [backend/](./backend/INDEX.md) |
| 🐹 **Go 语言** | 30 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | Go 中高级工程师 | [golang/](./golang/INDEX.md) |
| 🐬 **MySQL** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | DBA / 应用开发 | [mysql/](./mysql/INDEX.md) |
| 🟥 **Redis** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐ | 缓存 / 高并发场景 | [redis/](./redis/INDEX.md) |
| 🍃 **MongoDB** | 12 | ⭐⭐⭐ — ⭐⭐⭐⭐ | 文档库使用者 | [mongodb/](./mongodb/INDEX.md) |
| 📨 **Kafka** | 12 | ⭐⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 流式 / 消息系统 | [kafka/](./kafka/INDEX.md) |
| 🔎 **Elasticsearch** | 11 | ⭐⭐⭐⭐ — ⭐⭐⭐⭐⭐ | 搜索 / 日志分析 | [elasticsearch/](./elasticsearch/INDEX.md) |
| 🤖 **AI / LLM 后端** | 16 | ⭐⭐⭐ — ⭐⭐⭐⭐⭐ | LLM 应用工程化 | [ai-backend/](./ai-backend/INDEX.md) |

- `INDEX.md` —— 总目录、模块划分、学习路径
- `ROADMAP.md` —— Mermaid 可视化路线图
- `QUIZ.md` —— 配套测验与答案

---

## ✨ 这套课程的特点

- **深度优先**：每篇约 1.0–1.5 万字，讲清楚底层原理而不是 API 罗列
- **2026 时效**：跟进 Go 1.26、PostgreSQL 18、Kafka 4.0、Redis 8、TLS 后量子、Claude 4.x / GPT-5 / MCP 等最新变化
- **生产视角**：每章都有「生产实践」「陷阱清单」「2026 现状」小节
- **可练可考**：每篇附练习题，每个专题附 QUIZ
- **路线图驱动**：Backend 与 Golang 专题基于 [roadmap.sh](https://roadmap.sh) 系统组织

---

## 🎯 推荐学习路径

### 路径 A：Go 后端工程师全栈进阶（6 个月）

```
backend/ B01–B25  →  golang/ G01–G30  →  按业务选数据系统专题
```

### 路径 B：数据库专精（2–3 个月）

```
backend/ B08–B14（索引/事务/分片/复制/NoSQL）
   ↓
mysql/ 全 12 篇  →  redis/ 全 12 篇  →  mongodb/ 全 12 篇
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
ai-backend/ A06–A07（Embedding + RAG）+ elasticsearch/ 向量检索
   ↓
ai-backend/ A08–A10（Tool Use + Agent + MCP）
   ↓
ai-backend/ A11–A14（Gateway / SSE / 可观测 / 安全）+ backend/ B21 限流、B24 可观测
```

> 每个专题的 `INDEX.md` 里都有更细致的路径建议（API 工程师特化、性能特化、架构师视角、RAG / Agent 特化等）。

---

## 📂 目录结构

```
.
├── backend/         # 通用后端 25 篇（网络/API/数据库/架构/安全/运维）
├── golang/          # Go 30 篇（语言/并发/工程化/性能/生态）
├── mysql/           # MySQL 12 篇（InnoDB/MVCC/复制/调优/9.x 新特性）
├── redis/           # Redis 12 篇（数据结构/集群/Streams/Redis 8 / Valkey）
├── mongodb/         # MongoDB 12 篇（BSON/WiredTiger/副本集/分片）
├── kafka/           # Kafka 12 篇（KRaft/KIP-848/事务/Streams/Connect）
├── elasticsearch/   # ES 11 篇（Lucene/分片/Query DSL/BM25/向量检索）
└── ai-backend/      # AI/LLM 16 篇（Claude/OpenAI/RAG/Agent/MCP/Gateway/安全）
```

---

## 🛠️ 使用建议

1. **先看 INDEX，再选起点**：每个专题的 INDEX.md 有难度标记和模块说明，按需切入
2. **配合 ROADMAP 建立全景**：先看路线图建立心智模型，再读细节章节
3. **做 QUIZ 验证理解**：读完一个模块用 QUIZ 自检
4. **动手跑示例**：所有代码示例都可直接运行，配合修改加深理解

---

## 🔄 内容版本

- **最近一次内容基准对齐**：2026-05
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
