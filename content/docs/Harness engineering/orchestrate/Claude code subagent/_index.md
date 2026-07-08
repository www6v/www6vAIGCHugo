# Claude Code Subagent 原理与设计实现深度解析

> "Fork yourself when the intermediate tool output isn't worth keeping in your context." — Claude Code source code comments, `src/tools/AgentTool/prompt.ts`

作为 AI 编码代理领域的标杆产品，Claude Code（Anthropic）的 Subagent 系统代表了一种对 **上下文隔离、并行执行和专业化分工** 的精妙工程实践。本文深入源码与官方文档，逐层拆解其设计哲学与实现细节。

---

## 一、引言：为什么需要 Subagent？

大语言模型面临一个根本矛盾：**上下文窗口是有限的，而代码库是无限的**。

当代理需要同时完成以下工作时，单一会话必然陷入困境：
- 搜索某个 API 调用模式出现在哪些文件中（探索型）
- 修改 3 个相关文件（实现型）
- 阅读 2000 行日志分析错误原因（研究型）

如果这些工作都在同一个会话中进行，探索产生的大量中间结果会迅速污染上下文窗口，挤占实现所需的上下文空间。Anthropic 的解决方案是 **Subagent** —— 将探索、研究和实现委派到独立的上下文窗口中，主会话只接收精炼的摘要。

这与 Designing Data-Intensive Applications（DDIA）中 "分离关注点" 的设计原则一致：将上下文敏感的工作（探索）与上下文无关的工作（实现）解耦，使系统各部分可以独立扩展。

---

## 二、系统总览：四层 Agent 架构

Claude Code 并非只有一个 "子代理" 概念，而是构建了 **四层递进的代理体系**：

| 层次 | 实现方式 | 通信模式 | 适用场景 |
|------|---------|---------|---------|
| **主会话（Lead）** | Claude Code REPL | 用户交互 | 日常编码、协调工作 |
| **Subagent** | `Agent` tool，同一进程内独立 API 调用 | 单向：结果回传父会话 | 探索、研究、代码审查 |
| **Fork（派生）** | 隐式 fork，共享父会话 prompt cache | 异步通知，不拉取中间结果 | "中间输出不值得保留"的任务 |
| **Agent Teams** | 独立 Claude Code 进程 + tmux/进程内 | 双向：共享 TaskList + Mailbox | 需要相互协作的并行工作 |

```
┌─────────────────────────────────────────────────────────────┐
│                     主会话（Lead）                            │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  Subagent    │  │   Fork       │  │  Agent Teammate   │  │
│  │  (独立API)   │  │  (共享cache) │  │  (独立进程)       │  │
│  │  结果回传    │  │  异步通知    │  │  双向通信         │  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
│                                                             │
│  Agent tool ──────────────────────────────► 调度层            │
│  spawnMultiAgent.ts ─────────────────────► 团队协调           │
└─────────────────────────────────────────────────────────────┘
```

**源码参考**：
- Agent tool 核心：[`src/tools/AgentTool/AgentTool.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/AgentTool.tsx)
- Fork 机制：[`src/tools/AgentTool/forkSubagent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/forkSubagent.ts)
- 团队调度：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts)
- 官方文档：[Create custom subagents](https://code.claude.com/docs/en/sub-agents.md)、[Agent teams](https://code.claude.com/docs/en/agent-teams.md)

---

## 三、Subagent 定义系统

### 3.1 定义来源与五级优先级

Subagent 并非只能手工创建，而是来自 **五个层次** 的定义源，按优先级从高到低排列：

| 优先级 | 来源 | 范围 | 存储位置 |
|--------|------|------|---------|
| 1（最高） | Managed settings | 组织级 | 管理设置目录（企业部署） |
| 2 | `--agents` CLI flag | 当前会话 | 内存（不持久化） |
| 3 | `.claude/agents/` | 当前项目 | Git 追踪，团队共享 |
| 4 | `~/.claude/agents/` | 用户全局 | 本地，跨项目复用 |
| 5（最低） | Plugin `agents/` | 插件范围 | 插件安装目录 |

**设计解读**：优先级从窄到宽排列 —— 最具体的定义覆盖最通用的定义。这与 CSS 的层叠规则、Linux 的配置加载顺序（`/etc` → `~/.config` → CLI flags）一脉相承。

**源码参考**：[`src/tools/AgentTool/loadAgentsDir.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) — 递归扫描代理目录，同名冲突时最近工作目录优先（v2.1.178+）。

### 3.2 定义格式：Markdown + YAML frontmatter

每个 Subagent 是一个 Markdown 文件，YAML frontmatter 定义元数据，正文即系统提示：

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
permissionMode: default
maxTurns: 50
skills: [api-conventions, error-handling-patterns]
mcpServers: [github, {playwright: {type: stdio, command: npx, args: ["-y", "@playwright/mcp@latest"]}}]
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
memory: project
effort: high
isolation: worktree
color: blue
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
For each issue found, explain the problem, show the current code,
and provide an improved version.
```

### 3.3 核心字段全解析

| 字段 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `name` | string | ✅ | 唯一标识符，小写字母+连字符 |
| `description` | string | ✅ | **Claude 决定何时委派的关键** |
| `tools` | string[] | 否 | 工具白名单，默认继承全部 |
| `disallowedTools` | string[] | 否 | 工具黑名单 |
| `model` | string | 否 | `sonnet`/`opus`/`haiku`/`fable`/完整ID/`inherit` |
| `permissionMode` | string | 否 | 6 种权限模式 |
| `maxTurns` | number | 否 | 最大代理轮次 |
| `skills` | string[] | 否 | 启动时预加载的技能内容 |
| `mcpServers` | array | 否 | MCP 服务器（内联或引用） |
| `hooks` | object | 否 | 生命周期钩子 |
| `memory` | string | 否 | `user`/`project`/`local` |
| `effort` | string | 否 | `low`/`medium`/`high`/`xhigh`/`max` |
| `isolation` | string | 否 | `worktree`（Git 隔离） |
| `color` | string | 否 | UI 显示颜色 |
| `initialPrompt` | string | 否 | 首条自动提交的用户消息 |

**关键设计点**：
- `description` 是唯一触发自动委派的条件，因此需要写得足够精确。建议加入 "Use proactively" 字样鼓励主动委派。
- `tools` 和 `disallowedTools` 可共存：先应用黑名单，再在剩余池中解析白名单。
- MCP 支持模式匹配：`mcp__github` 移除整个 GitHub MCP 服务器的所有工具。

**源码参考**：
- Zod Schema 定义：[`src/tools/AgentTool/loadAgentsDir.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) (`AgentJsonSchema`)
- 字段解析与验证：[`src/utils/markdownConfigLoader.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/markdownConfigLoader.ts)

### 3.4 加载与热重载机制

Claude Code 对代理目录使用文件系统 watcher：
- **热重载**：文件变更后几秒内自动生效，无需重启
- **例外**：新创建的 `agents/` 目录需要重启（watcher 只监视已存在的目录）
- **去重报告**：`/doctor` 命令可检测并报告同作用域内的重复代理名（v2.1.196+）

**源码参考**：[`src/tools/AgentTool/loadAgentsDir.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) — `loadMarkdownFilesForSubdir()` 递归扫描，`memoize` 缓存结果。

---

## 四、Subagent 执行引擎

### 4.1 AgentTool 核心架构

Agent tool 是 Claude Code 代理系统的 **单一调度入口**（`src/tools/AgentTool/AgentTool.tsx`），统一处理所有子代理类型：

```
Agent tool input
    ↓
┌─────────────────────────────────────────────────────────┐
│  AgentTool.tsx（调度器）                                  │
│                                                         │
│  1. 解析 subagent_type → loadAgentsDir() 查找定义        │
│  2. 过滤工具池 → resolveAgentTools()                     │
│  3. 构建系统提示 → buildEffectiveSystemPrompt()          │
│  4. 初始化 MCP → initializeAgentMcpServers()             │
│  5. 注册钩子 → registerFrontmatterHooks()                │
│  6. 创建任务状态 → createProgressTracker()               │
│  7. 执行 → runAgent.ts (query loop)                     │
│  8. 进度更新 → updateAgentProgress()                     │
│  9. 完成/失败 → completeAgentTask() / failAgentTask()    │
│  10. 清理 → disconnectMCP, clearHooks, releaseColor     │
└─────────────────────────────────────────────────────────┘
    ↓
Agent result (summary only)
```

**输入 Schema**（`src/tools/AgentTool/AgentTool.tsx`）：
```typescript
const baseInputSchema = z.object({
  description: z.string(),        // 3-5 字任务描述
  prompt: z.string(),             // 任务内容
  subagent_type: z.string().optional(),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  // 多代理参数
  name: z.string().optional(),
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),
  // 隔离
  isolation: z.enum(['worktree']).optional(),
  cwd: z.string().optional(),
});
```

**源码参考**：[`src/tools/AgentTool/AgentTool.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/AgentTool.tsx) — 完整工具定义，含 lazySchema 按需加载。

### 4.2 完整生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                     Subagent 生命周期                         │
│                                                              │
│  [spawn]                                                     │
│    │                                                         │
│    ├─ 1. resolveAgentTools() → 计算可用工具集                 │
│    │                                                         │
│    ├─ 2. initializeAgentMcpServers() → 连接代理专属 MCP       │
│    │                                                         │
│    ├─ 3. buildEffectiveSystemPrompt() → 渲染系统提示           │
│    │    ├─ 加载 CLAUDE.md（Explore/Plan 除外）                │
│    │    ├─ 加载代理 memory prompt（如启用）                   │
│    │    └─ 加载代理 skills 内容                               │
│    │                                                         │
│    ├─ 4. registerFrontmatterHooks() → 注册代理专属钩子        │
│    │                                                         │
│    ├─ 5. createAgentId() + createProgressTracker()            │
│    │                                                         │
│    ├─ 6. runAgent() → API 调用循环                            │
│    │    ├─ query() → Anthropic API                           │
│    │    ├─ handle tool calls                                 │
│    │    ├─ updateProgress() → 实时更新进度                    │
│    │    └─ 直到完成 / maxTurns / 中断                         │
│    │                                                         │
│    ├─ 7. completeAgentTask() / failAgentTask()               │
│    │                                                         │
│    └─ 8. cleanup()                                           │
│         ├─ disconnectMcpServers()                            │
│         ├─ clearSessionHooks()                               │
│         ├─ releaseAgentColor()                               │
│         └─ recordSidechainTranscript()                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**源码参考**：
- 执行入口：[`src/tools/AgentTool/runAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/runAgent.ts)
- 进度追踪：[`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx)

### 4.3 模型解析链

Subagent 的模型选择遵循 **四级优先级**（从高到低）：

```
CLAUDE_CODE_SUBAGENT_MODEL 环境变量
    ↓（未设置则继续）
调用时的 model 参数
    ↓（未指定则继续）
定义中的 model frontmatter
    ↓（未定义则继续）
父会话模型（inherit，默认行为）
```

**重要细节**：
- `inherit` 不是字符串常量，而是语义占位符：`resolveTeammateModel()` 函数将其替换为父会话的实际模型。
- v2.1.198+ 起，子代理还继承父会话的 **extended thinking** 配置（开→开，关→关），没有独立设置。
- 组织管理员可通过 `availableModels` 策略限制子代理可用的模型。

**源码参考**：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts) — `resolveTeammateModel()` 和 `getDefaultTeammateModel()`。

### 4.4 内置 Subagent

Claude Code 自带 5 个内置 Subagent（`src/tools/AgentTool/built-in/`）：

| 代理 | 模型 | 工具 | 用途 | 特殊行为 |
|------|------|------|------|---------|
| **Explore** | 继承（上限 Opus） | 只读：Read, Grep, Glob | 代码探索、搜索 | 跳过 CLAUDE.md 和 git status |
| **Plan** | 继承 | 只读 | Plan mode 研究 | 同上 |
| **General-purpose** | 继承 | 全部 | 复杂多步任务 | — |
| **statusline-setup** | Sonnet | 受限 | 状态栏配置 | — |
| **claude-code-guide** | Haiku | 受限 | Claude Code 功能问答 | — |

**Explore 的模型继承策略**（v2.1.198+）：
- 主会话在 Claude API 上 → Explore 继承模型，上限 Opus
- 主会话在第三方 API（Bedrock/Foundry）上 → Explore 直接继承
- 如果用户定义了同名 Explore 代理 → 覆盖内置，使用自定义 model

**源码参考**：
- 通用代理：[`src/tools/AgentTool/built-in/generalPurposeAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/built-in/generalPurposeAgent.ts)
- 注册入口：[`src/tools/AgentTool/builtInAgents.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/builtInAgents.ts)
- 模型继承：[`src/utils/model/agent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/model/agent.ts) — `getAgentModel()`

### 4.5 上下文注入：什么到达 Subagent？

这是理解 Subagent 设计的关键：**子代理不会继承父会话的完整上下文**。

| 项目 | Explore | Plan | 其他代理 |
|------|---------|------|---------|
| CLAUDE.md | ❌ 跳过 | ❌ 跳过 | ✅ 加载 |
| Git status | ❌ 跳过 | ❌ 跳过 | ✅ 加载 |
| MCP 服务器 | ✅ 继承 | ✅ 继承 | ✅ 继承（+ 代理专属） |
| Skills | ✅ 可用 | ✅ 可用 | ✅ 可用（+ 预加载） |
| 父会话对话历史 | ❌ 不携带 | ❌ 不携带 | ❌ 不携带 |
| 代理专属系统提示 | ✅ | ✅ | ✅ |
| 代理 memory | ✅（如配置） | ✅（如配置） | ✅（如配置） |

**设计哲学**：子代理是一个 **干净的 slate（干净的起点）**，只携带项目上下文（CLAUDE.md、MCP、skills）和专属系统提示。这确保了探索结果不会污染主会话上下文。

---

## 五、Fork 机制：隐式派生子代理

### 5.1 设计哲学

> "Fork yourself when the intermediate tool output isn't worth keeping in your context. The criterion is qualitative — 'will I need this output again' — not task size."
> — [`src/tools/AgentTool/prompt.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/prompt.ts)

Fork 是 Claude Code 中最精妙的设计之一。它的核心思想是：**如果中间结果不值得保留，就不要让它进入你的上下文窗口**。

```
普通 Subagent          Fork
──────────────        ──────────────
独立 prompt cache     共享父 prompt cache
独立系统提示           继承父系统提示（字节级精确）
独立工具池             继承父工具池
结果回到父会话          异步通知，不拉取中间结果
适合 "探索后总结"      适合 "执行后丢弃"
```

**关键优化**：Fork 子代理共享父会话的 prompt cache，这意味着 **fork 几乎不增加额外的 token 成本** —— API 请求前缀字节完全相同，只有最后的 directive 文本块不同。

### 5.2 实现原理

**源码参考**：[`src/tools/AgentTool/forkSubagent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/forkSubagent.ts)

#### 核心数据结构

```typescript
export const FORK_AGENT = {
  agentType: 'fork',
  whenToUse: 'Implicit fork — inherits full conversation context.',
  tools: ['*'],                    // 精确继承父工具池
  maxTurns: 200,
  model: 'inherit',                // 保持上下文窗口长度一致
  permissionMode: 'bubble',        // 权限提示冒泡到父终端
  source: 'built-in',
  getSystemPrompt: () => '',       // 未使用——直接传递父会话已渲染字节
} satisfies BuiltInAgentDefinition;
```

#### 消息构建算法（buildForkedMessages）

为了实现 prompt cache 最大化共享，Fork 的消息构建遵循严格的字节级精确原则：

```
步骤 1: 保留完整的父 assistant 消息（所有 tool_use 块、thinking、text）
         ↓
步骤 2: 为每个 tool_use 块生成完全相同的 placeholder result:
         "Fork started — processing in background"
         ↓
步骤 3: 构建单一 user 消息，末尾追加 per-child directive
         ↓
结果: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
       ↑ 所有 fork 子代理的 API 请求前缀完全一致，仅最后 directive 不同
```

**为什么这很重要？** Anthropic 的 prompt caching 按字节前缀匹配。如果所有 fork 子代理的前缀完全相同，只有最后一个文本块不同，那么除了最后一个 block，其余全部命中 cache —— 这意味着 fork 的成本接近于零。

### 5.3 防递归机制

Fork 子代理本身包含 Agent tool，理论上可以继续 fork。但源码通过检测对话历史中的 `<fork-boilerplate>` 标签来阻止递归 fork：

```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false;
    const content = m.message.content;
    if (!Array.isArray(content)) return false;
    return content.some(
      block => block.type === 'text' &&
               block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    );
  });
}
```

### 5.4 互斥关系

- **Fork ↔ Coordinator mode**：互斥。Coordinator 已有自己的委派模型。
- **Fork ↔ 非交互式会话**：不支持。`getIsNonInteractiveSession()` 返回 true 时禁用。

**源码参考**：[`src/tools/AgentTool/forkSubagent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/forkSubagent.ts) — `isForkSubagentEnabled()` 门控函数。

---

## 六、工具与权限控制

### 6.1 工具过滤链（resolveAgentTools）

Subagent 的工具池通过 **先减后加** 的方式计算：

```
父会话完整工具池
    ↓
应用 disallowedTools（移除黑名单）
    ↓
如果定义了 tools → 在剩余池上取交集（白名单）
    ↓
最终工具池
```

**关键规则**：
- 如果 `tools` 和 `disallowedTools` 共存，`disallowedTools` 先执行。
- 一个工具同时出现在两个列表中 → 被移除（黑名单优先）。
- MCP 级模式：`mcp__github` 移除整个 GitHub MCP 服务器的所有工具。

### 6.2 Agent 工具的 Allowlist 语法

当代理作为主线程运行时（`claude --agent`），可以通过 `Agent(type)` 语法限制可派出的子代理类型：

```yaml
# 仅允许 worker 和 researcher 两种子代理
tools: Agent(worker, researcher), Read, Bash

# 允许所有子代理（无限制）
tools: Agent, Read, Bash

# 不写 Agent → 禁止任何子代理
tools: Read, Bash
```

**注意**：这个语法仅对主线程代理生效。Subagent 定义中写 `Agent` 表示允许嵌套子代理（括号内的类型列表被忽略）。

**源码参考**：[`src/tools/AgentTool/agentToolUtils.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/agentToolUtils.ts) — `resolveAgentTools()` 实现。

### 6.3 权限模式继承规则

| 父会话模式 | 子代理能否覆盖？ | 行为 |
|-----------|----------------|------|
| `bypassPermissions` | ❌ 否 | 强制优先，子代理也跳过权限 |
| `acceptEdits` | ❌ 否 | 强制优先 |
| `auto` | ❌ 否 | 子代理继承 auto，frontmatter 的 permissionMode 被忽略 |
| `default` / `plan` / `dontAsk` | ✅ 是 | 子代理可覆盖 |

**设计解读**：更宽松的权限模式具有强制性，防止子代理意外引入额外的权限检查。

---

## 七、MCP 服务器隔离

### 7.1 两种引用方式

Subagent 可以通过两种方式引用 MCP 服务器：

```yaml
mcpServers:
  # 方式 1: 字符串引用 → 共享父会话的 MCP 连接
  - github
  
  # 方式 2: 内联定义 → 子代理启动时连接，结束时断开
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
```

**关键区别**：
- 引用：复用父会话的连接，不增加额外开销
- 内联：子代理独占连接，适合那些 **不想出现在父会话上下文中的 MCP 工具**

**源码参考**：[`src/tools/AgentTool/runAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/runAgent.ts) — `initializeAgentMcpServers()` 函数。新创建的 MCP 连接在子代理结束时清理，共享连接不关闭。

### 7.2 安全策略

| 策略 | 效果 |
|------|------|
| 插件子代理 | 不支持 `mcpServers` 和 `hooks` 字段（安全限制） |
| `--strict-mcp-config` | 子代理仅使用指定 MCP 配置 |
| `--bare` | 跳过所有 MCP 自动发现 |
| 企业 MCP 策略 | 覆盖所有子代理，无论来源 |

---

## 八、持久记忆系统（Agent Memory）

### 8.1 三级存储范围

通过 `memory` 字段启用后，子代理获得一个跨会话持久的记忆目录：

| 范围 | 路径 | 用途 | Git 追踪 |
|------|------|------|---------|
| `user` | `~/.claude/agent-memory/<name>/` | 跨项目学习 | ❌ 不 |
| `project` | `.claude/agent-memory/<name>/` | 项目专属知识 | ✅ 可 |
| `local` | `.claude/agent-memory-local/<name>/` | 项目专属，不入库 | ❌ 不 |

**源码参考**：[`src/tools/AgentTool/agentMemory.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/agentMemory.ts) — `getAgentMemoryDir()` 和 `isAgentMemoryPath()`。

### 8.2 自动能力注入

启用 `memory` 后，系统自动：
1. 将 Read/Write/Edit 工具添加到子代理的工具池
2. 在系统提示中注入记忆使用说明（读取和写入 `MEMORY.md` 的指引）
3. 自动加载 `MEMORY.md` 的前 200 行或 25KB（取较小值）

### 8.3 安全路径检测

`isAgentMemoryPath()` 函数通过规范化路径并检测前缀，防止路径穿越攻击：

```typescript
export function isAgentMemoryPath(absolutePath: string): boolean {
  const normalizedPath = normalize(absolutePath);
  // User scope
  if (normalizedPath.startsWith(join(memoryBase, 'agent-memory') + sep))
    return true;
  // Project scope
  if (normalizedPath.startsWith(join(getCwd(), '.claude', 'agent-memory') + sep))
    return true;
  // Local scope
  // ...
  return false;
}
```

---

## 九、钩子系统（Hooks）

### 9.1 Frontmatter 钩子

子代理可以在 frontmatter 中定义会话级钩子：

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
  Stop:    # 运行时自动转换为 SubagentStop
    - type: command
      command: "./scripts/notify-done.sh"
```

**生命周期**：
- 钩子在子代理启动时注册（`registerFrontmatterHooks()`）
- 子代理结束时自动清理（`clearSessionHooks()`）
- `Stop` 事件在运行时自动转换为 `SubagentStop`

### 9.2 项目级钩子

除了 frontmatter 钩子，还可以在 `settings.json` 中定义响应子代理事件的项目级钩子：

```json
{
  "hooks": {
    "SubagentStart": [{
      "matcher": "db-agent",
      "hooks": [{ "type": "command", "command": "./scripts/setup-db.sh" }]
    }],
    "SubagentStop": [{
      "hooks": [{ "type": "command", "command": "./scripts/cleanup-db.sh" }]
    }]
  }
}
```

**源码参考**：[`src/utils/hooks/registerFrontmatterHooks.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/hooks/registerFrontmatterHooks.ts)、[`src/utils/hooks/sessionHooks.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/hooks/sessionHooks.ts)

---

## 十、Agent Teams：超越 Subagent 的协作

### 10.1 架构对比

当任务不仅需要并行，还需要 **相互通信和协作** 时，Subagent 不再够用。Agent Teams 提供了更强大的协作模型：

```
┌─────────────────────────────────────────────────────────┐
│                    Subagent 模型                          │
│                                                         │
│  Lead ────► Subagent A ────► 结果回传                    │
│       └───► Subagent B ────► 结果回传                    │
│                                                         │
│  ⚠ Subagent A 和 B 不能直接通信                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                   Agent Teams 模型                       │
│                                                         │
│         ┌──────── Shared TaskList ────────┐             │
│         │        (file locking)           │             │
│         └──────┬──────────────┬───────────┘             │
│                │              │                         │
│  Lead ◄────► Teammate A ◄──► Teammate B                │
│    ↑           │              │                         │
│    │           └──► Mailbox ◄─┘                         │
│    │                                                   │
│    └─── 双向通信，每个 teammate 是独立进程                 │
└─────────────────────────────────────────────────────────┘
```

| 维度 | Subagent | Agent Team |
|------|---------|-----------|
| 上下文 | 独立窗口，结果回传 | 完全独立窗口 |
| 通信 | 单向（→ 父会话） | 双向（Mailbox） |
| 协调 | 父会话管理所有工作 | 共享 TaskList，自协调 |
| 适用场景 | 结果重要的聚焦任务 | 需要讨论和协作的复杂工作 |
| Token 成本 | 低（仅摘要回传） | 高（每个 teammate 独立消耗） |

### 10.2 核心组件

| 组件 | 角色 | 源码参考 |
|------|------|---------|
| **Team Lead** | 主会话，负责任务分配 | `src/tools/AgentTool/AgentTool.tsx` |
| **Teammates** | 独立 Claude Code 进程 | `src/tools/shared/spawnMultiAgent.ts` |
| **TaskList** | 共享任务列表（文件锁防竞态） | `src/tasks/` 目录 |
| **Mailbox** | 代理间消息系统 | `src/utils/teammateMailbox.ts` |

**存储结构**：
```
~/.claude/teams/{team-name}/config.json  # 运行时状态（会话结束后删除）
~/.claude/tasks/{team-name}/             # 任务列表（会话间持久）
```

**源码参考**：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts)、[`src/utils/teammateMailbox.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateMailbox.ts)

### 10.3 子代理定义在 Team 中的复用

Agent Teammate 可以引用已有的 Subagent 定义：

```
"Spawn a teammate using the security-reviewer agent type"
    ↓
复用 .claude/agents/security-reviewer.md 的 tools 和 model
    ↓
定义正文追加到 teammate 系统提示（而非替换）
    ↓
协调工具（SendMessage、Task 管理）始终可用
```

**注意**：当 Subagent 定义作为 Teammate 运行时，`skills` 和 `mcpServers` 字段 **不被应用** —— Teammate 从项目和用户设置加载这些，和普通会话一样。

---

## 十一、性能优化策略

### 11.1 Prompt Cache 优化

Claude Code 对 prompt cache 的优化达到了字节级精度：

#### Agent 列表分离（v2.1.199+）

动态代理列表曾占 fleet cache_creation tokens 的 ~10.2%。MCP 变化、权限模式变化等都会导致 tool schema 变化，进而使整个 cache 失效。

**解决方案**：将代理列表从 tool description 中分离，作为 attachment message 注入（`shouldInjectAgentListInMessages()`）：

```typescript
// src/tools/AgentTool/prompt.ts
export function shouldInjectAgentListInMessages(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false);
}
```

#### Fork 字节级精确

如前所述，所有 fork 子代理的 API 请求前缀必须字节完全相同。实现方式是：
- Fork 子代理 **不使用** `getSystemPrompt()` 重建系统提示
- 而是直接传递父会话 **已渲染的字节**（`toolUseContext.renderedSystemPrompt`）

这避免了 GrowthBook 冷/热启动差异导致的 prompt 不一致。

### 11.2 后台执行模式

```typescript
// src/tools/AgentTool/AgentTool.tsx
const PROGRESS_THRESHOLD_MS = 2000;  // 2秒后显示后台提示

function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) ||
      getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
    return 120_000;  // 120秒自动后台
  }
  return 0;
}
```

v2.1.198 起，Claude 默认将子代理运行为后台任务。子代理运行超过 120 秒自动转为后台，完成后通知。

### 11.3 扩展思考（Extended Thinking）继承

v2.1.198 起，子代理继承父会话的 extended thinking 配置：
- 父会话开启 thinking → 子代理开启
- 父会话关闭 thinking → 子代理关闭
- **没有独立的子代理 thinking 设置**

**源码参考**：[`src/utils/model/agent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/model/agent.ts)

---

## 十二、安全设计

### 12.1 权限边界

Subagent 和 Teammate 在安全方面有严格的边界：

| 规则 | 说明 |
|------|------|
| 不能代表用户批准权限 | Teammate 无法代替用户点击 "Approve" |
| 中继声明不可信 | 在 auto 模式下，从其他代理中继的批准声明视为不可信输入 |
| 权限提示冒泡 | Teammate 的权限提示上推到 Lead 会话，由用户决定 |
| 无法绕过拒绝 | 被拒绝的操作不能通过中继到其他 teammate 来绕过 |

### 12.2 工具沙箱

以下工具对 Subagent 不可用（即使在 `tools` 列表中列出）：

| 工具 | 原因 |
|------|------|
| `AskUserQuestion` | 依赖主会话 UI |
| `EnterPlanMode` | 依赖主会话状态 |
| `ExitPlanMode` | 除非子代理的 permissionMode 为 `plan` |
| `ScheduleWakeup` | 依赖主会话调度 |
| `WaitForMcpServers` | 依赖主会话 MCP 状态 |

### 12.3 插件子代理的额外限制

通过插件安装的子代理不支持以下字段（安全原因）：
- `hooks` — 防止插件注入任意命令
- `mcpServers` — 防止插件连接到任意 MCP 服务器
- `permissionMode` — 防止插件跳过权限检查

---

## 十三、SDK 层面的 Subagent

Claude Agent SDK 提供了编程式定义和调用 Subagent 的 API：

```typescript
import { Agent } from '@anthropic-ai/claude-agent-sdk';

const agent = new Agent({
  agents: {
    'code-reviewer': {
      description: 'Expert code reviewer',
      prompt: 'You are a senior code reviewer...',
      tools: ['Read', 'Grep', 'Glob'],
      model: 'sonnet',
    },
  },
});
```

**关键环境变量**：
- `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` — 禁用所有内置代理，仅使用自定义代理集
- 适用于需要将代理定义完全控制在代码中的场景

**源码参考**：[`src/entrypoints/agentSdkTypes.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/entrypoints/agentSdkTypes.ts)

---

## 十四、总结与设计哲学

Claude Code 的 Subagent 系统展现了几个核心设计哲学：

### 1. 上下文隔离（Context Isolation）
> "Subagents work within their own context window."

探索与实现分离是 Subagent 存在的首要理由。通过让探索工作在独立窗口中进行，主会话上下文保持清洁，只接收精炼摘要。

### 2. 渐进式复杂性（Progressive Complexity）
从 **Subagent**（单向结果回传）→ **Fork**（共享 cache，异步通知）→ **Agent Teams**（双向通信，共享 TaskList），Claude Code 提供了三层递进的并行模型。用户可以根据任务的协作需求选择最合适的层次。

### 3. Prompt Cache 优先（Cache-First Design）
所有优化决策都以最大化 cache 命中率为目标：
- Fork 的字节级精确请求前缀
- Agent 列表从 tool schema 中分离
- 系统提示直接传递已渲染字节而非重建

### 4. 安全默认（Secure by Default）
- 工具继承为白名单模式（不写 `tools` = 继承全部，写了 = 仅允许列出的）
- 权限冒泡到用户，子代理不能代替用户做决定
- 插件子代理被额外限制（无 hooks、无 MCP、无权限模式覆盖）

### 5. 可扩展性（Extensibility）
- Markdown 定义文件（非代码）让非开发者也能创建子代理
- 五级优先级覆盖系统支持从个人到企业的各种场景
- SDK 提供编程式 API 用于自动化场景

### 参考文件索引

| 文件 | 路径 | 说明 |
|------|------|------|
| AgentTool 核心 | [`src/tools/AgentTool/AgentTool.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/AgentTool.tsx) | 代理工具定义、输入 schema、调度逻辑 |
| 代理执行 | [`src/tools/AgentTool/runAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/runAgent.ts) | API 调用循环、MCP 初始化、进度追踪 |
| 定义加载 | [`src/tools/AgentTool/loadAgentsDir.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) | Zod schema、目录扫描、优先级处理 |
| 提示构建 | [`src/tools/AgentTool/prompt.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/prompt.ts) | Agent 列表注入、fork 语义说明 |
| Fork 机制 | [`src/tools/AgentTool/forkSubagent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/forkSubagent.ts) | 消息构建、防递归、cache 优化 |
| 代理记忆 | [`src/tools/AgentTool/agentMemory.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/agentMemory.ts) | 三级 scope、路径安全 |
| 通用代理 | [`src/tools/AgentTool/built-in/generalPurposeAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/built-in/generalPurposeAgent.ts) | 内置代理定义 |
| 团队调度 | [`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts) | Teammate 生成、模型解析、tmux 管理 |
| 任务管理 | [`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx) | 进度追踪、生命周期管理 |
| 代理模型 | [`src/utils/model/agent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/model/agent.ts) | 模型选择、thinking 继承 |
| 团队邮箱 | [`src/utils/teammateMailbox.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateMailbox.ts) | 代理间消息系统 |
| 工具过滤 | [`src/tools/AgentTool/agentToolUtils.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/agentToolUtils.ts) | 工具池解析 |
| SDK 类型 | [`src/entrypoints/agentSdkTypes.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/entrypoints/agentSdkTypes.ts) | Agent SDK 接口定义 |

**官方文档**：
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents.md)
- [Agent teams](https://code.claude.com/docs/en/agent-teams.md)
- [Run agents in parallel](https://code.claude.com/docs/en/agents.md)
- [Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview.md)

---

*本文基于 Claude Code v2.1.x 源码与官方文档编写。源码参考来自 [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) 仓库。*
