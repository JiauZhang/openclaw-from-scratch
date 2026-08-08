# 第九章：沙箱与安全

本章深入 OpenClaw 的沙箱（Sandbox）系统，重点讲解它如何与 Agent 运行时交互，为 Agent 提供一个安全、隔离的执行环境。

## 9.1 沙箱概述

Sandbox 是 OpenClaw 的安全执行环境，它决定了 Agent 的代码在何处运行、能访问哪些资源、能调用哪些工具。对于 AI Agent 来说，沙箱是**安全的第一道防线**——它防止恶意或错误的模型输出对宿主机造成损害。

### 9.1.1 Sandbox Barrel

`src/agents/sandbox.ts` 是整个沙箱模块的公共导出表面（barrel），它将所有沙箱子模块统一暴露给 Agent 运行时。核心的导出分组包括：

| 模块 | 职责 | 核心导出 |
|------|------|----------|
| `config.ts` | 沙箱配置解析 | `resolveSandboxConfigForAgent`, `resolveSandboxScope` |
| `backend.ts` | 后端注册与工厂 | `registerSandboxBackend`, `getSandboxBackendFactory` |
| `docker.ts` | Docker 沙箱实现 | `buildSandboxCreateArgs`, `isDockerDaemonUnavailable` |
| `ssh.ts` | SSH 沙箱传输 | `createSshSandboxSession`, `runSshSandboxCommand` |
| `fs-bridge.ts` | 文件系统桥接 | `SandboxFsBridge`, `SandboxResolvedPath` |
| `tool-policy.ts` | 工具权限策略 | `isToolAllowed`, `resolveSandboxToolPolicyForAgent` |
| `context.ts` | 沙箱上下文解析 | `ensureSandboxWorkspaceForSession`, `resolveSandboxContext` |

### 9.1.2 沙箱配置解析

```typescript
// src/agents/sandbox/config.ts
export function resolveSandboxConfigForAgent(
  cfg?: OpenClawConfig,
  agentId?: string,
): SandboxConfig {
  // 先读取全局默认配置
  const agent = cfg?.agents?.defaults?.sandbox;

  // 再读取 Agent 特定的覆盖配置
  let agentSandbox: typeof agent | undefined;
  const agentConfig = cfg && agentId ? resolveAgentConfig(cfg, agentId) : undefined;
  if (agentConfig?.sandbox) {
    agentSandbox = agentConfig.sandbox;
  }

  // 合并作用域
  const scope = resolveSandboxScope({ ... });

  // 解析工具策略
  const toolPolicy = resolveSandboxToolPolicyForAgent(cfg, agentId);

  return {
    mode: agentSandbox?.mode ?? agent?.mode ?? "off",
    backend: agentSandbox?.backend?.trim() || agent?.backend?.trim() || "docker",
    scope,
    workspaceAccess: agentSandbox?.workspaceAccess ?? agent?.workspaceAccess ?? "none",
    docker: resolveSandboxDockerConfig({ ... }),
    ssh: resolveSandboxSshConfig({ ... }),
    browser: resolveSandboxBrowserConfig({ ... }),
    tools: { allow: toolPolicy.allow, deny: toolPolicy.deny },
    prune: resolveSandboxPruneConfig({ ... }),
  };
}
```

**关键层次**：全局默认 (`agents.defaults.sandbox`) → Agent 特定 (`agents.list[].sandbox`) → 最终合并。Agent 级别的配置会覆盖全局默认。`scope` 参数控制沙箱的作用域：`"session"`（每会话独立）、`"agent"`（每 Agent 共享）、`"shared"`（全局共享）。

### 9.1.3 沙箱上下文与会话工作区

`ensureSandboxWorkspaceForSession` 和 `resolveSandboxContext` 是 Agent 运行时与沙箱之间的桥梁。

```typescript
// src/agents/sandbox/context.ts
export async function ensureSandboxWorkspaceForSession(params: {
  config?: OpenClawConfig;
  sessionKey?: string;
  workspaceDir?: string;
}): Promise<SandboxWorkspaceInfo | null> {
  // 检查会话是否需要沙箱
  const resolved = resolveSandboxSession(params);
  if (!resolved) return null;

  // 创建工作区布局
  const { agentWorkspaceDir, scopeKey, skillsWorkspaceDir, workspaceDir } =
    await ensureSandboxWorkspaceLayout({ ... });

  // 返回工作区信息
  return { workspaceDir, containerWorkdir, skillsWorkspaceDir, workspaceAccess };
}
```

当 Agent 会话启动时，系统会：
1. 判断该会话是否需要沙箱（通过 `resolveSandboxRuntimeStatus`）
2. 解析沙箱配置（`resolveSandboxConfigForAgent`）
3. 创建工作区目录结构
4. 同步技能（Skills）到沙箱工作区
5. 创建后端实例（Docker 容器或 SSH 连接）

## 9.2 沙箱后端

沙箱后端是实际的执行环境抽象层。OpenClaw 使用**注册表模式**（Registry Pattern）来管理多个后端实现。

### 9.2.1 后端注册表

```typescript
// src/agents/sandbox/backend.ts
const SANDBOX_BACKEND_FACTORIES_STATE_KEY = Symbol.for("openclaw.sandboxBackendFactories");

// 获取进程级后端注册表（全局 Map）
function getSandboxBackendFactories(): Map<SandboxBackendId, RegisteredSandboxBackend> {
  const globalStore = globalThis as typeof globalThis & {
    [SANDBOX_BACKEND_FACTORIES_STATE_KEY]?: Map<SandboxBackendId, RegisteredSandboxBackend>;
  };
  globalStore[SANDBOX_BACKEND_FACTORIES_STATE_KEY] ??= new Map();
  return globalStore[SANDBOX_BACKEND_FACTORIES_STATE_KEY];
}

// 注册后端
export function registerSandboxBackend(
  id: string,
  registration: SandboxBackendRegistration,
): () => void {
  // 注册并返回撤销函数
}

// 查询后端工厂
export function getSandboxBackendFactory(id: string): SandboxBackendFactory | null {
  return getSandboxBackendFactories().get(normalizeSandboxBackendId(id))?.factory ?? null;
}
```

### 9.2.2 内置后端：Docker 和 SSH

OpenClaw 在模块加载时自动注册两个内置后端：

```typescript
// 自动注册 Docker 后端
registerSandboxBackend("docker", {
  factory: createDockerSandboxBackend,
  manager: dockerSandboxBackendManager,
  resolveWorkdir: ({ cfg }) => cfg.docker.workdir,
});

// 自动注册 SSH 后端
registerSandboxBackend("ssh", {
  factory: createSshSandboxBackend,
  manager: sshSandboxBackendManager,
  resolveWorkdir: ({ cfg, scopeKey }) =>
    resolveSshRuntimePaths(cfg.ssh.workspaceRoot, scopeKey).remoteWorkspaceDir,
});
```

`SandboxBackendFactory` 是核心接口，接收 `CreateSandboxBackendParams` 返回 `SandboxBackendHandle`：

```typescript
export type SandboxBackendFactory = (
  params: CreateSandboxBackendParams,
) => Promise<SandboxBackendHandle>;
```

`SandboxBackendHandle` 包含 `runtimeId`、`workdir`、`capabilities` 等信息，以及可选的 `createFsBridge` 方法。

### 9.2.3 后端与 Agent 的关系

对于 Agent 而言，后端是透明的。Agent 运行时通过配置中的 `backend` 字段（如 `"docker"` 或 `"ssh"`）选择后端，然后通过 `requireSandboxBackendFactory` 获取工厂函数创建实例。这种设计让 Agent 可以灵活切换执行环境，而无需修改核心逻辑。

## 9.3 工具策略

沙箱的核心安全机制之一是**工具权限策略**（Tool Policy），它决定了沙箱内的 Agent 可以调用哪些工具。

### 9.3.1 默认策略

```typescript
// src/agents/sandbox/constants.ts
export const DEFAULT_TOOL_ALLOW = [
  "exec", "process", "read", "write", "edit",
  "apply_patch", "image",
  "sessions_list", "sessions_history", "sessions_send",
  "sessions_spawn", "sessions_yield", "subagents", "session_status",
];

export const DEFAULT_TOOL_DENY = [
  "browser", "canvas", "nodes", "cron", "gateway",
  ...CHANNEL_IDS,  // 所有通道 ID（slack, discord, feishu 等）
];
```

默认允许的是执行命令、文件操作、会话管理等 Agent 核心工具；默认拒绝的是浏览器自动化、通道消息发送等有安全隐患的工具。

### 9.3.2 策略解析

```typescript
// src/agents/sandbox/tool-policy.ts
export function resolveSandboxToolPolicyForAgent(
  cfg?: OpenClawConfig,
  agentId?: string,
): SandboxToolPolicyResolved {
  // 优先级：Agent 级别 > 全局级别 > 默认
  const allowConfig = pickConfiguredList({
    agent: agentPolicy?.allow,
    global: globalPolicy?.allow,
  });
  const denyConfig = pickConfiguredDeny({
    agent: agentPolicy?.deny,
    global: globalPolicy?.deny,
  });

  // 合并 allow 和 alsoAllow
  const resolvedAllow = mergeAllowlist(allowConfig.values, alsoAllowConfig.values);
  const resolvedDeny = Array.isArray(denyConfig.values)
    ? [...denyConfig.values]
    : filterDefaultDenyForExplicitAllows({ ... });

  // 展开工具组并返回
  return { allow: expanded.allow, deny: expanded.deny, sources: { ... } };
}
```

### 9.3.3 工具检查

```typescript
export function isToolAllowed(policy: SandboxToolPolicy, name: string) {
  const { blockedByDeny, blockedByAllow } = classifyToolAgainstSandboxToolPolicy(name, policy);
  return !blockedByDeny && !blockedByAllow;
}
```

检查逻辑：
1. 若工具在 deny 列表中 → 拒绝
2. 若工具不在 allow 列表中（且 allow 列表非空）→ 拒绝
3. 否则 → 允许

**特殊规则**：`image` 工具在多模态工作流中至关重要，即使 Agent 没有显式列出它，系统也会自动将其加入 allow 列表，除非它被显式 deny。

### 9.3.4 策略对 Agent 的影响

当 Agent 运行在沙箱中时，Agent 循环的工具执行阶段会调用 `isToolAllowed` 来检查每个工具调用。被拒绝的工具不会执行，而是返回一个格式化的"策略阻止"消息，并且该信息会作为 tool result 返回给 LLM。

## 9.4 SSH 沙箱

SSH 沙箱允许 Agent 在远程服务器上执行命令，适用于需要专用硬件或特定网络环境的场景。

### 9.4.1 SSH 会话管理

```typescript
// src/agents/sandbox/ssh.ts
export async function createSshSandboxSessionFromSettings(
  settings: SshSandboxSettings,
): Promise<SshSandboxSession> {
  // 解析 SSH 目标（user@host:port）
  const parsed = parseSshTarget(settings.target);

  // 创建临时目录存放 SSH 配置和密钥材料
  const configDir = await fs.mkdtemp(
    path.join(resolveSshTmpRoot(), "openclaw-sandbox-ssh-"),
  );

  // 写入 SSH config（权限 600）
  const configPath = path.join(configDir, "config");
  await fs.writeFile(configPath, sshConfigText, { encoding: "utf8", mode: 0o600 });

  return { command: "ssh", configPath, host: hostAlias };
}

// 清理 SSH 会话
export async function disposeSshSandboxSession(session: SshSandboxSession): Promise<void> {
  await fs.rm(path.dirname(session.configPath), { recursive: true, force: true });
}
```

### 9.4.2 远程命令执行

```typescript
export async function runSshSandboxCommand(
  params: RunSshSandboxCommandParams,
): Promise<SandboxBackendCommandResult> {
  // 构建 ssh argv
  const argv = buildSshSandboxArgv({
    session: params.session,
    remoteCommand: params.remoteCommand,
    tty: params.tty,
  });

  // 使用 spawn 执行 ssh 子进程
  const child = spawn(argv[0], argv.slice(1), {
    stdio: ["pipe", "pipe", "pipe"],
    env: sshEnv,
    signal: params.signal,
  });

  // 收集 stdout/stderr
  // 处理 close 事件，返回 { stdout, stderr, code }
}
```

### 9.4.3 Shell 转义

SSH 沙箱需要将命令参数安全地传递给远程 shell。`shellEscape` 函数使用 POSIX 单引号转义：

```typescript
export function shellEscape(value: string): string {
  return `'${value.replaceAll("'", `'"'"'`)}'`;
}
```

这个实现将单引号替换为 `'"'"'`（结束单引号 → 双引号包裹文字单引号 → 恢复单引号），确保即使包含特殊字符的参数也能安全传递。

### 9.4.4 远程文件操作

SSH 沙箱支持将本地工作区目录上传到远程服务器：

```typescript
export async function uploadDirectoryToSshTarget(params: {
  session: SshSandboxSession;
  localDir: string;
  remoteDir: string;
  remoteRootDir?: string;
  signal?: AbortSignal;
}): Promise<void> {
  // 1. 检查本地符号链接是否安全（不逃逸工作区）
  await assertSafeUploadSymlinks(params.localDir);

  // 2. 构建远程 shell 命令：确保目录存在 + tar 解包
  const remoteCommand = buildRemoteCommand([
    "/bin/sh", "-c",
    `${ENSURE_REMOTE_REAL_DIRECTORY_SCRIPT}\ntar -xf - -C "$1"`,
    "openclaw-sandbox-upload",
    params.remoteDir,
    params.remoteRootDir ?? params.remoteDir,
  ]);

  // 3. 使用 tar 管道通过 ssh 传输
  const tar = spawn("tar", ["-C", params.localDir, "-cf", "-", "."]);
  const ssh = spawn(sshArgv[0], sshArgv.slice(1), { ... });
  tar.stdout.pipe(ssh.stdin);
}
```

这种 `tar | ssh` 的管道模式非常高效，它将本地目录打包后通过 SSH 连接直接传输到远程服务器，无需中间文件。

### 9.4.5 远程命令验证

SSH 沙箱包含一个强大的 shell 命令解析器，用于验证模型生成的命令是否安全：

```typescript
export function buildValidatedExecRemoteCommand(params: {
  command: string;
  workdir?: string;
  env: Record<string, string>;
}): string {
  // 检查命令语法是否合法
  assertValidExecRemoteCommand(params.command);
  return buildExecRemoteCommand(params);
}
```

`assertValidExecRemoteCommand` 实现了一个完整的 POSIX shell 解析器，检查：
- 引号是否匹配（单引号、双引号）
- 命令替换是否完整（`$()`、反引号）
- HERE-doc 是否有终止符
- 是否存在未解析的占位符（`<PLACEHOLDER>`）
- 算术扩展是否闭合

## 9.5 沙箱与 Agent 的集成

在实际的 Agent 运行中，沙箱的集成流程如下：

```
Agent 会话启动
  │
  ├─ resolveSandboxRuntimeStatus() → 判断是否启用沙箱
  │
  ├─ resolveSandboxConfigForAgent() → 解析沙箱配置
  │
  ├─ ensureSandboxWorkspaceLayout() → 创建工作区目录
  │    ├─ agentWorkspaceDir（Agent 本地工作区）
  │    ├─ sandboxWorkspaceDir（沙箱工作区副本）
  │    └─ skillsWorkspaceDir（技能工作区）
  │
  ├─ requireSandboxBackendFactory() → 获取后端工厂
  │
  ├─ backendFactory() → 创建后端实例（Docker 容器 / SSH 连接）
  │
  ├─ createSandboxFsBridge() → 创建文件系统桥接
  │
  └─ Agent 循环开始（工具调用受 tool-policy 保护）
```

在整个过程中，沙箱对 Agent 的主要约束体现在三个方面：
1. **执行环境隔离**：代码在容器或远程服务器中运行
2. **工具权限控制**：只有策略允许的工具才能被调用
3. **文件系统访问**：通过 `workspaceAccess`（`"none" | "ro" | "rw"`）控制对工作区的读写权限

这种多层安全设计确保了即使 LLM 生成的工具调用有误，也不会对宿主机造成实质性损害。