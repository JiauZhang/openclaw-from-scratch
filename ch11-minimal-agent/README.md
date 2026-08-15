# 第十一章：从零搭建你的最小 Agent

本章将带你从零构建一个**完整可运行的 Agent 系统**。我们将实现一个简化版的 OpenClaw Agent 核心，包含 Agent 循环、工具系统、LLM 调用和事件处理。

> 💡 **动手实践**：本章所有代码均可直接运行。你需要准备一个 OpenAI API Key。

## 11.1 项目结构

```
minimal-agent/
├── package.json          # 依赖配置
├── tsconfig.json         # TypeScript 配置
├── src/
│   ├── agent.ts          # Agent 核心类
│   ├── types.ts          # 类型定义
│   ├── llm.ts            # LLM 流式调用
│   ├── tools.ts          # 工具定义
│   └── index.ts          # 入口文件：测试脚本
```

## 11.2 安装依赖

```bash
mkdir minimal-agent && cd minimal-agent
npm init -y
npm install typescript @types/node openai zod tsx
```

### package.json

```json
{
  "name": "minimal-agent",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "tsx src/index.ts"
  },
  "dependencies": {
    "openai": "^4.80.0",
    "zod": "^3.24.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.7.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "outDir": "dist",
    "declaration": true
  },
  "include": ["src"]
}
```

## 11.3 类型定义

```typescript
// src/types.ts

/** 工具描述符——定义 Agent 可调用的工具 */
export interface ToolDescriptor {
  name: string;
  description: string;
  inputSchema: Record<string, unknown>;
  handler: (args: Record<string, unknown>) => Promise<string>;
}

/** Tool Call —— LLM 请求执行的工具调用 */
export interface ToolCall {
  id: string;
  type: "function";
  function: {
    name: string;
    arguments: string;
  };
}

/** Tool Result —— 工具执行结果 */
export interface ToolResult {
  role: "tool";
  tool_call_id: string;
  content: string;
}

/** 消息类型 */
export type Message = {
  role: "user" | "assistant" | "system" | "tool";
  content: string;
  tool_calls?: ToolCall[];
  tool_call_id?: string;
};

/** Agent 事件——用于事件流 */
export type AgentEvent =
  | { type: "turn_start"; messages: Message[] }
  | { type: "llm_start"; messages: Message[] }
  | { type: "llm_delta"; content: string }
  | { type: "llm_end"; message: Message }
  | { type: "tool_start"; toolCall: ToolCall }
  | { type: "tool_end"; toolCall: ToolCall; result: ToolResult }
  | { type: "turn_end"; message: Message }
  | { type: "agent_end"; messages: Message[] };

/** Token 用量 */
export interface TokenUsage {
  input: number;
  output: number;
  total: number;
}

/** Agent 选项 */
export interface AgentOptions {
  model: string;
  tools: ToolDescriptor[];
  systemPrompt: string;
  maxTurns?: number;
  apiKey: string;
}
```

## 11.4 Agent 核心实现

```typescript
// src/agent.ts
import OpenAI from "openai";
import type {
  AgentOptions,
  AgentEvent,
  Message,
  ToolCall,
  ToolResult,
  TokenUsage,
} from "./types.js";

export class Agent {
  private options: AgentOptions;
  private client: OpenAI;
  private messages: Message[] = [];
  private listeners: Set<(event: AgentEvent) => void> = new Set();
  private toolMap: Map<string, AgentOptions["tools"][0]> = new Map();
  private tokenUsage: TokenUsage = { input: 0, output: 0, total: 0 };

  constructor(options: AgentOptions) {
    this.options = { maxTurns: 25, ...options };
    this.client = new OpenAI({ apiKey: options.apiKey });

    // 构建工具名到工具描述符的映射
    for (const tool of options.tools) {
      this.toolMap.set(tool.name, tool);
    }
  }

  /** 订阅 Agent 事件 */
  subscribe(listener: (event: AgentEvent) => void): () => void {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  /** 发送事件到所有监听器 */
  private emit(event: AgentEvent): void {
    for (const listener of this.listeners) {
      listener(event);
    }
  }

  /** 获取 Token 使用统计 */
  getUsage(): TokenUsage {
    return { ...this.tokenUsage };
  }

  /** Agent 主循环 */
  async run(userInput: string): Promise<Message[]> {
    // 初始化系统提示词
    if (this.messages.length === 0) {
      this.messages.push({ role: "system", content: this.options.systemPrompt });
    }

    // 添加用户消息
    this.messages.push({ role: "user", content: userInput });

    // Agent 循环
    for (let turn = 0; turn < this.options.maxTurns!; turn++) {
      this.emit({ type: "turn_start", messages: this.messages });

      // === 第 1 步：调用 LLM ===
      const assistantMessage = await this.callLLM();

      // 检查是否包含工具调用
      if (assistantMessage.tool_calls && assistantMessage.tool_calls.length > 0) {
        // === 第 2 步：执行工具调用 ===
        for (const toolCall of assistantMessage.tool_calls) {
          this.emit({ type: "tool_start", toolCall });

          const toolResult = await this.executeTool(toolCall);

          this.emit({ type: "tool_end", toolCall, result: toolResult });
          this.messages.push(toolResult);
        }

        // 本轮有工具调用，继续循环
        this.emit({ type: "turn_end", message: assistantMessage });
        continue;
      }

      // 没有工具调用，Agent 输出最终回复
      this.messages.push(assistantMessage);
      this.emit({ type: "turn_end", message: assistantMessage });
      this.emit({ type: "agent_end", messages: this.messages });
      return this.messages;
    }

    // 达到最大轮次
    this.emit({ type: "agent_end", messages: this.messages });
    return this.messages;
  }

  /** 调用 LLM */
  private async callLLM(): Promise<Message> {
    // 构建 OpenAI 消息格式
    const openaiMessages = this.messages.map((msg) => {
      if (msg.role === "tool") {
        return {
          role: "tool" as const,
          tool_call_id: msg.tool_call_id!,
          content: msg.content,
        };
      }
      return {
        role: msg.role as "user" | "assistant" | "system",
        content: msg.content,
        tool_calls: msg.tool_calls as any,
      };
    });

    // 构建工具定义（OpenAI 格式）
    const tools = this.options.tools.map((t) => ({
      type: "function" as const,
      function: {
        name: t.name,
        description: t.description,
        parameters: t.inputSchema,
      },
    }));

    this.emit({ type: "llm_start", messages: this.messages });

    // 流式调用
    const stream = await this.client.chat.completions.create({
      model: this.options.model,
      messages: openaiMessages,
      tools: tools.length > 0 ? tools : undefined,
      stream: true,
    });

    let content = "";
    let toolCalls: ToolCall[] = [];

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta;

      // 累积文本
      if (delta?.content) {
        content += delta.content;
        this.emit({ type: "llm_delta", content: delta.content });
      }

      // 累积工具调用
      if (delta?.tool_calls) {
        for (const tc of delta.tool_calls) {
          if (tc.id) {
            toolCalls.push({
              id: tc.id,
              type: "function",
              function: { name: tc.function?.name || "", arguments: tc.function?.arguments || "" },
            });
          } else {
            // 追加到已有的 tool call
            const last = toolCalls[toolCalls.length - 1];
            if (last && tc.function?.arguments) {
              last.function.arguments += tc.function.arguments;
            }
          }
        }
      }

      // 记录 Token 用量
      if (chunk.usage) {
        this.tokenUsage.input = chunk.usage.prompt_tokens;
        this.tokenUsage.output = chunk.usage.completion_tokens;
        this.tokenUsage.total = chunk.usage.total_tokens;
      }
    }

    const message: Message = {
      role: "assistant",
      content,
      tool_calls: toolCalls.length > 0 ? toolCalls : undefined,
    };

    this.emit({ type: "llm_end", message });
    return message;
  }

  /** 执行工具调用 */
  private async executeTool(toolCall: ToolCall): Promise<ToolResult> {
    const tool = this.toolMap.get(toolCall.function.name);
    if (!tool) {
      return {
        role: "tool",
        tool_call_id: toolCall.id,
        content: `Error: Unknown tool "${toolCall.function.name}"`,
      };
    }

    try {
      const args = JSON.parse(toolCall.function.arguments);
      const result = await tool.handler(args);
      return { role: "tool", tool_call_id: toolCall.id, content: result };
    } catch (error) {
      return {
        role: "tool",
        tool_call_id: toolCall.id,
        content: `Error: ${error instanceof Error ? error.message : String(error)}`,
      };
    }
  }
}
```

## 11.5 工具定义

```typescript
// src/tools.ts
import { z } from "zod";
import type { ToolDescriptor } from "./types.js";

/** 计算器工具 */
const calculatorTool: ToolDescriptor = {
  name: "calculator",
  description: "执行数学计算，支持四则运算",
  inputSchema: {
    type: "object",
    properties: {
      expression: {
        type: "string",
        description: "数学表达式，如 '2 + 3 * 4'",
      },
    },
    required: ["expression"],
  },
  handler: async (args) => {
    const { expression } = z.object({ expression: z.string() }).parse(args);
    try {
      // ⚠️ 注意：eval 有安全风险，仅用于演示
      const result = Function(`"use strict"; return (${expression})`)();
      return `计算结果：${expression} = ${result}`;
    } catch (error) {
      return `计算错误：${(error as Error).message}`;
    }
  },
};

/** 天气查询工具（模拟） */
const weatherTool: ToolDescriptor = {
  name: "get_weather",
  description: "查询指定城市的天气信息",
  inputSchema: {
    type: "object",
    properties: {
      city: {
        type: "string",
        description: "城市名称，如 '北京'、'上海'",
      },
    },
    required: ["city"],
  },
  handler: async (args) => {
    const { city } = z.object({ city: z.string() }).parse(args);
    // 模拟天气数据
    const weathers: Record<string, string> = {
      "北京": "晴，25°C，湿度 45%",
      "上海": "多云，28°C，湿度 70%",
      "广州": "阵雨，30°C，湿度 85%",
      "深圳": "晴，29°C，湿度 65%",
    };
    const info = weathers[city] || `${city}，22°C，湿度 60%`;
    return `${city} 天气：${info}`;
  },
};

export const tools = [calculatorTool, weatherTool];
```

## 11.6 入口文件

```typescript
// src/index.ts
import { Agent } from "./agent.js";
import { tools } from "./tools.js";

const API_KEY = process.env.OPENAI_API_KEY;
if (!API_KEY) {
  console.error("请设置 OPENAI_API_KEY 环境变量");
  process.exit(1);
}

// 创建 Agent 实例
const agent = new Agent({
  model: "gpt-4o-mini",
  tools,
  systemPrompt: `你是一个有帮助的 AI 助手。你可以使用工具来回答问题。
- 使用 calculator 进行数学计算
- 使用 get_weather 查询天气

请尽可能使用工具来提供准确的信息。`,
  apiKey: API_KEY,
});

// 订阅事件——实时输出
agent.subscribe((event) => {
  switch (event.type) {
    case "turn_start":
      console.log("\n🔵 开始新轮次");
      break;
    case "llm_start":
      process.stdout.write("\n🤔 思考中...");
      break;
    case "llm_delta":
      process.stdout.write(event.content);
      break;
    case "llm_end":
      console.log("\n");
      break;
    case "tool_start":
      console.log(`🛠️  调用工具: ${event.toolCall.function.name}`);
      console.log(`   参数: ${event.toolCall.function.arguments}`);
      break;
    case "tool_end":
      console.log(`✅ 工具结果: ${event.result.content.slice(0, 100)}`);
      break;
    case "turn_end":
      console.log("---");
      break;
    case "agent_end":
      const usage = agent.getUsage();
      console.log(`\n📊 Token 用量: 输入 ${usage.input} + 输出 ${usage.output} = ${usage.total}`);
      break;
  }
});

// 运行测试
async function main() {
  console.log("🤖 Minimal Agent Demo\n");

  const testCases = [
    "计算 (123 + 456) * 2 的结果",
    "北京今天天气怎么样？",
    "你好，请自我介绍一下",
  ];

  for (const test of testCases) {
    console.log(`\n${"=".repeat(50)}`);
    console.log(`📝 用户: ${test}`);
    console.log("=".repeat(50));

    try {
      await agent.run(test);
    } catch (error) {
      console.error("Agent 运行错误:", error);
    }

    console.log("\n");
  }
}

main().catch(console.error);
```

## 11.7 运行测试

```bash
export OPENAI_API_KEY="sk-..."
npm start
```

预期输出示例：

```
🤖 Minimal Agent Demo

==================================================
📝 用户: 计算 (123 + 456) * 2 的结果
==================================================

🔵 开始新轮次
🤔 思考中...
🛠️  调用工具: calculator
   参数: {"expression": "(123 + 456) * 2"}
✅ 工具结果: 计算结果：(123 + 456) * 2 = 1158
🔵 开始新轮次
🤔 思考中...(123 + 456) * 2 = 1158
---
📊 Token 用量: 输入 85 + 输出 45 = 130

==================================================
📝 用户: 北京今天天气怎么样？
==================================================

🔵 开始新轮次
🤔 思考中...
🛠️  调用工具: get_weather
   参数: {"city": "北京"}
✅ 工具结果: 北京天气：晴，25°C，湿度 45%
🔵 开始新轮次
🤔 思考中...北京今天天气晴朗，气温 25°C...
---
📊 Token 用量: 输入 120 + 输出 60 = 180
```

## 11.8 关键模式解析

### 11.8.1 Agent 循环

```
用户输入 → [Agent 循环] → 最终回复
              │
              ├─ 调用 LLM（流式）
              ├─ 检查是否有工具调用
              ├─ 有 → 执行工具 → 回到 LLM 调用
              └─ 无 → 返回最终消息
```

### 11.8.2 工具描述符与 JSON Schema

每个工具都有一个 `inputSchema`，使用 JSON Schema 格式描述参数。LLM 根据这些 schema 生成结构化的工具调用参数。

### 11.8.3 顺序执行 vs 并行执行

本实现中工具按顺序执行（`for` 循环）。OpenClaw 的完整实现支持并行执行模式：

```typescript
// 并行执行示例
if (assistantMessage.tool_calls) {
  const results = await Promise.all(
    assistantMessage.tool_calls.map(tc => this.executeTool(tc))
  );
  for (const result of results) {
    this.messages.push(result);
  }
}
```

### 11.8.4 消息格式转换

`callLLM()` 方法将内部 `Message[]` 转换为 OpenAI API 的格式。在实际生产环境中，需要适配多种 LLM 提供商（Anthropic、Google 等），每个提供商的消息格式不同。

### 11.8.5 Token 用量追踪

系统通过 `chunk.usage` 字段追踪 Token 消耗。在 OpenClaw 的完整实现中，Token 用量会被归一化为统一的 `TokenUsage` 结构，并用于成本核算和上下文窗口管理。

## 11.9 扩展方向

### 11.9.1 添加更多工具

```typescript
// 搜索工具
const searchTool: ToolDescriptor = {
  name: "web_search",
  description: "搜索互联网信息",
  inputSchema: {
    type: "object",
    properties: { query: { type: "string" } },
    required: ["query"],
  },
  handler: async (args) => {
    // 调用搜索 API
    return "搜索结果...";
  },
};
```

### 11.9.2 添加插件系统

```typescript
interface Plugin {
  name: string;
  tools: ToolDescriptor[];
  onLoad?: () => Promise<void>;
  onUnload?: () => Promise<void>;
}

class PluginManager {
  private plugins: Map<string, Plugin> = new Map();

  async load(plugin: Plugin): Promise<void> {
    await plugin.onLoad?.();
    this.plugins.set(plugin.name, plugin);
  }

  getTools(): ToolDescriptor[] {
    return [...this.plugins.values()].flatMap(p => p.tools);
  }
}
```

### 11.9.3 添加 WebSocket 网关

```typescript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ port: 8080 });

wss.on("connection", (ws) => {
  const agent = new Agent({ ... });

  agent.subscribe((event) => {
    ws.send(JSON.stringify(event));
  });

  ws.on("message", (data) => {
    const { text } = JSON.parse(data.toString());
    agent.run(text);
  });
});
```

### 11.9.4 添加会话持久化

```typescript
import { readFile, writeFile } from "node:fs/promises";

class SessionStore {
  async save(sessionKey: string, messages: Message[]): Promise<void> {
    await writeFile(`sessions/${sessionKey}.json`, JSON.stringify(messages));
  }

  async load(sessionKey: string): Promise<Message[]> {
    try {
      return JSON.parse(await readFile(`sessions/${sessionKey}.json`, "utf-8"));
    } catch {
      return [];
    }
  }
}
```

## 11.10 与 OpenClaw 的对比

| 特性 | 最小实现 | OpenClaw |
|------|----------|----------|
| LLM 提供商 | 仅 OpenAI | 多提供商（Anthropic、Google、Ollama 等） |
| 工具系统 | 基础描述符 | 工具规划（可见/隐藏）、权限策略、审批 |
| 事件流 | 简单 EventEmitter | 标准化事件流、转录、回放 |
| 会话管理 | 无 | 文件存储、缓存、维护、归档 |
| 消息队列 | 无 | Steering/Follow-Up 双队列 |
| 沙箱 | 无 | Docker/SSH 沙箱、工具策略 |
| 通道集成 | 无 | Slack、Discord、飞书、Telegram 等 |
| 插件系统 | 无 | 完整 Plugin SDK、生命周期钩子 |

本章的最小实现涵盖了 Agent 的核心模式，是理解 OpenClaw 完整架构的最佳起点。

---

🎉 **恭喜你完成了全部 11 章的学习！**

**回顾学习路径**：
- [第 1 章](ch01-introduction/README.md) 到 [第 2 章](ch02-project-skeleton/README.md)：了解 OpenClaw 是什么、项目如何组织
- [第 3 章](ch03-agent-core/README.md) 到 [第 4 章](ch04-agent-loop/README.md)：掌握 Agent 核心抽象和核心循环（**系统的大脑**）
- [第 5 章](ch05-tool-system/README.md) 到 [第 6 章](ch06-llm-integration/README.md)：理解工具系统和 LLM 集成（**大脑的两只手**）
- [第 7 章](ch07-gateway-messaging/README.md) 到 [第 8 章](ch08-plugin-sdk/README.md)：掌握网关通信和插件扩展（**连接外部世界的桥梁**）
- [第 9 章](ch09-sandbox-security/README.md) 到 [第 10 章](ch10-config-session/README.md)：理解安全防护和运行时上下文（**护栏与底座**）
- [第 11 章](ch11-minimal-agent/README.md)：动手实现最小 Agent（**理论落地实践**）

**快速速查**：如需快速回忆关键术语，参见 [附录：核心概念速查表](appendix-concepts/README.md)。