# 精通 Claude API 工程化：SDK、Prompt Caching、Batch、Files 与 Extended Thinking

> 课程编号：A01
> 路线图来源：AI / LLM 后端工程 · 模块一 API 基础
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 5 月

---

## 引言：从"会调 API"到"工程化 API"

```go
client := anthropic.NewClient()
msg, _ := client.Messages.New(ctx, anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeOpus4_7,
    MaxTokens: 1024,
    Messages: []anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("Hello, Claude")),
    },
})
fmt.Println(msg.Content[0].Text)
```

五行代码做 demo——这是 Claude API 的"招牌"。但生产环境的 Claude 工程化远不止于此：prompt caching 命中率、batch 半价、streaming 断点重连、extended thinking 预算、tool use 多轮、citations、files API、retry / rate limit headers、成本控制。本章把生产级 **Claude API** 拆开。

2026 年 5 月，Anthropic 的主力模型版图：

| 模型 ID | 别名 | 角色 |
|---|---|---|
| `claude-opus-4-7` | Opus 4.7 | 顶配——复杂推理、长任务 Agent、代码生成 |
| `claude-sonnet-4-6` | Sonnet 4.6（1M ctx beta） | 通用主力——200k 默认 / 1M context beta |
| `claude-haiku-4-5-20251001` | Haiku 4.5 | 高吞吐 / 低延迟——分类、抽取、轻 RAG |

它们共享同一套 Messages API；不同模型只是改 `model` 字段。这是 Anthropic API 一直以来的设计：**一个 Messages API 走天下**。

---

## 第一章：SDK 选型——官方 anthropic-sdk-go vs 直接 HTTP

### 1.1 三条路

```
直接 HTTP (net/http + JSON)
        ↓
anthropic-sdk-go (官方 Go SDK, 2024-06 起官方维护)
        ↓
LangChain Go / LangChainGo (高层封装，少推荐)
```

**默认选择**：`github.com/anthropics/anthropic-sdk-go`。官方 SDK 维护跟随 API 更新最快——extended thinking、prompt caching、batch、citations、memory tool 等新特性出现后通常几天内有 release。

```bash
go get github.com/anthropics/anthropic-sdk-go
```

### 1.2 何时直接 HTTP

- 极端体积敏感（嵌入式 / 边缘）
- 需要自己控制连接池、超时、重试细节
- 测试代理 / 网关——想看到原始 JSON
- 对接 Bedrock / Vertex 等托管 endpoint（SDK 也支持，但有 quirk）

直接 HTTP 时记住几个 endpoint：

```
POST https://api.anthropic.com/v1/messages          # 主接口
POST https://api.anthropic.com/v1/messages/batches  # batch 接口
POST https://api.anthropic.com/v1/files             # files API
POST https://api.anthropic.com/v1/messages/count_tokens  # 算 token
```

请求头三件套：

```
x-api-key: sk-ant-...
anthropic-version: 2024-06-01
content-type: application/json
```

可选 `anthropic-beta` 启用 beta 特性，如 `prompt-caching-2024-07-31`（早期）/ `extended-cache-ttl-2025-04-11`、`message-batches-2024-09-24`、`extended-thinking-2025-XX` 等——具体 beta header 查官方 release notes。**2026 年大部分特性已 GA**，beta header 仅用于最新预览功能。

### 1.3 客户端初始化

```go
import "github.com/anthropics/anthropic-sdk-go"
import "github.com/anthropics/anthropic-sdk-go/option"

client := anthropic.NewClient(
    option.WithAPIKey(os.Getenv("ANTHROPIC_API_KEY")),
    option.WithMaxRetries(2),                 // 内部自动重试 429/5xx
    option.WithRequestTimeout(120*time.Second),
)
```

`option.WithAPIKey` 不传时自动读 `ANTHROPIC_API_KEY` 环境变量。SDK 内置重试基于 `Retry-After` header；线上**强烈建议**保留 `WithMaxRetries(2-3)`。

### 1.4 同步 / 异步 / 流式三态

```go
// 1. 同步——一次拿完整响应
resp, err := client.Messages.New(ctx, params)

// 2. 流式——SSE，逐 token
stream := client.Messages.NewStreaming(ctx, params)
for stream.Next() {
    event := stream.Current()
    // ...
}
stream.Err()

// 3. Batch——24h 异步，input 半价
batch, _ := client.Messages.Batches.New(ctx, batchParams)
```

三个 API 对应三种用法。下文逐一拆开。

---

## 第二章：Messages API 详解

### 2.1 核心 schema

```json
{
  "model": "claude-opus-4-7",
  "max_tokens": 1024,
  "system": "You are a helpful assistant.",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."},
    {"role": "user", "content": [
        {"type": "text", "text": "..."},
        {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}
    ]}
  ],
  "temperature": 1.0,
  "stop_sequences": ["\n\nHuman:"],
  "tools": [...]
}
```

要点：

- `system` 是**顶层字段**（不在 messages 里），可以是字符串或带 `cache_control` 的对象数组
- `messages` 必须以 `user` 开头、`assistant` 与 `user` 交替（如违反 API 报 400）
- `content` 既可以是单字符串、也可以是数组（含 text / image / tool_use / tool_result 等 block）
- `max_tokens` **必填**——Anthropic 不给默认值

### 2.2 五种 content block 类型

```go
type ContentBlock interface{ isContentBlock() }

// 1. text
{type: "text", text: "..."}

// 2. image
{type: "image", source: {type: "base64"|"url", media_type, data}}

// 3. tool_use (assistant 生成)
{type: "tool_use", id: "toolu_01abc", name: "get_weather", input: {...}}

// 4. tool_result (user 回填)
{type: "tool_result", tool_use_id: "toolu_01abc", content: "..."}

// 5. document (PDF / 文档块)
{type: "document", source: {type: "base64"|"url"|"file", ...}}
```

`thinking` block（extended thinking）是 assistant 单独的一类——见第八章。

### 2.3 Go SDK 的 block 构造器

```go
import "github.com/anthropics/anthropic-sdk-go"

// 文本 user message
anthropic.NewUserMessage(anthropic.NewTextBlock("explain RAG"))

// 多模态
anthropic.NewUserMessage(
    anthropic.NewTextBlock("describe this image"),
    anthropic.NewImageBlockBase64("image/png", base64Data),
)

// assistant 历史
anthropic.NewAssistantMessage(anthropic.NewTextBlock("..."))

// tool_result 回填
anthropic.NewUserMessage(anthropic.NewToolResultBlock(
    "toolu_01abc", `{"temp": 22}`, false, // is_error
))
```

### 2.4 多轮对话

```go
messages := []anthropic.MessageParam{
    anthropic.NewUserMessage(anthropic.NewTextBlock("hi")),
    anthropic.NewAssistantMessage(anthropic.NewTextBlock("hello!")),
    anthropic.NewUserMessage(anthropic.NewTextBlock("explain CRDTs")),
}
resp, err := client.Messages.New(ctx, anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeSonnet4_6,
    MaxTokens: 2048,
    Messages:  messages,
})
```

**API 是无状态的**——你必须把整个 history 每轮发回（这与 OpenAI Assistants 不同）。后端必须自己持久化对话。

### 2.5 响应结构

```go
type Message struct {
    ID           string                 // msg_01abc
    Type         string                 // "message"
    Role         string                 // "assistant"
    Content      []ContentBlock         // 一个或多个 block
    Model        string
    StopReason   string                 // end_turn | max_tokens | stop_sequence | tool_use
    StopSequence string
    Usage        Usage
}

type Usage struct {
    InputTokens              int
    OutputTokens             int
    CacheCreationInputTokens int  // 写入 cache 的 token 数（计费 1.25x）
    CacheReadInputTokens     int  // 命中 cache 的 token 数（计费 0.1x）
}
```

`StopReason` 是 prod 监控的关键指标：

| 值 | 含义 | 处理 |
|---|---|---|
| `end_turn` | 模型自然结束 | 正常 |
| `max_tokens` | 截断 | 提高 max_tokens 或分段 |
| `stop_sequence` | 命中 stop | 检查业务逻辑 |
| `tool_use` | 要调 tool | 进入 tool 循环 |
| `pause_turn` | 长任务暂停（continue 接续） | 继续 turn |
| `refusal` | 安全拒绝 | 记录、不重试 |

---

## 第三章：Prompt Caching——把长上下文变成可负担

### 3.1 原理

```
请求 1：[system: 50KB legal corpus] [user: question 1]   → 完整计费
请求 2（5min 内）：[system: 同样 50KB] [user: question 2] → system 部分 0.1x 计费！
```

Anthropic 把 prompt 的前缀做内部 cache（基于内容 hash + 模型 + 客户）。命中时**输入 token 价格降到 1/10**，第一次写入贵 25%（cache creation 1.25x）。

**收益场景**：

- 一份长 system prompt 多轮调用
- 长文档 + 多次问询
- Agent 长 turn——历史 turn 缓存
- Tool 定义稳定 → tool schema 缓存

### 3.2 cache_control 标记

```json
{
  "system": [
    {"type": "text", "text": "通用 instructions..."},
    {
      "type": "text",
      "text": "[巨长企业知识库 80k tokens]",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [...]
}
```

`ephemeral` 是目前唯一类型。**最多 4 个 cache breakpoint**（system 任意位置、tools、messages、tool_results 中）。

Go SDK：

```go
import "github.com/anthropics/anthropic-sdk-go"

system := []anthropic.TextBlockParam{
    {Text: anthropic.String("常规 instructions"), Type: anthropic.F("text")},
    {
        Text: anthropic.String(longCorpus),
        Type: anthropic.F("text"),
        CacheControl: anthropic.F(anthropic.CacheControlEphemeralParam{
            Type: anthropic.F("ephemeral"),
            // TTL: anthropic.F("1h"),  // 默认 5min；需要 1h 时设这个
        }),
    },
}
```

### 3.3 命中规则

**前缀完全匹配**才命中。也就是说：

```
req 1: [A][B][C]          ← 写入
req 2: [A][B][C][D]       ← 前 [A][B][C] 命中
req 3: [A][X][C][D]       ← B != X，全部 miss
req 4: [A][B][C][E]       ← 前 [A][B][C] 命中
```

**任何字节级修改都会破坏缓存**——包括日期 / 时间戳 / 随机 nonce 这些"看似无关"的字段。

**最小缓存长度**：

- Opus / Sonnet：≥ 1024 tokens
- Haiku：≥ 2048 tokens

低于阈值的 prefix 不会被缓存（哪怕你打了 cache_control 也无效）。

### 3.4 TTL：5min vs 1h

- **5 min**（默认）：每次命中都"刷新"5min；持续访问可一直续命
- **1 h**（beta `extended-cache-ttl-2025-04-11`）：cache_control 加 `"ttl": "1h"` 字段；价格略贵但适合 1h 内重复访问

```go
CacheControl: anthropic.F(anthropic.CacheControlEphemeralParam{
    Type: anthropic.F("ephemeral"),
    TTL:  anthropic.F("1h"),
})
```

**1h 适合**：周期性 RAG、企业知识库长会话、Agent 长任务。

### 3.5 计费模型

| 操作 | 单价（相对 base input） |
|---|---|
| 普通 input | 1x |
| Cache write（5min） | 1.25x |
| Cache write（1h） | 2.0x |
| Cache read | 0.1x |
| Output | 5x（与原价相同） |

**经济判断**：cache 在**至少 2 次以上的访问**才划算。一次性请求不要打 cache_control。

### 3.6 监控命中率

```go
fmt.Printf(
    "input=%d cache_create=%d cache_read=%d output=%d\n",
    resp.Usage.InputTokens,
    resp.Usage.CacheCreationInputTokens,
    resp.Usage.CacheReadInputTokens,
    resp.Usage.OutputTokens,
)
```

生产建议：把这四个值打成 metric。**目标命中率 ≥ 50%**（长任务 Agent 通常 80%+）。

---

## 第四章：Streaming——SSE 与 Go 实战

### 4.1 SSE 事件序列

```
event: message_start
data: {"type":"message_start","message":{"id":"msg_01","role":"assistant",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":" world"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"output_tokens":12}}

event: message_stop
data: {"type":"message_stop"}

event: ping
data: {"type":"ping"}
```

### 4.2 关键事件类型

| 事件 | 含义 |
|---|---|
| `message_start` | 开始；含 message metadata 与 usage 初始值（input tokens） |
| `content_block_start` | 新 block 开始；含 index 与 block 类型（text / tool_use / thinking） |
| `content_block_delta` | block 内增量；text_delta / input_json_delta / thinking_delta 等 |
| `content_block_stop` | block 结束 |
| `message_delta` | message 级增量；含最终 stop_reason 与 output_tokens |
| `message_stop` | 整 message 结束 |
| `ping` | keep-alive |
| `error` | 中途错误（如 `overloaded_error`） |

### 4.3 Go SDK streaming

```go
stream := client.Messages.NewStreaming(ctx, anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeSonnet4_6,
    MaxTokens: 1024,
    Messages:  []anthropic.MessageParam{anthropic.NewUserMessage(anthropic.NewTextBlock("写一首关于秋天的诗"))},
})

var fullText strings.Builder
for stream.Next() {
    event := stream.Current()
    switch evt := event.AsAny().(type) {
    case anthropic.ContentBlockDeltaEvent:
        if td, ok := evt.Delta.AsAny().(anthropic.TextDelta); ok {
            fmt.Print(td.Text)
            fullText.WriteString(td.Text)
        }
    case anthropic.MessageDeltaEvent:
        if evt.Delta.StopReason != "" {
            log.Printf("stop_reason=%s output_tokens=%d", evt.Delta.StopReason, evt.Usage.OutputTokens)
        }
    case anthropic.MessageStopEvent:
        // 结束
    }
}
if err := stream.Err(); err != nil {
    log.Printf("stream error: %v", err)
}
```

SDK 把 SSE 解析为强类型事件。**永远检查 `stream.Err()`**——中途 `error` 事件会导致流提前结束。

### 4.4 转发 SSE 给前端

```go
func chatHandler(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok { http.Error(w, "no streaming", 500); return }
    
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("X-Accel-Buffering", "no")  // nginx 关代理 buffer
    
    stream := client.Messages.NewStreaming(r.Context(), params)
    for stream.Next() {
        event := stream.Current()
        // 直接转发 SSE 事件 - 注意要换成前端能消费的协议
        data, _ := json.Marshal(event)
        fmt.Fprintf(w, "event: %s\ndata: %s\n\n", event.Type, data)
        flusher.Flush()
    }
}
```

要点：

- 设 `X-Accel-Buffering: no`——防止 nginx/CDN 缓存
- 每个 event 后 `flusher.Flush()` 立即下发
- ctx cancel 时 stream 自动 close（client 断开就停止计费 output）
- Cloudflare 等 CDN 也常有 buffer——必要时在响应头加 `Cache-Control: no-cache, no-transform`

详细 SSE 工程化见 **A12 — 精通流式输出与 SSE**。

---

## 第五章：Batch API——24h 异步，input 半价

### 5.1 适用场景

```
- 离线评测（跑 1000 道题）
- 数据集打标
- 文档批处理
- 历史数据 reprocess
- 嵌入索引构建（如果要走 generation 而非 embedding）
```

**核心收益**：input + output 都打**对折**。代价：最长 24h 才出结果（实际平均 ~1h）。

### 5.2 流程

```
1. 准备 N 个独立请求（每个有 custom_id）
2. POST /v1/messages/batches            ← 上传整个 batch
3. 轮询 batch 状态                       ← in_progress / canceling / ended
4. GET .../results                       ← 下载 JSONL 结果
5. 按 custom_id 关联回业务
```

### 5.3 Go 示例

```go
batchParams := anthropic.BetaMessageBatchNewParams{
    Requests: []anthropic.BetaMessageBatchNewParamsRequest{
        {
            CustomID: anthropic.F("review-doc-001"),
            Params: anthropic.F(anthropic.BetaMessageBatchNewParamsRequestParams{
                Model:     anthropic.F(anthropic.ModelClaudeSonnet4_6),
                MaxTokens: anthropic.F(int64(1024)),
                Messages:  anthropic.F([]anthropic.BetaMessageParam{...}),
            }),
        },
        // ... 可以一直加到 100,000 个请求 / 256 MB
    },
}
batch, err := client.Beta.Messages.Batches.New(ctx, batchParams)

// 轮询
for {
    cur, _ := client.Beta.Messages.Batches.Get(ctx, batch.ID)
    if cur.ProcessingStatus == "ended" { break }
    time.Sleep(60 * time.Second)
}

// 拉结果（JSONL stream）
results, _ := client.Beta.Messages.Batches.Results(ctx, batch.ID)
defer results.Close()
scanner := bufio.NewScanner(results)
for scanner.Scan() {
    line := scanner.Bytes()
    var r BatchResult
    json.Unmarshal(line, &r)
    if r.Result.Type == "succeeded" {
        process(r.CustomID, r.Result.Message)
    } else {
        log.Printf("%s failed: %v", r.CustomID, r.Result.Error)
    }
}
```

### 5.4 限制

- 单 batch 最多 **100,000** 请求 / **256 MB**
- 单个请求依然走正常 token limit
- 结果保留 **29 天**
- batch 不能用 streaming（结果整体返回）
- **不支持** prompt caching 写入（但**可以读取**——如果你之前在 5min/1h TTL 内写入了 cache，batch 请求可以命中）

### 5.5 何时不用 batch

- 用户对话（要实时）
- 想要 streaming
- 要在 1 小时内出结果但**已经**有 prompt caching 可用——caching 折扣（0.1x）比 batch（0.5x）更狠
- 批量 < 100 时——人工运营更省事

---

## 第六章：Files API——上传 PDF / 图片

### 6.1 用途

```
- PDF / 文档：放 system message 或 user content
- 图片：vision 输入
- batch 输入文件（极大 batch 时）
```

历史上要把文件 base64 内联到请求里——大文件 → 请求体爆。Files API 让你**预上传**，请求里只引用 `file_id`。

### 6.2 上传

```go
file, err := os.Open("contract.pdf")
defer file.Close()
uploaded, _ := client.Beta.Files.Upload(ctx, anthropic.BetaFileUploadParams{
    File: anthropic.F[io.Reader](file),
})
// uploaded.ID 是 file_xxx
```

或 curl：

```bash
curl https://api.anthropic.com/v1/files \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2024-06-01" \
  -F "file=@contract.pdf"
```

### 6.3 引用

```json
{
  "role": "user",
  "content": [
    {
      "type": "document",
      "source": {"type": "file", "file_id": "file_abc123"}
    },
    {"type": "text", "text": "总结这份合同"}
  ]
}
```

PDF 内容会被模型 OCR + 解析；扫描件也能识别。

### 6.4 限制

- 单文件最大 **32 MB**（PDF 32MB / 图片 5MB 这种细节看具体类型）
- 账号总配额——超出要清理
- 文件保留时间——长期任务要主动续期或重传

---

## 第七章：Tool Use——结构化调用

### 7.1 流程

```
1. 客户端定义 tools（JSON Schema）
2. 调 Messages API，模型返回 stop_reason: tool_use
3. 客户端执行 tool，回填 tool_result
4. 再调 Messages API（带历史），模型用 tool_result 继续
5. 循环直到 stop_reason: end_turn
```

### 7.2 定义 tool

```go
tools := []anthropic.ToolParam{
    {
        Name:        anthropic.F("get_weather"),
        Description: anthropic.F("Get current weather for a city"),
        InputSchema: anthropic.F(anthropic.ToolInputSchemaParam{
            Type: anthropic.F("object"),
            Properties: anthropic.F(map[string]any{
                "city":  map[string]any{"type": "string"},
                "units": map[string]any{"type": "string", "enum": []string{"c", "f"}},
            }),
            Required: anthropic.F([]string{"city"}),
        }),
    },
}
```

### 7.3 多轮 tool 循环

```go
messages := []anthropic.MessageParam{
    anthropic.NewUserMessage(anthropic.NewTextBlock("纽约现在多少度")),
}
for {
    resp, _ := client.Messages.New(ctx, anthropic.MessageNewParams{
        Model:     anthropic.ModelClaudeSonnet4_6,
        MaxTokens: anthropic.F(int64(1024)),
        Tools:     anthropic.F(tools),
        Messages:  anthropic.F(messages),
    })
    if resp.StopReason != "tool_use" {
        // 终止
        fmt.Println(extractText(resp))
        break
    }
    // 收集 assistant 回应（包含 tool_use blocks）
    messages = append(messages, anthropic.NewAssistantMessage(resp.Content...))
    
    var toolResults []anthropic.ContentBlockParamUnion
    for _, block := range resp.Content {
        if tu, ok := block.AsAny().(anthropic.ToolUseBlock); ok {
            result := executeTool(tu.Name, tu.Input) // 你的实现
            toolResults = append(toolResults, anthropic.NewToolResultBlock(tu.ID, result, false))
        }
    }
    messages = append(messages, anthropic.NewUserMessage(toolResults...))
}
```

### 7.4 parallel tool use

模型一次可以返回**多个** tool_use block。前面循环已经处理了——遍历 `resp.Content` 收集全部，并行执行，最后整体回填一个 user message。

详细 tool use 工程化见 **A08 — 精通 Tool Use**。

---

## 第八章：Extended Thinking——可见思考链

### 8.1 是什么

Claude 4.x 引入的"长思考"能力：模型在生成正式回复前先输出一段 `thinking` 内容（可见或加密），用 token 换更深推理。

```
用户问题 → [thinking: 数百到数千 token 推理] → [text: 最终答案]
```

### 8.2 启用

```go
resp, _ := client.Messages.New(ctx, anthropic.MessageNewParams{
    Model:     anthropic.ModelClaudeOpus4_7,
    MaxTokens: anthropic.F(int64(16000)),
    Thinking: anthropic.F(anthropic.ThinkingConfigEnabledParam{
        Type:         anthropic.F("enabled"),
        BudgetTokens: anthropic.F(int64(10000)),  // thinking 上限
    }),
    Messages: anthropic.F([]anthropic.MessageParam{
        anthropic.NewUserMessage(anthropic.NewTextBlock("证明费马小定理")),
    }),
})

for _, block := range resp.Content {
    switch b := block.AsAny().(type) {
    case anthropic.ThinkingBlock:
        log.Printf("[thinking] %s", b.Thinking) // 可见思考链
    case anthropic.TextBlock:
        fmt.Println(b.Text)
    }
}
```

`BudgetTokens` 限制 thinking 的 token 量；模型可能少用。

### 8.3 计费

thinking tokens **按 output 价计费**——和正常输出一样。Budget 设高就是花钱换准确率。

### 8.4 何时用

- 数学 / 形式推理
- 代码逻辑分析
- 复杂规划（Agent 任务分解）
- Eval / debug 时想看模型"心路"

**不要**对简单分类、抽取打开 thinking——浪费钱。

### 8.5 thinking + tool use

extended thinking 可以和 tool use 同时启用。模型先 thinking，再决定调 tool，再 thinking，再回复。这是 Agent 长任务的核心模式。

### 8.6 加密 thinking

若不想让 thinking 暴露给客户端（防 prompt 泄露），用 `thinking.type = "redacted"`——返回加密 block，下次请求可以原样传回作为上下文，但不可见明文。

---

## 第九章：Vision、Citations 与 Stop Sequences

### 9.1 Vision

```go
imageData, _ := base64.StdEncoding.DecodeString(...)
msg := anthropic.NewUserMessage(
    anthropic.NewImageBlockBase64("image/jpeg", base64.StdEncoding.EncodeToString(imageData)),
    anthropic.NewTextBlock("这张图里有多少只猫？"),
)
```

支持 PNG / JPEG / WebP / GIF。单张图≤ 5 MB；建议长边 ≤ 1568 px（再大模型也会压缩）。

URL 形式：

```json
{"type": "image", "source": {"type": "url", "url": "https://..."}}
```

### 9.2 Citations

让模型在答案中**自动标注**来源段落（用于 RAG）：

```json
{
  "role": "user",
  "content": [
    {
      "type": "document",
      "source": {"type": "text", "media_type": "text/plain", "data": "[长文]"},
      "citations": {"enabled": true}
    },
    {"type": "text", "text": "Acme 公司 2024 营收多少？"}
  ]
}
```

响应中 text block 会带 `citations` 数组，标出哪段文本来自原文哪个位置：

```json
{
  "type": "text",
  "text": "Acme 2024 营收 12 亿美元",
  "citations": [{"type": "char_location", "document_index": 0, "start_char_index": 1234, "end_char_index": 1280}]
}
```

适合需要"可追溯"的法律 / 学术 / 企业 RAG 场景。详见 **A07 — 精通 RAG 架构**。

### 9.3 Stop Sequences

```go
StopSequences: anthropic.F([]string{"END_OF_RESPONSE", "\n\nHuman:"}),
```

模型遇到这些字符串前缀就停。常见用法：

- 强制 JSON 输出后停（"}" 之后停止）
- 防越权回应（"Human:" 出现时停，避免模型自演自动剧本）

`StopReason == "stop_sequence"` 时 `StopSequence` 字段告诉你命中了哪一个。

---

## 第十章：错误处理、重试与限流

### 10.1 错误码

| HTTP | type | 含义 | 处理 |
|---|---|---|---|
| 400 | `invalid_request_error` | 请求格式错 | 不重试，修代码 |
| 401 | `authentication_error` | key 无效 | 不重试，运维 |
| 403 | `permission_error` | 没权限调该模型 | 不重试 |
| 404 | `not_found_error` | 资源不存在 | 不重试 |
| 413 | `request_too_large` | 请求 > 32MB | 拆分 |
| 429 | `rate_limit_error` | 限流 | 指数退避重试 |
| 500 | `api_error` | 服务端 bug | 重试 |
| 529 | `overloaded_error` | 整体过载 | 退避 + 切备用 model/region |

### 10.2 限流 headers

```
anthropic-ratelimit-requests-limit: 4000
anthropic-ratelimit-requests-remaining: 3987
anthropic-ratelimit-requests-reset: 2026-05-14T10:30:00Z
anthropic-ratelimit-tokens-limit: 400000
anthropic-ratelimit-tokens-remaining: 398000
anthropic-ratelimit-tokens-reset: 2026-05-14T10:30:00Z
anthropic-ratelimit-input-tokens-limit: 200000
anthropic-ratelimit-input-tokens-remaining: 199000
anthropic-ratelimit-input-tokens-reset: 2026-05-14T10:30:00Z
anthropic-ratelimit-output-tokens-limit: 80000
retry-after: 5
```

**双轨限流**：

- RPM（requests per minute）
- TPM（tokens per minute，input + output 各算一桶）

429 时尊重 `retry-after`；没有该头时用指数退避（1s → 2s → 4s → 8s）。

### 10.3 SDK 内置重试

```go
client := anthropic.NewClient(option.WithMaxRetries(3))
// 内部对 429 / 5xx / 529 自动重试，按 Retry-After 退避
```

### 10.4 自己实现重试

```go
func callWithRetry(ctx context.Context, params anthropic.MessageNewParams) (*anthropic.Message, error) {
    var lastErr error
    for i := 0; i < 4; i++ {
        msg, err := client.Messages.New(ctx, params)
        if err == nil { return msg, nil }
        lastErr = err
        
        var apiErr *anthropic.Error
        if errors.As(err, &apiErr) {
            switch apiErr.StatusCode {
            case 400, 401, 403, 404, 413:
                return nil, err  // 不重试
            case 429, 500, 502, 503, 529:
                wait := time.Duration(1<<i) * time.Second
                if ra := apiErr.Headers.Get("Retry-After"); ra != "" {
                    if sec, err := strconv.Atoi(ra); err == nil {
                        wait = time.Duration(sec) * time.Second
                    }
                }
                select {
                case <-ctx.Done(): return nil, ctx.Err()
                case <-time.After(wait):
                }
            default:
                return nil, err
            }
        }
    }
    return nil, lastErr
}
```

### 10.5 流中错误

流式响应里**也可能**收到 `event: error`：

```
event: error
data: {"type":"error","error":{"type":"overloaded_error","message":"..."}}
```

这种错误在流"开始后"才出现——一部分 token 已经吐给客户端了。处理：

- 转发错误到前端（让 UI 显示"中断"提示）
- 累积已收到的部分文本，作为不完整回复返回给应用层
- 如果是 retry-safe 的请求，可以从头重发整个 conversation

---

## 第十一章：生产实践

### 11.1 配置层

```go
type ClaudeConfig struct {
    APIKey          string
    DefaultModel    string                // claude-sonnet-4-6
    FallbackModel   string                // claude-haiku-4-5-20251001
    MaxRetries      int                   // 3
    RequestTimeout  time.Duration         // 120s
    StreamTimeout   time.Duration         // 600s（长推理任务）
    MaxConcurrency  int                   // 全局并发限制
    EnableCaching   bool                  // prompt caching 总开关
    CacheTTL        string                // "5m" / "1h"
}
```

### 11.2 并发控制

```go
import "golang.org/x/sync/semaphore"

var sem = semaphore.NewWeighted(50) // 全局 50 并发

func call(ctx context.Context, params anthropic.MessageNewParams) (*anthropic.Message, error) {
    if err := sem.Acquire(ctx, 1); err != nil { return nil, err }
    defer sem.Release(1)
    return client.Messages.New(ctx, params)
}
```

也可以按 model 分桶（Opus 限流远比 Haiku 紧），避免单一模型把全局桶吃满。

### 11.3 成本控制

```go
type CostMonitor struct {
    mu       sync.Mutex
    Spent    float64  // USD
    Budget   float64
    Rates    map[string]ModelPricing
}

type ModelPricing struct {
    InputPerM     float64
    OutputPerM    float64
    CacheWritePerM float64
    CacheReadPerM  float64
}

func (m *CostMonitor) Track(model string, u anthropic.Usage) error {
    m.mu.Lock(); defer m.mu.Unlock()
    p := m.Rates[model]
    cost := float64(u.InputTokens)*p.InputPerM/1e6 +
            float64(u.OutputTokens)*p.OutputPerM/1e6 +
            float64(u.CacheCreationInputTokens)*p.CacheWritePerM/1e6 +
            float64(u.CacheReadInputTokens)*p.CacheReadPerM/1e6
    m.Spent += cost
    if m.Spent > m.Budget {
        return errors.New("budget exceeded")
    }
    return nil
}
```

2026 年 5 月主流价格（API 公开价，USD / 1M tokens）：

| 模型 | input | output | cache write 5m | cache read |
|---|---|---|---|---|
| `claude-opus-4-7` | 15 | 75 | 18.75 | 1.5 |
| `claude-sonnet-4-6` | 3 | 15 | 3.75 | 0.3 |
| `claude-sonnet-4-6` (1M ctx 区段) | 6 | 22.5 | 7.5 | 0.6 |
| `claude-haiku-4-5-20251001` | 1 | 5 | 1.25 | 0.1 |

> 这里只是举例量级，实际价格以 [anthropic.com/pricing](https://www.anthropic.com/pricing) 为准。

### 11.4 超时分层

```go
ctx, cancel := context.WithTimeout(parentCtx, 600*time.Second)  // 总超时
defer cancel()

// SDK 层也设
client := anthropic.NewClient(option.WithRequestTimeout(120*time.Second))
```

extended thinking 任务可能跑几分钟——务必把 streaming 超时设得长，否则中途断流。

### 11.5 日志与 trace

```go
type CallLog struct {
    RequestID    string
    Model        string
    InputTokens  int
    OutputTokens int
    CacheHit     int
    Latency      time.Duration
    StopReason   string
    Cost         float64
    UserID       string
    Conversation string
}
```

至少要记录这些字段——后续做 RCA、计费、告警都依赖。集成 OpenTelemetry GenAI semantic convention 见 **A13 — 精通 LLM 可观测性**。

### 11.6 完整生产模板

```go
type Provider struct {
    client       anthropic.Client
    cfg          ClaudeConfig
    sem          *semaphore.Weighted
    cost         *CostMonitor
    log          *slog.Logger
}

func (p *Provider) Chat(ctx context.Context, conv *Conversation) (*Reply, error) {
    if err := p.sem.Acquire(ctx, 1); err != nil { return nil, err }
    defer p.sem.Release(1)
    
    ctx, cancel := context.WithTimeout(ctx, p.cfg.RequestTimeout)
    defer cancel()
    
    params := p.buildParams(conv)
    start := time.Now()
    msg, err := callWithRetry(ctx, params)
    if err != nil {
        // fallback 到便宜模型
        params.Model = anthropic.F(p.cfg.FallbackModel)
        msg, err = callWithRetry(ctx, params)
        if err != nil { return nil, err }
    }
    
    if err := p.cost.Track(string(params.Model.Value), msg.Usage); err != nil {
        p.log.Warn("budget exceeded", "err", err)
    }
    p.log.Info("claude call",
        "model", msg.Model,
        "input", msg.Usage.InputTokens,
        "output", msg.Usage.OutputTokens,
        "cache_hit", msg.Usage.CacheReadInputTokens,
        "stop", msg.StopReason,
        "latency", time.Since(start),
    )
    return toReply(msg), nil
}
```

---

## 第十二章：陷阱清单

### 1. 忘了 `max_tokens`

Anthropic 不给默认值。漏写 → 400。

### 2. cache_control 加在小 prompt 上

低于 1024 / 2048 token 阈值——cache_control 被忽略，多花 25% 的 cache write 钱却没缓存。**自己监控 `CacheCreationInputTokens > 0` 但 `CacheReadInputTokens == 0`**——这是"花钱没收益"信号。

### 3. messages 顺序错

必须 user/assistant 交替；连续两个 user → 400。Tool 循环时容易踩——tool_result 必须放在新的 user message 里。

### 4. cache 因毫秒变化 miss

在 system prompt 里塞了 `time.Now()` / 随机 nonce → 永远 miss。把时间戳放在**最后一个非 cached message**。

### 5. streaming 不 close

```go
stream := client.Messages.NewStreaming(ctx, params)
// 没 for stream.Next() 就 return → 连接 / token 泄漏
```

要么消费完，要么 `ctx cancel()` 主动断开。SDK 在 ctx done 时会清理。

### 6. tool_result 写错 role

tool_result 是 user message 的一种 content block，**不是** assistant 的。新手常写成 assistant role → 400。

### 7. 多次 call 用同一个 ctx

```go
ctx, _ := context.WithTimeout(parent, 30*time.Second)
for i := 0; i < 100; i++ {
    client.Messages.New(ctx, ...) // 30s 后全部失败
}
```

正确做法：每次新建 ctx，或用更大的总超时。

### 8. 用 `claude-3-5-sonnet-...` 老 model ID

2026 年很多老 ID 仍可用但已不推荐——它们性能差、价格不优。**生产**强制走 4.x。

### 9. 直接 print Content[0].Text

```go
fmt.Println(resp.Content[0].Text)
```

如果开了 extended thinking，Content[0] 可能是 ThinkingBlock 而不是 TextBlock。遍历找 TextBlock：

```go
for _, b := range resp.Content {
    if tb, ok := b.AsAny().(anthropic.TextBlock); ok {
        fmt.Println(tb.Text)
    }
}
```

### 10. batch 之后用普通价心算

Batch 在 dashboard 计费里是**独立科目**——不要按普通价算预算。

### 11. retry 把流式请求重发但前端已经收到部分 token

流式中途 5xx 后简单 retry → 前端 token 重复。要么从头重发并通知前端清屏，要么把已收到部分作为最终结果返回。

### 12. extended thinking budget 设太小

`BudgetTokens: 100` → 模型还没想清楚就停。复杂问题至少 2000-5000，证明 / 长 Agent 任务给 10000+。

---

## 第十三章：2026 现状

### 13.1 模型版图

```
顶配:        Opus 4.7         （15/75 USD per M tokens）
通用:        Sonnet 4.6       （3/15；1M ctx 区段 6/22.5）
快速:        Haiku 4.5        （1/5；2025-10-01 release 起 ID 含日期）
```

**何时选 Opus**：复杂推理、长 Agent 任务、代码大改、要 extended thinking。

**何时选 Sonnet 4.6**：默认主力——质量接近 Opus、价格 1/5。1M context beta 适合超长文档 / 项目级 codebase。

**何时选 Haiku**：分类、抽取、轻 RAG（top-k 后回答）、agent 中"工具调度员"角色、批量任务。

### 13.2 对比 GPT-5 / Gemini 2.5

| 维度 | Claude 4.x | GPT-5 | Gemini 2.5 |
|---|---|---|---|
| 长上下文 | Sonnet 1M | 400k（GPT-5 main） | 2M Pro / 1M Flash |
| 思考链 | extended thinking（可见 / 加密） | reasoning model（o3-mini）独立产品线 | thinking mode |
| Tool use | tool_use（成熟） | function calling（v3，Responses API） | function calling |
| 多模态 | vision（不含音频生成） | vision + audio + voice | vision + audio + 视频原生 |
| Prompt caching | ephemeral 5m/1h（成熟） | 自动 prompt caching（透明） | implicit + explicit |
| Batch | 50% | 50%（24h） | 50% |
| 代码能力 | Opus 4.7 业界顶尖（SWE-bench 高分） | GPT-5 强 | Gemini 2.5 Pro 强 |
| Agent loop | Anthropic agent SDK + Claude Code | OpenAI agents SDK | Google ADK |

**2026 年 5 月的真实选型经验**：

- 复杂 Agent / 高准代码 → Opus 4.7
- 默认主力 → Sonnet 4.6（性价比之王）
- 高吞吐分类 → Haiku 4.5 或 Gemini Flash
- 超长 context（书本 / 代码库） → Gemini 2.5 Pro（2M）或 Sonnet 4.6（1M）
- 需要 OpenAI 生态绑定（Whisper、TTS、Realtime API） → GPT-5 系列

### 13.3 MCP 与 Agent SDK

Anthropic 2024-11 推出 **MCP (Model Context Protocol)**，2025 年成为 Agent / IDE 接入 tool 的事实标准。Claude API 内部也已对 MCP 友好——你可以把 MCP server 暴露的 tool 直接喂进 `tools` 字段（详见 **A10 — 精通 MCP**）。

2026 年 5 月 Anthropic 主推的"agentic loop"模式：

```
loop:
  call_claude(messages, tools)
  if stop_reason == tool_use:
      execute_tools()
      append_results_to_messages()
      continue
  break
```

简洁、显式、可观测。**取代了 2023-2024 流行的 LangChain 高度抽象**——直接看 raw messages 反而更稳。

### 13.4 价格趋势

2024 → 2026 主流模型价格大约**降到 1/3 到 1/2**。Prompt caching 的普及 + batch + 模型小型化（Haiku）让 LLM 应用进入"分级使用、按场景下沉"的成熟期。**LLM token 不再是稀缺资源——但**仍要管。

---

## 第十四章：练习题

**练习 1**：解释为什么在 system message 末尾放当前时间会破坏 prompt cache 命中。如何重新设计？

**练习 2**：以下代码哪里有问题？

```go
messages := []anthropic.MessageParam{
    anthropic.NewUserMessage(anthropic.NewTextBlock("hi")),
    anthropic.NewUserMessage(anthropic.NewTextBlock("how are you")),
}
client.Messages.New(ctx, anthropic.MessageNewParams{
    Model: anthropic.ModelClaudeSonnet4_6,
    Messages: anthropic.F(messages),
})
```

**练习 3**：你要离线给 50 万条评论做情感分类。预算很紧。设计完整方案。

**练习 4**：写 Go 函数 `callWithFallback(ctx, prompt)` —— 优先用 Opus 4.7；如果 429 或 529 退化到 Sonnet 4.6；如果再次 429 退化到 Haiku 4.5。每次重试要把 model 切到下一档。

**练习 5**：解释 extended thinking 与 tool use 同时启用时的事件顺序（SSE 里）。

**练习 6**：客户端在 streaming 接收到一半时网络断开。后端如何记录：
- 已计费 input tokens
- 部分计费 output tokens
- 重发策略

**练习 7**：你做长 RAG 任务——每次问询都带 50KB 知识库 + 10 KB 历史。如何用 cache_control 把每次成本降到最低？

**练习 8**：tool_use 模型返回 `parallel tool_use` 两个 tool。其中一个执行失败（抛异常）。怎么写 `tool_result`？

---

## 参考答案

**练习 1**：cache 是按 prefix 精确匹配的。`"今天是 2026-05-14 10:23:45"` 每秒变 → 每次请求 prefix 不同 → 永远 miss。重设计：

```json
"system": [
  {"type": "text", "text": "<稳定的 instructions>", "cache_control": {"type": "ephemeral"}},
  {"type": "text", "text": "今天是 2026-05-14"}   // 不打 cache，放在 cached 段之后
]
```

或者干脆把时间放到 user message 第一句。**Cache 段不能含动态内容**。

**练习 2**：连续两个 user message → API 400。Messages API 要求 user / assistant 严格交替。修：合并成一个 user 或在中间插 assistant。同时 `MaxTokens` 也没设 → 也会 400。

**练习 3**：Haiku 4.5 + Batch + Prompt Caching。

- Model：Haiku 4.5（最便宜，1/5 USD per M output，分类任务质量已够）
- Batch：input/output 半价，50 万条肯定不实时
- 共享 prompt：所有请求 system 用同一个 `"你是情感分类器，输出 positive/negative/neutral，只输出标签"`——这部分如果超过 2048 token 可上 cache（实际此场景太短无收益，省略）
- 用 stop_sequence：`["positive", "negative", "neutral"]` → 模型一吐标签就停，output 控制在 1-2 token
- 估算：50w × (~100 input + ~2 output) tokens × Haiku batch 价（0.5/2.5 per M）≈ ($0.5×50w×100/1M + $2.5×50w×2/1M)/2 ≈ ~$26
- 实施：拆 5 个 batch（每个 10w 请求），并发提交，24h 内全部返回

**练习 4**：

```go
var models = []string{
    anthropic.ModelClaudeOpus4_7,
    anthropic.ModelClaudeSonnet4_6,
    anthropic.ModelClaudeHaiku4_5,
}

func callWithFallback(ctx context.Context, prompt string) (*anthropic.Message, error) {
    var lastErr error
    for _, m := range models {
        msg, err := client.Messages.New(ctx, anthropic.MessageNewParams{
            Model:     anthropic.F(m),
            MaxTokens: anthropic.F(int64(1024)),
            Messages:  anthropic.F([]anthropic.MessageParam{anthropic.NewUserMessage(anthropic.NewTextBlock(prompt))}),
        })
        if err == nil { return msg, nil }
        var apiErr *anthropic.Error
        if errors.As(err, &apiErr) && (apiErr.StatusCode == 429 || apiErr.StatusCode == 529) {
            lastErr = err
            continue // 试下一档
        }
        return nil, err // 其他错误不降级
    }
    return nil, lastErr
}
```

**练习 5**：

```
message_start
content_block_start (index=0, type=thinking)
  ... thinking_delta x N
content_block_stop (index=0)
content_block_start (index=1, type=tool_use)
  ... input_json_delta x N
content_block_stop (index=1)
message_delta (stop_reason=tool_use)
message_stop
```

如果模型决定先思考再回答（不调 tool）：thinking → text。如果思考后还要思考、调多个 tool：thinking → tool_use → thinking → tool_use → text。

**练习 6**：

- input tokens 在 `message_start` 事件里给（usage.input_tokens 已含 cache_creation/read）→ 立即计入计费
- output tokens 在每个 `message_delta` 累计（usage.output_tokens 单调递增），最终值在最后一个 `message_delta` 或 `message_stop` 之前
- 网络中断后：把"最后看到的 output_tokens" 当作已用计入
- 重发策略：因为是无状态 API，重发等于全新一次（input 又计费一次）。**幂等性要业务层做**——比如保存 conversation hash + 部分输出，重启时让用户确认"继续"或"重答"。也可以让前端只给用户看最后一次完整 response，部分内容不展示。

**练习 7**：

```
system:
  [稳定 instructions]                          ← cache_control ephemeral (5m)
  [50KB 企业知识库]                              ← cache_control ephemeral (1h)
messages:
  [前 n-1 轮对话]                                ← cache_control ephemeral (5m) 放在 history 末
  [本轮用户输入]                                  ← 不缓存
```

四个 breakpoint 分配：

1. system 末（稳定 instructions + 知识库整段）
2. tools 段末（如果有 tool）
3. 历史对话最后一条（这是 anthropic 推荐的"history 缓存"模式）

知识库改动频率低 → 1h TTL；对话历史 5min 够。

理论命中后：每次只算"本轮新增 input" 的 1x + 历史的 0.1x。10 轮对话后，cache_read 占 input 的 90% 以上是常见的。

**练习 8**：

```go
// 失败的 tool_result 用 is_error: true
{
  type: "tool_result",
  tool_use_id: "toolu_xxx",
  content: "<error message>",
  is_error: true,
}
// 成功的正常返回
{
  type: "tool_result",
  tool_use_id: "toolu_yyy",
  content: "<result>",
}
```

两个 tool_result 都放在**同一个 user message** 的 content 数组里（这是 parallel tool use 的关键——一次性回填全部）。

```go
toolResults := []anthropic.ContentBlockParamUnion{
    anthropic.NewToolResultBlock("toolu_xxx", "tool failed: network timeout", true),
    anthropic.NewToolResultBlock("toolu_yyy", `{"ok": true}`, false),
}
messages = append(messages, anthropic.NewUserMessage(toolResults...))
```

模型看到 `is_error: true` 会自动尝试错误恢复——重试、换 tool、向用户解释。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| SDK 选型 | 官方 `anthropic-sdk-go`，必要时直接 HTTP |
| Messages API | system 顶层、user/assistant 交替、max_tokens 必填 |
| Prompt caching | cache_control ephemeral、4 个 breakpoint、5m/1h TTL、1024/2048 阈值 |
| Cache 计费 | write 1.25x（5m）/ 2x（1h）；read 0.1x |
| Streaming | SSE event：message_start / content_block_* / message_delta / message_stop |
| Batch | input+output 半价、24h、最多 100k 请求 |
| Files | 32MB PDF / 5MB 图片、file_id 引用 |
| Tool use | stop_reason=tool_use → 收集 tool_use → 回填 tool_result → 再请求 |
| Extended thinking | budget_tokens、thinking block、可见 / 加密 |
| Citations | RAG 自动标注来源 |
| 重试 | 429 / 5xx / 529 退避；尊重 retry-after |
| 限流 | RPM + TPM 双轨；input/output 分桶 |
| 监控 | InputTokens / OutputTokens / CacheCreation / CacheRead / StopReason |
| 选型 2026 | Opus 顶配 / Sonnet 默认 / Haiku 高吞吐 |

铁律：

- 永远设 `max_tokens` 与 `RequestTimeout`
- 长 prompt 一定打 cache_control
- batch 离线任务永远走 batch
- streaming 必须消费完或 ctx cancel
- 监控 cache 命中率 + cost per call
- Opus/Sonnet/Haiku 按场景下沉
- 重试要尊重 Retry-After + 区分错误类型
- tool_use 循环用 raw messages 自己写，不要套高层抽象

下一篇 **A02 — 精通 OpenAI 兼容生态** 将拆开 Chat Completions / Responses / Assistants 三套 API、Go SDK 选型、OpenAI 协议在第三方平台的兼容现状。

---
