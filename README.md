# api-protocol-converter
OpenAI (Chat Completions / Responses) 与 Anthropic (Messages) API 协议转换专家。
覆盖三端点间的请求归一化、响应编码、流式事件互转、工具调用协议映射、以及错误处理。
当用户需要：
(1) 实现 OpenAI 格式到 Claude 格式的协议代理/转换层；
(2) 理解 Chat Completions、Responses、Messages 三端点的字段差异与转换规则；
(3) 编写协议转换代理或网关；
(4) 测试协议转换的正确性；
(5) 排查协议转换中的常见错误时使用。

技能 api-protocol-converter 已创建完成
目录结构如下：
  api-protocol-converter/
  ├── SKILL.md（213 行，核心指导）
  └── references/（7 个参考文件，按需加载）
      ├── chat-completions-spec.md      — /v1/chat/completions 协议完整规范
      ├── messages-spec.md              — /v1/messages 协议完整规范
      ├── responses-spec.md             — /v1/responses 协议完整规范
      ├── three-protocol-guide.md       — 三端点互转编码指南（IR设计、归一化、流式映射）
      ├── chat-vs-messages-diff.md      — Chat vs Messages 核心差异与转换关键点
      ├── openai-test-cases.md          — OpenAI 格式可执行 curl 测试用例（35 条）
      └── claude-test-cases.md          — Claude 格式可执行 curl 测试用例（20 条）

SKILL.md 核心内容：
  
1. 三端点快速对照表 — 10 维度对照 Chat Completions / Responses / Messages
2. 五大关键转换点 — system提取、工具定义映射、工具调用与回填（含硬规则）、tool_choice映射、流式事件互转
3. curl 快速对照示例 — Chat请求转Messages请求、Claude响应转Chat响应的完整对照
4. 实现路线建议 — 7步渐进式路线
5. 最小测试矩阵 — 12 条必须通过的验收标准
6. 不在第一版范围的内容 — 明确声明不支持的能力

对 references 文件做的修改：
- 移除了对 replit.md / replit2.md 的项目特定引用，改为同目录文档互引
- 环境变量通用化：将硬编码的密钥、地址、模型名替换为 ${API_KEY} / ${BASE_URL} / ${MODEL}
- 删除了对 main.py / scripts/ 的项目特定自动化引用
