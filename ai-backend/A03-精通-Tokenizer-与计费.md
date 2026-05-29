# 精通 Tokenizer 与计费：BPE、Context Window、Prompt Caching 与成本工程

> 课程编号：A03
> 路线图来源：AI / LLM 后端工程 · 模块一 API 基础
> 难度：⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 5 月

---

## 引言：Token 是 LLM 的"计量单位"

```
"Hello, world!" → ["Hello", ",", " world", "!"]      → 4 个 token
"你好，世界！"   → ["你", "好", "，", "世", "界", "！"]  → 6 个 token
"日本語"         → ["日本", "語"] 或 ["日", "本", "語"]   → tokenizer 决定
```

LLM 不是按字符或单词收费——它按 **token** 计费。Token 是模型词表里的一个子词单位。BPE / SentencePiece / WordPiece 等不同算法切出来的 token 边界不一样，**同一句话在 GPT、Claude、Gemini 上 token 数能差 15-30%**。

这看似小事，放到生产环境就是真金白银。一个用 Sonnet 4.6 处理 1M 输入的批量任务，按 0.3 USD/M 算缓存命中价大约 0.3 美元；按 3 USD/M 算未命中价是 3 美元——10 倍差距。再放大到日活十万的客服系统，月成本可以差**百万美元**。

本章把 LLM 后端的"度量基础"拆开：

1. BPE 怎么把文本切成 token
2. tiktoken / Claude tokenizer / SentencePiece 的差异
3. Go 里怎么估算 token、怎么调 count_tokens API
4. context window 从 8k 走到 2M 的历史与代价
5. input / output / cache write / cache read 的价差与组合
6. Batch API 半价机制
7. 长上下文经济学——prompt caching + context engineering
8. 2026 年主流模型价格对比表
9. 成本估算公式与 Go 实现
10. 生产中的预算控制、token 限流、配额隔离

读完你应该能：闭眼算出一个 prompt 大致花多少钱；看一眼业务流量就能说出"该用什么模型、要不要打 cache、要不要走 batch"。

---

## 第一章：什么是 Token

### 1.1 文本 → token 的三种切法

```
1. 字符级（character-level）  : 每个字符一个 token
   "hello" → ["h","e","l","l","o"]  5 个 token
   词表小（~256），序列长，模型难学语义。早期 RNN 才这么干。

2. 词级（word-level）         : 每个词一个 token
   "hello world" → ["hello","world"]  2 个 token
   词表大（百万级 OOV 严重），无法处理新词 / 拼错。

3. 子词级（subword）          : 折中
   "unbelievable" → ["un","believ","able"]  3 个 token
   词表中等（3 万 - 20 万），泛化好。**现代 LLM 全用这个。**
```

子词主流算法：

- **BPE（Byte-Pair Encoding）**：GPT 系、Claude、LLaMA 都用
- **WordPiece**：BERT 用
- **SentencePiece**：T5、Gemma、Qwen、ChatGLM 用（语言无关）
- **Unigram LM**：SentencePiece 可选的另一种切分

### 1.2 Token 的本质

Token 不是固定单位——**它是模型词表里的一个 ID**。

```
GPT-4o tokenizer (cl100k):  "hello" → ID 15339
Claude tokenizer:           "hello" → ID 不同
Gemini tokenizer:           "hello" → ID 不同
```

模型推理时输入是 token ID 序列，输出也是 token ID 序列，最后用 tokenizer 反查表变文本。**`tokenize` 与 `detokenize` 是无损双向函数**。

### 1.3 为什么用子词

```
词表大小：    150,000  →  ~50,000 - 100,000
英文一个 token ≈ 4 个字符 ≈ 0.75 个英文单词
中文一个 token ≈ 1 - 2 个汉字
代码一个 token ≈ 比英文略长（结构性强、空格多）
```

经验法则：

- **英文 1000 字 ≈ 1300 - 1500 token**
- **中文 1000 字 ≈ 700 - 1200 token**
- **代码 1000 字 ≈ 200 - 400 token**

代码相对省 token——因为 BPE 会把 `function`、`return`、`{`、`}` 等高频片段合并成单 token。中文每个汉字往往独占 1 个 token（部分高频词如"中国"才合并）。

### 1.4 Tokenizer 决定模型的"知识颗粒度"

Tokenizer 是模型训练前**先**固定下来的——一旦训练完成不能改。这意味着：

- 罕见词 / 新词 → 切成多个 token（更费钱、模型理解更差）
- 高频片段 → 合并成单 token（省钱、表达密集）
- emoji / 特殊字符 → 多个 byte token（一个 emoji 经常 2-4 token）

举例（Claude / GPT 类似）：

```
"GPT-4"          → 3 token  ['G', 'PT', '-4']
"transformer"    → 1 token  ['transformer']         （太常见了）
"polysemous"     → 4 token  ['p','oly','sem','ous']  （罕见词）
"🤖"             → 2-3 byte token                    （表情）
```

理解这点能解释**为什么 LLM 写诗时偏爱常见词**、为什么提示词里塞 emoji 浪费 token、为什么 JSON 比 XML 省 token（XML 标签每次重复都算 token）。

---

## 第二章：BPE 算法原理

### 2.1 算法直觉

BPE 是 1994 年的数据压缩算法，2015 年被 Sennrich 等人引入 NMT。核心思路：

```
1. 初始词表 = 所有单字符 / byte
2. 在大语料里统计相邻 token 对的出现次数
3. 取频率最高的对，合并成新 token，加入词表
4. 重复 2-3 直到达到目标词表大小
```

### 2.2 训练过程示意

```
语料：["low", "lower", "newest", "widest"]
初始 split:
  l o w </w>
  l o w e r </w>
  n e w e s t </w>
  w i d e s t </w>

第 1 轮：最高频对是 (e, s) → 5 次。合并：
  l o w </w>
  l o w e r </w>
  n e w es t </w>
  w i d es t </w>

第 2 轮：(es, t) → 4 次。合并：
  l o w </w>
  l o w e r </w>
  n e w est </w>
  w i d est </w>

第 3 轮：(l, o) → 3 次。合并 lo。
第 4 轮：(lo, w) → 3 次。合并 low。
...

最终词表里有：l, o, w, e, r, n, s, t, i, d, low, es, est, lower, ...
```

### 2.3 编码（encode）过程

```
输入：  "lowest"
1. 切到最细：l o w e s t
2. 应用合并规则（按训练顺序）：
   (e, s) → es      : l o w es t
   (es, t) → est    : l o w est
   (l, o) → lo      : lo w est
   (lo, w) → low    : low est
3. 输出 token：["low", "est"]
```

整个过程**贪心 + 确定性**——同一文本永远切出同样的 token 序列。

### 2.4 Byte-Level BPE（GPT-2 起）

朴素 BPE 词表是 Unicode 字符级——中文、Emoji 字符就有上万个，词表会爆。GPT-2 改成 **byte 级 BPE**：

```
"中"  UTF-8  →  3 bytes: E4 B8 AD
"🤖"  UTF-8  →  4 bytes: F0 9F A4 96
```

初始词表只有 256 个 byte，任何 Unicode 都能用 byte 序列表示。然后 BPE 在 byte 序列上合并。

好处：**词表小（5 万左右就够）、任何文本都能编码（包括二进制）**。代价：罕见语言 / 表情每字符 1-4 token。

GPT-4 用的 `cl100k_base` 与 GPT-4o 的 `o200k_base`、Claude 用的内部 tokenizer 都是 byte-level BPE 的变种。**`o200k_base` 词表 20 万——比 GPT-4 翻倍，对中文 / 代码更省 token**（约 1.4 倍效率）。

### 2.5 BPE 的工程实现

```go
type BPE struct {
    Vocab  map[string]int   // token → id
    Merges []Pair           // 训练时的合并顺序
}

type Pair struct{ A, B string }

func (b *BPE) Encode(text string) []int {
    // 1. byte 化（UTF-8）
    bytes := []byte(text)
    tokens := byteToInitialTokens(bytes)
    
    // 2. 应用 merges
    for _, m := range b.Merges {
        tokens = applyMerge(tokens, m)
    }
    
    // 3. 查词表
    ids := make([]int, len(tokens))
    for i, t := range tokens {
        ids[i] = b.Vocab[t]
    }
    return ids
}
```

实际实现里 `applyMerge` 不是线性扫描——用优先队列（priority queue by merge rank）能做到 O(n log n)。Hugging Face 的 `tokenizers` 库（Rust）和 OpenAI 的 `tiktoken`（Rust + Python）都这么实现。

---

## 第三章：主流 Tokenizer 对比

### 3.1 三大类

```
┌────────────────────────────────────────────────────────────┐
│ 厂商           │ 算法            │ 词表       │ 代表模型      │
├────────────────────────────────────────────────────────────┤
│ OpenAI         │ tiktoken (BPE)  │ 100k/200k  │ GPT-4/4o/5    │
│ Anthropic      │ 内部 BPE        │ ~100k      │ Claude 4.x    │
│ Google         │ SentencePiece   │ 128k/256k  │ Gemini 2.5    │
│ Meta           │ tiktoken-like   │ 128k/256k  │ LLaMA 3/4     │
│ Mistral        │ SentencePiece   │ ~32k/128k  │ Mistral Large │
│ 阿里 Qwen      │ tiktoken-like   │ 152k       │ Qwen 2.5/3    │
│ 智谱 GLM       │ SentencePiece   │ 130k       │ GLM-4.5/4.6   │
│ DeepSeek       │ tiktoken-like   │ 128k       │ DeepSeek V3   │
│ Moonshot Kimi  │ tiktoken-like   │ ~150k      │ Kimi K2       │
└────────────────────────────────────────────────────────────┘
```

### 3.2 同一段中文的 token 数对比

来看个真实例子（约 100 个汉字的新闻片段）：

```
原文 105 字：
"2026 年 5 月，OpenAI 正式发布 GPT-5 Turbo，主打 1M 上下文与新的多模态能力。
该模型在中文 MMLU 上得分 92.3，超过此前 Claude Opus 4.7 的 91.8。
开发者价格 input 5 USD / 1M tokens、output 25 USD。"
```

各 tokenizer 实测 token 数（近似值，2026.05）：

| Tokenizer | Token 数 | 单字平均 |
|---|---|---|
| `o200k_base` (GPT-4o/5) | 142 | 1.35 |
| Claude internal | 168 | 1.60 |
| Gemini SentencePiece | 156 | 1.49 |
| Qwen2.5 tiktoken-like | 138 | 1.31 |
| GLM-4 SentencePiece | 145 | 1.38 |

结论：**国产模型对中文最省**（自家训练语料中文占比高）；GPT-4o `o200k_base` 在主流模型中对中文友好度排第二；**Claude 中文 token 数大约比 GPT-4o 多 18%**——这是核算成本时容易踩的坑。

英文则反过来——Claude 与 GPT 接近，中文差更明显。

### 3.3 tiktoken / Claude tokenizer / SentencePiece 差异

#### tiktoken（OpenAI）

- 开源（Apache 2.0）：`github.com/openai/tiktoken`
- byte-level BPE
- 提供 `cl100k_base`（GPT-3.5/4）、`o200k_base`（GPT-4o/5）
- 离线即可算 token——本地 SDK 性能极高

```python
import tiktoken
enc = tiktoken.get_encoding("o200k_base")
len(enc.encode("hello world"))  # → 2
```

Go 版本：`github.com/pkoukk/tiktoken-go`（移植）。

#### Claude tokenizer

- **未开源**——Anthropic 内部 BPE
- 早期 SDK 有 `anthropic.count_tokens`（基于本地 tokenizer 文件）
- **2024 年起 Claude 3 系列起官方 SDK 移除本地 tokenizer，改用云端 `/v1/messages/count_tokens` API**
- 你可以本地"估"——通常用 GPT-4 tokenizer 估，再乘 1.15-1.2 系数

```bash
curl https://api.anthropic.com/v1/messages/count_tokens \
  -H "x-api-key: $KEY" \
  -H "anthropic-version: 2024-06-01" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [{"role": "user", "content": "hello"}]
  }'
# 返回 {"input_tokens": 8}
```

#### SentencePiece（Google / Meta / 国内多家）

- 开源（Apache 2.0）：`github.com/google/sentencepiece`
- 语言无关——直接在 raw text（UTF-8 byte）上做
- 支持 BPE 和 Unigram LM 两种切分模式
- 训练时用户提供 `vocab_size`、模型自动学切分

特点：把空格当作 `▁`（U+2581）特殊字符——`"hello world"` → `["▁hello","▁world"]`。这样反编码完全可逆（哪怕首字符是空格也保留）。

Gemini、Gemma、LLaMA 早期、所有 Qwen 系列、GLM 系列都基于 SentencePiece。

### 3.4 为什么不能直接用 tiktoken 估 Claude token 数

- 算法接近但**词表不同**——同一段文本切法不同
- Claude 词表对中文颗粒度比 `o200k` 更细，token 数偏多
- 工程实践：用 `tiktoken-go` 的 `o200k_base` 算结果 ×1.18 作为 Claude 上限近似（生产里测算误差 ±5%）

---

## 第四章：用 Go 估 Token 数

### 4.1 安装 tiktoken-go

```bash
go get github.com/pkoukk/tiktoken-go
```

注：纯 Go 实现，但 `bpe rank` 文件从 OpenAI CDN 下载，离线环境需要预下载。

### 4.2 估 OpenAI 模型 token 数

```go
package main

import (
    "fmt"
    tiktoken "github.com/pkoukk/tiktoken-go"
)

func countOpenAI(text, encoding string) int {
    enc, err := tiktoken.GetEncoding(encoding)
    if err != nil {
        panic(err)
    }
    return len(enc.Encode(text, nil, nil))
}

func main() {
    text := "2026 年 5 月，LLM 时代进入成熟期。"
    fmt.Println(countOpenAI(text, "o200k_base"))    // GPT-4o/5
    fmt.Println(countOpenAI(text, "cl100k_base"))   // GPT-4/3.5
}
```

`Encode(text, allowedSpecial, disallowedSpecial)` 的后两个参数控制 `<|endoftext|>` 等特殊 token 行为，正常文本用 nil 即可。

### 4.3 估包含 messages 的 token 数

Chat Completions API 每条 message 还有 role / name 等元数据要算。OpenAI 官方文档给的算法：

```go
// 适用于 gpt-4 / gpt-4o / gpt-5 系列
func countMessages(messages []Message, model string) int {
    enc, _ := tiktoken.EncodingForModel(model)
    tokensPerMessage := 3 // role + content + 分隔
    total := 0
    for _, m := range messages {
        total += tokensPerMessage
        total += len(enc.Encode(m.Role, nil, nil))
        total += len(enc.Encode(m.Content, nil, nil))
        if m.Name != "" {
            total += len(enc.Encode(m.Name, nil, nil))
        }
    }
    total += 3 // <|start|>assistant<|message|>
    return total
}
```

实际 token 数与 API 计费偏差通常 < 1%（多模态例外——图片要按 patch / tile 单独算）。

### 4.4 调 Anthropic count_tokens API

```go
package main

import (
    "context"
    "github.com/anthropics/anthropic-sdk-go"
)

func countClaude(ctx context.Context, client anthropic.Client, text string) (int64, error) {
    resp, err := client.Messages.CountTokens(ctx, anthropic.MessageCountTokensParams{
        Model: anthropic.F(anthropic.ModelClaudeSonnet4_6),
        Messages: anthropic.F([]anthropic.MessageParam{
            anthropic.NewUserMessage(anthropic.NewTextBlock(text)),
        }),
    })
    if err != nil {
        return 0, err
    }
    return resp.InputTokens, nil
}
```

注意：

- `count_tokens` 这个调用本身**不计费**（2026 年 5 月起 Anthropic 已确认免费）
- 但有 RPM 限制（与正常 messages 共享 RPM 桶）
- 返回值含 system / tools / messages 全部 input token
- 不返回 output token 估算——output 长度是模型决定的

### 4.5 离线粗估 Claude token

如果不想每次都打 API，用 tiktoken-go 估算 + 系数：

```go
func estimateClaudeTokens(text string) int {
    enc, _ := tiktoken.GetEncoding("o200k_base")
    base := len(enc.Encode(text, nil, nil))
    
    // 中文系数 1.18，英文 1.05，混合 1.12
    ratio := detectChineseRatio(text)
    factor := 1.05 + (1.18-1.05)*ratio
    
    return int(float64(base) * factor)
}

func detectChineseRatio(text string) float64 {
    chinese := 0
    total := 0
    for _, r := range text {
        total++
        if r >= 0x4E00 && r <= 0x9FFF {
            chinese++
        }
    }
    if total == 0 {
        return 0
    }
    return float64(chinese) / float64(total)
}
```

误差通常 ±5%；做配额预警、UI 提示足够。生产做严格预算控制的场景还是要调 count_tokens API。

### 4.6 SentencePiece 在 Go 里

```go
// github.com/eliben/sentencepiece-go 或自己写 cgo binding
// 大多数 Go 项目对 Gemini / 国产模型直接调云端 count API，少有本地 SP 需求
```

阿里、智谱、DeepSeek 等都提供 OpenAI 兼容 API 但**没有本地 tokenizer**——只能用云端接口或用 cl100k 粗估（实测误差 5-15%）。

### 4.7 多模态 token 估算

#### 图片

OpenAI 的算法（GPT-4o vision，2026 仍适用）：

```
"low" detail (固定 85 token)
"high" detail:
    1. 缩放到 ≤ 2048x2048（长边）
    2. 短边缩到 768
    3. 切 512x512 patch（向上取整）
    4. token = patches × 170 + 85
```

```go
func countImageTokens(width, height int, detail string) int {
    if detail == "low" {
        return 85
    }
    // high
    w, h := scaleDown(width, height, 2048)
    w, h = scaleShortSide(w, h, 768)
    tilesW := (w + 511) / 512
    tilesH := (h + 511) / 512
    return tilesW*tilesH*170 + 85
}
```

Claude vision 算法：图片按近似公式 `tokens ≈ (width × height) / 750`（向上取整 ~1000-1600 token / 标准图）。

#### 音频（Whisper / Gemini Audio）

每秒约 25 token（Gemini，2026 数据）。1 分钟语音 ≈ 1500 token。

#### 视频（Gemini）

每秒 1 帧抽样 × 每帧 258 token + 音轨。1 分钟视频 ≈ 17000 token。**视频极贵**——做应用要做"先识别关键帧"等预处理。

---

## 第五章：Context Window 演进

### 5.1 时间线

```
2022.06   GPT-3 (text-davinci-002)        4k
2023.03   GPT-4                          8k → 32k
2023.07   Claude 2                      100k (业界首次)
2023.11   GPT-4 Turbo                   128k
2024.02   Claude 3 (Opus/Sonnet/Haiku)  200k
2024.02   Gemini 1.5 Pro                1M  (业界首次)
2024.05   Gemini 1.5 Pro                2M  (实验)
2024.06   Claude 3.5 Sonnet             200k
2025.02   GPT-4.5 / o3                  200k
2025.06   Claude 4 series               200k
2025.10   Claude Sonnet 4.5 1M beta     1M
2025.12   GPT-5                         400k (main) / 1M (long-ctx beta)
2026.03   Claude Sonnet 4.6 1M (GA)     1M
2026.02   Gemini 3.1 Pro                2M (GA)
```

### 5.2 长 context 的工程意义

```
8k     :    勉强放下一段长邮件
32k    :    放得下一篇技术文章
128k   :    放得下一本 200 页书 / 一个中型项目代码
200k   :    放得下一本 350 页书 / 大多数 RAG 场景一次性塞
1M     :    放得下一整个中型 codebase（如 React 全源码）
2M     :    放得下完整代码库 + 历史 commits / 几小时视频
```

但 context 不是"越大越好"——存在**几个铁律**：

1. **价格非线性**：超过某阈值（如 200k）按更高梯度计费（Sonnet 4.6 在 ≤ 200k 段 3/15 USD，> 200k 段 6/22.5 USD）
2. **延迟非线性**：1M 上下文首 token 延迟 5-20 秒
3. **质量"中间丢失"现象（lost-in-the-middle）**：超长上下文中间部分召回率下降
4. **memory / KV cache 成本**：模型推理时 KV cache 占显存与 context 长度线性相关

### 5.3 200k 与 1M 的实测对比

```
任务：在 800k token 的代码库里找一个函数定义。

Sonnet 4.6（200k 截断）：要先 RAG 检索 / chunk，召回率取决于 embedding 质量
Sonnet 4.6（1M）：     可一次性扔进去，准确率高但单次成本 5-10x
Gemini 2.5 Pro（1M）： 可扔进去 + 视频 / 长音频；延迟最长（需 2M 用 Gemini 3.1 Pro）
```

**经验**：≤ 100k 直接全文喂；100k-500k 考虑 RAG + 缓存；> 500k 走专门的长上下文模型。

### 5.4 lost-in-the-middle 现象

Liu et al. 2023 的研究：在长 context 中间放关键信息，准确率比放开头 / 结尾低 20-40%。2024-2025 模型逐步缓解，2026 年的 Claude Sonnet 4.6 / GPT-5 / Gemini 2.5 在 needle-in-haystack 基准上接近 100% 召回，但**复杂综合推理仍有衰减**。

工程实践：

- 长 context 关键信息放开头 / 结尾
- 复杂任务 prompt 里写 `先复述关键信息再回答`，强制模型"重新关注"
- 把 100% 信息装进去不如**精挑 30%**——context engineering 比 context 大小更重要

---

## 第六章：计费模型

### 6.1 五个计费维度

```
1. Input tokens       基础——每次都算
2. Output tokens      基础——每次都算（通常 3-5x input 价）
3. Cache creation     首次写入 prompt cache（贵）
4. Cache read         命中 cache（极便宜）
5. 多模态附加          图片 / 音频 / 视频按"等效 token"
```

Batch 模式所有维度 ×0.5（除 cache read 已经够便宜）。

### 6.2 各家计费维度差异

#### Anthropic

```
input        x
output       5x
cache write  1.25x  (5min TTL)
cache write  2.0x   (1h TTL)
cache read   0.1x
batch        0.5x   (input & output)
```

显式 `cache_control` 标记 → 用户主动控制。

#### OpenAI

```
input              x
output             5x
cached input       0.5x      (自动 cache，5-60min)
batch              0.5x
```

OpenAI 的 prompt caching **自动启用**——不需要 cache_control。命中规则：前缀 ≥ 1024 token 且 5-60 分钟内重复请求。命中 cache 后 input ×0.5（注意比 Anthropic 0.1x 贵 5 倍——OpenAI 是"半价"，Anthropic 是"一折"）。

#### Gemini

```
input            x
output           5x
context cache    0.25x  (input)
implicit cache   自动 0.25x  (2.5 Pro 起，2026 GA)
batch            0.5x
```

Google 提供"显式上下文缓存"——用户先 POST 一段 cached_content，得到 ID，后续请求引用 ID 即可。命中后 input ×0.25。**Gemini 2.5 起还有 implicit caching**——自动识别 prefix。

#### 国产（Qwen / GLM / DeepSeek）

```
input        x
output       2-5x
cache read   0.1-0.2x （部分模型支持，命中规则各异）
batch        0.5x   （部分平台）
```

国产平台单价**比头部美厂低 50-80%**——同等质量级别下中国市场是"价格洼地"。但 SLA、并发上限、长 context 支持参差不齐——做生产前一定要测。

### 6.3 输入 / 输出价差

```
Sonnet 4.6 (≤ 200k 段):
  input  3 USD / 1M
  output 15 USD / 1M
  比例 5:1
```

几乎所有模型 output 都比 input 贵 3-5 倍。原因：

- Output 需要逐 token 自回归生成——每个 token 都要全网络前向
- Input 可批量并行处理（一次性吃进去）
- KV cache 在 output 阶段持续增长

**工程含义**：

- 长 input + 短 output 任务**最便宜**（RAG QA、分类、抽取）
- 长 output 任务**最贵**（代码生成、长文创作）
- 想省钱：把"思考"放到 prompt 里（input 便宜），让模型"直接回答"（output 短）

### 6.4 Cache 价差

```
普通 input         3 USD / 1M    (基线)
Cache write (5m)   3.75 USD       (1.25x)
Cache write (1h)   6 USD          (2x)
Cache read         0.3 USD        (0.1x)   ← 关键
```

**Cache read 比 cache write 便宜 10-20 倍**。这是 Anthropic 设计的核心动机——**鼓励长 system prompt + 多轮调用**。

收益场景实测（Claude Sonnet 4.6）：

```
任务：50KB 知识库 + 10 KB 历史 + 1 KB 问题 → 1 KB 答案

无 cache:
  input: 60k × 3 USD/M = $0.18
  output: 1k × 15 USD/M = $0.015
  总:  $0.195

打 cache（命中）:
  cache write 一次: 60k × 3.75 USD/M = $0.225 （首次）
  后续命中: 60k × 0.3 USD/M = $0.018 + new input 1k × 3 USD/M = $0.003 + output $0.015 ≈ $0.036
  
对话 10 轮：
  无 cache: 10 × $0.195 = $1.95
  有 cache: $0.225 + 9 × $0.036 = $0.55  （节省 71%）

对话 100 轮：
  无 cache: $19.5
  有 cache: $0.225 + 99 × $0.036 = $3.79 （节省 80%）
```

**结论**：长 prompt + 多轮 = 必打 cache。**单次请求 = 不要打 cache**（白付 25% cache write 没人 read）。

---

## 第七章：Batch API 半价

### 7.1 机制

```
正常请求:  立即返回      input 1x  output 5x
Batch:    24h 内异步    input 0.5x  output 2.5x
```

Batch API 用平台**闲时算力**——AI 公司白天用户高峰、夜间低峰，把不急的请求批量塞进低峰跑。用户拿半价，平台拿利用率。

### 7.2 三家对比

| 维度 | Anthropic | OpenAI | Gemini |
|---|---|---|---|
| 价格 | input/output 各 0.5x | 各 0.5x | 各 0.5x |
| 最大请求数 | 100k / batch | 50k / batch | 自适应 |
| 最大文件 | 256 MB | 200 MB | - |
| 完成 SLA | 24h | 24h | 24h |
| 结果保留 | 29 天 | 29 天 | 30 天 |
| Streaming | ❌ | ❌ | ❌ |
| Cache | ✅ 读取（不能写入） | ❌ | ✅ |

### 7.3 适合 batch 的任务

```
- 数据集打标         50k-100k 条
- 离线评测           1000-10000 条 (规模适中也值得)
- 文档摘要批处理     
- Embedding 后处理（如果用 chat model 而非 embedding model）
- 历史回灌           老数据 + 新模型重新跑
```

### 7.4 不适合 batch

```
- 用户交互（要实时）
- 流式输出
- 24h 内必须出结果但 < 100 条（人工运营更省事）
- 已经能 cache 命中的长 prompt 多轮场景（cache 0.1x 比 batch 0.5x 更狠）
```

### 7.5 Cache + Batch 组合

Anthropic 允许 Batch 请求**读取** prompt cache（不能写入）：

```
1. 正常请求一次（cache write，付 1.25x）
2. 5min/1h 内提交 Batch（命中 cache，input 0.1x 然后 0.5x batch 折扣 = 0.05x）
   实际 batch input 一般 0.5x，cache hit 0.1x 取较小者还是 0.1x？
   API 文档：cache read 价 + batch 折扣 = read 价 × 0.5
```

具体规则各家不同——以官方文档为准。**核心思想**：cache + batch 可以叠加，但叠加规则各有差异。

---

## 第八章：长上下文经济学

### 8.1 三种长上下文策略

```
A. 全部喂进去                简单粗暴，成本最高
B. RAG 检索 + 短上下文        要 embedding 基础设施
C. Prompt caching + 长上下文  最适合"反复访问同样大上下文"
```

### 8.2 三种策略成本对比

假设：50 MB 文档（约 12.5M token），用户问 100 个问题，每个答案 2k token。

#### 策略 A：每次都全文塞

```
每次 input: 12.5M token (Sonnet 4.6 long-ctx 6 USD/M) = $75
每次 output: 2k × 22.5 USD/M = $0.045
100 次: $7504.5
```

不可行——单用户成本 7500 美元。

#### 策略 B：RAG（top-k = 5，每块 1k token）

```
索引一次性成本: 12.5M / 1000 chunk × $0.13/M embedding ≈ $1.6
每次检索: top-5 chunk = 5k token
每次 input: 5k × 3 USD/M = $0.015 (普通 200k 段位)
每次 output: 2k × 15 = $0.03
100 次: 100 × $0.045 + $1.6 = $6.10
```

便宜 1200 倍——但要做 embedding、向量库、retrieval 调优。

#### 策略 C：Prompt caching + 长上下文

```
首次 cache write (1h): 12.5M × 7.5 USD/M = $93.75 (long-ctx 1h write 2x)
后续命中: input 12.5M × 0.6 USD/M = $7.5 (long-ctx read 0.1x = 0.6)
每次 output: 2k × 22.5 = $0.045
100 次: $93.75 + 99 × ($7.5 + 0.045) = $840.95
```

**比 A 省 90%，比 B 贵 130x**——但**不需要 RAG 基础设施**，质量上限高。

### 8.3 策略选择决策树

```
全部内容 ≤ 200k？
├─ 是 → 全塞 + cache（最简单，质量最高）
└─ 否 → 重复访问吗？
   ├─ 是 → 长 ctx + cache（避免 RAG 复杂度）
   └─ 否 → RAG（embedding + 向量库）

文档定期更新？
├─ 是 → RAG（增量重建）
└─ 否 → 长 ctx + cache 1h

用户量大？
├─ 是 → RAG（边际成本最低）
└─ 否 → 长 ctx + cache（开发省事）
```

### 8.4 Context Engineering

LLM 进入 1M 时代后，新的工程话题——**context engineering**——比传统 prompt engineering 更重要：

```
1. 决定哪些信息进 context        （selection）
2. 决定它们的顺序                 （ordering）
3. 决定哪些做 cache                （caching）
4. 决定哪些做 RAG                  （retrieval）
5. 决定哪些做 hierarchical         （summarization / hierarchical retrieval）
```

详细见 **A07 — 精通 RAG 架构**。

---

## 第九章：2026 年价格表（2026.05 数据）

价格单位：USD per 1M tokens。仅供数量级参考，实际以官方为准。

### 9.1 主流模型对照表

| 模型 | input | output | cache write 5m | cache read | batch input | batch output |
|---|---|---|---|---|---|---|
| **Anthropic** | | | | | | |
| Claude Opus 4.7 | 15 | 75 | 18.75 | 1.5 | 7.5 | 37.5 |
| Claude Sonnet 4.6 (≤ 200k) | 3 | 15 | 3.75 | 0.3 | 1.5 | 7.5 |
| Claude Sonnet 4.6 (> 200k) | 6 | 22.5 | 7.5 | 0.6 | 3 | 11.25 |
| Claude Haiku 4.5 | 1 | 5 | 1.25 | 0.1 | 0.5 | 2.5 |
| **OpenAI** | | | | | | |
| GPT-5 | 5 | 25 | 2.5 (cached) | - | 2.5 | 12.5 |
| GPT-5 mini | 1.5 | 8 | 0.75 | - | 0.75 | 4 |
| GPT-5 nano | 0.4 | 2 | 0.2 | - | 0.2 | 1 |
| o3 (reasoning) | 10 | 40 | 5 | - | 5 | 20 |
| **Google** | | | | | | |
| Gemini 2.5 Pro (≤ 200k) | 3.5 | 14 | - | 0.875 | 1.75 | 7 |
| Gemini 2.5 Pro (> 200k) | 7 | 21 | - | 1.75 | 3.5 | 10.5 |
| Gemini 2.5 Flash | 0.3 | 1.2 | - | 0.075 | 0.15 | 0.6 |
| Gemini 2.5 Flash Lite | 0.1 | 0.4 | - | 0.025 | 0.05 | 0.2 |
| **国产** | | | | | | |
| Qwen3 Max | ~1.4 | ~5.6 | - | ~0.14 | ~0.7 | ~2.8 |
| GLM-4.6 | ~0.6 | ~2.2 | - | ~0.15 | ~0.3 | ~1.1 |
| DeepSeek V3.5 | ~0.27 | ~1.1 | ~0.07 (off-peak cache hit) | ~0.07 | ~0.14 | ~0.55 |
| Kimi K2 | ~0.6 | ~2.5 | - | - | - | - |

> 价格表仅为 2026.05 月份的近似数据，量级供成本设计参考。实际计费按官方 pricing 页面为准。

### 9.2 性价比"区间"

```
极便宜    GPT-5 nano / Gemini 2.5 Flash Lite / DeepSeek V3.5  (< 1 USD input)
便宜      Haiku 4.5 / Gemini Flash / GLM-4.6 / Kimi K2          (1-2 USD)
标准      Sonnet 4.6 / Gemini Pro / GPT-5 mini                  (1.5-5 USD)
高端      GPT-5 / Opus 4.7 / o3                                  (5-15 USD)
极致      o3-pro / Opus 4.7 + thinking heavy                     (10+ USD 实际算)
```

价格每年下降 30-50%——2026 比 2024 整体便宜 60-70%。

### 9.3 实际选型经验

```
日常 chatbot：           Sonnet 4.6 默认；Haiku 4.5 兜底
高准代码 / Agent：       Opus 4.7
高吞吐分类 / 抽取：       Haiku 4.5 / Gemini Flash / DeepSeek V3.5
超长文档 (>200k)：       Sonnet 4.6 1M 或 Gemini 2.5 Pro 1M（需 2M 用 Gemini 3.1 Pro）
中文专项：               Qwen3 Max / GLM-4.6 / DeepSeek V3.5
强 reasoning：           o3 / Opus 4.7 (extended thinking)
本地 / 私有部署：         开源 LLaMA 4 / Qwen3 / DeepSeek V3.5
```

---

## 第十章：成本估算公式

### 10.1 公式

单次请求成本：

```
cost = input_tokens × input_price
     + output_tokens × output_price
     + cache_write × cache_write_price
     + cache_read × cache_read_price
```

Batch 折扣：

```
cost_batch = cost × 0.5
```

### 10.2 Go 实现：定价表 + 计算器

```go
package pricing

import (
    "fmt"
    "sync"
)

// 价格单位 USD / 1M tokens
type Pricing struct {
    InputPerM       float64
    OutputPerM      float64
    CacheWrite5mPerM float64
    CacheWrite1hPerM float64
    CacheReadPerM   float64
    LongCtxThreshold int  // 0 = 无分段
    LongInputPerM    float64
    LongOutputPerM   float64
}

var table = map[string]Pricing{
    "claude-opus-4-7": {
        InputPerM: 15, OutputPerM: 75,
        CacheWrite5mPerM: 18.75, CacheWrite1hPerM: 30, CacheReadPerM: 1.5,
    },
    "claude-sonnet-4-6": {
        InputPerM: 3, OutputPerM: 15,
        CacheWrite5mPerM: 3.75, CacheWrite1hPerM: 6, CacheReadPerM: 0.3,
        LongCtxThreshold: 200_000,
        LongInputPerM: 6, LongOutputPerM: 22.5,
    },
    "claude-haiku-4-5": {
        InputPerM: 1, OutputPerM: 5,
        CacheWrite5mPerM: 1.25, CacheWrite1hPerM: 2, CacheReadPerM: 0.1,
    },
    "gpt-5": {
        InputPerM: 5, OutputPerM: 25,
        CacheReadPerM: 2.5, // OpenAI cached input is "0.5x" not "0.1x"
    },
    "gemini-2.5-pro": {
        InputPerM: 3.5, OutputPerM: 14,
        CacheReadPerM: 0.875,
        LongCtxThreshold: 200_000,
        LongInputPerM: 7, LongOutputPerM: 21,
    },
    "deepseek-v3.5": {
        InputPerM: 0.27, OutputPerM: 1.1,
        CacheReadPerM: 0.07,
    },
}

type Usage struct {
    InputTokens         int
    OutputTokens        int
    CacheCreateInputTokens int  // 5m write
    CacheCreateInputTokens1h int  // 1h write
    CacheReadInputTokens int
    IsBatch             bool
    TotalContextLength  int  // 用来决定是否走 long-ctx 价
}

func Cost(model string, u Usage) (float64, error) {
    p, ok := table[model]
    if !ok {
        return 0, fmt.Errorf("unknown model: %s", model)
    }
    
    inputPrice := p.InputPerM
    outputPrice := p.OutputPerM
    if p.LongCtxThreshold > 0 && u.TotalContextLength > p.LongCtxThreshold {
        inputPrice = p.LongInputPerM
        outputPrice = p.LongOutputPerM
    }
    
    c := float64(u.InputTokens)*inputPrice/1e6 +
         float64(u.OutputTokens)*outputPrice/1e6 +
         float64(u.CacheCreateInputTokens)*p.CacheWrite5mPerM/1e6 +
         float64(u.CacheCreateInputTokens1h)*p.CacheWrite1hPerM/1e6 +
         float64(u.CacheReadInputTokens)*p.CacheReadPerM/1e6
    
    if u.IsBatch {
        c *= 0.5
    }
    return c, nil
}

// 累计统计
type Tracker struct {
    mu     sync.Mutex
    Total  float64
    Counts map[string]int
}

func NewTracker() *Tracker {
    return &Tracker{Counts: map[string]int{}}
}

func (t *Tracker) Track(model string, u Usage) error {
    c, err := Cost(model, u)
    if err != nil { return err }
    t.mu.Lock()
    defer t.mu.Unlock()
    t.Total += c
    t.Counts[model]++
    return nil
}
```

### 10.3 预算守门人

```go
type Budget struct {
    Limit     float64       // USD
    Spent     float64
    Reserved  float64       // 已发出未结算
    mu        sync.Mutex
}

func (b *Budget) Reserve(estimate float64) error {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.Spent + b.Reserved + estimate > b.Limit {
        return fmt.Errorf("budget exceeded: spent=%.2f reserved=%.2f estimate=%.2f limit=%.2f",
            b.Spent, b.Reserved, estimate, b.Limit)
    }
    b.Reserved += estimate
    return nil
}

func (b *Budget) Settle(estimate, actual float64) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.Reserved -= estimate
    b.Spent += actual
}

// 用法
estimate, _ := pricing.Cost(model, estimateUsage(prompt))
if err := budget.Reserve(estimate); err != nil {
    return err  // 直接拒
}
resp, err := callLLM(...)
actualCost, _ := pricing.Cost(model, actualUsage(resp))
budget.Settle(estimate, actualCost)
```

### 10.4 估 output token 上限

Input 你能预算；output 你**不能完全控**——但 `max_tokens` 是硬上限。所以**估算时按 `max_tokens` 算 worst case**：

```go
func estimateMaxCost(model string, inputTokens int, maxOutput int) float64 {
    u := Usage{
        InputTokens:  inputTokens,
        OutputTokens: maxOutput, // worst case
        TotalContextLength: inputTokens + maxOutput,
    }
    c, _ := Cost(model, u)
    return c
}
```

设 `max_tokens` 不要乱配——常见错误是默认 8192 但实际任务只要 200 token，预算估算就 40 倍偏高。

---

## 第十一章：生产实践

### 11.1 三层成本控制

```
┌─────────────────────────────────────────┐
│ 1. Per-Request Budget                    │
│    每个请求估算上限，超过预算直接拒        │
├─────────────────────────────────────────┤
│ 2. Per-User / Per-Tenant Quota           │
│    日 / 月 限额，到达后降级或拒绝          │
├─────────────────────────────────────────┤
│ 3. Global Spend Cap                      │
│    应用整体月限额，硬熔断                 │
└─────────────────────────────────────────┘
```

### 11.2 Token Rate Limiter

OpenAI、Anthropic 都同时限 **RPM + TPM**——后者按 token 算。自己实现 token bucket：

```go
package tokenlimit

import (
    "context"
    "sync"
    "time"
)

type TokenBucket struct {
    mu         sync.Mutex
    capacity   int      // 桶容量 = TPM
    available  float64
    refillRate float64  // tokens per second
    lastRefill time.Time
}

func New(tpm int) *TokenBucket {
    return &TokenBucket{
        capacity:   tpm,
        available:  float64(tpm),
        refillRate: float64(tpm) / 60.0,
        lastRefill: time.Now(),
    }
}

func (b *TokenBucket) refill() {
    now := time.Now()
    elapsed := now.Sub(b.lastRefill).Seconds()
    b.available = min(float64(b.capacity), b.available+elapsed*b.refillRate)
    b.lastRefill = now
}

// 等待直到能扣 n 个 token
func (b *TokenBucket) Acquire(ctx context.Context, n int) error {
    for {
        b.mu.Lock()
        b.refill()
        if b.available >= float64(n) {
            b.available -= float64(n)
            b.mu.Unlock()
            return nil
        }
        wait := time.Duration((float64(n)-b.available)/b.refillRate) * time.Second
        b.mu.Unlock()
        
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(wait):
        }
    }
}

func min(a, b float64) float64 { if a < b { return a }; return b }
```

调用前先估 token、`Acquire`、再发请求：

```go
estTokens := estimateTokens(prompt)
if err := bucket.Acquire(ctx, estTokens); err != nil { return err }
resp, err := client.Messages.New(ctx, ...)
```

注意：

- input + output 通常各有独立 bucket（Anthropic 区分得很细）
- 估错代价是 429——所以**保留 buffer**（留 20% margin）

### 11.3 多租户配额

```go
type TenantQuota struct {
    mu      sync.RWMutex
    quotas  map[string]*Quota  // tenant_id → Quota
    limits  QuotaLimits
}

type Quota struct {
    DailySpend   float64
    MonthlySpend float64
    DailyTokens  int
}

type QuotaLimits struct {
    DailySpendLimit   float64
    MonthlySpendLimit float64
    DailyTokenLimit   int
}

func (t *TenantQuota) Check(tenantID string, estCost float64, estTokens int) error {
    t.mu.RLock()
    defer t.mu.RUnlock()
    q := t.quotas[tenantID]
    if q == nil { return nil } // 新租户先放行
    
    if q.DailySpend+estCost > t.limits.DailySpendLimit {
        return ErrDailyQuotaExceeded
    }
    if q.MonthlySpend+estCost > t.limits.MonthlySpendLimit {
        return ErrMonthlyQuotaExceeded
    }
    if q.DailyTokens+estTokens > t.limits.DailyTokenLimit {
        return ErrTokenQuotaExceeded
    }
    return nil
}
```

生产环境配额数据要存 Redis / 数据库——单机内存会丢。

### 11.4 模型自动降级

```go
type ModelTier struct {
    Models []string  // 从贵到便宜
    Index  int       // 当前位置
}

func (t *ModelTier) Current() string { return t.Models[t.Index] }

func (t *ModelTier) Downgrade() bool {
    if t.Index >= len(t.Models)-1 { return false }
    t.Index++
    return true
}

// 用法：429 / 预算紧张 / 配额满 → 降级
tier := &ModelTier{Models: []string{
    "claude-opus-4-7",
    "claude-sonnet-4-6",
    "claude-haiku-4-5",
}}

for attempts := 0; attempts < 3; attempts++ {
    resp, err := callModel(ctx, tier.Current(), prompt)
    if err == nil { return resp, nil }
    
    if isRateLimited(err) || isBudgetExceeded(err) {
        if !tier.Downgrade() { return nil, err }
        continue
    }
    return nil, err
}
```

### 11.5 监控指标

```
core:
  llm_request_tokens_input    counter   (model, tenant)
  llm_request_tokens_output   counter   (model, tenant)
  llm_request_cost_usd        counter   (model, tenant)
  llm_request_latency_ms      histogram (model)
  llm_cache_hit_rate          gauge     (model)
  llm_budget_remaining_pct    gauge     (tenant)
  llm_rate_limit_429_total    counter   (model)
  
derived:
  cost_per_user_request       gauge     = cost / requests
  output_token_ratio          gauge     = output / input
```

`cache_hit_rate` 是 prompt caching 健康度的核心。生产目标 ≥ 50%（长 RAG 任务 80%+）。

### 11.6 异常成本告警

```go
// 简单滑窗
type SpendWindow struct {
    mu       sync.Mutex
    events   []spendEvent
    window   time.Duration
    alertCb  func(rate float64)
    threshold float64  // USD / minute
}

type spendEvent struct {
    at   time.Time
    cost float64
}

func (s *SpendWindow) Add(cost float64) {
    s.mu.Lock()
    defer s.mu.Unlock()
    now := time.Now()
    s.events = append(s.events, spendEvent{now, cost})
    
    // 清理过期
    cutoff := now.Add(-s.window)
    for len(s.events) > 0 && s.events[0].at.Before(cutoff) {
        s.events = s.events[1:]
    }
    
    var sum float64
    for _, e := range s.events { sum += e.cost }
    rate := sum / s.window.Minutes()
    if rate > s.threshold { s.alertCb(rate) }
}
```

### 11.7 离线计费报表

每天凌晨跑：

```sql
-- 假设 llm_call_log 表
SELECT
    tenant_id,
    model,
    DATE(created_at) AS day,
    SUM(input_tokens) AS in_tokens,
    SUM(output_tokens) AS out_tokens,
    SUM(cache_read_tokens) AS cache_read,
    SUM(cost_usd) AS total_cost
FROM llm_call_log
WHERE created_at >= NOW() - INTERVAL 1 DAY
GROUP BY tenant_id, model, day
ORDER BY total_cost DESC;
```

输出 → 邮件 / Slack / Looker。**异常**（单租户 cost > 当日 P95 × 3）触发自动通知。

---

## 第十二章：陷阱清单

### 1. 用 tiktoken 估 Claude token 数不加系数

直接 `tiktoken-go o200k_base.Encode` 来估 Claude → 中文场景**少算 15-25%**。要么打 ×1.18 系数，要么调 Anthropic 的 `count_tokens` API。

### 2. 忘了 long-context 价格分段

```
Sonnet 4.6 ≤ 200k:  3 / 15 USD
Sonnet 4.6 > 200k:  6 / 22.5 USD
```

`total = input + output` 一过 200k 整条请求按贵价算（不是只算超出部分）。预算估算要按上下文长度判断。

### 3. cache write 没人 read 就被丢弃

```go
// 单次请求打 cache_control
resp, _ := client.Messages.New(ctx, params)  
// 5min 过去，没有第二次相同前缀请求 → cache 过期
// 第一次多付 25% cache write 钱白扔
```

打 cache 的前提是**至少 2 次以上访问**。监控 `cache_creation > 0 && cache_read == 0` 比例——这是"白花钱"信号。

### 4. cache 因毫秒变化 miss

system prompt 里塞 `time.Now().Format("...")` → 每秒变 → 永远 miss。把动态内容放在 cached 段**之后**。

### 5. 用 OpenAI 的 cache 价 0.5x 套到 Anthropic

OpenAI cached input = 0.5x；**Anthropic cache read = 0.1x**。混算预算会高估 5 倍。

### 6. Batch 任务用了 streaming 模型

Batch **不支持 streaming**——提交带 `stream: true` 的请求会被拒。Batch 适合"output 一整段拿回来"的任务。

### 7. max_tokens 设过大导致预算估算虚高

```go
params.MaxTokens = anthropic.F(int64(8192))  // 默认抄文档
// 实际任务只要 200 token → worst case 估算虚高 40 倍
// 业务上 budget reserve 也虚高 → 限流误杀
```

按任务实际需要设。分类任务 50；抽取 500；写作 2000-4000。

### 8. 输入 token 估错导致 429

Token bucket 限流时如果估错（少估），消费 token 时实际多扣 → 后续请求 429。**保留 20% margin**：

```go
bucket.Acquire(ctx, int(float64(estimated)*1.2))
```

### 9. 多模态 token 没算

图片 ≈ 1000-1600 token / 张；忘加 → 预算严重低估。Whisper 音频 ≈ 25 token/s。**Vision 与 audio 必须显式估算**。

### 10. 用 chat model 跑 embedding 任务

```go
// 错：用 GPT-5 chat 跑 embedding（让它"输出向量"）
// 对：用 text-embedding-3 / voyage-3 / cohere-embed
```

Embedding 模型便宜 50-100 倍（< 0.1 USD/M token）。

### 11. 没区分 prompt 与 completion 计费

老代码可能把整个 `messages` 数组的 token 当作 input 来估——但 `response.content` 也是 token，按 output 价（5x input）算。Usage 字段必须分别取 `input_tokens` / `output_tokens`。

### 12. 多区域价格不同的坑

某些 model 在 EU / Asia 区域定价不同（合规 / 数据驻留溢价）。生产代码要把"价格表"按 region 维度建。

### 13. Cache 写入 token 也算 input token 配额

Anthropic 的 input token rate limit 同时算 cache write 与普通 input。一次大 prefix 写 cache 会**瞬间吃掉 TPM**——批量任务首请求容易触发。

### 14. 离线 batch 估算用了实时价格

Batch input/output 各 0.5x；记账时如果忘乘 0.5x，账单估算虚高一倍。

---

## 第十三章：2026 现状

### 13.1 价格趋势

```
2023.06   GPT-4: input 30 / output 60
2024.06   GPT-4o: 5 / 15
2025.06   GPT-5 mini: 1.5 / 6
2026.06   nano 模型 < 0.5 USD input
```

**两年价格降 90%**。原因：

- 模型架构优化（MoE、稀疏注意力）
- 推理硬件优化（H100 → B200 → MI400 → Trainium 3）
- 量化技术（FP8 推理普及）
- 竞争压低 margin

2026 年的"标杆"模型（Sonnet 4.6、Gemini 2.5 Pro、GPT-5）价格区间稳定在 3-7 USD input / 12-25 USD output。

### 13.2 Cache 普及

```
2024:  Anthropic 显式 cache_control（首家工程化）
2024:  Gemini context cache（显式）
2025:  OpenAI prompt caching（自动透明，无需控制）
2025:  Gemini implicit caching（自动）
2026:  几乎所有平台都有自动 / 半自动 cache
```

工程师默认应该**对长 prompt 假设有 cache**——剩下的就是命中率监控。

### 13.3 Tokenizer 的"统一化"趋势

```
2023:  各家差 30%（中文）
2026:  各家差 < 10%（除 Claude 仍偏高）
```

主流模型词表向 200k 集中（GPT-4o `o200k_base`、LLaMA 4、Qwen3 都是 ~200k）。中文表征更密集。

### 13.4 Context window 已不是瓶颈

```
2024 痛点：context 不够 → RAG 必须做
2026 现状：1M 已成为高端标配，2M 也可用
```

新瓶颈变成：

- 长 context 延迟（首 token 5-20s）
- 长 context 推理成本（KV cache 显存占用）
- 长 context 质量（lost-in-the-middle 缓解但未消除）

**Context engineering 比 context size 更重要**——选什么进 context、怎么排序、哪些 cache，这些工程问题决定上限。

### 13.5 Batch / 实时混合架构

2026 年成熟的 LLM 后端通常是混合架构：

```
实时通道：用户对话、Agent loop                  → 普通 API + cache
半实时（< 1min）：搜索结果重排、内容审核         → 普通 API + cache
准实时（< 1h）：周期任务、批量分类                → Batch API
异步（< 24h）：评测、训练数据生成                  → Batch API
```

### 13.6 监控的"成本可观测性"

OpenTelemetry GenAI Semantic Convention（2025-01 起 stable）：

```
gen_ai.usage.input_tokens         attribute
gen_ai.usage.output_tokens        attribute
gen_ai.usage.cached_input_tokens  attribute
gen_ai.cost.total_usd             attribute   (custom)
gen_ai.request.model              attribute
```

详见 **A13 — 精通 LLM 可观测性**。

### 13.7 自托管的经济学

```
托管 API (Sonnet 4.6):  3 USD / 1M input
自托管 LLaMA 4 70B:    GPU 4×H100 ≈ $40/h
  推理 ~ 50 token/s × 4 = 200 t/s
  1 小时 720k token = 0.72M token
  $40 / 0.72M = $55/M
```

自托管开源模型**比托管 API 贵 10 倍以上**——除非：

- 隐私 / 合规要求（数据不能出）
- 极端规模（每天处理 100M+ token，自有硬件）
- 极致定制（指定微调权重）

普通项目 2026 年仍优先用 API。

---

## 第十四章：练习题

**练习 1**：估算下面这段中文需要多少 Claude Sonnet 4.6 input token：

> "在 2026 年的 LLM 后端工程实践中，prompt caching 已经成为标配能力——它把长 system prompt 的费用降到 1/10，但要求开发者主动管理 cache 边界与 TTL。"

提示：先用 tiktoken `o200k_base` 估，再乘 1.18 系数。

**练习 2**：你有一份 80 KB 的企业知识库（约 25k token），客服系统每天调用 5000 次问答。每次回答平均 500 token。

- 不打 cache 的日成本？
- 打 5min cache 的日成本（假设 80% 命中率）？
- 打 1h cache 的日成本（假设 95% 命中率）？

用 Sonnet 4.6（≤ 200k 段位）价格表算。

**练习 3**：解释为什么 OpenAI 的 cached input ×0.5x 与 Anthropic 的 cache_read ×0.1x 设计差距这么大。这两种模式背后的"经济鼓励"分别是什么？

**练习 4**：以下 Go 代码估算 token 错在哪？

```go
func estimate(msgs []Message) int {
    enc, _ := tiktoken.GetEncoding("cl100k_base")
    total := 0
    for _, m := range msgs {
        total += len(enc.Encode(m.Content, nil, nil))
    }
    return total
}
```

写出修正版（提示：考虑 role、message 包装 overhead、可能的 system 字段）。

**练习 5**：实现 `EstimateClaudeBatchCost(messages, numRequests, isCached)` 函数，返回 USD 成本。要求：

- 支持 5m / 1h cache（输入 `cacheType` 参数）
- 自动判断 > 200k 走 long-ctx 价
- batch ×0.5x 折扣
- 已知 cache 命中场景下：第一次 write，后续 read

**练习 6**：用户在你的 SaaS 产品里上传了一份 1.5 MB（约 400k token）的 PDF，问 10 个问题。设计三种实现方案的成本对比（Sonnet 4.6）：

- 全部喂进去（每次 400k input）
- RAG（top-5 chunk × 1k token）
- Cache + 长 ctx（首次 cache write 1h，后续 read）

**练习 7**：你做日活 1 万的客服 bot。统计显示 cache 命中率只有 15%，怀疑 prompt 里有动态内容破坏了缓存。给出 3 个可能原因 + 3 个排查方法。

**练习 8**：写 Go middleware，对每个 HTTP 请求：

1. 用 `tiktoken-go` 估 input token
2. 用 token bucket 限流（TPM = 200k）
3. 调 LLM
4. 根据 usage 算实际 cost
5. 写入租户配额账本

**练习 9**：以下 prompt 在 Anthropic Sonnet 4.6 上 cache 命中率为何很低？怎么改？

```go
systemPrompt := fmt.Sprintf(
    "You are a helpful assistant. Current time: %s. User: %s",
    time.Now().Format("2006-01-02 15:04:05"),
    userID,
)
```

**练习 10**：解释 batch + cache 叠加的语义——Anthropic 文档说"batch 可以读 cache 不能写 cache"。这意味着什么样的工程模式？

---

## 参考答案

**练习 1**：

```
原文约 100 个汉字（含标点）。
tiktoken o200k_base：实测约 110 token（包含标点）
Claude 估算：110 × 1.18 ≈ 130 token
也可以调 count_tokens API 拿精确值。
```

**练习 2**：

```
Sonnet 4.6 价格：input 3 USD/M，output 15 USD/M
                cache write 5m 3.75，cache read 0.3
                cache write 1h 6（注：1h 是 2x 不是 1.6x）

(a) 不打 cache：
    每次：25k×3/1M + 500×15/1M = 0.075 + 0.0075 = $0.0825
    日成本：5000 × $0.0825 = $412.5

(b) 5min cache，80% 命中：
    Cache 假设每 5min 内有重复访问；
    每天 5000 次 / 86400s ≈ 0.058 req/s
    实际命中率难达 80%——除非把"common questions"做成共享对话
    假设 80% 命中：
       4000 次命中：4000×(25k×0.3/1M + 0×3/1M + 500×15/1M) = 4000×(0.0075+0.0075) = $60
       1000 次 cache write：1000×(25k×3.75/1M + 500×15/1M) = 1000×(0.09375+0.0075) = $101.25
       总：$161.25
    比无 cache 省 61%

(c) 1h cache，95% 命中：
       4750 次命中：4750×(25k×0.3/1M + 500×15/1M) = $71.25
       250 次 1h write：250×(25k×6/1M + 500×15/1M) = $39.375
       总：$110.6
    比无 cache 省 73%
```

**练习 3**：

- **Anthropic（0.1x）**：极致鼓励长 prompt + 多轮——把"上下文工程"作为产品差异化。降到 0.1x 后开发者愿意把企业知识库整段塞 system prompt。
- **OpenAI（0.5x）**：相对温和鼓励 + 自动透明启用——不让开发者管理 cache 边界，命中规则平台决定。设计哲学是"开箱即用"，对长 prompt 的优化度不如 Anthropic 极致。
- Anthropic 设计鼓励 **"长 system + 多轮"**；OpenAI 设计鼓励 **"高频前缀复用"（chatbot 历史窗口、agent 长对话）**。

**练习 4**：

错误：

1. 用了 cl100k_base（GPT-4，旧版），新模型应该用 o200k_base 或 model 对应 encoding
2. 没算 role / 消息分隔的 overhead（每条 ~3 token + role token）
3. 没算 system 字段（外部传入还是消息里）
4. 没算 tool schema（如果有）

修正：

```go
func estimate(msgs []Message, system string, model string) int {
    enc, _ := tiktoken.EncodingForModel(model)
    total := 0
    if system != "" {
        total += 3 + len(enc.Encode(system, nil, nil))
    }
    for _, m := range msgs {
        total += 3
        total += len(enc.Encode(m.Role, nil, nil))
        total += len(enc.Encode(m.Content, nil, nil))
        if m.Name != "" {
            total += 1 + len(enc.Encode(m.Name, nil, nil))
        }
    }
    total += 3 // assistant 准备 token
    return total
}
```

**练习 5**：

```go
type CacheType int
const (
    NoCache CacheType = iota
    Cache5m
    Cache1h
)

func EstimateClaudeBatchCost(
    inputTokens, outputTokens, numRequests int,
    cacheType CacheType,
    isBatch bool,
) float64 {
    var inputPrice, outputPrice float64
    longCtx := (inputTokens + outputTokens) > 200_000
    if longCtx {
        inputPrice, outputPrice = 6, 22.5
    } else {
        inputPrice, outputPrice = 3, 15
    }
    
    if cacheType == NoCache {
        total := float64(numRequests) * (
            float64(inputTokens)*inputPrice/1e6 +
            float64(outputTokens)*outputPrice/1e6,
        )
        if isBatch { total *= 0.5 }
        return total
    }
    
    // 第一次 write，后续 read
    writePrice := 3.75
    readPrice := 0.3
    if longCtx {
        writePrice = 7.5
        readPrice = 0.6
    }
    if cacheType == Cache1h {
        writePrice *= 2 / 1.25 // 1h = 2x base，5m = 1.25x base
    }
    
    firstCost := float64(inputTokens)*writePrice/1e6 + float64(outputTokens)*outputPrice/1e6
    subsequentCost := float64(inputTokens)*readPrice/1e6 + float64(outputTokens)*outputPrice/1e6
    total := firstCost + float64(numRequests-1)*subsequentCost
    
    if isBatch { total *= 0.5 }
    return total
}
```

**练习 6**：

```
400k token 已超 200k → long-ctx 价 (input 6, output 22.5)

(a) 全塞每次：
    10 × (400k×6/1M + 1k×22.5/1M) = 10 × (2.4 + 0.0225) = $24.22

(b) RAG (top-5 × 1k = 5k input)：
    Embedding 一次：400 chunk × 0.13 USD/M ≈ $0.05 (一次性)
    10 × (5k×3/1M + 1k×15/1M) = 10 × (0.015 + 0.015) = $0.30
    总：$0.35  (200k 段位)

(c) Cache 1h + 长 ctx：
    首次 write: 400k×12/1M + 1k×22.5/1M = $4.82 (long-ctx 1h write 2x = 12)
    9 次 read: 9 × (400k×0.6/1M + 1k×22.5/1M) = 9 × $0.2625 = $2.36
    总：$7.18

比例：RAG 最便宜，cache+long 中等，全塞最贵。
但 (a)(c) 工程复杂度低、不需要 embedding 基础设施；
RAG 需要切 chunk + 索引 + retrieval 调优——单次成本不是全部成本。
```

**练习 7**：

可能原因：

1. system prompt 里有 `time.Now()` / 日期 / nonce
2. 用户 ID / session ID 拼进了 system prompt 顶部
3. 每次调用 system prompt 微调（A/B 实验）
4. Prefix 不到 1024 token（阈值不够）
5. 多区域 endpoint（cache 不跨区共享）

排查方法：

1. 抓两次连续相同问题的 raw request，diff system + history 部分——找出不一致字段
2. 加日志：每次输出 system prompt 的 SHA256 前 8 字节，看是否真稳定
3. 测试：构造一个固定 1.5k system + 同样 user，发 2 次，看第二次 `cache_read_input_tokens` 是否 > 0；若是 0 → 写入失败 / prefix 不稳定
4. 用 `count_tokens` 确认 prefix ≥ 1024
5. 把动态字段集中放在 `messages` 最后一条 user message 里，其余统一稳定

**练习 8**：

```go
type LLMMiddleware struct {
    Limiter  *TokenBucket
    Tracker  *Tracker
    Quota    *TenantQuota
    Pricing  map[string]Pricing
    Client   *anthropic.Client
}

func (m *LLMMiddleware) Handle(w http.ResponseWriter, r *http.Request) {
    var req ChatRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // 1. 估 token
    enc, _ := tiktoken.GetEncoding("o200k_base")
    est := len(enc.Encode(req.Prompt, nil, nil))
    est = int(float64(est) * 1.2) // 20% buffer
    
    // 2. 限流
    ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
    defer cancel()
    if err := m.Limiter.Acquire(ctx, est); err != nil {
        http.Error(w, "rate limited", 429)
        return
    }
    
    // 3. 配额预检
    tenant := r.Header.Get("X-Tenant-ID")
    estCost, _ := Cost(req.Model, Usage{InputTokens: est, OutputTokens: req.MaxTokens})
    if err := m.Quota.Check(tenant, estCost, est); err != nil {
        http.Error(w, "quota exceeded", 429)
        return
    }
    
    // 4. 调 LLM
    resp, err := m.Client.Messages.New(ctx, buildParams(req))
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    
    // 5. 实际成本入账
    actualCost, _ := Cost(req.Model, Usage{
        InputTokens:  int(resp.Usage.InputTokens),
        OutputTokens: int(resp.Usage.OutputTokens),
        CacheCreateInputTokens: int(resp.Usage.CacheCreationInputTokens),
        CacheReadInputTokens:   int(resp.Usage.CacheReadInputTokens),
    })
    m.Tracker.Track(req.Model, ...)
    m.Quota.Settle(tenant, estCost, actualCost)
    
    json.NewEncoder(w).Encode(resp)
}
```

**练习 9**：

```go
// 错版
systemPrompt := fmt.Sprintf(
    "You are a helpful assistant. Current time: %s. User: %s",
    time.Now().Format("2006-01-02 15:04:05"),
    userID,
)
```

问题：

- `time.Now()` 每秒变 → system prefix 永远不同 → 永远 miss
- `userID` 每个用户不同 → 跨用户无法共享 prefix（但 Anthropic cache 本来就是按客户 hash，所以单用户多轮还是会命中）；但其他用户用同一 prompt 模板时拿不到共享 cache

改法：

```go
// 把动态部分放在第一条 user message
systemPrompt := "You are a helpful assistant."  // 保持稳定 → 命中 cache（如果 ≥ 1024 token）

// 实际用法
messages := []Message{
    {Role: "user", Content: fmt.Sprintf("[meta] time=%s user=%s\n[question] %s", time.Now(), userID, question)},
}

// 或者使用 cache breakpoint 分段
system := []TextBlock{
    {Text: "<稳定通用 instructions, 长度 > 1024 token>", CacheControl: ephemeral},
    {Text: fmt.Sprintf("当前时间: %s 用户ID: %s", time.Now(), userID)},  // 不缓存
}
```

**练习 10**：

含义：

- **不能写**：batch 请求的输入即使打了 `cache_control` 也不会创建新 cache（避免 batch 大量请求互相 cache write，平台层面禁掉）
- **可以读**：如果之前正常 API 请求在 5m / 1h 内写了 cache，batch 请求带相同 prefix 可以命中（计费按 cache_read × 0.5 batch 折扣）

工程模式：

```
1. 用一个"实时" API 调用先写 cache：构造一份"标准 prompt"，调一次正常 API
2. 立即提交 N 个 batch 请求，prefix 与上一步完全一致
3. Batch 命中 cache_read（input 极便宜）+ 0.5x batch 折扣 = 0.05x 真实成本
4. 适合：先用样本 prompt 触发 cache，再用 batch 跑数据集
```

实际使用要测——具体规则随 API 变化，以官方为准。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Token 算法 | BPE / SentencePiece / WordPiece；GPT/Claude 用 BPE，Gemini/国产多用 SentencePiece |
| 切分单位 | 英文 1 token ≈ 4 字符；中文 1 token ≈ 1-2 字 |
| Tokenizer | tiktoken / 内部 / SentencePiece；Claude 内部未开源 |
| 估算工具 | `tiktoken-go` / Anthropic count_tokens API |
| Context window | 200k 默认 / 1M-2M 高端；分段计费 |
| 输入输出价差 | output 是 input 的 3-5 倍 |
| Cache write | 1.25x（5m）/ 2x（1h） |
| Cache read | 0.1x（Anthropic）/ 0.25x（Gemini）/ 0.5x（OpenAI） |
| Batch | 0.5x，24h，最多 100k 请求 |
| 多模态 | 图片 1000-1600 / 张；音频 25 token/s；视频极贵 |
| 成本估算 | input_tokens × input_price + output_tokens × output_price + cache 各项 |
| 预算控制 | per-request / per-tenant / global 三层 |
| 模型降级 | Opus → Sonnet → Haiku 兜底 |
| Cache 监控 | 命中率 ≥ 50% 是健康线 |
| 2026 趋势 | 价格降 / cache 普及 / 1M context 标配 / context engineering |

铁律：

- 永远监控 `input / output / cache_read / cache_write` 四个 token 维度
- 长 prompt 一定打 cache_control + 监控命中率
- 离线批量永远走 batch
- 用 long-ctx 模型前先估单次成本
- 多模态 token 必须显式估算
- 模型按场景下沉——别用 Opus 跑分类
- Token bucket 限流时保留 20% margin
- 预算守门人放在 LLM 调用入口

下一篇 **A04 — 精通 Prompt 工程** 将拆开 few-shot / CoT / structured output / 评测体系——把 prompt 从"凭感觉写"变成可工程化、可版本化、可评测的资产。

---
