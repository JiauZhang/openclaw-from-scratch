# 附录：核心概念速查表

## 1. Agent（Agent 核心抽象）

Agent 是 OpenClaw 的核心抽象，它封装了 LLM 对话的完整生命周期。`Agent` 类（`packages/agent-core/src/agent.ts`）管理消息转录、工具执行、事件分发和消息队列。它通过依赖注入接收 `streamFn`，实现与 LLM 提供商的解耦。Agent 提供 `prompt()`、`continue()`、`steer()` 和 `followUp()` 等 API，支持同步请求和异步引导。

## 2. Agent Loop（Agent 主循环）

Agent Loop 是 `while(true)` 模式的核心实现（`packages/agent-core/src/agent-loop.ts`）。每次迭代包含：向 LLM 发送请求 → 解析响应 → 执行工具调用 → 将结果送回 LLM。当 LLM 不再请求工具调用时循环终止。循环支持 `beforeToolCall` 和 `afterToolCall` 钩子，以及 `toolExecution` 模式（`"sequential"` 或 `"parallel"`）。

## 3. Tool Descriptor（工具描述符）

Tool Descriptor 是工具的定义规范，包含 `name`、`description` 和 `inputSchema`（JSON Schema 格式）。LLM 根据描述符的 schema 生成结构化的工具调用参数。工具描述符是 OpenClaw 工具系统的核心契约，也是 LLM 与外部系统交互的桥梁。

## 4. Tool Plan（工具规划）

Tool Plan 是工具可见性控制机制，将工具分为**可见工具**（visible）和**隐藏工具**（hidden）。可见工具会出现在 LLM 的 tool 定义中，可以被直接调用；隐藏工具仅用于特定场景的后台执行。这种设计允许 Agent 控制 LLM 的"工具视野"，减少混乱并提高安全性。

## 5. StreamFn（流式函数）

StreamFn 是 LLM 流式调用的核心接口，接收消息列表、工具定义和模型配置，返回 `AssistantMessageEvent` 流。它封装了不同 LLM 提供商的 API 差异，向 Agent 循环提供统一的流式调用体验。OpenClaw 的完整实现支持流式文本、流式工具调用、思考过程流式输出等。

## 6. Event Stream（事件流）

Event Stream 是 Agent 运行时的可观察事件模式。Agent 在运行过程中会发出 `turn_start`、`llm_delta`、`tool_start`、`tool_end`、`turn_end`、`agent_end` 等事件。外部组件可以通过 `subscribe()` 方法监听这些事件，实现实时日志、UI 更新、转录归档等功能。

## 7. Gateway（网关）

Gateway 是 OpenClaw 的 WebSocket 控制平面，负责消息路由、会话管理和 Agent 分发。它接收来自不同通道（Slack、Discord、飞书等）的消息，解析路由规则，将消息传递给对应的 Agent 实例，并将 Agent 的回复分发回对应的通道。Gateway 是 OpenClaw 作为"多通道 AI 网关"的核心体现。

## 8. Plugin SDK（插件 SDK）

Plugin SDK 是 OpenClaw 的扩展层，允许第三方开发者注册工具、提供者、生命周期钩子和通道集成。插件通过 `registerSandboxBackend`、`registerTool`、`registerProvider` 等 API 与核心系统集成。完整的插件生命周期包括加载、初始化、运行和卸载阶段。

## 9. Channel（通道）

Channel 是 IM 平台集成层，负责将 OpenClaw 与外部消息平台连接。每个通道实现（如 Slack、Discord、飞书、Telegram）处理平台特定的消息格式、认证、事件订阅和消息发送。Channel 通过 Gateway 与 Agent 运行时通信，实现跨平台的统一消息处理。

## 10. Sandbox（沙箱）

Sandbox 是安全执行环境，为 Agent 的代码执行提供隔离。它支持 Docker 容器和 SSH 远程服务器两种后端。Sandbox 通过工具权限策略（Tool Policy）控制 Agent 在沙箱内可以调用的工具，通过文件系统桥接（FS Bridge）管理工作区访问权限，是 Agent 安全体系的核心组件。