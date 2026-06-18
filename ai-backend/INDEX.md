# AI / LLM 后端工程深度课程 · 总目录

> 面向 Go / 后端工程师的 LLM 应用工程化系统进阶，共 16 篇万字长文
> 每篇约 10000-15000 字，含底层原理、Go 代码示例、生产实践、陷阱清单与练习题
> 适合从"会调 OpenAI API"到"构建生产级 LLM 系统"的进阶
>
> **📅 内容基准：2026 年 6 月**——Claude Opus 4.8 / Sonnet 4.6 / Haiku 4.5、GPT-5.5、Gemini 3、MCP 主流化、prompt caching 普及、structured output / tool use 稳定、Langfuse / Arize Phoenix 可观测性事实标准、Pinecone / pgvector / Milvus 三足鼎立。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| A01 | [精通 Claude API 工程化](./A01-精通-Claude-API-工程化.md) | ⭐⭐⭐ | Anthropic SDK / prompt caching / batch / files / extended thinking |
| A02 | [精通 OpenAI 兼容生态](./A02-精通-OpenAI-兼容生态.md) | ⭐⭐⭐ | Chat Completions / Responses / Assistants / 兼容层 |
| A03 | [精通 Tokenizer、上下文窗口与计费](./A03-精通-Tokenizer-与计费.md) | ⭐⭐⭐ | BPE / tiktoken / 计费模型 / context window |
| A04 | [精通 Prompt 工程](./A04-精通-Prompt-工程.md) | ⭐⭐⭐⭐ | few-shot / CoT / structured output / 评测 |
| A05 | [精通长对话与上下文管理](./A05-精通-长对话与上下文管理.md) | ⭐⭐⭐⭐ | compaction / sliding window / memory / summarization |
| A06 | [精通 Embedding 与向量库](./A06-精通-Embedding-与向量库.md) | ⭐⭐⭐⭐ | embedding 模型 / pgvector / Pinecone / Milvus / Weaviate |
| A07 | [精通 RAG 架构](./A07-精通-RAG-架构.md) | ⭐⭐⭐⭐⭐ | chunking / hybrid / reranking / 评测 / 优化 |
| A08 | [精通 Tool Use / Function Calling](./A08-精通-Tool-Use.md) | ⭐⭐⭐⭐ | function calling / 多轮 tool 循环 / 错误恢复 |
| A09 | [精通 Agent 系统设计](./A09-精通-Agent-系统设计.md) | ⭐⭐⭐⭐⭐ | ReAct / planning / reflection / memory |
| A10 | [精通 MCP](./A10-精通-MCP.md) | ⭐⭐⭐⭐ | Model Context Protocol / server / resource / tool |
| A11 | [精通 LLM Gateway 与流量治理](./A11-精通-LLM-Gateway.md) | ⭐⭐⭐⭐ | 路由 / 限流 / 降级 / AB 测试 / 多模型 |
| A12 | [精通流式输出与 SSE](./A12-精通-流式输出与-SSE.md) | ⭐⭐⭐⭐ | SSE / token streaming / Go 实战 |
| A13 | [精通 LLM 可观测性](./A13-精通-LLM-可观测性.md) | ⭐⭐⭐⭐ | cost / latency / quality / Langfuse / OTel |
| A14 | [精通 LLM 安全](./A14-精通-LLM-安全.md) | ⭐⭐⭐⭐ | prompt injection / output validation / PII / red team |
| A15 | [精通 LLM Evaluation](./A15-精通-LLM-Evaluation.md) | ⭐⭐⭐⭐ | Ragas / LLM-as-Judge / golden set / 离线 + 在线 / Braintrust |
| A16 | [精通 LLM 成本与延迟优化](./A16-精通-LLM-成本与延迟优化.md) | ⭐⭐⭐⭐ | prompt cache / batch / 分级路由 / TTFT / SLA |

---

## 🗺️ 按模块组织

### 🟢 模块一：API 基础（A01-A03）

> 任何 LLM 应用的第一步——把模型调用工程化。

- **A01 Claude API**：Anthropic SDK、prompt caching（5 分钟 TTL / 1 小时 beta）、batch API、files API、extended thinking、citations
- **A02 OpenAI 生态**：Chat Completions vs Responses API、Assistants 何时用、兼容层（litellm/portkey）选型
- **A03 Token 与计费**：BPE 原理、tiktoken / claude tokenizer、context window 演进、上下文/输出/缓存的差价、batch 半价

### 🔵 模块二：Prompt 与上下文（A04-A05）

> 让模型"听懂"你的需求；让长对话不爆炸。

- **A04 Prompt 工程**：CoT、few-shot 选样、structured output（JSON mode / response_format）、Anthropic 的 XML 标签、prompt 模板与版本化、prompt 评测
- **A05 长对话管理**：compaction 算法、滑动窗口、外部 memory、对话状态机、Claude 内置 memory tool

### 🟡 模块三：检索增强（A06-A07）

> 让模型用你的私有数据回答——RAG 的全部工程。

- **A06 Embedding 与向量库**：embedding 模型选型（OpenAI text-embedding-3、Voyage、BGE、Cohere）、距离度量、pgvector / Pinecone / Milvus / Weaviate / Qdrant 对比
- **A07 RAG 架构**：chunking 策略、混合检索（dense + BM25）、reranking（Cohere / Voyage）、评测（Ragas / 自建集）、常见失败模式

### 🔴 模块四：Agent 与工具（A08-A10）

> 让模型"做事"而不只是"说话"。

- **A08 Tool Use**：Anthropic tool_use / OpenAI function calling、JSON schema 设计、多轮 tool 循环、错误恢复、parallel tool calls
- **A09 Agent 系统**：ReAct、Plan-and-Execute、Reflection、ReWOO、Multi-Agent、Anthropic 的 "agentic loop" 实践
- **A10 MCP**：Model Context Protocol（2024-11 推出，2026 主流）、server / resource / tool / sampling、跟传统 API 的区别

### 🟣 模块五：生产化（A11-A16）

> 把 demo 跑成可上线的系统。

- **A11 LLM Gateway**：多模型路由、限流（按 token / 请求 / 用户）、降级与 fallback、AB 测试、成本控制
- **A12 流式输出**：SSE 协议、Go 实现（net/http）、token-level vs message-level、断线重连、Backpressure
- **A13 可观测性**：trace + metric + log 三柱、Langfuse / Arize Phoenix / Helicone、OpenTelemetry GenAI semantic convention、cost / latency / quality 三维监控
- **A14 安全**：prompt injection（direct / indirect）、jailbreak、output validation（Guardrails / NeMo）、PII redaction、红队测试
- **A15 Evaluation**：RAG triad、Ragas、LLM-as-Judge（位置 / 长度 / 自我偏置）、离线 + 在线评测、CI 门禁、A/B 实验
- **A16 成本与延迟**：prompt cache（5min / 1h）、batch API、模型路由分级、TTFT 优化、Agent 预算守护、多 provider fallback、SLA

---

## 🎯 学习路径建议

### 路径 A：完整通学（3 个月）

按编号顺序，每周 2-3 篇。每篇配套：
1. 跑通课程里的 Go 代码示例
2. 在自己的 demo 项目中应用
3. 做练习题

### 路径 B：RAG 工程师特化（1 个月）

- **A01-A02**（API 基础）
- **A03**（计费 / context window）
- **A06**（embedding + 向量库）
- **A07**（RAG 完整架构）
- **A13**（可观测性）

### 路径 C：Agent 工程师特化（1 个月）

- **A01**（Claude API）
- **A04**（Prompt 工程）
- **A05**（长对话管理）
- **A08**（Tool use）
- **A09**（Agent 系统）
- **A10**（MCP）

### 路径 D：LLM 平台工程师（1-2 个月）

- **A02-A03**（多模型 / 计费）
- **A11**（Gateway）
- **A12**（流式输出）
- **A13**（可观测性）
- **A14**（安全）
- **A15-A16**（评测 + 成本延迟）

### 路径 E：快速入门（2 周）

- **A01**（Claude）+ **A02**（OpenAI）：API 全打通
- **A04**（Prompt 工程）：第一周末
- **A07**（RAG）：第二周末
- **A12**（SSE）：搞定一个能 chat 的产品

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)
- **官方文档**：
  - [Anthropic API](https://docs.anthropic.com/)
  - [OpenAI API](https://platform.openai.com/docs)
  - [MCP](https://modelcontextprotocol.io/)
  - [Langfuse](https://langfuse.com/)
- **Go SDK**：
  - [`github.com/anthropics/anthropic-sdk-go`](https://github.com/anthropics/anthropic-sdk-go)
  - [`github.com/sashabaranov/go-openai`](https://github.com/sashabaranov/go-openai)
  - [`github.com/mark3labs/mcp-go`](https://github.com/mark3labs/mcp-go)

---

## 🛠️ 工具速查

| 任务 | 工具 / 命令 |
|---|---|
| Token 数估算 | `tiktoken`（OpenAI）/ Anthropic `count_tokens` |
| Prompt 模板 | LangChain / LlamaIndex 模板 / 自建版本控制 |
| Prompt 评测 | promptfoo / Ragas / 自建评测集 + LLM-as-judge |
| 向量库本地 | pgvector / Qdrant / Chroma |
| 向量库托管 | Pinecone / Weaviate Cloud / Milvus Cloud |
| Reranker | Cohere Rerank / Voyage rerank / bge-reranker |
| LLM Gateway | litellm / Portkey / Helicone / 自建 |
| 可观测性 | Langfuse / Arize Phoenix / Helicone / Datadog LLM |
| 流式调试 | `curl -N` / `httpie --stream` / SSE 客户端 |
| 安全扫描 | Garak / promptfoo red team / Lakera |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 解释 prompt caching 的命中条件与计费规则
- [ ] 在 Go 里实现一个支持 streaming + tool use 的 Claude / OpenAI 客户端
- [ ] 设计一个 RAG 系统：选 embedding、选向量库、写 chunking、加 reranking、做评测
- [ ] 写一个多轮 tool use 循环，处理 tool error 与超时
- [ ] 用 Anthropic 的 agentic loop 写一个能"自主完成任务"的 Agent
- [ ] 实现一个 MCP server 暴露内部工具
- [ ] 设计一个支持多 provider 路由、限流、降级的 LLM Gateway
- [ ] 用 SSE 把 token stream 推给前端，支持断线重连
- [ ] 接 Langfuse 监控 trace / cost / latency
- [ ] 防御 prompt injection（system prompt 加固、output schema 验证、PII 过滤）
- [ ] 用 Ragas + LLM-as-Judge 跑离线 RAG 评测并在 CI 卡回归
- [ ] 用 prompt cache + 分级路由把生产 LLM 成本降到原来的 30% 而质量无显著下降

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **A01 Claude** | Claude 4.8 Opus / 4.6 Sonnet / 4.5 Haiku；Sonnet 4.6 支持 1M context；prompt caching 5min/1h；extended thinking GA；citations、files、batch、memory tool 等多个工具就绪 |
| **A02 OpenAI** | GPT-5 系列发布；Responses API 替代 Chat Completions 成主推；Assistants v2 → 待并入 Responses |
| **A03 Token** | 主流模型上下文：Claude 1M / Gemini 1M / GPT-5 400k；prompt caching 让长上下文经济性可控 |
| **A07 RAG** | hybrid retrieval 成标配；late interaction（ColBERT/ColPali）兴起；BBQ 量化（ES 8.18 GA）让大库可负担 |
| **A09 Agent** | Anthropic agentic loop / OpenAI agents SDK / LangGraph 三大方向；多 Agent 协作模式（Orchestrator-Worker）落地 |
| **A10 MCP** | MCP 已是 IDE / Agent / Tool 的事实标准（Claude Code、Cursor、Windsurf 全部原生支持）；Anthropic / OpenAI / Google 联合推进 |
| **A11 Gateway** | LiteLLM、Portkey、Helicone 成熟；OpenAI / Anthropic / Bedrock / Vertex 多源路由 |
| **A13 可观测性** | OpenTelemetry GenAI semantic convention 稳定；Langfuse 开源版成事实标准 |
| **A14 安全** | OWASP LLM Top 10 (2025) 是基线；NIST AI RMF 在企业普及 |
