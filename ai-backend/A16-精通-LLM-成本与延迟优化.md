# 精通 LLM 成本与延迟优化：Prompt Cache、批处理、推测解码与分级路由

> 课程编号：A16
> 路线图来源：AI / LLM 后端工程 · 模块五 生产化
> 难度：⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 5 月

---

## 引言：成本与延迟是 LLM 应用的两条命

经常听到这样的话：

- "我们的产品功能挺好，但每个用户日均花我们 $0.50 在 LLM 上，根本扛不住增长"
- "首字（TTFT）2 秒、整体 8 秒，用户都流失了，可换便宜模型质量又不够"
- "本来一个用户问完一次问题就完了，现在 Agent 转一圈调 30 次 LLM，账单飞了"

LLM 应用商业化的两座大山：

1. **单位经济性**（Unit Economics）：每个用户 / 每次请求多少钱
2. **延迟可接受性**：用户能不能等

它们经常**对立**：

- 用更强模型→质量好但又贵又慢
- 用更长 context→更准确但又贵又慢
- 多步 reasoning→更靠谱但又贵又慢

本章从 6 个维度系统拆解：**输入成本** / **输出成本** / **请求次数** / **延迟分解** / **吞吐量** / **SLA**。每节都给出"做了之后能省多少 / 快多少"的量化经验。

---

## 第一章：成本结构——钱花在哪

### 1.1 LLM API 的计费维度

```
费用 = 输入 token × 输入单价 + 输出 token × 输出单价
        + cache write × 缓存写单价
        - cache read × 缓存读折扣
```

以 Claude Sonnet 4.6 为例（2026/05 估值）：

| 类目 | 价格（per MTok） |
|---|---|
| Input | $3 |
| Output | $15 |
| Cache write (5min) | $3.75 |
| Cache read | $0.30 |
| Cache write (1h, GA) | $6 |

**关键观察**：

- **输出比输入贵 5×**——所以 verbose 输出最烧钱
- **缓存读只要 1/10**——重复 prefix 几乎免费
- **缓存写要多 25%**——首次写入有溢价，但 5min 内多读就赚回来了

OpenAI / Gemini 类似定价结构，但 cache 折扣比例略有差异。

### 1.2 成本画像三类应用

```
类型 A：对话型（Chat / 客服）
  - 输入：长 (system + history)
  - 输出：中等
  - 频率：中
  - 关键：prompt cache + 上下文摘要

类型 B：批处理型（数据提取 / 摘要）
  - 输入：中等
  - 输出：中等到短
  - 频率：高
  - 关键：batch API + 缓存 system prompt

类型 C：Agent 型（多步 reasoning + 工具）
  - 输入：累积爆炸（每步都把 history 重发）
  - 输出：中等
  - 频率：单任务多 call
  - 关键：cache + history 裁剪 + 任务粒度路由
```

### 1.3 成本审计：先量化再优化

不要凭感觉优化。每个 endpoint 打点：

```python
# 任何 LLM call 都打这一组指标
log_event("llm_call", {
    "model": resp.model,
    "input_tokens": resp.usage.input_tokens,
    "output_tokens": resp.usage.output_tokens,
    "cache_creation_input_tokens": resp.usage.cache_creation_input_tokens,
    "cache_read_input_tokens": resp.usage.cache_read_input_tokens,
    "latency_ms": elapsed,
    "ttft_ms": time_to_first_token,
    "endpoint": "/chat/v1",
    "user_id": uid,
    "trace_id": tid,
})
```

把它喂到 ClickHouse / BigQuery / Datadog，能看到：

```
endpoint        avg_input  avg_output  cost/call  P95 latency
─────────────────────────────────────────────────────────
/chat/v1        5800       340         $0.024     3.4s
/agent/plan     12000      890         $0.062     8.1s
/extract/v2     800        120         $0.004     1.2s
```

立刻看出谁是"金主"——优先优化它。

---

## 第二章：Prompt Cache——降本第一发动机

### 2.1 原理

LLM 推理时要把 prompt 喂进模型，从注意力机制生成 KV cache（每个 token 的 K / V 矩阵）。**KV cache 的计算是输入成本的主要来源**——计算完之后立即丢弃实在浪费。

Prompt cache 的本质：**把 KV cache 存住，下次同 prefix 直接复用**。

```
请求 1: [system... 5000 tokens] [user: 你好]
        ↓ 计算全部 5001 tokens 的 KV cache
        ↓ 写入缓存（5min TTL）

请求 2: [system... 5000 tokens] [user: 今天天气]
        ↓ 命中前 5000 tokens 缓存（read）
        ↓ 只算后面新增的 user message
        ↓ 输入 token 计费 1/10
```

### 2.2 Anthropic Prompt Cache 用法

```python
import anthropic
client = anthropic.Anthropic()

SYSTEM = """你是企业知识库助手。以下是规章制度（5000 tokens 长）：
...."""

resp = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=500,
    system=[{
        "type": "text",
        "text": SYSTEM,
        "cache_control": {"type": "ephemeral"},  # 标记缓存
    }],
    messages=[{"role": "user", "content": "请假流程是什么？"}],
)
```

第二次同 system prompt 调用，`cache_read_input_tokens` 字段会显示命中：

```python
resp.usage
# {
#   "input_tokens": 12,                  # 新 user message
#   "cache_read_input_tokens": 5000,     # 命中缓存
#   "cache_creation_input_tokens": 0,
#   "output_tokens": 80
# }
```

### 2.3 缓存什么放在哪

**Anthropic 缓存规则**（2026/05）：

- 缓存最多 4 个 breakpoint（`cache_control` 位置）
- 缓存按 prefix 命中——前缀完全相同才命中
- 缓存最小 1024 tokens（Sonnet），不够不生效
- TTL 默认 5min，GA 支持 1h cache（cache write 为 base 的 2x，5min 为 1.25x）

**正确摆放顺序**：

```
[system instructions  ←─ cache_control (静)
 tool definitions     ←─ cache_control (静)
 RAG context          ←─ cache_control (半静)
 conversation history (动)
 user message        ]
```

把静态的放前面、动态的放后面，这样静态部分总能命中。

### 2.4 实际省多少

举一个企业客服 case：

| 场景 | 不开缓存 | 开缓存 |
|---|---|---|
| System prompt | 5000 tokens × $3 = $0.015 | 5000 × $0.30 = $0.0015 |
| User + history | 500 tokens × $3 = $0.0015 | 500 × $3 = $0.0015 |
| 单次成本 | $0.0165 | $0.003 |
| **省** | — | **82%** |

如果 system prompt 10K + 频繁短查询，开缓存能省 **85-90%** 输入成本。

### 2.5 缓存陷阱

- **prompt 变一个字节就失效**——任何 dynamic 内容（用户名 / 时间）放后面
- **TTL 5min**：低频接口缓存命中率很低——比如每 10min 才一次调用，缓存被 evict
- **冷启动**：第一个请求要写缓存（贵 25%）+ 后续才便宜，scale up 时容易踩
- **多 user 隔离**：如果 system 里嵌了用户名，缓存就分流了——尽量保 system 全局
- **provider 区别**：OpenAI 自动缓存（无需 hint），GPT-5 及更新模型折扣约 90%（cached input 付 10%）；Anthropic 需显式 cache_control，命中折扣约 90%

---

## 第三章：Batch API——离线场景降本利器

### 3.1 何时用 Batch

- 离线批量任务（embedding 一批文档、抽取一批数据、给一批 review 评分）
- 不需要实时返回（24 小时内出结果都行）
- 量大（万级以上）

**Anthropic Batch API** / **OpenAI Batch API** 价格：**50% off**。

### 3.2 使用例

```python
import anthropic
client = anthropic.Anthropic()

# 构建批量
requests = [
    {
        "custom_id": f"task-{i}",
        "params": {
            "model": "claude-sonnet-4-6",
            "max_tokens": 200,
            "messages": [{"role": "user", "content": f"摘要：{doc}"}],
        },
    }
    for i, doc in enumerate(documents)
]

# 提交
batch = client.messages.batches.create(requests=requests)

# 轮询
while True:
    batch = client.messages.batches.retrieve(batch.id)
    if batch.processing_status == "ended": break
    time.sleep(60)

# 拉结果
for result in client.messages.batches.results(batch.id):
    print(result.custom_id, result.result)
```

### 3.3 注意

- **完成时间**：通常几分钟到几小时，承诺 24h 内
- **配额**：batch 用单独的 quota，不挤占实时
- **不能流式**：只能等完出结果
- **错误重试**：batch 里单条失败不会影响其它，根据 result 状态自行重试

---

## 第四章：模型路由分级——便宜的能做就让便宜的做

### 4.1 分级思想

不是所有请求都要最强模型。三档常见路由：

```
请求
  │
  ▼
意图分类器 (Haiku / 规则)
  │
  ├──► 简单 (闲聊 / FAQ)        → Haiku 4.5  ($5/M out)
  ├──► 中等 (摘要 / 通用 QA)     → Sonnet 4.6 ($15/M out)
  └──► 复杂 (推理 / 代码 / 长链路) → Opus 4.7  ($25/M out)
```

**省钱效应**：如果 60% 走 Haiku、30% Sonnet、10% Opus，平均成本只有"全用 Sonnet"的 30%。

### 4.2 路由实现

**规则 + 关键词**（快、便宜、可解释）：

```python
def route(query: str, history: list) -> str:
    if len(history) > 10 or "推理" in query or "对比" in query:
        return "claude-opus-4-7"
    if len(query) < 50 and is_smalltalk(query):
        return "claude-haiku-4-5"
    return "claude-sonnet-4-6"
```

**LLM 分类**（精准但慢）：

```python
async def llm_route(query):
    cls = await haiku.classify(query, labels=["simple","medium","complex"])
    return MODEL_MAP[cls]
```

为了避免双倍延迟，分类器只用 Haiku（200-500ms），后续问答的延迟才占主导。

**Confidence-based fallback**：

```python
# 先用 Haiku 试，模型 self-confidence 低就升级
resp = haiku.answer(query)
if resp.confidence < 0.6:
    resp = sonnet.answer(query)
```

### 4.3 路由的陷阱

- **路由器本身不能错**——错把复杂任务交给 Haiku，质量直接崩
- **持续监控分流比例**：意图分类器跑偏会导致全跑 Opus
- **质量 + 成本同步评测**：分级路由后跑 A15 章的离线评测，确认没降太多

---

## 第五章：延迟分解——TTFT vs 总时长

### 5.1 LLM 调用的延迟构成

```
client ─── network ───► gateway ─── network ───► LLM provider
                                                       │
                                                       ▼
                                         prefill (处理 input 全部 token)
                                                       │
                                                       ▼
                                          first token (TTFT)
                                                       │
                                                       ▼
                                          decode (逐 token 生成)
                                                       │
                                                       ▼
                                          last token (total)
```

关键指标：

- **TTFT (Time To First Token)**：首字时间，主要受 prefill 影响
- **ITL (Inter-Token Latency)**：每个 token 间隔，约 20-50ms
- **Total Latency** = TTFT + (output_tokens × ITL)

**经验值**（Sonnet 4.6, 2026/05）：

| 输入 token | TTFT |
|---|---|
| 500 | ~400ms |
| 5000 | ~1.2s |
| 20000 | ~3.5s |
| 100000 | ~10s |

输入越长，TTFT 越长——prefill 是 O(N²) 注意力。

### 5.2 优化 TTFT

1. **prompt cache**：缓存命中后 prefill 走快路径——TTFT 减半甚至更多
2. **压缩 system prompt**：5000 → 2000 tokens 可减 60% TTFT
3. **更换近 region 端点**：跨地域 100ms 起步
4. **streaming**：用户感知"开始有内容"——即便总时长不变，体感快很多

### 5.3 优化总延迟

1. **降低输出长度**：让模型简明回答（"用 3 句话总结"）
2. **并行调用**：多个独立 LLM call 并发，不要串行
3. **预生成**：缓存常见 query 的最终回答
4. **speculative decoding**：见下一节

### 5.4 流式输出（streaming）

**用户体感公式**：

```
体感速度 ≈ TTFT 而非 total latency
```

非流式：用户等 6 秒看到完整答案 → 焦虑
流式：用户 800ms 后看到首字、逐步刷出 → 感觉"快"

技术细节见 [A12 流式输出与 SSE](./A12-精通-流式输出与-SSE.md)，本章只强调：**有 UI 的应用必须开 streaming**。

---

## 第六章：Speculative Decoding——同价格更快

### 6.1 原理

普通 decode 是一个 token 一个 token 算的，慢。

Speculative decoding（推测解码）：

```
1. 用一个小模型（draft）快速猜 N 个 token
2. 用大模型（target）一次性验证 N 个 token
3. 接受前 K 个被验证通过的，丢弃后面
```

效果：在保持 target 模型输出分布**完全相同**的前提下，速度 1.5-3×。

### 6.2 商业平台支持

- **vLLM**：v0.6+ 内置 speculative decoding
- **TGI (Hugging Face)**：支持 medusa / EAGLE / Lookahead
- **Anthropic / OpenAI 闭源**：用户不可见但服务端可能已用
- **Groq / SambaNova / Cerebras**：定制硬件 + 推测解码达 1000+ token/s

应用层只能感知到"更快了"，但是部署自有模型时这是关键优化。

### 6.3 适用边界

- 输出 token 越多收益越大（attention 占比降低）
- batch 较小时收益更明显
- 输出可预测（代码 / 模板）效果好；高熵随机输出收益小

---

## 第七章：Agent 场景的成本陷阱

Agent 是最容易"烧钱"的形态。

### 7.1 history 累积

```
Step 1: [system] + [user]
Step 2: [system] + [user] + [tool_call_1] + [tool_result_1]
Step 3: [system] + [user] + [tool_call_1] + [tool_result_1] +
        [tool_call_2] + [tool_result_2]
...
Step N: 累积 N 倍 history
```

每步都把全部 history 发给 LLM——10 步任务的总 token 是单步的 30-50 倍。

### 7.2 优化策略

**A. Prompt cache 是 Agent 的救命药**

每步前缀（system + tool defs + 前 N-1 步历史）尽量命中缓存。Anthropic 的 1h cache（GA）非常适合 Agent 长任务。

**B. Tool result 压缩 / 摘要**

```
[tool result: 5000 token 的 search 结果]
   ↓ summary 提取关键信息
[tool result summary: 500 tokens]
```

Tool 结果是膨胀大头，建议在 tool wrapper 里就压缩。

**C. 子任务上下文裁剪**

完成的子任务的中间 step 可以剔除，只保留 "子任务 X 已完成，结果：..."。

**D. 短任务用小模型**

Agent 内部的"是否需要更多工具调用"这种判断，用 Haiku 跑就行，不要每步都 Opus。

### 7.3 成本上限保护

Agent 容易死循环——加硬性上限：

```python
class AgentBudget:
    def __init__(self, max_steps=20, max_cost_usd=0.50):
        self.steps = 0
        self.cost = 0.0
        self.max_steps = max_steps
        self.max_cost_usd = max_cost_usd

    def check(self):
        if self.steps >= self.max_steps:
            raise BudgetExceeded("max_steps")
        if self.cost >= self.max_cost_usd:
            raise BudgetExceeded("max_cost")

    def add(self, resp):
        self.steps += 1
        self.cost += compute_cost(resp.usage, resp.model)
        self.check()
```

线上 Agent 必须有此类拦截，否则一个 bug 一夜就能烧几千刀。

---

## 第八章：吞吐量——批并发的极限

### 8.1 一个用户、一个 worker 的简单模型不够用

```
单线程 LLM call → 单线程吞吐 = 1 / latency
                = 1 / 3s = 0.33 QPS

要 100 QPS → 需要 300 并发
```

LLM API 本来就是异步等待为主——用 asyncio / Go goroutine 高并发不难。

### 8.2 RateLimit 与 RPM/TPM

提供商按 **RPM**（requests/min）+ **TPM**（tokens/min）限速。

- 突发流量超 RPM → 429
- 慢慢累积超 TPM → 429
- 不同 tier 配额不同（pay-as-you-go vs enterprise）

**做法**：

```python
from asyncio import Semaphore
from aiolimiter import AsyncLimiter

sem = Semaphore(50)              # 并发上限
rpm = AsyncLimiter(1000, 60)     # 每分钟 1000 个请求
tpm = AsyncLimiter(800_000, 60)  # 每分钟 80 万 tokens

async def call_llm(prompt):
    async with sem, rpm:
        await tpm.acquire(estimate_tokens(prompt))
        return await client.messages.create(...)
```

### 8.3 多 provider fallback

```python
PROVIDERS = [
    ("anthropic", "claude-sonnet-4-6"),
    ("bedrock", "anthropic.claude-sonnet-4-6"),
    ("vertex", "claude-sonnet-4-6@anthropic"),
]

async def call_with_fallback(messages):
    for provider, model in PROVIDERS:
        try:
            return await call(provider, model, messages)
        except (RateLimitError, ServerError):
            continue
    raise AllProvidersDown()
```

不只是省钱——单 provider 出故障时业务能扛住。

---

## 第九章：SLA 设计

### 9.1 SLO 定义

```
- 可用性 (Availability)    ≥ 99.5%
- TTFT P95                 < 1200ms
- 端到端 P95               < 5000ms
- 错误率                   < 0.5%
```

LLM 上游本身可用性 99.5-99.9%（公开 status page），所以业务侧难以做到 99.99%——多 provider fallback 是必备。

### 9.2 Error budget

```
月度 SLO 99.5% → 每月可承受错误时间 = 30 * 24 * 60 * 0.5% = 216 分钟

超过预算 → 暂停新功能上线，全力修稳定性
```

### 9.3 降级与熔断

```python
@circuit_breaker(failures=10, window=60s, half_open_after=30s)
async def llm_call(...): ...

# 熔断打开时降级
def degraded_response(query):
    return "抱歉，AI 服务暂时不可用，请稍后再试。"
    # 或更高级：返回上次缓存 / 走纯检索答案 / 转人工
```

### 9.4 缓存层是 SLA 救生圈

- 即使 LLM 全挂，热门 query 可以从语义缓存返回
- 高峰期把"近 1h 重复 query"全部命中缓存（85% 重复率不夸张）
- 见 [A05 长对话与上下文](./A05-精通-长对话与上下文管理.md) 的语义缓存章节

---

## 第十章：实战：把一个 RAG 应用成本降 70%

下面是个真实优化清单——一个客服 RAG 系统，初始月成本 $40k，优化后 $12k。

### 现状

```
QPS 平均 5（高峰 20）
每次 query：
  - system prompt 1500 tokens (静)
  - retrieved context 4000 tokens (半静，30% query 命中相同 chunk)
  - user message 80 tokens
  - history 1000 tokens
  - output 300 tokens
模型：Sonnet 4.6
单次成本：~$0.025
月成本：5 QPS × 86400 × 30 / 1000 × $0.025 ≈ $325/day × 30 ≈ $9750
```

嗯，写小了。换成 50 QPS：$97k。先按 50 QPS / $40k 调整（开了部分 cache）。

### 优化清单

| 优化 | 预期收益 | 实测 |
|---|---|---|
| 1. system prompt 开 Anthropic prompt cache | -25% | -22% |
| 2. RAG context 也用 cache（按 chunk hash 选 breakpoint） | -15% | -18% |
| 3. history 摘要：超过 5 轮压缩成 200 token | -10% | -8% |
| 4. 闲聊 / FAQ 走 Haiku（命中 25% query） | -15% | -12% |
| 5. 答案长度限制（"用 3 句话回答"）| -8% | -7% |
| 6. 缓存命中前 1h 重复 query（语义缓存）| -10% | -5% |
| 7. 离线评测确认质量没掉 | 不省钱但必须做 | ✓ |
| 8. 监控 + alarm | 不省钱但必须做 | ✓ |

最终：

```
原 $40k → $12k，省 70%。
质量评测：helpfulness 4.21 → 4.18（无显著下降）
延迟 P95：4.2s → 2.6s（cache 带飞）
```

---

## 第十一章：陷阱清单

1. **过早优化**：还没产品 PMF 就拼命压成本，反而拖慢迭代。先量化、定 baseline、再优化。
2. **追便宜模型而丢质量**：换 Haiku 省 80% 成本但 helpfulness 跌 0.5 → 用户流失成本远超 LLM。
3. **缓存粒度太细**：每个 user 一份 cache，几乎没命中——尽量保 system 全局共享。
4. **缓存粒度太粗**：把动态数据塞 cache breakpoint 之前——永远 miss。
5. **不监控 cache_hit_rate**：以为开了 cache 就万事大吉，结果 hit rate 5%。
6. **batch API 用错场景**：实时接口套了 batch，用户等 24h——只用于真离线任务。
7. **路由器质量没保障**：分类器跑偏 → 简单问题走 Opus 烧钱、复杂问题走 Haiku 答错。
8. **Agent 没预算上限**：bug 让 Agent 死循环 5000 步，一夜烧 $3000。
9. **TTFT 不监控**：用户体感差但仪表盘只看 total，发现不了。
10. **没多 provider fallback**：上游 1h 故障 = 业务 1h 挂——SLA 直接破。
11. **没流式**：3s 等待 vs 500ms 首字，转化率差 30%+。
12. **没 prompt 版本号**：A/B 想回滚发现 prompt 谁也找不全——见 A04 Prompt 工程。

---

## 第十二章：2026 现状

- **Anthropic / OpenAI / Gemini 都有 prompt cache**——折扣比例 50-90% 不等
- **Anthropic 1h cache GA**——专为 Agent 长任务设计
- **OpenAI Batch API 50% off**——离线任务事实标准
- **Speculative decoding** 成主流：Groq / SambaNova 硬件加持下 1000+ tok/s
- **MoE 模型路由**：DeepSeek / Mixtral / Llama-MoE 自带分级，单模型就能"软路由"
- **Edge 推理崛起**：Llama 3.x / Phi-4 / Qwen3 在终端跑，敏感数据无需上云
- **Cache 提供商**：Helicone / Portkey / LiteLLM 都内置 LLM Gateway 级缓存
- **Token 优化器**：LLMLingua（微软）压缩 prompt 2-20×，质量损失小
- **Cost-aware orchestration**：LangGraph / DSPy 把成本作为 trace 一等属性，自动建议路由策略

---

## 第十三章：练习题

1. ⭐ 解释 prompt cache 的 prefix 匹配机制。为什么动态内容要放在 prompt 末尾？
2. ⭐ Batch API 的 50% 折扣适合什么场景？为什么不适合实时聊天？
3. ⭐⭐ 写一段代码：估算一次 LLM call 的成本（含 cache read / write 分别计算）。
4. ⭐⭐ 设计三级模型路由（Haiku / Sonnet / Opus），描述路由决策逻辑与监控指标。
5. ⭐⭐ Agent 系统平均每任务 8 步、history 累积爆炸，给出至少 4 条优化方案。
6. ⭐⭐⭐ 一个 RAG 系统月成本 $20k，给出从大到小的 6 步优化方案，每步预估收益。
7. ⭐⭐⭐ 设计一个"成本预算守护"机制：单次 Agent 任务超过 $0.50 强制终止，且把信号反馈给上游。
8. ⭐⭐⭐ 当 LLM 提供商 1 小时大故障时，给出降级策略（缓存 / fallback / 部分功能关闭）。

<details>
<summary>📝 参考思路</summary>

1. cache 按 prefix 精确匹配，任何位置改动会让后面所有内容失效——所以动态内容（用户名 / 时间）放最末，保 prefix 稳定。
2. 适合离线大量任务（embedding / 数据抽取 / 评分），不适合需要秒级返回的聊天，因为 batch 完成时间不可控。
3. ```python
   def cost(usage, model):
       p = PRICES[model]
       return (usage.input_tokens * p.input
             + usage.output_tokens * p.output
             + usage.cache_creation * p.cache_write
             + usage.cache_read * p.cache_read) / 1_000_000
   ```
4. 简单：闲聊 / FAQ / 长度 < 50 → Haiku；中等：通用 QA / 摘要 → Sonnet；复杂：推理 / 长 chain / 代码 → Opus。监控：每模型流量占比、每模型质量分（helpfulness）、单次平均成本。
5. ① system / tool def 用 cache；② tool result 摘要压缩；③ 完成子任务后裁剪中间步骤；④ 内部 routing 决策用 Haiku，仅最终 reasoning 用 Sonnet/Opus；⑤ 1h cache 替代 5min（长任务）。
6. 见第十章实战。
7. AgentBudget 上下文，每步累加成本，超过阈值抛 `BudgetExceeded`；外层捕获后给用户友好提示 + 写 trace + 告警。
8. ① 语义缓存命中近 1h 热问；② 切换 fallback provider（Bedrock / Vertex）；③ 关闭非核心 Agent 流；④ 增大缓存 TTL；⑤ 给用户"AI 服务降级中"提示，保留人工通道。

</details>

---

## 小结

LLM 应用商业化的两条命：**成本** 与 **延迟**。本章给的优化栈：

```
量化（不量化别优化）
  │
  ├─ Prompt Cache       ← 杠杆最大，省 70-90% 输入
  ├─ 模型路由           ← 简单问题不要用旗舰
  ├─ Batch API          ← 离线任务直接 50% off
  ├─ History 压缩       ← Agent 救星
  ├─ 流式 / SLA / 熔断  ← 体验与可用性
  └─ 多 provider        ← 抗故障
```

至此 ai-backend 16 章完整闭环：

- A01-A05：与模型对话的工程基础
- A06-A09：检索 / 工具 / Agent 高阶能力
- A10-A12：MCP / Gateway / 流式
- A13-A14：可观测 / 安全
- **A15-A16：评测 / 成本——把 LLM 应用从"能跑"做到"可商业化"**

下一步：把这套搬到生产，建议先做 A13 + A15 + A16 三件事——可观测、有评测、可控成本，这就是合格的 AI 后端工程。
