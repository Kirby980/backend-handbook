# AI / LLM 后端工程路线图 · 知识检测测验

> 每章 5 道题，共 85 题。答案在最末尾"参考答案"章节。
> 题目类型：单选、概念辨析、代码/配置输出。
> 建议：先盖住答案，独立完成；> 80% 正确视为掌握该章；< 50% 建议重读。

---

## A01 — Claude API 工程化

**1.1 ⭐** Claude prompt caching 默认的 cache 生存期（TTL）是？
- A. 24 小时
- B. 1 小时
- C. 5 分钟
- D. 60 秒

**1.2 ⭐** 关于 Claude Messages API 的 `tool_use` / `tool_result`，下面说法**错误**的是？
- A. `tool_use` 出现在 assistant 角色的消息里
- B. `tool_result` 出现在 user 角色的消息里
- C. 一次 assistant 回复中可以包含多个 `tool_use`（parallel tool calls）
- D. `tool_result` 中的 `content` 必须是 JSON 字符串

**1.3 ⭐⭐** 以下哪种情况**不会**使 prompt caching 命中？
- A. 第二次请求 `system` 完全相同、最后追加了 user 消息
- B. 第二次请求把同一 system prompt 拆成两段
- C. 第二次请求在 cache 写入后 4 分钟发起
- D. 第二次请求换了 model（Sonnet → Haiku）

**1.4 ⭐⭐** Claude Batch API 主要省的是？
- A. output token 单价
- B. input token 单价（约 50%）
- C. 延迟
- D. 请求数限额

**1.5 ⭐⭐⭐** 调 Claude API 收到 `overloaded_error` 时**最稳健**的策略是？
- A. 立即原 prompt 重试
- B. 指数退避 + jitter，并降级到更小模型
- C. 抛错给上游
- D. 切到 OpenAI

---

## A02 — OpenAI 兼容生态

**2.1 ⭐** 2026 年 OpenAI 主推、统一了 Assistants / tools / state 的 API 是？
- A. Chat Completions
- B. Responses API
- C. Completions（旧版）
- D. Assistants v1

**2.2 ⭐** OpenAI `response_format` 设为 `json_schema` 时，下列哪条最关键？
- A. 模型一定不会输出非法 JSON
- B. 必须指定 `strict: true` 才能保证字段类型严格符合 schema
- C. 它会自动加 system prompt
- D. 只对 GPT-5 生效

**2.3 ⭐⭐** OpenAI 兼容协议在生态中的作用主要是？
- A. 让所有模型互相替代
- B. 提供事实标准，让 vLLM / Together / DeepSeek 等可以用同一份客户端代码
- C. 替代 gRPC
- D. 安全协议

**2.4 ⭐⭐** Chat Completions 与 Responses API 的核心差别**不**包括？
- A. Responses 内建对 tools / state 的管理
- B. Responses 支持多轮交互而 Chat Completions 不行
- C. Responses 把 Assistants 的能力合并进来
- D. Responses 更接近"一次调用 ↔ 一次 agent step"

**2.5 ⭐⭐⭐** 如果你需要在 100 万 token 上下文上做问答，下列模型组合最**经济**？
- A. GPT-5 Chat Completions
- B. Claude Sonnet 5 + prompt caching
- C. o3-mini 反复调用
- D. GPT-5-mini 不开缓存

---

## A03 — Tokenizer 与计费

**3.1 ⭐** BPE 的核心思想是？
- A. 按字符切
- B. 按词典词切
- C. 按高频字节对反复合并
- D. 按字节切

**3.2 ⭐** OpenAI 用的是哪个 tokenizer？
- A. SentencePiece
- B. tiktoken（基于 BPE）
- C. WordPiece
- D. Unigram

**3.3 ⭐⭐** 关于 Claude 与 OpenAI 的 tokenizer**正确**的是？
- A. 二者使用同一份词表
- B. Claude 的 tokenizer 和 OpenAI 不兼容，但中文 token 数大体接近
- C. Claude 不支持按 token 计费
- D. Claude 提供本地 tokenizer 包，无需 API 调用

**3.4 ⭐⭐** 同一段长 prompt，cached input、cache write、normal input 三种计费**通常**的相对关系？
- A. cache write < cached input < normal input
- B. cached input < normal input < cache write
- C. cached input < cache write < normal input（cache write 通常约 1.25 × normal）
- D. 三者相同

**3.5 ⭐⭐⭐** 长上下文应用最**重要**的成本控制手段不是？
- A. 启用 prompt caching
- B. 用 batch API 跑非实时任务
- C. 把模型替换为 Haiku
- D. 把对话历史 compaction

---

## A04 — Prompt 工程

**4.1 ⭐⭐** Chain of Thought（CoT）最适合**哪类**任务？
- A. 单步分类
- B. 多步推理 / 数学 / 复杂判断
- C. 翻译
- D. 实体抽取

**4.2 ⭐⭐** Anthropic Claude 推荐的"结构化输出"主要手段是？
- A. JSON mode
- B. XML 标签 + 在 assistant 角色 prefill 起始 tag
- C. function call
- D. 改 temperature 为 0

**4.3 ⭐⭐⭐** Few-shot 选择示例时**最差**的做法是？
- A. 选与当前样本语义最相似的
- B. 用 BM25 / 向量召回示例库
- C. 始终用固定 3 条（不区分样本）
- D. 兼顾正负样本与边界 case

**4.4 ⭐⭐⭐** Prompt 评测最适合用作"自动化回归"的是？
- A. 人工 5 星打分
- B. LLM-as-judge + 黄金集对照
- C. 让模型自评
- D. BLEU 分数

**4.5 ⭐⭐⭐⭐** 关于 system prompt 和 user prompt 的相对权重，**正确**的是？
- A. user prompt 优先级最高
- B. system prompt 给模型"角色 / 规则 / 不可变约束"，在对话长后比 user 更稳定
- C. system prompt 不影响 token 计费
- D. system prompt 不能用 XML

---

## A05 — 长对话与上下文管理

**5.1 ⭐⭐** "Lost in the middle"指的是？
- A. 模型在长上下文中间位置的内容召回率下降
- B. 中间消息会被自动丢弃
- C. SSE 流式中段会丢
- D. 中间的 user 消息权重最低

**5.2 ⭐⭐** 滑动窗口策略最大的问题是？
- A. 慢
- B. 早期重要事实（如用户身份、约定）会被丢弃
- C. 容易内存溢出
- D. 不兼容 streaming

**5.3 ⭐⭐⭐** Compaction 触发时机最合理的是？
- A. 每来一条消息就 compact 一次
- B. 当对话 token 数超过阈值（如 context window 的 60-70%）
- C. 用户主动点"清理"
- D. 永远不 compact，让 context 自然增长

**5.4 ⭐⭐⭐** Anthropic 的 memory tool 设计上**不**包括下列哪个能力？
- A. 模型主动写入 memory
- B. 后续会话能召回历史 memory
- C. 自动跨用户共享 memory
- D. memory 可以是结构化字段

**5.5 ⭐⭐⭐⭐** 把"对话状态"和"对话历史"分离的好处**不**包括？
- A. 业务状态（如订单 id）从 system 加载，不依赖 LLM 重述
- B. 历史压缩不会丢业务关键字段
- C. 状态可以直接走 DB 查询，提高一致性
- D. 减少 system prompt 总 token

---

## A06 — Embedding 与向量库

**6.1 ⭐⭐** 余弦相似度与 dot product 的关系是？
- A. 完全不同
- B. 向量归一化后两者等价
- C. dot product 总是 ≥ cosine
- D. 仅在 2D 下相同

**6.2 ⭐⭐** pgvector 默认推荐的索引是？
- A. ivfflat
- B. HNSW
- C. brute force
- D. LSH

**6.3 ⭐⭐⭐** 关于 embedding 维度迁移（如 1536 → 3072），**正确**的是？
- A. 直接重建索引即可
- B. 不同维度之间的向量无法对比，必须全量重 embedding 并迁移索引
- C. pgvector 自动处理
- D. 与上层完全无关

**6.4 ⭐⭐⭐** Binary quantization（如 BBQ）的核心代价是？
- A. 大幅增加内存
- B. recall 略降，但内存通常省到 1/32
- C. 完全无损
- D. 必须配合 GPU

**6.5 ⭐⭐⭐⭐** Pinecone 与 Milvus 自托管的主要区别**不**包括？
- A. Pinecone 是托管 SaaS，Milvus 通常自部署
- B. Pinecone 不支持向量过滤
- C. Milvus 适合超大规模 + 自有运维能力
- D. Pinecone serverless 模型更适合按量

---

## A07 — RAG 架构

**7.1 ⭐⭐** RAG 中"chunking 切得太细"最直接的后果是？
- A. 索引膨胀
- B. 单 chunk 内的语义不完整，召回后模型理解失败
- C. embedding 计算变慢
- D. 检索精度提升

**7.2 ⭐⭐** Hybrid retrieval 通常是把**哪两种**信号融合？
- A. dense（向量）+ sparse（BM25 / SPLADE）
- B. dense + 关键词
- C. dense + LLM 直接答题
- D. BM25 + 编辑距离

**7.3 ⭐⭐⭐** Reranker 在 RAG 中放在哪一步？
- A. embedding 前
- B. 检索召回之后、最终上下文组装之前
- C. 模型生成之后
- D. 仅用于评测

**7.4 ⭐⭐⭐** 关于 Ragas 三大指标，**错误**的是？
- A. faithfulness 衡量答案是否被 retrieved context 支持
- B. context_precision 衡量 retrieved context 中有多少与问题相关
- C. answer_relevancy 衡量答案与问题的相关度
- D. faithfulness 衡量答案与 ground truth 的字面一致

**7.5 ⭐⭐⭐⭐** RAG 的常见失败模式**不**包括？
- A. embedding 版本漂移导致召回退化
- B. lost in the middle 让中段证据被忽略
- C. 模型 temperature=0 就不会幻觉
- D. 上下文太满，反而稀释了答案信号

---

## A08 — Tool Use / Function Calling

**8.1 ⭐⭐** Anthropic Claude 多轮 tool use 中，包含 tool 调用结果的消息角色应是？
- A. assistant
- B. system
- C. user
- D. tool

**8.2 ⭐⭐** Parallel tool calls 的关键好处是？
- A. 节省 token
- B. 一次推理里发起多个独立 tool，整轮 latency 下降
- C. 让模型答得更准
- D. 自动错误恢复

**8.3 ⭐⭐⭐** Tool 调用失败时，最佳做法是？
- A. 抛异常给客户端，让对话终止
- B. 把错误以 `tool_result` 形式返还模型，让模型决定是否重试或换路径
- C. 自动重试 10 次
- D. 把错误写日志后忽略

**8.4 ⭐⭐⭐** Tool schema 中 description 写得好的**最重要价值**是？
- A. 给前端展示
- B. 帮助模型判断"该不该调"、"参数怎么填"
- C. 节省 token
- D. 通过 OpenAPI 校验

**8.5 ⭐⭐⭐⭐** Streaming 模式下，处理 tool 调用参数最常见的难点是？
- A. SSE 不支持 tool call
- B. 模型按 token 增量输出 JSON，必须等结束或用 partial JSON parser 增量解析
- C. tool call 必须关闭 streaming
- D. 一定要 buffer 整段才能用

---

## A09 — Agent 系统设计

**9.1 ⭐⭐⭐** ReAct 模式中的"Act"具体是？
- A. 模型生成最终答案
- B. 模型调用工具（产生外部影响 / 获取信息）
- C. 模型反思
- D. 用户操作

**9.2 ⭐⭐⭐** Plan-and-Execute 相对 ReAct 的主要差别是？
- A. P&E 不需要 LLM
- B. P&E 先一次性产出整体计划，再分步执行；ReAct 是循环里逐步决定
- C. ReAct 不能调 tool
- D. P&E 始终更快

**9.3 ⭐⭐⭐⭐** Anthropic 推荐的"agentic loop"基本组成**不**包括？
- A. tool 调用循环
- B. 检查 `stop_reason`
- C. 强制串行（绝不并行）
- D. 上下文管理

**9.4 ⭐⭐⭐⭐** 生产 Agent 系统**必须**有的"防失控"机制**不**包括？
- A. max_steps / max_tokens 上限
- B. 单次会话成本上限
- C. 让 Agent 不限次数地自我反思
- D. 工具白名单 + 危险工具人工审批

**9.5 ⭐⭐⭐⭐⭐** Multi-Agent "Orchestrator-Worker"模式中 Orchestrator 的核心职责是？
- A. 替 Worker 跑 tool
- B. 拆任务、分派 Worker、聚合结果、决定是否再迭代
- C. 写日志
- D. 校验 schema

---

## A10 — MCP（Model Context Protocol）

**10.1 ⭐⭐⭐** MCP 解决的核心问题是？
- A. 替代 HTTP
- B. 把"模型 ↔ 工具 / 数据源"集成从 N×M 降到 N+M
- C. 加速 LLM 推理
- D. 让 LLM 自动学新工具

**10.2 ⭐⭐⭐** MCP 协议底层基于？
- A. gRPC
- B. JSON-RPC 2.0（可走 stdio / SSE / HTTP）
- C. GraphQL
- D. WebSocket 自定义协议

**10.3 ⭐⭐⭐⭐** MCP 的四类核心原语**不**包括？
- A. Tools
- B. Resources
- C. Prompts
- D. Pipelines

**10.4 ⭐⭐⭐⭐** 远程 MCP server 推荐的鉴权机制是？
- A. Basic Auth
- B. OAuth 2.1
- C. 自签 token
- D. 不鉴权（仅靠传输安全）

**10.5 ⭐⭐⭐⭐⭐** Sampling 原语让 MCP server 能做什么？
- A. 限制速率
- B. 反向请求 client 代为调用 LLM（让 server 也能"用 LLM"）
- C. 采集日志
- D. 配置 tool

---

## A11 — LLM Gateway

**11.1 ⭐⭐⭐** LLM Gateway 与传统 API Gateway **新增**的特性**不**包括？
- A. token 维度限流
- B. 多 provider 路由 + fallback
- C. semantic cache（按语义命中）
- D. 字段级 SQL 转写

**11.2 ⭐⭐⭐** Semantic cache 的命中逻辑通常是？
- A. 严格 SHA256 命中
- B. query embedding 与缓存 embedding 余弦相似度超过阈值
- C. 关键词全文匹配
- D. URL hash

**11.3 ⭐⭐⭐⭐** Token-based rate limit 通常按下列哪种维度同时统计？
- A. 仅按用户
- B. 用户 / API key / 模型 / 时间窗多维度
- C. 仅按 IP
- D. 仅按 endpoint

**11.4 ⭐⭐⭐⭐** LLM 降级链最合理的设计是？
- A. 首选最贵模型，失败立刻 502
- B. 主模型超时/限流时降级到等效或更小的模型，并明确成本/质量影响
- C. 永远只用一个模型
- D. 随机选模型

**11.5 ⭐⭐⭐⭐⭐** litellm / Portkey / Helicone 的最主要差别**不**是？
- A. 是否提供托管 SaaS
- B. 是否原生支持流式 + tool use
- C. 是否提供监控 / Tracing
- D. 是否是 BSD 协议

---

## A12 — 流式输出与 SSE

**12.1 ⭐⭐** SSE 的 MIME type 是？
- A. `application/stream+json`
- B. `text/event-stream`
- C. `application/x-sse`
- D. `text/plain`

**12.2 ⭐⭐** TTFT（Time To First Token）是衡量什么的关键指标？
- A. 总耗时
- B. 模型生成第一个 token 的延迟（决定用户感知响应速度）
- C. token 速率
- D. 输入长度

**12.3 ⭐⭐⭐** Go 实现 SSE server 时，下列**最重要**的操作是？
- A. 使用 `http.Hijacker`
- B. 每写完一条 event 后调用 `Flusher.Flush()` 立即把缓冲推给客户端
- C. 设 `Content-Length`
- D. 关闭 Keep-Alive

**12.4 ⭐⭐⭐** SSE 客户端断线重连时，下列**正确**的是？
- A. 必须从头开始
- B. 浏览器 EventSource 会自动重连并带 `Last-Event-ID` 头，server 应据此恢复进度
- C. 重连后无法获取前面数据
- D. 重连必须由后端主动发起

**12.5 ⭐⭐⭐⭐** Nginx 作为反向代理时，做 SSE 透传**关键配置**是？
- A. 开启 gzip
- B. 关闭 `proxy_buffering` 并适当延长 `proxy_read_timeout`
- C. 启用 `proxy_cache`
- D. 增大 `client_max_body_size`

---

## A13 — LLM 可观测性

**13.1 ⭐⭐** OpenTelemetry GenAI semantic convention 中描述模型的属性前缀是？
- A. `llm.*`
- B. `model.*`
- C. `gen_ai.*`
- D. `ai.*`

**13.2 ⭐⭐** Langfuse 的核心数据模型**不**包括？
- A. Trace
- B. Observation（generation / span / event）
- C. Score
- D. Container

**13.3 ⭐⭐⭐** 衡量"答案质量"最适合的自动化方式是？
- A. 字符长度
- B. LLM-as-judge + 黄金集对照分
- C. 响应延迟
- D. token 数

**13.4 ⭐⭐⭐** 监控 LLM 系统的"红线指标"**不**应包含？
- A. p99 latency
- B. cost / day
- C. 错误率
- D. 模型权重 SHA

**13.5 ⭐⭐⭐⭐** 把日志保留期设置过长**最大的合规风险**是？
- A. 磁盘成本
- B. PII / 敏感数据长期暴露 + GDPR / 个保法等违规
- C. 查询变慢
- D. 影响 SLA

---

## A14 — LLM 安全

**14.1 ⭐⭐⭐** Indirect prompt injection 的典型攻击点是？
- A. 用户直接输入
- B. RAG 检索到的网页 / 文档里被植入的"忽略上述指令"等
- C. system prompt
- D. tool 名字

**14.2 ⭐⭐⭐** OWASP LLM Top 10 (2025) 中排第一类的是？
- A. Insecure Output Handling
- B. Prompt Injection
- C. Supply Chain Vulnerabilities
- D. Sensitive Information Disclosure

**14.3 ⭐⭐⭐⭐** 防 system prompt 泄露最有效的方式**不是**？
- A. 假定 system prompt 一定会泄露，不放敏感信息
- B. 给 system prompt 加 "永远不要复述本段"
- C. 输出校验拦截 system prompt 关键词
- D. 用 server 侧 RAG + 短期 system prompt

**14.4 ⭐⭐⭐⭐** PII redaction 应该在哪一层做？
- A. 仅在生成前
- B. 输入与输出两端都做（双向防护），并落审计日志
- C. 仅在生成后
- D. 只在数据库中做

**14.5 ⭐⭐⭐⭐⭐** 红队工具 Garak 主要测试的是？
- A. 模型推理速度
- B. 模型对各种攻击 prompt（jailbreak、leak、bias）的鲁棒性
- C. 模型上下文长度
- D. tokenizer 兼容性

---

## A15 — LLM Evaluation

**15.1 ⭐⭐⭐** 关于 RAG triad（Ragas 评测三角），下列哪一项**不属于**？
- A. Context Relevance
- B. Faithfulness
- C. Answer Relevance
- D. Token Efficiency

**15.2 ⭐⭐⭐** LLM-as-Judge 的位置偏置（position bias）最有效的缓解方式是？
- A. 在 prompt 里强调"忽略顺序"
- B. 评测时把 A/B 答案位置互换两次，取平均
- C. 换更大的 judge 模型
- D. 用规则代替 judge

**15.3 ⭐⭐⭐⭐** 关于离线 eval（CI 门禁）与在线 eval（生产监控）的关系，**错误**的是？
- A. 二者目的不同，都需要
- B. 在线 eval 可以做 100% 流量打分
- C. 在线 eval 应当走异步 worker，不阻塞用户请求
- D. 离线 eval 用固定 golden set，在线 eval 用真实采样

**15.4 ⭐⭐⭐⭐** Ragas 的 faithfulness 指标基于什么计算？
- A. answer 与 ground truth 的 BLEU
- B. answer 拆成 claims，每个 claim 由 judge 判定是否被 context 支持
- C. answer 长度 / context 长度
- D. answer 中关键词命中 context 的比例

**15.5 ⭐⭐⭐⭐⭐** 多个 judge 评同一样本，最稳定的聚合方式通常是？
- A. 取最大值
- B. 取最小值
- C. 多 judge ensemble 多数投票或平均
- D. 取第一个 judge 的结果

---

## A16 — LLM 成本与延迟优化

**16.1 ⭐⭐⭐** Anthropic Prompt Cache 的缓存命中规则是？
- A. 按 message 整体哈希
- B. 按前缀（prefix）精确匹配，从头开始
- C. 按 user 标识隔离
- D. 按模型版本独立缓存

**16.2 ⭐⭐⭐** 关于 Batch API 50% 折扣**正确**的是？
- A. 适合实时聊天
- B. 适合离线大批量任务，承诺 24h 内完成
- C. 也支持 streaming
- D. 失败一条整批失败

**16.3 ⭐⭐⭐⭐** Agent 多步任务总成本爆炸的最大根因是？
- A. tool 调用本身贵
- B. history 每步累积，重复发整段历史
- C. 模型选错
- D. 工具响应慢

**16.4 ⭐⭐⭐⭐** 关于 LLM 调用的 TTFT（Time To First Token），**正确**的是？
- A. 输入越长 TTFT 越短
- B. TTFT 主要受 prefill 阶段影响，输入越长越久
- C. streaming 不影响 TTFT
- D. cache 命中后 TTFT 增加

**16.5 ⭐⭐⭐⭐⭐** 优化 LLM 应用成本，下列**最不建议**作为第一步的是？
- A. 把可缓存的 system prompt 加上 cache_control
- B. 分级路由：简单查询走 Haiku
- C. 直接把所有调用换到最便宜的模型
- D. 离线 + 在线评测对照防止质量退化

---

## A17 — Agent Harness 工程

**17.1 ⭐⭐⭐** 关于 Tool Runner 和 Claude Agent SDK，**正确**的是？
- A. 两者是同一个包的不同入口
- B. Tool Runner 是普通 SDK 里的循环封装（无内置工具），Agent SDK 是 Claude Code 打包成库（自带工具）；两者都是"仅 harness、自托管"
- C. Tool Runner 提供托管沙箱，Agent SDK 不提供
- D. 两者都提供托管部署

**17.2 ⭐⭐⭐⭐** harness 开了服务端 compaction，但 token 消耗不降反升。最可能的原因是？
- A. compaction 阈值设太低
- B. 每轮只把响应里的 text 抽出来拼回 messages，丢掉了 compaction 块
- C. 模型不支持 compaction
- D. 没有开 prompt caching

**17.3 ⭐⭐⭐⭐** 把一个动作从 bash 升格成专用 tool，下列**不属于**升格理由的是？
- A. 需要人工审批门控
- B. 需要"文件自上次读取后已变更就拒绝写入"这类陈旧检查
- C. 需要标记为并行安全以便并发调度
- D. 能让模型少消耗 thinking token

**17.4 ⭐⭐⭐⭐** agent 一轮发起 25 个并行 `read_file`，system prompt 固定、模型没换、tool 集没变，但 prompt cache 命中率长期为 0。最可能的原因是？
- A. 并行请求太多触发限流
- B. 单轮 content block 数超过了 cache 断点 20 块的回溯窗口
- C. tool_result 不参与缓存
- D. 缓存 TTL 过期

**17.5 ⭐⭐⭐⭐⭐** 关于 `max_tokens` 与 `task_budget`，**正确**的是？
- A. 两者等价，只是 beta 与 GA 的区别
- B. `max_tokens` 是模型看不到的硬上限（到点截断）；`task_budget` 是模型能看到倒计时的软预算（到点自己收尾）
- C. `task_budget` 会统计你每次重发的完整历史
- D. `task_budget` 没有下限，可以设成任意值

---

# 参考答案

## A01 Claude API 工程化
1.1 **C**　1.2 **D**（`tool_result` 的 content 可以是字符串或多模态块，不强制 JSON）　1.3 **B**（system 改变 = cache key 改变）　1.4 **B**　1.5 **B**

## A02 OpenAI 兼容生态
2.1 **B**　2.2 **B**　2.3 **B**　2.4 **B**（Chat Completions 也支持多轮）　2.5 **B**

## A03 Tokenizer 与计费
3.1 **C**　3.2 **B**　3.3 **B**　3.4 **C**　3.5 **C**（替换模型只是手段之一，不是"最重要"的成本控制；A、B、D 都是同时常用的更普适手段）

## A04 Prompt 工程
4.1 **B**　4.2 **B**　4.3 **C**　4.4 **B**　4.5 **B**

## A05 长对话与上下文管理
5.1 **A**　5.2 **B**　5.3 **B**　5.4 **C**（不会自动跨用户共享，跨用户必须显式授权）　5.5 **D**（分离反而可能让 system 更长；好处主要是稳定性和一致性）

## A06 Embedding 与向量库
6.1 **B**　6.2 **B**　6.3 **B**　6.4 **B**　6.5 **B**（Pinecone 支持向量过滤）

## A07 RAG 架构
7.1 **B**　7.2 **A**　7.3 **B**　7.4 **D**（faithfulness 看的是是否被 context 支持，不是 ground truth 字面一致）　7.5 **C**（temperature=0 也会幻觉）

## A08 Tool Use / Function Calling
8.1 **C**（Claude 把 tool_result 放在 user 角色）　8.2 **B**　8.3 **B**　8.4 **B**　8.5 **B**

## A09 Agent 系统设计
9.1 **B**　9.2 **B**　9.3 **C**（agentic loop 允许并行）　9.4 **C**　9.5 **B**

## A10 MCP
10.1 **B**　10.2 **B**　10.3 **D**（MCP 原语分两组——服务端：Tools / Resources / Prompts；客户端：Sampling / Roots / Elicitation；Pipelines 不属于任何一类）　10.4 **B**　10.5 **B**

## A11 LLM Gateway
11.1 **D**（SQL 转写不属于 LLM Gateway 的核心）　11.2 **B**　11.3 **B**　11.4 **B**　11.5 **D**（协议不是主要差别）

## A12 流式输出与 SSE
12.1 **B**　12.2 **B**　12.3 **B**　12.4 **B**　12.5 **B**

## A13 LLM 可观测性
13.1 **C**　13.2 **D**（Langfuse 没有"Container"模型）　13.3 **B**　13.4 **D**　13.5 **B**

## A14 LLM 安全
14.1 **B**　14.2 **B**　14.3 **B**（system prompt 加自然语言"不要复述"很容易被绕过）　14.4 **B**　14.5 **B**

## A15 LLM Evaluation
15.1 **D**（Token Efficiency 不属于 RAG triad）　15.2 **B**　15.3 **B**（在线评测应当采样而非 100%，judge 成本高）　15.4 **B**　15.5 **C**

## A16 LLM 成本与延迟优化
16.1 **B**　16.2 **B**　16.3 **B**　16.4 **B**　16.5 **C**（盲目换便宜模型可能损失质量，应先评测 + 缓存 + 分级路由）

## A17 Agent Harness 工程
17.1 **B**　17.2 **B**（compaction 块丢了，压缩状态消失，每轮从完整历史重发）　17.3 **D**（升格是为了让 harness 能门控/校验/渲染/调度，跟模型思考量无关）　17.4 **B**（25 对 tool_use/tool_result = 50 块，远超 20 块回溯窗口）　17.5 **B**

---

> 评分参考（85 题满分）：
> - 74+：优秀，可以面试 LLM 平台 / AI 后端中高级岗位
> - 58-74：基本掌握
> - 42-58：建议针对错题章节重读
> - < 42：先做 A01 / A04 / A07 / A15 / A16 五篇打底，再来挑战
