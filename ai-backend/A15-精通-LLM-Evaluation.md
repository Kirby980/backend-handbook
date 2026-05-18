# 精通 LLM Evaluation：从 Ragas 到 LLM-as-Judge 的工程化评测

> 课程编号：A15
> 路线图来源：AI / LLM 后端工程 · 模块五 生产化
> 难度：⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 5 月

---

## 引言：不评测的 LLM 应用就是裸奔

回想十年前后端工程师的两件法宝：**单元测试** + **APM**。前者保证 commit 没把功能写坏，后者监控线上是否健康。

LLM 应用很多时候两样都没有：

- 没有"绿色 / 红色"的单元测试——LLM 输出是自然语言，没办法 `assert.Equal`
- 没有传统 P99 监控——延迟正常不代表答案对，答案对不代表用户满意

于是常见的剧情：

- Prompt 改了一行 "请用中文回答"，线上 80% 的 query 答非所问，迭代两周后才发现
- RAG 系统 "看起来还行"，但 30% 的 query 引用了错误段落，没指标
- 换了个便宜模型，节省了 60% 成本，**却悄悄丢失了 15% 的回答质量**
- 用户投诉"模型变笨了"，工程师说"没改代码啊"——其实供应商把模型悄悄换了

**Evaluation（评测）就是 LLM 应用的"自动化测试 + APM"**。它解决：

1. **回归保护**：改 prompt / 换模型 / 调 chunk 大小，能在合入前知道是变好还是变坏
2. **可观测**：线上每条 query 都打分，看长期趋势
3. **A/B 决策**：模型 A 比模型 B 真的好吗？提升显著吗？
4. **数据飞轮**：差评样本入训练集 / 测试集，持续提升

本章给一套从离线到在线的完整评测体系，并对比 Ragas / LangSmith / Braintrust / OpenAI Evals / DeepEval 等主流框架。

---

## 第一章：评测维度——你到底想测什么

不同 LLM 应用形态评测的维度不同。先想清楚"测什么"，再选"怎么测"。

### 1.1 通用对话 / 单轮 QA

| 维度 | 含义 | 测法 |
|---|---|---|
| **Helpfulness** | 是否真的解决用户问题 | LLM-as-judge / 人工 |
| **Correctness** | 事实是否准确 | golden answer 对比 / LLM judge |
| **Safety** | 有无违禁内容（暴力 / 政治 / PII 泄露） | 分类器 + 规则 |
| **Tone** | 语气是否符合品牌 | LLM judge / 人工 |
| **Format** | 是否按要求格式（JSON / Markdown / 长度） | 正则 / Schema 校验 |

### 1.2 RAG 系统

Ragas 论文（2023）总结的"RAG triad"已经成为业界共识：

```
                    Query
                      │
            ┌─────────┴─────────┐
            │                   │
       Retrieval           Generation
       (检索)               (生成)
            │                   │
            ▼                   ▼
        Contexts            Answer
            │                   │
            └────────┬──────────┘
                     │
                  Triad
```

- **Context Relevance**：检索到的 context 与 query 相关吗？（衡量检索质量）
- **Faithfulness / Groundedness**：answer 完全基于 context 吗？有没有幻觉？（衡量生成是否忠于来源）
- **Answer Relevance**：answer 真的回答了 query 吗？（衡量是否切题）

补充维度：

- **Context Precision / Recall**：相关 context 在前几条吗？相关 context 都召回了吗？
- **Citation Accuracy**：引用编号 [1] [2] 真的指向相关段落吗？

### 1.3 Agent 系统

Agent 评测最难，因为是多步决策：

| 维度 | 含义 |
|---|---|
| **Task Success** | 终态是否达成（订单是否真的下了）|
| **Tool Selection** | 每步选的工具对吗 |
| **Argument Correctness** | 工具参数对吗 |
| **Trajectory Efficiency** | 完成任务用了几步，有没有兜圈子 |
| **Recovery** | 工具失败后能否纠错 |

Agent 评测往往需要**模拟环境**——真打线上 API 太贵太慢。

### 1.4 代码生成

- **Compile / Lint**：能不能编译
- **Test Pass**：跑测试通过率（HumanEval 范式）
- **Static Analysis**：有没有安全漏洞（如 SQL 注入）
- **Diff Minimality**：是否只改了该改的，没乱动

### 1.5 结构化抽取 / 分类

- **Schema Validity**：JSON 是否合 schema
- **Field-level F1**：每个字段 precision/recall
- **Hallucinated Fields**：有没有出现 schema 里没定义的字段

---

## 第二章：评测方法论

### 2.1 五种打分方式

```
打分方式             成本    准确度   可重复  适用场景
─────────────────────────────────────────────────
精确匹配 (exact)     ¢       ★★     ★★★     有 golden answer
基于规则 (regex)     ¢       ★★     ★★★     格式 / 长度
传统指标 (BLEU/ROUGE)¢       ★★     ★★★     翻译 / 摘要
LLM-as-Judge         $$      ★★★★   ★★      开放生成
人工标注             $$$$    ★★★★★  ★★★     金标 / 难样本
```

工程上的常见组合：

1. **能用规则就用规则**——免费、确定
2. **不能用规则用 LLM-as-Judge**——便宜可扩展
3. **关键样本人工核对**——做 ground truth 校准 judge
4. **持续把 judge 错的样本送人工**——形成飞轮

### 2.2 LLM-as-Judge：让模型给模型打分

核心思想：用一个**更强的模型**当裁判。比如评测 Sonnet 4.6 的输出可以用 Opus 4.7 / GPT-5 当 judge。

**Pairwise（成对偏好）** 更稳定：

```
[问题] {query}
[回答 A] {answer_a}
[回答 B] {answer_b}

请判断哪个回答更好？A / B / Tie。
理由：...
```

**Pointwise（绝对评分）** 1-5 分：

```
[问题] {query}
[标准答案] {reference}
[模型回答] {answer}

请评分 1-5：
1=完全错  2=部分错  3=部分对  4=基本对  5=完全对
理由：...
```

**经验法则**：

- pairwise 比 pointwise 稳定（人类对"比较"也比"打分"更稳定）
- judge 模型至少比被评模型强一档（弱 judge 倾向给弱模型偏高分）
- **位置偏置（position bias）**：judge 倾向选 A——记得**两次评测交换位置取平均**
- **长度偏置（length bias）**：judge 倾向更长答案——评测 prompt 里强调"长度无关"
- **自我偏好（self-preference）**：模型倾向自己输出风格——尽量不用同家族当 judge

### 2.3 离线评测 vs 在线评测

```
                    评测体系
                       │
          ┌────────────┴────────────┐
          │                         │
       离线评测                  在线评测
       (CI / 回归)              (生产 / 监控)
          │                         │
   ┌──────┴──────┐           ┌──────┴──────┐
   │  golden set │           │  采样打分    │
   │  快照对比    │           │  用户反馈    │
   │  PR 门禁     │           │  长期趋势    │
   └─────────────┘           └─────────────┘
```

**离线（offline）**：

- 一套固定的测试集（golden dataset），每次 PR / commit 跑一遍
- 与基线对比：通过率上升 / 下降多少
- 类似单元测试：合入门禁

**在线（online）**：

- 线上真实流量，按比例采样
- 用 LLM judge / 用户反馈打分
- 监控时序：今天的 helpfulness 比上周降了 5%？告警
- 类似 APM：发现问题

成熟团队**两者都做**：离线挡 PR 回归，在线发现 drift。

### 2.4 Golden Dataset 怎么建

```
来源 1：客服 / 真实 query → 人工标答（最贵但最有价值）
来源 2：合成数据 LLM 生成 + 人工筛选
来源 3：开源 benchmark（MMLU / BBH / HumanEval / TriviaQA）
来源 4：线上 bad case 沉淀（持续飞轮）
```

**规模经验**：

| 阶段 | 样本数 | 用途 |
|---|---|---|
| MVP | 50-100 | 手工核验，每次发布前过一遍 |
| 早期 | 500-1000 | LLM-judge 自动跑 |
| 成熟 | 5000+ | 分层（easy/medium/hard）+ 标签 |

**陷阱**：

- 不要把训练数据 / Few-shot 例子放进测试集——污染
- 持续更新：模型在 2024 年的数据上跑得好，不代表 2026 年用户 query 也好
- 分层抽样：easy 80% + medium 15% + hard 5%，反映真实分布

---

## 第三章：Ragas——开源 RAG 评测的事实标准

Ragas（**Ra**g **as**sessment）是 RAG 评测最流行的开源框架。

### 3.1 核心指标

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,           # 生成是否忠于 context
    answer_relevancy,       # answer 是否切题
    context_precision,      # 相关 context 排在前面吗
    context_recall,         # 相关 context 都召回了吗
    answer_correctness,     # 与 golden answer 比对
)

from datasets import Dataset

data = Dataset.from_dict({
    "question": ["How do I reset password?", ...],
    "answer": ["Click forgot password ...", ...],
    "contexts": [["From user manual: ...", "..."], ...],
    "ground_truth": ["Go to settings > security ...", ...],
})

result = evaluate(
    dataset=data,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)

print(result)
# {'faithfulness': 0.87, 'answer_relevancy': 0.91, 'context_precision': 0.75, ...}
```

### 3.2 Faithfulness 是怎么算的

Ragas 把 answer 拆成**事实陈述（claims）**列表，每个 claim 让 judge 判断"是否能从 context 推出"：

```
Answer: "苹果创立于 1976 年，由乔布斯和沃兹尼亚克在车库里创办。"

Claims:
  1. 苹果创立于 1976 年
  2. 创始人是乔布斯和沃兹尼亚克
  3. 创立地点是车库

Context: "Apple was founded in 1976 by Steve Jobs and Steve Wozniak."

Verdict:
  1. ✓ supported
  2. ✓ supported
  3. ✗ unsupported  (context 没说车库)

Faithfulness = 2/3 = 0.67
```

这就是为什么**长 answer 容易扣分**——claim 多了某条没 context 支撑就拉低分。生产可以加约束："只回答 context 里有的内容"，强制模型保守。

### 3.3 实战：把 Ragas 嵌进 CI

```python
# eval.py
import sys
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

def run_eval(dataset_path: str, baseline_path: str | None = None):
    dataset = load_dataset(dataset_path)

    # 跑当前版本
    answers = []
    for row in dataset:
        a = my_rag.answer(row["question"])
        answers.append(a)

    result = evaluate(
        dataset=build_ragas_dataset(dataset, answers),
        metrics=[faithfulness, answer_relevancy],
    )

    # 与基线对比
    if baseline_path:
        baseline = load_json(baseline_path)
        for metric, score in result.items():
            delta = score - baseline[metric]
            print(f"{metric}: {score:.3f} ({delta:+.3f})")
            if delta < -0.02:  # 容忍 2% 内波动
                print(f"❌ regression on {metric}")
                sys.exit(1)

    print("✅ no regression")
    save_json("metrics.json", result)

if __name__ == "__main__":
    run_eval("tests/golden.jsonl", "baseline.json")
```

GitHub Action：

```yaml
- name: RAG Eval
  run: python eval.py
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
- name: Update baseline if main
  if: github.ref == 'refs/heads/main'
  run: cp metrics.json baseline.json && git commit ...
```

### 3.4 Ragas 的坑

- **judge 费用**：100 个样本 × 4 个指标 ≈ 400-1000 个 LLM call，注意预算
- **判别一致性**：同一样本跑两次结果不同——指标用 N=3 取平均更稳
- **指标偏置**：faithfulness 偏好短而保守的答案，不要单独优化它
- **多语言**：Ragas 默认 prompt 是英文，中文需要 override prompt 模板

---

## 第四章：LLM-as-Judge 自己手搓 vs 用框架

### 4.1 何时自搓

- 评测维度业务特殊（如"是否符合公司客服语气"）
- 指标需要嵌入业务逻辑（订单号是否在数据库存在）
- 想精细控制 prompt + 模型 + 缓存

### 4.2 自搓的最小骨架（Python）

```python
import anthropic, json
from typing import Literal

client = anthropic.Anthropic()

JUDGE_PROMPT = """你是严谨的评测员。请按 1-5 分评价以下回答。

[问题] {question}
[参考答案] {reference}
[模型回答] {answer}

评分标准:
5 = 完全正确，覆盖参考答案所有要点
4 = 基本正确，有小遗漏
3 = 部分正确，有错误或重要遗漏
2 = 大部分错误
1 = 完全错误

输出 JSON: {{"score": int, "reason": str}}
长度无关，只看内容。"""

def judge(question: str, reference: str, answer: str) -> dict:
    resp = client.messages.create(
        model="claude-opus-4-7",
        max_tokens=200,
        messages=[{
            "role": "user",
            "content": JUDGE_PROMPT.format(
                question=question, reference=reference, answer=answer),
        }],
    )
    return json.loads(resp.content[0].text)

# 批量评测 + 重复 N 次取平均
def judge_n(question, reference, answer, n=3):
    scores = [judge(question, reference, answer)["score"] for _ in range(n)]
    return sum(scores) / len(scores)
```

### 4.3 自搓的工程化关键点

- **缓存 judge 调用**：同 (q, ref, ans) 元组的 judge 结果存 Redis，省 70% 钱
- **并行**：用 `asyncio.gather` 或 `concurrent.futures`，注意 rate limit
- **审计**：把 judge prompt 与原始响应都存下来，半年后回头看 judge 标准有没有漂移
- **校准**：每月抽 100 个 judge 结果让人工核对，监测 judge 的"准确率"
- **prompt cache**：judge prompt 模板用 prompt cache（A16 章详述），TTL 5min 减少 token 计费

### 4.4 用框架 vs 自搓：怎么选

| 维度 | 自搓 | 框架（Ragas / DeepEval / Promptfoo） |
|---|---|---|
| 灵活度 | ★★★★★ | ★★★ |
| 上手速度 | ★★ | ★★★★★ |
| 维护成本 | ★★ | ★★★★ |
| 与业务深度结合 | ★★★★★ | ★★★ |
| 通用指标支持 | ★ | ★★★★★ |

**建议路径**：先用 Ragas / Promptfoo 起步，业务特殊指标自己加，最后整套自研。

---

## 第五章：商业评测平台对比

### 5.1 LangSmith

LangChain 出品，与 LangChain 深度集成。

- **强项**：trace + eval 一体，写 LangChain 就自动有 trace
- **特点**：支持 dataset 版本管理、Pairwise 实验对比、human review 流程
- **局限**：非 LangChain 应用需要手动 instrument
- **定价**：按 trace + eval 量计

### 5.2 Braintrust

后起之秀，AI native eval 平台。

- **强项**：UI 设计好、prompt playground 强、CI 集成方便
- **特点**：把"实验"作为一等公民——每次 commit 自动跑实验、Web 上看历史对比
- **局限**：相对新，企业级 RBAC 还在演进
- **定价**：按 span / eval 量计

### 5.3 Phoenix（Arize）

开源 + 商业版，OpenTelemetry 原生。

- **强项**：可观测 + eval 一体、可自托管、OTel 标准
- **特点**：支持 RAG / Agent / Drift detection
- **局限**：UI 复杂度高
- **定价**：开源免费，托管按用量

### 5.4 Promptfoo

开源 CLI，最像"jest for prompts"。

- **强项**：YAML 写测试用例，CLI 跑，CI 集成简单
- **特点**：天然支持多模型对比矩阵
- **局限**：Web 界面 / 团队协作弱
- **定价**：开源免费 + 企业版

### 5.5 OpenAI Evals

OpenAI 官方框架，开源。

- **强项**：与 OpenAI 模型深度集成，有大量公开 eval
- **特点**：Python + YAML 配置
- **局限**：偏 OpenAI 生态、UI 弱
- **定价**：开源，但跑 eval 自己付 API 费

### 5.6 选型建议

```
小团队 / 开源优先          → Promptfoo + Phoenix
LangChain 重度用户         → LangSmith
重视实验对比 + UI          → Braintrust
RAG 专项                  → Ragas（开源）+ 上面任一平台
合规 / 自托管要求          → Phoenix self-hosted
```

---

## 第六章：在线评测——生产流量打分

### 6.1 采样策略

不要对 100% 流量打分——judge 也要花钱。

```python
import random

def should_eval(query_id: str, sample_rate: float = 0.05) -> bool:
    # 一致性 hash 而不是纯随机，方便复测
    return hash(query_id) % 10000 < int(sample_rate * 10000)
```

**分层采样**：

- 5% 通用流量
- 100% 用户点了 👎 的样本
- 100% 触发新 feature flag 的样本
- 100% 高价值用户 / VIP

### 6.2 打分链路

```
用户 query
   │
   ▼
LLM 应用 ─── 同步返回 answer 给用户
   │
   ▼ (异步)
消息队列 (Kafka / SQS)
   │
   ▼
Eval Worker (judge / 规则)
   │
   ▼
TSDB (Prometheus / ClickHouse)
   │
   ▼
Dashboard / 告警
```

**绝对不要把 judge 放在同步路径上**——judge 几秒，用户等不起。

### 6.3 必看指标看板

| 指标 | 计算 | 告警阈值 |
|---|---|---|
| Helpfulness avg | LLM judge 1-5 平均 | < 3.5 |
| Hallucination rate | unfaithful 样本占比 | > 5% |
| User thumb-down rate | 👎 / 总反馈 | > 8% |
| Refusal rate | 模型拒答占比 | 异常 ±50% |
| Citation accuracy | 引用正确率 | < 90% |
| Mean latency P95 | 端到端 | 业务定 |

监控**变化率**比绝对值更重要——指标突然下降通常是 prompt / 模型变更引起。

### 6.4 用户反馈：显式 + 隐式

```typescript
// 显式：👍 👎
function rateAnswer(messageId: string, rating: "up" | "down") {
  analytics.track("answer_rated", { messageId, rating });
}

// 隐式：用户行为信号
- 用户问完没追问 (满意?)
- 用户立即追问 "不对，我是说..." (不满意)
- 用户复制了 answer 内容 (有用)
- 用户停留时长
- 用户最终成单 (终极指标)
```

**隐式信号噪声大**——可以用 LLM 把"对话上下文 → 用户满意度"做训练数据。

---

## 第七章：Agent 评测——多步轨迹的难题

### 7.1 轨迹评测

Agent 的"对错"不是单步而是整条 trajectory。

```python
def eval_agent_trajectory(trajectory: list[Step], task: Task) -> dict:
    return {
        "task_success": did_complete_task(trajectory, task),  # 终态对吗
        "tool_correctness": [judge_step(s) for s in trajectory],  # 每步对吗
        "trajectory_efficiency": len(trajectory) / task.expected_steps,
        "recovered_from_error": detect_recovery(trajectory),
    }
```

### 7.2 模拟环境

线上的 Agent 调真 API（订单、支付、发邮件），eval 不能用真环境。两种解法：

**Mock Sandbox**：

```python
# Agent 看到的 tool 接口不变
class MockOrderAPI:
    def __init__(self, fixture):
        self.fixture = fixture
        self.state = {}

    def create_order(self, items):
        # 不调真服务，记录调用 + 返回 fixture
        self.state["last_order"] = items
        return self.fixture["create_order_response"]
```

**Replay**：

录制线上某个 case 的所有 tool call + 响应，replay 给新版本 Agent，看走的路径是否合理。

### 7.3 Levels of Eval

```
L1: tool calling 单元测试（given input, expect tool & args）
L2: 单 turn agent task（给指令，看一步内能否调对 tool）
L3: multi-turn agent task（给目标，看 5-20 步能否完成）
L4: 长任务（小时级，开放任务，多工具，难度接近 SWE-bench）
```

不要一上来跑 L4——先把 L1-L2 打稳。

---

## 第八章：A/B 实验设计

LLM 应用上线新 prompt / 新模型时，要做严肃 A/B：

### 8.1 假设与样本量

```
H0: 新 prompt 与旧 prompt 效果相同
H1: 新 prompt 显著更好

样本量 = (Zα + Zβ)² · σ² / δ²

  - α = 显著性水平 (常用 0.05)
  - β = power (常用 0.8)
  - σ = 指标标准差
  - δ = 想检测的最小差异 (effect size)
```

实际操作：

- 改 prompt 想测 +5% helpfulness——通常 1000-2000 样本足够
- 改模型想测 +0.5% 转化率——可能需要数万样本
- 极小 effect 的 A/B 慎做，可能根本检测不出来

### 8.2 假阳性

跑 20 个 metric 总有 1 个 p<0.05——这就是 **multiple testing problem**。修法：

- 预定义**主指标**（primary metric），其它是 secondary
- 用 Bonferroni 或 Benjamini-Hochberg 校正
- 不要 p-hack（跑出好看的就停）

### 8.3 别忘了反向指标

- 新模型 helpfulness 涨了 → 平均 token 数也涨了 → 成本涨了 30%
- 新 prompt 准确率高 → 拒答率也高 → 用户体验降
- 永远配套监控**反向指标**

---

## 第九章：评测的反模式

### 9.1 用训练数据测

模型对训练数据"过拟合"，看着 95% 实际线上 60%。

**修法**：建立独立 holdout set，永不进入训练 / few-shot。

### 9.2 单一指标论英雄

只看 BLEU / 只看 Faithfulness——总能找到办法刷分。

**修法**：3-5 个互补指标，必要时加人工抽查。

### 9.3 judge 与被评模型同源

用 GPT-4 评 GPT-4，self-preference 导致偏差。

**修法**：换家族 judge，或 ensemble 多 judge。

### 9.4 缺基线

没有"上一个版本"对比，单点 0.85 不知道是好是坏。

**修法**：维护 baseline.json，每次回归与之对比。

### 9.5 评测数据陈旧

去年的 query 早就解决了，新 query 完全没在测试集里。

**修法**：定期注入线上新流量（脱敏 + 标注）。

### 9.6 评测 ≠ 上线

离线指标好，线上用户不一定满意——离线指标只是 proxy。

**修法**：离线 + 在线 + 用户反馈三层。

---

## 第十章：陷阱清单

1. **judge LLM 的成本可能比业务调用本身还高**——尤其每个样本 N=3 重复 + 5 个指标 × 1000 样本 × $0.003 / call。预算化。
2. **judge 不是真相**——LLM judge 与人工评分的相关性大约 0.7-0.85，不要把 judge 输出当 ground truth。
3. **结构化 judge 输出经常坏掉**——judge 偶尔不出 JSON。要么用 tool use 强制 schema，要么 try/except 重试。
4. **测试集分布偏移**——你的 1000 条 golden 反映去年用户行为，模型在新场景表现谁也不知道。
5. **小指标差异没意义**——0.85 vs 0.83 在 100 样本上根本不显著，别声明胜利。
6. **不要在 prompt 里写 "use chain of thought" 让 judge 输出推理**——但又要解析最终分——常常解析失败。让 judge 用 tool use 输出。
7. **judge 模型升级会让历史指标失效**——judge 用 Sonnet 4.5 跑了一年，换 4.6 之后所有指标都跳——保留固定版本。
8. **位置偏置不修就废**——pairwise 一定要 A/B 交换两次取平均。
9. **缺 hard set**——平均分 4.5 看着好看，hard subset 可能 2.0——分层报告。
10. **没人看的看板等于没有**——评测指标必须有 owner，每周 review。

---

## 第十一章：2026 现状

- **OpenAI Evals 框架仍是开源标杆**，但商业平台体验远超
- **Braintrust 与 LangSmith 双雄**——前者 UI 优、后者生态强
- **Ragas 成 RAG 评测事实标准**，已被 Phoenix / Braintrust / LangSmith 内置
- **LLM-as-Judge 已是行业默认**——但社区对其偏置（length / position / self-preference）研究热门
- **Constitutional AI / RLAIF**：用 judge 给模型 RL 反馈，已是前沿训练 pipeline
- **多 judge ensemble**：取多家模型 judge 的多数投票，比单 judge 提升 0.1-0.15 一致性
- **AgentBench / SWE-bench / GAIA** 成 Agent 评测公开 benchmark
- **可观测 + Eval 融合**：OTel + Phoenix / Langfuse 让 trace 即 eval，新一代标配

---

## 第十二章：练习题

1. ⭐ 解释 RAG triad 的三个维度，分别衡量什么？
2. ⭐ 为什么 pairwise judge 比 pointwise 更稳定？
3. ⭐⭐ LLM-as-Judge 的三种偏置（位置 / 长度 / 自我），各自怎么缓解？
4. ⭐⭐ 写一个 Ragas 评测脚本：评测 faithfulness + answer_relevancy，并在分数低于上次 0.02 时让 CI 失败。
5. ⭐⭐⭐ 设计一套 Agent 评测体系，覆盖 L1-L3 三个层级，并描述测试用例的来源。
6. ⭐⭐⭐ 线上某指标突然降了 5%，给出排查 checklist（数据 / 模型 / prompt / 上下游）。
7. ⭐⭐⭐ 用 Braintrust 或 LangSmith 设计 prompt A/B 实验，包含主指标 / 反向指标 / 样本量估算。

<details>
<summary>📝 参考思路</summary>

1. context relevance（检索召回是否相关）/ faithfulness（生成是否忠于检索）/ answer relevance（回答是否切题），互补三角形。
2. 人类对"哪个更好"判断更稳定，绝对分数受心情 / 锚定 / 标准化影响；pairwise 也免去了不同 judge 用不同绝对刻度的问题。
3. 位置 → A/B 互换两次取平均；长度 → judge prompt 明示"长度无关"+ 用 token 数加正则化；自我 → 换家族模型当 judge / 多 judge ensemble。
4. 见第三章 3.3 节代码。
5. L1 单 step（assert tool name + args）；L2 单 turn task（"创建订单 X"，验终态）；L3 multi-turn task（"帮我找到便宜机票并下单"，验全链路 + 中途扰动恢复）。用例来源：客服真实 case 脱敏 + 合成扰动。
6. ① 是不是模型供应商升级？② 是不是 prompt 改了？③ 是不是新 feature 流量分布变了？④ 是不是测试集污染了？⑤ 是不是 judge 升级了？逐项对照 git log + 部署日志 + 流量指标。
7. 主指标：helpfulness（用 judge 1-5 打分）；反向指标：token 成本、拒答率、P95 延迟；样本量：估算 σ=0.8，δ=0.1，α=0.05，β=0.8 → ~1000 样本/桶。

</details>

---

## 小结

LLM 应用没有评测就是"靠肉眼维护"——能撑半年但绝撑不过一年。

本章给的是一条工程化路径：

```
golden dataset
     │
     ▼
Ragas / 自搓 judge —— 离线门禁（CI 挡回归）
     │
     ▼
线上采样 + judge worker —— 在线监控（发现 drift）
     │
     ▼
用户反馈 + 飞轮 —— 持续补 hard case 进 golden
```

下一步去看 [A16 LLM 成本与延迟优化](./A16-精通-LLM-成本与延迟优化.md)——评测是质量保障，成本与延迟是商业可行性保障，两手都要硬。
