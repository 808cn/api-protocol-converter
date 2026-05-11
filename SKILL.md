---
name: api-protocol-converter
description: OpenAI (Chat Completions / Responses) 与 Anthropic (Messages) API 协议转换专家。覆盖三端点间的请求归一化、响应编码、流式事件互转、工具调用协议映射、以及错误处理。当用户需要：(1) 实现 OpenAI 格式到 Claude 格式的协议代理/转换层；(2) 理解 Chat Completions、Responses、Messages 三端点的字段差异与转换规则；(3) 编写协议转换代理或网关；(4) 测试协议转换的正确性；(5) 排查协议转换中的常见错误时使用。
---

# API 协议转换器

实现 OpenAI (`/v1/chat/completions`、`/v1/responses`) 与 Anthropic (`/v1/messages`) 三端点之间的协议转换。

## 核心原则

1. **不要写 6 组两两直连转换器**——先归一化到统一中间结构 (IR)，再从 IR 编码回目标端点
2. **对外永远是客户端期望的格式**，对内永远是目标 API 的原生格式
3. **不支持的能力必须显式报错**，不要静默吞掉
4. **先实现公共交集**（文本、工具调用、流式），再扩展高级能力

## 三端点快速对照

| 维度 | Chat Completions | Responses | Messages (Claude) |
|---|---|---|---|
| 端点 | `/v1/chat/completions` | `/v1/responses` | `/v1/messages` |
| 认证 | `Authorization: Bearer KEY` | `Authorization: Bearer KEY` | `x-api-key: KEY` |
| system 指令 | `messages[].role=system` | `instructions` | 顶层 `system` 字段 |
| 工具定义 | `tools[].function.parameters` | `tools[].parameters` | `tools[].input_schema` |
| 工具选择 | `"none"/"auto"/"required"/指定函数` | 同 Chat | `{type:none/auto/any/tool}` |
| 工具调用输出 | `message.tool_calls[]` | `output[].type=function_call` | `content[].type=tool_use` |
| 工具结果回填 | `role:"tool"` + `tool_call_id` | `function_call_output` | `role:"user"` + `tool_result` |
| 参数类型 | JSON 字符串 | JSON 字符串 | **对象** |
| 停止原因 | `finish_reason: stop/tool_calls/length` | `status: completed` | `stop_reason: end_turn/tool_use/max_tokens` |
| 流式格式 | `data: {chunk}` + `data: [DONE]` | `event: response.*` 系列 | `event: message_start/content_block_delta/message_stop` |

## 五大关键转换点

### 1. system 提取与放置

- Chat `role:system` / Responses `instructions` -> Messages 顶层 `system`
- Messages `system` -> Chat `role:system` / Responses `instructions`
- 多条 system 消息用 `\n\n` 合并

### 2. 工具定义映射

Chat/Responses 格式：
```json
{"type": "function", "function": {"name": "get_weather", "parameters": {...}}}
```
或 Responses 格式：
```json
{"type": "function", "name": "get_weather", "parameters": {...}}
```
-> Messages 格式：
```json
{"name": "get_weather", "input_schema": {...}}
```

关键：去掉 `type:"function"` 包装，`parameters -> input_schema`。

### 3. 工具调用与回填

**最关键的转换——最容易出错的部分。**

Chat 工具调用：
```json
{"role": "assistant", "tool_calls": [{"id": "call_1", "function": {"name": "get_weather", "arguments": "{\"city\":\"上海\"}"}}]}
```
-> Messages：
```json
{"role": "assistant", "content": [{"type": "tool_use", "id": "call_1", "name": "get_weather", "input": {"city": "上海"}}]}
```

Chat 工具结果：
```json
{"role": "tool", "tool_call_id": "call_1", "content": "{\"temp\":26}"}
```
-> Messages：
```json
{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "call_1", "content": "{\"temp\":26}"}]}
```

**硬规则：**
- `arguments` (字符串) <-> `input` (对象)：必须做 `JSON.parse/stringify`
- `role:tool` 不能直接传给 Claude，必须包进 `role:user` 的 `tool_result`
- Claude 顺序要求最严格：`assistant tool_use` 后必须紧跟 `user tool_result`
- 同一条 `user` 消息里，`tool_result` 排前面，文本排后面

### 4. tool_choice 映射

| 来源 | IR 统一值 | Chat | Responses | Messages |
|---|---|---|---|---|
| 不允许工具 | `{type:"none"}` | `"none"` | `"none"` | `{type:"none"}` |
| 自动选择 | `{type:"auto"}` | `"auto"` | `"auto"` | `{type:"auto"}` |
| 必须调用 | `{type:"required"}` | `"required"` | `"required"` | `{type:"any"}` |
| 指定函数 | `{type:"tool",name:"x"}` | `{type:function,name:x}` | `{type:function,name:x}` | `{type:"tool",name:"x"}` |

并行工具：Chat `parallel_tool_calls=false` <-> Messages `disable_parallel_tool_use=true`（语义反向）。

### 5. 流式事件互转

**建议统一为 `StreamEventIR` 再编码，不要直接改字段。**

核心事件映射：

| IR 事件 | Chat SSE | Responses SSE | Messages SSE |
|---|---|---|---|
| `message_start` | `delta.role=assistant` | `response.output_item.added` | `event: message_start` |
| `text_delta` | `delta.content` | `response.output_text.delta` | `content_block_delta(text_delta)` |
| `tool_call_start` | `delta.tool_calls[{index,id,name}]` | `response.output_item.added(fc)` | `content_block_start(tool_use)` |
| `tool_call_arguments_delta` | `delta.tool_calls[{index,arguments}]` | `response.function_call_arguments.delta` | `content_block_delta(input_json_delta)` |
| `message_done(stop)` | `finish_reason=stop` + `[DONE]` | `response.completed` | `message_delta` + `message_stop` |
| `message_done(tool_calls)` | `finish_reason=tool_calls` + `[DONE]` | `response.completed` | `message_delta(tool_use)` + `message_stop` |

流式转换必须维护的状态：content block index、tool call index、已累计的 partial_json、当前 stop_reason。

## curl 快速对照示例

### Chat Completions -> Messages 请求转换

**原始 Chat 请求：**
```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.4",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "查上海天气"},
      {"role": "assistant", "content": null, "tool_calls": [{"id":"call_1","type":"function","function":{"name":"get_weather","arguments":"{\"city\":\"上海\"}"}}]},
      {"role": "tool", "tool_call_id": "call_1", "content": "{\"temp\":26}"}
    ],
    "tools": [{"type":"function","function":{"name":"get_weather","parameters":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}}]
  }'
```

**转换后的 Messages 请求：**
```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-7",
    "max_tokens": 1024,
    "system": "You are a helpful assistant.",
    "messages": [
      {"role": "user", "content": "查上海天气"},
      {"role": "assistant", "content": [{"type":"tool_use","id":"call_1","name":"get_weather","input":{"city":"上海"}}]},
      {"role": "user", "content": [{"type":"tool_result","tool_use_id":"call_1","content":"{\"temp\":26}"}]}
    ],
    "tools": [{"name":"get_weather","input_schema":{"type":"object","properties":{"city":{"type":"string"}},"required":["city"]}}]
  }'
```

### Messages -> Chat Completions 响应转换

**Claude 原生响应：**
```json
{"type":"message","role":"assistant","content":[{"type":"text","text":"我来查一下。"},{"type":"tool_use","id":"toolu_123","name":"get_weather","input":{"city":"上海"}}],"stop_reason":"tool_use"}
```

**转换后的 Chat 响应：**
```json
{"object":"chat.completion","choices":[{"index":0,"message":{"role":"assistant","content":"我来查一下。","tool_calls":[{"id":"toolu_123","type":"function","function":{"name":"get_weather","arguments":"{\"city\":\"上海\"}"}}]},"finish_reason":"tool_calls"}]}
```

## 实现路线建议

不要一开始就三端同时开工。推荐顺序：

1. Chat <-> Messages 非流式文本
2. Chat <-> Messages 非流式工具调用
3. Chat <-> Messages 流式文本
4. Chat <-> Messages 流式工具调用
5. Responses <-> IR 非流式
6. Responses <-> IR 流式
7. 最后补 Responses <-> Chat/Messages 完整链路

## 最小测试矩阵

至少覆盖以下 12 条才能宣称"三端互转已兼容"：

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

## 详细参考资料

根据需要按主题加载：

- **三端点完整互转编码指南（IR 设计、归一化规则、流式映射、模块划分）**：参阅 [three-protocol-guide.md](references/three-protocol-guide.md)
- **Chat Completions 协议完整规范**：参阅 [chat-completions-spec.md](references/chat-completions-spec.md)
- **Messages 协议完整规范**：参阅 [messages-spec.md](references/messages-spec.md)
- **Responses 协议完整规范**：参阅 [responses-spec.md](references/responses-spec.md)
- **Chat vs Messages 两种协议核心差异与转换关键点**：参阅 [chat-vs-messages-diff.md](references/chat-vs-messages-diff.md)
- **OpenAI 格式 API 可执行 curl 测试用例**：参阅 [openai-test-cases.md](references/openai-test-cases.md)
- **Claude 格式 API 可执行 curl 测试用例**：参阅 [claude-test-cases.md](references/claude-test-cases.md)

## 不在第一版互转范围的内容

以下能力需要单独做能力分层，不要直接纳入公共交集：

- Responses 的内置工具、reasoning 项、MCP 专属项
- Messages 的 thinking、server tools、`pause_turn`
- 各端点的音频专属输出
- 各端点的 provider 专属实验字段
- `previous_response_id` 的跨端点续接（需要本地会话存储支持）