# Ralph Loop、OpenClaw 与 Hermes Agent 长程任务运行机制深度对比

> 基于源码分析（2026年6月），聚焦三者如何实现长程任务（long-running tasks）的启动、执行、容错与终止。

---

## 一、项目概览

| 维度 | Ralph Loop (snarktank/ralph) | OpenClaw (openclaw/openclaw) | Hermes Agent (NousResearch/hermes-agent) |
|------|-----|------|-----|
| 语言 | Bash + Markdown/JSON | TypeScript (Node.js) | Python |
| Stars | ~19.9k | ~376k | ~180k |
| 核心思想 | **文件驱动**的自主循环 | **事件驱动**的 Gateway + ACP 架构 | **会话驱动**的 Agent 循环 + 调度器 |
| 长程任务模式 | PRD 驱动的迭代编码循环 | 嵌入式 Agent Loop + 子代理注册表 + 命令队列 | 对话循环 (conversation_loop) + Cron 调度器 |
| 状态持久化 | `progress.txt` + `prd.json` | 内存 + SQLite session store + 磁盘状态 | SQLite session DB + `~/.hermes/cron/jobs.json` |

---

## 二、Ralph Loop：极简的文件驱动循环

### 2.1 核心架构

Ralph 是三者中最简单的设计——本质上是一个 **Bash `for` 循环**，每次迭代启动一个独立的 AI 编码 Agent（amp 或 claude），让其完成 `prd.json` 中的一条用户故事。

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  prd.json    │────▶│ ralph.sh     │────▶│ AI Agent    │
│  (PRD状态)   │     │ (for循环)     │     │ (amp/claude)│
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                │
                                  ┌─────────────┴─────────────┐
                                  │ 检查 <promise>COMPLETE    │
                                  │ 更新 prd.json passes:true  │
                                  │ 追加 progress.txt          │
                                  └───────────────────────────┘
```

### 2.2 源码剖析：`ralph.sh`

```bash
# 核心循环（摘自 ralph.sh）
for i in $(seq 1 $MAX_ITERATIONS); do
  # 启动 AI 编码 Agent
  if [[ "$TOOL" == "amp" ]]; then
    OUTPUT=$(cat "$SCRIPT_DIR/prompt.md" | amp --dangerously-allow-all 2>&1) || true
  else
    OUTPUT=$(claude --dangerously-skip-permissions --print < "$CLAUDE.md" 2>&1) || true
  fi
  
  # 检查完成信号
  if echo "$OUTPUT" | grep -q "<promise>COMPLETE</promise>"; then
    echo "Ralph completes all tasks!"
    exit 0
  fi
  
  echo "Iteration $i complete. Continuing..."
  sleep 2
done

echo "Ralph reached max iterations ($MAX_ITERATIONS) without completing all tasks."
exit 1
```

**关键设计点：**

1. **无状态迭代**：每次迭代启动一个全新的 Agent 实例，无上下文记忆。Agent 通过读取 `progress.txt` 中的 "Codebase Patterns" 部分获取前序经验。

2. **完成信号协议**：Agent 在回复中包含 `<promise>COMPLETE</promise>` 标记，Bash 脚本通过 `grep` 检测。这是一种极简的信号量机制。

3. **PRD 状态机**：`prd.json` 充当任务队列和状态机：
   ```json
   {
     "userStories": [
       {"id": "US-001", "priority": 1, "passes": false, ...},
       {"id": "US-002", "priority": 2, "passes": false, ...}
     ]
   }
   ```
   每次迭代 Agent 找到最高优先级的 `passes: false` 故事，实现后设为 `true`。

4. **进度归档**：切换分支时自动归档旧进度到 `archive/YYYY-MM-DD-feature/`。

### 2.3 Agent 指令：`prompt.md`

Prompt 定义了 Agent 的完整工作流：
1. 读取 `prd.json` 和 `progress.txt`
2. 切换到正确分支
3. 选取最高优先级未完成的用户故事
4. 实现该故事
5. 运行质量检查（typecheck, lint, test）
6. 更新 `AGENTS.md`（发现可复用模式时）
7. 提交代码
8. 更新 `prd.json` 设置 `passes: true`
9. 追加 `progress.txt` 进度日志
10. 检查是否全部完成 → 输出 `<promise>COMPLETE</promise>`

### 2.4 关键约束

从 `skills/ralph/SKILL.md` 中可以看到：

> **Each story must be completable in ONE Ralph iteration (one context window).**

Ralph 每轮启动新的 Agent，没有上下文记忆。如果故事太大，LLM 会在上下文耗尽前无法完成，产出破坏性代码。因此故事粒度控制是 Ralph 成功的关键。

### 2.5 容错机制

| 容错手段 | 实现方式 |
|---------|---------|
| 最大迭代限制 | `MAX_ITERATIONS` 参数（默认 10） |
| Agent 崩溃恢复 | `|| true` 忽略失败，继续下一轮 |
| 上下文限制 | 通过 `progress.txt` 的模式总结传递知识 |
| 分支隔离 | 每个 PRD 使用独立 git 分支 |

### 2.6 局限性

- **无并发**：只能串行执行，一轮一次
- **无状态恢复**：如果脚本被中断（Ctrl+C），没有恢复机制
- **依赖外部 Agent**：需要 amp 或 claude CLI 可用
- **简单完成检测**：grep 匹配可能被误触发

---

## 三、OpenClaw：事件驱动的 Gateway + ACP 架构

### 3.1 核心架构

OpenClaw 的长程任务机制是其最复杂的部分，涉及 **五层多代理交互**（来自源码注释）：

| 层级 | 机制 | 方向 | 核心源文件 |
|------|------|------|----------|
| ① 路由 | Binding + resolve-route | Channel → Agent | `src/routing/session-key.ts` |
| ② 会话 | Session key + isolated storage | Agent-internal | `src/agents/agent-scope.ts` |
| ③ 委派 | sessions_spawn + subagent | Parent → Child | `src/agents/subagent-spawn.ts`, `subagent-announce.ts` |
| ④ 对等 | sessions_send + A2A flow | Agent ↔ Agent | `sessions-send-tool.ts` |
| ⑤ 调度 | Command queue + lanes | Global 并发控制 | `src/agents/lanes.ts` |

```
┌──────────────────────────────────────────────────────────────┐
│                      Gateway (Node.js)                        │
│                                                              │
│  ┌─────────┐  ┌───────────────┐  ┌──────────────────────┐   │
│  │ Channels │─▶│ Command Queue  │─▶│  ACP Session Manager │   │
│  │ (TG/Disc│  │  + Lanes       │  │  (turn-runner)        │   │
│  │ ord/...)│  └───────────────┘  └──────────┬───────────┘   │
│  └─────────┘                                │               │
│                                             ▼               │
│                               ┌─────────────────────────┐   │
│                               │  Subagent Registry      │   │
│                               │  (lifecycle/delivery/   │   │
│                               │   steer/orphan recovery)│   │
│                               └─────────────────────────┘   │
│                                             │               │
│                                             ▼               │
│                               ┌─────────────────────────┐   │
│                               │  ACP Runtime (LLM API)   │   │
│                               │  (Claude/Codex/etc.)     │   │
│                               └─────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Command Queue + Lanes：并发控制核心

**源码：`src/process/command-queue.ts`**

OpenClaw 使用 **Lane（车道）** 模型来序列化命令执行：

```typescript
type LaneState = {
  lane: string;
  queue: QueueEntry[];
  activeTaskIds: Set<number>;
  maxConcurrent: number;
  draining: boolean;
  generation: number;
};
```

**Lane 类型**（`src/agents/lanes.ts`）：

| Lane | 用途 |
|------|------|
| `main` | 默认入站聊天 Lane |
| `subagent` | 子代理执行 |
| `cron` | Cron 任务外层 Lane |
| `cron-nested` | Cron 内部 Agent 工作 |
| `nested:<sessionKey>` | 每个会话的嵌套工作 |

**队列排水（drain）机制**：

```typescript
function drainLane(lane: string) {
  const state = getLaneState(lane);
  state.draining = true;

  const pump = () => {
    while (state.activeTaskIds.size < state.maxConcurrent && state.queue.length > 0) {
      const entry = state.queue.shift();
      const taskId = getQueueState().nextTaskId++;
      state.activeTaskIds.add(taskId);
      void (async () => {
        const result = await runQueueEntryTask(lane, entry);
        completeTask(state, taskId, taskGeneration);
        notifyActiveTaskWaiters();
        pump();  // 递归排水
      })();
    }
  };
  pump();
}
```

**关键设计**：
- 每个 Lane 独立队列，互不阻塞
- 支持优先级排序（foreground/background/normal）
- 支持任务超时（`taskTimeoutMs` + 滑动超时计算）
- Drain 模式用于优雅重启

### 3.3 ACP Turn Runner：单次执行单元

**源码：`src/acp/control-plane/manager.turn-runner.ts`**

`runManagerTurn` 是 OpenClaw 执行单次 LLM 交互的核心函数：

```typescript
export async function runManagerTurn(params: {...}): Promise<void> {
  const turnStartedAt = Date.now();
  
  // 1. 记录后台任务上下文
  if (taskContext) {
    createBackgroundTaskRecord(taskContext, turnStartedAt);
  }
  
  // 2. 后端故障转移（failover）
  for (let backendIdx = 0; backendIdx < candidateBackends.length; backendIdx++) {
    for (let attempt = 0; attempt < 2; attempt++) {
      try {
        // 3. 确保运行时句柄
        const ensured = await params.ensureRuntimeHandle({...});
        
        // 4. 消费 ACP 事件流
        const turnPromise = consumeAcpTurnStream({
          runtime, turn: {handle, text, signal: combinedSignal},
          onOutputEvent: (event) => {
            sawTurnOutput = true;
            // 进度追踪
            taskProgressSummary = appendBackgroundTaskProgressSummary(
              taskProgressSummary, event.text
            );
          },
        });
        
        // 5. 带超时的等待
        const turnOutcome = await awaitTurnWithTimeout({
          turnPromise, timeoutMs: turnTimeoutMs + GRACE_MS,
          onTimeout: async () => { cleanupTimedOutTurn(...) },
        });
        
        // 6. 记录完成
        markBackgroundTaskTerminal(taskContext.runId, {status: "succeeded"});
        return;
      } catch (error) {
        // 故障恢复逻辑
        retryFreshHandle = await prepareFreshManagerRuntimeHandleRetry({...});
        if (retryFreshHandle) continue;
      }
    }
  }
}
```

**关键设计**：
- **后端故障转移**：支持主后端 + 多个 fallback 后端的自动切换
- **重试机制**：每个后端最多重试 2 次
- **超时管理**：可配置的 turn 超时 + 优雅期
- **进度追踪**：后台任务的文本输出累积为进度摘要

### 3.4 Subagent Registry：子代理生命周期管理

**源码：`src/agents/subagent-registry.ts`**

这是 OpenClaw 最复杂的部分之一——管理子代理的完整生命周期：

```
Subagent 生命周期：
  spawned → running → [paused/yielded] → completed/failed → announced
```

**关键常量**（源码中可见）：

```typescript
const SUBAGENT_ANNOUNCE_TIMEOUT_MS = 120_000;      // 2分钟完成通知超时
const LIFECYCLE_ERROR_RETRY_GRACE_MS = 15_000;     // 错误重试宽限期
const STALE_ACTIVE_SUBAGENT_GRACE_MS = 60_000;      // 活跃子代理变宽限期
const SUSPENDED_DELIVERY_HARD_CAP = 50;             // 挂起交付硬上限
const SESSION_RUN_TTL_MS = 5 * 60_000;              // 会话运行 TTL
```

**核心功能**：
1. **注册与持久化**：子代理状态写入磁盘（SQLite + JSON）
2. **交付重试**：完成通知失败时自动重试（最多 `MAX_ANNOUNCE_RETRY_COUNT` 次）
3. **孤儿恢复**：进程重启后恢复孤立的子代理记录
4. **转向（Steering）**：支持运行时注入指导到活跃子代理
5. **深度限制**：防止无限嵌套委派

### 3.5 子代理完成生命周期（subagent-announce.ts）

```
1. 子代理完成 → readSubagentOutput() 读取输出
2. 构建内部事件上下文
3. 尝试唤醒父级运行（如果父级仍活跃）
4. 回退到请求者 Agent 交接
5. 通过 Gateway send 方法交付到请求者聊天频道
6. 使用 SILENT_REPLY_TOKEN 抑制重复通知
```

### 3.6 A2A（Agent-to-Agent）Ping-Pong 流程

```
- maxPingPongTurns 控制最大轮数（默认 2）
- buildAgentToAgentReplyContext() 包装每轮角色/上下文
- runAgentStep() 在专用 Lane 上执行每轮
- 终止条件：isReplySkip() / isNonDeliverableSessionsReply() / 达到最大轮数
- 最终通知通过 Gateway send 方法交付
```

---

## 四、Hermes Agent：会话驱动的 Agent 循环 + Cron 调度器

### 4.1 核心架构

Hermes Agent 的长程任务分为两个层次：

```
层次 1：对话循环（conversation_loop.py）
  ┌─────────────────────────────────────────────┐
  │  while (api_calls < max_iterations          │
  │     && iteration_budget.remaining > 0) {    │
  │    1. 检查中断                              │
  │    2. 消耗迭代预算                           │
  │    3. 构建 API 消息                         │
  │    4. 调用 LLM API                          │
  │    5. 解析响应                               │
  │    6a. 有工具调用 → 执行 → 追加消息 → 继续  │
  │    6b. 有文本响应 → 完成 → 返回             │
  │    6c. 截断 → 重试 / continue              │
  │    6d. 空响应 → 重试 (最多3次) / 故障转移   │
  │  }                                          │
  │  预算耗尽 → 要求模型总结                     │
  └─────────────────────────────────────────────┘

层次 2：Cron 调度器（cron/scheduler.py + cron/jobs.py）
  ┌─────────────────────────────────────────────┐
  │  tick() 每 60 秒执行：                       │
  │    1. 获取到期的 Cron 任务                   │
  │    2. 并行池提交并行任务                     │
  │    3. 顺序池提交序列化任务                   │
  │    4. 每个任务启动独立 Agent 运行            │
  │    5. 输出保存到 ~/.hermes/cron/output/      │
  │    6. 交付结果到指定平台                     │
  └─────────────────────────────────────────────┘
```

### 4.2 Conversation Loop：核心迭代机制

**源码：`agent/conversation_loop.py`**

```python
def run_conversation(agent, user_message, ...):
    # 初始化迭代预算（默认 90 次）
    agent.iteration_budget = IterationBudget(agent.max_iterations)
    
    # 主循环
    while (api_call_count < agent.max_iterations 
           and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
        
        # 1. 中断检查
        if agent._interrupt_requested:
            interrupted = True
            break
        
        api_call_count += 1
        
        # 2. 消耗预算（或触发优雅退出）
        if agent._budget_grace_call:
            agent._budget_grace_call = False
        elif not agent.iteration_budget.consume():
            break  # 预算耗尽
        
        # 3. /steer 指导注入（运行时中途注入文本）
        _pre_api_steer = agent._drain_pending_steer()
        if _pre_api_steer:
            # 注入到最近的 tool 消息中
        
        # 4. 调用 LLM API
        response = await agent._call_api(api_messages, ...)
        
        # 5. 解析响应
        if 有工具调用:
            # 执行工具
            tool_results = agent._execute_tool_calls(tool_calls, ...)
            # 追加 tool 结果到消息列表
            messages.extend(tool_results)
            continue  # 继续循环
        
        elif 有文本响应:
            final_response = response
            break  # 完成
        
        elif 截断响应:
            # 重试，要求继续
            length_continue_retries += 1
            continue
        
        elif 空响应:
            # 最多重试 3 次
            if agent._empty_content_retries < 3:
                agent._empty_content_retries += 1
                continue
            # 否则尝试故障转移
            if agent._fallback_chain:
                agent._try_activate_fallback()
                continue
    
    # 预算耗尽 → 要求模型总结（去除工具调用能力）
    if budget_exhausted:
        final_response = agent._handle_max_iterations(messages, api_call_count)
```

**关键设计**：
- **IterationBudget**（`agent/iteration_budget.py`）：线程安全的迭代计数器
  ```python
  class IterationBudget:
      def __init__(self, max_total):
          self.max_total = max_total  # 默认 90
          self._used = 0
          self._lock = threading.Lock()
      
      def consume(self) -> bool:
          with self._lock:
              if self._used >= self.max_total:
                  return False
              self._used += 1
              return True
      
      def refund(self):
          # execute_code 的迭代可以退还预算
          with self._lock:
              if self._used > 0:
                  self._used -= 1
  ```

- **预算宽限期（Grace Call）**：预算耗尽后给模型最后一次机会
- **中断机制**：支持用户发送新消息时中断当前运行
- **Steer 机制**：运行时中途注入指导文本到活跃对话
- **故障转移链**：主模型失败时自动切换到备用模型

### 4.3 Context Compression：长对话管理

Hermes 使用上下文压缩来管理长对话：

```python
# 预飞行压缩检查
if compressor.should_compress(_preflight_tokens):
    for _pass in range(3):  # 最多 3 轮压缩
        messages, active_system_prompt = agent._compress_context(
            messages, system_message, approx_tokens=_preflight_tokens
        )
        if len(messages) >= _orig_len:
            break  # 无法进一步压缩
```

### 4.4 Cron 调度器：定时长程任务

**源码：`cron/scheduler.py` + `cron/jobs.py`**

```python
# scheduler.py 核心
_parallel_pool: Optional[concurrent.futures.ThreadPoolExecutor] = None
_sequential_pool: Optional[concurrent.futures.ThreadPoolExecutor] = None

def tick():
    """每 60 秒从后台线程调用"""
    due_jobs = get_due_jobs()
    
    for job in due_jobs:
        if job_needs_sequential_execution:
            # 序列化池（环境变量/配置隔离的任务）
            _get_sequential_pool().submit(lambda: run_job(job))
        else:
            # 并行池
            _get_parallel_pool(max_workers).submit(lambda: run_job(job))

def run_job(job):
    """执行单个 Cron 任务"""
    # 1. 构建 Agent prompt（含注入检测）
    prompt = _build_job_prompt(job)
    
    # 2. 配置工具集限制
    disabled = _resolve_cron_disabled_toolsets(cfg)
    enabled = _resolve_cron_enabled_toolsets(job, cfg)
    
    # 3. 启动 Agent 运行
    result = run_conversation(agent, prompt, ...)
    
    # 4. 保存输出
    save_job_output(job_id, timestamp, result)
    
    # 5. 交付结果
    _deliver_result(job, result)
    
    # 6. 推进下次运行时间
    advance_next_run(job)
```

**调度类型**（`cron/jobs.py`）：

```python
def parse_schedule(schedule: str) -> Dict[str, Any]:
    # "30m"              → 一次性，30分钟后
    # "2h"               → 一次性，2小时后
    # "every 30m"        → 循环，每30分钟
    # "every 2h"         → 循环，每2小时
    # "0 9 * * *"        → cron 表达式
    # "2026-02-03T14:00" → 一次性，指定时间
```

**容错机制**：
- 文件锁防止多个 tick 重叠执行
- Grace window：一次性任务错过执行时间后的宽限期（120秒）
- 半周期 grace 计算：循环任务错过后的追赶逻辑
- 并行池可配置最大工作线程数
- 输出原子写入（临时文件 + 重命名）

### 4.5 Async/Sync Bridging

**源码：`agent/async_utils.py`**

```python
def safe_schedule_threadsafe(coro, loop, ...):
    """从同步线程安全调度协程到事件循环"""
    if loop is None:
        coro.close()  # 防止协程泄漏
        return None
    
    try:
        return asyncio.run_coroutine_threadsafe(coro, loop)
    except Exception:
        coro.close()  # 调度失败时关闭协程
        return None
```

这确保了在多线程环境中，异步操作不会泄漏未等待的协程。

### 4.6 Background Review：后台自我改进

```python
# 对话完成后，后台触发 review
if _should_review_memory or _should_review_skills:
    agent._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

Hermes 在用户对话完成后，会在后台 fork 一个新线程，让 Agent 自我审查和更新 memory/skills，不阻塞用户交互。

---

## 五、三种机制对比分析

### 5.1 循环模型对比

| 特性 | Ralph Loop | OpenClaw | Hermes Agent |
|------|-----------|----------|-------------|
| **循环粒度** | 进程级（每次启动新 Agent） | 事件级（单个 turn） | 工具调用级（单次 API call） |
| **状态保持** | 文件（`progress.txt`） | 内存 + SQLite + 磁盘 | SQLite + 内存 |
| **中断恢复** | ❌ 无 | ✅ Session resume + orphan recovery | ✅ Session resume + pending steer |
| **并发执行** | ❌ 串行 | ✅ 多 Lane 并发 | ✅ 线程池并行 |
| **子代理委派** | ❌ 不支持 | ✅ 完整 Subagent Registry | ✅ delegate_task + IterationBudget |
| **运行时指导** | ❌ 不支持 | ✅ Steering queue | ✅ /steer 命令 |
| **故障转移** | ❌ 无 | ✅ 后端 failover | ✅ fallback chain |
| **预算控制** | 最大迭代数 | Lane 超时 + turn 超时 | IterationBudget + max_iterations |
| **优雅退出** | ❌ 无 | ✅ Drain 模式 | ✅ 中断检查 + graceful summary |

### 5.2 长程任务设计哲学

**Ralph Loop**：极简主义
> "每次都是一个全新的开始，状态通过文件传递。简单到无法出错，但也简单到无法处理复杂场景。"

- 适合：短期、独立、可分解的编码任务
- 不适合：需要上下文保持、多步骤依赖、需要人工干预的任务

**OpenClaw**：企业级事件驱动
> "每一个交互都是事件，每一个事件都在 Lane 上排队。状态可恢复，任务可委派，交付可重试。"

- 适合：7×24 运行的个人助手，多平台、多 Agent、复杂任务编排
- 复杂度代价：代码量巨大，学习曲线陡峭

**Hermes Agent**：实用主义会话驱动
> "一次对话就是多轮 API 调用，预算耗尽就要求总结。Cron 调度器负责定时任务，后台 review 负责自我改进。"

- 适合：交互式对话 + 定时自动化 + 自我进化
- 特色：Budget refund 机制、Context compression、Background review

### 5.3 状态持久化对比

| 状态类型 | Ralph | OpenClaw | Hermes |
|---------|-------|----------|--------|
| 任务进度 | `prd.json` (JSON) | Session store (SQLite) | Session DB (SQLite) |
| Agent 记忆 | `progress.txt` (Markdown) | Agent steering queue + memory | Memory manager + skills |
| 子代理状态 | N/A | Subagent Registry (SQLite + JSON) | N/A (delegate_task 是同步的) |
| Cron 任务 | N/A | Built-in cron | `jobs.json` (JSON) |
| 系统提示 | N/A | ACP session meta | Session DB 缓存 |

### 5.4 容错深度对比

```
Ralph:
  失败 → || true → 继续下一轮 → 达到 MAX_ITERATIONS → 退出
  
OpenClaw:
  失败 → 重试 (2次) → 后端 failover → 超时清理 → 
  孤儿恢复 → 交付重试 → 挂起队列
  
Hermes:
  失败 → 重试 (3次) → fallback chain → 预算耗尽 → 
  要求总结 → 上下文压缩 → 后台 review
```

---

## 六、源码级关键技术细节

### 6.1 Ralph 的 PRD 状态机

```
初始状态: 所有 story.passes = false
每次迭代:
  1. 找到 priority 最小且 passes=false 的 story
  2. 实现该 story
  3. 设置 story.passes = true
  4. 追加 progress.txt
终止条件: 所有 story.passes = true → <promise>COMPLETE</promise>
```

### 6.2 OpenClaw 的 Lane 优先级队列

```typescript
// 优先级排序逻辑
function enqueueLaneEntry(state, entry) {
  const insertAt = state.queue.findIndex(
    (queued) => queued.priority < entry.priority ||
      (queued.priority === entry.priority && queued.sequence > entry.sequence)
  );
  // foreground(1) > normal(0) > background(-1)
  if (insertAt < 0) state.queue.push(entry);
  else state.queue.splice(insertAt, 0, entry);
}
```

### 6.3 Hermes 的 Iteration Budget 线程安全

```python
class IterationBudget:
    """线程安全的迭代计数器"""
    def __init__(self, max_total):
        self.max_total = max_total  # 父 Agent 默认 90，子 Agent 默认 50
        self._used = 0
        self._lock = threading.Lock()
    
    def consume(self) -> bool:
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True
    
    def refund(self):
        """execute_code 的迭代不消耗预算"""
        with self._lock:
            if self._used > 0:
                self._used -= 1
```

### 6.4 OpenClaw 的 Subagent 孤儿恢复

```
进程重启时:
  1. 从磁盘恢复 Subagent Registry
  2. reconcileOrphanedRestoredRuns() 检测孤儿
  3. 对于 "running" 但没有活跃运行上下文的子代理:
     - STALE_ACTIVE_SUBAGENT_GRACE_MS (60s) 后标记为 stale
     - 尝试恢复或终止
  4. 对于挂起交付的子代理:
     - 根据类型设置过期时间 (2h/6h/24h)
     - 超过 SUSPENDED_DELIVERY_HARD_CAP (50) 时停止重试
```

### 6.5 Hermes 的 Cron Grace Window

```python
def _compute_grace_seconds(schedule: dict) -> int:
    """计算任务错过后允许追赶的宽限期
    使用调度周期的一半，限制在 120s ~ 2h 之间
    """
    MIN_GRACE = 120
    MAX_GRACE = 7200  # 2 hours
    # ...
    # 每5分钟的任务 → ~150s grace
    # 每天的任务 → 2h grace
```

---

## 七、总结与启示

### 7.1 三种模式适用场景

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| 快速原型/一次性批量编码 | Ralph Loop | 简单、无配置、开箱即用 |
| 7×24 个人助手/多平台集成 | OpenClaw | 最完整的生命周期管理 |
| 交互式 AI + 定时任务 + 自我改进 | Hermes Agent | 平衡了复杂度和功能 |

### 7.2 设计模式总结

1. **文件驱动状态**（Ralph）：最朴素但最透明的方式，状态可审计、可手动修改
2. **事件驱动队列**（OpenClaw）：最强大的并发控制，支持复杂的多 Agent 协作
3. **预算驱动的迭代**（Hermes）：最实用的长程任务控制，预算耗尽自动降级

### 7.3 关键教训

- **Ralph** 证明了：极简设计在特定场景下可以非常有效，但缺乏恢复能力是致命弱点
- **OpenClaw** 证明了：完善的生命周期管理（注册、持久化、孤儿恢复、交付重试）是 7×24 运行的前提
- **Hermes** 证明了：预算控制 + 上下文压缩 + 后台自我改进 是可持续长程运行的关键三角
