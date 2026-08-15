# 第四章：Agent 核心循环

本章深入讲解 OpenClaw 最核心的部分——**Agent 核心循环**（agent loop）。这是 Agent 的大脑，负责协调 LLM 调用、工具执行、消息队列和事件分发。

## 4.1 整体流程概览

```mermaid
graph TB
    A["用户输入 prompt"] --> B["agentLoop() / runAgentLoop()"]
    B --> C["发出 agent_start 事件"]
    C --> D["发出 turn_start 事件"]
    D --> E["发出 prompt 消息事件"]
    E --> F["runLoop(): 主循环"]

    subgraph "主循环 runLoop()"
        direction TB
        G{"检查 steering 消息"} -->|有| H["注入消息到上下文"]
        G -->|无| I["streamAssistantResponse()<br/>调用 LLM"]
        H --> I
        I --> J{"stopReason?"}
        J -->|error/aborted| K["发出 agent_end 事件"]
        J -->|toolUse| L["executeToolCalls()<br/>执行工具"]
        J -->|end_turn/stop| M["检查 follow-up 消息"]
        L --> N{"hasMoreToolCalls?"}
        N -->|是| G
        N -->|否| M
        M -->|有后续消息| G
        M -->|无| O["发出 agent_end 事件"]
    end

    K --> P["返回新消息列表"]
    O --> P
```

## 4.2 入口函数

`agent-loop.ts` 提供了四个主要入口函数，分为**同步流式**和**异步回调**两组：

### 4.2.1 流式入口（返回 EventStream）

```typescript
// packages/agent-core/src/agent-loop.ts

/** 从新的 prompt 开始，返回事件流 */
export function agentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): EventStream<AgentEvent, AgentMessage[]>;

/** 从现有上下文继续，返回事件流 */
export function agentLoopContinue(
  context: AgentContext,
  config: AgentLoopConfig,
  signal?: AbortSignal,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): EventStream<AgentEvent, AgentMessage[]>;
```

### 4.2.2 异步回调入口（接收 EventSink）

```typescript
/** 从新的 prompt 开始，通过 emit 回调发布事件 */
export async function runAgentLoop(
  prompts: AgentMessage[],
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,   // (event: AgentEvent) => Promise<void> | void
  signal?: AbortSignal,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): Promise<AgentMessage[]>;

/** 从现有上下文继续，通过 emit 回调发布事件 */
export async function runAgentLoopContinue(
  context: AgentContext,
  config: AgentLoopConfig,
  emit: AgentEventSink,
  signal?: AbortSignal,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): Promise<AgentMessage[]>;
```

### 两组入口的关系

```mermaid
graph LR
    subgraph "流式入口（UI 使用）"
        AL["agentLoop()"]
        ALC["agentLoopContinue()"]
    end

    subgraph "内部实现"
        RAL["runAgentLoop()"]
        RALC["runAgentLoopContinue()"]
    end

    subgraph "核心循环"
        RL["runLoop()"]
    end

    AL -->|创建 EventStream| RAL
    ALC -->|创建 EventStream| RALC
    RAL --> RL
    RALC --> RL
```

流式入口创建 `EventStream` 对象，将事件推入流中，适合 UI 层通过 `for await...of` 消费。异步回调入口则直接调用 `emit` 回调，适合 `Agent` 类内部使用（它通过 `subscribe` 机制管理监听器）。

## 4.3 runLoop() 主循环

这是整个 Agent 的核心所在。`runLoop()` 函数是一个 `while(true)` 循环，持续执行直到没有更多工作要做：

```typescript
async function runLoop(
  initialContext: AgentContext,
  newMessages: AgentMessage[],
  initialConfig: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): Promise<void> {
  let currentContext = initialContext;
  let config = initialConfig;
  let firstTurn = true;
  let turnOpen = true;
  let pendingMessages: AgentMessage[] = (await config.getSteeringMessages?.()) || [];

  // 外层循环：处理 follow-up 消息
  while (true) {
    let hasMoreToolCalls = true;

    // 内层循环：处理工具调用和 steering 消息
    while (hasMoreToolCalls || pendingMessages.length > 0) {
      // 检查中止信号
      if (await stopIfAborted()) return;

      // 发出 turn_start 事件
      if (!firstTurn) {
        await emit({ type: "turn_start" });
        turnOpen = true;
      } else {
        firstTurn = false;
      }

      // 处理 pending 消息（注入到上下文）
      if (pendingMessages.length > 0) {
        for (const message of pendingMessages) {
          await emit({ type: "message_start", message });
          await emit({ type: "message_end", message });
          currentContext.messages.push(message);
          newMessages.push(message);
        }
      }

      // 流式获取 assistant 响应
      const message = await streamAssistantResponse(
        currentContext, config, signal, emit, streamFn, runtime,
      );
      newMessages.push(message);

      // 处理错误/中止
      if (message.stopReason === "error" || message.stopReason === "aborted") {
        await emit({ type: "turn_end", message, toolResults: [] });
        await emit({ type: "agent_end", messages: newMessages });
        return;
      }

      // 提取工具调用
      const toolCalls = message.content.filter((c) => c.type === "toolCall");
      const toolResults: ToolResultMessage[] = [];
      hasMoreToolCalls = false;

      if (message.stopReason === "toolUse" && toolCalls.length > 0) {
        const executedToolBatch = await executeToolCalls(
          currentContext, message, config, signal, emit,
        );
        toolResults.push(...executedToolBatch.messages);
        hasMoreToolCalls = !executedToolBatch.terminate;

        // 将工具结果追加到上下文
        for (const result of toolResults) {
          currentContext.messages.push(result);
          newMessages.push(result);
        }
      }

      // 结束本轮
      await emit({ type: "turn_end", message, toolResults });
      turnOpen = false;

      // prepareNextTurn 钩子
      const nextTurnContext = { message, toolResults, context: currentContext, newMessages };
      const nextTurnSnapshot = await config.prepareNextTurn?.(nextTurnContext);
      if (nextTurnSnapshot) {
        currentContext = nextTurnSnapshot.context ?? currentContext;
        config = updateConfig(config, nextTurnSnapshot);
      }

      // shouldStopAfterTurn 检查
      if (await config.shouldStopAfterTurn?.(nextTurnContext)) {
        await emit({ type: "agent_end", messages: newMessages });
        return;
      }

      // 获取新的 steering 消息
      pendingMessages = (await config.getSteeringMessages?.()) || [];
    }

    // 检查 follow-up 消息
    const followUpMessages = (await config.getFollowUpMessages?.()) || [];
    if (followUpMessages.length > 0) {
      pendingMessages = followUpMessages;
      continue;
    }

    // 没有更多消息，退出
    break;
  }

  await emit({ type: "agent_end", messages: newMessages });
}
```

### 循环流程详解

```mermaid
sequenceDiagram
    participant U as 用户
    participant RL as runLoop()
    participant LLM as LLM 提供商
    participant T as 工具系统

    RL->>RL: 检查 steering 消息
    RL->>LLM: streamAssistantResponse()
    Note over LLM: 流式返回消息块
    LLM-->>RL: text_start, text_delta, ...

    alt stopReason === "toolUse"
        RL->>T: executeToolCalls()
        T-->>RL: 工具结果
        RL->>RL: 追加到上下文
        RL->>RL: 检查 hasMoreToolCalls
    else stopReason === "end_turn" or "stop"
        RL->>RL: 检查 follow-up 消息
    else stopReason === "error" or "aborted"
        RL->>RL: 直接结束
    end

    RL->>RL: prepareNextTurn 钩子
    RL->>RL: shouldStopAfterTurn 检查
    RL->>RL: 获取 steering 消息（下一轮）
```

## 4.4 streamAssistantResponse() — LLM 调用

这是将 Agent 上下文转换为 LLM 请求并处理流式响应的关键函数：

```typescript
async function streamAssistantResponse(
  context: AgentContext,
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  emit: AgentEventSink,
  streamFn?: StreamFn,
  runtime?: AgentCoreStreamRuntimeDeps,
): Promise<AssistantMessage> {
  // 1. 上下文变换（AgentMessage[] → AgentMessage[]）
  let messages = context.messages;
  if (config.transformContext) {
    messages = await config.transformContext(messages, signal);
  }

  // 2. 转换为 LLM 兼容格式（AgentMessage[] → Message[]）
  const llmMessages = await config.convertToLlm(messages);

  // 3. 构建 LLM 上下文
  const llmContext: Context = {
    systemPrompt: context.systemPrompt,
    messages: llmMessages,
    tools: context.tools,
  };

  // 4. 解析流式函数
  const streamFunction = resolveAgentCoreStreamFn(runtime, streamFn);

  // 5. 解析 API key（动态获取，对短生命期令牌重要）
  const resolvedApiKey =
    (config.getApiKey ? await config.getApiKey(config.model.provider) : undefined) || config.apiKey;

  // 6. 调用 LLM
  const response = await streamFunction(config.model, llmContext, {
    ...config,
    apiKey: resolvedApiKey,
    signal,
  });

  // 7. 处理流式事件
  let partialMessage: AssistantMessage | null = null;
  let addedPartial = false;

  for await (const event of response) {
    switch (event.type) {
      case "start": {
        const message = event.partial;
        partialMessage = message;
        context.messages.push(message);
        addedPartial = true;
        await emit({ type: "message_start", message: { ...message } });
        break;
      }
      case "text_delta":
      case "thinking_delta":
      // ... 其他流式事件
        if (partialMessage) {
          const message = resolveAssistantMessageUpdate(event, partialMessage);
          partialMessage = message;
          context.messages[context.messages.length - 1] = message;
          await emit({ type: "message_update", assistantMessageEvent: event, message: { ...message } });
        }
        break;
      case "done":
      case "error": {
        const finalMessage = removeNonExecutableToolCalls(await response.result());
        // ... 更新上下文，发出 message_end 事件
        return finalMessage;
      }
    }
  }
}
```

### 数据流转换

```mermaid
graph LR
    subgraph "Agent 内部"
        AM["AgentMessage[]<br/>包含自定义类型"]
    end

    subgraph "transformContext"
        TC["可选：裁剪、注入"]
    end

    subgraph "convertToLlm"
        CTL["过滤自定义类型<br/>保留 user/assistant/toolResult"]
    end

    subgraph "LLM 提供商"
        LM["Message[]<br/>标准 LLM 消息"]
    end

    AM --> TC --> CTL --> LM
```

## 4.5 事件流系统

### 4.5.1 EventStream 创建

```typescript
function createAgentStream(): EventStream<AgentEvent, AgentMessage[]> {
  return new EventStreamConstructor<AgentEvent, AgentMessage[]>(
    (event: AgentEvent) => event.type === "agent_end",               // isDone 判断
    (event: AgentEvent) => (event.type === "agent_end" ? event.messages : []), // getResult
  );
}
```

`EventStream` 是一个可观察的流，它：
- 继承自 `AsyncIterable`，可以通过 `for await...of` 消费
- 在 `agent_end` 事件时标记为完成
- 结束时返回 `AgentMessage[]`（本轮新增的消息）

### 4.5.2 完整事件序列

```mermaid
sequenceDiagram
    participant UI as UI 层
    participant Agent as Agent 实例
    participant Loop as agent-loop

    UI->>Agent: prompt("Hello")
    Agent->>Loop: runAgentLoop()
    Loop-->>UI: agent_start
    Loop-->>UI: turn_start
    Loop-->>UI: message_start (user message)
    Loop-->>UI: message_end (user message)
    Loop-->>UI: message_start (assistant, partial)
    Loop-->>UI: message_update (text_delta)
    Loop-->>UI: message_update (text_delta)
    Loop-->>UI: message_update (toolcall_start)
    Loop-->>UI: message_end (assistant, final)
    Loop-->>UI: tool_execution_start (tool-1)
    Loop-->>UI: tool_execution_start (tool-2)
    Loop-->>UI: tool_execution_update (tool-1 progress)
    Loop-->>UI: tool_execution_end (tool-1)
    Loop-->>UI: tool_execution_end (tool-2)
    Loop-->>UI: message_start (toolResult-1)
    Loop-->>UI: message_end (toolResult-1)
    Loop-->>UI: message_start (toolResult-2)
    Loop-->>UI: message_end (toolResult-2)
    Loop-->>UI: turn_end
    Loop-->>UI: agent_end [messages]
```

### 4.5.3 事件类型汇总

| 事件类型 | 含义 | 触发时机 |
|---------|------|---------|
| `agent_start` | Agent 运行开始 | 循环开始时 |
| `turn_start` | 新轮次开始 | 每次 LLM 调用前 |
| `message_start` | 消息开始 | 消息加入上下文时 |
| `message_update` | 流式消息更新 | assistant 流式响应中 |
| `message_end` | 消息结束 | 消息完成时 |
| `tool_execution_start` | 工具开始执行 | 工具准备就绪时 |
| `tool_execution_update` | 工具执行进度 | 工具执行中 |
| `tool_execution_end` | 工具执行完成 | 工具执行后 |
| `turn_end` | 轮次结束 | 工具结果处理后 |
| `agent_end` | Agent 运行结束 | 所有工作完成时 |

## 4.6 AgentLoopConfig 回调详解

### 4.6.1 convertToLlm

将 `AgentMessage[]` 转换为 LLM 兼容的 `Message[]`。每个 `AgentMessage` 必须被转换为 `UserMessage`、`AssistantMessage` 或 `ToolResultMessage`。无法转换的消息（如 UI 通知）应被过滤掉。

```typescript
// 默认实现
function defaultConvertToLlm(messages: AgentMessage[]): Message[] {
  return messages.filter(
    (message) =>
      message.role === "user" || message.role === "assistant" || message.role === "toolResult",
  );
}
```

### 4.6.2 transformContext

在 `convertToLlm` 之前对 `AgentMessage[]` 进行变换，适用于：
- **上下文窗口管理**：裁剪过长的历史记录
- **外部上下文注入**：从外部源注入额外信息

```typescript
// 示例：上下文裁剪
transformContext: async (messages) => {
  if (estimateTokens(messages) > MAX_TOKENS) {
    return pruneOldMessages(messages);
  }
  return messages;
}
```

### 4.6.3 getApiKey

动态解析 API key，对短生命期 OAuth 令牌（如 GitHub Copilot）特别有用——这些令牌可能在长时间的工具执行阶段过期，需要在每次 LLM 调用时重新获取。

### 4.6.4 beforeToolCall

工具执行前的钩子，可以**阻断**工具调用：

```typescript
export interface BeforeToolCallResult {
  block?: boolean;   // true 表示阻断执行
  reason?: string;   // 阻断原因
}
```

### 4.6.5 afterToolCall

工具执行后的钩子，可以**修改**工具结果：

```typescript
export interface AfterToolCallResult {
  content?: (TextContent | ImageContent)[];  // 替换内容
  details?: unknown;                         // 替换详情
  isError?: boolean;                         // 替换错误标志
  terminate?: boolean;                       // 提示提前终止
}
```

`terminate` 的特殊语义：当**同一批**所有工具结果都设置了 `terminate: true` 时，Agent 循环会在本轮结束后停止，不再继续 LLM 调用。

### 4.6.6 prepareNextTurn

轮次之间的准备钩子，可以更新模型、推理级别或上下文：

```typescript
export interface AgentLoopTurnUpdate {
  context?: AgentContext;  // 替换上下文
  model?: Model;           // 替换模型
  thinkingLevel?: ThinkingLevel; // 替换思考级别
}
```

### 4.6.7 getSteeringMessages / getFollowUpMessages

```mermaid
graph TD
    subgraph "一轮结束"
        TE["turn_end"]
    end

    TE --> SS["shouldStopAfterTurn?"]
    SS -->|返回 true| AE["agent_end"]
    SS -->|返回 false| GSM["getSteeringMessages()"]

    GSM -->|有消息| P["pendingMessages = 消息"]
    GSM -->|无消息| GFM["getFollowUpMessages()"]

    P -->|下一轮循环| NL["下一轮 LLM 调用"]
    GFM -->|有消息| P
    GFM -->|无消息| AE
```

## 4.7 工具执行

### 4.7.1 executionMode 策略

```typescript
export type ToolExecutionMode = "sequential" | "parallel";
```

#### 顺序执行（sequential）

```mermaid
sequenceDiagram
    participant L as loop
    participant T1 as Tool-1
    participant T2 as Tool-2

    L->>T1: tool_execution_start
    Note over T1: 准备 → 验证 → beforeToolCall → 执行
    T1-->>L: tool_execution_end
    L->>L: tool_execution_end (emit)
    L->>L: message_start / message_end (toolResult)

    L->>T2: tool_execution_start
    Note over T2: 准备 → 验证 → beforeToolCall → 执行
    T2-->>L: tool_execution_end
    L->>L: tool_execution_end (emit)
    L->>L: message_start / message_end (toolResult)
```

#### 并行执行（parallel）

```mermaid
sequenceDiagram
    participant L as loop
    participant T1 as Tool-1
    participant T2 as Tool-2

    L->>T1: tool_execution_start (emit)
    L->>T2: tool_execution_start (emit)
    Note over L: 准备阶段（顺序）
    L->>T1: 准备（resolveTool → prepare → validate → beforeToolCall）
    L->>T2: 准备（resolveTool → prepare → validate → beforeToolCall）

    Note over T1,T2: 执行阶段（并发）
    T1-->>L: tool_execution_end (按完成顺序)
    T2-->>L: tool_execution_end (按完成顺序)

    Note over L: 结果消息（按原始顺序）
    L->>L: message_start / message_end (toolResult-1)
    L->>L: message_start / message_end (toolResult-2)
```

### 4.7.2 工具参数验证

使用 TypeBox schema 进行参数验证：

```typescript
// packages/agent-core/src/validation.ts
export { validateToolArguments, validateToolCall } from "@openclaw/ai/validation";
```

验证流程：

```typescript
async function prepareToolCall(
  currentContext: AgentContext,
  assistantMessage: AssistantMessage,
  toolCall: AgentToolCall,
  config: AgentLoopConfig,
  signal: AbortSignal | undefined,
  resolvedToolCalls: Map<AgentToolCall, ResolvedToolCallOutcome>,
): Promise<PreparedToolCall | ImmediateToolCallOutcome> {
  // 1. 解析工具（查找或延迟解析）
  const resolution = await resolveToolCallTool(...);

  // 2. prepareArguments（参数兼容性 shim）
  let preparedToolCall = prepareToolCallArguments(tool, toolCall);

  // 3. validateToolArguments（TypeBox schema 验证）
  let validatedArgs = validateToolArguments(tool, preparedToolCall);

  // 4. beforeToolCall 钩子（可阻断）
  const beforeResult = await config.beforeToolCall?.({ ... });

  // 5. 如果任何步骤失败，返回 ImmediateToolCallOutcome
  // 6. 全部通过，返回 PreparedToolCall
}
```

### 4.7.3 错误处理

```mermaid
graph TD
    TC["工具调用开始"] --> RT["resolveToolCallTool()<br/>查找工具定义"]

    RT -->|未找到| ERR1["Immediate: Tool not found"]
    RT -->|延迟解析| DR["resolveDeferredTool()"]
    DR -->|仍返回 undefined| ERR1
    DR -->|返回工具| PA["prepareArguments()"]

    PA -->|失败| ERR2["Immediate: prepare error"]
    PA -->|成功| VA["validateToolArguments()<br/>TypeBox 验证"]

    VA -->|失败| ERR3["Immediate: 参数验证错误<br/>errorKind: argument-validation"]
    VA -->|成功| BC["beforeToolCall() 钩子"]

    BC -->|block: true| ERR4["Immediate: 被阻断"]
    BC -->|通过| EX["执行工具 execute()"]

    EX -->|成功| OK["返回工具结果"]
    EX -->|抛出异常| ERR5["捕获异常，返回错误结果"]

    subgraph "Immediate 结果"
        ERR1
        ERR2
        ERR3
        ERR4
    end
```

## 4.8 Agent 命令编排

在 OpenClaw 的应用层，`src/agents/agent-command.ts` 负责将用户输入编排为一次完整的 Agent 运行。它是 `agent-core` 与 OpenClaw 会话管理、模型选择、权限控制之间的桥梁。

### 4.8.1 执行流程

```mermaid
graph TB
    subgraph "agentCommand() 入口"
        AC["agentCommand()"]
        ACI["agentCommandFromIngress()"]
    end

    subgraph "准备阶段 prepareAgentCommandExecution()"
        PA["解析会话 session"]
        PM["解析模型 model"]
        PT["解析思考级别 thinking"]
        PC["解析配置 config"]
    end

    subgraph "执行阶段"
        MF["runWithModelFallback()<br/>模型回退"]
        AR["runAgentAttempt()<br/>运行 Agent"]
    end

    subgraph "收尾阶段"
        D["deliverAgentCommandResult()<br/>投递结果"]
        PS["persistCliTurnTranscript()<br/>持久化记录"]
        CC["runCliTurnCompactionLifecycle()<br/>上下文压缩"]
    end

    AC --> PA
    ACI --> PA
    PA --> PM
    PM --> PT
    PT --> PC
    PC --> MF
    MF --> AR
    AR --> D
    AR --> PS
    AR --> CC
```

### 4.8.2 模型选择与回退

```mermaid
graph TD
    subgraph "模型选择优先级"
        ER["显式运行覆盖<br/>--provider / --model"]
        CM["通道模型覆盖<br/>modelByChannel"]
        SM["存储的模型覆盖<br/>session.modelOverride"]
        DM["默认模型<br/>agent.defaults.model"]
    end

    subgraph "模型回退"
        F1["主模型尝试"]
        F2["回退候选 1"]
        F3["回退候选 2"]
        FN["...更多回退"]
    end

    ER -->|最高优先级| MS["选定模型"]
    CM --> MS
    SM --> MS
    DM -->|最低优先级| MS

    MS --> F1
    F1 -->|失败| F2
    F2 -->|失败| F3
    F3 -->|失败| FN
```

### 8.3 会话生命周期管理

`agent-command.ts` 管理会话的完整生命周期：

1. **会话锁定**：通过 `beginSessionWorkAdmission()` 确保同一会话不会同时运行多个 Agent。
2. **状态持久化**：每次运行结束后更新 `SessionEntry`，包括模型、token 用量、思考级别等。
3. **上下文压缩**：通过 `runCliTurnCompactionLifecycle()` 在必要时压缩对话历史。
4. **投递结果**：通过 `deliverAgentCommandResult()` 将结果投递到目标通道（如 Slack、Discord 等）。

## 4.9 总结

Agent 核心循环是整个 OpenClaw 系统的中枢神经系统。它通过精心的分层设计，将 LLM 调用、工具执行、消息队列和事件分发有机地结合起来。

| 组件 | 职责 |
|------|------|
| `agentLoop()` / `agentLoopContinue()` | 流式入口，返回 `EventStream` |
| `runAgentLoop()` / `runAgentLoopContinue()` | 异步回调入口，接收 `EventSink` |
| `runLoop()` | 主循环，协调所有步骤 |
| `streamAssistantResponse()` | LLM 调用与流式响应处理 |
| `executeToolCalls()` | 工具执行调度（顺序/并行） |
| `EventStream` | 事件流，连接 Agent 与 UI |

下一章将介绍 **工具系统**——Agent 如何通过工具描述符、规划器和协议转换与外部世界交互。

> 延伸阅读：如果你想先动手写代码，也可以直接跳转到 [第 11 章](ch11-minimal-agent/README.md) 从零搭建一个完整的最小 Agent 实现。