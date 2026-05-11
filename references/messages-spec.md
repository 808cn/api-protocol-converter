# `/v1/messages` 协议规范

本文基于 Anthropic Claude 官方文档整理，聚焦：

- `POST /v1/messages`

说明：

- 以下内容以我在 **2026-04-12** 查阅的 Claude 官方文档为准。
- 下文优先覆盖最常用、最稳定、最容易出兼容问题的字段与行为；Beta 能力、模型专属能力、服务端工具细节可能继续演进。

## 1. 通用 HTTP 约定

### 基础地址

```text
https://api.anthropic.com
```

### 请求方法

```http
POST /v1/messages
```

### 必需请求头

```http
x-api-key: ANTHROPIC_API_KEY
anthropic-version: 2023-06-01
content-type: application/json
```

说明：

- `x-api-key` 为 Claude API 密钥。
- `anthropic-version` 是必填版本头；绝大多数兼容实现也要求该头存在。
- `content-type` 通常为 `application/json`。

### 常见可选请求头

- `anthropic-beta: <beta1>,<beta2>`
  - 开启 Beta 特性，例如压缩、细粒度工具流等
- `request-id`
  - 不是请求头，而是服务端响应头；排障时非常重要

### 典型响应头

- `request-id: req_xxx`
- `anthropic-organization-id: org_xxx`

### 错误响应形态

Claude 错误对象和 OpenAI 风格不同，常见形态如下：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "messages: field required"
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

与 OpenAI 风格相比，常见差异：

- 顶层有 `type: "error"`
- 错误详情放在 `error` 对象中
- 通常没有 `param`、`code` 这类字段
- 额外返回 `request_id` 便于追踪

---

## 2. 请求体规范

## 2.1 必填字段

- `model: string`
- `max_tokens: number`
- `messages: array`

最小请求体：

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 1024,
  "messages": [
    {"role": "user", "content": "你好"}
  ]
}
```

## 2.2 常用可选字段

- `system: string | array`
  - 系统提示词
  - 注意：`/v1/messages` **没有** `system` 角色消息，系统提示必须走顶层 `system`
- `stream: boolean`
  - `true` 时返回 SSE 流
- `temperature: number`
  - 范围通常是 `0.0` 到 `1.0`
- `top_k: number`
  - 高级采样参数
- `top_p: number`
  - nucleus sampling，高级采样参数
- `stop_sequences: string[]`
  - 自定义停止序列
- `metadata: object`
  - 常见为 `{"user_id":"opaque-id"}`
- `service_tier: "auto" | "standard_only"`
  - 是否允许优先容量
- `inference_geo: string`
  - 推理地理区域
- `thinking: object`
  - 扩展思考配置
- `tools: array`
  - 工具定义
- `tool_choice: object`
  - 控制工具使用策略
- `output_config: object`
  - 输出格式控制，最常见是 `json_schema`
- `context_management: object`
  - 某些 Beta 特性使用，例如压缩

## 2.3 字段细节说明

### `model`

- 类型：`string`
- 含义：指定 Claude 模型
- 示例：
  - `claude-sonnet-4-5`
  - `claude-sonnet-4-5-20250929`
  - `claude-opus-4-6`
  - `claude-haiku-4-5`

说明：

- 模型列表会持续更新，具体可用值以服务端实际支持为准。
- 某些旧模型已弃用，兼容层应避免把模型名写死在校验逻辑里。

### `max_tokens`

- 类型：`number`
- 含义：本次最大输出 token 上限

说明：

- 模型可能在到达上限前自然停止。
- 启用 `thinking` 时，思考 token 也计入 `max_tokens`。
- 对长输出任务，不建议把 `max_tokens` 设很大却仍使用非流式请求。

### `system`

可用字符串：

```json
{
  "system": "你是一个严谨的技术助手。"
}
```

也可用文本块数组：

```json
{
  "system": [
    {"type": "text", "text": "你是一个严谨的技术助手。"}
  ]
}
```

关键点：

- `/v1/messages` 请求中的 `messages[].role` 只应使用 `user` 和 `assistant`。
- 不应在 `messages` 内放 `{"role":"system"}`。

### `stream`

- 类型：`boolean`
- 默认：`false`
- 含义：是否启用 SSE 流式输出

### `temperature`

- 类型：`number`
- 常见范围：`0.0` 到 `1.0`
- 默认：`1.0`

经验建议：

- 事实问答、分类、抽取：`0` 到 `0.2`
- 普通聊天、代码解释：`0.2` 到 `0.7`
- 创意生成：`0.7` 到 `1.0`

说明：

- 即使 `temperature=0`，结果也不保证完全确定。
- 一般只调 `temperature` 即可，不建议和 `top_p` 同时大幅调整。

### `top_k` / `top_p`

- `top_k`：只从概率最高的前 K 个 token 中采样
- `top_p`：只从累计概率达到阈值的 token 集中采样

说明：

- 两者都属于高级参数。
- Claude 官方建议：大多数场景只用 `temperature`。

### `stop_sequences`

- 类型：`string[]`
- 含义：命中自定义停止序列时提前结束生成

示例：

```json
{
  "stop_sequences": ["\n\nHuman:", "<END>"]
}
```

命中后响应里通常会出现：

- `stop_reason: "stop_sequence"`
- `stop_sequence: "实际命中的序列"`

### `metadata`

最常见：

```json
{
  "metadata": {
    "user_id": "7d4f3e94-bfe2-4b24-8b2b-opaque"
  }
}
```

说明：

- `user_id` 建议使用 UUID、哈希或其他不透明标识。
- 不要直接传姓名、邮箱、手机号等个人可识别信息。

### `service_tier`

- 可选值：`"auto"`、`"standard_only"`
- 含义：是否允许使用优先容量

### `thinking`

可选形态：

- `{"type":"enabled","budget_tokens":2048}`
- `{"type":"adaptive"}`
- `{"type":"disabled"}`

示例：

```json
{
  "thinking": {
    "type": "enabled",
    "budget_tokens": 2048
  }
}
```

说明：

- `enabled` 时通常要求 `budget_tokens >= 1024`
- `budget_tokens` 必须小于 `max_tokens`
- 返回中会出现 `thinking` / `redacted_thinking` 内容块
- 多轮时，历史思考块存在特殊保留规则，特别是与工具调用组合时不能随意改写

### `output_config`

最常见是结构化 JSON 输出：

```json
{
  "output_config": {
    "format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "properties": {
          "answer": {"type": "string"},
          "confidence": {"type": "number"}
        },
        "required": ["answer", "confidence"],
        "additionalProperties": false
      }
    }
  }
}
```

可选字段中还可能出现：

- `effort: "low" | "medium" | "high" | "max"`

说明：

- `output_config.format` 是 Claude 侧结构化输出的核心字段。
- 若你在做兼容层，不应错误地把 OpenAI 的 `response_format` 直接原样转发到 Claude 原生接口。

### `context_management`

这是较新的上下文管理入口，部分能力需要配合 `anthropic-beta` 使用。

例如服务端压缩：

```json
{
  "context_management": {
    "edits": [
      {
        "type": "compact_20260112"
      }
    ]
  }
}
```

通常还要带：

```http
anthropic-beta: compact-2026-01-12
```

---

## 3. `messages` 消息格式

`messages` 是 Claude Messages API 的核心上下文数组。

## 3.1 基本规则

- 每项都必须有 `role` 和 `content`
- 合法常见角色只有：`user`、`assistant`
- 没有 `system` 角色
- 连续的 `user` 或连续的 `assistant` 轮次会被服务端合并理解
- 单次请求最多可携带 `100000` 条消息

最小示例：

```json
[
  {"role": "user", "content": "Hello, Claude"}
]
```

多轮示例：

```json
[
  {"role": "user", "content": "你好"},
  {"role": "assistant", "content": "你好，我是 Claude。"},
  {"role": "user", "content": "请解释什么是递归。"}
]
```

## 3.2 `content` 可以是字符串或内容块数组

下面两种写法等价：

```json
{"role": "user", "content": "你好"}
```

```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "你好"}
  ]
}
```

兼容实现建议：

- 接收时同时兼容字符串和数组
- 存储或转发前可统一标准化为数组形式

## 3.3 输入内容块类型

常见输入块包括：

- `text`
- `image`
- `document`
- `tool_result`
- 历史回传场景下的 `thinking`
- 历史回传场景下的 `redacted_thinking`
- 历史回传场景下的 `tool_use`
- 历史回传场景下的 `server_tool_use`
- 历史回传场景下的服务端工具结果块

说明：

- 新请求里最常用的仍然是 `text`、`image`、`document`。
- 在多轮工具调用中，调用方需要把上一轮 assistant 返回的 `tool_use` 原样保留，再追加新的 `tool_result`。
- 启用 thinking 后，某些轮次必须保留对应 thinking 块，不能自行重写。

## 3.4 `text` 文本块

示例：

```json
{
  "type": "text",
  "text": "请总结这份文档"
}
```

可选扩展：

- `cache_control`
- `citations`

`cache_control` 示例：

```json
{
  "type": "text",
  "text": "这是可缓存的长上下文",
  "cache_control": {
    "type": "ephemeral",
    "ttl": "5m"
  }
}
```

## 3.5 `image` 图片块

支持两类来源：

- `source.type = "base64"`
- `source.type = "url"`

URL 示例：

```json
{
  "type": "image",
  "source": {
    "type": "url",
    "url": "https://example.com/demo.png"
  }
}
```

Base64 示例：

```json
{
  "type": "image",
  "source": {
    "type": "base64",
    "media_type": "image/png",
    "data": "iVBORw0KGgoAAAANSUhEUgAA..."
  }
}
```

常见 `media_type`：

- `image/jpeg`
- `image/png`
- `image/gif`
- `image/webp`

## 3.6 `document` 文档块

支持常见来源：

- PDF Base64
- 纯文本
- URL PDF
- 内嵌内容块

PDF 示例：

```json
{
  "type": "document",
  "source": {
    "type": "base64",
    "media_type": "application/pdf",
    "data": "JVBERi0xLjQKJc..."
  }
}
```

纯文本示例：

```json
{
  "type": "document",
  "source": {
    "type": "text",
    "media_type": "text/plain",
    "data": "这里是一篇很长的文档内容"
  }
}
```

## 3.7 `assistant` 预填充（prefill）

Claude Messages API 支持把最后一条 `assistant` 消息作为“已写出前缀”，让模型从该前缀继续生成。

示例：

```json
{
  "messages": [
    {"role": "user", "content": "太阳的希腊神名是什么？A. Sol B. Helios C. Sun"},
    {"role": "assistant", "content": "The best answer is ("}
  ]
}
```

此时返回可能是：

```json
{
  "content": [
    {"type": "text", "text": "B)"}
  ]
}
```

注意：

- 这不是所有模型都支持得完全一致。
- 官方文档明确指出：部分模型对 prefill 有限制，若不支持会返回 `400 invalid_request_error`。
- 兼容层如果做“补全续写”能力，必须针对具体模型做验证。

---

## 4. 工具调用规范

Claude 原生工具协议与 OpenAI `tool_calls` 风格不同。

Claude 的核心模式是：

1. 请求里提供 `tools`
2. 模型响应里返回 `content[].type = "tool_use"`
3. 调用方执行工具
4. 把结果作为下一轮 `user` 消息中的 `tool_result` 回填

## 4.1 `tools` 定义

最常见的客户端工具定义：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a city.",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name, e.g. Boston"
      }
    },
    "required": ["location"]
  }
}
```

常见字段：

- `name: string`
- `description: string`
- `input_schema: object`
- `strict: boolean`
- `eager_input_streaming: boolean`
- `cache_control`

说明：

- `input_schema` 使用 JSON Schema。
- `description` 越清晰，模型产出的工具参数通常越稳定。
- `strict: true` 有助于约束工具名和输入结构。

## 4.2 `tool_choice`

Claude 常见工具选择方式：

- `{"type":"auto"}`
  - 模型自行决定是否调用工具
- `{"type":"any"}`
  - 模型必须调用某个可用工具
- `{"type":"tool","name":"get_weather"}`
  - 强制调用指定工具
- `{"type":"none"}`
  - 禁止工具调用

这些模式还常配合：

- `disable_parallel_tool_use: true | false`

说明：

- `auto` + `disable_parallel_tool_use=true` 表示最多调用一个工具。
- `any` + `disable_parallel_tool_use=true` 表示必须且仅调用一个工具。

## 4.3 模型输出的 `tool_use`

当 Claude 决定调工具时，响应 `content` 中会出现：

```json
{
  "type": "tool_use",
  "id": "toolu_01D7FLrfh4GYq7yT1ULFeyMV",
  "name": "get_weather",
  "input": {
    "location": "Boston"
  }
}
```

与 OpenAI 差异：

- Claude 把工具调用放在 `content[]` 里，而不是 `message.tool_calls`
- `input` 是对象，不是 JSON 字符串
- 停止原因通常是 `stop_reason: "tool_use"`

## 4.4 `tool_result` 回填

工具执行完成后，需要在下一轮 `user` 消息里回传：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_01D7FLrfh4GYq7yT1ULFeyMV",
      "content": "{\"temperature\":23,\"unit\":\"celsius\"}"
    }
  ]
}
```

关键点：

- `tool_result` 要放在 `role = "user"` 的消息里，不是单独的 `tool` 角色
- `tool_use_id` 必须对应上一轮的工具调用 ID
- `content` 可以是字符串，也可以是内容块数组
- 可设置 `is_error: true` 表示工具执行失败

错误回填示例：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_failed_001",
      "is_error": true,
      "content": "File not found"
    }
  ]
}
```

## 4.5 完整工具多轮示例

### 第一步：请求模型决定是否调用工具

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 2048,
  "tools": [
    {
      "name": "Read",
      "description": "Read the contents of a file at the given path.",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {"type": "string"}
        },
        "required": ["file_path"]
      }
    }
  ],
  "messages": [
    {"role": "user", "content": "Please read /app/config.json"}
  ]
}
```

### 第二步：模型返回 `tool_use`

```json
{
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_demo_read_config",
      "name": "Read",
      "input": {
        "file_path": "/app/config.json"
      }
    }
  ],
  "stop_reason": "tool_use"
}
```

### 第三步：调用方执行工具并回填 `tool_result`

```json
{
  "model": "claude-sonnet-4-5",
  "max_tokens": 2048,
  "tools": [
    {
      "name": "Read",
      "description": "Read the contents of a file at the given path.",
      "input_schema": {
        "type": "object",
        "properties": {
          "file_path": {"type": "string"}
        },
        "required": ["file_path"]
      }
    }
  ],
  "messages": [
    {"role": "user", "content": "Please read /app/config.json"},
    {"role": "assistant", "content": [
      {
        "type": "tool_use",
        "id": "toolu_demo_read_config",
        "name": "Read",
        "input": {"file_path": "/app/config.json"}
      }
    ]},
    {"role": "user", "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_demo_read_config",
        "content": "{\"port\":3010,\"model\":\"claude-sonnet-4.6\"}"
      }
    ]}
  ]
}
```

---

## 5. 响应体规范

非流式返回的顶层对象类型固定为 `message`。

典型示例：

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "Hello! How can I assist you today?"
    }
  ],
  "model": "claude-sonnet-4-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 12,
    "output_tokens": 8
  }
}
```

## 5.1 顶层字段

- `id: string`
- `type: "message"`
- `role: "assistant"`
- `content: array`
- `model: string`
- `stop_reason: string | null`
- `stop_sequence: string | null`
- `usage: object`

## 5.2 `content` 常见输出块

最常见输出块：

- `text`
- `tool_use`
- `thinking`
- `redacted_thinking`
- `server_tool_use`
- 服务端工具结果块，例如 `web_search_tool_result`

### `text`

```json
{
  "type": "text",
  "text": "这里是回答内容"
}
```

### `thinking`

```json
{
  "type": "thinking",
  "thinking": "我需要先分析问题，再给出答案。",
  "signature": "EqQBCgIYAhI..."
}
```

说明：

- 仅在启用扩展思考时出现。
- 如果后续轮次涉及工具继续推理，这些块可能必须原样带回。

### `redacted_thinking`

```json
{
  "type": "redacted_thinking",
  "data": "..."
}
```

### `tool_use`

```json
{
  "type": "tool_use",
  "id": "toolu_123",
  "name": "Read",
  "input": {
    "file_path": "/tmp/demo.txt"
  }
}
```

## 5.3 `stop_reason`

常见值：

- `end_turn`
  - 正常结束本轮回答
- `max_tokens`
  - 达到输出上限
- `stop_sequence`
  - 命中自定义停止序列
- `tool_use`
  - 模型要求调用工具
- `pause_turn`
  - 长轮次暂停，可把当前 assistant 响应原样带回后继续
- `refusal`
  - 模型因安全/策略原因拒绝继续

兼容实现重点：

- 不要只判断 `end_turn`。
- 遇到 `tool_use` 时应进入工具执行分支。
- 遇到 `pause_turn` 时应支持续跑，而不是直接报错。

## 5.4 `usage`

常见字段：

- `input_tokens`
- `output_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`
- `cache_creation`
- `inference_geo`
- `server_tool_use`
- `service_tier`

示例：

```json
{
  "usage": {
    "input_tokens": 1200,
    "output_tokens": 256,
    "cache_creation_input_tokens": 800,
    "cache_read_input_tokens": 0,
    "service_tier": "standard"
  }
}
```

说明：

- Claude 的计费 token 和可见文本并非一一对应。
- 即使输出看起来很短，`output_tokens` 也可能不是 0。

---

## 6. 流式响应规范

当 `stream=true` 时，`/v1/messages` 返回的是 **SSE 事件流**。

## 6.1 典型事件顺序

Claude 官方推荐按以下顺序理解流：

1. `message_start`
2. `content_block_start`
3. 若干个 `content_block_delta`
4. `content_block_stop`
5. 一个或多个 `message_delta`
6. `message_stop`

期间还可能插入：

- `ping`
- `error`

## 6.2 典型 SSE 示例

```text
event: message_start
data: {"type":"message_start","message":{"id":"msg_xxx","type":"message","role":"assistant","content":[],"model":"claude-sonnet-4-5","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":15}}

event: message_stop
data: {"type":"message_stop"}
```

## 6.3 文本增量

文本块增量：

```text
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"部分文本"}}
```

## 6.4 工具参数增量

当输出块是 `tool_use` 时，参数会以部分 JSON 形式流出：

```text
event: content_block_delta
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"location\": \"San Fra"}}
```

关键点：

- 流里看到的是 `partial_json` 字符串片段
- 最终完整的 `tool_use.input` 仍应是对象
- 客户端要在 `content_block_stop` 后拼装完整 JSON 再解析

## 6.5 thinking 增量

启用扩展思考时，增量类型可能为：

- `thinking_delta`
- `signature_delta`

示例：

```text
event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"先分析输入条件..."}}
```

## 6.6 流式错误事件

即使 HTTP 状态码已经是 `200`，流中仍可能出现错误事件：

```text
event: error
data: {"type":"error","error":{"type":"overloaded_error","message":"Overloaded"}}
```

这意味着：

- 流式客户端不能只看 HTTP 状态码
- 必须监听 `event: error`

---

## 7. `curl` 收发示例

以下示例均使用：

```bash
export ANTHROPIC_API_KEY="你的密钥"
```

## 7.1 最小文本请求

### 请求

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "请用一句话解释递归。"}
    ]
  }'
```

### 响应示例

```json
{
  "id": "msg_123",
  "type": "message",
  "role": "assistant",
  "content": [
    {
      "type": "text",
      "text": "递归是函数通过调用自身来把大问题逐步拆成更小同类问题的求解方法。"
    }
  ],
  "model": "claude-sonnet-4-5",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "usage": {
    "input_tokens": 18,
    "output_tokens": 24
  }
}
```

## 7.2 带 `system` 的请求

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "system": "你是一个严谨的 API 文档助手，输出尽量简洁。",
    "messages": [
      {"role": "user", "content": "解释 HTTP 401 和 403 的差异。"}
    ]
  }'
```

## 7.3 多模态图片输入

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "描述这张图片的主要内容"},
          {
            "type": "image",
            "source": {
              "type": "url",
              "url": "https://example.com/demo.jpg"
            }
          }
        ]
      }
    ]
  }'
```

## 7.4 非流式工具调用

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 2048,
    "tool_choice": {"type": "any"},
    "tools": [
      {
        "name": "Read",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"}
          },
          "required": ["file_path"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Please read the file at /project/src/index.ts"}
    ]
  }'
```

期望关注点：

- 返回对象 `type=message`
- `content[]` 内出现 `type=tool_use`
- `stop_reason=tool_use`

## 7.5 流式请求

```bash
curl -N https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 2048,
    "stream": true,
    "messages": [
      {"role": "user", "content": "请列出5种常见排序算法并说明时间复杂度。"}
    ]
  }'
```

期望关注点：

- SSE 中出现 `message_start`
- 出现 `content_block_delta`
- 结尾有 `message_delta`
- 最终有 `message_stop`

## 7.6 结构化 JSON 输出

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "max_tokens": 1024,
    "output_config": {
      "format": {
        "type": "json_schema",
        "schema": {
          "type": "object",
          "properties": {
            "summary": {"type": "string"},
            "keywords": {
              "type": "array",
              "items": {"type": "string"}
            }
          },
          "required": ["summary", "keywords"],
          "additionalProperties": false
        }
      }
    },
    "messages": [
      {"role": "user", "content": "总结 TCP 三次握手。"}
    ]
  }'
```

---

## 8. 与兼容实现相关的关键差异

这是做 Claude 兼容网关、协议转换、自动化测试时最容易出错的部分。

## 8.1 与 OpenAI Chat Completions 的核心差异

- 路径不同：Claude 是 `/v1/messages`
- 鉴权头不同：Claude 用 `x-api-key`
- 版本头强制：Claude 要求 `anthropic-version`
- 顶层系统提示不同：Claude 用 `system` 字段，不用 `messages[].role=system`
- 工具协议不同：Claude 用 `tool_use` / `tool_result`
- 响应对象不同：Claude 顶层 `type=message`
- 流式事件不同：Claude 是具名 SSE 事件，不是 `data: [DONE]`

## 8.2 常见兼容坑

### 坑 1：把 `system` 当成消息角色发送

错误示例：

```json
{
  "messages": [
    {"role": "system", "content": "You are helpful."},
    {"role": "user", "content": "Hello"}
  ]
}
```

正确做法：

```json
{
  "system": "You are helpful.",
  "messages": [
    {"role": "user", "content": "Hello"}
  ]
}
```

### 坑 2：把工具结果放成 `tool` 角色

Claude 原生不是：

```json
{"role":"tool", ...}
```

而是：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_xxx",
      "content": "..."
    }
  ]
}
```

### 坑 3：错误期待 OpenAI 风格 `tool_calls`

Claude 返回的是：

```json
{
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_xxx",
      "name": "Read",
      "input": {"file_path": "/tmp/a.txt"}
    }
  ]
}
```

不是：

```json
{
  "tool_calls": [...]
}
```

### 坑 4：流式解析只看 `data:` 不看 `event:`

Claude 流需要同时解析：

- `event: message_start`
- `event: content_block_delta`
- `event: message_delta`
- `event: message_stop`

如果你只按 OpenAI `[DONE]` 方式处理，通常会解析错误。

### 坑 5：工具流参数是部分 JSON 字符串

流式 `tool_use` 时，`input` 不会一次性给完整对象，而是多个 `partial_json` 片段。

### 坑 6：thinking 块被错误删改

启用扩展思考且进入工具链路时：

- 某些轮次必须保留相应 thinking 块
- 不能随意篡改签名
- 否则后续请求可能报校验错误或破坏推理连续性

### 坑 7：只识别 `end_turn`

兼容层应至少处理：

- `end_turn`
- `tool_use`
- `pause_turn`
- `max_tokens`
- `stop_sequence`
- `refusal`

---

## 9. 错误码与限制

常见 HTTP 错误：

- `400` `invalid_request_error`
- `401` `authentication_error`
- `403` `permission_error`
- `404` `not_found_error`
- `413` `request_too_large`
- `429` `rate_limit_error`
- `500` `api_error`
- `529` `overloaded_error`

## 9.1 请求大小限制

官方文档中的常见限制：

- Messages API：`32 MB`
- Token Counting API：`32 MB`
- Batch API：`256 MB`
- Files API：`500 MB`

超过限制时通常返回：

- HTTP `413`
- `request_too_large`

## 9.2 长请求建议

对于长输出、长推理、长时间工具链任务：

- 优先使用 `stream: true`
- 或使用 batch / 异步能力
- 不建议在非流式请求里设置极大 `max_tokens`

## 9.3 token 预估

在发送真实请求前，可使用：

```http
POST /v1/messages/count_tokens
```

它可以帮助估算：

- 普通文本消息 token
- 工具定义 token
- 图片与文档 token
- 长上下文是否接近窗口上限

### `/v1/messages/count_tokens` 最小请求示例

```bash
curl https://api.anthropic.com/v1/messages/count_tokens \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "system": "你是一个严谨的助手。",
    "messages": [
      {"role": "user", "content": "请解释什么是二叉搜索树。"}
    ]
  }'
```

返回示例：

```json
{
  "input_tokens": 31
}
```

说明：

- `count_tokens` 只做 token 估算，不生成回答。
- 它的输入结构与 `/v1/messages` 高度一致，通常也支持 `system`、`messages`、`tools`、图片、文档等内容。
- 返回核心字段通常是 `input_tokens`。

### `/v1/messages/count_tokens` 带工具的请求示例

```bash
curl https://api.anthropic.com/v1/messages/count_tokens \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "system": "你是一个代码助手。",
    "tools": [
      {
        "name": "Read",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"}
          },
          "required": ["file_path"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "请读取 /project/src/index.ts 并总结其功能。"}
    ]
  }'
```

返回示例：

```json
{
  "input_tokens": 137
}
```

说明：

- 工具定义本身也会计入输入 token。
- 如果你在做 Claude 兼容层，建议在真正转发 `/v1/messages` 前，先用 `/v1/messages/count_tokens` 做预算检查。

### `/v1/messages/count_tokens` 带多模态内容的请求示例

```bash
curl https://api.anthropic.com/v1/messages/count_tokens \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-5",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "描述这张图片。"},
          {
            "type": "image",
            "source": {
              "type": "url",
              "url": "https://example.com/demo.png"
            }
          }
        ]
      }
    ]
  }'
```

返回示例：

```json
{
  "input_tokens": 248
}
```

说明：

- 图片和文档同样会被计入 token 估算。
- 具体 token 数会受模型、图片尺寸、文档内容长度等因素影响。

---

## 10. 推荐校验清单

如果你在实现 Claude 兼容 API、网关转换器或自动化测试，可以至少校验以下项目：

1. 必须存在 `x-api-key`、`anthropic-version`、`content-type`
2. 请求体必须包含 `model`、`max_tokens`、`messages`
3. `messages[].role` 默认仅接受 `user`、`assistant`
4. 若有系统提示，应写入顶层 `system`
5. `content` 既要兼容字符串，也要兼容块数组
6. 非流式返回顶层 `type` 应为 `message`
7. 工具调用时应返回 `content[].type=tool_use`
8. 工具回填时应接受 `role=user + content[].type=tool_result`
9. 流式场景应正确解析 `message_start` 到 `message_stop` 全链路事件
10. 错误对象应兼容 Claude 原生 `{"type":"error","error":{...}}`

---

## 11. 总结

`/v1/messages` 是 Claude 原生对话接口，核心特点可以概括为：

- 顶层 `system`，不是 `system` 角色消息
- `messages` 只围绕 `user` / `assistant` 展开
- 工具调用使用 `tool_use` / `tool_result`，不是 OpenAI `tool_calls`
- 流式响应使用具名 SSE 事件
- 扩展思考、上下文管理、结构化输出和服务端工具是较新的增强能力

如果你的目标是做 Claude 原生接入，建议严格遵循本文字段语义。

如果你的目标是做 OpenAI 风格到 Claude 风格的协议转换，最关键的是正确处理：

- `system` 转换
- 工具协议转换
- SSE 事件转换
- `stop_reason` 语义映射
- thinking / tool_result / pause_turn 等 Claude 特有行为
