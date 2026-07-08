# Claude Code Agent Teams 原理与设计实现深度解析

> "Agent teams let you coordinate multiple Claude Code instances working together. One session acts as the team lead, coordinating work, assigning tasks, and synthesizing results." — [Claude Code Agent Teams 官方文档](https://code.claude.com/docs/en/agent-teams.md)

Claude Code 的 Agent Teams 是其多代理体系中最复杂也最具野心的设计。与 Subagent 的"单向委派、结果回传"不同，Agent Teams 构建了一个 **多代理协作网络**：每个 teammate 拥有独立的上下文窗口，通过共享 TaskList 自协调工作，通过 Mailbox 直接通信，形成了一个去中心化的代理协作系统。

本文深入 `oboard/claude-code-rev` 源码与官方文档，逐层拆解 Agent Teams 的设计原理、实现机制与工程取舍。

---

## 一、引言：从 Subagent 到 Agent Teams

### 1.1 Subagent 的根本局限

Subagent 解决了"上下文污染"问题，但它有一个不可逾越的边界：**通信是单向的**。

```
Lead ────► Subagent A ────► 结果回传
     └───► Subagent B ────► 结果回传

⚠ Subagent A 和 B 之间无法通信
⚠ 所有协调必须通过 Lead 中转
⚠ 适合"结果重要、过程不重要"的任务
```

当任务需要代理之间**交换发现、挑战彼此、协作构建**时，Subagent 模型就不够用了。

### 1.2 Agent Teams 的核心价值

Agent Teams 解决了 Subagent 的三个根本限制：

| 维度 | Subagent | Agent Team |
|------|---------|-----------|
| 通信方向 | 单向（→ 父会话） | 双向（teammate ↔ teammate） |
| 协调方式 | Lead 集中管理 | 共享 TaskList 自协调 |
| 适用场景 | 结果导向的聚焦任务 | 需要讨论和协作的复杂工作 |

**设计哲学**：Subagent 是"委派"，Agent Team 是"协作"。前者是主从架构，后者是对等架构（peer-to-peer）。

**源码参考**：
- [官方文档：Agent Teams](https://code.claude.com/docs/en/agent-teams.md)
- 团队生成：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts)
- 进程内执行：[`src/utils/swarm/inProcessRunner.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/inProcessRunner.ts)
- 任务框架：[`src/utils/task/framework.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/task/framework.ts)

---

## 二、系统总览：Agent Teams 架构全景

### 2.1 核心组件架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Agent Team 架构                            │
│                                                              │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────┐│
│  │  Teammate A  │◄──►│  Shared TaskList │◄──►│  Teammate B  ││
│  │  (进程内)     │    │  (文件锁防竞态)   │    │  (tmux 分屏)  ││
│  │              │    └────────┬─────────┘    │              ││
│  │  AsyncLS    │             │               │  tmux pane   ││
│  │  上下文隔离  │             │               │  进程隔离     ││
│  └──────┬──────┘             │               └──────┬───────┘│
│         │                    │                      │       │
│         │    ┌───────────────┴───────────────┐      │       │
│         └───►│         Mailbox               │◄─────┘       │
│              │    (代理间消息系统)            │              │
│              │  - UserMessage                │              │
│              │  - PermissionResponse         │              │
│              │  - ShutdownRequest            │              │
│              │  - PeerDmSummary              │              │
│              └───────────────┬───────────────┘              │
│                              │                              │
│                    ┌─────────┴─────────┐                    │
│                    │   Team Lead       │                    │
│                    │   (主会话)         │                    │
│                    │                   │                    │
│                    │  - 任务分配        │                    │
│                    │  - 计划审批        │                    │
│                    │  - 结果综合        │                    │
│                    │  - 权限冒泡接收    │                    │
│                    └───────────────────┘                    │
│                                                              │
│  存储层：                                                     │
│  ~/.claude/teams/{team-name}/config.json  (运行时状态)        │
│  ~/.claude/tasks/{team-name}/              (任务列表)         │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 四大核心组件

| 组件 | 角色 | 核心源码 | 关键技术 |
|------|------|---------|---------|
| **Team Lead** | 主会话，负责任务分配和协调 | `AgentTool.tsx` | 任务创建工具、计划审批 |
| **Teammates** | 独立代理实例 | `inProcessRunner.ts` / `spawnMultiAgent.ts` | AsyncLocalStorage / tmux |
| **TaskList** | 共享任务列表 | `tasks.ts` / `framework.ts` | 文件锁防竞态 |
| **Mailbox** | 代理间消息系统 | `teammateMailbox.ts` | 消息队列、权限冒泡 |

**源码参考**：
- Agent tool 核心：[`src/tools/AgentTool/AgentTool.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/AgentTool.tsx)
- 任务框架：[`src/utils/task/framework.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/task/framework.ts)
- 团队辅助：[`src/utils/swarm/teamHelpers.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/teamHelpers.ts)
- 消息系统：[`src/utils/teammateMailbox.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateMailbox.ts)

---

## 三、启用与初始化机制

### 3.1 门控开关：实验性功能

Agent Teams 默认禁用，需要通过环境变量显式启用：

```json
// settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**未启用时的行为**：
- 不创建团队目录
- 不生成 teammates
- 不显示协调 UI
- `isAgentSwarmsEnabled()` 返回 false

```typescript
// src/utils/agentSwarmsEnabled.ts
export function isAgentSwarmsEnabled(): boolean {
  return isEnvTruthy(process.env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS);
}
```

**设计解读**：作为实验性功能，Claude Code 采用了 **feature flag** 模式，而非默认开启。这与 Martin Fowler 在《Feature Toggles》中描述的最佳实践一致：将未完成的实验性功能隐藏在开关后面，避免影响主流程。

### 3.2 团队命名与存储结构

团队名称由会话 ID 派生：

```
team-name = "session-" + sessionId.substring(0, 8)
// 示例：session-a1b2c3d4
```

存储结构分为两层：

```
~/.claude/
├── teams/
│   └── session-a1b2c3d4/
│       └── config.json          # 团队运行时状态（会话结束后删除）
│
└── tasks/
    └── session-a1b2c3d4/
        ├── tasks.json           # 任务列表（会话间持久）
        └── task-{id}.json       # 单个任务详情
```

**关键区别**：
- `teams/` 目录存储**运行时状态**（tmux pane IDs、session IDs 等），会话结束时自动清理
- `tasks/` 目录存储**任务数据**，跨会话持久保留，`cleanupPeriodDays` 控制保留策略

**源码参考**：[`src/utils/swarm/teamHelpers.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/teamHelpers.ts) — `getTeamDir()`、`getTeamFilePath()`、`registerTeamForSessionCleanup()`。

### 3.3 TeamFile 完整格式

```typescript
// src/utils/swarm/teamHelpers.ts
type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string          // Leader 的实际会话 UUID
  hiddenPaneIds?: string[]        // UI 隐藏的 pane IDs
  teamAllowedPaths?: TeamAllowedPath[]  // 所有 teammate 可编辑的路径
  members: Array<{
    agentId: string               // 格式：name@team
    name: string
    agentType?: string
    model?: string
    prompt?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType     // "in-process" | "tmux" | "iterm2"
    isActive?: boolean            // false = idle, undefined/true = active
    mode?: PermissionMode         // 当前权限模式
  }>
}
```

**安全提示**：官方文档明确指出"don't edit it by hand or pre-author it"——团队配置文件是纯运行时状态，手动编辑会被下一次状态更新覆盖。

---

## 四、Teammate 生成机制

### 4.1 生成触发：三种路径

```
┌─────────────────────────────────────────────────────────┐
│              Teammate 生成触发路径                       │
│                                                         │
│  1. 用户显式请求                                        │
│     "Spawn 3 teammates to investigate different angles"  │
│     → Claude 按指令生成指定数量/角色的 teammate          │
│                                                         │
│  2. Claude 主动建议                                     │
│     Claude 判断任务需要并行处理                           │
│     → 提出建议，用户确认后才生成                          │
│     → "This task would benefit from parallel work..."    │
│                                                         │
│  3. 通过 Subagent 定义引用                               │
│     "Spawn a teammate using the security-reviewer type"  │
│     → 复用 .claude/agents/security-reviewer.md 的定义    │
│                                                         │
│  ⚠ 无论哪种路径，用户确认后才会生成（保持控制权）           │
└─────────────────────────────────────────────────────────┘
```

### 4.2 两种运行模式：进程内 vs 分屏

| 维度 | 进程内（In-Process） | 分屏（Split Pane） |
|------|---------------------|-------------------|
| **实现** | 同一 Node.js 进程，AsyncLocalStorage 隔离 | 独立 Claude Code 进程 |
| **依赖** | 无额外依赖 | tmux 或 iTerm2 + it2 CLI |
| **隔离** | 上下文隔离（AsyncLocalStorage） | 进程隔离 |
| **子代理** | 子代理在前台运行 | 子代理可后台运行 |
| **性能** | 轻量，无进程启动开销 | 较重，需要完整进程启动 |
| **适用** | 大多数场景 | 需要同时查看所有 teammate 输出 |

#### 进程内模式的核心技术：AsyncLocalStorage

```typescript
// src/utils/swarm/inProcessRunner.ts
import { AsyncLocalStorage } from 'async_hooks';

// 每个 teammate 有独立的 AsyncLocalStorage 上下文
const teammateContextStore = new AsyncLocalStorage<TeammateContext>();

export function runWithTeammateContext<T>(
  context: TeammateContext,
  fn: () => Promise<T>,
): Promise<T> {
  return teammateContextStore.run(context, fn);
}
```

**设计解读**：Node.js 的 `AsyncLocalStorage` 是 Agent Teams 进程内隔离的核心。它类似于 Java 的 `ThreadLocal`，但在异步上下文中工作 —— 每个异步调用链有独立的存储，即使多个 teammate 在同一进程中共存，它们也能保持身份隔离。

#### 分屏模式的后端检测

```typescript
// src/utils/swarm/backends/registry.ts
export async function detectAndGetBackend(): Promise<BackendType> {
  // 1. 检查是否在 tmux 会话内
  if (await isInsideTmux()) return 'tmux';
  
  // 2. 检查 iTerm2 + it2 CLI
  if (await isIterm2WithIt2()) return 'iterm2';
  
  // 3. 回退到进程内
  return 'in-process';
}
```

**源码参考**：
- 进程内生成：[`src/utils/swarm/spawnInProcess.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/spawnInProcess.ts)
- 分屏生成：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts)
- 后端检测：[`src/utils/swarm/backends/registry.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/backends/registry.ts)

### 4.3 进程内生成完整流程

```
spawnInProcessTeammate(config, context)
    │
    ├─ 1. 生成 agentId（格式：name@team）
    │      formatAgentId(name, teamName) → "researcher@my-team"
    │
    ├─ 2. 创建独立 AbortController
    │      createAbortController()
    │      ⚠ teammate 不应因 leader 的查询中断而被 abort
    │
    ├─ 3. 获取父会话 ID
    │      getSessionId() → 用于 transcript 关联
    │
    ├─ 4. 创建 TeammateIdentity（纯数据，存储在 AppState）
    │      {
    │        agentId: "researcher@my-team",
    │        agentName: "researcher",
    │        teamName: "my-team",
    │        color: "blue",
    │        planModeRequired: false,
    │        parentSessionId: "a1b2c3d4-..."
    │      }
    │
    ├─ 5. 创建 TeammateContext（用于 AsyncLocalStorage）
    │      createTeammateContext({ ...identity, abortController })
    │
    ├─ 6. 注册 Perfetto 追踪（性能分析）
    │      registerPerfettoAgent(agentId)
    │
    ├─ 7. 注册 InProcessTeammateTaskState 到 AppState
    │      registerTask(taskState, setAppState)
    │
    ├─ 8. 注册清理回调
    │      registerCleanup(() => unregisterPerfettoAgent(agentId))
    │
    └─ 9. 返回 spawn result
           { success, agentId, taskId, abortController, teammateContext }
```

**源码参考**：[`src/utils/swarm/spawnInProcess.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/spawnInProcess.ts) — `spawnInProcessTeammate()` 完整实现。

### 4.4 分屏生成流程

```
spawnTeammate(config)
    │
    ├─ 1. 解析后端类型（tmux / iterm2）
    │      getBackendByType() → 检测可用后端
    │
    ├─ 2. 确保 tmux session 存在
    │      ensureSession(SWARM_SESSION_NAME)
    │      tmux new-session -d -s {SWARM_SESSION_NAME}
    │
    ├─ 3. 分配颜色和 pane
    │      assignTeammateColor(name) → "blue"
    │      createTeammatePaneInSwarmView() → pane ID
    │
    ├─ 4. 构建 teammate 命令
    │      claude --agent {agentType} --teammate-mode in-process
    │
    ├─ 5. 发送命令到 pane
    │      sendCommandToPane(paneId, command)
    │
    ├─ 6. 注册任务到框架
    │      registerTask(taskState, setAppState)
    │
    ├─ 7. 写入 Mailbox（通知 teammate 已加入）
    │      writeToMailbox(agentId, { type: "join", ... })
    │
    └─ 8. 返回 SpawnOutput
           { teammate_id, agent_id, tmux_session_name, tmux_pane_id, ... }
```

**源码参考**：[`src/tools/shared/spawnMultiAgent.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts) — `spawnTeammate()` 和 `resolveTeammateModel()`。

---

## 五、TaskList 系统：共享任务协调

### 5.1 任务状态机

```
                    ┌─────────────┐
                    │   pending   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │in_progress│ │ blocked  │ │ cancelled│
        └────┬─────┘ └────┬─────┘ └──────────┘
             │            │
        ┌────┴────┐       │（依赖完成后自动解除）
        ▼         ▼
  ┌──────────┐ ┌──────────┐
  │completed │ │  failed  │
  └──────────┘ └──────────┘
```

### 5.2 任务分配：两种模式

#### Lead 分配模式
```
用户："Spawn 3 teammates to review the PR"
    ↓
Lead 创建任务：
  - Task 1: "Review auth module for security issues" → Teammate A
  - Task 2: "Check performance impact" → Teammate B
  - Task 3: "Validate test coverage" → Teammate C
    ↓
Teammate 认领并执行
```

#### 自认领（Self-claim）模式
```
Teammate A 完成 Task 1
    ↓
自动扫描 TaskList：
  - 查找状态为 pending 且依赖已解决的任务
  - 使用文件锁防止竞态
    ↓
Teammate A 认领下一个可用任务
```

### 5.3 文件锁防竞态

当多个 teammate 同时尝试认领同一任务时，使用文件锁防止竞态：

```typescript
// src/utils/tasks.ts (概念性)
import { flock } from 'fs';

async function claimTask(taskId: string, agentId: string): Promise<boolean> {
  const lockFile = getTaskLockPath(taskId);
  
  // 获取排他锁
  await acquireExclusiveLock(lockFile);
  
  try {
    const task = await readTask(taskId);
    if (task.status !== 'pending') return false;
    
    // 更新状态为 in_progress
    task.status = 'in_progress';
    task.claimedBy = agentId;
    task.claimedAt = Date.now();
    await writeTask(taskId, task);
    return true;
  } finally {
    releaseLock(lockFile);
  }
}
```

**设计解读**：文件锁是 Unix 系统中最基础的进程间同步机制。选择文件锁而非内存锁，是因为 teammates 可能是**独立进程**（分屏模式），内存锁无法跨进程工作。

### 5.4 任务依赖管理

任务可声明依赖其他任务：

```json
{
  "id": "task-3",
  "title": "Integrate auth and performance changes",
  "status": "blocked",
  "dependencies": ["task-1", "task-2"],
  "assignee": null
}
```

- `task-1` 和 `task-2` 完成后，`task-3` 自动解除阻塞
- teammate 可以认领任何未被阻塞的 pending 任务
- Lead 可以手动调整依赖关系

**源码参考**：
- 任务框架：[`src/utils/task/framework.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/task/framework.ts) — `registerTask()`、`updateTaskState()`
- 任务工具：`src/tools/TaskCreateTool/`、`src/tools/TaskUpdateTool/`、`src/tools/TaskListTool/`

---

## 六、Mailbox 系统：代理间通信

### 6.1 消息模型

每个 teammate 有独立的 mailbox，消息通过结构化 JSON 传递：

```typescript
// src/utils/teammateMailbox.ts (概念性)
type MailboxMessage = {
  from: string;        // 发送者 agentId
  to: string;          // 接收者 agentId
  type: MessageType;
  timestamp: number;
  content: unknown;
};

type MessageType =
  | 'UserMessage'         // 用户或 Lead 发送的消息
  | 'PermissionResponse'  // 权限请求响应
  | 'ShutdownRequest'     // 关闭请求
  | 'PeerDmSummary'       // 代理间私聊摘要
  | 'TaskAssignment'      // 任务分配
  | 'TaskCompletion'      // 任务完成通知
  | 'PlanApproval';       // 计划审批结果
```

### 6.2 权限冒泡机制

这是 Agent Teams 安全设计的核心：

```
Teammate 执行危险操作（如 rm -rf）
    ↓
触发权限提示
    ↓
通过 Mailbox 发送 PermissionRequest 到 Lead
    ↓
Lead 的终端显示权限提示
    ↓
用户批准/拒绝
    ↓
PermissionResponse 通过 Mailbox 回传给 teammate
```

**关键安全规则**：
1. teammate **不能代表用户**批准权限
2. 在 auto 模式下，中继的批准声明视为**不可信输入**
3. 被拒绝的操作不能通过中继到其他 teammate 来绕过

```typescript
// src/utils/swarm/permissionSync.ts (概念性)
async function sendPermissionRequest(teammateId: string, request: PermissionRequest) {
  // 通过 Mailbox 发送到 Lead
  await writeToMailbox(TEAM_LEAD_NAME, {
    type: 'PermissionRequest',
    from: teammateId,
    request: request,
  });
  
  // 等待 Lead 的响应（轮询）
  const response = await pollPermissionResponse(teammateId);
  return response;
}
```

**源码参考**：[`src/utils/teammateMailbox.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateMailbox.ts) — `writeToMailbox()`、`readMailbox()`、`createIdleNotification()`。

---

## 七、进程内执行引擎

### 7.1 InProcessTeammateTask 状态

```typescript
// src/tasks/InProcessTeammateTask/types.ts
type InProcessTeammateTaskState = {
  type: 'in_process_teammate'
  
  // 身份（纯数据，存储在 AppState）
  identity: TeammateIdentity
  
  // 执行
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController          // Runtime only
  currentWorkAbortController?: AbortController // Runtime only
  
  // Plan mode
  awaitingPlanApproval: boolean
  
  // 权限
  permissionMode: PermissionMode
  
  // 状态
  error?: string
  result?: AgentToolResult
  progress?: AgentProgress
  
  // 对话历史（capped at 50）
  messages?: Message[]
  pendingUserMessages: string[]
  
  // 生命周期
  isIdle: boolean
  shutdownRequested: boolean
  
  // 回调
  onIdleCallbacks?: Array<() => void>
  
  // 进度追踪
  lastReportedToolCount: number
  lastReportedTokenCount: number
}
```

### 7.2 消息上限机制：50 条硬限制

```typescript
// src/tasks/InProcessTeammateTask/types.ts
export const TEAMMATE_MESSAGES_UI_CAP = 50;

export function appendCappedMessage<T>(
  prev: readonly T[] | undefined,
  item: T,
): T[] {
  if (prev === undefined || prev.length === 0) return [item];
  if (prev.length >= TEAMMATE_MESSAGES_UI_CAP) {
    // 丢弃最旧的，保留最新的 49 条 + 新消息
    const next = prev.slice(-(TEAMMATE_MESSAGES_UI_CAP - 1));
    next.push(item);
    return next;
  }
  return [...prev, item];
}
```

**为什么是 50？**

源码注释给出了精确的数据支撑：
> "BQ analysis (round 9, 2026-03-20) showed ~20MB RSS per agent at 500+ turn sessions and ~125MB per concurrent agent in swarm bursts. Whale session 9a990de8 launched 292 agents in 2 minutes and reached 36.8GB. The dominant cost is this array holding a second full copy of every message."

**关键洞察**：`task.messages` 是 AppState 中的 UI 镜像，不是真实对话历史。真实对话保存在：
1. `allMessages` 数组（`inProcessRunner` 中）
2. 磁盘上的 agent transcript 路径

这种**双层存储**设计避免了内存爆炸：UI 只保留最新的 50 条消息用于缩放视图，完整对话通过磁盘持久化。

### 7.3 InProcessTeammateTask 执行流程

```typescript
// src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx
export const InProcessTeammateTask: Task = {
  name: 'InProcessTeammateTask',
  type: 'in_process_teammate',
  async kill(taskId, setAppState) {
    killInProcessTeammate(taskId, setAppState);
  }
};
```

实际的执行由 `inProcessRunner.ts` 驱动：

```
runWithTeammateContext(teammateContext, async () => {
  // 1. 构建系统提示
  // 2. 初始化 MCP 服务器
  // 3. 注册钩子
  // 4. 执行 runAgent() 循环
  // 5. 处理 Mailbox 消息（权限请求、关闭请求等）
  // 6. 更新进度
  // 7. 空闲通知
  // 8. 清理
})
```

**源码参考**：
- 任务类型：[`src/tasks/InProcessTeammateTask/types.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/InProcessTeammateTask/types.ts)
- 任务管理：[`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx)
- 执行器：[`src/utils/swarm/inProcessRunner.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/inProcessRunner.ts)

### 7.4 Plan Mode 审批流

```
Teammate (planModeRequired=true)
    │
    ├─ 1. 收到任务
    │
    ├─ 2. 进行研究（只读工具可用）
    │
    ├─ 3. 制定计划
    │    <plan>...</plan>
    │
    ├─ 4. 提交审批 → Lead 收到 PlanApprovalRequest
    │
    ├─ 5. Lead 审查计划
    │    ├─ 批准 → teammate 退出 plan mode，开始实施
    │    └─ 拒绝 + 反馈 → teammate 修改计划，重新提交
    │
    └─ 6. 实施完成
```

**源码参考**：`src/utils/swarm/permissionSync.ts`、`src/hooks/useSwarmPermissionPoller.ts`

---

## 八、显示模式与 UI 交互

### 8.1 四种显示模式

| 模式 | 配置值 | 行为 | 依赖 |
|------|--------|------|------|
| **in-process** | `"in-process"` | 所有 teammate 在同一终端 | 无 |
| **auto** | `"auto"` | tmux/iTerm2 时自动分屏 | tmux 或 iTerm2 |
| **tmux** | `"tmux"` | 强制 tmux 分屏 | tmux |
| **iterm2** | `"iterm2"` | iTerm2 原生分屏 | iTerm2 + it2 CLI |

```json
// ~/.claude/settings.json
{
  "teammateMode": "auto"
}
```

**CLI 覆盖**：
```bash
claude --teammate-mode auto
```

### 8.2 空闲隐藏机制

v2.1.199 起的精确行为：

```
所有 teammate 完成工作 → 空闲
    ↓
等待 30 秒
    ↓
空闲行隐藏（不是停止，只是 UI 隐藏）
    ↓
如果超过 3 个空闲 teammate → 折叠为 "N idle agents"
    ↓
发送消息给空闲 teammate → 行重新显示
```

### 8.3 键盘交互

| 按键 | 行为 |
|------|------|
| `↑/↓` | 选择 teammate |
| `Enter` | 查看选中 teammate 的转录 |
| `Esc` | 中断选中 teammate 的当前轮次 |
| `x` | 停止 teammate |
| `Ctrl+T` | 切换任务列表 |
| `Space` | 完成/确认 |
| `←` | 返回 |

**查看 teammate 转录时**：
- 纯文本和 skills 发送到该 teammate
- 内置命令（`/model`、`/compact`）仍在 Lead 会话执行
- `/effort` 应用于被查看的 teammate

**源码参考**：
- UI 组件：[`src/components/Spinner/TeammateSpinnerTree.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/components/Spinner/TeammateSpinnerTree.tsx)
- 详情对话框：[`src/components/tasks/InProcessTeammateDetailDialog.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/components/tasks/InProcessTeammateDetailDialog.tsx)
- 状态辅助：[`src/state/teammateViewHelpers.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/state/teammateViewHelpers.ts)

---

## 九、权限与安全设计

### 9.1 权限继承规则

| 父会话模式 | teammate 能否覆盖 | 行为 |
|-----------|-----------------|------|
| `bypassPermissions` | ❌ | 强制继承，所有 teammate 跳过权限检查 |
| `acceptEdits` | ❌ | 强制继承，自动接受编辑 |
| `auto` | ❌ | 强制继承，分类器评估相同规则 |
| `default` / `plan` / `dontAsk` | ✅ | teammate 可在生成后修改 |

**源码参考**：[`src/utils/swarm/inProcessRunner.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/inProcessRunner.ts) — 权限模式继承检查。

### 9.2 安全边界

```
┌─────────────────────────────────────────────────────────┐
│                  Agent Teams 安全边界                    │
│                                                         │
│  ❌ teammate 不能代表用户批准权限提示                      │
│  ❌ 中继的批准声明在 auto 模式下视为不可信输入              │
│  ❌ 被拒绝的操作不能通过中继到其他 teammate 来绕过           │
│  ❌ teammate 无法创建嵌套团队（无 TeamCreate 权限）         │
│  ❌ teammate 不能提升为 lead                              │
│                                                         │
│  ✅ 权限请求冒泡到 Lead 会话，由用户决定                    │
│  ✅ teammate 只能在自己的工作目录内操作                     │
│  ✅ 敏感工具（AskUserQuestion 等）对 teammate 不可用        │
│  ✅ 权限提示在 Lead 终端显示，用户直接交互                  │
└─────────────────────────────────────────────────────────┘
```

### 9.3 受限工具列表

以下工具对 teammate 不可用，即使在 `tools` 列表中列出：

| 工具 | 限制原因 |
|------|---------|
| `AskUserQuestion` | 依赖主会话 UI |
| `EnterPlanMode` | 依赖主会话状态 |
| `ExitPlanMode` | 除非 teammate 的 permissionMode 为 `plan` |
| `ScheduleWakeup` | 依赖主会话调度 |
| `WaitForMcpServers` | 依赖主会话 MCP 状态 |
| `TeamCreate` | 防止嵌套团队 |
| `TeamDelete` | 防止 teammate 删除团队 |

---

## 十、Subagent 定义在 Team 中的复用

### 10.1 复用机制

Agent Teammate 可以引用已有的 Subagent 定义：

```
用户："Spawn a teammate using the security-reviewer agent type"
    ↓
加载 .claude/agents/security-reviewer.md
    ↓
应用 tools allowlist 和 model
    ↓
定义正文追加到 teammate 系统提示（不是替换）
    ↓
协调工具（SendMessage, Task 管理）始终可用
```

### 10.2 字段应用对比

这是理解 Subagent vs Teammate 区别的关键：

| 字段 | 作为 Subagent | 作为 Teammate | 原因 |
|------|-------------|-------------|------|
| `name` | ✅ 使用 | ✅ 使用 | 身份标识 |
| `description` | ✅ 使用 | ✅ 使用 | 委派判断 |
| `tools` | ✅ 应用 | ✅ 应用 | 能力限制 |
| `model` | ✅ 应用 | ✅ 应用 | 模型选择 |
| `prompt`（正文） | ✅ 作为系统提示 | ✅ 追加到系统提示 | teammate 有额外上下文 |
| `permissionMode` | ✅ 应用 | ❌ 忽略 | teammate 权限由 Lead 控制 |
| `skills` | ✅ 预加载 | ❌ 忽略 | teammate 从项目设置加载 |
| `mcpServers` | ✅ 连接 | ❌ 忽略 | teammate 从项目设置加载 |
| `hooks` | ✅ 注册 | ❌ 忽略 | teammate 使用项目级 hooks |
| `memory` | ✅ 启用 | ❌ 忽略 | teammate 使用标准记忆 |
| `maxTurns` | ✅ 应用 | ✅ 应用 | 执行限制 |

**设计解读**：Teammate 是一个**更重**的实体 —— 它加载完整的项目上下文（CLAUDE.md、MCP、skills），因此不需要也不应该从 Subagent 定义中重复加载这些。只有 `tools`、`model` 和 `prompt` 这种 teammate 专属的配置才会从 Subagent 定义中应用。

**源码参考**：[`src/tools/AgentTool/loadAgentsDir.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) — 代理定义加载逻辑。

---

## 十一、Hook 系统集成

### 11.1 团队生命周期钩子

| 事件 | 触发时机 | 退出码 2 的行为 |
|------|---------|----------------|
| `SubagentStart` | teammate 开始执行 | 阻止启动并发送反馈 |
| `SubagentStop` | teammate 完成 | 阻止完成并发送反馈 |
| `TeammateIdle` | teammate 即将空闲 | 发送反馈，保持 teammate 工作 |
| `TaskCreated` | 任务创建时 | 阻止创建并发送反馈 |
| `TaskCompleted` | 任务标记完成时 | 阻止完成并发送反馈 |

### 11.2 配置示例

```json
{
  "hooks": {
    "SubagentStart": [{
      "matcher": "security-reviewer",
      "hooks": [{ "type": "command", "command": "./scripts/setup-scan.sh" }]
    }],
    "TeammateIdle": [{
      "hooks": [{ "type": "command", "command": "./scripts/check-progress.sh" }]
    }],
    "TaskCompleted": [{
      "matcher": ".*",
      "hooks": [{ "type": "command", "command": "./scripts/validate-output.sh" }]
    }]
  }
}
```

### 11.3 匹配器模式

- `matcher` 值是 teammate 的 `name`（非插件）或插件 scoped 标识符（如 `my-plugin:db-agent`）
- 包含冒号的 scoped 名称按未锚定正则表达式评估，建议用 `^` 和 `$` 锚定
- 短横线名称（如 `db-agent`）在 v2.1.195+ 精确匹配

**源码参考**：[`src/utils/hooks/sessionHooks.ts`](https://github.com/oboard/claude-code-rev/blob/main/src/utils/hooks/sessionHooks.ts)

---

## 十二、远程 Agent 任务

### 12.1 五种远程任务类型

| 类型 | 说明 | 源码 |
|------|------|------|
| `remote-agent` | 通用远程代理 | `RemoteAgentTask.tsx` |
| `ultraplan` | 超计划模式 | 同上 |
| `ultrareview` | 超审查模式 | 同上 |
| `autofix-pr` | PR 自动修复 | 同上 |
| `background-pr` | 后台 PR 处理 | 同上 |

### 12.2 远程会话管理

```typescript
// src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
type RemoteAgentTaskState = {
  type: 'remote_agent';
  remoteTaskType: RemoteTaskType;
  sessionId: string;          // 原始会话 ID
  command: string;
  title: string;
  todoList: TodoList;
  log: SDKMessage[];
  isLongRunning?: boolean;
  pollStartedAt: number;      // 本地轮询器启动时间
  isRemoteReview?: boolean;
  reviewProgress?: {
    stage?: 'finding' | 'verifying' | 'synthesizing';
    bugsFound: number;
    bugsVerified: number;
    bugsRefuted: number;
  };
  isUltraplan?: boolean;
  ultraplanPhase?: Exclude<UltraplanPhase, 'running'>;
};
```

**远程任务完成检查器**：
```typescript
export type RemoteTaskCompletionChecker = (
  remoteTaskMetadata: RemoteTaskMetadata | undefined
) => Promise<string | null>;

// 每个 poll tick 调用，返回非 null 字符串表示任务完成
const completionCheckers = new Map<RemoteTaskType, RemoteTaskCompletionChecker>();
```

**源码参考**：[`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx)

---

## 十三、已知限制

| 限制 | 说明 | 影响 |
|------|------|------|
| **无会话恢复** | `/resume` 和 `/rewind` 不恢复进程内 teammate | 恢复后需要重新生成 teammates |
| **任务状态滞后** | teammate 有时未能标记任务完成 | 依赖任务可能永久阻塞 |
| **关闭缓慢** | teammate 完成当前请求后才关闭 | 可能需要等待 API 调用完成 |
| **单团队限制** | 每会话只能有一个团队 | 无法并行多团队 |
| **无嵌套团队** | teammate 不能生成自己的 teammate | 只能 Lead 管理团队 |
| **无后台子代理** | 进程内 teammate 的子代理在前台运行 | 可能阻塞 teammate |
| **Lead 固定** | 主会话永远是 lead | 不能转移领导权 |
| **分屏依赖** | 需要 tmux 或 iTerm2 | VS Code 集成终端不支持 |
| **权限设置固定** | 生成时继承 Lead 权限 | 生成后可修改但不能预设 |
| **实验性** | 默认禁用 | API 和行为可能变化 |

---

## 十四、最佳实践

### 14.1 团队规模

**推荐 3-5 个 teammate**。原因：
- Token 成本线性增长：每个 teammate 有独立上下文窗口
- 协调开销增加：更多 teammate 意味着更多通信和潜在冲突
- 收益递减：超过某一点后，额外的 teammate 不能成比例加速工作

**经验法则**：每个 teammate 保持 5-6 个任务，保持高效且不频繁上下文切换。

### 14.2 任务粒度

```
❌ 太小：协调开销超过收益
   "Read the auth module"

❌ 太大：teammate 工作时间过长，无检查点，浪费风险增加
   "Rewrite the entire authentication system"

✅ 刚好：自包含单元，产出明确交付物
   "Review src/auth/ for token handling vulnerabilities"
   "Write integration tests for the /api/users endpoint"
   "Refactor the database migration scripts"
```

### 14.3 避免文件冲突

```
✅ 好：每个 teammate 负责不同文件
   Teammate A: src/auth/login.ts
   Teammate B: src/auth/logout.ts
   Teammate C: tests/auth.test.ts

❌ 坏：多个 teammate 编辑同一文件
   Teammate A: src/auth/index.ts
   Teammate B: src/auth/index.ts  → 覆盖风险
```

### 14.4 竞争假设调试模式

```
"Spawn 5 agent teammates to investigate different hypotheses.
Have them talk to each other to try to disprove each other's theories,
like a scientific debate. Update the findings doc with whatever consensus
emerges."
```

**为什么有效？** 顺序调查受锚定效应影响：一旦一个理论被探索，后续调查会偏向它。多个独立调查者主动反驳彼此的理论，存活下来的理论更可能是真正的根因。

---

## 十五、总结与设计哲学

Claude Code Agent Teams 的设计展现了五个核心哲学：

### 1. 协作优先（Collaboration-First）

> "Unlike subagents, which run within a single session and can only report back to the main agent, you can also interact with individual teammates directly."

Agent Teams 不是 Subagent 的简单扩展，而是一种**全新的协作范式** —— 从主从架构（Master-Slave）演进为对等网络（Peer-to-Peer）。每个 teammate 是独立的代理实体，有自己的上下文、身份和决策能力。

### 2. 自协调（Self-Coordinating）

通过 **Shared TaskList + Mailbox**，Agent Teams 实现了去中心化协调：
- TaskList 用文件锁保证并发安全
- Mailbox 实现代理间直接通信
- teammate 可以自认领任务，无需 Lead 分配

这与 DDIA 中 "Shared-Nothing 架构" 的理念一致：每个节点独立工作，通过共享数据结构协调，而非通过中心节点路由。

### 3. 上下文隔离（Context Isolation）

每个 teammate 有独立的上下文窗口，不继承 Lead 的对话历史。这确保了：
- 探索结果不污染主会话
- teammate 之间的工作互不干扰
- 每个 teammate 从项目上下文（CLAUDE.md）开始，而非 Lead 的对话历史

### 4. 安全边界（Security Boundaries）

- 权限冒泡到用户，teammate 不能代替用户做决定
- 中继的批准声明不可信
- 工具沙箱限制 teammate 能力
- 防止嵌套团队和领导权转移

### 5. 渐进式复杂性（Progressive Complexity）

从 Subagent（单向委派）→ Fork（共享 cache）→ Agent Teams（双向通信），Claude Code 提供了**三层递进的并行模型**，让用户根据任务的协作需求选择最合适的层次。

---

## 参考文献

### 官方文档

1. [Agent Teams 官方文档](https://code.claude.com/docs/en/agent-teams.md) — Claude Code Agent Teams 完整指南，包含架构、使用场景、最佳实践和已知限制
2. [Subagents 官方文档](https://code.claude.com/docs/en/sub-agents.md) — 子代理创建、配置和调用指南
3. [Agents 并行工作](https://code.claude.com/docs/en/agents.md) — Subagent、Agent View、Agent Teams 和 Dynamic Workflows 对比
4. [Hooks 官方文档](https://code.claude.com/docs/en/hooks.md) — 钩子系统，包括 SubagentStart/SubagentStop/TeammateIdle 等事件
5. [Permissions 官方文档](https://code.claude.com/docs/en/permissions.md) — 权限系统，包括权限模式和工具级规则
6. [Worktrees 官方文档](https://code.claude.com/docs/en/worktrees.md) — Git 隔离与并行会话
7. [Features Overview](https://code.claude.com/docs/en/features-overview.md) — CLAUDE.md、Skills、Subagents、Hooks、MCP、Plugins 对比

### 源代码（oboard/claude-code-rev）

8. [AgentTool.tsx](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/AgentTool.tsx) — Agent tool 核心定义，输入 schema，调度逻辑（233KB）
9. [spawnMultiAgent.ts](https://github.com/oboard/claude-code-rev/blob/main/src/tools/shared/spawnMultiAgent.ts) — 团队生成共享模块，tmux/iTerm2 后端（35KB）
10. [inProcessRunner.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/inProcessRunner.ts) — 进程内 teammate 执行器，AsyncLocalStorage 隔离
11. [spawnInProcess.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/spawnInProcess.ts) — 进程内 teammate 生成，AbortController 管理
12. [InProcessTeammateTask.tsx](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx) — 进程内 teammate 任务管理（16KB）
13. [types.ts](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/InProcessTeammateTask/types.ts) — TeammateIdentity 和 InProcessTeammateTaskState 定义（4.3KB）
14. [teamHelpers.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/teamHelpers.ts) — 团队文件读写，TeamFile 格式，sanitizeName（21KB）
15. [teammateMailbox.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateMailbox.ts) — 代理间消息系统，权限冒泡（33KB）
16. [teammateContext.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/teammateContext.ts) — teammate 上下文管理，AsyncLocalStorage
17. [RemoteAgentTask.tsx](https://github.com/oboard/claude-code-rev/blob/main/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx) — 远程代理任务，五种远程类型（126KB）
18. [TeamCreateTool.ts](https://github.com/oboard/claude-code-rev/blob/main/src/tools/TeamCreateTool/TeamCreateTool.ts) — 团队创建工具，唯一名称生成
19. [framework.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/task/framework.ts) — 任务框架，registerTask/updateTaskState/evictTerminalTask
20. [loadAgentsDir.ts](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/loadAgentsDir.ts) — 代理定义加载，Zod schema，递归扫描（26KB）
21. [forkSubagent.ts](https://github.com/oboard/claude-code-rev/blob/main/src/tools/AgentTool/forkSubagent.ts) — Fork 子代理机制，cache 优化（24KB）
22. [agentContext.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/agentContext.ts) — 代理上下文工具函数
23. [teammateLayoutManager.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/teammateLayoutManager.ts) — 分屏布局管理，颜色分配
24. [leaderPermissionBridge.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/leaderPermissionBridge.ts) — Lead 权限桥，工具使用确认队列
25. [permissionSync.ts](https://github.com/oboard/claude-code-rev/blob/main/src/utils/swarm/permissionSync.ts) — 权限同步，权限请求创建和发送

---

*本文基于 Claude Code v2.1.x 源码与官方文档编写。源码参考来自 [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) 仓库。*
