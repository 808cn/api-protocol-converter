# 三端点协议互转编码指南（仅用于指导编码）

本文只用于指导以下三个端点之间的协议转换实现，不讨论 Replit 部署、门户页面、发布流程。

目标端点只有三个：

- OpenAI `POST /v1/chat/completions`
- OpenAI `POST /v1/responses`
- Anthropic `POST /v1/messages`

如果代理层需要支持三者之间的相互转换，**不要写 6 组两两直连转换器**，而应先归一化到统一中间结构，再从中间结构编码回目标端点。

---

## 0. 文档适用范围

本文只覆盖三端点都能相对稳定互转、并且最容易影响编码正确性的公共子集：

- 文本输入与文本输出
- `system / instructions` 类指令
- 自定义函数工具
- `tool_choice`
- 多轮工具调用
- 工具结果回填
- 流式文本增量
- 流式函数参数增量
- 停止原因映射
- usage 基本映射

本文默认 **不把以下能力纳入第一版互转范围**，除非你单独做能力分层：

- OpenAI Responses 的内置工具、reasoning 项、MCP 专属项
- Anthropic 的 thinking、server tools、`pause_turn`
- 各端点的音频专属输出
- 各端点的 provider 专属实验字段
- 无法在另外两端稳定表达的高级结构化输出细节

结论：**先实现“公共交集”，再做能力扩展**。

---

## 1. 先定义统一中间结构（IR）

建议内部统一为三类结构：

- `RequestIR`：入站请求归一化结果
- `ResponseIR`：非流式响应归一化结果
- `StreamEventIR`：流式事件归一化结果

这样可以把问题从：

- `chat -> messages`
- `messages -> chat`
- `chat -> responses`
- `responses -> chat`
- `messages -> responses`
- `responses -> messages`

变成：

- `chat -> IR`
- `responses -> IR`
- `messages -> IR`
- `IR -> chat`
- `IR -> responses`
- `IR -> messages`

编码复杂度会明显下降。

---

## 2. 推荐的 `RequestIR`

建议把三种端点的请求先统一成下面这种结构：

```ts
interface RequestIR {
  source: "chat" | "responses" | "messages";
  model: string;
  system: TextBlock[];
  messages: IRMessage[];
  tools: IRTool[];
  toolChoice: IRToolChoice | null;
  allowParallelToolCalls: boolean;
  stream: boolean;
  maxOutputTokens?: number;
  responseRef?: {
    previousResponseId?: string;
  };
}

interface IRMessage {
  role: "user" | "assistant";
  blocks: IRBlock[];
}

type IRBlock =
  | { type: "text"; text: string }
  | { type: "tool_call"; id: string; name: string; argumentsText: string }
  | { type: "tool_result"; toolCallId: string; contentText: string; isError?: boolean };

interface IRTool {
  name: string;
  description?: string;
  inputSchema?: Record<string, unknown>;
  strict?: boolean;
}

type IRToolChoice =
  | { type: "none" }
  | { type: "auto" }
  | { type: "required" }
  | { type: "tool"; name: string };

interface TextBlock {
  type: "text";
  text: string;
}
```

### IR 的补充约定

为了让三端互转更稳定，建议把下面这些约定直接固化到代码里：

1. `blocks[]` 必须保序  
   所有文本、工具调用、工具结果都按原始出现顺序进入 IR，不能重排。

2. 相邻文本块可合并  
   归一化后如果出现连续 `text block`，可以在 IR 层合并，减少回编码复杂度。

3. `argumentsText` 在 IR 中始终保存为字符串  
   只有在编码到 Claude `tool_use.input` 时才 `JSON.parse`；平时不要在 IR 中来回对象化。

4. 空内容不要强行造块  
   - Chat `assistant.content = null` 且有 `tool_calls` 时，不要生成空文本块
   - Messages `content = []` 时，不要补空字符串
   - Responses 若某个 output item 没文本，就不要补假的 `text block`

5. `system[]` 保留顺序，但回编码时允许合并  
   - 编码到 Chat：可拆成多条 `system` 消息，或合并成一条
   - 编码到 Messages：建议合并成顶层一个 `system`
   - 编码到 Responses：建议合并成一个 `instructions`
   - 合并时统一用 `\n\n` 连接

### 为什么 `IRMessage.role` 只保留 `user/assistant`

因为：

- Chat Completions 的 `role: "tool"`
- Responses 的 `function_call_output`
- Messages 的 `tool_result`

本质上都属于“**用户把工具结果回填给模型**”这一类语义。  
为了统一三种端点，建议在 IR 中把工具结果都归并为：

- `role: "user"`
- `blocks[].type = "tool_result"`

这样后续转回 Claude 时最自然，转回 Chat / Responses 时也更容易恢复。

---

## 3. 三端点请求归一化规则

## 3.1 `chat/completions -> RequestIR`

### 字段映射

| Chat 字段 | RequestIR |
|---|---|
| `model` | `model` |
| `stream` | `stream` |
| `max_tokens` | `maxOutputTokens` |
| `tools[].function` | `tools[]` |
| `tool_choice` | `toolChoice` |
| `parallel_tool_calls` | `allowParallelToolCalls` |
| `messages[role=system]` | `system[]` |
| `messages[role=user/assistant/tool]` | `messages[]` |

### 归一化规则

1. 把所有 `role: "system"` 消息提取到 `system[]`
2. 普通 `user/assistant` 文本消息转成 `blocks: [{type:"text"}]`
3. 若 assistant 消息里带 `tool_calls`：
   - 生成一个 `role: "assistant"` 的 `IRMessage`
   - `tool_calls[].id -> block.id`
   - `tool_calls[].function.name -> block.name`
   - `tool_calls[].function.arguments -> block.argumentsText`
4. 若 assistant 同时有文本和 `tool_calls`：
   - 文本保留为 `text block`
   - 工具调用保留为 `tool_call block`
   - 顺序按原消息语义输出到 `blocks[]`
5. 若收到 `role: "tool"`：
   - 转成 `role: "user"`
   - `blocks = [{ type:"tool_result", toolCallId: tool_call_id, contentText: content }]`

### `tool_choice` 映射到 IR

建议统一成：

- `"none" -> {type:"none"}`
- `"auto" -> {type:"auto"}`
- `"required" -> {type:"required"}`
- `{"type":"function","function":{"name":"x"}} -> {type:"tool",name:"x"}`
- `{"type":"allowed_tools", ...}`
  - 第一版建议直接拒绝，或降级到 `required/auto`
  - 不建议静默忽略

### `tools` 映射到 IR

OpenAI Chat:

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Get weather",
    "parameters": {"type":"object"},
    "strict": true
  }
}
```

归一化为：

```json
{
  "name": "get_weather",
  "description": "Get weather",
  "inputSchema": {"type":"object"},
  "strict": true
}
```

---

## 3.2 `responses -> RequestIR`

这是三者里最容易写错的一种，因为 Responses 有两套上下文表达方式：

- 显式 `input`
- 隐式 `previous_response_id`

### 字段映射

| Responses 字段 | RequestIR |
|---|---|
| `model` | `model` |
| `stream` | `stream` |
| `max_output_tokens` | `maxOutputTokens` |
| `tools[]` | `tools[]` |
| `tool_choice` | `toolChoice` |
| `parallel_tool_calls` | `allowParallelToolCalls` |
| `instructions` | `system[]` |
| `input` | `messages[]` |
| `previous_response_id` | `responseRef.previousResponseId` |

### `instructions` 处理

建议统一吸收到 `system[]`。  
如果 `instructions` 是字符串：

```json
{"type":"text","text":"..."}
```

如果是数组，则逐项拆成文本块。

### `input` 处理

#### 情况 A：`input` 是字符串

等价于：

```json
{
  "role": "user",
  "blocks": [{"type":"text","text":"input字符串"}]
}
```

#### 情况 B：`input` 是数组

常见有三类项：

1. `message`
2. `function_call_output`
3. 其他 provider / tool 专属项

### `message` 项归一化

Responses 常见输入消息：

```json
{
  "role": "user",
  "content": [
    {"type":"input_text","text":"hello"}
  ]
}
```

归一化规则：

- `role: system/developer` -> 归并到 `system[]`
- `role: user` -> `IRMessage(role="user")`
- `role: assistant` -> `IRMessage(role="assistant")`
- `input_text.text -> text block`
- 第一版如果遇到 `input_image` / `input_file`，建议明确标记为“暂不支持”或单独扩展，不要默默丢弃

### `function_call_output` 归一化

Responses 工具回填：

```json
{
  "type": "function_call_output",
  "call_id": "call_001",
  "output": "{\"temperature\":23}"
}
```

统一转成：

```json
{
  "role": "user",
  "blocks": [
    {"type":"tool_result","toolCallId":"call_001","contentText":"{\"temperature\":23}"}
  ]
}
```

### `previous_response_id` 的处理原则

这是 Responses 与另外两端最大的结构差异之一。

- Chat / Messages 默认要求客户端显式提供完整上下文
- Responses 可以只给 `previous_response_id`，让 OpenAI 服务端自己续上下文

因此：

1. 如果你是 **Responses 入站 -> 其他端点出站**
   - 且只有 `previous_response_id`，没有足够的 `input` 历史
   - 那么你必须：
     - 要么有本地会话存储可恢复历史
     - 要么拒绝转换
2. **不要假设** Chat 或 Messages 能理解 `previous_response_id`
3. 第一版最稳妥策略：
   - 如果要支持 `responses -> chat/messages`
   - 就要求代理层持久化每次 Responses 的归一化上下文

---

## 3.3 `messages -> RequestIR`

### 字段映射

| Messages 字段 | RequestIR |
|---|---|
| `model` | `model` |
| `stream` | `stream` |
| `max_tokens` | `maxOutputTokens` |
| `system` | `system[]` |
| `messages[]` | `messages[]` |
| `tools[]` | `tools[]` |
| `tool_choice` | `toolChoice` |
| `tool_choice.disable_parallel_tool_use` | `allowParallelToolCalls` 的反向表达 |

### `system` 归一化

Claude 的 `system` 不在 `messages[]` 内。  
归一化时直接拆进 `system[]`。

### `tools` 归一化

Claude 工具定义：

```json
{
  "name": "get_weather",
  "description": "Get weather",
  "input_schema": {"type":"object"},
  "strict": true
}
```

归一化后：

```json
{
  "name": "get_weather",
  "description": "Get weather",
  "inputSchema": {"type":"object"},
  "strict": true
}
```

### `tool_choice` 归一化

- `{type:"none"} -> {type:"none"}`
- `{type:"auto"} -> {type:"auto"}`
- `{type:"any"} -> {type:"required"}`
- `{type:"tool",name:"x"} -> {type:"tool",name:"x"}`

并行语义：

- `disable_parallel_tool_use = true` -> `allowParallelToolCalls = false`
- `disable_parallel_tool_use = false` 或未提供 -> `allowParallelToolCalls = true`

### `messages[]` 归一化

Claude 的消息内容可以是字符串，也可以是 block 数组。

建议统一处理为 block 数组，然后映射：

- `text -> IR text block`
- `tool_use -> IR tool_call block`
- `tool_result -> IR tool_result block`

注意：

- `assistant` 消息里可包含 `tool_use`
- `user` 消息里可包含 `tool_result`
- `tool_use.input` 是对象，归一化时要 `JSON.stringify(input)` 存入 `argumentsText`
- `tool_result.content` 若是数组，第一版建议序列化成字符串；如果后续需要多模态结果，再扩展 IR

---

## 4. 从 `RequestIR` 编码回三个端点

## 4.1 `RequestIR -> chat/completions`

### 编码规则

1. `system[]` 还原为若干条 `role:"system"` 消息
2. `IRMessage(role="user")`
   - `text block` -> `role:"user", content:string`
   - `tool_result block` -> `role:"tool", tool_call_id, content`
3. `IRMessage(role="assistant")`
   - 纯文本 -> `role:"assistant", content:string`
   - 工具调用 -> `role:"assistant", tool_calls:[...]`
   - 文本 + 工具调用并存 -> `content + tool_calls` 同时输出
4. `tools[]`
   - `inputSchema -> function.parameters`
5. `toolChoice`
   - `required -> "required"`
   - `tool(name) -> {"type":"function","function":{"name":"..."}}`
6. `allowParallelToolCalls -> parallel_tool_calls`

### 特别注意

- Chat 的工具参数字段必须是 JSON 字符串：`function.arguments`
- 如果 IR 里工具调用参数无法序列化为合法字符串，应直接报错，不要输出半结构化对象

---

## 4.2 `RequestIR -> responses`

### 编码规则

1. `system[]` 优先编码到 `instructions`
2. `messages[]` 编码到 `input`
3. `IRMessage(role="user")`
   - `text block -> input_text`
   - `tool_result block -> function_call_output`
4. `IRMessage(role="assistant")`
   - 文本历史项 -> `type:"message", role:"assistant", content:"..."` 或 `content:[{type:"input_text",text:"..."}]`
   - 若同一条 assistant 历史里包含工具调用，第一版建议拒绝直接编码为 `input` 历史，优先改走显式会话重放或本地状态恢复
5. `tools[]`
   - `inputSchema -> parameters`
6. `toolChoice`
   - `required -> "required"`
   - `tool(name) -> {"type":"function","name":"..."}`
7. `allowParallelToolCalls -> parallel_tool_calls`

### 关于 assistant 历史消息

Responses 支持把 assistant 历史作为输入再发回，但其内容类型与最终 `output` 的 shape 并不完全一样。  
编码时建议只保留公共子集：

- assistant 文本
- assistant 工具调用结果对应的后续工具回填

如果你没有明确验证某类 assistant 历史项在目标模型上可用，宁可拒绝，不要猜。

### 关于 `previous_response_id`

如果目标就是 `/v1/responses`，你可以把 `responseRef.previousResponseId` 原样带回。  
但如果当前 `RequestIR` 是从 chat/messages 归一化而来，通常并没有这个字段；不要伪造。

---

## 4.3 `RequestIR -> messages`

### 编码规则

1. `system[]` 合并编码到顶层 `system`
2. `messages[]` 编码为 Claude 原生 block 结构
3. `IRMessage(role="assistant")`
   - `text block -> {type:"text",text}`
   - `tool_call block -> {type:"tool_use",id,name,input}`
   - 其中 `argumentsText` 必须先 `JSON.parse` 成对象再放入 `input`
4. `IRMessage(role="user")`
   - `text block -> {type:"text",text}`
   - `tool_result block -> {type:"tool_result",tool_use_id,content,is_error?}`
5. `tools[]`
   - `inputSchema -> input_schema`
6. `toolChoice`
   - `required -> {type:"any"}`
   - `tool(name) -> {type:"tool",name}`
7. `allowParallelToolCalls=false -> disable_parallel_tool_use=true`

### Claude 分支的硬规则

这是编码时最关键的几条：

1. `assistant.tool_use` 后面必须立刻跟对应的 `user.tool_result`
2. 这两条消息之间不能插入别的消息
3. 若同一条 `user` 消息同时带文本和 `tool_result`：
   - `tool_result` block 必须排前面
   - 文本 block 必须排后面
4. `tool_use.input` 必须是对象，不是 JSON 字符串
5. 若 `argumentsText` 不是合法 JSON，不能发给 Claude

如果违反这些规则，Claude 很容易返回 `400`。

---

## 5. 非流式响应的统一结构

建议把三种端点的非流式响应都先归一化成：

```ts
interface ResponseIR {
  source: "chat" | "responses" | "messages";
  id: string;
  model: string;
  output: IRMessage[];
  finishReason: "stop" | "tool_calls" | "length" | "content_filter" | "unknown";
  usage?: {
    inputTokens?: number;
    outputTokens?: number;
    totalTokens?: number;
  };
  responseRef?: {
    responseId?: string;
  };
}
```

### 为什么 `output` 仍然用 `IRMessage[]`

因为：

- Chat 非流式本质上是一个 assistant message
- Messages 非流式本质上也是一个 assistant message
- Responses 非流式虽然是 `output[]` 数组，但其中常见项也能归并成 assistant 文本 / assistant 工具调用

这样编码回目标端点时可以复用同一套 block 逻辑。

---

## 6. 三端点非流式响应归一化

## 6.1 `chat/completions -> ResponseIR`

### 取值规则

- `id -> id`
- `model -> model`
- `choices[0].message -> output[0]`
- `choices[0].finish_reason -> finishReason`
- `usage.prompt_tokens -> inputTokens`
- `usage.completion_tokens -> outputTokens`
- `usage.total_tokens -> totalTokens`

### `message` 归一化

- `content -> text block`
- `tool_calls[] -> tool_call blocks`
- `finish_reason=tool_calls -> finishReason="tool_calls"`

---

## 6.2 `responses -> ResponseIR`

Responses 的难点是：`output[]` 可能混合：

- `message`
- `function_call`
- reasoning 项
- 内置工具项

### 第一版建议的归一化策略

只接收两类公共项：

1. `message`
2. `function_call`

### 归一化规则

- `message.role=assistant` 的 `output_text -> text block`
- `function_call` -> `tool_call block`
  - `call_id -> block.id`
  - `name -> block.name`
  - `arguments -> block.argumentsText`
- `response.id -> responseRef.responseId`
- `usage.input_tokens -> inputTokens`
- `usage.output_tokens -> outputTokens`
- `usage.total_tokens -> totalTokens`
- 若 `output[]` 中出现 `function_call`，则 `finishReason = "tool_calls"`
- 若 `status = completed` 且没有 `function_call`，则 `finishReason = "stop"`
- 若 `status = incomplete` 且能确认是 token 截断，可映射为 `"length"`；否则用 `"unknown"`

### 顺序规则

Responses 的 `output[]` 顺序可能有意义。  
归一化时不要假定：

- 第一个一定是文本
- 第一个一定是函数调用

应按 `output[]` 原始顺序扫描，并归并到 `IRMessage(role="assistant")` 的 `blocks[]` 中。

### 不能静默吞掉的项

如果收到以下项：

- reasoning
- 内置工具调用
- provider 特有输出项

第一版建议：

- 明确报“当前转换器不支持该输出项”
- 不要假装成功

---

## 6.3 `messages -> ResponseIR`

### 归一化规则

Claude 非流式响应通常是一个 assistant message：

- `content[].type=text -> text block`
- `content[].type=tool_use -> tool_call block`
  - `id -> block.id`
  - `name -> block.name`
  - `input -> JSON.stringify(input)`
- `stop_reason=tool_use -> finishReason="tool_calls"`
- `stop_reason=end_turn -> finishReason="stop"`
- `usage.input_tokens -> inputTokens`
- `usage.output_tokens -> outputTokens`

---

## 7. 从 `ResponseIR` 编码回三个端点

## 7.1 `ResponseIR -> chat/completions`

编码为标准 Chat Completions 响应：

```json
{
  "id": "chatcmpl_xxx",
  "object": "chat.completion",
  "model": "...",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "...",
        "tool_calls": []
      },
      "finish_reason": "stop"
    }
  ]
}
```

### 编码规则

- `text block -> message.content`
- `tool_call block -> message.tool_calls[]`
- `finishReason="tool_calls" -> finish_reason="tool_calls"`
- usage 按 OpenAI 字段名输出

---

## 7.2 `ResponseIR -> responses`

编码为标准 Responses 响应：

- assistant 文本 -> `output[].type = "message"`
- assistant 工具调用 -> `output[].type = "function_call"`
- `responseRef.responseId` 若存在，可用于 `id`

### 注意

Responses 的 `message` 项与 `function_call` 项通常是并列的 `output[]` 元素，不一定塞进同一个对象里。  
因此从 `ResponseIR` 回编码到 Responses 时，建议：

- `text block` 生成一个 `message` output item
- 每个 `tool_call block` 生成一个 `function_call` output item
- 顺序按 `blocks[]` 顺序输出

---

## 7.3 `ResponseIR -> messages`

Claude 非流式响应编码规则：

- `text block -> content[].type=text`
- `tool_call block -> content[].type=tool_use`
  - `argumentsText` 先 `JSON.parse`
- `finishReason="tool_calls" -> stop_reason="tool_use"`
- `finishReason="stop" -> stop_reason="end_turn"`

---

## 8. 统一流式事件模型（强烈建议）

三端点的流式格式差别很大，最好不要直接互转事件，而是先归一化成统一事件流。

建议内部定义：

```ts
type StreamEventIR =
  | { type: "message_start"; role: "assistant" }
  | { type: "text_delta"; text: string }
  | { type: "tool_call_start"; index: number; id: string; name: string }
  | { type: "tool_call_arguments_delta"; index: number; id: string; delta: string }
  | { type: "tool_call_done"; index: number; id: string; argumentsText?: string }
  | { type: "message_done"; finishReason: "stop" | "tool_calls" | "length" | "unknown" }
  | { type: "usage"; inputTokens?: number; outputTokens?: number; totalTokens?: number }
  | { type: "error"; message: string };
```

只要能把三端点都映射到这套事件，流式互转就会简单很多。

---

## 9. 三端点流式解码规则

## 9.1 `chat/completions stream -> StreamEventIR`

Chat 流的核心信息在：

- `choices[0].delta.role`
- `choices[0].delta.content`
- `choices[0].delta.tool_calls`
- `choices[0].finish_reason`

### 解码规则

- 第一帧 assistant role -> `message_start`
- `delta.content` -> `text_delta`
- `delta.tool_calls[].id/name` 首次出现 -> `tool_call_start`
- `delta.tool_calls[].function.arguments` 增量 -> `tool_call_arguments_delta`
- `finish_reason=tool_calls` -> `message_done(tool_calls)`
- `finish_reason=stop` -> `message_done(stop)`

### 实现注意

- Chat 的 `tool_calls` 增量可能分多帧出现
- `id/name` 往往只在第一帧完整出现
- 后续帧可能只补 `arguments`
- 必须按 `tool_calls[].index` 缓存状态

---

## 9.2 `responses stream -> StreamEventIR`

Responses 流不是 `chat.completion.chunk`，而是事件流。

应重点处理这些事件：

- `response.output_item.added`
- `response.content_part.added`
- `response.reasoning_text.done`
- `response.output_text.delta`
- `response.function_call_arguments.delta`
- `response.function_call_arguments.done`
- `response.output_item.done`
- `response.completed`

### 解码规则

1. `response.output_item.added`
   - 如果 item 是 assistant message，可视为 `message_start`
   - 如果 item 是 `function_call`，可视为 `tool_call_start`
2. `response.content_part.added`
   - 用于声明新的 `content` 槽位开始写入
   - 不能默认 `content_index=0` 一定是 `output_text`
   - `part.type` 可能是 `output_text`，也可能是 `reasoning_text`
3. `response.output_text.delta`
   - -> `text_delta`
4. `response.reasoning_text.done`
   - 表示 reasoning part 的完整文本结束
   - 若要做 `responses -> chat/messages` 历史恢复，不能丢掉这部分
5. `response.function_call_arguments.delta`
   - -> `tool_call_arguments_delta`
6. `response.function_call_arguments.done`
   - -> `tool_call_done`
7. `response.completed`
   - -> `message_done`

### 实现注意

- `output_index` 很关键，必须保留
- `call_id` 是工具调用稳定 ID，优先用于跨轮追踪
- `item_id` 可以作为内部辅助状态键，但不要把它当成 tool call ID 对外暴露
- 流式聚合时最好按 `item_id + content_index` 缓存文本 / reasoning 增量
- 如果代理把 Chat 的 `reasoning_content` 转成 Responses 流，通常应先发 `response.content_part.added`，再发 reasoning/text 增量或 done 事件

---

## 9.3 `messages stream -> StreamEventIR`

Claude 流常见事件：

- `message_start`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`
- `message_delta`
- `message_stop`

### 解码规则

1. `message_start` -> `message_start`
2. `content_block_start` 若 block 是 text
   - 后续 `text_delta` -> `text_delta`
3. `content_block_start` 若 block 是 `tool_use`
   - 立刻发 `tool_call_start`
   - `id = tool_use.id`
   - `name = tool_use.name`
4. `content_block_delta.type = input_json_delta`
   - -> `tool_call_arguments_delta`
5. `content_block_stop` 对应工具块结束
   - 可发 `tool_call_done`
6. `message_delta.stop_reason = tool_use`
   - -> `message_done(tool_calls)`
7. `message_delta.stop_reason = end_turn`
   - -> `message_done(stop)`

### 实现注意

- Claude 的工具参数是 `partial_json` 片段，不是对象
- 必须按 `content block index` 缓冲
- 最终完整 `input` 只有在拼接完成后才适合解析
- 中间增量阶段不要强行 `JSON.parse`

---

## 10. 从统一流式事件编码回三端点

## 10.1 `StreamEventIR -> chat/completions stream`

输出格式必须是 OpenAI SSE：

```text
data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[...]}

data: [DONE]
```

### 编码规则

- `message_start` -> 首帧 `delta.role = "assistant"`
- `text_delta` -> `delta.content`
- `tool_call_start` -> `delta.tool_calls = [{index,id,type:"function",function:{name,arguments:""}}]`
- `tool_call_arguments_delta` -> `delta.tool_calls = [{index,function:{arguments:delta}}]`
- `message_done(tool_calls)` -> `finish_reason = "tool_calls"`
- `message_done(stop)` -> `finish_reason = "stop"`
- 结束时补 `data: [DONE]`

### 关键点

- 对同一个 tool call，`id/name/type` 通常只应在首个增量帧发一次
- 后续只继续发 `arguments` 增量

---

## 10.2 `StreamEventIR -> responses stream`

输出必须是 Responses 事件流，而不是 Chat SSE。

### 编码建议

- `message_start` -> `response.output_item.added`（assistant message）
- `text_delta` -> `response.output_text.delta`
- `tool_call_start` -> `response.output_item.added`（function_call item）
- `tool_call_arguments_delta` -> `response.function_call_arguments.delta`
- `tool_call_done` -> `response.function_call_arguments.done`
- `message_done` -> `response.completed`

### 关键点

- Responses 的文本和函数调用通常是不同 output item
- 因此编码器要自己维护：
  - 当前 message item 的 `output_index`
  - 当前 function call item 的 `output_index`
  - `call_id`
  - `item_id`

---

## 10.3 `StreamEventIR -> messages stream`

输出必须是 Claude SSE。

### 编码建议

- `message_start` -> `event: message_start`
- `text_delta` -> `event: content_block_delta` + `text_delta`
- `tool_call_start` -> `event: content_block_start`，block 类型为 `tool_use`
- `tool_call_arguments_delta` -> `event: content_block_delta`，类型为 `input_json_delta`
- `tool_call_done` -> `event: content_block_stop`
- `message_done` -> `event: message_delta` + `event: message_stop`

### 关键点

- Claude 工具块在流里是 block 级别语义，不是 message-level `tool_calls`
- 必须维护 `content block index`
- `tool_use.input` 在流中是逐段 JSON，不应提前转对象

---

## 11. 贯穿三端点的硬约束

以下约束必须贯穿请求转换、响应转换和流式转换三层。

### 11.1 工具调用 ID 必须稳定

三端之间的对应关系建议如下：

- Chat `tool_calls[].id`
- Responses `function_call.call_id`
- Messages `tool_use.id`

统一都映射到 IR 的：

- `tool_call.id`
- `tool_result.toolCallId`

不要在跨端转换时无故重写 ID。

### 11.2 参数类型必须分清

- Chat / Responses 的工具参数：**JSON 字符串**
- Messages 的工具参数：**对象**

也就是：

- `chat/responses -> messages`：`JSON.parse`
- `messages -> chat/responses`：`JSON.stringify`

### 11.3 工具结果的“回填方向”要统一

从语义上看，以下三者是同一种东西：

- Chat `role:"tool"`
- Responses `function_call_output`
- Messages `role:"user" + content[].type="tool_result"`

统一后都应落到 IR 的 `user.tool_result block`。

### 11.4 Claude 的顺序要求最严格

如果目标端是 `/v1/messages`，必须保证：

- `assistant tool_use` 后立即跟 `user tool_result`
- 中间不能插别的消息
- 同一条 user 消息里，`tool_result` 必须排在文本前面

### 11.5 不支持的能力必须显式报错

第一版不要静默吞掉这些内容：

- Responses reasoning / built-in tools
- Claude thinking / server tools / `pause_turn`
- 不可识别的内容块类型
- 不可解析的工具参数 JSON
- 仅靠 `previous_response_id` 但又无本地历史时的跨端转换

---

## 12. 推荐的代码模块划分

建议至少拆成下面几层：

- `ir/types.ts`
  - 定义 `RequestIR` / `ResponseIR` / `StreamEventIR`
- `normalizers/chat.ts`
  - `chat -> IR`
- `normalizers/responses.ts`
  - `responses -> IR`
- `normalizers/messages.ts`
  - `messages -> IR`
- `encoders/chat.ts`
  - `IR -> chat`
- `encoders/responses.ts`
  - `IR -> responses`
- `encoders/messages.ts`
  - `IR -> messages`
- `stream/chat.ts`
  - Chat 流编解码
- `stream/responses.ts`
  - Responses 流编解码
- `stream/messages.ts`
  - Messages 流编解码
- `tooling/state.ts`
  - 工具调用索引、流式缓冲、会话恢复

如果不做这层拆分，后面加第二个目标端点时代码会迅速失控。

---

## 13. 建议最先实现的转换顺序

如果要控制实现风险，建议按下面顺序编码：

1. `chat <-> messages` 非流式文本
2. `chat <-> messages` 非流式工具调用
3. `chat <-> messages` 流式文本
4. `chat <-> messages` 流式工具调用
5. `responses <-> IR` 非流式文本
6. `responses <-> IR` 非流式函数调用
7. `responses <-> IR` 流式文本
8. `responses <-> IR` 流式函数参数增量
9. 最后补 `responses <-> chat/messages` 的完整跨端链路

不要一开始就三端同时开工。

---

## 14. 最小测试矩阵（必须有）

至少覆盖以下场景：

1. 纯文本，非流式
2. 纯文本，流式
3. 单工具调用，非流式
4. 单工具调用，流式参数增量
5. 多工具调用，非流式
6. 多工具调用，流式
7. assistant 文本 + tool call 混合输出
8. tool_result 回填后的第二轮继续回答
9. Claude 顺序错误时能正确报错
10. Responses 只有 `previous_response_id` 且无历史时能拒绝跨端转换
11. 非法 JSON arguments 时能明确报错
12. 未支持 output/content 类型时能明确报错

如果这 12 条没过，就不要宣称“三端互转已兼容”。

---

## 15. 这份文档的最终结论

从编码角度看，这三个端点最大的协议差异只有四类：

1. **上下文表达方式不同**
   - Chat：`messages`
   - Responses：`instructions + input + previous_response_id`
   - Messages：`system + messages`

2. **工具调用表达方式不同**
   - Chat：`tool_calls`
   - Responses：`function_call`
   - Messages：`tool_use`

3. **工具结果回填方式不同**
   - Chat：`role=tool`
   - Responses：`function_call_output`
   - Messages：`user.tool_result`

4. **流式事件模型不同**
   - Chat：`chat.completion.chunk`
   - Responses：`response.*` 事件
   - Messages：`message_start/content_block_delta/...`

因此正确做法不是“写一堆 if/else 直接改字段”，而是：

- 先归一化到 IR
- 再从 IR 编码到目标端
- 对流式也做统一事件层
- 对不在公共子集里的能力显式拒绝

这是最适合指导编码、也最不容易失控的实现路线。

---

## 16. 参考

本地文档（同目录下）：

- [chat-completions-spec.md](chat-completions-spec.md)
- [messages-spec.md](messages-spec.md)
- [responses-spec.md](responses-spec.md)

官方文档：

- OpenAI Chat Completions / Function Calling: <https://platform.openai.com/docs/guides/function-calling?api-mode=chat>
- OpenAI Responses: <https://platform.openai.com/docs/api-reference/responses>
- OpenAI Responses Streaming: <https://platform.openai.com/docs/guides/streaming-responses>
- Anthropic Messages API: <https://docs.anthropic.com/en/api/messages>
- Anthropic Tool Use: <https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implement-tool-use>
- Anthropic Messages Streaming: <https://docs.anthropic.com/en/api/messages-streaming>
