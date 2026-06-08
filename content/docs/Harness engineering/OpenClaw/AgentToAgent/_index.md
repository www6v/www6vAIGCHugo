# OpenClaw Agent-to-Agent：原理、实现机制、应用配置与 Memory 共享

> **本文源码版本**：OpenClaw 2026.4.21（f788c88）
> **关键源码文件**：`sessions-helpers-JB7DCYqu.js`、`openclaw-tools-CxKgYaee.js`、`resolve-route-BXujw34s.js`、`session-key-CLMDuI6A.js`、`agent-scope-BgFMM1Pq.js`、`runtime-schema-CrEOE13h.js`

---

## 一、概述

### 1.1 什么是 Agent-to-Agent（A2A）

Agent-to-Agent（简称 A2A）是 OpenClaw 多 Agent 架构中最核心的协作机制。它允许**运行在同一 Gateway 下的多个独立 Agent** 相互通信、协作完成复杂任务。

在传统单 Agent 模式下，一个 Gateway 只服务一个 AI 大脑，所有对话都局限在同一个 Session 内。而多 Agent 架构打破了这个边界——每个 Agent 拥有独立的 Workspace、模型配置、工具集和会话历史，同时通过 A2A 机制可以横向交互。

### 1.2 多 Agent 架构回顾

OpenClaw 的多 Agent 架构基于三个核心概念：

| 概念 | 说明 |
|------|------|
| **Agent** | 一个完整的隔离大脑，包含 Workspace + State Directory + Session Store + Auth Profiles |
| **Binding** | 路由规则，将 (channel, accountId, peer) 映射到某个 agentId |
| **SessionKey** | 会话的唯一标识，格式为 `agent:<agentId>:<channel>:<peerKind>:<peerId>` |

### 1.3 A2A 的典型场景

- **协作**：客服 Agent 遇到技术问题，转发给技术专家 Agent
- **接力**：翻译 Agent 完成翻译后，交给排版 Agent 格式化
- **问答**：分析 Agent 向知识 Agent 查询信息，综合后回复用户
- **通知**：监控 Agent 发现异常，通知运维 Agent 处理

### 1.4 本文覆盖范围

本文从源码层面深入分析 OpenClaw 的 A2A 实现机制，包括策略控制、会话可见性、`sessions_send` 工具、Ping-Pong 多轮对话、跨 Agent 记忆搜索，并提供完整的应用配置示例。

---

## 二、A2A 的架构基础

### 2.1 核心概念

**Agent 的完整组成**：

每个 Agent 是一个完全独立的"大脑"，由以下四个部分组成：

1. **Workspace**（`~/.openclaw/workspace-<agentId>`）：存放 `AGENTS.md`、`SOUL.md`、`USER.md`、`TOOLS.md`、Skills 等文件
2. **State Directory**（`~/.openclaw/agents/<agentId>/agent`）：存放 `auth-profiles.json`、模型注册表、配置
3. **Session Store**（`~/.openclaw/agents/<agentId>/sessions`）：聊天记录、路由状态、会话历史
4. **Auth Profiles**：每个 Agent 独立的 OAuth/Token 凭证

源码中，Agent 的 State Directory 由 `resolveAgentDir()` 函数确定（`agent-scope-BgFMM1Pq.js`）：

```javascript
function resolveAgentDir(cfg, agentId, env = process.env) {
    const id = normalizeAgentId(agentId);
    const configured = resolveAgentConfig(cfg, id)?.agentDir?.trim();
    if (configured) return resolveUserPath(configured, env);
    const root = resolveStateDir(env);
    return path.join(root, "agents", id, "agent");
}
```

### 2.2 三层控制模型

A2A 的核心设计哲学是**"隔离是默认，协作需显式"**。整个系统采用三层递进式控制：

```
入站消息 → [路由层] → [可见性层] → [策略层] → 执行
           Binding    Visibility   A2A Policy
```

每一层都可以阻止跨 Agent 的访问，只有三层全部通过，通信才能完成。

### 2.3 关键源码模块

| 源码文件 | 职责 |
|----------|------|
| `sessions-helpers-JB7DCYqu.js` | `createSessionVisibilityGuard()` + `createAgentToAgentPolicy()` |
| `openclaw-tools-CxKgYaee.js` | `createSessionsSendTool()` + `runSessionsSendA2AFlow()` |
| `resolve-route-BXujw34s.js` | `resolveAgentRoute()` — 入站消息路由引擎 |
| `session-key-CLMDuI6A.js` | `buildAgentPeerSessionKey()` + `parseAgentSessionKey()` |
| `agent-scope-BgFMM1Pq.js` | `resolveAgentConfig()` + `resolveSessionAgentId()` |
| `runtime-schema-CrEOE13h.js` | 完整的配置 Schema 定义 |

### 2.4 Agent 注册与管理

通过 CLI 命令管理多 Agent：

```bash
# 创建新 Agent
openclaw agents add coding
openclaw agents add social

# 查看已配置的 Agent 及路由绑定
openclaw agents list --bindings

# 设置 Agent 身份
openclaw agents set-identity --name "代码助手" --emoji "💻" coding

# 添加路由绑定
openclaw agents bind coding --channel telegram --account work

# 删除 Agent
openclaw agents delete social
```

源码中（`agents.config-C_JRHNX6.js`），Agent 配置的增删操作：

```javascript
function applyAgentConfig(cfg, params) {
    const agentId = normalizeAgentId(params.agentId);
    const list = listAgentEntries(cfg);
    const index = findAgentEntryIndex(list, agentId);
    const base = index >= 0 ? list[index] : { id: agentId };
    const nextEntry = {
        ...base,
        ...name ? { name } : {},
        ...params.workspace ? { workspace: params.workspace } : {},
        ...params.model ? { model: params.model } : {},
        ...mergedIdentity ? { identity: mergedIdentity } : {}
    };
    // ... 更新配置列表
}
```

---

## 三、A2A 策略控制机制

### 3.1 总开关：`tools.agentToAgent.enabled`

A2A 功能默认**关闭**。必须在 `openclaw.json` 中显式启用：

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
    },
  },
}
```

源码 `createAgentToAgentPolicy()`（`sessions-helpers-JB7DCYqu.js`）中，`enabled` 字段直接决定策略是否生效：

```javascript
function createAgentToAgentPolicy(cfg) {
    const routingA2A = cfg.tools?.agentToAgent;
    const enabled = routingA2A?.enabled === true;  // 必须显式为 true
    // ...
}
```

### 3.2 Allowlist 机制

启用后，还需要配置哪些 Agent 之间可以通信：

```json5
{
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["chat", "expert", "family"],  // 允许这些 Agent 互相通信
    },
  },
}
```

`matchesAllow()` 函数实现了**通配符匹配**，将 `*` 转换为正则的 `.*`：

```javascript
const matchesAllow = (agentId) => {
    if (allowPatterns.length === 0) return true;  // 空列表 = 全部允许
    return allowPatterns.some((pattern) => {
        const raw = normalizeOptionalString(typeof pattern === "string" ? pattern : String(pattern ?? "")) ?? "";
        if (!raw) return false;
        if (raw === "*") return true;  // 通配符匹配所有
        if (!raw.includes("*")) return raw === agentId;  // 精确匹配
        // 将 * 转为 .* 进行正则匹配
        const escaped = raw.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
        return new RegExp(`^${escaped.replaceAll("\\*", ".*")}$`, "i").test(agentId);
    });
};
```

**双重校验**是关键：`isAllowed()` 要求**请求方和目标方都在 allowlist 中**：

```javascript
const isAllowed = (requesterAgentId, targetAgentId) => {
    if (requesterAgentId === targetAgentId) return true;  // 同 Agent 直接放行
    if (!enabled) return false;
    return matchesAllow(requesterAgentId) && matchesAllow(targetAgentId);  // 双方都要匹配
};
```

### 3.3 同 Agent 豁免

同一个 Agent ID 下的不同 Session 之间访问**不受 A2A 策略限制**——这是通过 `requesterAgentId === targetAgentId` 的判断直接放行的。

---

## 四、会话可见性控制

### 4.1 四级可见性

`tools.sessions.visibility` 控制 Agent 能看到哪些会话：

| 级别 | 范围 | 默认 |
|------|------|------|
| `self` | 仅当前会话 | 否 |
| `tree` | 当前会话 + 其派生的 subagent | **是** |
| `agent` | 同一 Agent ID 下的所有会话 | 否 |
| `all` | 所有会话（跨 Agent 仍需 A2A Policy） | 否 |

### 4.2 可见性守卫

`createSessionVisibilityGuard()` 函数返回一个守卫对象，包含 `check()` 方法：

```javascript
async function createSessionVisibilityGuard(params) {
    const requesterAgentId = resolveAgentIdFromSessionKey(params.requesterSessionKey);
    const spawnedKeys = params.visibility === "tree"
        ? await listSpawnedSessionKeys({ requesterSessionKey: params.requesterSessionKey })
        : null;

    const check = (targetSessionKey) => {
        const targetAgentId = resolveAgentIdFromSessionKey(targetSessionKey);

        // 跨 Agent 检查
        if (targetAgentId !== requesterAgentId) {
            if (params.visibility !== "all")
                return { allowed: false, status: "forbidden", error: crossVisibilityMessage(params.action) };
            if (!params.a2aPolicy.enabled)
                return { allowed: false, status: "forbidden", error: a2aDisabledMessage(params.action) };
            if (!params.a2aPolicy.isAllowed(requesterAgentId, targetAgentId))
                return { allowed: false, status: "forbidden", error: a2aDeniedMessage(params.action) };
            return { allowed: true };
        }

        // 同 Agent 内部检查
        if (params.visibility === "self" && targetSessionKey !== params.requesterSessionKey)
            return { allowed: false, status: "forbidden", error: selfVisibilityMessage(params.action) };
        if (params.visibility === "tree" && targetSessionKey !== params.requesterSessionKey && !spawnedKeys?.has(targetSessionKey))
            return { allowed: false, status: "forbidden", error: treeVisibilityMessage(params.action) };
        return { allowed: true };
    };
    return { check };
}
```

**双层校验流程**：

1. **第一层**：判断 `targetAgentId !== requesterAgentId` → 跨 Agent
2. **第二层**（跨 Agent 时）：必须同时满足三个条件：
   - `visibility === "all"`
   - `a2aPolicy.enabled === true`
   - `a2aPolicy.isAllowed(requesterAgentId, targetAgentId) === true`

### 4.3 Sandbox 下的可见性收缩

沙箱环境中的 Agent 可见性被强制收缩：

```javascript
function resolveSandboxedSessionToolContext(params) {
    // 沙箱中 visibility 强制为 spawned
    const visibility = params.sandboxed === true && cfg.agents?.defaults?.sandbox?.sessionToolsVisibility === "spawned"
        ? "spawned"
        : resolveSessionToolsVisibility(params.cfg);
    return {
        // ...
        restrictToSpawned: params.sandboxed === true && visibility === "spawned"
    };
}
```

沙箱 Agent 只能访问自己派生的 subagent 会话，**无法跨 Agent 访问**。

### 4.4 各工具的可见性应用

以下工具统一受 visibility + a2aPolicy 控制：

- `sessions_list` — 列出会话
- `sessions_history` — 查看会话历史
- `sessions_send` — 向会话发送消息
- `session_status` — 查看会话状态

---

## 五、`sessions_send` 工具 — A2A 通信的核心通道

### 5.1 工具定义

`createSessionsSendTool()` 创建了 Agent 可用的 `sessions_send` 工具：

```javascript
const SessionsSendToolSchema = Type.Object({
    sessionKey: Type.Optional(Type.String()),
    label: Type.Optional(Type.String({ minLength: 1, maxLength: 512 })),
    agentId: Type.Optional(Type.String({ minLength: 1, maxLength: 64 })),
    message: Type.String(),
    timeoutSeconds: Type.Optional(Type.Number({ minimum: 0 }))
});
```

| 参数 | 说明 |
|------|------|
| `sessionKey` | 目标会话的完整 key |
| `label` | 目标会话的标签（如 "coding-session"） |
| `agentId` | 配合 label 使用，指定目标 Agent |
| `message` | 要发送的消息内容 |
| `timeoutSeconds` | 等待超时时间，0 表示 fire-and-forget |

### 5.2 完整执行流程

```
sessions_send("agent:expert:main", "请帮我分析这段代码的性能问题")
    │
    ├─ ① resolveSessionReference()        // 解析 sessionKey
    ├─ ② createSessionVisibilityGuard().check()  // 可见性 + A2A 策略检查
    ├─ ③ readLatestAssistantReplySnapshot()       // 读取基线回复
    ├─ ④ buildAgentToAgentMessageContext()        // 构建跨 Agent 上下文
    ├─ ⑤ startAgentRun()                         // 调用 Gateway agent 方法
    ├─ ⑥ waitForAgentRunAndReadUpdatedAssistantReply()  // 等待完成
    └─ ⑦ startA2AFlow()                         // 触发 Ping-Pong + Announce
```

核心实现（`openclaw-tools-CxKgYaee.js`）：

```javascript
const sendParams = {
    message,
    sessionKey: resolvedKey,
    idempotencyKey,
    deliver: false,                    // 不直接投递到 channel
    channel: INTERNAL_MESSAGE_CHANNEL, // 使用内部消息通道
    lane: resolveNestedAgentLaneForSession(resolvedKey),
    extraSystemPrompt: agentMessageContext,  // A2A 上下文
    inputProvenance: {               // 输入来源标记
        kind: "inter_session",
        sourceSessionKey: opts?.agentSessionKey,
        sourceChannel: opts?.agentChannel,
        sourceTool: "sessions_send"
    }
};

const start = await startAgentRun({
    callGateway: gatewayCall,
    runId,
    sendParams,
    sessionKey: displayKey
});
```

**输入来源标记**让目标 Agent 可以识别消息来源：

```json
{
  "kind": "inter_session",
  "sourceSessionKey": "agent:chat:ddingtalk:direct:15921375071",
  "sourceChannel": "ddingtalk",
  "sourceTool": "sessions_send"
}
```

### 5.3 两种运行模式

| 模式 | timeoutSeconds | 行为 |
|------|----------------|------|
| 等待回复 | > 0 | 启动目标 Agent，等待回复后返回 |
| Fire-and-Forget | 0 | 启动目标 Agent 后立即返回，不等待 |

---

## 六、Agent 间 Ping-Pong 多轮对话

### 6.1 配置

```json5
{
  session: {
    agentToAgent: {
      maxPingPongTurns: 5,  // 最多 5 轮，上限 5，设为 0 禁用
    },
  },
}
```

源码中（`openclaw-tools-CxKgYaee.js`）：

```javascript
const DEFAULT_PING_PONG_TURNS = 5;
const MAX_PING_PONG_TURNS = 5;

function resolvePingPongTurns(cfg) {
    const raw = cfg?.session?.agentToAgent?.maxPingPongTurns;
    const fallback = DEFAULT_PING_PONG_TURNS;
    if (typeof raw !== "number" || !Number.isFinite(raw)) return fallback;
    return Math.max(0, Math.min(MAX_PING_PONG_TURNS, Math.floor(raw)));
}
```

### 6.2 核心流程

`runSessionsSendA2AFlow()` 实现了完整的 Ping-Pong 对话：

```javascript
async function runSessionsSendA2AFlow(params) {
    // 角色交替循环
    for (let turn = 1; turn <= params.maxPingPongTurns; turn += 1) {
        const currentRole = currentSessionKey === params.requesterSessionKey ? "requester" : "target";
        const replyPrompt = buildAgentToAgentReplyContext({
            currentRole,
            turn,
            maxTurns: params.maxPingPongTurns,
            // ...
        });
        const replyText = await runAgentStep({
            sessionKey: currentSessionKey,
            message: incomingMessage,
            extraSystemPrompt: replyPrompt,
            sourceSessionKey: nextSessionKey,
            sourceTool: "sessions_send"
        });
        if (!replyText || isReplySkip(replyText)) break;  // 终止条件
        latestReply = replyText;
        incomingMessage = replyText;
        // 角色 swap
        const swap = currentSessionKey;
        currentSessionKey = nextSessionKey;
        nextSessionKey = swap;
    }
}
```

**终止条件**有三个：
1. 回复包含 `REPLY_SKIP_TOKEN`（`AFTER_THIS_REPLY_AGENT_TO_AGENT_CONVERSATION_IS_OVER`）
2. 达到最大轮次（`maxPingPongTurns`）
3. 超时或异常

### 6.3 Announce 机制

Ping-Pong 结束后，通过 Announce 机制将最终回复投递到目标 channel：

```javascript
const announcePrompt = buildAgentToAgentAnnounceContext({
    requesterSessionKey: params.requesterSessionKey,
    targetSessionKey: params.displayKey,
    originalMessage: params.message,
    roundOneReply: primaryReply,
    latestReply
});

const announceReply = await runAgentStep({
    sessionKey: params.targetSessionKey,
    message: "Agent-to-agent announce step.",
    extraSystemPrompt: announcePrompt,
    sourceTool: "sessions_send"
});

// 投递到目标 channel
await sessionsSendA2ADeps.callGateway({
    method: "send",
    params: {
        to: announceTarget.to,
        message: announceReply.trim(),
        channel: announceTarget.channel,
        accountId: announceTarget.accountId,
        threadId: announceTarget.threadId,
    }
});
```

Announce 也支持跳过：如果回复包含 `ANNOUNCE_SKIP_TOKEN`，则不投递。

---

## 七、Label 解析与跨 Agent 会话发现

### 7.1 Label 模式

`sessions_send` 支持通过 `label` 而非完整 `sessionKey` 来定位目标会话：

```json
{
  "sessionKey": "",
  "label": "coding-task",
  "agentId": "expert",
  "message": "请分析这段代码"
}
```

内部通过 `sessions.resolve` Gateway 方法解析：

```javascript
const resolvedKey = normalizeOptionalString((await gatewayCall({
    method: "sessions.resolve",
    params: { label: labelParam, agentId: requestedAgentId },
    timeoutMs: 10000
}))?.key) ?? "";
```

### 7.2 权限控制

跨 Agent label 解析同样受 A2A 策略和沙箱限制：

```javascript
if (restrictToSpawned && requestedAgentId && requestedAgentId !== requesterAgentId)
    return jsonResult({ status: "forbidden", error: "Sandboxed sessions_send label lookup is limited to this agent" });

if (requesterAgentId && requestedAgentId && requestedAgentId !== requesterAgentId) {
    if (!a2aPolicy.enabled)
        return jsonResult({ status: "forbidden", error: "Agent-to-agent messaging is disabled." });
    if (!a2aPolicy.isAllowed(requesterAgentId, requestedAgentId))
        return jsonResult({ status: "forbidden", error: "Agent-to-agent messaging denied by tools.agentToAgent.allow." });
}
```

---

## 八、Memory 共享机制

### 8.1 记忆引擎架构

OpenClaw 支持多种记忆引擎：

| 引擎 | 说明 | 后端 |
|------|------|------|
| **Builtin** | 文件记忆，读取 `MEMORY.md` + `memory/*.md` | 文件系统 |
| **QMD** | 向量语义记忆（Quantum Memory Database） | LanceDB |
| **Honcho** | 轻量级记忆管理 | 文件系统 |

配置选择记忆引擎：

```json5
{
  memory: {
    backend: "qmd",  // 或 "builtin"
  },
}
```

### 8.2 Memory 隔离默认策略

默认情况下，**每个 Agent 的记忆完全隔离**：

- 各自独立的 `MEMORY.md` 和 `memory/*.md` 文件
- `agentDir` 下的 session 数据完全隔离
- QMD 索引各自独立

### 8.3 QMD 跨 Agent 记忆搜索：`extraCollections`

这是 OpenClaw 实现跨 Agent 记忆共享的核心机制——**按需引入，非全局共享**。

```json5
{
  agents: {
    defaults: {
      memorySearch: {
        qmd: {
          extraCollections: [
            { path: "~/agents/family/sessions", name: "family-sessions" }
          ]
        }
      }
    },
    list: [
      {
        id: "main",
        memorySearch: {
          qmd: { extraCollections: [{ path: "notes" }] }
        }
      },
      { id: "family", workspace: "~/workspaces/family" }
    ]
  },
  memory: { backend: "qmd", qmd: { includeDefaultMemory: false } }
}
```

**关键设计**：

- 通过 `extraCollections` **显式挂载**其他 Agent 的 session 集合
- 不是全局共享，而是**定向引入**
- 只读搜索：只能搜索，不能修改其他 Agent 的记忆
- Collection name 在 path 外部时必须显式指定，避免意外暴露

源码描述（`runtime-schema-CrEOE13h.js`）：

> "Use this when you need directional transcript search across agents; add collections here to scope QMD recalls without creating a shared global transcript namespace."

### 8.4 QMD 索引机制

QMD 引擎负责将 session 记录索引为向量，支持语义搜索：

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `memory.qmd.indexSessions` | 是否索引 session 记录 | false（实验性） |
| `memory.qmd.sessionExportDir` | session 导出目录 | `~/.openclaw/state/...` |
| `memory.qmd.sessionRetention` | session 保留天数 | - |
| `memory.qmd.updateInterval` | 索引更新间隔 | 5m |
| `memory.qmd.updateOnStartup` | 启动时是否更新 | true |
| `memory.qmd.waitForBootSync` | 阻塞启动等待同步完成 | false |

### 8.5 共享记忆的四级层级

| 层级 | 描述 | 所需配置 |
|------|------|----------|
| **Level 0** | 完全隔离 | 默认，无需配置 |
| **Level 1** | `extraCollections` 只读搜索 | `agents.list[].memorySearch.qmd.extraCollections` |
| **Level 2** | A2A `sessions_send` 实时通信 | `tools.agentToAgent.enabled` + `allow` + `visibility=all` |
| **Level 3** | QMD 索引 + A2A 实时通信（完整协作） | Level 1 + Level 2 组合 |

### 8.6 安全模型

- `extraCollections` 必须**显式配置**，不会自动泄露
- Workspace 内的 path 自动保持 Agent 隔离
- Collection name 在 path 外部时必须显式指定
- 只读搜索：即使挂载了其他 Agent 的 collection，也只能搜索，不能修改

---

## 九、应用配置实践

### 9.1 场景一：客服 Agent + 技术专家 Agent 协作

```json5
{
  agents: {
    list: [
      { id: "support", name: "客服助手", workspace: "~/.openclaw/workspace-support", model: "anthropic/claude-sonnet-4-6" },
      { id: "expert", name: "技术专家", workspace: "~/.openclaw/workspace-expert", model: "anthropic/claude-opus-4-6" },
    ],
  },
  bindings: [
    { agentId: "support", match: { channel: "whatsapp" } },
  ],
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["support", "expert"],
    },
    sessions: { visibility: "all" },
  },
  session: {
    agentToAgent: { maxPingPongTurns: 3 },
  },
}
```

客服 Agent 遇到技术问题时：

```javascript
// 在 support Agent 的 SOUL.md 中设定规则
// "遇到技术问题，通过 sessions_send 转发给 expert Agent"
await sessions_send({
    sessionKey: "agent:expert:whatsapp:group:tech-group",
    message: "用户遇到了以下技术问题：...",
    timeoutSeconds: 30
});
```

### 9.2 场景二：多 Agent Ping-Pong 问答接力

```json5
{
  agents: {
    list: [
      { id: "analyst", name: "数据分析师", model: "anthropic/claude-sonnet-4-6" },
      { id: "critic", name: "审查员", model: "anthropic/claude-opus-4-6" },
    ],
  },
  tools: {
    agentToAgent: { enabled: true, allow: ["analyst", "critic"] },
    sessions: { visibility: "all" },
  },
  session: { agentToAgent: { maxPingPongTurns: 5 } },
}
```

Analyst 分析 → Critic 审查 → Analyst 修正 → Critic 确认，最多 5 轮。

### 9.3 场景三：家庭 Agent 共享记忆

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main",
        memorySearch: { qmd: { extraCollections: [
          { path: "~/.openclaw/agents/family/sessions", name: "family-sessions" }
        ]}}},
      { id: "family", workspace: "~/.openclaw/workspace-family" },
    ],
  },
  memory: { backend: "qmd" },
}
```

主 Agent 可以搜索家庭 Agent 的历史对话，但家庭 Agent 不能访问主 Agent 的记忆（单向引入）。

### 9.4 场景四：沙箱 Agent 的受限 A2A

```json5
{
  agents: {
    list: [
      {
        id: "sandbox-agent",
        sandbox: { mode: "all", scope: "agent" },
        tools: {
          allow: ["exec", "read", "sessions_list", "sessions_send"],
          deny: ["write", "edit", "browser", "cron"],
        },
      },
    ],
  },
}
```

沙箱 Agent 中 `visibility` 强制为 `spawned`，只能和自己派生的 subagent 通信。

### 9.5 场景五：全链路协作

完整配置模板，结合 A2A + Memory 共享 + 跨 Agent 搜索：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-sonnet-4-6",
      memorySearch: { qmd: { extraCollections: [] } },
    },
    list: [
      {
        id: "main",
        name: "主助手",
        workspace: "~/.openclaw/workspace-main",
        memorySearch: {
          qmd: { extraCollections: [
            { path: "~/.openclaw/agents/coding/sessions", name: "coding-sessions" },
            { path: "~/.openclaw/agents/family/sessions", name: "family-sessions" },
          ]},
        },
      },
      {
        id: "coding",
        name: "代码助手",
        workspace: "~/.openclaw/workspace-coding",
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "family",
        name: "家庭助手",
        workspace: "~/.openclaw/workspace-family",
      },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "ddingtalk" } },
    { agentId: "coding", match: { channel: "telegram", account: "dev" } },
    { agentId: "family", match: { channel: "whatsapp", peer: { kind: "group", id: "family-group" } } },
  ],
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["main", "coding", "family"],
    },
    sessions: { visibility: "all" },
  },
  memory: { backend: "qmd", qmd: { includeDefaultMemory: true, indexSessions: true } },
}
```

### 9.6 安全最佳实践

1. **最小权限原则**：按需开启 A2A，不要全局 `allow: ["*"]`
2. **白名单优于全局开启**：明确列出需要的 Agent ID
3. **沙箱隔离不可绕过**：沙箱中的 Agent 即使配置了 A2A 也无法跨 Agent 访问
4. **单向引入记忆**：`extraCollections` 是只读的，不会泄露自身记忆
5. **定期审计**：使用 `openclaw agents list --bindings` 检查 Agent 路由配置

---

## 十、源码深度剖析

### 10.1 关键函数调用链路

```
sessions_send 调用
    │
    ├─ resolveSessionToolContext()           // 解析会话上下文
    │   └─ resolveSandboxedSessionToolContext()  // 沙箱上下文
    │
    ├─ createAgentToAgentPolicy(cfg)         // 构建 A2A 策略
    │   ├─ enabled = cfg.tools.agentToAgent.enabled === true
    │   └─ matchesAllow(pattern)             // 通配符匹配
    │
    ├─ createSessionVisibilityGuard()        // 构建可见性守卫
    │   └─ check(targetSessionKey)           // 双层校验
    │
    ├─ resolveSessionReference()             // 解析目标 session
    │   └─ Gateway sessions.resolve          // 解析 label/sessionKey
    │
    ├─ startAgentRun()                       // 启动目标 Agent
    │   └─ Gateway agent 方法                // deliver: false, channel: internal
    │
    ├─ waitForAgentRunAndReadUpdatedAssistantReply()  // 等待完成
    │
    └─ runSessionsSendA2AFlow()              // Ping-Pong + Announce
        ├─ for (turn 1..maxPingPongTurns)   // 角色交替
        │   └─ runAgentStep()               // 当前 Agent 回复
        └─ Gateway send                      // Announce 投递
```

### 10.2 SessionKey 命名空间体系

SessionKey 是 Agent 间通信的**地址系统**：

```
agent:<agentId>:<rest>
```

| 格式 | 说明 | 示例 |
|------|------|------|
| `agent:<id>:main` | Agent 主会话 | `agent:chat:main` |
| `agent:<id>:<channel>:direct:<peerId>` | DM | `agent:chat:ddingtalk:direct:15921375071` |
| `agent:<id>:<channel>:group:<peerId>` | 群组 | `agent:chat:whatsapp:group:family` |
| `...:thread:<threadId>` | 线程扩展 | `agent:chat:discord:group:123:thread:456` |
| `agent:<id>:cron:<job>:run:<id>` | Cron 运行 | `agent:main:cron:backup:run:abc` |
| `agent:<id>:subagent:...` | Subagent | `agent:main:subagent:coding-1` |

源码（`session-key-CLMDuI6A.js`）：

```javascript
function buildAgentPeerSessionKey(params) {
    const peerKind = params.peerKind ?? "direct";
    if (peerKind === "direct") {
        // DM 格式：agent:<id>:<channel>:direct:<peerId>
        return `agent:${normalizeAgentId(params.agentId)}:${channel}:direct:${peerId}`;
    }
    // Group 格式：agent:<id>:<channel>:group:<peerId>
    return `agent:${normalizeAgentId(params.agentId)}:${channel}:${peerKind}:${peerId}`;
}
```

### 10.3 缓存策略

OpenClaw 使用多层缓存优化路由性能：

```javascript
// Agent 查找缓存
const agentLookupCacheByCfg = new WeakMap();

// Binding 评估缓存
const evaluatedBindingsCacheByCfg = new WeakMap();
const MAX_EVALUATED_BINDINGS_CACHE_KEYS = 2000;

// 路由解析缓存
const resolvedRouteCacheByCfg = new WeakMap();
const MAX_RESOLVED_ROUTE_CACHE_KEYS = 4000;
```

使用 `WeakMap` 以 config 对象引用为 key，config 对象被 GC 时缓存自动清理。

### 10.4 错误处理

| 错误类型 | 触发条件 | 返回 |
|----------|----------|------|
| `forbidden` (visibility) | `visibility !== "all"` | "Session visibility is restricted" |
| `forbidden` (a2a disabled) | `a2aPolicy.enabled === false` | "Agent-to-agent messaging is disabled" |
| `forbidden` (a2a denied) | `a2aPolicy.isAllowed() === false` | "Agent-to-agent messaging denied by allowlist" |
| `error` (not found) | sessionKey/label 不存在 | "No session found with label: ..." |
| `timeout` | 超过 timeoutSeconds | "Agent run timed out" |

---

## 十一、总结与展望

### 11.1 A2A 设计哲学

OpenClaw 的 A2A 机制遵循一个核心原则：**隔离是默认，协作需显式**。

三层控制模型（路由 → 可见性 → 策略）确保了：
- 默认情况下 Agent 之间完全隔离
- 协作需要逐层显式开启
- 每一层都有独立的配置和回退机制

### 11.2 三层安全模型总结

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: Binding 路由层                              │
│  控制入站消息路由到哪个 Agent                          │
│  配置: bindings[].match                               │
├─────────────────────────────────────────────────────┤
│  Layer 2: Visibility 可见性层                         │
│  控制 Agent 能看到哪些会话                             │
│  配置: tools.sessions.visibility (self/tree/agent/all)│
├─────────────────────────────────────────────────────┤
│  Layer 3: A2A Policy 策略层                           │
│  控制 Agent 能向哪些目标发送消息                        │
│  配置: tools.agentToAgent.enabled + allow             │
└─────────────────────────────────────────────────────┘
```

### 11.3 Memory 共享的渐进式路径

从 Level 0（完全隔离）到 Level 3（完整协作），用户可以按需逐步提升 Agent 间的协作能力，每一步都是**显式配置、可控回退**的。

### 11.4 未来展望

- **跨 Gateway A2A**：当前 A2A 限于同一 Gateway 内的 Agent，未来可能支持跨实例通信
- **A2A 协议标准化**：与 Anthropic 的 Agent-to-Agent Protocol 等外部标准对齐
- **更细粒度的权限控制**：按工具、按数据类型进行更精细的 A2A 授权
- **Memory 双向共享**：当前 extraCollections 是单向只读，未来可能支持双向可写
