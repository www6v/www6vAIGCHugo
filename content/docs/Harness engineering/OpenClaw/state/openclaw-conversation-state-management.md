# OpenClaw 对话状态管理：从 Session 存储到 Context Engine 的源码级拆解

> **基于 OpenClaw v2026.4.21 源码深度分析**
> 作者：小伟 | 2026-04-30

---

## 导读

一个 LLM Agent 能"记住"什么、如何管理对话上下文、如何在有限的 context window 中塞入最有价值的信息，这些是 Agent 系统的核心挑战。OpenClaw 的对话状态管理不是简单的"把历史消息传给模型"，而是一个分层架构：从持久化存储 → Session 生命周期 → Session Manager → Context Engine → Compaction → Transcript Repair，每一层各司其职。

本文将从源码级别完整拆解这一架构。

---

## 1. 总览：对话状态管理架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    OpenClaw 对话状态管理体系                                │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  存储层：持久化                                                       │ │
│  │  Session Store (sessions.json) + JSONL Transcript (session.jsonl)   │ │
│  │  + Session Store Cache + Session Write Lock                          │ │
│  └────────────────────────────────┬───────────────────────────────────┘ │
│                                   │                                     │
│  ┌────────────────────────────────▼───────────────────────────────────┐ │
│  │  会话层：Session 生命周期                                              │ │
│  │  Session 创建 / 路由 / 分组 / 维护（prune/rotate/maintenance）      │ │
│  └────────────────────────────────┬───────────────────────────────────┘ │
│                                   │                                     │
│  ┌────────────────────────────────▼───────────────────────────────────┐ │
│  │  引擎层：Session Manager + Context Engine                             │ │
│  │  SessionManager (pi-coding-agent) + Context Engine (assemble/ingest)│ │
│  │  + Context Window Guard + Token Estimation                          │ │
│  └────────────────────────────────┬───────────────────────────────────┘ │
│                                   │                                     │
│  ┌────────────────────────────────▼───────────────────────────────────┐ │
│  │  维护层：Compaction + Maintenance                                     │ │
│  │  Context Engine Maintenance + Compaction + Session Truncation       │ │
│  │  + Transcript Rewrite + Tool Result Truncation                      │ │
│  └────────────────────────────────┬───────────────────────────────────┘ │
│                                   │                                     │
│  ┌────────────────────────────────▼───────────────────────────────────┐ │
│  │  安全层：Repair + Guard                                               │ │
│  │  Session File Repair + Tool Result Context Guard                    │ │
│  │  + Tool Use/Result Pairing Sanitize + Thinking Block Repair         │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 存储层：Session Store + JSONL Transcript

### 2.1 双存储结构

OpenClaw 使用**两种文件格式**来管理对话状态：

| 文件 | 路径 | 格式 | 用途 |
|---|---|---|---|
| `sessions.json` | `~/.openclaw/agents/<agentId>/sessions/sessions.json` | JSON | Session 元数据索引（轻量） |
| `<sessionId>.jsonl` | `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl` | JSONL | 完整对话记录（逐行 JSON） |

**源码**：[paths.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/paths.ts) / [store.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store.ts)

### 2.2 Session Store 加载

**源码**：[store-load.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-load.ts)

```typescript
// src/config/sessions/store-load.ts

// 从文件系统加载 sessions.json
async function loadSessionStore(storePath, options) {
  // 1. 优先从缓存读取
  if (isSessionStoreCacheEnabled()) {
    const cached = readSessionStoreCache(storePath);
    if (cached) return cached;
  }

  // 2. 从磁盘读取
  const content = await fs.readFile(storePath, "utf-8");
  const store = JSON.parse(content);

  // 3. 清理过期条目
  pruneStaleEntries(store);

  // 4. 限制最大条目数
  capEntryCount(store, config.maxEntries);

  // 5. 写入缓存
  writeSessionStoreCache(storePath, store);

  return store;
}
```

### 2.3 Session 维护策略

**源码**：[store-maintenance.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-maintenance.ts)

```typescript
// src/config/sessions/store-maintenance.ts

// 默认维护策略
const DEFAULT_SESSION_PRUNE_AFTER_MS = 720 * 60 * 60 * 1000;  // 30 天
const DEFAULT_SESSION_MAX_ENTRIES = 500;                        // 最大 500 条
const DEFAULT_SESSION_ROTATE_BYTES = 10485760;                 // 10 MB 轮转
const DEFAULT_SESSION_MAINTENANCE_MODE = "enforce";             // 强制模式
const DEFAULT_SESSION_DISK_BUDGET_HIGH_WATER_RATIO = 0.8;      // 高水位 80%

function resolveMaintenanceConfigFromInput(maintenance) {
  return {
    mode: maintenance?.mode ?? DEFAULT_SESSION_MAINTENANCE_MODE,
    pruneAfterMs: resolvePruneAfterMs(maintenance),
    maxEntries: maintenance?.maxEntries ?? DEFAULT_SESSION_MAX_ENTRIES,
    rotateBytes: resolveRotateBytes(maintenance),
    maxDiskBytes: resolveMaxDiskBytes(maintenance),
    highWaterBytes: resolveHighWaterBytes(maintenance, maxDiskBytes)
  };
}

// 清理过期条目
function pruneStaleEntries(store, overrideMaxAgeMs) {
  const cutoffMs = Date.now() - maxAgeMs;
  for (const [key, entry] of Object.entries(store)) {
    if (entry?.updatedAt != null && entry.updatedAt < cutoffMs) {
      delete store[key];  // 直接删除
    }
  }
}
```

### 2.4 High-Water 磁盘预算策略

```typescript
// High-Water 策略：写入超过 cap 时批量清理，而非每次都触发完整清理
function resolveHighWaterBytes(maintenance, maxDiskBytes) {
  if (maxDiskBytes == null) return null;
  return Math.floor(maxDiskBytes * 0.8);  // 80% 水位线
}
```

### 2.5 Session 文件轮转

**源码**：[store-load.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-load.ts)

```typescript
// src/config/sessions/store-load.ts

// 当 session 文件超过 rotateBytes 时自动轮转
function rotateSessionFile(sessionFile, rotateBytes) {
  const stat = fs.statSync(sessionFile);
  if (stat.size > rotateBytes) {
    // 归档旧文件
    const archivePath = `${sessionFile}.${Date.now()}.archive`;
    fs.renameSync(sessionFile, archivePath);
  }
}
```

### 2.6 Session Store 缓存

**源码**：[store-cache.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-cache.ts)

```typescript
// Session Store 缓存机制
function isSessionStoreCacheEnabled() { ... }
function readSessionStoreCache(storePath) { ... }
function writeSessionStoreCache(storePath, store) { ... }
function setSerializedSessionStore(key, value) { ... }
function getSerializedSessionStore(key) { ... }

// TTL 缓存
function resolveCacheTtlMs() { ... }  // 默认 TTL
function createExpiringMapCache() { ... }  // 过期 Map 缓存
```

### 2.7 Session Write Lock

**源码**：[session-write-lock.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-write-lock.ts)

```typescript
// src/agents/session-write-lock.ts

// 默认参数
const DEFAULT_STALE_MS          = 1_800_000;   // 30 分钟
const DEFAULT_MAX_HOLD_MS       =   300_000;   // 5 分钟
const DEFAULT_WATCHDOG_INTERVAL =    60_000;   // 1 分钟
const DEFAULT_TIMEOUT_GRACE_MS  =   120_000;   // 2 分钟

// 获取 session 写锁
async function acquireSessionWriteLock(params) {
  const lockPath = `${sessionFile}.lock`;

  // 1. 可重入检查
  const held = HELD_LOCKS.get(normalizedSessionFile);
  if (allowReentrant && held) {
    held.count += 1;
    return { release: () => releaseHeldLock(...) };
  }

  // 2. 重试循环
  while (Date.now() - startedAt < timeoutMs) {
    try {
      // wx 标志：文件存在则失败
      const handle = await fs.open(lockPath, "wx");
      await handle.writeFile(JSON.stringify({
        pid: process.pid,
        createdAt: new Date().toISOString(),
        starttime: getProcessStartTime(process.pid)
      }));
      return { release };
    } catch (err) {
      if (err.code !== "EEXIST") throw err;
      // 检查 stale lock
      if (await shouldReclaimContendedLockFile(lockPath)) {
        await fs.rm(lockPath, { force: true });
        continue;
      }
      // 指数退避
      await sleep(delay);
    }
  }
}

// Watchdog 后台清理
function ensureWatchdogStarted(intervalMs) {
  setInterval(() => runLockWatchdogCheck(), intervalMs);
}
```

---

## 3. 会话层：Session 生命周期

### 3.1 Session Key 路由

**源码**：[session-key.ts](https://github.com/openclaw/openclaw/blob/main/src/session-key.ts) / [store.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store.ts)

```typescript
// src/config/sessions/store.ts

// Group Session Key 解析
function resolveGroupSessionKey(ctx) {
  const from = ctx.From;
  const chatType = ctx.ChatType;
  const provider = ctx.Provider;

  // 构建 key: provider:kind:id
  return {
    key: `${provider}:${kind}:${finalId}`,
    channel: provider,
    id: finalId,
    chatType: kind
  };
}

// Group 显示名称构建
function buildGroupDisplayName(params) {
  const providerKey = params.provider ?? "group";
  const detail = params.space || params.subject || "";
  const token = normalizeGroupLabel(detail || fallbackId);
  return token ? `${providerKey}:${token}` : providerKey;
}
```

### 3.2 Session 元数据合并

**源码**：[types.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/types.ts)

```typescript
// src/config/sessions/types.ts

// 合并 session 条目（保留活跃度信息）
function mergeSessionEntryPreserveActivity(existing, next) {
  const merged = mergeSessionEntry(existing, next);
  // 保留活跃度相关字段
  return merged;
}
```

### 3.3 Session 交付上下文

**源码**：[delivery-context.shared.ts](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/delivery-context.shared.ts)

```typescript
// Session 交付字段归一化
function normalizeSessionDeliveryFields(entry) {
  return {
    ...normalizeDeliveryContext(entry),
    ...normalizeSessionDeliveryFields(entry)
  };
}

// 从 session 构建交付上下文
function deliveryContextFromSession(entry) {
  return {
    channel: entry.channel,
    accountId: entry.accountId,
    targetId: entry.targetId,
    // ...
  };
}
```

### 3.4 Session 元数据

**源码**：[session-meta.ts](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-meta.ts)

```typescript
// Session 元数据管理
// - sessionStartedAt    → daily reset 基准
// - lastInteractionAt   → idle reset 基准
// - updatedAt           → 列表/修剪用，不影响 freshness
```

### 3.5 Session 身份标识

**源码**：[session-identity.ts](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-identity.ts) / [session-identifiers.ts](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-identifiers.ts)

```typescript
// Session 身份标识管理
// - 唯一 ID
// - 关联 Agent
// - 关联 Channel
// - 关联 Conversation
```

---

## 4. 引擎层：Session Manager + Context Engine

### 4.1 Session Manager 初始化

OpenClaw 使用 `@mariozechner/pi-coding-agent` 库中的 `SessionManager` 作为核心会话管理器。

**源码**：[session-manager-init.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-manager-init.ts) / [session-manager-cache.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-manager-cache.ts)

```typescript
// SessionManager 初始化流程
// 1. 从 JSONL 文件加载 transcript
// 2. 构建 SessionManager 实例
// 3. 缓存 SessionManager（减少重复加载）
// 4. 预 warm session 文件
```

### 4.2 Session Manager 缓存

**源码**：[session-manager-cache.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-manager-cache.ts)

```typescript
// Session Manager 缓存机制
// - 缓存 key: sessionKey
// - 缓存 value: SessionManager 实例
// - 过期策略: TTL + 主动失效
// - 访问追踪: trackSessionManagerAccess()
```

### 4.3 Context Engine 生命周期钩子

**源码**：[context-engine-maintenance.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/context-engine-maintenance.ts)

```typescript
// src/agents/pi-embedded-runner/context-engine-maintenance.ts

// 安装 Context Engine 循环钩子
function installContextEngineLoopHook(params) {
  const { contextEngine, sessionId, sessionKey, sessionFile, tokenBudget, modelId } = params;

  return {
    async afterTurn(messages) {
      // 1. 调用 contextEngine.afterTurn()
      if (typeof contextEngine.afterTurn === "function") {
        await contextEngine.afterTurn({ messages, sessionKey });
      }

      // 2. Ingest 新消息到 Context Engine
      if (newMessages.length > 0) {
        if (typeof contextEngine.ingestBatch === "function") {
          await contextEngine.ingestBatch({ messages: newMessages });
        } else {
          for (const message of newMessages) {
            await contextEngine.ingest({ message });
          }
        }
      }

      // 3. Assemble 完整上下文
      const assembled = await contextEngine.assemble({
        tokenBudget,
        modelId
      });

      return assembled;
    }
  };
}
```

**核心流程**：

```
每轮对话结束后：
1. afterTurn()  → 通知 Context Engine 本轮完成
2. ingest()     → 将新消息摄入 Context Engine
3. assemble()   → 组装下一轮所需的上下文
```

### 4.4 Context Engine 维护（Maintenance）

**源码**：[context-engine-maintenance.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/context-engine-maintenance.ts)

```typescript
// src/agents/pi-embedded-runner/context-engine-maintenance.ts

// 延迟维护任务常量
const TURN_MAINTENANCE_TASK_KIND = "context_engine_turn_maintenance";
const TURN_MAINTENANCE_TASK_LABEL = "Context engine turn maintenance";
const TURN_MAINTENANCE_WAIT_POLL_MS = 100;           // 等待轮询间隔
const TURN_MAINTENANCE_LONG_WAIT_MS = 10000;         // 长等待超时

// 执行上下文引擎维护
async function executeContextEngineMaintenance(params) {
  if (typeof params.contextEngine.maintain !== "function") return;

  const result = await params.contextEngine.maintain({
    sessionId: params.sessionId,
    sessionKey: params.sessionKey,
    sessionFile: params.sessionFile,
    runtimeContext: buildContextEngineMaintenanceRuntimeContext({
      sessionManager: params.sessionManager,
      rewriteTranscriptEntries: async (request) => {
        // 重写 transcript 条目
        return rewriteTranscriptEntriesInSessionManager({
          sessionManager: params.sessionManager,
          replacements: request.replacements
        });
      }
    })
  });

  if (result.changed) {
    log.info(`[context-engine] maintenance(${params.reason}) changed transcript`);
  }
  return result;
}

// 运行维护
async function runContextEngineMaintenance(params) {
  // 如果配置为后台模式，调度延迟任务
  if (params.reason === "turn" && params.contextEngine.info.turnMaintenanceMode === "background") {
    scheduleDeferredTurnMaintenance(params);
    return;
  }
  // 否则同步执行
  return executeContextEngineMaintenance(params);
}
```

### 4.5 延迟维护调度

```typescript
// 延迟维护调度器
function scheduleDeferredTurnMaintenance(params) {
  const sessionKey = normalizeSessionKey(params.sessionKey);
  if (!sessionKey) return;

  // 检查是否已有活跃维护任务
  const activeRun = activeDeferredTurnMaintenanceRuns.get(sessionKey);
  if (activeRun) {
    activeRun.rerunRequested = true;  // 标记需要重新运行
    return;
  }

  // 创建任务描述符
  const task = buildTurnMaintenanceTaskDescriptor({ sessionKey });

  // 入队到 session lane
  enqueueCommandInLane(
    resolveDeferredTurnMaintenanceLane(sessionKey),
    async () => runDeferredTurnMaintenanceWorker({
      contextEngine: params.contextEngine,
      sessionId: params.sessionId,
      sessionKey,
      sessionFile: params.sessionFile,
      sessionManager: params.sessionManager,
      runId: task.runId
    })
  );
}
```

### 4.6 延迟维护 Worker

```typescript
// 延迟维护 Worker
async function runDeferredTurnMaintenanceWorker(params) {
  const shutdownAbort = createDeferredTurnMaintenanceAbortSignal();

  try {
    const sessionLane = resolveSessionLane(params.sessionKey);

    // 1. 等待 session lane 空闲
    for (;;) {
      while (getQueueSize(sessionLane) > 0) {
        await sleepWithAbort(TURN_MAINTENANCE_WAIT_POLL_MS, shutdownAbort.abortSignal);
      }
      await Promise.resolve();
      if (getQueueSize(sessionLane) === 0) break;
    }

    // 2. 开始维护
    startTaskRunByRunId({
      runId: params.runId,
      progressSummary: "Running deferred maintenance."
    });

    const result = await executeContextEngineMaintenance({
      contextEngine: params.contextEngine,
      executionMode: "background"
    });

    // 3. 完成
    completeTaskRunByRunId({
      runId: params.runId,
      progressSummary: result?.changed
        ? "Deferred maintenance completed with transcript changes."
        : "Deferred maintenance completed."
    });

  } catch (err) {
    // 错误处理
    failTaskRunByRunId({ runId: params.runId, error: reason });
  } finally {
    shutdownAbort.dispose();
  }
}
```

---

## 5. 维护层：Compaction + Transcript 管理

### 5.1 Transcript Rewrite

**源码**：[transcript-rewrite.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/transcript-rewrite.ts)

```typescript
// Transcript 重写机制

// 在 SessionManager 中重写条目
async function rewriteTranscriptEntriesInSessionManager(params) {
  const { sessionManager, replacements } = params;
  // 根据 replacements 替换 transcript 中的条目
  // 返回重写的条目数和释放的字节数
}

// 在 Session 文件中重写条目
async function rewriteTranscriptEntriesInSessionFile(params) {
  const { sessionFile, sessionId, sessionKey, request } = params;
  // 直接操作 JSONL 文件
}
```

### 5.2 Tool Result 截断

**源码**：[tool-result-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-truncation.ts)

```typescript
// src/agents/pi-embedded-runner/tool-result-truncation.ts

// 动态计算 tool result 最大字符数
function resolveLiveToolResultMaxChars(params) {
  // 根据当前上下文窗口剩余空间动态调整
  // 避免单个工具结果吃掉太多 token
}

// 截断 SessionManager 中过大的 tool results
function truncateOversizedToolResultsInSessionManager(params) {
  const { sessionManager, contextWindowTokens } = params;

  const branch = sessionManager.getBranch();
  const messages = branch.messages;

  // 估算每个消息的字符数
  // 截断超过限制的 tool results
  // 重写 transcript
  const rewriteResult = rewriteTranscriptEntriesInSessionManager({
    sessionManager,
    replacements
  });
}
```

### 5.3 字符估算

**源码**：[session-file-repair.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-file-repair.ts)

```typescript
// 字符估算常量
const IMAGE_CHAR_ESTIMATE = 8000;  // 图片估算 8000 字符

// 估算消息字符数（带缓存）
function estimateMessageCharsCached(msg, cache) {
  const hit = cache.get(msg);
  if (hit !== undefined) return hit;
  const estimated = estimateMessageChars(msg);
  cache.set(msg, estimated);
  return estimated;
}

// 不同类型消息的字符估算
function estimateMessageChars(msg) {
  if (msg.role === "user") {
    // 用户消息：直接计算
    return typeof content === "string" ? content.length : estimateContentBlockChars(content);
  }
  if (msg.role === "assistant") {
    // 助手消息：text + thinking + toolCall
    let chars = 0;
    for (const block of content) {
      if (block.type === "text") chars += block.text.length;
      else if (block.type === "thinking") chars += block.thinking.length;
      else if (block.type === "toolCall") chars += JSON.stringify(block.arguments ?? {}).length;
    }
    return chars;
  }
  if (isToolResultMessage(msg)) {
    // Tool result：加权计算（4/2 倍）
    const chars = estimateContentBlockChars(getToolResultContent(msg));
    return Math.max(chars, Math.ceil(chars * 2));
  }
}

// 上下文总字符估算（带缓存）
function estimateContextChars(messages, cache) {
  return messages.reduce((sum, msg) => sum + estimateMessageCharsCached(msg, cache), 0);
}
```

### 5.4 Compaction 常量

**源码**：[pi-compaction-constants.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-compaction-constants.ts)

```typescript
// Compaction 相关常量
// - 最大 active transcript 字节
// - 最大 total transcript 字节
// - Compaction 触发阈值
```

### 5.5 Compaction 超时

**源码**：[compaction-safety-timeout.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compaction-safety-timeout.ts)

```typescript
// Compaction 安全超时
// - 防止 compaction 过程卡死
// - 超时后自动重试
```

### 5.6 Compaction Retry + Aggregate Timeout

**源码**：[compaction-retry-aggregate-timeout.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts)

```typescript
// Compaction 失败自动重试
// - 指数退避
// - 聚合超时计算（防止重试总耗时超过 agent timeout）
```

### 5.7 Session Truncation

**源码**：[session-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-truncation.ts)

```typescript
// Compaction 后物理截断 transcript 文件
// 截断规则：
// 1. 保留 session header
// 2. 保留非 message 条目（custom, model_change, thinking_level_change, compaction 等）
// 3. 截断旧消息，保留最近的消息
// 4. 确保 tool_use 和 tool_result 成对出现
```

---

## 6. 安全层：Repair + Guard

### 6.1 Session 文件自修复

**源码**：[session-file-repair.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-file-repair.ts)

```typescript
// src/agents/session-file-repair.ts

// Session 文件自修复
async function repairSessionFileIfNeeded(params) {
  const sessionFile = params.sessionFile.trim();

  // 1. 读取文件
  const content = await fs.readFile(sessionFile, "utf-8");
  const lines = content.split(/\r?\n/);

  // 2. 逐行解析 JSON
  const entries = [];
  let droppedLines = 0;
  for (const line of lines) {
    if (!line.trim()) continue;
    try {
      const entry = JSON.parse(line);
      entries.push(entry);
    } catch {
      droppedLines += 1;  // 记录丢弃行数
    }
  }

  // 3. 验证 session header
  if (!isSessionHeader(entries[0])) {
    return { repaired: false, reason: "invalid session header" };
  }

  // 4. 如果有损坏的行，执行修复
  if (droppedLines > 0) {
    const cleaned = entries.map(entry => JSON.stringify(entry)).join("\n");

    // 备份原文件
    const backupPath = `${sessionFile}.bak-${process.pid}-${Date.now()}`;
    await fs.writeFile(backupPath, content);

    // 原子写入
    const tmpPath = `${sessionFile}.repair-${process.pid}-${Date.now()}.tmp`;
    await fs.writeFile(tmpPath, cleaned);
    await fs.rename(tmpPath, sessionFile);

    return {
      repaired: true,
      droppedLines,
      backupPath
    };
  }

  return { repaired: false, droppedLines: 0 };
}
```

### 6.2 Transcript Repair

**源码**：[session-transcript-repair.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-transcript-repair.ts)

```typescript
// src/agents/session-transcript-repair.ts

// 检测 thinking 块
function isThinkingLikeBlock(block) {
  return block?.type === "thinking" || block?.type === "redacted_thinking";
}

// 检测原始工具调用块
function isRawToolCallBlock(block) {
  return typeof block?.type === "string" &&
    (block.type === "toolCall" || block.type === "toolUse" || block.type === "functionCall");
}

// 检测工具调用 ID
function hasToolCallId(block) {
  return typeof block?.id === "string" && block.id.trim().length > 0;
}

// 清理工具调用块
function sanitizeToolCallBlock(block) {
  const normalizedName = normalizeLowercaseStringOrEmpty(block.name?.trim());

  // 特殊处理 sessions_spawn（红附件）
  if (normalizedName === "sessions_spawn") {
    return {
      ...block,
      arguments: redactSessionsSpawnAttachmentsArgs(block.arguments),
      input: redactSessionsSpawnAttachmentsArgs(block.input)
    };
  }
  return block;
}

// 检测 replay 安全的 thinking 助手回合
function isReplaySafeThinkingAssistantTurn(content, allowedToolNames) {
  let sawToolCall = false;
  // 检查是否包含不允许的工具调用
  // 检查 tool_call 是否有对应的 tool_result
  return true;  // 安全
}

// 清理工具调用输入
function sanitizeToolCallInputs(block) { ... }

// 创建缺失的 tool result
function makeMissingToolResult(block) { ... }

// 修复 tool use / result 配对
function repairToolUseResultPairing(messages) { ... }

// 清理 tool use / result 配对
function sanitizeToolUseResultPairing(messages) { ... }
```

### 6.3 Tool Result Context Guard

**源码**：[tool-result-context-guard.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-context-guard.ts)

```typescript
// src/agents/pi-embedded-runner/tool-result-context-guard.ts

// 安装 Tool Result Context Guard
function installToolResultContextGuard(params) {
  // 确保 tool result 不包含敏感信息
  // 过滤 session 管理工具的结果
  // 防止 agent-to-agent 通信的信息泄露
}
```

---

## 7. 完整对话状态管理序列图

```
┌────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌────────────┐
│  用户消息   │  │ Session Store│  │Session Manager│  │Context     │  │  LLM API   │
│            │  │ + JSONL      │  │ (pi-coding)  │  │  Engine    │  │            │
└─────┬──────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └─────┬──────┘
      │                │                  │                │               │
      │ 新消息到达      │                  │                │               │
      │───────────────▶│                  │                │               │
      │                │                  │                │               │
      │                │ repairIfNeeded() │                │               │
      │                │─────────────────▶│                │               │
      │                │                  │                │               │
      │                │ 加载 JSONL        │                │               │
      │                │▶────────────────▶│                │               │
      │                │                  │                │               │
      │                │ 初始化 SessionManager             │               │
      │                │─────────────────────────────────▶│               │
      │                │                  │                │               │
      │                │ 获取写锁          │                │               │
      │                │ acquireLock()     │                │               │
      │                │◀─────────────────│                │               │
      │                │                  │                │               │
      │                │                  │ assemble()     │               │
      │                │                  │───────────────▶│               │
      │                │                  │                │               │
      │                │                  │ 上下文组装      │               │
      │                │                  │◀───────────────│               │
      │                │                  │                │               │
      │                │                  │ 调用 LLM API   │               │
      │                │                  │───────────────────────────────▶│
      │                │                  │                │               │
      │                │                  │◀───────────────────────────────│
      │                │                  │  模型响应       │               │
      │                │                  │                │               │
      │                │                  │ ingest()       │               │
      │                │                  │───────────────▶│               │
      │                │                  │                │               │
      │                │                  │ afterTurn()    │               │
      │                │                  │───────────────▶│               │
      │                │                  │                │               │
      │                │ 写入 transcript   │                │               │
      │                │◀─────────────────│                │               │
      │                │                  │                │               │
      │                │ 释放写锁          │                │               │
      │                │ releaseLock()     │                │               │
      │                │◀─────────────────│                │               │
      │                │                  │                │               │
      │                │                  │ 触发 maintenance               │
      │                │                  │───────────────▶│               │
      │                │                  │                │               │
      │                │                  │ maintain()     │               │
      │                │                  │◀───────────────│               │
      │                │                  │                │               │
      │                │                  │ 需要 compaction?                │
      │                │                  │───Yes──▶ 压缩                   │
      │                │                  │                │               │
      │                │ 更新 sessions.json               │               │
      │                │◀─────────────────│                │               │
      │                │                  │                │               │
      │                │                  │ prune/rotate?  │               │
      │                │                  │───Yes──▶ 维护                   │
```

---

## 8. 关键设计模式

### 8.1 JSONL 格式的优势

```typescript
// JSONL（JSON Lines）格式的优势：
// 1. 逐行追加，无需重写整个文件
// 2. 损坏时只需丢弃坏行，不影响其他数据
// 3. 流式读取，支持大文件
// 4. 自修复能力强
```

### 8.2 三层会话存储

```
Store 层（sessions.json）:
  - 轻量元数据索引
  - 快速查询所有 session
  - 维护策略执行

SessionManager 层（内存）:
  - 完整对话上下文
  - Token 估算
  - 消息组装

JSONL 层（磁盘）:
  - 持久化对话记录
  - 自修复能力
  - 原子写入
```

### 8.3 延迟维护模式

```typescript
// Context Engine 维护支持两种模式：
// 1. 同步模式：立即执行，适用于小维护
// 2. 延迟模式：等待 session lane 空闲后执行，适用于大维护

// 延迟模式的优势：
// - 不阻塞用户交互
// - 等待其他操作完成后执行
// - 支持 abort（进程关闭时取消）
```

### 8.4 自修复能力

```
Session 文件的自修复流程：
1. 读取文件 → 逐行解析
2. 丢弃无法解析的行（记录数量）
3. 验证 session header
4. 原子写入修复后的内容
5. 保留原文件备份
```

### 8.5 Tool Use/Result 配对保护

```
对话中 tool_use 和 tool_result 必须成对出现：
- 有 tool_use 无 tool_result → 创建缺失的 result
- 有 tool_result 无 tool_use → 丢弃孤立的 result
- tool_call_id 不匹配 → 修复 ID
```

### 8.6 Token 预算控制

```
Context Engine 组装上下文时：
- tokenBudget 控制总 token 数
- estimateContextChars() 估算字符数
- 根据模型上下文窗口动态调整
- 优先保留最近的消息
- Compaction 后截断旧消息
```

---

## 9. 配置示例

### 9.1 Session 维护配置

```json5
{
  "session": {
    "maintenance": {
      "mode": "enforce",       // warn | enforce
      "pruneAfter": "30d",     // 30 天后清理
      "maxEntries": 500,       // 最多 500 条
      "rotateBytes": "10MB"    // 10MB 轮转
    }
  }
}
```

### 9.2 Context 配置

```json5
{
  "context": {
    "tokenBudget": 128000,     // 总 token 预算
    "maxActiveTranscriptBytes": 1048576,  // 1MB
    "maxTotalTranscriptBytes": 10485760   // 10MB
  }
}
```

---

## 10. 源码模块索引

| 模块 | 功能 | GitHub 链接 |
|---|---|---|
| **存储层** | | |
| `config/sessions/store.ts` | Session Store 核心 | [查看](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store.ts) |
| `config/sessions/store-load.ts` | Store 加载 + 维护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-load.ts) |
| `config/sessions/store-cache.ts` | Store 缓存 | [查看](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/store-cache.ts) |
| `config/sessions/types.ts` | Session 类型定义 | [查看](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/types.ts) |
| `config/sessions/delivery-context.shared.ts` | 交付上下文 | [查看](https://github.com/openclaw/openclaw/blob/main/src/config/sessions/delivery-context.shared.ts) |
| `session-write-lock.ts` | Session 写锁 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/session-write-lock.ts) |
| **会话层** | | |
| `session-key.ts` | Session Key 管理 | [查看](https://github.com/openclaw/openclaw/blob/main/src/session-key.ts) |
| `acp/runtime/session-meta.ts` | Session 元数据 | [查看](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-meta.ts) |
| `acp/runtime/session-identity.ts` | Session 身份 | [查看](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-identity.ts) |
| `acp/runtime/session-identifiers.ts` | Session 标识符 | [查看](https://github.com/openclaw/openclaw/blob/main/src/acp/runtime/session-identifiers.ts) |
| **引擎层** | | |
| `pi-embedded-runner/session-manager-init.ts` | Session Manager 初始化 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-manager-init.ts) |
| `pi-embedded-runner/session-manager-cache.ts` | Session Manager 缓存 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-manager-cache.ts) |
| `pi-embedded-runner/context-engine-maintenance.ts` | Context Engine 维护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/context-engine-maintenance.ts) |
| `pi-embedded-runner/install-context-engine-loop-hook` | Context Engine 钩子 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/context-engine-maintenance.ts) |
| **维护层** | | |
| `pi-embedded-runner/transcript-rewrite.ts` | Transcript 重写 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/transcript-rewrite.ts) |
| `pi-embedded-runner/tool-result-truncation.ts` | Tool Result 截断 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-truncation.ts) |
| `pi-embedded-runner/session-truncation.ts` | Session 截断 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/session-truncation.ts) |
| `pi-compaction-constants.ts` | Compaction 常量 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-compaction-constants.ts) |
| `pi-embedded-runner/compaction-safety-timeout.ts` | Compaction 超时 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/compaction-safety-timeout.ts) |
| `pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts` | Compaction 重试 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/compaction-retry-aggregate-timeout.ts) |
| **安全层** | | |
| `session-file-repair.ts` | Session 文件自修复 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/session-file-repair.ts) |
| `session-transcript-repair.ts` | Transcript 修复 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/session-transcript-repair.ts) |
| `pi-embedded-runner/tool-result-context-guard.ts` | Tool Result 保护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-context-guard.ts) |

---

## 11. 参考资料

| 资源 | 链接 |
|---|---|
| OpenClaw 文档 | https://docs.openclaw.ai |
| GitHub 仓库 | https://github.com/openclaw/openclaw |
| DeepWiki（AI 源码导读） | https://deepwiki.com/openclaw/openclaw |
| Session 管理 | https://docs.openclaw.ai/concepts/session.md |
| Compaction | https://docs.openclaw.ai/concepts/compaction.md |
| Context Engine | https://docs.openclaw.ai/concepts/context-engine.md |

---

_本文基于 OpenClaw v2026.4.21 源码分析，所有代码引用均来自官方 GitHub 仓库。_
