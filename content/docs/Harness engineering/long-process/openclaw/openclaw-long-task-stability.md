# OpenClaw 如何保证长程任务的稳定性：从架构到源码实现

> OpenClaw v2026.4.21，源码版本分析。所有结论来自官方文档与 `openclaw` npm 包源码（TypeScript → ESM bundle），源码路径标注格式为 `src/<path>`。

---

## 概览

OpenClaw 不是简单地把一个 LLM API 调用丢出去就完了。它在 Gateway 层构建了一整套 **任务生命周期管理 + 可靠性机制**，涵盖任务账本、超时保护、队列串行化、文件级写锁、上下文压缩、进程守护等多层防护。

---

## 1. Background Tasks — 任务生命周期账本

### 1.1 状态机

所有脱离主会话的工作（ACP runs、subagent spawns、cron 执行、CLI agent 命令）都会创建一条 **Task 记录**，经历完整状态机：

```
queued → running → succeeded / failed / timed_out / cancelled / lost
```

### 1.2 源码实现

**任务执行器**（`src/tasks/task-executor.ts`）定义了 6 个核心操作：

```typescript
// src/tasks/task-executor.ts
function createQueuedTaskRun(params) { /* → status: "queued" */ }
function createRunningTaskRun(params) { /* → status: "running" */ }
function startTaskRunByRunId(params)  { /* markTaskRunningByRunId */ }
function recordTaskRunProgressByRunId(params) { /* 进度更新 */ }
function completeTaskRunByRunId(params) { /* → status: "succeeded" */ }
function failTaskRunByRunId(params)     { /* → status: "failed" */ }
```

**分离任务运行时**（`src/tasks/detached-task-runtime.ts`）封装了这些操作，并提供 **可插拔的生命周期钩子**：

```typescript
// src/tasks/detached-task-runtime.ts
const DETACHED_TASK_RECOVERY_WARN_MS = 5_000;

const DEFAULT_DETACHED_TASK_LIFECYCLE_RUNTIME = {
  createQueuedTaskRun,
  createRunningTaskRun,
  startTaskRunByRunId,
  recordTaskRunProgressByRunId,
  completeTaskRunByRunId,
  failTaskRunByRunId,
  setDetachedTaskDeliveryStatusByRunId,
  cancelDetachedTaskRunById
};

async function tryRecoverTaskBeforeMarkLost(params) {
  // 5 秒超时保护：恢复钩子不能阻塞太久
  const startedAt = Date.now();
  const result = await hook(params);
  const elapsedMs = Date.now() - startedAt;
  if (elapsedMs >= 5_000) log.warn("Detached task recovery hook was slow");
  ...
}
```

`tryRecoverTaskBeforeMarkLost` 是关键函数 —— 当 backing session 消失时，先尝试通过 durable cron run history 等恢复任务状态，只有确认无法恢复才标记为 `lost`。

**任务注册表持久化**（`src/tasks/task-registry.paths.ts`）使用 **SQLite** 而非 JSON 文件：

```typescript
// src/tasks/task-registry.paths.ts
function resolveTaskRegistrySqlitePath() {
  return path.join(resolveTaskRegistryDir(), "runs.sqlite");
}
```

SQLite 保证了任务状态的 **原子写入** 和 **并发安全**，比 JSON 文件更适合高并发场景。

### 1.3 终端状态判定

```typescript
// src/tasks/task-executor-policy.ts
function isTerminalTaskStatus(status) {
  return status === "succeeded" || status === "failed" ||
         status === "timed_out" || status === "cancelled" ||
         status === "lost";
}
```

---

## 2. Push-based 完成通知

### 2.1 两种投递路径

- **Direct delivery**：直接发到原 channel，保留 thread/topic 路由
- **Session-queued delivery**：投递失败时，作为系统事件入队

### 2.2 Heartbeat Wake 机制（`src/infra/heartbeat-wake.ts`）

这是整个推送体系的核心。任务完成会 **立即触发 heartbeat wake**：

```typescript
// src/infra/heartbeat-wake.ts
const DEFAULT_COALESCE_MS = 250;   // 合并窗口 250ms
const DEFAULT_RETRY_MS    = 1_000;  // 失败重试 1s

// 优先级体系：retry > interval > action(wake/exec-event/hook) > default
const REASON_PRIORITY = {
  RETRY:   0,
  INTERVAL: 1,
  DEFAULT:  2,
  ACTION:   3
};
```

**关键设计：**

1. **合并（coalesce）**：多个 wake 请求在 250ms 窗口内合并为一次 heartbeat，防止抖动
2. **优先级**：retry 优先级最高，action wake 次之，interval 最低
3. **重试机制**：如果 heartbeat 执行时发现有 `requests-in-flight`，自动以 1s 间隔重试
4. **生成号保护**（`handlerGeneration`）：防止旧 runner 的清理逻辑误杀新 handler

```typescript
function requestHeartbeatNow(opts) {
  queuePendingWakeReason(opts);
  schedule(opts?.coalesceMs ?? 250, "normal");
}
```

---

## 3. 多层超时保护

### 3.1 超时层级

| 超时层 | 默认值 | 源码位置 |
|---|---|---|
| Agent runtime timeout | 48h (172800s) | `agents.defaults.timeoutSeconds` |
| Model idle timeout | 120s | provider 级 idle watchdog |
| Cron outer timeout | cron 自配 | cron scheduler 外环 |
| Exec process timeout | 30m (1800s) | `tools.exec.timeoutSec` |
| Stuck session watchdog | 可配 | `diagnostics.stuckSessionWarnMs` |

### 3.2 源码实现

**Agent Loop 中的超时处理**（[src/agents/pi-embedded-runner/](https://github.com/openclaw/openclaw/tree/main/src/agents/pi-embedded-runner/)）：

```typescript
// src/agents/pi-embedded-runner/ （bundle 中可见的关键逻辑）
// 空闲超时：模型长时间不输出 chunk
if (idleTimedOut) {
  const timeoutText = "The model did not produce a response before the LLM idle timeout.";
  // 触发 abort
}

// 总超时：整个 agent run 超时
if (timedOut && !timedOutDuringCompaction) {
  // 在 compaction 期间的超时不计入失败，支持自动重试
}
```

**关键设计**：compaction 期间的超时被单独标记（`timedOutDuringCompaction`），不会导致整个 run 失败，而是触发自动 retry。这保证了在长对话压缩时不会因为额外延迟而误判超时。

---

## 4. 命令队列 + Session Write Lock — 防并发冲突

### 4.1 Command Queue（[src/process/command-queue.ts](https://github.com/openclaw/openclaw/blob/main/src/process/command-queue.ts)）

**Lane 模型**：每个 session 有独立的 lane，不同 lane 可并发：

```typescript
// src/process/command-queue.ts

// 全局单例状态（通过 Symbol.for 跨 chunk 共享）
const COMMAND_QUEUE_STATE_KEY = Symbol.for("openclaw.commandQueueState");

function getLaneState(lane) {
  return {
    lane,
    queue: [],             // 等待队列
    activeTaskIds: new Set(),  // 活跃任务 ID 集合
    maxConcurrent: 1,      // 默认串行
    draining: false,       // 排空中
    generation: 0          // 代际号，防旧任务干扰
  };
}
```

**核心入队逻辑**：

```typescript
function enqueueCommandInLane(lane, task, opts) {
  if (getQueueState().gatewayDraining)
    return Promise.reject(new GatewayDrainingError());

  return new Promise((resolve, reject) => {
    state.queue.push({
      task: () => task(),
      resolve, reject,
      enqueuedAt: Date.now(),
      warnAfterMs: opts?.warnAfterMs ?? 2_000,
      onWait: opts?.onWait  // 排队过久时触发 warn 日志
    });
    drainLane(cleaned);
  });
}
```

**排空（pump）机制**：

```typescript
function drainLane(lane) {
  const pump = () => {
    while (state.activeTaskIds.size < state.maxConcurrent && state.queue.length > 0) {
      const entry = state.queue.shift();
      const taskId = getQueueState().nextTaskId++;
      state.activeTaskIds.add(taskId);

      (async () => {
        try {
          const result = await entry.task();
          if (completeTask(state, taskId, taskGeneration)) {
            notifyActiveTaskWaiters();
            pump();  // 递归排空下一个
          }
          entry.resolve(result);
        } catch (err) {
          // 非预期的 lane 失败才会记录 error
          if (!isExpectedNonErrorLaneFailure(err))
            diagnosticLogger.error(`lane task error: lane=${lane}`);
          entry.reject(err);
        }
      })();
    }
  };
  pump();
}
```

**关键设计：**
- **Generation 机制**：`resetAllLanes()` 时 bump `generation`，旧任务的完成回调因 generation 不匹配被忽略，防止 SIGUSR1 热重启后 stale task ID 阻塞新任务
- **GatewayDrainingError**：重启时标记 draining，新任务直接拒绝而非静默杀死
- **waitForActiveTasks**：支持等待所有活跃任务完成（用于优雅关闭）

### 4.2 Session Write Lock（[src/agents/session-write-lock.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-write-lock.ts)）

文件级写锁，保护 session transcript 的一致性：

```typescript
// src/agents/session-write-lock.ts
const DEFAULT_STALE_MS          = 1_800_000;  // 30 分钟
const DEFAULT_MAX_HOLD_MS       =   300_000;  // 5 分钟
const DEFAULT_WATCHDOG_INTERVAL =    60_000;  // 1 分钟
const DEFAULT_TIMEOUT_GRACE_MS  =   120_000;  // 2 分钟
```

**锁获取逻辑**：

```typescript
async function acquireSessionWriteLock(params) {
  const lockPath = `${normalizedSessionFile}.lock`;

  // 1. 可重入检查（默认允许）
  const held = HELD_LOCKS.get(normalizedSessionFile);
  if (allowReentrant && held) {
    held.count += 1;
    return { release: () => releaseHeldLock(...) };
  }

  // 2. 重试循环（直到 timeout）
  const startedAt = Date.now();
  while (Date.now() - startedAt < timeoutMs) {
    try {
      // wx 标志：文件存在则失败，实现互斥
      const handle = await fs.open(lockPath, "wx");
      await handle.writeFile(JSON.stringify({
        pid: process.pid,
        createdAt: new Date().toISOString(),
        starttime: getProcessStartTime(process.pid)
      }), "utf8");
      HELD_LOCKS.set(normalizedSessionFile, { count: 1, handle, ... });
      return { release: () => releaseHeldLock(...) };
    } catch (err) {
      if (err.code !== "EEXIST") throw err;
      // 3. 检查是否为 stale lock（进程已死 or 超时）
      if (await shouldReclaimContendedLockFile(lockPath, inspected)) {
        await fs.rm(lockPath, { force: true });
        continue;
      }
      // 4. 指数退避
      const delay = Math.min(1_000, 50 * attempt, remainingMs);
      await new Promise(r => setTimeout(r, delay));
    }
  }
  throw new Error(`session file locked (timeout ${timeoutMs}ms)`);
}
```

**Watchdog 定时器**（后台周期性扫描）：

```typescript
function ensureWatchdogStarted(intervalMs) {
  watchdogState.timer = setInterval(() => {
    runLockWatchdogCheck();  // 释放超时锁
  }, intervalMs);  // 默认 60s
}

async function runLockWatchdogCheck() {
  for (const [sessionFile, held] of HELD_LOCKS) {
    const heldForMs = Date.now() - held.acquiredAt;
    if (heldForMs > held.maxHoldMs) {
      // 强制释放超时锁
      await releaseHeldLock(sessionFile, held, { force: true });
    }
  }
}
```

**进程终止时自动释放**：

```typescript
function registerCleanupHandlers() {
  process.on("exit", releaseAllLocksSync);
  for (const signal of ["SIGINT", "SIGTERM", "SIGQUIT", "SIGABRT"])
    process.on(signal, () => handleTerminationSignal(signal));
}
```

**Stale lock 检测**（防 PID 回收）：

```typescript
function inspectLockPayload(payload, staleMs, nowMs) {
  const pidAlive = pid !== null && isPidAlive(pid);
  const pidRecycled = pidAlive && storedStarttime !== getProcessStartTime(pid);

  const staleReasons = [];
  if (!pidAlive) staleReasons.push("dead-pid");
  if (pidRecycled) staleReasons.push("recycled-pid");  // 关键！
  if (ageMs > staleMs) staleReasons.push("too-old");

  return { stale: staleReasons.length > 0, staleReasons };
}
```

通过记录进程启动时间（`starttime`），即使 PID 被新进程复用，也能准确识别 stale lock。

### 4.3 KeyedAsyncQueue（[src/plugin-sdk/keyed-async-queue.ts](https://github.com/openclaw/openclaw/blob/main/src/plugin-sdk/keyed-async-queue.ts)）

更轻量级的 per-key 串行化原语：

```typescript
// src/plugin-sdk/keyed-async-queue.ts
function enqueueKeyedTask(params) {
  const current = (params.tails.get(params.key) ?? Promise.resolve())
    .catch(() => void 0)
    .then(params.task)
    .finally(() => params.hooks?.onSettle?.());

  const tail = current.then(() => void 0, () => void 0);
  params.tails.set(params.key, tail);

  // 自动清理
  const cleanup = () => {
    if (params.tails.get(params.key) === tail) params.tails.delete(params.key);
  };
  tail.then(cleanup, cleanup);

  return current;
}
```

通过链式 `.then()` 实现同一 key 的任务自动排队，不同 key 并发执行。

---

## 5. Compaction — 上下文窗口管理

### 5.1 源码模块分布

Compaction 在 [src/agents/pi-embedded-runner/](https://github.com/openclaw/openclaw/tree/main/src/agents/pi-embedded-runner/) 下有多个模块协作：

| 模块 | 功能 | GitHub 链接 |
|---|---|---|
| `compact.ts` | 主 compaction 入口 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact.ts) |
| `compact.runtime.ts` | 运行时封装 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact.runtime.ts) |
| `compact.queued.ts` | 带 lane 队列的 compaction | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact.queued.ts) |
| `compaction-runtime-context.ts` | 上下文构建 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compaction-runtime-context.ts) |
| `compaction-hooks.ts` | 生命周期钩子 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compaction-hooks.ts) |
| `compaction-safety-timeout.ts` | 安全超时 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compaction-safety-timeout.ts) |
| `run/preemptive-compaction.ts` | 预压缩 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/preemptive-compaction.ts) |
| `session-truncation.ts` | compact 后截断 transcript | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-truncation.ts) |

### 5.2 预压缩（Preemptive Compaction）

在 agent run 开始前检查 transcript 文件大小（[run/preemptive-compaction.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/preemptive-compaction.ts)）：

```typescript
// src/agents/pi-embedded-runner/run/preemptive-compaction.ts
// 当 active JSONL 文件达到 maxActiveTranscriptBytes 阈值时，
// 在实际 LLM 调用前先触发 compaction
```

### 5.3 Compaction Retry + Aggregate Timeout（[run/compaction-retry-aggregate-timeout.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts)）

```typescript
// src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts
// compaction 失败时自动 retry，并聚合超时计算
// 防止多次 retry 导致总耗时超过 agent timeout
```

### 5.4 Session Truncation（[session-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-truncation.ts)）

Compact 后 transcript 文件仍然包含所有历史条目。`session-truncation.ts` 实现了物理截断：

```typescript
// src/agents/pi-embedded-runner/session-truncation.ts
// 截断规则：
// 1. 保留 session header
// 2. 保留非 message 条目（custom, model_change, thinking_level_change, compaction 等）
// 3. 保留未压缩分支的条目
// 4. 保留从 firstKeptEntryId 开始的"未压缩尾部"
// 5. 被移除条目的子条目重新绑定到最近的保留祖先
```

### 5.5 Compaction 失败分类（[compact-reasons.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact-reasons.ts)）

```typescript
// src/agents/pi-embedded-runner/compact-reasons.ts
function classifyCompactionReason(reason) {
  if (text.includes("nothing to compact")) return "no_compactable_entries";
  if (text.includes("below threshold"))     return "below_threshold";
  if (text.includes("timed out"))           return "timeout";
  if (text.includes("400") || text.includes("401")) return "provider_error_4xx";
  if (text.includes("500") || text.includes("502")) return "provider_error_5xx";
  ...
}
```

### 5.6 Compaction 生命周期钩子

```typescript
// before_compaction / after_compaction 钩子
runBeforeCompactionHooks(...);   // 压缩前
runAfterCompactionHooks(...);    // 压缩后
runPostCompactionSideEffects(...); // 后置副作用
```

Hook 可用于在压缩前触发 memory flush，或在压缩后发送通知。

---

## 6. Session 生命周期管理

### 6.1 存储结构

- **Store**：`~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **Transcripts**：`~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

`sessions.json` 分离了三种时间戳：

```
sessionStartedAt   → daily reset 基准
lastInteractionAt  → idle reset 基准
updatedAt          → 列表/修剪用，不影响 freshness
```

这意味着 heartbeat、cron、exec 等系统事件写入的元数据 **不会延长 session 的活跃寿命**。

### 6.2 Session Write Lock 的集成

每次 agent run 开始前：

```typescript
// 1. 获取 session write lock
const lock = await acquireSessionWriteLock({
  sessionFile,
  timeoutMs,
  maxHoldMs: resolveSessionLockMaxHoldFromTimeout({ timeoutMs })
});

// 2. 打开 SessionManager
const sessionManager = SessionManager.open(sessionFile);

// 3. run 结束后释放锁
await lock.release();
```

### 6.3 Session Maintenance

```json5
{
  "session": {
    "maintenance": {
      "mode": "enforce",      // warn | enforce
      "pruneAfter": "30d",
      "maxEntries": 500
    }
  }
}
```

生产环境下，Gateway 使用 **high-water buffer** 策略：写入超过 cap 时批量清理，而非每次 isolated cron session 都触发完整清理。

---

## 7. Gateway Daemon + 进程守护

### 7.1 子进程桥接

启动长驻子进程时，必须附加 **child-process bridge**：

- 转发终止信号到子进程
- 子进程 exit/error 时自动 detach listeners
- 防止 systemd restart 时产生 orphan 进程

### 7.2 Gateway Lock（[src/gateway/gateway-lock.ts](https://github.com/openclaw/openclaw/blob/main/src/gateway/gateway-lock.ts)）

保证 **一台主机只有一个 Gateway 实例**：

```typescript
// 使用文件锁防止多个 Gateway 同时运行
```

### 7.3 重启安全

```typescript
// SIGUSR1 热重启时的关键步骤：
// 1. markGatewayDraining() → 新任务直接拒绝
// 2. waitForActiveTasks(timeout) → 等待活跃任务完成
// 3. resetAllLanes() → bump generation，清 stale task IDs
// 4. 新进程接管
```

---

## 8. 清理机制

### 8.1 Isolated Cron Run 清理

```typescript
// 清理路径：
// 1. 关闭 tracked browser tabs
// 2. 处置 MCP stdio 子进程 (disposeSessionMcpRuntime)
// 3. 清理 cron session 所有权
```

### 8.2 Exec Background Session TTL

```json5
{
  "tools": {
    "exec": {
      "cleanupMs": 1_800_000,    // 30 分钟
      "notifyOnExit": true,       // 退出时通知
      "notifyOnExitEmptySuccess": false
    }
  }
}
```

---

## 架构全图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Gateway Daemon                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Command Queue (lanes)                    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │   │
│  │  │ Session A│  │ Session B│  │ Global   │   串行化 +     │   │
│  │  │ Lane     │  │ Lane     │  │ Lane     │   并发控制     │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘               │   │
│  └───────┼─────────────┼─────────────┼──────────────────────┘   │
│          │             │             │                           │
│  ┌───────▼─────────────▼─────────────▼──────────────────────┐   │
│  │              Session Write Lock (file-level .lock)       │   │
│  │  - .lock sidecar file + PID + starttime                  │   │
│  │  - Watchdog timer (60s) → 强制释放超时锁                  │   │
│  │  - Exit/signal handlers → 同步释放所有锁                   │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │                    Agent Loop (PI)                        │   │
│  │  Prompt → Model → Tool → Stream → Reply                   │   │
│  │  - Idle timeout watchdog (120s)                           │   │
│  │  - Total timeout (48h)                                    │   │
│  │  - Auto-compaction on context overflow                    │   │
│  │  - Model failover (provider fallback chain)               │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │              Task Registry (SQLite)                       │   │
│  │  queued → running → terminal                              │   │
│  │  - 5min grace before "lost"                               │   │
│  │  - tryRecoverTaskBeforeMarkLost hook                      │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│  ┌──────────────────────────▼───────────────────────────────┐   │
│  │              Heartbeat Wake System                        │   │
│  │  - Coalesce window: 250ms                                 │   │
│  │  - Priority: retry > action > interval > default          │   │
│  │  - Generation-based handler protection                    │   │
│  │  - Push → channel delivery or session-queued              │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 总结

OpenClaw 的长程任务稳定性不是靠某一个银弹，而是多层防御体系的组合：

| 层级 | 机制 | 解决什么问题 |
|---|---|---|
| **任务层** | SQLite 持久化 + 状态机 + recovery hook | 任务丢失、状态不一致 |
| **通知层** | Push-based heartbeat wake + coalesce | 轮询延迟、通知抖动 |
| **超时层** | 4 层超时（total/idle/cron/exec）+ compaction 超时豁免 | 卡死、无响应 |
| **并发层** | Per-session lane + global lane + generation | 竞态、热重启后阻塞 |
| **写入层** | 文件级写锁 + watchdog + PID/starttime 检测 | Transcript 损坏、stale lock |
| **上下文层** | Auto-compaction + preemptive + successor transcripts + truncation | Context overflow、文件膨胀 |
| **进程层** | Child-process bridge + gateway lock + draining | Orphan 进程、双实例 |
| **清理层** | TTL + best-effort browser/MCP cleanup | 资源泄漏 |

**核心设计哲学**：push-based（不轮询）、防丢（SQLite + recovery）、防卡（多层超时 + watchdog）、防泄漏（TTL + cleanup）、防竞态（lane + write lock + generation）。

---

## 参考资料

| 文档 | 链接 |
|---|---|
| Gateway 架构 | https://docs.openclaw.ai/concepts/architecture.md |
| Agent Loop（执行循环） | https://docs.openclaw.ai/concepts/agent-loop.md |
| Agent Runtimes（运行时） | https://docs.openclaw.ai/concepts/agent-runtimes.md |
| Background Tasks（任务账本） | https://docs.openclaw.ai/automation/tasks.md |
| Cron Jobs（定时任务） | https://docs.openclaw.ai/automation/cron-jobs.md |
| Session Management（会话管理） | https://docs.openclaw.ai/concepts/session.md |
| Compaction（上下文压缩） | https://docs.openclaw.ai/concepts/compaction.md |
| Background Exec + Process | https://docs.openclaw.ai/gateway/background-process.md |
| Gateway 源码仓库 | https://github.com/openclaw/openclaw |
| DeepWiki（AI 源码导读） | https://deepwiki.com/openclaw/openclaw |

| 源码模块 | 功能 | GitHub 链接 |
|---|---|---|
| `src/tasks/task-executor.ts` | 任务生命周期操作 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/tasks/task-executor.ts) |
| `src/tasks/detached-task-runtime.ts` | 分离任务运行时 + recovery hook | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/tasks/detached-task-runtime.ts) |
| `src/tasks/task-registry.paths.ts` | SQLite 持久化路径 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/tasks/task-registry.paths.ts) |
| `src/tasks/task-executor-policy.ts` | 状态判定 + 通知策略 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/tasks/task-executor-policy.ts) |
| `src/infra/heartbeat-wake.ts` | Heartbeat 唤醒 + 合并 + 优先级 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/infra/heartbeat-wake.ts) |
| `src/process/command-queue.ts` | Lane 队列 + generation + draining | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/process/command-queue.ts) |
| `src/agents/session-write-lock.ts` | 文件级写锁 + watchdog + PID 检测 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/session-write-lock.ts) |
| `src/plugin-sdk/keyed-async-queue.ts` | Per-key 串行化原语 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/plugin-sdk/keyed-async-queue.ts) |
| `src/plugin-sdk/file-lock.ts` | 通用文件锁 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/plugin-sdk/file-lock.ts) |
| `src/agents/pi-embedded-runner/compact.queued.ts` | 带队列的 compaction | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact.queued.ts) |
| `src/agents/pi-embedded-runner/compact-reasons.ts` | Compaction 失败分类 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compact-reasons.ts) |
| `src/agents/pi-embedded-runner/session-truncation.ts` | Compact 后 transcript 截断 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-truncation.ts) |
| `src/agents/pi-embedded-runner/run/preemptive-compaction.ts` | 预压缩 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/preemptive-compaction.ts) |
| `src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts` | Compaction 重试超时 | [查看源码](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts) |
