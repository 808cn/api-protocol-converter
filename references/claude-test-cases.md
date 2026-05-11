# Claude 格式 API 测试用例

本文档可直接执行的 `curl` 用例，覆盖：

- 基础问答
- 流式响应
- 工具调用与 `tool_result`
- Claude Code / Agentic 场景
- 边界接口（`/health`、`/v1/messages/count_tokens`）
- 长对话 / 截断相关场景

## 环境变量

使用前请替换为你自己的目标地址、密钥和模型名：

```bash
export BASE_URL="https://api.anthropic.com"    # 替换为你的代理地址或 Anthropic 官方地址
export API_KEY="sk-ant-your-api-key"            # 替换为你的 Anthropic API 密钥
export MODEL="claude-sonnet-4-5"                # 替换为你要测试的模型
```

## Claude Code 配置示例

```bash
export ANTHROPIC_BASE_URL="https://api.anthropic.com"  # 替换为你的地址
export ANTHROPIC_AUTH_TOKEN="sk-ant-your-api-key"      # 替换为你的密钥
export ANTHROPIC_MODEL="claude-sonnet-4-5"
```

以下 `curl` 用例可直接用于验证任何兼容 `/v1/messages` 协议的端点。

## 通用请求头

所有 Claude 格式请求需要以下三个请求头：

```bash
-H "x-api-key: ${API_KEY}" \
-H "anthropic-version: 2023-06-01" \
-H "content-type: application/json"
```

## 一、基础问答

### 1. 简单中文问答

对应：`简单中文问答`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "用一句话解释什么是递归。"}
    ]
  }'
```

关注点：`type=message`、`stop_reason=end_turn`、`content[0].type=text`。

### 2. 英文问答

对应：`英文问答`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "What is the difference between async/await and Promises in JavaScript? Be concise."}
    ]
  }'
```

### 3. 多轮对话

对应：`多轮对话`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "My name is TestBot. Remember it."},
      {"role": "assistant", "content": "Got it! I will remember your name is TestBot."},
      {"role": "user", "content": "What is my name?"}
    ]
  }'
```

关注点：回复中应包含 `TestBot`。

### 4. 代码生成

对应：`代码生成`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "messages": [
      {"role": "user", "content": "Write a JavaScript function that reverses a string. Return only the code, no explanation."}
    ]
  }'
```

## 二、流式问答

### 5. 流式简单问答

对应：`流式简单问答`

```bash
curl -N "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "stream": true,
    "messages": [
      {"role": "user", "content": "请列出5种常见的排序算法并简单说明时间复杂度。"}
    ]
  }'
```

关注点：SSE 中应出现 `message_start`、`content_block_delta`、`message_delta`、`message_stop`，且 `stop_reason=end_turn`。

### 6. 流式长输出

对应：`流式长输出（测试空闲超时修复）`

```bash
curl -N "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "stream": true,
    "messages": [
      {"role": "user", "content": "请用中文详细介绍快速排序算法：包括原理、实现思路、时间复杂度分析、最优/最差情况、以及完整的 TypeScript 代码实现。内容要详细，至少500字。"}
    ]
  }'
```

关注点：持续有数据返回，不应中途卡死；最终 `stop_reason=end_turn`。

## 三、工具调用（非流式）

### 7. 单工具调用：Read

对应：`单工具调用 — Read file`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Read",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string", "description": "Absolute path of the file to read."}
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

关注点：返回 `content[].type=tool_use`，`name=Read`，参数使用 `file_path` 而不是 `path`。

### 8. 单工具调用：Bash

对应：`单工具调用 — Bash command`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Bash",
        "description": "Execute a bash command in the terminal.",
        "input_schema": {
          "type": "object",
          "properties": {
            "command": {"type": "string", "description": "The command to execute."}
          },
          "required": ["command"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Run \"ls -la\" to list the current directory."}
    ]
  }'
```

### 8A. 单工具调用：Glob

对应：`单工具调用 — Glob`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Glob",
        "description": "Find files by glob pattern.",
        "input_schema": {
          "type": "object",
          "properties": {
            "pattern": {"type": "string"},
            "path": {"type": "string"}
          },
          "required": ["pattern"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Find all TypeScript files under /project/src using Glob."}
    ]
  }'
```

关注点：返回 `tool_use.name=Glob`，参数里包含 `pattern`，可选 `path` 不应丢失。

### 8B. 单工具调用：Grep

对应：`单工具调用 — Grep`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Grep",
        "description": "Search file contents by regex.",
        "input_schema": {
          "type": "object",
          "properties": {
            "pattern": {"type": "string"},
            "path": {"type": "string"},
            "glob": {"type": "string"}
          },
          "required": ["pattern"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Use Grep to search for function main under /project/src/*.ts."}
    ]
  }'
```

关注点：返回 `tool_use.name=Grep`，参数里 `pattern/path/glob` 结构不应损坏。

### 8C. 单工具调用：Edit

对应：`单工具调用 — Edit`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Edit",
        "description": "Edit a file by exact string replacement.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"},
            "old_string": {"type": "string"},
            "new_string": {"type": "string"},
            "replace_all": {"type": "boolean"}
          },
          "required": ["file_path", "old_string", "new_string"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Use Edit to replace foo with bar in /project/src/app.ts."}
    ]
  }'
```

关注点：返回 `tool_use.name=Edit`，参数里 `old_string/new_string/replace_all` 不应丢失或串位。

### 9. 校验 `stop_reason=tool_use`

对应：`工具调用 — stop_reason = tool_use`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
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
      {"role": "user", "content": "Read the file /src/main.ts"}
    ]
  }'
```

### 10. `tool_result` 多轮回填

对应：`工具调用后追加 tool_result 的多轮对话`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
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
      {"role": "user", "content": "Read the config file at /app/config.json"},
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
          "content": "{\"port\":3010,\"model\":\"claude-sonnet-4.6\",\"timeout\":120}"
        }
      ]}
    ]
  }'
```

关注点：最终应返回文本回答；不应说“无法访问文件”或“没有工具能力”。

### 11. 多工具并行调用

对应：`多工具并行调用（Read + Bash）`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "tools": [
      {
        "name": "Read",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {"file_path": {"type": "string"}},
          "required": ["file_path"]
        }
      },
      {
        "name": "Bash",
        "description": "Execute a bash command in the terminal.",
        "input_schema": {
          "type": "object",
          "properties": {"command": {"type": "string"}},
          "required": ["command"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "I need to check the directory listing and read the package.json file. Please do both."}
    ]
  }'
```

关注点：应至少产生一个 `tool_use`；常见是 `Read`、`Bash` 两者之一或两个都出现。

## 四、工具调用（流式）

### 12. 流式工具调用：Read

对应 ：`流式工具调用 — Read`

```bash
curl -N "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 2048,
    "stream": true,
    "tools": [
      {
        "name": "Read",
        "description": "Read the contents of a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {"file_path": {"type": "string"}},
          "required": ["file_path"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Please read /project/README.md"}
    ]
  }'
```

关注点：

- 应看到 `content_block_start`，其中 `content_block.type=tool_use`
- 工具参数通过 `input_json_delta` 分块返回
- 最终 `stop_reason=tool_use`

### 13. 流式工具调用：Write 长内容

对应：`流式工具调用 — Write file（测试长 content 截断修复）`

```bash
curl -N "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "stream": true,
    "tools": [
      {
        "name": "Write",
        "description": "Write content to a file at the given path.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"},
            "content": {"type": "string"}
          },
          "required": ["file_path", "content"]
        }
      }
    ],
    "messages": [
      {
        "role": "user",
        "content": "Write a new file at /tmp/hello.ts with content: a TypeScript class called HelloWorld with a greet() method that returns \"Hello, World!\". Include full class definition with constructor and method."
      }
    ]
  }'
```

关注点：`Write` 工具参数里的 `content` 不应因为代码块或长字符串而损坏。

## 五、Agentic / Claude Code 场景

以下场景用于验证 IDE/Agent 模式的工具链路。

### 14. 项目结构探索 + 总结

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "tools": [
      {
        "name": "list_files",
        "description": "List files and directories in a given path.",
        "input_schema": {
          "type": "object",
          "properties": {"path": {"type": "string"}},
          "required": ["path"]
        }
      },
      {
        "name": "Read",
        "description": "Reads a file from the local filesystem.",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string"},
            "start_line": {"type": "integer"},
            "end_line": {"type": "integer"}
          },
          "required": ["file_path"]
        }
      },
      {
        "name": "attempt_completion",
        "description": "Signal task completion with a final summary.",
        "input_schema": {
          "type": "object",
          "properties": {
            "result": {"type": "string"}
          },
          "required": ["result"]
        }
      }
    ],
    "messages": [
      {
        "role": "user",
        "content": "Explore the project structure, read the key entry files, then summarize the architecture."
      }
    ]
  }'
```

关注点：应优先产出 `list_files` / `Read` / `attempt_completion` 等结构化工具调用，而不是直接纯文本结束。

### 15. `tool_choice=any` 强制工具调用

对应多个 `toolChoice: { type: 'any' }` 场景。

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "tool_choice": {"type": "any"},
    "tools": [
      {
        "name": "Read",
        "description": "Reads a file from the local filesystem.",
        "input_schema": {
          "type": "object",
          "properties": {"file_path": {"type": "string"}},
          "required": ["file_path"]
        }
      },
      {
        "name": "attempt_completion",
        "description": "Signal task completion with a final summary.",
        "input_schema": {
          "type": "object",
          "properties": {"result": {"type": "string"}},
          "required": ["result"]
        }
      }
    ],
    "messages": [
      {
        "role": "user",
        "content": "Use the Read tool to read /project/package.json. Then call attempt_completion with a summary of project name, version, and dependencies."
      }
    ]
  }'
```

关注点：响应必须包含至少一个 `tool_use`，不应只返回纯文本。

### 16. Bash 构建检查

对应：`跑构建并检查输出`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "tools": [
      {
        "name": "Bash",
        "description": "Executes a given bash command in a persistent shell session.",
        "input_schema": {
          "type": "object",
          "properties": {
            "command": {"type": "string"},
            "timeout": {"type": "integer"}
          },
          "required": ["command"]
        }
      }
    ],
    "messages": [
      {
        "role": "user",
        "content": "Use the Bash tool to run these commands one at a time: 1. cd /project && npm run build 2. cd /project && npm test. Report what each command outputs."
      }
    ]
  }'
```

## 六、边界与辅助接口

### 17. 身份问题（不泄露 Cursor）

对应：`身份问题`

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Who are you?"}
    ]
  }'
```

关注点：回复应表现为 Claude / Anthropic 身份，不应暴露 Cursor 支持助手身份。

### 18. 健康检查

对应：`/health 接口`

```bash
curl -s "${BASE_URL}/health"
```

关注点：返回 `ok=true`，并包含当前配置路径、可用模型与上游地址。

### 19. Token 计数

对应：`/v1/messages/count_tokens 接口`

```bash
curl -s "${BASE_URL}/v1/messages/count_tokens" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [
      {"role": "user", "content": "Hello world"}
    ]
  }'
```

### 20. 长对话 + 工具结果 + 合法 `stop_reason`

对应`长对话工具请求`。

```bash
curl -s "${BASE_URL}/v1/messages" \
  -H "x-api-key: ${API_KEY}" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 4096,
    "stream": false,
    "system": "You are a helpful coding assistant.",
    "tools": [
      {
        "name": "Read",
        "description": "Read a file from disk",
        "input_schema": {
          "type": "object",
          "properties": {
            "file_path": {"type": "string", "description": "Path to the file"}
          },
          "required": ["file_path"]
        }
      },
      {
        "name": "Bash",
        "description": "Execute a shell command",
        "input_schema": {
          "type": "object",
          "properties": {
            "command": {"type": "string", "description": "The command to execute"}
          },
          "required": ["command"]
        }
      }
    ],
    "messages": [
      {"role": "user", "content": "Please investigate why module18 is missing and continue the debugging workflow."},
      {"role": "assistant", "content": [
        {"type": "text", "text": "Let me check module1 first."},
        {"type": "tool_use", "id": "tool_1", "name": "Read", "input": {"file_path": "src/module1.ts"}}
      ]},
      {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_1", "content": "export class Module1 {}"}
      ]},
      {"role": "assistant", "content": [
        {"type": "text", "text": "Continuing."},
        {"type": "tool_use", "id": "tool_2", "name": "Read", "input": {"file_path": "src/module18.ts"}}
      ]},
      {"role": "user", "content": [
        {"type": "tool_result", "tool_use_id": "tool_2", "content": "File not found: src/module18.ts"}
      ]}
    ]
  }'
```

关注点：

- 返回 `type=message`
- `stop_reason` 应为 `end_turn`、`tool_use` 或 `max_tokens` 之一
- 长对话下不应因为连续消息或 `tool_result` 结构导致报错
