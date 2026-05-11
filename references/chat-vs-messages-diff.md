# 协议区别：OpenAI Chat Completions vs Anthropic Messages（Claude）

## 一句话结论

从 `replit.md` 到 `replit2.md`，真正增加的难点不是“多接一个模型”，而是：**要把 OpenAI Chat Completions 的工具调用协议，完整翻译成 Claude Messages 的工具协议，再把 Claude 的返回结果重新翻译回 OpenAI 格式**。  
最核心的区别集中在 **system 位置、工具定义结构、assistant 工具调用表达方式、tool result 回传方式、以及流式事件格式** 这 5 个方面。

---

## 1. 两种协议的主要区别

| 维度 | OpenAI Chat Completions | Anthropic Messages / Claude | 转换时的关键点 |
|---|---|---|---|
| 入口 | `/v1/chat/completions` | `/v1/messages` | 外部保持 OpenAI，内部 Claude 走 Messages |
| system 提示词 | 放在 `messages` 里的 `role: "system"` | 顶层 `system` 字段，不在 `messages` 里 | 需要先提取 system，再从消息数组移除 |
| 普通消息结构 | `messages[]`，常见 `content` 是字符串 | `messages[]`，`content` 可以是字符串或 block 数组 | Claude 分支建议统一转成 block 语义来处理 |
| 工具定义 | `tools[].function.parameters` | `tools[].input_schema` | `parameters -> input_schema` |
| 工具选择 | `tool_choice: none/auto/required/指定函数` | `tool_choice: {type:none/auto/any/tool}` | `required -> any`，指定函数 -> `{type:"tool",name}` |
| 并行工具 | `parallel_tool_calls: true/false` | `disable_parallel_tool_use: false/true` | 语义是反向字段 |
| assistant 发起工具调用 | `message.tool_calls[]` | `content[]` 里的 `tool_use` block | `tool_calls` 和 `tool_use` 需要双向映射 |
| 工具参数 | `function.arguments` 是 **JSON 字符串** | `tool_use.input` 是 **对象** | 必须做字符串 <-> 对象转换 |
| 工具结果回传 | 单独一条 `role: "tool"` 消息，带 `tool_call_id` | 放到下一条 `role: "user"` 消息里，`content[].type = "tool_result"` | 这是最容易漏掉、也是最关键的转换 |
| 工具调用停止原因 | `finish_reason: "tool_calls"` | `stop_reason: "tool_use"` | 需要做停止原因映射 |
| 流式格式 | OpenAI SSE：`data: {chat.completion.chunk}` | Anthropic SSE：`message_start/content_block_delta/message_stop...` | 不能直接透传，需要重组为 OpenAI chunk |
| 流式工具参数 | `delta.tool_calls[*].function.arguments` 增量 | `input_json_delta.partial_json` 增量 | 需要按 block/index 缓冲并持续输出 OpenAI 风格增量 |

---

## 2. 为什么 `replit.md` 的简单做法不够

`replit.md` 里的 Claude 分支只覆盖了：

- 提取 `system`
- 过滤成 `user/assistant`
- 普通文本流式增量转发
- 非流式文本结果包成 OpenAI 风格响应

这只够 **纯文本对话**。  
到了 `replit2.md`，要求已经升级为“真正兼容 OpenAI Chat Completions 协议”，因此至少还要补上：

1. `tools` 转换
2. `tool_choice` 转换
3. `assistant.tool_calls -> Claude tool_use` 转换
4. `role=tool -> Claude tool_result` 转换
5. Claude 非流式 `tool_use -> OpenAI tool_calls` 转换
6. Claude 流式 `input_json_delta -> OpenAI SSE tool_calls delta` 转换
7. 多轮工具调用历史重建

也就是说，**第一版是“文本代理”**，**第二版才是“协议翻译器”**。

---

## 3. 转换时最关键的实现部分

## 3.1 OpenAI -> Claude：请求转换

### A. system 提取

OpenAI：

```json
{"role":"system","content":"You are a helpful assistant."}
```

Claude 需要改成：

```json
{"system":"You are a helpful assistant."}
```

并且从 `messages` 中移除 system 消息。

### B. tools 定义转换

OpenAI：

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "Get weather",
    "parameters": {"type": "object", "properties": {...}}
  }
}
```

Claude：

```json
{
  "name": "get_weather",
  "description": "Get weather",
  "input_schema": {"type": "object", "properties": {...}}
}
```

最关键就是：

- 去掉外层 `type: "function"`
- `function.parameters -> input_schema`
- `function.name -> name`
- `function.description -> description`

### C. tool_choice 映射

建议按下面规则映射：

- OpenAI `"none"` -> Claude `{ "type": "none" }`
- OpenAI `"auto"` -> Claude `{ "type": "auto" }`
- OpenAI `"required"` -> Claude `{ "type": "any" }`
- OpenAI 指定函数 -> Claude `{ "type": "tool", "name": "函数名" }`
- OpenAI `parallel_tool_calls=false` -> Claude `disable_parallel_tool_use=true`

### D. 消息历史重建

这是最关键的一步。

OpenAI 多轮工具调用通常长这样：

```json
[
  {"role":"user","content":"查一下上海天气"},
  {
    "role":"assistant",
    "content": null,
    "tool_calls": [
      {
        "id":"call_1",
        "type":"function",
        "function":{"name":"get_weather","arguments":"{\"city\":\"上海\"}"}
      }
    ]
  },
  {"role":"tool","tool_call_id":"call_1","content":"{\"temp\":26}"}
]
```

Claude 需要改成：

```json
[
  {"role":"user","content":"查一下上海天气"},
  {
    "role":"assistant",
    "content":[
      {"type":"tool_use","id":"call_1","name":"get_weather","input":{"city":"上海"}}
    ]
  },
  {
    "role":"user",
    "content":[
      {"type":"tool_result","tool_use_id":"call_1","content":"{\"temp\":26}"}
    ]
  }
]
```

这里必须保证：

- `assistant.tool_calls[].id` 原样保留为 `tool_use.id`
- `function.arguments` 先解析 JSON 字符串，再放入 `input` 对象
- `role: "tool"` 不能直接传给 Claude，必须包进 `role: "user"` 的 `tool_result` block
- 如果 assistant 同时有文本和工具调用，要转成 Claude 的 `content[]` 混合块，不能丢文本
- Claude 对顺序要求更严格：`assistant` 的 `tool_use` 后面必须紧跟对应的 `user.tool_result`，中间不能夹别的消息

---

## 3.2 Claude -> OpenAI：非流式响应转换

Claude 返回工具调用时，常见是：

```json
{
  "role": "assistant",
  "content": [
    {"type":"text","text":"我来查一下。"},
    {"type":"tool_use","id":"toolu_123","name":"get_weather","input":{"city":"上海"}}
  ],
  "stop_reason": "tool_use"
}
```

对外必须转成 OpenAI：

```json
{
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "我来查一下。",
        "tool_calls": [
          {
            "id": "toolu_123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\":\"上海\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

关键规则：

- `tool_use.id -> tool_calls[].id`
- `tool_use.name -> tool_calls[].function.name`
- `tool_use.input` 对象 -> `function.arguments` JSON 字符串
- `stop_reason: tool_use -> finish_reason: tool_calls`
- 若只有普通文本，没有 `tool_use`，则正常输出 `message.content`
- 若文本和工具混合出现，`content` 和 `tool_calls` 两边都要保留

---

## 3.3 OpenAI tool result -> Claude tool_result

OpenAI 客户端在执行完工具后，常会回传：

```json
{
  "role": "tool",
  "tool_call_id": "call_1",
  "content": "{\"temp\":26}"
}
```

Claude 不能直接吃这条消息，必须转成：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "call_1",
      "content": "{\"temp\":26}"
    }
  ]
}
```

实现上要注意：

- `tool_call_id` 必须能匹配到上一轮 assistant 发出的工具调用 ID
- 多个连续的 `role=tool`，可以合并成一个 Claude `user` 消息里的多个 `tool_result` block
- 工具执行失败时，要补 `is_error: true`
- 如果同一条 Claude `user` 消息里既有 `tool_result` 又有文本，`tool_result` block 必须排在前面，文本排在后面

---

## 3.4 Claude -> OpenAI：流式转换

这是第二版里最难、也是最容易做错的部分。

### Claude 流式事件长这样

- `message_start`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`
- `message_delta`
- `message_stop`

其中：

- 文本增量是 `delta.type = "text_delta"`
- 工具参数增量是 `delta.type = "input_json_delta"`

### 对外必须发成 OpenAI SSE chunk

OpenAI 客户端希望看到的是：

```text
data: {"object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"}}]}

data: {"object":"chat.completion.chunk","choices":[{"delta":{"content":"我来"}}]}

data: {"object":"chat.completion.chunk","choices":[{"delta":{"tool_calls":[{"index":0,"id":"call_1","type":"function","function":{"name":"get_weather","arguments":""}}]}}]}

data: {"object":"chat.completion.chunk","choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"city\":\"上"}}]}}]}

data: {"object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"tool_calls"}]}

data: [DONE]
```

### 必须做的映射规则

1. **文本块开始/增量**  
   Claude `text_delta` -> OpenAI `delta.content`

2. **工具块开始**  
   当 Claude `content_block_start` 表示一个 `tool_use` block 时：
   - 立刻输出一个 OpenAI `delta.tool_calls` 起始块
   - 其中要带上：`index`、`id`、`type:function`、`function.name`
   - `function.arguments` 初始可为空字符串
   - 这些字段通常只应在该 tool call 的第一帧 delta 中发出，后续帧只继续补 arguments 增量

3. **工具参数增量**  
   Claude `input_json_delta.partial_json` -> OpenAI `delta.tool_calls[0].function.arguments` 增量字符串

4. **工具块结束**  
   需要按 `content block index` 收尾；必要时在本地缓冲完整参数，保证 JSON 拼接正确

5. **最终停止原因**  
   Claude `message_delta.stop_reason = tool_use` -> OpenAI `finish_reason = tool_calls`  
   Claude `end_turn` -> OpenAI `stop`

### 实现建议

流式转换时，至少维护以下状态：

- Claude `content block index -> tool call index`
- 每个工具调用的 `id`
- 每个工具调用已累计的 `partial_json`
- 当前响应是否出现过文本块 / 工具块
- 最终 `stop_reason`

如果没有这些状态，工具参数流式增量很容易乱序或丢失。

---

## 4. 实现时必须坚持的几个不变量

1. **对外永远是 OpenAI 格式**  
   客户端不应看到 Claude 原生的 `tool_use` / `tool_result`。

2. **对内 Claude 永远收到 Messages 原生格式**  
   不要把 OpenAI 的 `tool_calls` 或 `role=tool` 直接透传给 Claude。

3. **工具调用 ID 必须稳定**  
   OpenAI `tool_call_id`、Claude `tool_use_id`、流式增量里的 `id` 必须能互相对上。

4. **arguments 与 input 的类型必须正确**  
   OpenAI 是 JSON 字符串，Claude 是对象；这一步不能偷懒。

5. **多轮工具调用历史必须重建正确**  
   上一轮 assistant 的工具调用、下一轮 user 的工具结果，顺序必须严格对应。

---

## 5. 对这次任务最实用的结论

如果只是完成 `replit.md`，做一个“文本兼容代理”就够了。  
如果要满足 `replit2.md`，则需要实现一个真正的 **OpenAI Chat Completions <-> Claude Messages 协议转换层**，尤其要补齐下面这条链路：

- `tools`
- `tool_choice`
- `assistant.tool_calls`
- `role=tool`
- Claude `tool_use`
- Claude `tool_result`
- Claude `input_json_delta` 流式工具参数
- `finish_reason / stop_reason` 映射

一句话概括：**两种协议最大差异不在“文本聊天”，而在“工具调用的表达模型完全不同”**。

---

## 6. 参考

本地文档（同目录下）：

- [chat-completions-spec.md](chat-completions-spec.md)
- [messages-spec.md](messages-spec.md)

官方文档：

- OpenAI Chat Completions / Function Calling: <https://platform.openai.com/docs/guides/function-calling?api-mode=chat>
- Anthropic Messages API: <https://docs.anthropic.com/en/api/messages>
- Anthropic Streaming Messages: <https://docs.anthropic.com/en/api/messages-streaming>
