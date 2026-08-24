---
title: nanobot 从 0 到 1 到 100 构建 Roadmap
published: 2026-08-24
description: 以 AI 应用开发工程师的视角，规划从零实现一个类 nanobot 个人 AI Agent 框架的三阶段路线：最小可用、工程框架、完整平台。
tags: [nanobot, AI Agent, LLM, Roadmap, 工程实践]
category: AI 学习
draft: false
slug: nanobot-roadmap-0-to-100
---

# nanobot 从 0 到 1 到 100 构建 Roadmap

> 角色视角：AI 应用开发工程师。  
> 目标：不是简单复刻源码目录，而是理解如何一步步设计、实现并扩展一个像 nanobot 这样的个人 AI Agent 框架。  
> 主线：消息进入 -> 构建上下文 -> 调用模型 -> 执行工具 -> 保存状态 -> 返回用户 -> 长期运行与扩展生态。

## 0. Roadmap 总览

nanobot 可以拆成三段建设路径：

| 阶段 | 目标 | 产物形态 |
|---|---|---|
| 0 -> 1 | 做出最小可用 AI Agent | CLI 输入一句话，调用 LLM，返回回答 |
| 1 -> 10 | 做出可用工程框架 | 支持会话、工具调用、配置、Provider 抽象、基础安全 |
| 10 -> 100 | 做出完整 AI Agent 平台 | 多渠道、WebUI、MCP、记忆、自动化、长任务、插件化、部署和可观测 |

对应 nanobot 的核心模块：

```text
用户入口
  -> channels / cli / webui / api
消息解耦
  -> bus
Agent 编排
  -> agent/loop.py
上下文构建
  -> agent/context.py, templates, skills, memory
模型执行
  -> agent/runner.py, providers
工具能力
  -> agent/tools
状态持久化
  -> session, memory, cron store
长期服务
  -> gateway, websocket, channel manager
安全边界
  -> security, workspace policy, sandbox, SSRF guard
```

## 1. 构建心智模型

在动手构建前，先建立三个核心判断。

### 1.1 AI Agent 不是一次模型调用

普通 LLM 应用：

```text
输入 prompt -> 调用模型 -> 输出 answer
```

Agent 应用：

```text
用户消息
  -> 上下文组装
  -> 模型思考
  -> 模型请求工具
  -> 程序执行工具
  -> 工具结果回填给模型
  -> 模型继续推理
  -> 最终回复
  -> 历史和记忆持久化
```

为什么这样设计：

- 模型本身不能真正访问文件、运行命令、联网、定时执行任务。
- Agent 框架要把外部能力封装成工具交给模型调用。
- 工具调用需要状态管理、安全校验和错误恢复。

后续扩展点：

- 增加新的工具，如数据库查询、知识库检索、邮件发送。
- 增加新的模型 Provider。
- 增加新的入口渠道，如企业微信、飞书、Slack、网页。

Practice：

1. 用伪代码写出一个最小 Agent loop。
2. 手写一组 messages，包含 `system/user/assistant/tool` 四类 role。
3. 解释为什么 tool result 必须再次送回模型，而不是直接展示给用户。

### 1.2 nanobot 的核心边界

nanobot 把复杂系统拆成几个边界：

| 边界 | 解决的问题 | 典型文件 |
|---|---|---|
| Channel | 外部平台如何接入 | `nanobot/channels/` |
| Bus | 渠道和 Agent 如何解耦 | `nanobot/bus/` |
| AgentLoop | 一次用户 turn 如何编排 | `nanobot/agent/loop.py` |
| ContextBuilder | 模型输入如何构建 | `nanobot/agent/context.py` |
| AgentRunner | 模型和工具循环如何执行 | `nanobot/agent/runner.py` |
| Provider | 不同模型 API 如何适配 | `nanobot/providers/` |
| Tool | 模型能调用哪些能力 | `nanobot/agent/tools/` |
| Session/Memory | 对话和长期信息如何保存 | `nanobot/session/`, `nanobot/agent/memory.py` |
| Security | 文件、Shell、Web 如何限制风险 | `nanobot/security/` |

为什么这样设计：

- 每个模块只处理一种变化来源。
- 新增渠道不应该改 AgentRunner。
- 新增 Provider 不应该改 WebUI。
- 新增工具不应该污染 AgentLoop 主流程。

后续扩展点：

- Channel 插件化。
- Tool 插件化。
- Provider registry。
- Skills 作为非代码能力扩展。

Practice：

1. 画出 Channel -> Bus -> Loop -> Runner -> Provider/Tool 的调用图。
2. 判断下面需求该改哪个模块：新增飞书卡片样式、新增数据库工具、新增 Gemini Provider、新增上下文压缩策略。

## 2. Phase 0：项目脚手架和最小 LLM 调用

目标：从 0 开始建立一个可以运行的最小 AI 应用。

### 2.1 建立 Python 项目结构

建议目录：

```text
mini_nanobot/
  app.py
  config.py
  providers.py
  runner.py
  pyproject.toml
  tests/
```

最小功能：

- 读取配置。
- 创建 Provider。
- 发送 messages。
- 打印模型回复。

为什么这样设计：

- 初期不要急着做 WebUI、渠道、工具和记忆。
- 先验证“模型调用链路”是否跑通。
- 所有后续模块都依赖这个最小闭环。

后续扩展点：

- 支持多个 Provider。
- 支持流式输出。
- 支持 retry、timeout、usage 统计。

Practice：

1. 写一个 `ask(message: str) -> str` 函数。
2. 把 system prompt 和 user prompt 组装成 messages。
3. 模拟一个 FakeProvider，不调用真实 API，只返回固定答案，用它写单元测试。

### 2.2 定义 Provider 接口

最小接口：

```python
class LLMProvider:
    async def chat(self, messages: list[dict], tools: list[dict] | None = None) -> LLMResponse:
        ...
```

对应 nanobot：

- `nanobot/providers/base.py`
- `LLMProvider`
- `LLMResponse`
- `ToolCallRequest`

为什么这样设计：

- Agent 核心不应该依赖某个具体厂商 SDK。
- Provider 是 Strategy Pattern，同一套 AgentRunner 可以使用不同模型。
- 工具调用、usage、reasoning、错误信息需要统一数据结构。

后续扩展点：

- OpenAI-compatible Provider。
- Anthropic Provider。
- Azure OpenAI Provider。
- Bedrock Provider。
- FallbackProvider。

Practice：

1. 定义一个 `LLMResponse` dataclass。
2. 写 `FakeProvider` 返回普通文本。
3. 写 `FakeToolCallProvider` 返回一个 tool call。
4. 思考：为什么 Provider 不应该直接执行工具？

## 3. Phase 1：从一次调用到 AgentRunner

目标：实现模型和工具之间的循环。

### 3.1 实现 AgentRunner

最小循环：

```text
messages = initial_messages
for iteration in range(max_iterations):
    response = provider.chat(messages, tools)
    if response has tool_calls:
        append assistant tool_call message
        execute tools
        append tool result messages
        continue
    else:
        append final assistant message
        return final answer
```

对应 nanobot：

- `nanobot/agent/runner.py`
- `AgentRunSpec`
- `AgentRunResult`
- `AgentRunner.run()`
- `AgentRunner._run_core()`

为什么这样设计：

- AgentRunner 只关心模型和工具，不关心消息来自 CLI 还是 WebUI。
- `max_iterations` 防止模型无限循环调用工具。
- 每轮都把工具结果追加回 messages，让模型基于结果继续决策。

后续扩展点：

- 流式输出 hook。
- 工具并发执行。
- 工具结果压缩。
- 空回复重试。
- 长上下文治理。
- 持续目标续跑。

Practice：

1. 写一个只支持一个工具的 Runner。
2. 增加 `max_iterations`，并测试模型一直请求工具时会停止。
3. 让 FakeProvider 第一轮返回 tool call，第二轮返回 final answer。
4. 打印每轮 messages，观察 assistant/tool 消息如何追加。

### 3.2 设计 Tool 接口

最小接口：

```python
class Tool:
    name: str
    description: str
    parameters: dict

    async def execute(self, **kwargs):
        ...
```

对应 nanobot：

- `nanobot/agent/tools/base.py`
- `nanobot/agent/tools/schema.py`
- `nanobot/agent/tools/registry.py`

为什么这样设计：

- `name` 是模型调用函数时使用的稳定标识。
- `description` 决定模型什么时候使用工具。
- `parameters` 是模型生成参数的 JSON Schema。
- `execute` 是真实世界副作用发生的地方。

后续扩展点：

- 只读工具和写入工具分级。
- 工具权限控制。
- 工具参数自动类型转换。
- 工具运行上下文。
- MCP 外部工具接入。

Practice：

1. 实现 `time_now` 工具，返回当前时间。
2. 实现 `calculator` 工具，只允许加减乘除。
3. 给工具参数写 JSON Schema。
4. 输入错误参数时返回结构化错误，不要抛出未处理异常。

### 3.3 实现 ToolRegistry

职责：

- 注册工具。
- 输出 tool definitions 给 Provider。
- 根据模型返回的 tool name 找工具。
- 解析和校验参数。
- 统一包装工具错误。

对应 nanobot：

- `nanobot/agent/tools/registry.py`
- `ToolRegistry.register()`
- `ToolRegistry.get_definitions()`
- `ToolRegistry.prepare_call()`
- `ToolRegistry.execute()`

为什么这样设计：

- 模型输出不可完全信任，必须先校验再执行。
- 错误信息要能反馈给模型，让模型换一种方式尝试。
- 工具定义要稳定排序，便于 prompt cache 和可复现行为。

后续扩展点：

- 内置工具和 MCP 工具分组排序。
- 工具名纠错提示。
- 工具并发安全标记。
- 工具执行审计日志。

Practice：

1. 模拟模型调用不存在的工具，确认返回清晰错误。
2. 模拟参数不是 JSON object，确认拒绝执行。
3. 实现工具名建议，例如 `readfile` 提示 `read_file`。

## 4. Phase 2：从 Runner 到 AgentLoop

目标：把单次 Runner 包装成面向用户 turn 的完整流程。

### 4.1 引入 MessageBus

最小设计：

```python
class MessageBus:
    inbound: asyncio.Queue[InboundMessage]
    outbound: asyncio.Queue[OutboundMessage]
```

对应 nanobot：

- `nanobot/bus/events.py`
- `nanobot/bus/queue.py`

为什么这样设计：

- 渠道收消息和 Agent 处理消息速度不一定相同。
- 多个渠道可以共享同一个 Agent 核心。
- 后续支持 gateway 长期运行、后台任务和 WebUI 更自然。

后续扩展点：

- Runtime events。
- Stream delta events。
- Progress events。
- Outbound rich UI metadata。

Practice：

1. 实现一个 producer 往 inbound 放消息。
2. 实现一个 consumer 从 inbound 取消息并回复到 outbound。
3. 模拟两个 channel 同时发消息，观察队列行为。

### 4.2 定义 InboundMessage / OutboundMessage

核心字段：

```text
InboundMessage:
  channel
  sender_id
  chat_id
  content
  media
  metadata
  session_key

OutboundMessage:
  channel
  chat_id
  content
  media
  metadata
  event
```

为什么这样设计：

- Agent 核心不关心 Telegram、Slack、WebSocket 的原始协议。
- 所有渠道进入核心前都变成统一事件。
- `session_key` 让不同聊天拥有不同历史。

后续扩展点：

- 多模态 media。
- 回复引用 reply_to。
- buttons。
- channel-specific metadata。
- structured UI event。

Practice：

1. 用 `channel:chat_id` 生成 session key。
2. 模拟同一个用户在两个 chat 中发消息，验证 session 不串。
3. 在 metadata 中放一个 trace id，贯穿处理链路。

### 4.3 实现 AgentLoop

职责：

- 从 bus 消费 inbound。
- 处理命令和系统消息。
- 决定 session。
- 构建上下文。
- 调用 AgentRunner。
- 保存历史。
- 发布 outbound。

对应 nanobot：

- `nanobot/agent/loop.py`
- `TurnState`
- `AgentLoop.run()`
- `AgentLoop._process_message()`

为什么这样设计：

- AgentLoop 是产品层编排器，负责一次用户 turn 的生命周期。
- AgentRunner 是模型层执行器，保持更纯粹。
- 状态机让复杂流程可观测、可测试、可恢复。

nanobot 的 turn 状态：

```text
RESTORE -> COMPACT -> COMMAND -> BUILD -> RUN -> SAVE -> RESPOND -> DONE
```

后续扩展点：

- command router。
- auto compact。
- checkpoint restore。
- mid-turn user injection。
- hooks。
- progress streaming。
- sustained goal。

Practice：

1. 实现一个只有 BUILD -> RUN -> RESPOND 的简化 AgentLoop。
2. 给每个状态记录耗时。
3. 加一个 `/help` 命令，不进入模型调用。
4. 模拟 Runner 抛错，确认 loop 能返回错误消息并不中断服务。

## 5. Phase 3：上下文构建与 Prompt 系统

目标：把用户输入、系统身份、历史、记忆、技能组合成模型可用上下文。

### 5.1 构建 ContextBuilder

对应 nanobot：

- `nanobot/agent/context.py`
- `ContextBuilder.build_system_prompt()`
- `ContextBuilder.build_messages()`
- `nanobot/templates/agent/*.md`

组成部分：

```text
system prompt:
  identity
  platform policy
  tool contract
  bootstrap files
  memory
  active skills
  skills summary
  recent history summary

messages:
  system
  history
  current user message
  runtime context
```

为什么这样设计：

- AI 应用的行为很大程度由上下文决定。
- system prompt、skills、memory 都是运行时行为的一部分。
- 上下文要可组合，不能散落在业务代码里。

后续扩展点：

- Jinja2 prompt templates。
- Workspace-specific AGENTS.md / SOUL.md / USER.md。
- Skills loader。
- Runtime context provider。
- 多模态图片输入。

Practice：

1. 写一个 `build_system_prompt()`，拼接身份、工具规则和当前时间。
2. 加入最近 5 条历史消息。
3. 加入一个 `skills` 文本块，让模型学会某个领域规则。
4. 观察 system prompt 不同顺序对模型输出的影响。

### 5.2 管理上下文窗口

问题：模型上下文有限，不能无限塞历史。

对应 nanobot：

- `nanobot/session/manager.py`
- `nanobot/agent/context_governance.py`
- `nanobot/agent/autocompact.py`

为什么这样设计：

- 长对话会超过 context window。
- 工具结果可能非常长。
- 历史里可能包含模型不该模仿的内部标记。

后续扩展点：

- 按 token 截断。
- 历史压缩。
- 工具结果摘要。
- orphan tool result 修复。
- malformed tool call 修复。

Practice：

1. 实现一个按消息数量截断历史的函数。
2. 实现一个按字符数截断 tool result 的函数。
3. 构造一个以 tool 消息开头的非法历史，写函数修复它。

## 6. Phase 4：会话、记忆和持久化

目标：让 Agent 能跨 turn 保留上下文。

### 6.1 SessionManager

对应 nanobot：

- `nanobot/session/manager.py`
- `Session`
- `Session.get_history()`

职责：

- 创建和加载 session。
- 保存 user/assistant/tool messages。
- 生成可 replay 的历史。
- 清理内部噪声。
- 支持 session list 和 metadata。

为什么这样设计：

- 对话历史是 Agent 连续性的基础。
- 不同渠道/聊天需要隔离历史。
- 需要避免坏历史永久污染模型输入。

后续扩展点：

- JSONL 持久化。
- Session metadata。
- Fork session。
- Session delete。
- WebUI session index。

Practice：

1. 用 JSONL 保存每条消息。
2. 实现 `get_history(max_messages=20)`。
3. 删除 assistant 回复中的本地路径或工具调用 echo。
4. 写测试验证两个 session 不串历史。

### 6.2 Memory 和 Dream

对应 nanobot：

- `nanobot/agent/memory.py`
- `nanobot/templates/memory/MEMORY.md`
- `docs/memory.md`

Session 与 Memory 的区别：

| 类型 | 作用 | 生命周期 |
|---|---|---|
| Session | 近期对话 replay | 随聊天增长，可压缩 |
| Memory | 长期事实、偏好、项目状态 | 跨会话保留 |

Dream 是周期性整理机制，把历史中的长期信息写入 memory。

为什么这样设计：

- 全量历史无法永久放进上下文。
- 用户偏好、项目事实应该长期保留。
- 记忆更新必须谨慎，避免错误信息污染未来。

后续扩展点：

- 两阶段 memory consolidation。
- Memory diff。
- Git-backed memory auto commit。
- Workspace-level memory。
- Memory visibility policy。

Practice：

1. 设计一个 `MEMORY.md` 格式，分为用户偏好、项目事实、待办。
2. 从 10 条历史中人工提取长期记忆。
3. 写一个简单 consolidator，把历史摘要追加到 memory。
4. 思考：哪些信息不应该写入长期记忆？

## 7. Phase 5：配置系统和模型运行时

目标：让项目从硬编码变成可配置。

### 7.1 Pydantic Config Schema

对应 nanobot：

- `nanobot/config/schema.py`
- `nanobot/config/loader.py`
- `nanobot/config/paths.py`

配置范围：

- Provider API key / apiBase / proxy。
- Model presets。
- Fallback models。
- Agent defaults。
- Tools 开关。
- Channels 配置。
- Gateway / API / WebUI。
- Security 策略。

为什么这样设计：

- AI Agent 的运行参数很多，必须集中管理。
- Pydantic 可以做类型校验、默认值、alias。
- 明确 schema 比散落 dict 更利于维护和文档化。

后续扩展点：

- camelCase/snake_case 兼容。
- 环境变量插值。
- 配置迁移。
- 多实例配置。
- WebUI settings API。

Practice：

1. 用 Pydantic 写 `ProviderConfig`。
2. 支持 `apiKey` alias 到 `api_key`。
3. 加载 JSON 配置并创建 Provider。
4. 配置缺少 api key 时给出清晰错误。

### 7.2 ModelRuntimeResolver

对应 nanobot：

- `nanobot/agent/model_runtime.py`
- `nanobot/providers/factory.py`
- `nanobot/agent/model_presets.py`

为什么这样设计：

- 当前模型可能在运行时切换。
- Provider chain 可能受 preset、fallback、config reload 影响。
- AgentLoop 需要拿到一个稳定的 runtime snapshot 来处理当前 turn。

后续扩展点：

- Runtime model switching。
- WebUI 模型选择。
- Provider reload。
- MCP reload。
- 多模型路由。

Practice：

1. 定义两个 model preset：fast 和 smart。
2. 写函数 `resolve_runtime(preset_name)`。
3. 模拟运行中切换 preset，下一轮 turn 生效。

## 8. Phase 6：文件、Shell、Web 等高价值工具

目标：让 Agent 从聊天机器人变成能做事的助手。

### 8.1 Filesystem Tools

对应 nanobot：

- `nanobot/agent/tools/filesystem.py`
- `nanobot/agent/tools/path_utils.py`
- `nanobot/security/workspace_access.py`

能力：

- read file。
- write file。
- edit file。
- list dir。
- apply patch。

为什么这样设计：

- AI 编程和项目分析都需要读取/修改文件。
- 文件能力必须有 workspace 边界。
- 路径解析要统一，不能每个工具自己拼路径。

后续扩展点：

- extra read allowed dirs。
- extra write allowed dirs。
- exact file allowlist。
- file edit activity event。
- diff preview。

Practice：

1. 实现 `read_file(path)`，限制只能读 workspace 内文件。
2. 尝试读取 `../secret.txt`，确认被拒绝。
3. 实现 `list_dir`，限制最大返回条数。
4. 为路径解析写 5 个安全测试。

### 8.2 Shell Tool

对应 nanobot：

- `nanobot/agent/tools/shell.py`
- `nanobot/agent/tools/sandbox.py`
- `.agent/security.md`

为什么这样设计：

- Shell 很强，也很危险。
- workspace restriction 是应用层保护。
- sandbox backend 是更强隔离，但平台相关。

后续扩展点：

- allow patterns。
- deny patterns。
- session-based exec。
- long-running command。
- bubblewrap sandbox。
- Windows PowerShell 兼容。

Practice：

1. 实现只能在 workspace 运行的 `exec(command)`。
2. 禁止 `cd ..` 或明显逃逸路径。
3. 给命令设置 timeout。
4. 测试命令失败时如何把 stderr 返回给模型。

### 8.3 Web Search / Fetch

对应 nanobot：

- `nanobot/agent/tools/web.py`
- `nanobot/security/network.py`

为什么这样设计：

- Agent 经常需要实时信息。
- Web fetch 存在 SSRF 风险，不能允许模型访问内网地址。
- URL 重定向后也要继续校验目标。

后续扩展点：

- 搜索 Provider。
- Fetch HTML 清洗。
- URL allowlist。
- SSRF whitelist。
- 内容长度限制。

Practice：

1. 实现 `validate_url_target(url)`，拒绝 localhost 和私有网段。
2. 测试 `http://127.0.0.1`、`http://169.254.169.254` 被拒绝。
3. fetch 后截断内容，避免超长 tool result。

## 9. Phase 7：Channel 和 Gateway

目标：让 Agent 从 CLI 程序变成长运行服务。

### 9.1 BaseChannel

对应 nanobot：

- `nanobot/channels/base.py`

接口：

```text
start()
stop()
send(outbound)
send_delta(...)
_handle_message(...)
```

为什么这样设计：

- 不同平台协议差异很大，但 Agent 核心需要统一入口。
- Channel 负责平台协议、鉴权、消息格式转换。
- AgentLoop 不应该知道 Telegram 或 Slack 的细节。

后续扩展点：

- Streaming。
- Reasoning delta。
- File edit events。
- Pairing。
- Channel-specific rich UI。

Practice：

1. 实现一个 `ConsoleChannel`，从 stdin 读消息，从 stdout 输出回复。
2. 实现一个 `MemoryChannel`，用于测试，不连接真实平台。
3. 加 allowlist，只允许指定 sender_id。

### 9.2 ChannelManager

对应 nanobot：

- `nanobot/channels/manager.py`
- `nanobot/channels/registry.py`

职责：

- 根据配置发现 enabled channels。
- 初始化 channel。
- 启动/停止 channel。
- 从 outbound bus 路由回复。
- 统一处理发送重试。

为什么这样设计：

- Gateway 可能同时启用多个渠道。
- outbound 路由应该集中管理。
- Channel 插件发现不应该散落在业务逻辑中。

后续扩展点：

- pkgutil scan。
- entry point plugins。
- 多实例 channel。
- hot reload。
- delivery retry policy。

Practice：

1. 注册两个 fake channel。
2. 发出两条 outbound，确认路由到对应 channel。
3. 模拟 send 失败，做 1/2/4 秒重试。

### 9.3 Gateway

对应 nanobot：

- `nanobot/cli/commands.py`
- `nanobot/cli/gateway.py`
- `nanobot/gateway/service.py`
- `nanobot/cron/service.py`

Gateway 启动内容：

- AgentLoop。
- ChannelManager。
- WebSocket channel。
- Cron service。
- Dream / heartbeat system jobs。
- Health endpoint。

为什么这样设计：

- CLI one-shot 适合调试，不适合长期服务。
- Gateway 是部署形态。
- 长期任务、WebUI、聊天渠道都需要常驻进程。

后续扩展点：

- graceful shutdown。
- health check。
- restart command。
- background jobs。
- multi workspace。

Practice：

1. 用 asyncio 启动 AgentLoop 和 ChannelManager 两个 task。
2. 捕获 SIGINT，优雅停止。
3. 增加 `/health` HTTP endpoint。

## 10. Phase 8：WebUI 和 WebSocket 协议

目标：构建可视化工作台。

### 10.1 WebSocket Channel

对应 nanobot：

- `nanobot/channels/websocket.py`
- `nanobot/webui/ws_http.py`
- `nanobot/webui/gateway_services.py`
- `docs/websocket.md`

为什么这样设计：

- WebUI 需要实时流式输出。
- 一个浏览器连接可能管理多个 chat。
- WebSocket 比轮询更适合 delta、reasoning、progress、file edit events。

后续扩展点：

- Auth token。
- Reconnect。
- Multi chat multiplex。
- Media upload。
- Workspace scope control。
- Transcription WS。

Practice：

1. 设计一个最小 WS 协议：`send_message`、`delta`、`final`。
2. 让服务端每 100ms 返回一个 delta。
3. 前端按 chat_id 分发事件。

### 10.2 React WebUI

对应 nanobot：

- `webui/src/lib/nanobot-client.ts`
- `webui/src/components/thread/ThreadShell.tsx`
- `webui/src/components/thread/ThreadComposer.tsx`
- `webui/src/components/thread/ThreadMessages.tsx`
- `webui/src/components/settings/SettingsView.tsx`

核心组件：

```text
NanobotClient
  -> WebSocket 连接和事件分发
ThreadShell
  -> 当前会话外壳
ThreadComposer
  -> 输入、附件、发送
ThreadMessages
  -> 消息列表
AgentActivityCluster
  -> 工具/推理/文件编辑活动
SettingsView
  -> 设置、渠道、技能、模型
```

为什么这样设计：

- 前端需要处理连接状态、流式消息、会话切换、附件、设置。
- WebSocket client 是协议层，React 组件是 UI 层。
- 多 chat 复用一个 socket，减少连接管理复杂度。

后续扩展点：

- 文件预览。
- Diff 展示。
- Token usage heatmap。
- Channel setup wizard。
- Skill catalog。
- Voice recorder。

Practice：

1. 做一个最小聊天 UI：输入框、发送按钮、消息列表。
2. 支持流式 delta 更新最后一条 assistant 消息。
3. 断线后显示 reconnecting 状态。
4. 切换 chat 时保留每个 chat 的消息。

## 11. Phase 9：MCP、Skills 和插件化生态

目标：让框架可扩展，而不是所有能力都写进 core。

### 11.1 MCP Tools

对应 nanobot：

- `nanobot/agent/tools/mcp.py`
- `docs/guides/configure-mcp-tools.md`

为什么这样设计：

- MCP 让外部工具服务可以被 Agent 调用。
- 不需要把所有工具代码都放进 nanobot。
- 标准协议利于生态扩展。

后续扩展点：

- stdio MCP。
- HTTP/SSE MCP。
- MCP reconnect。
- MCP preset。
- MCP security validation。

Practice：

1. 接入一个简单 MCP server。
2. 把 MCP tool 转换为 ToolRegistry 里的工具定义。
3. 测试 MCP server 崩溃后重连。

### 11.2 Skills

对应 nanobot：

- `nanobot/skills/`
- `nanobot/agent/skills.py`
- `nanobot/templates/agent/skills_section.md`

为什么这样设计：

- 有些能力是“知识和流程”，不是代码工具。
- Skill 可以通过 Markdown 告诉 Agent 某类任务怎么做。
- 不需要为每种工作流都写 Python 逻辑。

后续扩展点：

- Workspace skills。
- Built-in skills。
- Skill catalog。
- Skill enable/disable。
- Skill packaging。

Practice：

1. 写一个 `code-review` skill，规定审查输出格式。
2. 把 skill 加入 system prompt。
3. 比较启用/禁用 skill 的输出差异。

### 11.3 插件化

插件化方向：

- Channel plugin。
- Tool plugin。
- Provider registry。
- Skill package。
- MCP preset。

为什么这样设计：

- 核心保持小而稳定。
- 扩展能力在边缘增加。
- 第三方可以独立发布能力。

Practice：

1. 设计一个 entry point 发现机制。
2. 写一个外部包注册新工具。
3. 验证主程序无需改源码即可加载工具。

## 12. Phase 10：自动化、Cron 和长任务

目标：让 Agent 能主动执行任务，而不仅是被动聊天。

### 12.1 Cron Service

对应 nanobot：

- `nanobot/cron/service.py`
- `nanobot/agent/tools/cron.py`
- `nanobot/cron/session_turns.py`
- `nanobot/templates/HEARTBEAT.md`

为什么这样设计：

- 用户希望 Agent 定时提醒、定期检查、持续跟进。
- Cron job 要绑定 session，结果发回原聊天上下文。
- Heartbeat 是系统级周期检查。

后续扩展点：

- 用户创建 reminders。
- protected system jobs。
- job persistence。
- missed run recovery。
- timezone support。

Practice：

1. 实现一个每分钟触发一次的 job。
2. job 触发时向 bus 发布 system inbound message。
3. 让结果回到创建 job 的 session。

### 12.2 Long-running Goal

对应 nanobot：

- `nanobot/agent/tools/long_task.py`
- `nanobot/session/goal_state.py`
- `nanobot/templates/agent/goal_runtime.md`

为什么这样设计：

- 有些任务不能一轮完成，例如代码改造、研究、持续监控。
- 需要保存 goal 状态，允许自动续跑或用户中途注入。
- 需要限制预算和停止条件。

后续扩展点：

- sustained goal state。
- wall timeout。
- continuation message。
- user interruption。
- goal completion/block 状态。

Practice：

1. 设计 goal state：objective、status、token_budget、turn_count。
2. 当模型还没完成时，自动注入 continuation message。
3. 连续失败三次后标记 blocked。

## 13. Phase 11：安全体系

目标：把 Agent 从 demo 变成可运行在真实环境的系统。

### 13.1 Workspace 安全边界

对应 nanobot：

- `nanobot/security/workspace_access.py`
- `nanobot/security/workspace_policy.py`
- `nanobot/agent/tools/path_utils.py`
- `.agent/security.md`

为什么这样设计：

- 文件工具和 Shell 工具非常危险。
- 用户通常只希望 Agent 操作指定 workspace。
- 路径逃逸必须统一处理。

后续扩展点：

- read allowed dirs。
- write allowed dirs。
- exact file allowlist。
- per-tool capability。
- WebUI workspace selector。

Practice：

1. 实现 `resolve_workspace_path(path)`。
2. 测试 symlink 是否能逃逸 workspace。
3. 给 read/write 分别设计权限。

### 13.2 SSRF 防护

对应 nanobot：

- `nanobot/security/network.py`
- `nanobot/agent/tools/web.py`

为什么这样设计：

- 模型可能被诱导 fetch 内网 URL。
- 云 metadata endpoint 可能泄露凭据。
- redirect 后目标也必须验证。

后续扩展点：

- CIDR whitelist。
- DNS rebinding 防护。
- redirect validation。
- per-tool network policy。

Practice：

1. 拒绝 loopback、private、link-local、metadata IP。
2. 写测试覆盖 IPv4、IPv6、域名解析。
3. 模拟 302 redirect 到内网地址，确认拒绝。

### 13.3 Prompt 和 Memory 污染防护

对应 nanobot：

- `nanobot/session/manager.py`
- `nanobot/agent/context_governance.py`
- `.agent/gotchas.md`

为什么这样设计：

- 模型会模仿历史中的模式。
- 内部路径、工具调用 echo、调试文本不应该进入长期上下文。
- 错误记忆会持续影响未来行为。

Practice：

1. 清理 assistant 历史中的本地路径。
2. 清理工具调用 echo。
3. 设计 memory 写入前的人工确认流程。

## 14. Phase 12：测试、可观测和工程质量

目标：让系统可维护、可演进。

### 14.1 测试分层

nanobot 的测试布局值得学习：

```text
tests/agent/
tests/tools/
tests/providers/
tests/channels/
tests/session/
tests/config/
tests/security/
tests/webui/
webui/src/tests/
```

为什么这样设计：

- Agent 系统很容易出现跨模块回归。
- Provider 和 Tool 可以用 fake/mock 测试，不必依赖真实 API。
- 安全边界必须有回归测试。

Practice：

1. 给 ToolRegistry 写参数校验测试。
2. 给 AgentRunner 写 fake tool call 测试。
3. 给 workspace path resolver 写攻击用例测试。
4. 给 WebSocket client 写重连测试。

### 14.2 可观测性

需要记录：

- turn id。
- session key。
- model/preset。
- 每个状态耗时。
- tool calls。
- provider usage。
- retry wait。
- error kind。

对应 nanobot：

- `StateTraceEntry`
- runtime events。
- outbound progress events。
- WebUI activity timeline。

为什么这样设计：

- Agent 错误常常不是单点异常，而是上下文、模型、工具、多轮循环组合问题。
- 没有 trace，很难知道卡在哪一阶段。

Practice：

1. 给每次 turn 生成 trace id。
2. 记录 BUILD/RUN/SAVE/RESPOND 耗时。
3. 打印每次 tool call 的 name、status、duration。

## 15. Phase 13：部署和发布

目标：让项目可以被别人安装、运行和维护。

### 15.1 Packaging

对应 nanobot：

- `pyproject.toml`
- `hatch_build.py`
- `nanobot/web/dist`
- `webui/vite.config.ts`

为什么这样设计：

- Python 包需要包含后端代码和构建后的 WebUI。
- 开发态 WebUI 和生产态 WebUI 构建路径不同。
- CLI entry point 让用户直接运行 `nanobot`。

后续扩展点：

- PyPI release。
- Docker image。
- install script。
- version check。

Practice：

1. 写一个最小 `pyproject.toml`，注册 CLI command。
2. 构建 wheel 并本地安装。
3. 把 WebUI dist 作为 package data 打进去。

### 15.2 Docker 和服务化

对应 nanobot：

- `Dockerfile`
- `docker-compose.yml`
- `entrypoint.sh`
- `docs/deployment.md`

为什么这样设计：

- Gateway 是长期服务，适合容器部署。
- 配置、workspace、日志需要 volume。
- 健康检查便于运维。

Practice：

1. 写 Dockerfile 启动 gateway。
2. 用 volume 挂载 config 和 workspace。
3. 增加 healthcheck。

## 16. 从 0 到 100 的里程碑清单

### Milestone 1：最小聊天程序

功能：

- CLI 输入。
- 单 Provider。
- 返回模型回复。

验收：

- `python app.py "hello"` 能输出 answer。
- 有 FakeProvider 测试。

### Milestone 2：AgentRunner + ToolRegistry

功能：

- 支持 tool call。
- 支持 max_iterations。
- 支持工具参数校验。

验收：

- FakeProvider 第一轮请求工具，第二轮输出最终答案。
- 不存在工具和错误参数都有测试。

### Milestone 3：MessageBus + AgentLoop

功能：

- Inbound/Outbound 队列。
- AgentLoop 消费消息。
- 简单 session key。

验收：

- 两个 chat_id 的历史不串。
- `/help` 命令不进入模型。

### Milestone 4：Context 和 Session

功能：

- system prompt。
- history replay。
- JSONL session。
- 历史截断。

验收：

- 第二轮对话能看到第一轮上下文。
- 超过 max_messages 后自动截断。

### Milestone 5：Provider 抽象和配置

功能：

- OpenAI-compatible Provider。
- model presets。
- config loader。
- fallback provider。

验收：

- 改配置可切换模型。
- 主模型失败时走 fallback。

### Milestone 6：文件/Shell/Web 工具和安全

功能：

- read/list/write/edit。
- shell exec。
- web fetch。
- workspace restriction。
- SSRF guard。

验收：

- 越界路径被拒绝。
- localhost/private fetch 被拒绝。
- Shell timeout 生效。

### Milestone 7：Channel 和 Gateway

功能：

- BaseChannel。
- ChannelManager。
- WebSocket channel。
- Gateway 常驻。

验收：

- Gateway 可同时启动 AgentLoop 和 channel。
- outbound 可正确路由。
- Ctrl+C 可优雅退出。

### Milestone 8：WebUI

功能：

- 聊天界面。
- 流式输出。
- 多 session。
- 设置页。
- 工具活动展示。

验收：

- 浏览器能发送消息并看到流式回复。
- 切换 chat 不丢消息。
- 工具调用过程可见。

### Milestone 9：Memory、Skills、MCP

功能：

- Memory store。
- Dream consolidation。
- Skills loader。
- MCP tools。

验收：

- 长期记忆能进入 system prompt。
- Skill 启用后改变模型行为。
- MCP tool 可被模型调用。

### Milestone 10：自动化、长任务、部署

功能：

- Cron reminders。
- Heartbeat。
- sustained goal。
- Docker deployment。
- Observability。

验收：

- 定时任务能触发 Agent turn。
- 长任务能续跑并记录状态。
- Docker 启动 gateway 可访问 health。

## 17. 关键设计原则总结

### 17.1 核心保持小，扩展放边缘

对应 nanobot 设计约束：

- `AgentLoop` 和 `AgentRunner` 是核心路径，不轻易塞业务功能。
- 新能力优先放到 channel、tool、skill、MCP、provider。

为什么：

- 核心越大，越难测试和维护。
- Agent 行为复杂，必须控制主链路复杂度。

### 17.2 明确边界胜过魔法

例子：

- Provider registry 明确声明模型后端。
- Tool schema 明确声明参数。
- Config schema 明确声明配置字段。
- Security policy 明确声明可访问范围。

为什么：

- AI 应用本身已有不确定性，工程边界必须确定。
- 可解释、可调试、可测试比“自动猜测”更重要。

### 17.3 Prompt、Skill、Memory 都是代码的一部分

为什么：

- 它们直接影响模型行为。
- 改 prompt 可能导致功能回归。
- Memory 污染会长期影响 Agent。

实践建议：

- Prompt 改动要小。
- Skill 要有明确适用场景。
- Memory 写入要谨慎。
- 对关键 prompt 行为写测试或 golden case。

### 17.4 安全是 Agent 的功能，不是附加项

为什么：

- Tool 让模型可以影响真实世界。
- 文件、Shell、Web 都可能造成数据泄露或破坏。
- 渠道接入后还要考虑谁能使用 Agent。

实践建议：

- 默认限制 workspace。
- 默认拒绝内网 fetch。
- 默认要求 channel allowlist 或 pairing。
- 高危工具必须有测试。

## 18. 个人学习与实践路线

如果你要以 AI 应用开发工程师身份掌握 nanobot，推荐顺序如下：

### 第 1 阶段：AI Agent 基础

学习内容：

- messages 格式。
- function calling。
- tool result 回填。
- system prompt。

练习：

- 写最小 Runner。
- 写 2 个工具。
- 用 FakeProvider 测试工具调用。

### 第 2 阶段：工程主链路

学习内容：

- MessageBus。
- AgentLoop。
- ContextBuilder。
- SessionManager。

练习：

- 实现 CLI channel。
- 实现 JSONL session。
- 实现 `/help` 命令。

### 第 3 阶段：真实能力

学习内容：

- 文件工具。
- Shell 工具。
- Web fetch。
- 安全边界。

练习：

- 做 workspace-restricted file tools。
- 做 SSRF validation。
- 写安全回归测试。

### 第 4 阶段：平台化

学习内容：

- ChannelManager。
- Gateway。
- WebSocket。
- React WebUI。

练习：

- 做一个最小浏览器聊天界面。
- 支持流式回复。
- 支持多 chat。

### 第 5 阶段：高级 Agent 能力

学习内容：

- Memory。
- Skills。
- MCP。
- Cron。
- Long-running goals。

练习：

- 写一个 skill。
- 接一个 MCP 工具。
- 实现一个定时提醒。
- 设计一个长任务状态机。

## 19. 建议阅读 nanobot 源码顺序

```text
1. docs/concepts.md
2. docs/architecture.md
3. nanobot/bus/events.py
4. nanobot/bus/queue.py
5. nanobot/providers/base.py
6. nanobot/agent/tools/base.py
7. nanobot/agent/tools/registry.py
8. nanobot/agent/runner.py
9. nanobot/agent/context.py
10. nanobot/session/manager.py
11. nanobot/agent/loop.py
12. nanobot/providers/factory.py
13. nanobot/config/schema.py
14. nanobot/channels/base.py
15. nanobot/channels/manager.py
16. nanobot/channels/websocket.py
17. webui/src/lib/nanobot-client.ts
18. nanobot/security/workspace_access.py
19. nanobot/security/network.py
20. nanobot/agent/memory.py
```

读源码时始终问三个问题：

1. 这个模块接收什么输入，输出什么？
2. 它为什么不放在别的模块里？
3. 如果我要扩展它，稳定接口在哪里？

## 20. 最终能力检查表

完成这份 roadmap 后，你应该能做到：

- 解释 nanobot 的整体架构。
- 独立实现一个最小 AI Agent runner。
- 理解 tool calling 的消息格式。
- 新增一个安全的只读工具。
- 新增或理解一个 Provider。
- 理解 session 和 memory 的区别。
- 理解 WebUI 如何通过 WebSocket 接入 Agent。
- 判断一个需求应该放在 channel、tool、provider、skill 还是 Agent core。
- 为工具、Provider、AgentRunner 写测试。
- 识别文件、Shell、Web 工具的安全风险。

## 21. 最后建议

构建 nanobot 这类项目，不建议一开始追求“大而全”。最稳的路线是：

```text
最小模型调用
  -> AgentRunner
  -> ToolRegistry
  -> AgentLoop
  -> Session
  -> Config
  -> 安全工具
  -> Channel/Gateway
  -> WebUI
  -> Memory/Skills/MCP/Cron
  -> 部署和可观测
```

这样做的好处是每一步都有可运行产物，每个模块都有清晰边界。AI 应用的复杂度会自然增长，工程设计的任务就是让复杂度长在正确的位置。
