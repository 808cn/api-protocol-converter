# `/v1/responses` 协议规范

本文基于 OpenAI 官方文档整理，聚焦：

- `POST /v1/responses`

说明：

- 以下内容以我在 **2026-04-02** 查阅的 OpenAI 官方文档为准。
- OpenAI 官方推荐：**新项目优先使用 `/v1/responses`**。
- 下文只覆盖最常用、最稳定、最容易出兼容问题的协议字段；模型专属字段与内置工具细节请以具体模型文档为准。

## 1. 通用 HTTP 约定

### 基础地址

```text
https://api.openai.com
```

### 请求方法

```http
POST /v1/responses
```

### 认证头

```http
Authorization: Bearer OPENAI_API_KEY
Content-Type: application/json
```

可选头：

- `OpenAI-Organization: org_xxx`
- `OpenAI-Project: proj_xxx`
- `X-Client-Request-Id: <你自己的请求 ID>`

### 错误响应形态

常见错误对象形态：

```json
{
  "error": {
    "message": "错误说明",
    "type": "invalid_request_error",
    "param": "input",
    "code": "invalid_value"
  }
}
```

---

## 2. 请求体规范

## 2.1 核心字段

常用字段：

- `model: string`
- `input: string | array`
  - 可以是简单字符串，也可以是输入项数组
- `instructions: string | array`
  - 系统或开发者级指令
- `stream: boolean`
  - 官方默认 **`false`**
- `tools: array`
  - 可包含内置工具、MCP 工具、自定义函数工具等
- `tool_choice`
  - 工具选择策略
- `parallel_tool_calls: boolean`
  - 是否允许并行工具调用
- `previous_response_id: string`
  - 引用上一轮 response，形成多轮上下文
- `max_output_tokens: number`
- `text`
  - 文本输出配置；支持普通文本、`json_object`、`json_schema`
- `temperature: number`
- `top_p: number`
- `store: boolean`
- `truncation`
  - 例如 `disabled` 或 `auto`

最小请求体：

```json
{
  "model": "gpt-5",
  "input": "Hello!"
}
```

## 2.2 `instructions`

官方说明：

- `instructions` 会以 system 或 developer 的优先级插入模型上下文
- 与 `previous_response_id` 一起使用时，上一轮的 `instructions` 不会自动继承到下一轮

示例：

```json
{
  "model": "gpt-5",
  "instructions": "你是一个严谨的技术助手。",
  "input": "请解释哈希表。"
}
```

---

## 3. `input` 输入格式

## 3.1 直接传字符串

```json
{
  "model": "gpt-5",
  "input": "Hello!"
}
```

语义上等价于一条 `user` 文本输入。

## 3.2 传输入项数组

`input` 可包含多个 `ResponseInputItem`。最常见的是 `message` 项。

示例：

```json
{
  "role": "user",
  "content": [
    {"type": "input_text", "text": "请描述这张图片"},
    {"type": "input_image", "image_url": "https://example.com/demo.jpg"}
  ]
}
```

常见内容类型：

- `input_text`
- `input_image`
- `input_file`

常见 `role`：

- `user`
- `assistant`
- `system`
- `developer`

## 3.3 `assistant` 历史项与 `phase`

Responses API 支持把历史 assistant 输出作为输入再发回模型。某些新模型还支持 `phase`：

- `commentary`
- `final_answer`

这主要用于更复杂的代理、多阶段推理或代码类工作流。

---

## 4. 工具与函数调用

## 4.1 自定义函数工具定义

最常见的函数工具定义如下：

```json
{
  "type": "function",
  "name": "get_weather",
  "description": "Get current weather",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {"type": "string"}
    },
    "required": ["location"]
  },
  "strict": true
}
```

说明：

- `parameters` 是 JSON Schema
- `strict: true` 时要求更严格地遵守 schema

## 4.2 模型输出的函数调用项

在 `output` 数组中，模型可能返回：

```json
{
  "type": "function_call",
  "id": "fc_xxx",
  "call_id": "call_xxx",
  "name": "get_weather",
  "arguments": "{\"location\":\"Boston, MA\"}",
  "status": "completed"
}
```

说明：

- `arguments` 是 JSON 字符串，不是对象
- 实际执行工具前，调用方应自行做 JSON 解析与 schema 校验

## 4.3 工具结果回填

执行完工具后，把结果作为新的 `input` 项传回：

```json
{
  "type": "function_call_output",
  "call_id": "call_xxx",
  "output": "{\"temperature\":23,\"unit\":\"celsius\"}"
}
```

`output` 可以是：

- 字符串
- 内容块数组

## 4.4 `tool_choice`

Responses API 常见工具控制模式：

- `"none"`
- `"auto"`
- `"required"`
- `{"type":"allowed_tools","mode":"auto|required","tools":[...]}`
- `{"type":"function","name":"get_weather"}`

## 4.5 `parallel_tool_calls`

- 类型：`boolean`
- 含义：是否允许模型并行调用多个工具

---

## 5. 文本与结构化输出

Responses API 把结构化输出放在 `text.format` 下。

常见模式：

- `{ "type": "text" }`
- `{ "type": "json_object" }`
- `{ "type": "json_schema", "name": "...", "schema": {...}, "strict": true }`

示例：

```json
{
  "text": {
    "format": {
      "type": "json_schema",
      "name": "answer_card",
      "schema": {
        "type": "object",
        "properties": {
          "answer": {"type": "string"},
          "confidence": {"type": "number"}
        },
        "required": ["answer", "confidence"],
        "additionalProperties": false
      },
      "strict": true
    },
    "verbosity": "medium"
  }
}
```

---

## 6. 响应体规范

## 6.1 非流式响应

Responses API 的返回对象类型为 `response`：

```json
{
  "id": "resp_xxx",
  "object": "response",
  "created_at": 1741290958,
  "status": "completed",
  "model": "gpt-4.1-2025-04-14",
  "output": [
    {
      "id": "msg_xxx",
      "type": "message",
      "status": "completed",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "Hi there! How can I assist you today?",
          "annotations": []
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 37,
    "output_tokens": 11,
    "total_tokens": 48
  }
}
```

常见 `status`：

- `completed`
- `failed`
- `in_progress`
- `cancelled`
- `queued`
- `incomplete`

## 6.2 `output` 的常见项

`output` 数组的长度和顺序与模型实际行为相关，不能假定第一个元素永远是文本消息。

常见项包括：

- `message`
- `function_call`
- reasoning 项
- 内置工具调用项

---

## 7. 流式响应规范

当 `stream=true` 时，`/v1/responses` 返回的是 **Responses 专属 SSE 事件**，不是 `chat.completion.chunk`。

典型事件序列：

```text
event: response.created
data: {...}

event: response.in_progress
data: {...}

event: response.output_item.added
data: {...}

event: response.content_part.added
data: {...}

event: response.output_text.delta
data: {...}

event: response.output_text.done
data: {...}

event: response.output_item.done
data: {...}

event: response.completed
data: {...}
```

关键特征：

- 事件 `data` 内部自带 `type`
- 文本增量在 `response.output_text.delta`
- 完整结束由 `response.completed` 表示

### 7.1 关键事件补充定义

以下 3 个事件在实现跨协议转换、流式聚合恢复、以及带 reasoning 的模型兼容时尤其关键。

#### `response.content_part.added`

官方定义：当一个新的 content part 被加入某个 output item 时发出。

关键字段：

- `item_id: string`
  - 该 content part 所属 output item 的 ID
- `output_index: integer`
  - 该 output item 在 `response.output[]` 中的索引
- `content_index: integer`
  - 该 content part 在 `item.content[]` 中的索引
- `part: object`
  - 新增的内容块本体
- `type: "response.content_part.added"`

官方示例：

```json
{
  "type": "response.content_part.added",
  "item_id": "msg_123",
  "output_index": 0,
  "content_index": 0,
  "part": {
    "type": "output_text",
    "text": "",
    "annotations": []
  },
  "sequence_number": 1
}
```

实现提示：

- 不能只看 `output_index`，流式聚合时最好同时保留 `item_id`
- `part.type` 不一定只有 `output_text`，也可能是 reasoning 相关 part
- 若代理要把 chat 流式结果转成 Responses 流，通常要先发 `response.output_item.added`，再发 `response.content_part.added`

#### `response.output_text.delta`

官方定义：当某个 `output_text` content part 的文本值增量更新时发出。

关键字段：

- `item_id: string`
- `output_index: integer`
- `content_index: integer`
- `delta: string`
  - 本次文本增量
- `type: "response.output_text.delta"`

官方示例：

```json
{
  "event_id": "event_4142",
  "type": "response.output_text.delta",
  "response_id": "resp_001",
  "item_id": "msg_007",
  "output_index": 0,
  "content_index": 0,
  "delta": "Sure, I can h"
}
```

实现提示：

- 恢复完整文本时应按 `item_id + content_index` 归并 delta
- 如果只靠最终 `response.output_item.done` 兜底，遇到中断流时更容易丢内容
- 从别的协议转入 Responses 流时，建议补齐 `item_id`

#### `response.reasoning_text.done`

官方定义：当 reasoning 文本内容完整结束时发出。

关键字段：

- `item_id: string`
- `output_index: integer`
- `content_index: integer`
  - reasoning content part 的索引
- `text: string`
  - 完整 reasoning 文本
- `type: "response.reasoning_text.done"`

官方示例：

```json
{
  "type": "response.reasoning_text.done",
  "item_id": "rs_123",
  "output_index": 0,
  "content_index": 0,
  "text": "The user is asking...",
  "sequence_number": 4
}
```

实现提示：

- 如果上游模型要求多轮时必须回传 reasoning/thinking，上游 reasoning 不能在代理里丢失
- 非流式 `output[]` 里已经可能有 reasoning 项；流式路径也要同步保留
- 代理在做 `responses -> chat/messages` 历史展开时，不能只保留 `output_text` 和 `function_call`

---

## 8. `curl` 收发示例

以下示例均使用：

```bash
export OPENAI_API_KEY="你的密钥"
```

## 8.1 最小文本请求

### 请求

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "instructions": "你是一个严谨的技术助手。",
    "input": "请用一句话解释哈希表。",
    "stream": false
  }'
```

### 响应示例

```json
{
  "id": "resp_123",
  "object": "response",
  "created_at": 1741290958,
  "status": "completed",
  "model": "gpt-5",
  "output": [
    {
      "id": "msg_123",
      "type": "message",
      "status": "completed",
      "role": "assistant",
      "content": [
        {
          "type": "output_text",
          "text": "哈希表是一种通过哈希函数把键映射到存储位置，从而实现高效查找、插入和删除的数据结构。",
          "annotations": []
        }
      ]
    }
  ],
  "usage": {
    "input_tokens": 18,
    "output_tokens": 29,
    "total_tokens": 47
  }
}
```

## 8.2 函数工具调用

### 第一步，请求模型产生函数调用

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": "查询波士顿天气",
    "tools": [
      {
        "type": "function",
        "name": "get_weather",
        "description": "获取天气",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string"}
          },
          "required": ["location"]
        },
        "strict": true
      }
    ],
    "tool_choice": "auto",
    "parallel_tool_calls": true
  }'
```

### 可能的响应

```json
{
  "id": "resp_456",
  "object": "response",
  "status": "completed",
  "output": [
    {
      "type": "function_call",
      "id": "fc_001",
      "call_id": "call_001",
      "name": "get_weather",
      "arguments": "{\"location\":\"Boston, MA\"}",
      "status": "completed"
    }
  ]
}
```

## 8.3 回填函数执行结果

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": [
      {
        "type": "function_call_output",
        "call_id": "call_001",
        "output": "{\"temperature\":23,\"unit\":\"celsius\"}"
      }
    ],
    "previous_response_id": "resp_456"
  }'
```

## 8.4 结构化 JSON 输出

### 请求

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "input": "请用 JSON 描述 Python。",
    "text": {
      "format": {
        "type": "json_schema",
        "name": "language_card",
        "schema": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "typed": {"type": "boolean"}
          },
          "required": ["name", "typed"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

### 响应示例

```json
{
  "id": "resp_789",
  "object": "response",
  "status": "completed",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "status": "completed",
      "content": [
        {
          "type": "output_text",
          "text": "{\"name\":\"Python\",\"typed\":false}",
          "annotations": []
        }
      ]
    }
  ]
}
```

## 8.5 流式请求

### 请求

```bash
curl -N https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5",
    "instructions": "You are a helpful assistant.",
    "input": "Hello!",
    "stream": true
  }'
```

### SSE 响应示例

```text
event: response.created
data: {"type":"response.created","response":{"id":"resp_xxx","object":"response","status":"in_progress"}}

event: response.in_progress
data: {"type":"response.in_progress","response":{"id":"resp_xxx","object":"response","status":"in_progress"}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"id":"msg_xxx","type":"message","status":"in_progress","role":"assistant","content":[]}}

event: response.content_part.added
data: {"type":"response.content_part.added","item_id":"msg_xxx","output_index":0,"content_index":0,"part":{"type":"output_text","text":"","annotations":[]}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","item_id":"msg_xxx","output_index":0,"content_index":0,"delta":"Hi"}

event: response.output_text.done
data: {"type":"response.output_text.done","item_id":"msg_xxx","output_index":0,"content_index":0,"text":"Hi there! How can I assist you today?"}

event: response.output_item.done
data: {"type":"response.output_item.done","output_index":0,"item":{"id":"msg_xxx","type":"message","status":"completed","role":"assistant","content":[{"type":"output_text","text":"Hi there! How can I assist you today?","annotations":[]}]}}

event: response.completed
data: {"type":"response.completed","response":{"id":"resp_xxx","object":"response","status":"completed"}}
```

### 带 reasoning 的补充 SSE 示例

当模型流式输出 reasoning 内容时，还可能出现类似事件：

```text
event: response.content_part.added
data: {"type":"response.content_part.added","item_id":"msg_xxx","output_index":0,"content_index":0,"part":{"type":"reasoning_text","text":""}}

event: response.reasoning_text.done
data: {"type":"response.reasoning_text.done","item_id":"msg_xxx","output_index":0,"content_index":0,"text":"The user is asking..."}
```

说明：

- 这通常表示 `item.content[0]` 是 reasoning part
- 若同一条 assistant message 后续还有可见文本，则后面的 `output_text` part 往往会出现在更大的 `content_index`
- 代理做跨协议转换时，不能默认 `content_index=0` 一定是 `output_text`

---

## 9. 官方来源

- API Overview  
  https://developers.openai.com/api/reference/overview
- Create a model response  
  https://developers.openai.com/api/reference/resources/responses/methods/create
- Streaming API responses  
  https://developers.openai.com/api/docs/guides/streaming-responses
- Streaming events: `response.content_part.added`  
  https://platform.openai.com/docs/api-reference/responses-streaming/response/output_text
- Streaming events: `response.reasoning_text.done`  
  https://platform.openai.com/docs/api-reference/responses-streaming/response/output_text
- Streaming events: `response.output_text.delta`  
  https://platform.openai.com/docs/api-reference/responses-streaming/response/output_text
- Function calling  
  https://platform.openai.com/docs/guides/function-calling
- Migrate to the Responses API  
  https://developers.openai.com/api/docs/guides/migrate-to-responses
