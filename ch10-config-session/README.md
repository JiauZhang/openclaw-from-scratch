# 第十章：配置与会话

本章介绍 OpenClaw 的配置系统和会话管理机制，它们共同构成了 Agent 运行时的上下文环境。

## 10.1 配置系统

配置系统是 OpenClaw 的基石，所有 Agent 行为、工具策略、通道绑定等都由配置文件驱动。

### 10.1.1 OpenClawConfig 类型

`OpenClawConfig` 是整个配置系统的顶层类型，定义在 `src/config/types.openclaw.ts` 中：

```typescript
// src/config/types.openclaw.ts
export type OpenClawConfig = {
  /** JSON schema URL */
  $schema?: string;
  /** 认证提供者配置 */
  auth?: AuthConfig;
  /** Agent 定义、默认值、绑定 */
  agents?: AgentsConfig;
  /** 工具暴露策略 */
  tools?: ToolsConfig;
  /** 通道配置 */
  channels?: ChannelsConfig;
  /** 模型提供者、模型目录、定价 */
  models?: ModelsConfig;
  /** 插件注册 */
  plugins?: PluginsConfig;
  // ... 数十个可选配置段
};
```

配置系统使用**品牌类型**（Branded Types）来区分不同的配置状态：

```typescript
// 编写后的原始配置（包含 $include 和 ${ENV} 占位符）
export type SourceConfig = BrandedConfigState<"source">;
// 解析后的配置（$include 和 ${ENV} 已展开）
export type ResolvedSourceConfig = BrandedConfigState<"resolved-source">;
// 运行时配置（已应用默认值）
export type RuntimeConfig = BrandedConfigState<"runtime">;
```

### 10.1.2 配置加载与运行时快照

配置的加载流程如下：

```
SourceConfig（用户编写的 YAML/JSON/JSONC）
  │  $include 解析
  │  ${ENV} 环境变量替换
  ▼
ResolvedSourceConfig（解析后的源配置）
  │  运行时默认值填充
  │  模式验证
  ▼
RuntimeConfig（运行时配置，不可变快照）
```

核心 API 通过 `src/config/config.ts` 暴露：

```typescript
// 获取运行时配置快照
import { getRuntimeConfig, loadConfig } from "./config/config.js";

// 加载配置（解析 include、环境变量、验证）
const configSnapshot = await loadConfig();

// 获取当前运行时配置（懒加载）
const runtimeConfig = getRuntimeConfig();

// 获取运行时快照元数据
const metadata = getRuntimeConfigSnapshotMetadata();
```

`getRuntimeConfig()` 返回的是一个**不可变快照**，确保在 Agent 运行过程中配置不会意外变化。

### 10.1.3 Agent 特定配置

Agent 配置位于 `agents` 段下，分为**默认值**和**列表**两部分：

```typescript
// src/config/types.agents.ts
export type AgentsConfig = {
  defaults?: AgentDefaultsConfig;
  list?: AgentConfig[];
};

export type AgentConfig = {
  id: string;         // Agent 唯一标识符
  default?: boolean;  // 是否为默认 Agent
  name?: string;      // 显示名称
  model?: AgentModelConfig;
  tools?: AgentToolsConfig;
  sandbox?: AgentSandboxConfig;
  // ... 其他配置
};
```

关键配置项示例：

```yaml
# openclaw.yaml
agents:
  defaults:
    model: "anthropic/claude-sonnet-4-20250514"
    timeoutSeconds: 172800  # 48 小时
    sandbox:
      mode: "non-main"
      backend: "docker"
      docker:
        image: "openclaw-sandbox:bookworm-slim"
        workdir: "/workspace"
        memory: "8g"
        cpus: "4"
  list:
    - id: "my-agent"
      name: "我的助手"
      model: "openai/gpt-4o"
      timeoutSeconds: 3600  # 覆盖默认值，1 小时
      sandbox:
        mode: "all"
```

## 10.2 会话管理

会话（Session）是 Agent 对话的持久化单位，包含了消息历史、元数据、路由信息等。

### 10.2.1 会话键

会话键（Session Key）是会话的唯一标识符，用于在会话存储中定位数据。

```typescript
// src/config/sessions/session-key.ts

// 派生会话键：从消息上下文中确定应使用的存储桶
export function deriveSessionKey(scope: SessionScope, ctx: MsgContext) {
  if (scope === "global") {
    return "global";
  }
  // 群组会话使用群组键
  const resolvedGroup = resolveGroupSessionKey(ctx);
  if (resolvedGroup) {
    return resolvedGroup.key;
  }
  // 私聊会话使用发送者 ID
  const from = ctx.From ? normalizeE164(ctx.From) : "";
  return from || "unknown";
}

// 解析最终的会话存储键
export function resolveSessionKey(
  scope: SessionScope,
  ctx: MsgContext,
  mainKey?: string,
  agentId: string = DEFAULT_AGENT_ID,
) {
  // 显式会话键优先
  const explicit = ctx.SessionKey?.trim();
  if (explicit) {
    return normalizeExplicitSessionKey(explicit, ctx);
  }
  // 否则基于作用域和上下文派生
  const raw = deriveSessionKey(scope, ctx);
  // 群组/频道会话保持隔离
  if (isGroup) {
    return `agent:${canonicalAgentId}:${raw}`;
  }
  // 私聊归入 Agent 的主会话桶
  return canonical;
}
```

会话键的层次结构：
- **Global scope**：固定为 `"global"`
- **私聊**：归入 `agent:<agentId>:main` 主会话
- **群组/频道**：格式为 `agent:<agentId>:group:<groupId>` 或 `agent:<agentId>:channel:<channelId>`
- **显式键**：通过 `SessionKey` 字段直接指定，绕过派生逻辑

### 10.2.2 会话存储

会话存储（Session Store）是与会话的持久化接口，基于文件系统实现：

```typescript
// src/config/sessions/store.ts
export async function updateSessionStore<T>(
  storePath: string,
  mutator: (store: Record<string, SessionEntry>) => Promise<T> | T,
  opts?: UpdateSessionStoreOptions<T>,
): Promise<T> {
  // 使用排他写锁，确保并发安全
  return await runExclusiveSessionStoreWrite(storePath, async () => {
    const store = loadMutableSessionStoreForWriter(storePath);
    const result = await mutator(store);
    await saveSessionStoreUnlocked(storePath, store, { ... });
    return result;
  });
}
```

会话存储的关键能力：
- **排他写入**：通过 `runExclusiveSessionStoreWrite` 保证同一时刻只有一个写入者
- **缓存**：`store-cache.ts` 维护序列化缓存，避免重复解析 JSON
- **维护**：自动裁剪过期条目、旋转文件、清理孤儿转录
- **单条目持久化**：热路径只重写修改的条目，而非整个文件

### 10.2.3 会话生命周期

会话的完整生命周期包括创建、持久化和恢复：

**创建**：当 Agent 收到第一条消息时，如果该会话键不存在，会自动创建新的会话条目。

```typescript
export async function recordSessionMetaFromInbound(params: {
  storePath: string;
  sessionKey: string;
  ctx: MsgContext;
}): Promise<SessionEntry | null> {
  // 从消息上下文中提取元数据补丁
  const patch = deriveSessionMetaPatch({ ctx, sessionKey, existing });
  // 合并到现有条目或创建新条目
  const next = existing
    ? mergeSessionEntryPreserveActivity(existing, patch)
    : mergeSessionEntry(existing, patch);
  // 持久化
  return await persistResolvedSessionEntry({ storePath, store, resolved, next, ... });
}
```

**持久化**：每次 Agent 轮次结束后，新的消息和工具结果会被追加到会话条目中。

**恢复**：当 Agent 在同一会话中继续对话时，会从存储中加载完整的消息历史作为上下文。

**重置**：会话可以重置（`resetSessionEntryLifecycle`），这会创建新的会话文件并归档旧的转录。

**删除**：过期或不需要的会话可以被删除（`deleteSessionEntryLifecycle`），同时清理关联的转录文件。

## 10.3 Agent 超时解析

Agent 运行超时是一个重要的安全机制，防止 Agent 无限期运行。

### 10.3.1 超时解析函数

```typescript
// src/agents/timeout.ts
const DEFAULT_AGENT_TIMEOUT_SECONDS = 48 * 60 * 60;  // 48 小时
export const DEFAULT_AGENT_TIMEOUT_MS = DEFAULT_AGENT_TIMEOUT_SECONDS * 1000;

export function resolveAgentTimeoutMs(opts: {
  cfg?: OpenClawConfig;
  overrideMs?: number | null;
  overrideSeconds?: number | null;
  minMs?: number;
}): number {
  // 1. 从配置中获取默认超时
  const defaultMs = clampTimeoutMs(resolveAgentTimeoutSeconds(opts.cfg) * 1000);

  // 2. 检查 per-run 覆盖（毫秒）
  if (overrideMs !== undefined) {
    if (overrideMs === 0) return NO_TIMEOUT_MS;     // 0 = 无超时
    if (overrideMs < 0) return defaultMs;            // 负数 = 使用默认值
    return clampTimeoutMs(overrideMs);
  }

  // 3. 检查 per-run 覆盖（秒）
  if (overrideSeconds !== undefined) {
    if (overrideSeconds === 0) return NO_TIMEOUT_MS;
    if (overrideSeconds < 0) return defaultMs;
    return clampTimeoutMs(overrideSeconds * 1000);
  }

  // 4. 返回默认值
  return defaultMs;
}
```

### 10.3.2 超时优先级

```
用户显式设置 0（无超时）
  → MAX_TIMER_TIMEOUT_MS（约 2^31-1 毫秒 ≈ 24.8 天）
用户显式设置正数
  → 对应的毫秒值（受 clampTimerTimeoutMs 限制）
用户显式设置负数
  → 使用配置值
配置中 agents.defaults.timeoutSeconds
  → 配置值（默认 172800 秒 = 48 小时）
```

## 10.4 Agent 配置路径

Agent 的配置路径系统决定了运行时文件、主题、二进制文件、会话文件等的位置。

### 10.4.1 路径解析函数

```typescript
// src/agents/config.ts
import { homedir } from "node:os";
import { join } from "node:path";

// 从 package.json 读取应用名和配置目录名
export const APP_NAME: string = pkg.openclawConfig?.name || "openclaw";
export const CONFIG_DIR_NAME: string = pkg.openclawConfig?.configDir || ".openclaw";
export const VERSION: string = pkg.version || "0.0.0";

// 获取 Agent 配置目录（~/.openclaw/agent/）
export function getAgentDir(): string {
  const envDir = process.env[`${APP_NAME.toUpperCase()}_AGENT_DIR`];
  if (envDir) return expandTildePath(envDir);
  return join(homedir(), CONFIG_DIR_NAME, "agent");
}

// 获取主题目录
export function getThemesDir(): string {
  if (isBunBinary) {
    return join(getPackageDir(), "theme");
  }
  return join(getPackageSourceOrDistDir(), "agents", "modes", "interactive", "theme");
}

// 获取托管二进制文件目录（fd, rg 等）
export function getBinDir(): string {
  return join(getAgentDir(), "bin");
}

// 获取会话目录
export function getSessionsDir(): string {
  return join(getAgentDir(), "sessions");
}
```

### 10.4.2 包检测

配置路径系统需要检测运行环境，以正确解析资源路径：

```typescript
// 检测是否为 Bun 编译的二进制文件
export const isBunBinary =
  import.meta.url.includes("$bunfs") ||
  import.meta.url.includes("~BUN") ||
  import.meta.url.includes("%7EBUN");
```

不同的运行环境对应不同的路径策略：

| 环境 | 包目录 | 主题目录 |
|------|--------|----------|
| Bun 二进制 | 可执行文件所在目录 | 可执行文件旁的 `theme/` |
| Node.js（dist/） | 向上查找 `package.json` | `dist/agents/modes/interactive/theme/` |
| tsx（src/） | 当前目录的父目录 | `src/agents/modes/interactive/theme/` |

### 10.4.3 环境变量覆盖

路径系统支持通过环境变量覆盖：

- `OPENCLAW_PACKAGE_DIR`：覆盖包资源基础目录
- `OPENCLAW_AGENT_DIR`：覆盖 Agent 配置目录（对应 `~/.openclaw/agent/`）

这些覆盖支持 `~` 和 `~/` 路径展开，方便在 Nix/Guix 等不可变存储环境中使用。

## 10.5 配置与会话的关系

配置和会话在 Agent 运行时中紧密协作：

```
配置文件（openclaw.yaml）
  │
  ├─→ 解析 Agent 定义（agents.list[]）
  │     ├─ model、tools、sandbox 等配置
  │     └─ timeoutSeconds 等运行时参数
  │
  ├─→ 提供会话存储路径（getSessionsDir()）
  │
  └─→ 影响会话行为
        ├─ session 段配置键策略
        ├─ tools 段配置工具策略
        └─ agents.defaults.sandbox 配置沙箱策略

会话（Session Store）
  │
  ├─→ 使用 resolveSessionKey() 确定存储键
  ├─→ 使用 updateSessionStore() 读写数据
  ├─→ 使用 resolveAgentTimeoutMs() 控制运行时长
  └─→ 使用 getSessionsDir() 确定存储位置
```

这种分工使得配置系统关注"应该怎么做"，而会话系统关注"已经做了什么"，两者共同构成了 Agent 运行时的完整上下文。

---

上一章（[第 9 章](ch09-sandbox-security/README.md)）讲解了沙箱的运行时机制，沙箱配置的具体来源正是本章的 `OpenClawConfig`。下一章（[第 11 章](ch11-minimal-agent/README.md)）将带你**从零搭建一个完整可运行的最小 Agent**——将前 10 章的所有概念落地为真实代码。

> 🔗 术语呼应：本章的 **`AgentsConfig.defaults.model`** 被第 6 章 `resolveDefaultModelForAgent` 读取作为默认模型；**`resolveAgentTimeoutMs`** 控制第 4 章 Agent 循环的运行时长；**`resolveSessionKey`** 生成的会话键是沙箱第 9 章 `ensureSandboxWorkspaceForSession` 的核心参数。