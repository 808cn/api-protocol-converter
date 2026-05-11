# OpenAI 格式 API 测试用例

本文档按以下两份协议规范整理：

- `/v1/chat/completions`
- `/v1/responses`

目标：

- 优先覆盖标准 OpenAI 协议请求
- 修正文档中原先不符合标准协议的示例
- 补充缺失的关键测试场景

说明：

- 第一部分是“标准协议用例”，优先用于验收 OpenAI 协议兼容性
- 最后一部分是“兼容性补充用例”，这些不是标准 OpenAI 协议，但可用于验证代理层的兼容/降级能力

## 环境变量

使用前请替换为你自己的目标地址、密钥和模型名：

```bash
export BASE_URL="https://api.openai.com"      # 替换为你的代理地址或 OpenAI 官方地址
export API_KEY="sk-your-api-key"               # 替换为你的 API 密钥
export MODEL="gpt-4o"                          # 替换为你要测试的模型
```

建议：

- 非流式请求使用 `curl -sS`
- 流式请求使用 `curl -N`
- 需要提取 JSON 字段的用例建议安装 `jq`

---

## 一、`/v1/chat/completions` 标准协议用例

### 1. 最小非流式请求

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "你好。"}
    ]
  }'
```

关注点：

- 返回 `object=chat.completion`
- `choices[0].message.role=assistant`
- `choices[0].message.content` 为普通文本

### 2. 流式请求

```bash
curl -N "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "stream": true,
    "messages": [
      {"role": "user", "content": "请列出 3 种常见排序算法。"}
    ]
  }'
```

关注点：

- 返回 OpenAI SSE 格式
- 最后一帧为 `data: [DONE]`

### 3. `developer` 指令 + 用户消息

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "developer", "content": "你是一个严谨的技术助手。"},
      {"role": "user", "content": "请用一句话解释什么是哈希表。"}
    ]
  }'
```

### 4. 多轮对话历史

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "My name is TestBot. Remember it."},
      {"role": "assistant", "content": "Got it. Your name is TestBot."},
      {"role": "user", "content": "What is my name?"}
    ]
  }'
```

### 5. 标准多模态 `image_url`

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "请描述这张图片。"},
          {
            "type": "image_url",
            "image_url": {
              "url": "https://example.com/demo.jpg",
              "detail": "auto"
            }
          }
        ]
      }
    ]
  }'
```

关注点：

- 请求结构符合 Chat Completions 规范
- 即使服务端走降级路径，也不应 500

### 6. 标准函数工具定义

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "Please read /project/src/index.ts"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "read_file",
          "description": "Read file contents",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {"type": "string"}
            },
            "required": ["file_path"]
          },
          "strict": true
        }
      }
    ]
  }'
```

关注点：

- 若模型决定调工具，应返回 `choices[0].message.tool_calls`
- `function.arguments` 必须是 JSON 字符串
- 参数字段应保留 `file_path`

### 7. `tool_choice="required"`

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "tool_choice": "required",
    "messages": [
      {
        "role": "user",
        "content": "Use the read_file tool to read /project/src/index.ts. Do not answer with plain text first."
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "read_file",
          "description": "Read file contents",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {"type": "string"}
            },
            "required": ["file_path"]
          }
        }
      }
    ]
  }'
```

### 8. `tool_choice` 指定函数

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "tool_choice": {
      "type": "function",
      "function": {
        "name": "read_file"
      }
    },
    "messages": [
      {"role": "user", "content": "Read /project/README.md"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "read_file",
          "description": "Read file contents",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {"type": "string"}
            },
            "required": ["file_path"]
          }
        }
      }
    ]
  }'
```

### 9. `role=tool` 完整回填流程

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "Read the config file at /app/config.json"},
      {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_read_config",
            "type": "function",
            "function": {
              "name": "read_file",
              "arguments": "{\"file_path\":\"/app/config.json\"}"
            }
          }
        ]
      },
      {
        "role": "tool",
        "tool_call_id": "call_read_config",
        "content": "{\"port\":3010,\"model\":\"claude-sonnet-4.6\",\"timeout\":120}"
      }
    ]
  }'
```

关注点：

- 服务端应接受标准 `role=tool`
- 最终回答应基于工具返回内容生成

### 10. 并行工具调用

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "parallel_tool_calls": true,
    "messages": [
      {"role": "user", "content": "北京和上海的天气如何？"}
    ],
    "tools": [
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
          }
        }
      }
    ]
  }'
```

关注点：

- 若模型选择并行工具调用，`tool_calls` 应为数组
- 每个工具调用都应有独立 `id`

### 11. 流式工具调用分块

```bash
curl -N "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "stream": true,
    "messages": [
      {
        "role": "user",
        "content": "Write a new file at /tmp/hello.ts with a TypeScript class HelloWorld and a greet() method."
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "write_file",
          "description": "Write file",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {"type": "string"},
              "content": {"type": "string"}
            },
            "required": ["file_path", "content"]
          }
        }
      }
    ]
  }'
```

关注点：

- 先出现 `tool_calls[].function.name`
- 再逐步出现 `tool_calls[].function.arguments`
- 最终 `finish_reason=tool_calls`

### 12. 多轮对话 + 多工具混合回填

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "请查询北京天气和当前北京时间。"},
      {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_weather_bj",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"location\":\"北京\"}"
            }
          },
          {
            "id": "call_time_bj",
            "type": "function",
            "function": {
              "name": "get_time",
              "arguments": "{\"timezone\":\"Asia/Shanghai\"}"
            }
          }
        ]
      },
      {
        "role": "tool",
        "tool_call_id": "call_weather_bj",
        "content": "{\"location\":\"北京\",\"weather\":\"晴\",\"temperature_c\":26,\"rain\":false}"
      },
      {
        "role": "tool",
        "tool_call_id": "call_time_bj",
        "content": "{\"timezone\":\"Asia/Shanghai\",\"local_time\":\"2026-04-10 09:30:00\"}"
      },
      {
        "role": "assistant",
        "content": "北京当前晴，26 摄氏度，现在时间是 2026-04-10 09:30:00。"
      },
      {
        "role": "user",
        "content": "基于刚才的天气和时间，再给我一句出行建议，并说明是否需要带伞。"
      }
    ],
    "tools": [
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
          }
        }
      },
      {
        "type": "function",
        "function": {
          "name": "get_time",
          "description": "Get local time",
          "parameters": {
            "type": "object",
            "properties": {
              "timezone": {"type": "string"}
            },
            "required": ["timezone"]
          }
        }
      }
    ]
  }'
```

关注点：

- 服务端应接受同一轮中的多个不同工具调用
- `role=tool` 回填后，后续 `user` 追问应能消费前文工具结果
- 最终回答不应丢失前一轮工具上下文

### 13. `response_format=json_object`

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "response_format": {"type": "json_object"},
    "messages": [
      {
        "role": "user",
        "content": "Return a JSON object with keys name and language for Python."
      }
    ]
  }'
```

### 14. `response_format=json_schema`

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "language_card",
        "schema": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "typed": {"type": "boolean"}
          },
          "required": ["name", "typed"],
          "additionalProperties": false
        }
      }
    },
    "messages": [
      {"role": "user", "content": "Describe TypeScript in JSON only."}
    ]
  }'
```

关注点：

- 返回正文不应包裹在 Markdown 代码块中
- 输出应尽量符合 schema

---

## 二、`/v1/responses` 标准协议用例

### 15. 最小字符串 `input`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": "Hello, how are you?"
  }'
```

关注点：

- 返回对象为 `object=response`
- `output[0].type=message`

### 16. `instructions` + `input`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "instructions": "你是一个严谨的技术助手。",
    "input": "请用一句话解释哈希表。"
  }'
```

### 17. 标准 `input` 数组

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": [
      {
        "role": "developer",
        "content": [
          {"type": "input_text", "text": "You are a coding assistant."}
        ]
      },
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "What is 2+2?"}
        ]
      },
      {
        "role": "assistant",
        "content": [
          {"type": "output_text", "text": "4", "annotations": []}
        ]
      },
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "And 3+3?"}
        ]
      }
    ]
  }'
```

### 18. `previous_response_id` 多轮上下文

先发第一轮：

```bash
RESP_ID=$(
  curl -sS "${BASE_URL}/v1/responses" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${API_KEY}" \
    -d '{
      "model": "'"${MODEL}"'",
      "input": "请记住这个代号：Alpha-42。只回复：已记住。"
    }' | jq -r '.id'
)

echo "${RESP_ID}"
```

再发第二轮：

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "previous_response_id": "'"${RESP_ID}"'",
    "input": "我刚才让你记住的代号是什么？"
  }'
```

关注点：

- 第二轮请求应成功接受 `previous_response_id`
- 多轮上下文应生效

### 19. 标准函数工具定义

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": "Please read /project/README.md",
    "tools": [
      {
        "type": "function",
        "name": "read_file",
        "description": "Read a file",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"}
          },
          "required": ["file_path"]
        },
        "strict": true
      }
    ]
  }'
```

关注点：

- 若模型决定调工具，应返回 `output[].type=function_call`
- `arguments` 必须是 JSON 字符串

### 20. 标准 `function_call` + `function_call_output` 回填

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": [
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "List files in the current directory."}
        ]
      },
      {
        "type": "function_call",
        "id": "fc_list_dir_001",
        "call_id": "call_list_dir_001",
        "name": "list_dir",
        "arguments": "{\"path\":\".\"}",
        "status": "completed"
      },
      {
        "type": "function_call_output",
        "call_id": "call_list_dir_001",
        "output": "[\"README.md\",\"main.py\"]"
      }
    ]
  }'
```

关注点：

- 服务端应接受标准 `function_call_output`
- 最终回答应基于工具结果生成

### 21. `tool_choice` 指定函数

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": "Read /project/src/index.ts",
    "tool_choice": {
      "type": "function",
      "name": "read_file"
    },
    "tools": [
      {
        "type": "function",
        "name": "read_file",
        "description": "Read a file",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"}
          },
          "required": ["file_path"]
        }
      }
    ]
  }'
```

### 22. `text.format=json_object`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": "Return a JSON object with keys framework and language for FastAPI.",
    "text": {
      "format": {
        "type": "json_object"
      }
    }
  }'
```

### 23. `text.format=json_schema`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": "Describe FastAPI in JSON only.",
    "text": {
      "format": {
        "type": "json_schema",
        "name": "framework_card",
        "schema": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "language": {"type": "string"}
          },
          "required": ["name", "language"],
          "additionalProperties": false
        },
        "strict": true
      }
    }
  }'
```

### 24. 流式 Responses 事件

```bash
curl -N "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "stream": true,
    "input": "Please read /project/README.md",
    "tools": [
      {
        "type": "function",
        "name": "read_file",
        "description": "Read a file",
        "parameters": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"}
          },
          "required": ["file_path"]
        }
      }
    ]
  }'
```

关注点：

- 应看到 `response.created`
- 若触发工具调用，应看到 `response.output_item.added`
- 工具参数流式输出应使用 `response.function_call_arguments.delta`
- 最终应看到 `response.completed`

### 25. 标准 `input_image`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": [
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "Describe this image."},
          {"type": "input_image", "image_url": "https://example.com/demo.jpg"}
        ]
      }
    ]
  }'
```

### 26. 多轮对话 + 多工具混合 `input`

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "input": [
      {
        "role": "developer",
        "content": [
          {"type": "input_text", "text": "你是一个严谨的出行助手。"}
        ]
      },
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "请查询北京天气和当前北京时间。"}
        ]
      },
      {
        "type": "function_call",
        "id": "fc_weather_bj_001",
        "call_id": "call_weather_bj_001",
        "name": "get_weather",
        "arguments": "{\"location\":\"北京\"}",
        "status": "completed"
      },
      {
        "type": "function_call_output",
        "call_id": "call_weather_bj_001",
        "output": "{\"location\":\"北京\",\"weather\":\"晴\",\"temperature_c\":26,\"rain\":false}"
      },
      {
        "type": "function_call",
        "id": "fc_time_bj_001",
        "call_id": "call_time_bj_001",
        "name": "get_time",
        "arguments": "{\"timezone\":\"Asia/Shanghai\"}",
        "status": "completed"
      },
      {
        "type": "function_call_output",
        "call_id": "call_time_bj_001",
        "output": "{\"timezone\":\"Asia/Shanghai\",\"local_time\":\"2026-04-10 09:30:00\"}"
      },
      {
        "role": "assistant",
        "content": [
          {
            "type": "output_text",
            "text": "北京当前晴，26 摄氏度，现在时间是 2026-04-10 09:30:00。",
            "annotations": []
          }
        ]
      },
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "基于前面的天气和时间，再给我一句出行建议，并说明是否需要带伞。"}
        ]
      }
    ],
    "tools": [
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
        }
      },
      {
        "type": "function",
        "name": "get_time",
        "description": "Get local time",
        "parameters": {
          "type": "object",
          "properties": {
            "timezone": {"type": "string"}
          },
          "required": ["timezone"]
        }
      }
    ]
  }'
```

关注点：

- 标准 `input` 数组中可同时存在多个不同工具的 `function_call` / `function_call_output`
- 后续 `user` 追问应能消费前面的多工具结果和 assistant 历史输出
- 若模型再次选择工具，返回应继续保持标准 `output[].type=function_call`

### 27. `previous_response_id` + 多工具结果跨轮回填

第一轮：请求模型同时调用两个工具。

```bash
FIRST_RESP=$(
  curl -sS "${BASE_URL}/v1/responses" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${API_KEY}" \
    -d '{
      "model": "'"${MODEL}"'",
      "input": "请查询北京天气和当前北京时间。",
      "tools": [
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
          }
        },
        {
          "type": "function",
          "name": "get_time",
          "description": "Get local time",
          "parameters": {
            "type": "object",
            "properties": {
              "timezone": {"type": "string"}
            },
            "required": ["timezone"]
          }
        }
      ]
    }'
)

echo "${FIRST_RESP}" | jq .
```

从第一轮响应中提取：

- `RESP_ID=.id`
- 天气工具的 `call_id`
- 时间工具的 `call_id`

第二轮：把两个工具结果一起回填，并继续追问。

```bash
RESP_ID="替换为第一轮返回的 response id"
CALL_ID_WEATHER="替换为第一轮 get_weather 的 call_id"
CALL_ID_TIME="替换为第一轮 get_time 的 call_id"

curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "previous_response_id": "'"${RESP_ID}"'",
    "input": [
      {
        "type": "function_call_output",
        "call_id": "'"${CALL_ID_WEATHER}"'",
        "output": "{\"location\":\"北京\",\"weather\":\"晴\",\"temperature_c\":26,\"rain\":false}"
      },
      {
        "type": "function_call_output",
        "call_id": "'"${CALL_ID_TIME}"'",
        "output": "{\"timezone\":\"Asia/Shanghai\",\"local_time\":\"2026-04-10 09:30:00\"}"
      },
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "基于这两个工具结果，给我一句出行建议，并说明是否需要带伞。"}
        ]
      }
    ]
  }'
```

关注点：

- 第二轮应成功接受 `previous_response_id`
- 同一轮可同时回填多个 `function_call_output`
- 模型应能结合前一轮工具调用和本轮多个工具结果继续回答

---

## 三、公共接口与基础校验

### 28. `/v1/models`

```bash
curl -sS "${BASE_URL}/v1/models" \
  -H "Authorization: Bearer ${API_KEY}"
```

关注点：

- 返回对象为 `list`
- `data[0].id` 应为公网模型名，例如 `glm-5.1`

### 29. Chat Completions 缺少必填字段时的错误对象

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'"
  }'
```

关注点：

- 应返回 JSON 错误对象
- 不应返回 HTML 错页

### 30. Responses 缺少必填字段时的错误对象

```bash
curl -sS "${BASE_URL}/v1/responses" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'"
  }'
```

关注点：

- 应返回 JSON 错误对象
- 不应返回 HTML 错页

---

## 四、兼容性补充用例（非标准 OpenAI 协议，可选）

本节不是标准 OpenAI 协议验收项，只用于验证代理层兼容/降级能力。

### 31. 连续同角色消息

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {"role": "user", "content": "Message 1"},
      {"role": "user", "content": "Message 2"},
      {"role": "assistant", "content": "Response"},
      {"role": "user", "content": "Final question"}
    ]
  }'
```

### 32. Chat Completions 兼容 `input_image` 块

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "Describe this image."},
          {"type": "input_image", "image_url": {"url": "https://example.com/demo.jpg"}}
        ]
      }
    ]
  }'
```

关注点：

- 该请求不是标准 Chat Completions 内容块
- 若服务端做兼容转换，请求不应 500

### 33. Anthropic 风格 `image` 内容块兼容

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "请确认请求未崩溃并简短回复。"},
          {
            "type": "image",
            "source": {
              "type": "url",
              "media_type": "image/jpeg",
              "data": "https://example.com/demo.jpg"
            }
          }
        ]
      }
    ]
  }'
```

### 34. 未知图片块兜底

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "请确认请求未崩溃并简短回复。"},
          {"type": "weird_image", "path": "/tmp/demo.jpg"}
        ]
      }
    ]
  }'
```

### 35. 大字段工具参数容错

```bash
curl -sS "${BASE_URL}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${API_KEY}" \
  -d '{
    "model": "'"${MODEL}"'",
    "tool_choice": "required",
    "messages": [
      {
        "role": "user",
        "content": "Use the write_file tool to write /tmp/demo.py with the exact content def main():\n    print(\"hello\")\n```md\nliteral fence\n```. Do not answer with plain text first."
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "write_file",
          "description": "Write file contents",
          "parameters": {
            "type": "object",
            "properties": {
              "file_path": {"type": "string"},
              "content": {"type": "string"}
            },
            "required": ["file_path", "content"]
          }
        }
      }
    ]
  }'
```

关注点：

- 若模型返回 `tool_calls`，`function.arguments` 中应保留多行文本、引号与代码围栏
- 请求不应因大字段工具参数而 500
