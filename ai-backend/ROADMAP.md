# AI / LLM 后端工程路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 5 月**——Claude 4.x / GPT-5 / Gemini 2.5、MCP 主流化、prompt caching 普及、Langfuse / Phoenix 事实标准、pgvector / Pinecone / Milvus 三足鼎立。

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始 AI 后端之旅]) --> M1[模块 1: API 基础]

    M1 --> A01[A01 Claude API]
    M1 --> A02[A02 OpenAI 生态]
    M1 --> A03[A03 Token 与计费]

    A03 --> M2[模块 2: Prompt 与上下文]
    M2 --> A04[A04 Prompt 工程]
    M2 --> A05[A05 长对话管理]

    A05 --> M3[模块 3: 检索增强]
    M3 --> A06[A06 Embedding + 向量库]
    M3 --> A07[A07 RAG 架构]

    A07 --> M4[模块 4: Agent 与工具]
    M4 --> A08[A08 Tool Use]
    M4 --> A09[A09 Agent 系统]
    M4 --> A10[A10 MCP]

    A10 --> M5[模块 5: 生产化]
    M5 --> A11[A11 LLM Gateway]
    M5 --> A12[A12 流式 + SSE]
    M5 --> A13[A13 可观测性]
    M5 --> A14[A14 安全]
    M5 --> A15[A15 Evaluation]
    M5 --> A16[A16 成本与延迟]

    A16 --> End([生产级 LLM 工程师])

    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef api fill:#48bb78,stroke:#2f855a,color:#fff
    classDef prompt fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef rag fill:#ecc94b,stroke:#b7791f,color:#000
    classDef agent fill:#9f7aea,stroke:#6b46c1,color:#fff
    classDef prod fill:#ed8936,stroke:#c05621,color:#fff

    class M1,M2,M3,M4,M5 module
    class A01,A02,A03 api
    class A04,A05 prompt
    class A06,A07 rag
    class A08,A09,A10 agent
    class A11,A12,A13,A14,A15,A16 prod
```

---

## 🟢 模块 1：API 基础（A01-A03）

```mermaid
graph LR
    A01[A01 Claude API<br>caching/batch/thinking] --> A02[A02 OpenAI 生态<br>Chat/Responses]
    A02 --> A03[A03 Token+计费<br>BPE/tiktoken]

    classDef api fill:#48bb78,stroke:#2f855a,color:#fff
    class A01,A02,A03 api
```

**API 选型决策**：

```mermaid
flowchart TD
    Start[新建 LLM 应用] --> Need{首要需求?}
    Need -->|超长上下文 + 推理| Claude["Claude Sonnet/Opus<br>1M context"]
    Need -->|低延迟低成本| Haiku["Claude Haiku / GPT-5-mini"]
    Need -->|多模态 + 长视频| Gemini[Gemini 2.5]
    Need -->|生态 + Assistants| GPT[GPT-5 / Responses API]
    Need -->|自托管 + 隐私| Local["Llama / Qwen / DeepSeek 本地"]

    Claude --> Cache{开启 prompt caching?}
    Cache -->|是| Save["省 90% input cost<br>5min/1h TTL"]
    Cache -->|否| Bill[正常计费]

    style Claude fill:#4299e1,color:#fff
    style Save fill:#48bb78,color:#fff
```

---

## 🔵 模块 2：Prompt 与上下文（A04-A05）

```mermaid
graph TB
    A04[A04 Prompt 工程]
    A05[A05 长对话管理]

    A04 --> Tech{技巧}
    Tech --> Few[few-shot]
    Tech --> CoT[Chain of Thought]
    Tech --> JSON[Structured Output]
    Tech --> Tag[XML 标签<br>Anthropic 专用]

    A05 --> Long{超长对话}
    Long --> Slide[滑动窗口]
    Long --> Compact[Compaction<br>压缩历史]
    Long --> Mem[外部 memory<br>retrieval]

    classDef prompt fill:#4299e1,stroke:#2b6cb0,color:#fff
    class A04,A05 prompt
```

---

## 🟡 模块 3：检索增强（A06-A07）

```mermaid
graph LR
    Doc[文档/PDF/网页]
    Doc --> Chunk[Chunking<br>recursive/sentence/semantic]
    Chunk --> Embed[Embedding<br>OpenAI/Voyage/BGE]
    Embed --> VDB[(向量库<br>pgvector/Pinecone/Milvus)]

    Query[用户问题] --> QEmbed[Query Embedding]
    QEmbed --> Search{检索}
    VDB --> Search
    Search --> Hybrid[Hybrid<br>dense + BM25]
    Hybrid --> Rerank[Reranker<br>Cohere/Voyage]
    Rerank --> Context[Top-K 上下文]
    Context --> LLM[LLM 生成]

    style VDB fill:#ecc94b
    style Rerank fill:#f56565,color:#fff
    style LLM fill:#48bb78,color:#fff
```

**向量库选型**：

```mermaid
flowchart TD
    Scale{规模} -->|< 1000 万 + 已有 PG| PG[pgvector<br>简单可控]
    Scale -->|托管 + 简单| Pine[Pinecone]
    Scale -->|大规模 + 自托管| Mil[Milvus / Qdrant]
    Scale -->|混合 dense+sparse 内置| Wea[Weaviate]
    Scale -->|已用 ES + 不想新栈| ES[Elasticsearch 向量字段]

    style PG fill:#48bb78,color:#fff
    style Mil fill:#9f7aea,color:#fff
```

---

## 🔴 模块 4：Agent 与工具（A08-A10）

```mermaid
graph TB
    User[用户请求]
    User --> Agent[Agent Loop]
    Agent --> Think[思考/规划]
    Think --> Decide{需要工具?}
    Decide -->|是| Tool[调用 Tool]
    Tool --> Observe[观察结果]
    Observe --> Think
    Decide -->|否| Reply[直接回复]
    Reply --> End[结束]

    Tool -.可选标准.-> MCP[MCP server<br>统一协议]

    style Agent fill:#9f7aea,color:#fff
    style MCP fill:#ed8936,color:#fff
```

**Agent 模式对比**：

```mermaid
flowchart LR
    Style{Agent 模式}
    Style -->|单一循环| ReAct["ReAct<br>think→act→observe"]
    Style -->|先规划后执行| PE["Plan-Execute<br>分两阶段"]
    Style -->|多 Agent 协作| Multi["Orchestrator<br>+ Workers"]
    Style -->|Anthropic 推荐| Loop["agentic loop<br>消息循环"]

    style Loop fill:#4299e1,color:#fff
```

---

## 🟣 模块 5：生产化（A11-A14）

```mermaid
graph TB
    Client[Client] --> GW[A11 LLM Gateway<br>路由/限流/降级]
    GW --> Stream[A12 SSE 流式<br>token-level]
    Stream --> LLM[LLM Provider]

    GW -.指标.-> Obs[A13 可观测性<br>Langfuse/Phoenix]
    GW -.防护.-> Sec[A14 安全<br>injection/PII]

    classDef prod fill:#ed8936,color:#fff
    class GW,Stream,Obs,Sec prod
```

**生产架构图**：

```mermaid
graph TB
    User[用户] --> WAF[WAF]
    WAF --> APIGW[API Gateway<br>认证/限流]
    APIGW --> LLMGW[LLM Gateway<br>多模型路由]

    LLMGW --> Cache{Semantic Cache}
    Cache -->|命中| Return[直接返回]
    Cache -->|未命中| Route{Route}

    Route --> Claude[Claude API]
    Route --> OpenAI[OpenAI API]
    Route --> Local[本地模型]

    Claude & OpenAI & Local --> Filter[Output Filter<br>PII/Toxicity]
    Filter --> SSE[SSE Stream]
    SSE --> User

    LLMGW -.trace.-> Lang[Langfuse]
    LLMGW -.metric.-> Prom[Prometheus]

    style LLMGW fill:#ed8936,color:#fff
    style Cache fill:#48bb78,color:#fff
    style Filter fill:#f56565,color:#fff
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（3 个月）

```mermaid
gantt
    title 3 个月 AI 后端通学
    dateFormat YYYY-MM-DD
    section 月 1
    A01-A03 API 基础            :a1, 2026-05-12, 14d
    A04-A05 Prompt 与上下文     :a2, after a1, 14d
    section 月 2
    A06-A07 RAG 全套            :a3, after a2, 21d
    A08 Tool Use                :a4, after a3, 7d
    section 月 3
    A09-A10 Agent + MCP         :a5, after a4, 14d
    A11-A14 生产化              :a6, after a5, 14d
```

### 路径 B：RAG 工程师特化

```mermaid
graph LR
    R1[A01-A02<br>API 基础] --> R2[A03 计费]
    R2 --> R3[A06 Embedding<br>+ 向量库]
    R3 --> R4[A07 RAG 架构]
    R4 --> R5[A13 可观测性]

    style R3 fill:#ecc94b
    style R4 fill:#f56565,color:#fff
```

### 路径 C：Agent 工程师特化

```mermaid
graph LR
    G1[A01 Claude] --> G2[A04 Prompt]
    G2 --> G3[A05 长对话]
    G3 --> G4[A08 Tool Use]
    G4 --> G5[A09 Agent]
    G5 --> G6[A10 MCP]

    style G5 fill:#9f7aea,color:#fff
    style G6 fill:#ed8936,color:#fff
```

---

## 🧠 AI 后端核心知识思维导图

```mermaid
mindmap
  root((AI 后端))
    API
      Claude A01
      OpenAI A02
      Token A03
    Prompt
      工程 A04
      长对话 A05
    检索
      Embedding A06
      RAG A07
    Agent
      Tool Use A08
      系统 A09
      MCP A10
    生产
      Gateway A11
      Streaming A12
      可观测 A13
      安全 A14
```

---

## 📊 难度与重要性矩阵

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| A01 Claude API | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 任何 LLM 项目的起点 |
| A02 OpenAI 生态 | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 必备第二个 provider |
| A03 Token 与计费 | ⭐⭐⭐ | 🔥🔥🔥🔥 | 成本失控的常见根因 |
| A04 Prompt 工程 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 决定模型上限 |
| A05 长对话 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 聊天产品必踩坑 |
| A06 Embedding+向量库 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | RAG 前置 |
| A07 RAG | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | LLM 最常见应用形态 |
| A08 Tool Use | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | Agent 基本盘 |
| A09 Agent | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 2026 最热方向 |
| A10 MCP | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 新协议但已成主流 |
| A11 LLM Gateway | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 多 provider 必须 |
| A12 SSE | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | UX 决定生死 |
| A13 可观测性 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 没有就是黑盒 |
| A14 安全 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 上线前必看 |

---

## 🔗 与已有课程的关系

| AI 后端章节 | 关联已有课程 |
|---|---|
| A01 Claude API | backend/B02 HTTP、B22 认证 |
| A06 向量库 | backend/B14 NoSQL、elasticsearch/08 向量检索 |
| A07 RAG | elasticsearch 全套（特别是 BM25 + 向量） |
| A11 LLM Gateway | backend/B20 韧性、B21 限流 |
| A12 SSE | golang/G27 net/http、backend/B02 HTTP |
| A13 可观测性 | backend/B24 可观测性、golang/G22 pprof |
| A14 安全 | backend/B23 OWASP（含 OWASP LLM Top 10）、B22 认证 |

---

## 🆕 2026 关键技术演进图

```mermaid
graph LR
    subgraph 模型层
    M1[Claude 3.5] --> M2[Claude 4.x<br>1M context]
    M3[GPT-4] --> M4[GPT-5<br>Responses API]
    M5[Gemini 1.5] --> M6[Gemini 2.5<br>2M context]
    end

    subgraph 协议层
    P1[Function Call] --> P2[Tool Use<br>+ parallel]
    P3[Custom API] --> P4[MCP<br>2024-11 推出]
    end

    subgraph 工程层
    E1[手写 prompt] --> E2[Prompt 工程化<br>版本化+评测]
    E3[简单 RAG] --> E4[Hybrid + Rerank<br>+ 评测]
    E5[基础日志] --> E6[Langfuse / Phoenix<br>OTel GenAI]
    end

    subgraph 安全层
    S1[无防护] --> S2[OWASP LLM Top 10]
    S2 --> S3[Garak + 红队]
    end

    style M2 fill:#fff3e0
    style P4 fill:#fff3e0
    style E6 fill:#fff3e0
    style S3 fill:#fff3e0
```
