# 精通 OpenAI 兼容生态：Chat、Responses、Assistants 与第三方兼容层

> 课程编号：A02
> 路线图来源：AI / LLM 后端工程 · 模块一 API 基础
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月

---

## 引言：OpenAI 不只一套 API，三套并存中

```go
// 老世界：Chat Completions（2023-）
client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F(openai.ChatModelGPT5),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.UserMessage("hi"),
    }),
})

// 新主推：Responses API（2025-）
client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F("gpt-5"),
    Input: openai.F[any]("hi"),
})

// 半旧不新：Assistants API v2（待并入 Responses）
client.Beta.Assistants.New(ctx, ...)
```

OpenAI 在 2024-2025 同时维护这三套 API，让"OpenAI 协议"成为了一个**模糊概念**——你说兼容 OpenAI，往往特指**兼容 Chat Completions 的请求 / 响应 schema**。本章拆开这三套 API，澄清现状、写 Go 实战代码、对比第三方平台的兼容层落地。

2026 年 5 月主流 OpenAI 系模型 ID：

| ID | 别名 | 角色 |
|---|---|---|
| `gpt-5.5` | GPT-5.5 | 当前旗舰，复杂推理 / 长任务 |
| `gpt-5` | GPT-5 main | 顶配，复杂推理 / 长任务 |
| `gpt-5-mini` | GPT-5 mini | 平衡价格 / 速度 |
| `gpt-5-nano` | GPT-5 nano | 极致便宜 / 高吞吐 |
| 推理模式 | GPT-5.x thinking/reasoning | 数学 / 代码 / 形式推理（o3/o3-mini 已退役） |
| `gpt-4o` / `gpt-4o-mini` | 老一代多模态 | 仍存在 / 价格已大幅下调 |

---

## 第一章：三套 API 形态对比

### 1.1 视图

```
   ┌────────────────────────────────────────────────────┐
   │ Chat Completions API (2023~)                       │
   │  /v1/chat/completions                              │
   │  - 无状态                                            │
   │  - messages 数组 (role + content)                  │
   │  - tools / function calling                        │
   │  - 工业标准、兼容层最广                              │
   └────────────────────────────────────────────────────┘
   ┌────────────────────────────────────────────────────┐
   │ Assistants API v2 (2024)                           │
   │  /v1/assistants, /v1/threads, /v1/runs             │
   │  - 有状态（thread 持久化对话）                        │
   │  - 内置 tools: code interpreter / file search      │
   │  - 复杂、轮询模型                                    │
   │  - 已宣布将并入 Responses                            │
   └────────────────────────────────────────────────────┘
   ┌────────────────────────────────────────────────────┐
   │ Responses API (2025~ 主推)                          │
   │  /v1/responses                                      │
   │  - 可选有状态 (previous_response_id)                │
   │  - 内置 tools (web search, file search,            │
   │     computer use, code interpreter)                │
   │  - 流式 + 多模态 + reasoning + structured output    │
   │  - 取代 Chat Completions + Assistants 的目标       │
   └────────────────────────────────────────────────────┘
```

### 1.2 三句话总结

- **Chat Completions**：最广兼容、最简单；新模型新特性会延迟支持，但永远跟着主线发版。
- **Assistants v2**：2024 推出，2025 起被 Responses 取代，OpenAI 已宣布**最早 2026 年中 deprecate**。新项目不要用。
- **Responses**：2025 起 OpenAI 主推。**新项目应直接选 Responses**，除非你必须跟某个非 OpenAI 的"OpenAI 兼容"端点对话。

### 1.3 兼容生态的视角

外部世界（vLLM、TGI、Together、DeepSeek、阿里千问、Anthropic 提供的 OpenAI 兼容 endpoint 等）**只兼容 Chat Completions**。它们没有也不打算实现 Assistants / Responses。

所以现实选择：

| 场景 | 推荐 API |
|---|---|
| 只用 OpenAI 官方、长期单一供应商 | Responses |
| 多供应商路由（OpenAI / Anthropic / 国产 / 自建 vLLM） | Chat Completions |
| 已在用 Assistants | 计划迁移到 Responses |
| 新 Agent 项目 + 用 OpenAI | Responses |
| 简单 chatbot / 工具型 LLM 调用 | Chat Completions（足够） |

---

## 第二章：Go SDK 选型

### 2.1 官方 openai-go

```bash
go get github.com/openai/openai-go
```

```go
import (
    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
)

client := openai.NewClient(
    option.WithAPIKey(os.Getenv("OPENAI_API_KEY")),
    option.WithMaxRetries(2),
)
```

特点：

- 跟随 API 更新最快（Responses、新模型、tool 类型）
- 全 typed params（`openai.F[T]()` 包装）
- 内置重试、流式
- 文档生成自 OpenAPI schema

**新项目应当优先**。

### 2.2 社区 sashabaranov/go-openai

```bash
go get github.com/sashabaranov/go-openai
```

```go
import "github.com/sashabaranov/go-openai"

client := openai.NewClient(os.Getenv("OPENAI_API_KEY"))
resp, _ := client.CreateChatCompletion(ctx, openai.ChatCompletionRequest{
    Model: openai.GPT4oMini,
    Messages: []openai.ChatCompletionMessage{
        {Role: openai.ChatMessageRoleUser, Content: "hi"},
    },
})
```

特点：

- 社区维护多年，API 稳定
- 上手友好（plain struct，少 `openai.F` 包装）
- Responses API 支持滞后
- 兼容层最常用——很多兼容 OpenAI 的国产 / 自建服务都是按它的 client 行为测的

**主要适合**：

- 只用 Chat Completions
- 切换多 provider（DeepSeek、千问、自建 vLLM）
- 喜欢 plain struct 的代码风格

### 2.3 第三选项：直接 HTTP

```go
req := struct {
    Model    string   `json:"model"`
    Messages []msg    `json:"messages"`
    Stream   bool     `json:"stream,omitempty"`
}{...}
body, _ := json.Marshal(req)
http.NewRequest("POST", "https://api.openai.com/v1/chat/completions", bytes.NewReader(body))
```

适用：

- 极简依赖
- 自建网关（中间层做 OpenAI 协议代理）
- LLM Gateway / litellm-like 项目

### 2.4 SDK 对比速查

| 维度 | openai-go（官方） | sashabaranov | 直接 HTTP |
|---|---|---|---|
| Chat Completions | ✓ | ✓ | ✓ |
| Responses | ✓ | 滞后 | 自己写 |
| Assistants v2 | ✓ | 部分 | 自己写 |
| Files / Embedding | ✓ | ✓ | 自己写 |
| Audio (Whisper/TTS) | ✓ | ✓ | 自己写 |
| Realtime API | ✓ | 缺 | 自己写 |
| Batch | ✓ | ✓ | ✓ |
| 兼容层友好（自定义 baseURL） | ✓ | ✓ | ✓ |
| 维护频率 | 高 | 中 | - |

**主流选择**：

- OpenAI 单一供应商 → `openai-go` 官方
- 多 provider + 主要走 Chat Completions → `sashabaranov`
- 写 Gateway → 直接 HTTP

---

## 第三章：Chat Completions API（兼容层基础）

### 3.1 请求结构

```json
{
  "model": "gpt-5-mini",
  "messages": [
    {"role": "system", "content": "你是 Go 专家"},
    {"role": "user", "content": "解释 sync.Map"}
  ],
  "temperature": 1.0,
  "max_completion_tokens": 1024,
  "stream": false,
  "tools": [...],
  "tool_choice": "auto",
  "response_format": {"type": "json_schema", "json_schema": {...}}
}
```

**注意 2026 年的变化**：`max_tokens` 在 GPT-5 系列被 `max_completion_tokens` 取代（因为 reasoning tokens 也算输出）。老模型仍接受 `max_tokens`。

### 3.2 角色

```
system     系统指令
user       用户输入
assistant  模型输出（含 tool_calls）
tool       工具调用结果（必须紧跟 assistant 的 tool_calls）
developer  GPT-5 引入：比 system 更高优先级（指令体系）
```

`developer` 是 GPT-5 系列推出的新角色：开发者写在这里的指令"应当"被模型优先服从、不可被 user 推翻。系统设计上类似 Claude 的 system + agent instructions 分层。

### 3.3 Go 调用

```go
import "github.com/openai/openai-go"

resp, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F(openai.ChatModelGPT5Mini),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.SystemMessage("你是 Go 专家"),
        openai.UserMessage("sync.Map vs map+mutex 怎么选？"),
    }),
    MaxCompletionTokens: openai.Int(1024),
})
fmt.Println(resp.Choices[0].Message.Content)
```

### 3.4 多轮对话

无状态 API——每次把完整历史发回：

```go
messages := []openai.ChatCompletionMessageParamUnion{
    openai.UserMessage("你好"),
    openai.AssistantMessage("你好！"),
    openai.UserMessage("解释 CRDT"),
}
```

### 3.5 响应结构

```go
type ChatCompletion struct {
    ID      string
    Choices []Choice
    Usage   CompletionUsage
    Model   string
    Created int64
}

type Choice struct {
    Index        int
    Message      ChatCompletionMessage
    FinishReason string  // stop | length | tool_calls | content_filter
}

type CompletionUsage struct {
    PromptTokens            int
    CompletionTokens        int
    TotalTokens             int
    PromptTokensDetails     PromptTokensDetails     // cached_tokens
    CompletionTokensDetails CompletionTokensDetails // reasoning_tokens
}
```

**`PromptTokensDetails.CachedTokens`**：OpenAI 的自动 prompt caching 命中数。无需 client 标注——OpenAI 内部自动判断长 prompt 复用、自动半价（详见第十章）。

**`CompletionTokensDetails.ReasoningTokens`**：GPT-5 / o3 系列的"推理 token"消耗，按 output 价计算但不会出现在 message 里（隐藏的"思考"）。

### 3.6 FinishReason 监控

| 值 | 含义 |
|---|---|
| `stop` | 自然结束 |
| `length` | 触发 max_completion_tokens 截断 |
| `tool_calls` | 模型要调 tool |
| `content_filter` | 安全过滤拦截 |
| `function_call` | 老式 function call（已被 tool_calls 取代） |

---

## 第四章：Responses API——2025 主推

### 4.1 设计目标

把 Assistants 的"有状态 + 内置 tools"和 Chat Completions 的"简单 stateless"合并成**一套**：

```
Responses API
├── 输入可以是 string、message[]、或 multi-modal content
├── 可选 previous_response_id → 自动接续上一轮（有状态）
├── 内置 tools（web_search、file_search、computer_use、code_interpreter）
├── 自定义 tools（function tool）
├── 多模态 input + output
├── 结构化 output（json_schema）
├── 流式 SSE
└── reasoning（GPT-5 / o3 系列）
```

### 4.2 基本调用

```go
resp, _ := client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F(openai.ChatModelGPT5),
    Input: openai.F[any]("总结这篇论文"),
})
fmt.Println(resp.OutputText)  // 便捷字段，已合并所有 text output
```

`Input` 可以是：

- string（单轮）
- `[]ResponseInputItem`（多轮 user/assistant）
- 多模态：`[]ResponseInputItem` 含 text / image / file 等

### 4.3 有状态：previous_response_id

```go
// 第一轮
r1, _ := client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F(openai.ChatModelGPT5),
    Input: openai.F[any]("我叫 Alice"),
})

// 第二轮（自动接续 r1 的上下文）
r2, _ := client.Responses.New(ctx, openai.ResponseNewParams{
    Model:              openai.F(openai.ChatModelGPT5),
    PreviousResponseID: openai.F(r1.ID),
    Input:              openai.F[any]("我叫什么？"),
})
fmt.Println(r2.OutputText)  // "Alice"
```

OpenAI 服务端保存上轮 context。客户端只发增量。

**注意**：

- 默认 30 天保留期
- 跨账号 / 跨组织不共享
- 不适合需要客户端自主管理对话（如需要修剪、改写）的场景
- 可以用 `store=false` 显式不持久化（每次完整带 history）

### 4.4 内置 tools

```go
r, _ := client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F(openai.ChatModelGPT5),
    Input: openai.F[any]("OpenAI 最近发布了什么"),
    Tools: openai.F([]openai.ResponseTool{
        {Type: openai.F("web_search_preview")},
    }),
})
```

工具列表（2026 年 5 月）：

| Tool | 用途 |
|---|---|
| `web_search_preview` | 自动网页搜索 + 引用 |
| `file_search` | 上传文件后做 RAG（向量索引 OpenAI 托管） |
| `code_interpreter` | 沙箱里跑代码（Python） |
| `computer_use_preview` | 控制虚拟桌面（鼠标、键盘、截屏） |
| `image_generation` | 调 DALL-E / GPT-Image-1 生图 |

调用过程**完全在 OpenAI 服务端**——你不需要自己实现 web 搜索、不需要管 vector store；只需在 `Tools` 字段声明启用。

### 4.5 自定义 tools

```go
Tools: openai.F([]openai.ResponseTool{
    {
        Type:        openai.F("function"),
        Name:        openai.F("get_weather"),
        Description: openai.F("获取天气"),
        Parameters: openai.F(map[string]any{
            "type": "object",
            "properties": map[string]any{
                "city": map[string]any{"type": "string"},
            },
            "required": []string{"city"},
        }),
    },
}),
```

模型决定调用时，response.output 里出现 `function_call` item：

```go
for _, item := range resp.Output {
    if fc, ok := item.AsAny().(openai.ResponseFunctionToolCall); ok {
        result := callMyTool(fc.Name, fc.Arguments)
        // 下一轮 input 里塞 function_call_output
    }
}
```

### 4.6 Reasoning（GPT-5 / o3 系列）

```go
r, _ := client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F(openai.ChatModelO3),
    Input: openai.F[any]("证明费马小定理"),
    Reasoning: openai.F(openai.ResponseNewParamsReasoning{
        Effort: openai.F("high"), // low / medium / high
    }),
})
fmt.Printf("reasoning tokens: %d\n", r.Usage.OutputTokensDetails.ReasoningTokens)
```

`Effort` 是 OpenAI 对"想多深"的简化抽象——不暴露内部 budget 数字，由模型自己决定。

reasoning tokens **不会**回传给客户端（推理是黑盒），但会按 output 价计费。

### 4.7 与 Chat Completions 的关键差异

| 维度 | Chat Completions | Responses |
|---|---|---|
| 路径 | `/v1/chat/completions` | `/v1/responses` |
| 输入 | `messages: [{role, content}]` | `input: string \| items[]` |
| 状态 | 无 | 可选 (`previous_response_id`) |
| 内置 tool | 无 | web/file_search/computer/code_interpreter |
| reasoning effort | 无 | 有 |
| output 字段 | `choices[0].message` | `output[]` + `output_text` 便捷字段 |
| 流式 event 名 | 单一 chunk | 细化（参见第八章） |
| structured output | response_format | text.format（统一） |

---

## 第五章：Assistants API v2（过渡期，新项目别用）

### 5.1 心智模型

```
Assistant (一个可复用的"角色"，含 instructions + tools)
   ↓
Thread (一段对话)
   ↓
Messages (thread 内的消息)
   ↓
Run (执行 thread + assistant 的一次)
```

### 5.2 流程

```go
// 1. 创建 assistant
asst, _ := client.Beta.Assistants.New(ctx, openai.BetaAssistantNewParams{
    Model:        openai.F(openai.ChatModelGPT4o),
    Instructions: openai.F("你是法律助手"),
    Tools:        openai.F([]openai.AssistantTool{...}),
})

// 2. 新对话
thread, _ := client.Beta.Threads.New(ctx, openai.BetaThreadNewParams{})

// 3. 加消息
client.Beta.Threads.Messages.New(ctx, thread.ID, openai.BetaThreadMessageNewParams{
    Role: openai.F(openai.BetaThreadMessageNewParamsRoleUser),
    Content: openai.F([]openai.MessageContentPartParamUnion{
        openai.TextContentBlockParam{Text: "..."},
    }),
})

// 4. 启动 run
run, _ := client.Beta.Threads.Runs.New(ctx, thread.ID, openai.BetaThreadRunNewParams{
    AssistantID: openai.F(asst.ID),
})

// 5. 轮询 run.status
for run.Status != "completed" {
    time.Sleep(500 * time.Millisecond)
    run, _ = client.Beta.Threads.Runs.Get(ctx, thread.ID, run.ID)
    // ... handle requires_action / in_progress / failed
}

// 6. 读消息
msgs, _ := client.Beta.Threads.Messages.List(ctx, thread.ID, ...)
```

### 5.3 为什么过渡

- **复杂**：assistant / thread / run / messages 四对象、状态机
- **状态不可控**：service 端持久化，但 OpenAI 自己内部都看不到 thread context
- **多 provider 路由不可能**：完全 OpenAI 私有协议
- **2025 OpenAI 已宣布**：Responses 是替代方案，Assistants v2 进入弃用倒计时（2026-2027 关停）

**结论**：

- 已经在用：稳，但开始规划迁移到 Responses（OpenAI 提供迁移指南）
- 没在用：**永远别用**

---

## 第六章：Function Calling 协议

### 6.1 在 Chat Completions 里

```json
{
  "model": "gpt-5-mini",
  "messages": [{"role": "user", "content": "纽约多少度"}],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather",
        "parameters": {
          "type": "object",
          "properties": {"city": {"type": "string"}},
          "required": ["city"]
        }
      }
    }
  ],
  "tool_choice": "auto"
}
```

响应：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [
        {
          "id": "call_abc",
          "type": "function",
          "function": {"name": "get_weather", "arguments": "{\"city\":\"NYC\"}"}
        }
      ]
    },
    "finish_reason": "tool_calls"
  }]
}
```

**注意**：`arguments` 是**字符串化的 JSON**，不是 JSON object。需要自己 `json.Unmarshal`。

### 6.2 回填 tool 结果

```go
messages = append(messages,
    openai.AssistantMessage(openai.ChatCompletionAssistantMessageParam{
        ToolCalls: openai.F([]openai.ChatCompletionMessageToolCallParam{toolCall}),
    }),
    openai.ToolMessage("call_abc", `{"temp": 22}`),
)
```

`tool` role 的 message 必须**紧跟** 含 tool_calls 的 assistant message——顺序错就 400。

### 6.3 Parallel tool calls

GPT-5 / GPT-4o 系列默认**并行**多个 tool call：

```json
"tool_calls": [
  {"id": "call_a", ...},
  {"id": "call_b", ...},
  {"id": "call_c", ...}
]
```

要并行执行，分别 `tool` message 回填。**全部回完才能继续**——一个不回就 400。

可以关掉：

```go
ParallelToolCalls: openai.Bool(false),
```

某些 reasoning models 不支持 parallel，需主动关闭。

### 6.4 tool_choice 控制

```go
ToolChoice: openai.F[any]("auto")           // 模型决定（默认）
ToolChoice: openai.F[any]("none")           // 强制不调 tool
ToolChoice: openai.F[any]("required")       // 强制必调一个
ToolChoice: openai.F[any](map[string]any{   // 强制调指定 tool
    "type": "function",
    "function": map[string]any{"name": "get_weather"},
})
```

### 6.5 在 Responses API 里

格式简化（不再有外层 `function` 包装）：

```json
{
  "tools": [
    {"type": "function", "name": "get_weather", "description": "...", "parameters": {...}}
  ]
}
```

响应：

```json
{
  "output": [
    {"type": "function_call", "call_id": "fc_abc", "name": "get_weather", "arguments": "{\"city\":\"NYC\"}"}
  ]
}
```

回填用 `function_call_output`：

```json
{
  "input": [
    {"type": "function_call", "call_id": "fc_abc", ...},
    {"type": "function_call_output", "call_id": "fc_abc", "output": "{\"temp\": 22}"}
  ]
}
```

---

## 第七章：Structured Output（JSON Schema）

### 7.1 三档"严格度"

```
1. response_format: {"type": "text"}              ← 自由文本（默认）
2. response_format: {"type": "json_object"}       ← 强制 JSON（schema 自由）
3. response_format: {"type": "json_schema", ...}  ← 强制符合 schema（推荐）
```

第 3 档是 GPT-4o 起引入的 **Structured Outputs**：保证 100% 符合给定 JSON schema。

### 7.2 严格 schema 示例

```go
schema := map[string]any{
    "name": "review",
    "strict": true,
    "schema": map[string]any{
        "type": "object",
        "properties": map[string]any{
            "summary":   map[string]any{"type": "string"},
            "sentiment": map[string]any{"type": "string", "enum": []string{"positive", "negative", "neutral"}},
            "score":     map[string]any{"type": "integer", "minimum": 1, "maximum": 5},
        },
        "required":             []string{"summary", "sentiment", "score"},
        "additionalProperties": false,
    },
}
resp, _ := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F(openai.ChatModelGPT5Mini),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.UserMessage("评价：这酒店还行，前台冷淡，房间干净"),
    }),
    ResponseFormat: openai.F[openai.ChatCompletionNewParamsResponseFormatUnion](
        openai.ResponseFormatJSONSchemaParam{
            Type:       openai.F(openai.ResponseFormatJSONSchemaTypeJSONSchema),
            JSONSchema: openai.F(schema),
        },
    ),
})
var review struct {
    Summary   string `json:"summary"`
    Sentiment string `json:"sentiment"`
    Score     int    `json:"score"`
}
json.Unmarshal([]byte(resp.Choices[0].Message.Content), &review)
```

**严格模式的限制**：

- 必须 `additionalProperties: false`
- 所有字段都得在 `required` 数组里
- 不支持 `pattern`、`minLength`、`format` 等正则字段
- 嵌套深度限制（≤ 5 层）
- enum 总元素 ≤ 500

**严格度收益**：不再需要 `try/except json.Unmarshal` 重试循环。

### 7.3 Responses API 里

```go
client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F(openai.ChatModelGPT5Mini),
    Input: openai.F[any]("..."),
    Text: openai.F(openai.ResponseTextConfig{
        Format: openai.F(openai.ResponseTextFormatJSONSchema{
            Type:   openai.F("json_schema"),
            Name:   openai.F("review"),
            Strict: openai.F(true),
            Schema: openai.F(schemaMap),
        }),
    }),
})
```

字段名变了（`text.format` 而不是 `response_format`），但语义一致。

### 7.4 与 Anthropic 的对比

| 平台 | 严格 JSON 方案 |
|---|---|
| OpenAI | response_format: json_schema, strict: true |
| Anthropic | 用 tool_use（让模型只调一个 tool，强制 schema）；2025 起加了 `output_schema`（部分模型）但未普及 |
| Gemini | response_mime_type + response_schema |

跨 provider 一致输出最稳的做法仍是 **"让模型调 tool"**——以 tool input schema 强制结构。

---

## 第八章：Streaming——SSE，与 Anthropic 对比

### 8.1 Chat Completions 流式

```
data: {"id":"chatcmpl_xx","choices":[{"index":0,"delta":{"role":"assistant","content":""}}]}

data: {"id":"chatcmpl_xx","choices":[{"index":0,"delta":{"content":"Hello"}}]}

data: {"id":"chatcmpl_xx","choices":[{"index":0,"delta":{"content":" world"}}]}

data: {"id":"chatcmpl_xx","choices":[{"index":0,"finish_reason":"stop"}]}

data: [DONE]
```

特点：

- **统一一类事件**——每个 chunk 都是 `data: {ChatCompletionChunk}`
- 结尾用 `data: [DONE]` 字面量标识结束
- 多 choice / 多 tool_call 通过 `index` 区分

### 8.2 Go 流式调用

```go
stream := client.Chat.Completions.NewStreaming(ctx, params)
for stream.Next() {
    chunk := stream.Current()
    if len(chunk.Choices) > 0 {
        delta := chunk.Choices[0].Delta
        fmt.Print(delta.Content)
    }
}
if err := stream.Err(); err != nil { /* ... */ }
```

### 8.3 Responses 流式

```
event: response.created
data: {...}

event: response.output_item.added
data: {"item": {"type": "message", ...}}

event: response.output_text.delta
data: {"delta": "Hello"}

event: response.output_text.delta
data: {"delta": " world"}

event: response.output_text.done
data: {...}

event: response.completed
data: {"usage": {...}}
```

Responses 流式用**命名事件**（带 `event:` 头），更精细化——每个 output item（message / function_call / reasoning）有独立事件流。

### 8.4 与 Anthropic 的对比

| 维度 | OpenAI Chat Completions | OpenAI Responses | Anthropic Messages |
|---|---|---|---|
| 事件命名 | 单一 chunk | 命名事件 | 命名事件 |
| 结束标志 | `data: [DONE]` 字面量 | `response.completed` | `message_stop` |
| 流中 usage | 最后 chunk 才有 | 多个事件含部分 usage | message_start + message_delta |
| tool_call delta | 一字一字流（JSON 字符串拼接） | function_call_arguments.delta | input_json_delta |
| reasoning | 不流（内部 token） | 不流 | thinking_delta（可见） |

**Anthropic 流式更详细**——thinking、tool 入参、citations 都可流。OpenAI Chat Completions 流式相对扁平。Responses 流式接近 Anthropic 的精细度。

### 8.5 转发给前端

```go
func chatHandler(w http.ResponseWriter, r *http.Request) {
    flusher, _ := w.(http.Flusher)
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("X-Accel-Buffering", "no")

    stream := client.Chat.Completions.NewStreaming(r.Context(), params)
    for stream.Next() {
        chunk := stream.Current()
        data, _ := json.Marshal(chunk)
        fmt.Fprintf(w, "data: %s\n\n", data)
        flusher.Flush()
    }
    if err := stream.Err(); err != nil {
        fmt.Fprintf(w, "event: error\ndata: %s\n\n", err.Error())
    } else {
        fmt.Fprintf(w, "data: [DONE]\n\n")
    }
}
```

详细 SSE 工程化见 **A12**。

---

## 第九章：OpenAI 兼容生态——把"OpenAI 协议"当通用 LLM API

### 9.1 谁兼容了 OpenAI

```
自建推理框架：vLLM / TGI / TensorRT-LLM / llama.cpp / SGLang
托管服务：    Together AI / Fireworks / Groq / Replicate / Anyscale
中国厂商：    DeepSeek / 阿里千问（Qwen）/ 智谱（GLM）/ 月之暗面（Kimi）/ Moonshot / 字节豆包 / 百度文心
本地工具：    Ollama / LM Studio / LocalAI
聚合层：      OpenRouter / litellm / Portkey / Helicone
"竞品" 兼容：  Anthropic 提供 OpenAI 兼容 endpoint / Google 也有
```

它们都暴露：

```
POST {baseURL}/v1/chat/completions
POST {baseURL}/v1/embeddings
POST {baseURL}/v1/models
```

请求 / 响应 schema **照搬** OpenAI Chat Completions。

### 9.2 接入：换 baseURL 即可

```go
client := openai.NewClient(
    option.WithBaseURL("https://api.deepseek.com/v1"),
    option.WithAPIKey(os.Getenv("DEEPSEEK_API_KEY")),
)
resp, _ := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F("deepseek-chat"),  // 自家 model 名
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.UserMessage("hi"),
    }),
})
```

整个客户端代码**几乎不改**——只换 baseURL + model 名 + API key。

### 9.3 兼容度等级

```
Level 1 (基础)：messages、model、stream、max_tokens          ← 几乎全部都支持
Level 2 (中等)：tools/function_calling                       ← 多数支持，schema 可能略不同
Level 3 (高)：  structured output (json_schema strict)       ← 少数支持
Level 4 (高)：  prompt caching                                ← OpenAI / Anthropic / DeepSeek 等支持，多数不支持
Level 5 (低)：  vision / multi-modal                          ← 部分支持，schema 不一致
Level 6 (低)：  reasoning / thinking                          ← 几乎没有跨供应商共识
Level 7 (无)：  Responses / Assistants                        ← 几乎没人实现
```

跨供应商时**只能依赖 Level 1-2**——再高就要按 provider 写适配代码。

### 9.4 vLLM——自建 OpenAI 兼容推理

```bash
pip install vllm
vllm serve meta-llama/Llama-3.1-70B-Instruct --port 8000
# 暴露 http://localhost:8000/v1/chat/completions
```

```go
client := openai.NewClient(
    option.WithBaseURL("http://localhost:8000/v1"),
    option.WithAPIKey("dummy"),  // vLLM 不验证
)
```

vLLM 是事实标准——支持 PagedAttention、continuous batching、speculative decoding。**自建大模型推理首选**。

### 9.5 OpenRouter / litellm——一套接口跑所有

OpenRouter（hosted）：

```go
client := openai.NewClient(
    option.WithBaseURL("https://openrouter.ai/api/v1"),
    option.WithAPIKey(os.Getenv("OPENROUTER_API_KEY")),
)
client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F("anthropic/claude-opus-4-8"),  // 用 OpenAI 协议调 Claude
})
```

litellm（self-hosted gateway）：本地起一个网关，前端用 OpenAI 协议，后端路由到 OpenAI / Anthropic / Bedrock / 千问。详见 **A11 — LLM Gateway**。

### 9.6 国产模型的 OpenAI 兼容

| 厂商 | baseURL | 主流模型 |
|---|---|---|
| DeepSeek | api.deepseek.com/v1 | deepseek-chat (V3.x), deepseek-reasoner (R1) |
| 阿里千问 | dashscope.aliyuncs.com/compatible-mode/v1 | qwen-max, qwen-plus, qwen-turbo, qwen2.5-72b-instruct |
| 智谱 | open.bigmodel.cn/api/paas/v4 | glm-4.6, glm-4-flash |
| 月之暗面 Kimi | api.moonshot.cn/v1 | moonshot-v1-128k, kimi-latest |
| 字节豆包 | ark.cn-beijing.volces.com/api/v3 | doubao-pro / doubao-lite |
| 百度文心 | qianfan.baidubce.com/v2 | ernie-4.0-turbo / ernie-x1 |

注意点：

- **每家都有自己的 tool/function call 细节不一致**（参数顺序、返回结构）
- 千问 / 智谱 的 `tools` schema 部分字段与 OpenAI 不同
- 流式 chunk 字段顺序 / id 格式有差异
- 部分供应商不支持 `parallel_tool_calls` / `tool_choice` 复杂值
- **特定供应商兼容性最佳**：DeepSeek 与 OpenAI Chat Completions 一致度最高

### 9.7 Anthropic 的 OpenAI 兼容 endpoint

```
POST https://api.anthropic.com/v1/chat/completions
```

把请求映射成 Anthropic 内部 Messages API。**用于迁移用户**，不是 Anthropic 推荐的主路径。功能完整度 < 直接 Messages API（无 cache_control、无 batch、无 thinking）。

---

## 第十章：模型选型——GPT-5.5 / GPT-5-mini / 4o-mini

### 10.1 2026 年 5 月主力图谱

> 2026-04-23 GPT-5.5 发布（API 2026-04-24 上线），成为当前旗舰；推理能力已并入 GPT-5.x 的 thinking/reasoning 模式。整条 o 系列（o1/o3/o3-mini/o4-mini）已于 2026-02-13 从 ChatGPT 退役，o3 API 字符串处于 sunset 倒计时，**不应作为新项目推荐**。

```
GPT-5.5       （顶配，含推理）~ Opus 4.8 / Gemini 3.1 Pro 同档
GPT-5 mini    （平衡）  ~ Sonnet 4.6 / Gemini 3 Flash
GPT-5 nano    （极便宜） ~ Haiku 4.5 / Gemini 3.1 Flash Lite
（推理）       已并入 GPT-5.x thinking/reasoning 模式，o3/o3-mini 已退役
gpt-4o-mini   （遗留多模态便宜）大幅降价仍在用
```

### 10.2 价格参考（2026-05，USD / 1M tokens）

| 模型 | input | cached input | output |
|---|---|---|---|
| `gpt-5.5`（旗舰） | ~5 | ~0.5 | ~30 |
| `gpt-5` | ~10 | ~1 | ~40 |
| `gpt-5-mini` | ~0.6 | ~0.06 | ~2.4 |
| `gpt-5-nano` | ~0.1 | ~0.01 | ~0.4 |
| `o3`（已退役 / sunset，仅供参考） | ~30 | ~7.5 | ~120 |
| `o3-mini`（已退役，仅供参考） | ~3 | ~0.75 | ~12 |
| `gpt-4o-mini` | ~0.15 | ~0.075 | ~0.6 |

> 实际价格以 [openai.com/pricing](https://openai.com/pricing) 为准。

### 10.3 何时选什么

- **chatbot / 通用问答**：gpt-5-mini（性价比之王）
- **复杂 Agent / 高准代码**：gpt-5.5 或 gpt-5
- **数学 / 形式推理 / 谜题**：gpt-5.5 / gpt-5 的 thinking/reasoning 模式（o3/o3-mini 已退役）
- **高吞吐分类、批量打标**：gpt-5-nano / 4o-mini
- **长文档**：gpt-5（400k context）
- **多模态（图像 + 音频）**：gpt-4o（多模态 GA 时间最长） / gpt-5

### 10.4 reasoning 模型的特殊性

> 下面以 `o3-mini` 为例说明 reasoning 模型的 API 行为差异。注意 o3/o3-mini 已退役，新项目应改用 GPT-5.x 的 thinking/reasoning 模式（同样具备下列特性）。

```go
client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F("o3-mini"),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.UserMessage("证明 sqrt(2) 不是有理数"),
    }),
    MaxCompletionTokens: openai.Int(8000),
    ReasoningEffort:     openai.F("medium"),
})
```

注意：

- **`temperature` 等多数参数被忽略**——reasoning model 不接受
- **不支持流式 reasoning 内容**——只能流式正式回复
- **`max_completion_tokens` 必须够大**——内含 reasoning + 正式输出
- **不支持 streaming + tool_use 组合**（某些 model 限制）

---

## 第十一章：生产实践

### 11.1 配置层

```go
type OpenAIConfig struct {
    APIKey         string
    BaseURL        string             // 默认 openai 官方；切到兼容供应商时改
    DefaultModel   string
    FallbackModel  string
    Organization   string             // 多组织时
    Project        string             // GPT-5 起支持的 project header
    RequestTimeout time.Duration
    MaxRetries     int
    MaxConcurrency int
}
```

### 11.2 客户端构造

```go
client := openai.NewClient(
    option.WithAPIKey(cfg.APIKey),
    option.WithBaseURL(cfg.BaseURL),
    option.WithOrganization(cfg.Organization),
    option.WithProject(cfg.Project),
    option.WithMaxRetries(cfg.MaxRetries),
    option.WithRequestTimeout(cfg.RequestTimeout),
    option.WithHTTPClient(&http.Client{
        Transport: &http.Transport{
            MaxIdleConnsPerHost: 100,
            IdleConnTimeout:     90 * time.Second,
        },
    }),
)
```

### 11.3 限流

OpenAI rate-limit headers：

```
x-ratelimit-limit-requests:   3500
x-ratelimit-remaining-requests: 3499
x-ratelimit-reset-requests:    6ms
x-ratelimit-limit-tokens:     180000
x-ratelimit-remaining-tokens: 179975
x-ratelimit-reset-tokens:     8ms
```

不像 Anthropic 用 RFC3339 时间，OpenAI 用相对时间字符串（"6ms" / "8s" / "1m"）。429 时尊重 `retry-after`（秒）。

### 11.4 重试

```go
func callWithRetry(ctx context.Context, params openai.ChatCompletionNewParams) (*openai.ChatCompletion, error) {
    var lastErr error
    for i := 0; i < 4; i++ {
        resp, err := client.Chat.Completions.New(ctx, params)
        if err == nil { return resp, nil }
        lastErr = err
        
        var apiErr *openai.Error
        if errors.As(err, &apiErr) {
            switch apiErr.StatusCode {
            case 429, 500, 502, 503:
                wait := time.Duration(1<<i) * time.Second
                if ra := apiErr.Header().Get("Retry-After"); ra != "" {
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

### 11.5 自动 prompt caching

OpenAI 在 GPT-4o 起引入**自动 prompt caching**——无需 client 标注、无 cache_control 字段，OpenAI 内部检测长 prompt（≥ 1024 tokens 的稳定前缀）并自动缓存：

- 命中：input price × 0.5（GPT-5 后期改为 0.1x，详见官网当前价格）
- 写入：免费（不像 Anthropic 1.25x）
- TTL：5-10 分钟（不固定，OpenAI 内部决定）
- 监控：`Usage.PromptTokensDetails.CachedTokens`

**与 Anthropic 的关键区别**：

- 透明，无需代码改动
- 缓存粒度 OpenAI 自己决定，开发者无控制权
- 价格优惠（0.5x / 0.1x）但**不像 Anthropic 那么便宜**（Anthropic 0.1x）
- 不像 Anthropic 那样能"主动控制"长 prompt 命中

实践建议：

- **保持 prompt 前缀稳定**（动态时间戳放后面）
- 业务侧不要乱改 system prompt
- 监控 `cached_tokens / prompt_tokens` 比率

### 11.6 Batch API

```go
// 1. 准备 JSONL 文件
type BatchRequest struct {
    CustomID string                 `json:"custom_id"`
    Method   string                 `json:"method"` // "POST"
    URL      string                 `json:"url"`    // "/v1/chat/completions"
    Body     map[string]any         `json:"body"`
}
var buf bytes.Buffer
for i, prompt := range prompts {
    json.NewEncoder(&buf).Encode(BatchRequest{
        CustomID: fmt.Sprintf("req-%d", i),
        Method:   "POST",
        URL:      "/v1/chat/completions",
        Body: map[string]any{
            "model": "gpt-5-mini",
            "messages": []map[string]string{{"role": "user", "content": prompt}},
            "max_completion_tokens": 256,
        },
    })
}

// 2. 上传文件
file, _ := client.Files.New(ctx, openai.FileNewParams{
    File:    openai.F[io.Reader](&buf),
    Purpose: openai.F(openai.FilePurposeBatch),
})

// 3. 创建 batch
batch, _ := client.Batches.New(ctx, openai.BatchNewParams{
    InputFileID:      openai.F(file.ID),
    Endpoint:         openai.F(openai.BatchNewParamsEndpointV1ChatCompletions),
    CompletionWindow: openai.F("24h"),
})

// 4. 轮询
for batch.Status != "completed" {
    time.Sleep(60 * time.Second)
    batch, _ = client.Batches.Get(ctx, batch.ID)
    if batch.Status == "failed" || batch.Status == "expired" {
        return errors.New(batch.Status)
    }
}

// 5. 下载 output 文件
out, _ := client.Files.Content(ctx, batch.OutputFileID)
defer out.Close()
scanner := bufio.NewScanner(out)
for scanner.Scan() {
    // 每行一个 JSON 含 custom_id + response
}
```

Batch 与 Anthropic 类似：**input + output 半价**，24h 异步处理。

### 11.7 完整生产模板

```go
type Provider struct {
    client       openai.Client
    cfg          OpenAIConfig
    sem          *semaphore.Weighted
    log          *slog.Logger
}

func (p *Provider) Chat(ctx context.Context, conv *Conversation) (*Reply, error) {
    if err := p.sem.Acquire(ctx, 1); err != nil { return nil, err }
    defer p.sem.Release(1)
    
    ctx, cancel := context.WithTimeout(ctx, p.cfg.RequestTimeout)
    defer cancel()
    
    params := p.buildParams(conv)
    start := time.Now()
    
    resp, err := callWithRetry(ctx, params)
    if err != nil {
        // fallback
        params.Model = openai.F(p.cfg.FallbackModel)
        resp, err = callWithRetry(ctx, params)
        if err != nil { return nil, err }
    }
    
    p.log.Info("openai call",
        "model", resp.Model,
        "prompt_tokens", resp.Usage.PromptTokens,
        "cached_tokens", resp.Usage.PromptTokensDetails.CachedTokens,
        "completion_tokens", resp.Usage.CompletionTokens,
        "reasoning_tokens", resp.Usage.CompletionTokensDetails.ReasoningTokens,
        "finish", resp.Choices[0].FinishReason,
        "latency", time.Since(start),
    )
    return toReply(resp), nil
}
```

---

## 第十二章：陷阱清单

### 1. 在 GPT-5 上传 `max_tokens`

GPT-5 系列改用 `max_completion_tokens`。老模型仍接 `max_tokens`。混着写时一段代码两个 case 是常见 bug。

### 2. tool message 不紧跟 assistant tool_calls

API 严格要求 assistant(tool_calls) → tool → tool → ... → 才能继续 user。中间插入 user 会 400。

### 3. parallel tool calls 一个不回填

模型并行调了 3 个 tool，你只回填了 2 个 tool result → 400 "missing tool response"。必须全部回填，哪怕错误也要回填 is_error。

### 4. function arguments 字符串不解析

```go
fc.Function.Arguments    // 这是 string！
// 必须 json.Unmarshal([]byte(fc.Function.Arguments), &args)
```

### 5. structured output 用 strict: false

非严格模式不保证 schema 符合——模型可能加字段、删字段。生产**永远 strict: true**（前提是 schema 满足 strict 模式约束）。

### 6. 把 Assistants v2 当 Responses 同款

Assistants v2 是过渡产物；新项目用 Responses。新人最易踩——以为"assistant"听起来更"agent"就选了，结果掉进维护陷阱。

### 7. previous_response_id 跨 store=false

```go
PreviousResponseID: openai.F(r1.ID),
Store:              openai.Bool(false), // 不会保存这个新 response
```

`store=false` 时**不会**生成可被下次引用的 response_id（更准确说：本次 response 不持久化）。设计长会话要明白哪些应当持久化。

### 8. reasoning model 设 temperature

```go
client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model:       openai.F("o3-mini"),
    Temperature: openai.F(0.7),  // 被忽略，可能报警告
})
```

o 系列基本不接受 temperature / top_p。

### 9. base_url 末尾 / 不一致

```go
option.WithBaseURL("https://api.deepseek.com/v1/")   // 末尾 /
option.WithBaseURL("https://api.deepseek.com/v1")    // 不带 /
```

某些兼容服务对此敏感。**遵从供应商文档**。

### 10. 在国产兼容 endpoint 上用 OpenAI 特性

```go
ResponseFormat: openai.F(openai.ResponseFormatJSONSchemaParam{...}),
```

不是所有"OpenAI 兼容"供应商支持 strict json_schema / parallel tool calls / vision。**先读供应商文档**。

### 11. 流式忘了消费 `[DONE]`

```go
for {
    line, _ := reader.ReadString('\n')
    if line == "data: [DONE]" { break }
}
```

不处理 [DONE] → 客户端永远卡在等下一个 chunk。Go SDK 内部处理了，自己写 HTTP 时记住。

### 12. 多组织 / 多项目混淆

GPT-5 起强烈推荐用 project key + project header，便于计费分摊：

```go
option.WithProject("proj_xxx"),
```

否则所有调用都计入"default project"——账单一片混乱。

### 13. tool_calls 流式拼接错误

```
delta_1: tool_calls: [{index:0, function: {name: "get_w", arguments: ""}}]
delta_2: tool_calls: [{index:0, function: {arguments: "{\"ci"}}]
delta_3: tool_calls: [{index:0, function: {arguments: "ty\": "}}]
delta_4: tool_calls: [{index:0, function: {arguments: "\"NYC\"}"}}]
```

`arguments` 是**逐字符流**——需要按 index 累加。新手以为每个 chunk 是完整 JSON → 解析失败。

---

## 第十三章：2026 现状

### 13.1 GPT-5 时代

GPT-5 系列在 2025 年发布，2026 年 5 月已稳定（当前旗舰为 2026-04-23 发布的 GPT-5.5）：

- main: 复杂推理、长上下文（400k）
- mini: 性价比主力，绝大部分应用场景
- nano: 极便宜，分类、抽取、批量

特性：

- 内置 `developer` 角色（更高优先级 system 指令）
- prompt caching 默认开启（透明）
- structured output 大幅改进（嵌套 / enum 支持更好）
- 多模态原生支持（图像、音频）
- Responses API first（Chat Completions 在跟进，但新特性优先 Responses）

### 13.2 三套 API 的命运

```
Chat Completions  ── 长期存在（兼容生态需要）
Responses         ── 主推、扩展、统一
Assistants v2     ── 弃用倒计时（最早 2026 中关停）
```

Anthropic 是单一 Messages API，对比之下 OpenAI 的多 API 历史包袱是个独特复杂度。

### 13.3 OpenAI vs Anthropic vs Gemini 选型经验

| 维度 | OpenAI | Anthropic | Gemini |
|---|---|---|---|
| 最强 reasoning | GPT-5.5 / GPT-5 thinking | Claude Opus 4.8 | Gemini 3 Pro Deep Think |
| 性价比主力 | GPT-5 mini | Sonnet 4.6 | Gemini 3 Flash |
| 长上下文 | 400k | 1M (Sonnet 4.6 beta) | 1M (2.5 Pro)；2M 在 Gemini 3.1 Pro |
| 代码 | GPT-5.5、GPT-5 | Opus 4.8 业内顶尖 | Gemini 3 Pro 强 |
| 多模态 | 图像 / 音频 / 视频 / TTS / Whisper / Realtime | 图像（无音频生成） | 图像 / 音频 / 视频原生 |
| 内置 tools | web/file_search/code_interpreter/computer_use | （需自建 / 通过 MCP） | grounding (Google 搜索) |
| API 数 | 3 套（Chat/Resp/Assist） | 1 套（Messages） | 2 套（v1beta/Gemini API） |
| 兼容生态 | OpenAI 协议是事实标准 | 自家协议；提供 OAI 兼容 endpoint | 自家协议；OpenAI 兼容子集 |
| 企业级支持 | 强（Azure OpenAI） | 强（Bedrock / Vertex） | 强（Vertex AI） |

**2026 年常见架构**：多模型路由 + LiteLLM/Portkey 兜底。**单一供应商绑死已经少见**——成本、可靠性、特定能力都鼓励多源。

### 13.4 兼容生态的政治

```
OpenAI Chat Completions schema ≈ LLM 接入业界标准（事实，非官方）
                ↓
       从 OpenAI 协议出发，
       想要更高质量 → 上 OpenAI Responses / Anthropic / Gemini 各家原生 API
       想要更低成本 → 切 DeepSeek / 千问 / 自建 vLLM
```

开发者群体对"OpenAI 协议"的依赖已经超过 OpenAI 自己——他们想替换的反而推不动。**这是协议的成功，也是技术债**。

### 13.5 Realtime API & Voice

OpenAI 2024 起推出 **Realtime API**（WebSocket，低延迟语音对话），2025-2026 GA。Go SDK 已支持：

```go
client.Realtime.Connect(ctx)
```

适合实时电话客服、AI 助手、语音 Agent。其他供应商在追赶（Anthropic、Gemini 各有方案，但 OpenAI 是先发）。

---

## 第十四章：练习题

**练习 1**：解释为什么 OpenAI 选 `previous_response_id` 而不是像 Anthropic 完全无状态。两种设计的 tradeoff。

**练习 2**：以下代码哪里有问题？

```go
messages := []openai.ChatCompletionMessageParamUnion{
    openai.UserMessage("北京天气如何"),
    openai.AssistantMessage(openai.ChatCompletionAssistantMessageParam{
        ToolCalls: openai.F([]openai.ChatCompletionMessageToolCallParam{{
            ID: openai.F("call_abc"),
            Function: openai.F(openai.ChatCompletionMessageToolCallFunctionParam{
                Name:      openai.F("get_weather"),
                Arguments: openai.F(`{"city":"BJ"}`),
            }),
        }}),
    }),
    openai.UserMessage("那上海呢"),
}
```

**练习 3**：你要写一个客服 bot，要求：（1）能调用内部 API 查询订单；（2）输出严格 JSON `{intent, action, params}`；（3）流式回前端；（4）支持多 provider。设计 architecture。

**练习 4**：写一个函数 `extractEntities(ctx, text) ([]Entity, error)`，用 structured output 抽取人名、地名、组织名。

**练习 5**：解释 OpenAI 的"自动 prompt caching"为什么对开发者更省心，但同时也让"主动优化"变难。

**练习 6**：你公司同时用 OpenAI 和 Anthropic。如何设计统一的 LLM client 接口，让上层业务无感切换？写出 Go 接口 + 两个适配器骨架。

**练习 7**：reasoning model（o3）和普通 model（GPT-5）在哪些**API 字段**上行为不同？

**练习 8**：用 Responses API 写一个 Agent 循环：用户问 "在 OpenAI 文档里搜 prompt caching 然后总结"。要用 `web_search_preview` 内置工具。

---

## 参考答案

**练习 1**：

- **OpenAI 选 previous_response_id**：把对话状态托管在 OpenAI 服务端。优点：客户端简单（只发增量）、服务器可优化（cache、内部 KV reuse）、Agent 多步任务自然延续。缺点：状态不可控（你看不到内部上下文）、跨 provider 不可能、改写历史 / 修剪不便、隐私合规风险（数据在 OpenAI 留 30 天）。
- **Anthropic 全无状态**：每次发完整 history。优点：客户端完全控制、易迁移、可任意编辑历史、合规友好。缺点：每次发完整 history（贵）——依赖 prompt caching 缓解；客户端复杂度高。
- **Tradeoff**：本质是"控制权 vs 便利"。生产长任务推荐 Anthropic 模式（自己管），快速原型 / 单一供应商可以用 OpenAI 模式（省心）。

**练习 2**：assistant 发出了 tool_calls 但**没有回填 tool message**就继续 user message。API 会 400 "the following tool_call_ids did not have response messages: call_abc"。

修：在 assistant message 后插入 tool message：

```go
openai.ToolMessage("call_abc", `{"temp": 10, "condition": "sunny"}`),
```

**练习 3**：

架构：

```
前端 ──SSE──> 自研 Gateway ──Chat Completions 协议──> {OpenAI / Anthropic compat / DeepSeek}
                  │
                  ├ tool 注册：get_order(order_id), check_status(...)
                  ├ structured output：response_format json_schema strict
                  └ 路由：按 cost / latency / quality 选 provider
```

实现要点：

1. **协议层**：用 Chat Completions schema（最广兼容），不用 Responses（多 provider 不支持）
2. **structured output**：所有 provider 都试 json_schema strict；不支持的退到 json_object + 自己 validate
3. **tool 循环**：自己写多轮 loop，不依赖 Responses 的 built-in
4. **流式**：Gateway 解析上游 SSE chunk → 自己重新封装成内部 SSE 给前端（统一格式）
5. **provider 抽象**：见练习 6

**练习 4**：

```go
type Entity struct {
    Type string `json:"type"` // "PERSON", "LOCATION", "ORGANIZATION"
    Text string `json:"text"`
    Span [2]int `json:"span"` // [start, end] char offset
}

func extractEntities(ctx context.Context, text string) ([]Entity, error) {
    schema := map[string]any{
        "name":   "entities",
        "strict": true,
        "schema": map[string]any{
            "type": "object",
            "properties": map[string]any{
                "entities": map[string]any{
                    "type": "array",
                    "items": map[string]any{
                        "type": "object",
                        "properties": map[string]any{
                            "type": map[string]any{"type": "string", "enum": []string{"PERSON", "LOCATION", "ORGANIZATION"}},
                            "text": map[string]any{"type": "string"},
                            "span": map[string]any{"type": "array", "items": map[string]any{"type": "integer"}, "minItems": 2, "maxItems": 2},
                        },
                        "required":             []string{"type", "text", "span"},
                        "additionalProperties": false,
                    },
                },
            },
            "required":             []string{"entities"},
            "additionalProperties": false,
        },
    }
    resp, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
        Model: openai.F(openai.ChatModelGPT5Mini),
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.SystemMessage("从文本中抽取实体，输出 JSON。span 是字符偏移。"),
            openai.UserMessage(text),
        }),
        ResponseFormat: openai.F[openai.ChatCompletionNewParamsResponseFormatUnion](
            openai.ResponseFormatJSONSchemaParam{
                Type:       openai.F(openai.ResponseFormatJSONSchemaTypeJSONSchema),
                JSONSchema: openai.F(schema),
            },
        ),
    })
    if err != nil { return nil, err }
    var result struct{ Entities []Entity `json:"entities"` }
    if err := json.Unmarshal([]byte(resp.Choices[0].Message.Content), &result); err != nil { return nil, err }
    return result.Entities, nil
}
```

**练习 5**：

省心：

- 不需要在 prompt 里标记 cache 段
- 不需要算 cache write 是否值（OpenAI 全免）
- 升级、改 schema 不破坏 caching

变难：

- **粒度不可控**：你不知道 OpenAI 把哪段当 cache key
- **TTL 不可控**：可能 5 min 内被驱逐
- **不能预热**：Anthropic 你可以主动写一次让后续命中；OpenAI 无 API
- **monitoring 不够细**：只知道 cached_tokens 总数，不知道命中段在哪
- **跨 region 不一致**：不同 region 的 cache 不共享

**练习 6**：

```go
type LLMClient interface {
    Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
    ChatStream(ctx context.Context, req *ChatRequest) (StreamReader, error)
}

type ChatRequest struct {
    Model       string
    Messages    []Message
    Tools       []Tool
    MaxTokens   int
    Temperature float32
    JSONSchema  *Schema
}

type Message struct {
    Role    string // system, user, assistant, tool
    Content string
    Name    string // tool name
    ToolCallID string
    ToolCalls []ToolCall
}

// ----------- 适配器 -----------

type OpenAIAdapter struct{ c openai.Client }

func (a *OpenAIAdapter) Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error) {
    msgs := convertMessagesToOAI(req.Messages)
    params := openai.ChatCompletionNewParams{
        Model:               openai.F(req.Model),
        Messages:            openai.F(msgs),
        MaxCompletionTokens: openai.Int(int64(req.MaxTokens)),
    }
    if req.JSONSchema != nil { /* ... */ }
    if len(req.Tools) > 0    { /* ... */ }
    resp, err := a.c.Chat.Completions.New(ctx, params)
    if err != nil { return nil, err }
    return convertOAIToResponse(resp), nil
}

type AnthropicAdapter struct{ c anthropic.Client }

func (a *AnthropicAdapter) Chat(ctx context.Context, req *ChatRequest) (*ChatResponse, error) {
    msgs, system := convertMessagesToAnthropic(req.Messages)
    params := anthropic.MessageNewParams{
        Model:     anthropic.F(req.Model),
        MaxTokens: anthropic.F(int64(req.MaxTokens)),
        System:    anthropic.F(system),
        Messages:  anthropic.F(msgs),
    }
    resp, err := a.c.Messages.New(ctx, params)
    if err != nil { return nil, err }
    return convertAnthropicToResponse(resp), nil
}
```

关键点：

- `Message.Role` 设计要兼容两边（`tool` vs `tool_result`、`system` 在 OpenAI 在 messages、在 Anthropic 在顶层）
- structured output：OpenAI 用 response_format；Anthropic 用 tool_use 强制 schema
- streaming：两边 event 类型不同，要在适配层统一映射成内部 event

**练习 7**：

| 字段 | GPT-5 | o3 |
|---|---|---|
| `temperature` | 接受 | **忽略** |
| `top_p` | 接受 | **忽略** |
| `presence_penalty` | 接受 | 忽略 |
| `frequency_penalty` | 接受 | 忽略 |
| `max_completion_tokens` | 接受 | **必须够大**（含 reasoning） |
| `reasoning_effort` | 接受（GPT-5 也有 reasoning） | 接受 |
| `tools` / parallel | 接受 | 某些模型不支持 parallel |
| `response_format` json_schema | 接受 | 接受 |
| `stream` 含 tool_use | 支持 | 部分模型受限 |

**练习 8**：

```go
r, err := client.Responses.New(ctx, openai.ResponseNewParams{
    Model: openai.F("gpt-5"),
    Input: openai.F[any]("到 OpenAI 文档搜索 'prompt caching'，然后总结主要内容"),
    Tools: openai.F([]openai.ResponseTool{
        {Type: openai.F("web_search_preview")},
    }),
})
if err != nil { return err }

fmt.Println(r.OutputText)  // 已含总结 + 引用
// 详细 item 历史
for _, item := range r.Output {
    switch x := item.AsAny().(type) {
    case openai.ResponseFunctionWebSearch:
        log.Printf("searched: %s", x.Action)
    case openai.ResponseOutputMessage:
        log.Printf("said: %s", x.Content)
    }
}
```

Responses 内置 web_search 完全在 OpenAI 服务端运行——你不需要部署搜索代理 / 不需要 RAG 索引。代价是只能搜 OpenAI 选择的搜索引擎结果集。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 三套 API | Chat（兼容广）/ Responses（主推）/ Assistants（弃用倒计时） |
| Go SDK 选型 | `openai-go` 官方 / `sashabaranov` 社区 / 直接 HTTP |
| Chat Completions | 无状态 messages 数组、tool_calls、response_format |
| Responses | 可选 previous_response_id、内置 tools、reasoning effort |
| Function calling | arguments 是字符串、parallel 必须全回填、tool message 紧跟 |
| Structured output | response_format json_schema strict（推荐） |
| Streaming | data chunk + `[DONE]`（Chat）/ 命名事件（Responses） |
| 兼容生态 | 换 baseURL 即可；只能依赖基础特性（messages + tools） |
| 模型选型 | GPT-5.5 旗舰 / GPT-5 顶 / mini 平衡 / nano 便宜（o3 推理已退役，并入 GPT-5.x thinking） |
| Prompt caching | 自动启用，无 cache_control，监控 cached_tokens |
| Batch | 半价 24h，JSONL 文件 → POST |
| 限流 | RPM + TPM 头，retry-after 字符串时长 |
| 与 Anthropic 对比 | 三套 vs 一套、内置 tools vs 自管、有状态 vs 无状态 |
| 2026 现状 | GPT-5 系列稳定、Responses 主推、Realtime GA |

铁律：

- 新项目用 Responses（OpenAI 单一）或 Chat Completions（多 provider）
- 别碰 Assistants v2
- structured output 一律 strict: true
- tool_calls 必须每个回填
- streaming tool arguments 按 index 累加
- max_completion_tokens 在 GPT-5 / o3 必填
- 多供应商路由用 LLM Gateway（A11）

下一篇 **A03 — 精通 Tokenizer、上下文窗口与计费** 将拆开 BPE 编码原理、tiktoken / claude_tokenizer、context window 演进、Cost 模型与优化策略。

---
