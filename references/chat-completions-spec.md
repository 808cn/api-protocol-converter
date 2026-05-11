# `/v1/chat/completions` 协议规范

本文基于 OpenAI 官方文档整理，聚焦：

- `POST /v1/chat/completions`

说明：

- 以下内容以我在 **2026-04-02** 查阅的 OpenAI 官方文档为准。
- OpenAI 官方仍支持该端点，但明确建议：**新项目优先使用 `/v1/responses`**。
- 下文只覆盖最常用、最稳定、最容易出兼容问题的协议字段；模型专属字段请以具体模型文档为准。

## 1. 通用 HTTP 约定

### 基础地址

```text
https://api.openai.com
```

### 请求方法

```http
POST /v1/chat/completions
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
    "param": "messages",
    "code": "invalid_value"
  }
}
```

---

## 2. 请求体规范

## 2.1 必填字段

- `model: string`
- `messages: array`

最小请求体：

```json
{
  "model": "gpt-4o",
  "messages": [
    {"role": "user", "content": "你好"}
  ]
}
```

## 2.2 常用可选字段

- `stream: boolean`
  - `true` 时返回 SSE 流
- `tools: array`
  - 模型可调用的工具列表
- `tool_choice`
  - 控制工具选择策略
- `parallel_tool_calls: boolean`
  - 是否允许并行工具调用，官方默认 `true`
- `response_format`
  - 控制输出为普通文本、`json_object` 或 `json_schema`
- `max_completion_tokens: number`
  - completion 最大 token 上限；包含可见输出 token 与 reasoning token
- `temperature: number`
- `top_p: number`
- `store: boolean`
- `metadata: object`
- `logprobs: boolean`

## 2.3 已废弃但仍可能出现的字段

- `functions`
  - 已废弃，建议改用 `tools`
- `function_call`
  - 已废弃，建议改用 `tool_choice`
- `max_tokens`
  - 已废弃，建议改用 `max_completion_tokens`

---

## 3. `messages` 消息格式

`messages` 按顺序组成对话上下文。

支持的常见 `role`：

- `developer`
- `system`
- `user`
- `assistant`
- `tool`

说明：

- OpenAI 官方对较新模型建议优先使用 `developer` 指令。
- `tool` 角色用于把外部工具执行结果回填给模型。

## 3.1 `developer` / `system`

通常为纯文本：

```json
{
  "role": "system",
  "content": "You are a helpful assistant."
}
```

## 3.2 `user`

可以是纯文本，也可以是内容块数组。

纯文本示例：

```json
{
  "role": "user",
  "content": "请总结下面代码。"
}
```

多模态内容块示例：

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "描述这张图片"},
    {
      "type": "image_url",
      "image_url": {
        "url": "https://example.com/demo.jpg",
        "detail": "auto"
      }
    }
  ]
}
```

常见内容块类型：

- `text`
- `image_url`
- `input_audio`
- `file`

## 3.3 `assistant`

普通文本回复：

```json
{
  "role": "assistant",
  "content": "这里是模型的回复。"
}
```

若模型要调用工具，则 `assistant` 消息会带 `tool_calls`：

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [
    {
      "id": "call_abc123",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"location\":\"Boston, MA\"}"
      }
    }
  ]
}
```

## 3.4 `tool`

工具结果回填格式：

```json
{
  "role": "tool",
  "tool_call_id": "call_abc123",
  "content": "{\"temperature\":23,\"unit\":\"celsius\"}"
}
```

---

## 4. 工具与结构化输出

## 4.1 `tools`

最常见的是函数工具：

```json
{
  "type": "function",
  "function": {
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
}
```

说明：

- `name` 最长 64 字符，只能包含字母、数字、下划线、短横线
- `parameters` 是 JSON Schema
- `strict: true` 时要求严格遵循 schema

## 4.2 `tool_choice`

常见模式：

- `"none"`
- `"auto"`
- `"required"`
- `{"type":"function","function":{"name":"get_weather"}}`
- `{"type":"allowed_tools","allowed_tools":[...]}`

## 4.3 `parallel_tool_calls`

- 类型：`boolean`
- 默认：`true`
- 含义：是否允许模型并行调用多个工具

## 4.4 `response_format`

用于控制输出格式：

- `{ "type": "text" }`
- `{ "type": "json_object" }`
- `{ "type": "json_schema", "json_schema": { ... } }`

示例：

```json
{
  "response_format": {
    "type": "json_schema",
    "json_schema": {
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
    }
  }
}
```

---

## 5. 响应体规范

## 5.1 非流式响应

典型返回对象：

```json
{
  "id": "chatcmpl_xxx",
  "object": "chat.completion",
  "created": 1741569952,
  "model": "gpt-5.4",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?",
        "refusal": null,
        "annotations": []
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 9,
    "total_tokens": 18
  },
  "system_fingerprint": null
}
```

常见 `finish_reason`：

- `stop`
- `length`
- `content_filter`
- `tool_calls`
- `function_call`
  - 已废弃，仅兼容旧格式

## 5.2 流式响应

当 `stream=true` 时，返回 SSE：

```text
data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[...]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[...]}

data: [DONE]
```

关键特征：

- `object` 为 `chat.completion.chunk`
- 增量内容在 `choices[0].delta`
- 最后一帧 `finish_reason` 才会落定
- 若启用 `stream_options.include_usage=true`，在 `[DONE]` 前可能多一帧 usage 汇总

---

## 6. `curl` 收发示例

以下示例均使用：

```bash
export OPENAI_API_KEY="你的密钥"
```

## 6.1 最小文本请求

### 请求

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "请用一句话解释递归。"}
    ]
  }'
```

### 响应示例

```json
{
  "id": "chatcmpl_123",
  "object": "chat.completion",
  "created": 1741569952,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "递归是函数在定义中调用自身，用更小的同类问题逐步求解原问题。"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 24,
    "total_tokens": 44
  }
}
```

## 6.2 函数工具调用

### 请求

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "波士顿今天的天气怎么样？"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
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
      }
    ],
    "tool_choice": "auto",
    "parallel_tool_calls": true
  }'
```

### 响应示例

```json
{
  "id": "chatcmpl_456",
  "object": "chat.completion",
  "created": 1741569999,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\":\"Boston, MA\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

## 6.3 工具结果回填

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "波士顿今天的天气怎么样？"},
      {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\":\"Boston, MA\"}"
            }
          }
        ]
      },
      {
        "role": "tool",
        "tool_call_id": "call_abc123",
        "content": "{\"temperature\":23,\"unit\":\"celsius\"}"
      }
    ]
  }'
```

## 6.4 流式请求

### 请求

```bash
curl -N https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "stream": true,
    "messages": [
      {"role": "user", "content": "请列出三种常见排序算法。"}
    ]
  }'
```

### SSE 响应示例

```text
data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"冒泡排序、"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"快速排序、"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"归并排序。"},"finish_reason":null}]}

data: {"id":"chatcmpl_xxx","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

---

## 7. 官方来源

- API Overview  
  https://developers.openai.com/api/reference/overview
- Create chat completion  
  https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create
- Chat Completions streaming events  
  https://developers.openai.com/api/reference/resources/chat/subresources/completions/streaming-events
- Function calling  
  https://platform.openai.com/docs/guides/function-calling
- Migrate to the Responses API  
  https://developers.openai.com/api/docs/guides/migrate-to-responses
