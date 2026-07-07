# Codex `/goal` 的原理机制与实现方式

> 基于 OpenAI Codex 开源仓库 (`openai/codex`) 源码深度分析。核心实现位于 `codex-rs/ext/goal/`。

## 一、整体架构概览

`/goal` 不是单纯的 UI 命令，而是一个**贯穿 TUI → Extension → Tool → Steering → Persistence → Analytics 六层的完整子系统**。它是 Codex 实现**长时任务自主持续执行**的核心机制。

```
┌─────────────────────────────────────────────────────────┐
│                    TUI Layer                             │
│  slash_input.rs → SlashCommand::Goal                     │
│  goal_menu.rs / thread_goal_actions.rs / goal_status.rs  │
└─────────────────────┬───────────────────────────────────┘
                      │ AppServer Protocol
┌─────────────────────▼───────────────────────────────────┐
│              GoalExtension (Extension API)               │
│  ThreadLifecycle / TurnLifecycle / ToolContributor       │
│  extension.rs                                            │
└──────┬──────────────┬───────────────┬────────────────────┘
       │              │               │
┌──────▼─────┐ ┌─────▼──────┐ ┌──────▼──────────┐
│GoalTool    │ │GoalRuntime │ │GoalAccounting   │
│(3 tools)   │ │(steering)  │ │(tokens + time)  │
│tool.rs     │ │runtime.rs  │ │accounting.rs    │
│            │ │steering.rs │ │                 │
└──────┬─────┘ └─────┬──────┘ └──────┬──────────┘
       │              │               │
┌──────▼──────────────▼───────────────▼──────────────────┐
│              Persistence Layer                          │
│  SQLite: thread_goals table (state/src/runtime/goals.rs)│
│  GoalStore: CRUD + atomic accounting updates            │
└─────────────────────────────────────────────────────────┘
```

## 二、核心概念：Goal 是什么？

Goal 是一个**持久化的线程级任务目标**，与普通对话轮次（turn）不同，它具有以下关键特征：

| 特性 | 普通对话 | Goal 模式 |
|------|---------|-----------|
| 生命周期 | 单次请求-响应 | 跨 turn 持续，直到 complete/blocked/budget_limited |
| 上下文注入 | 用户消息 | 自动注入 steering prompt（隐藏 prompt） |
| Token 预算 | 无 | 可选 `token_budget` 上限 |
| 时间统计 | 无 | 自动累计 `time_used_seconds` |
| 自动续跑 | 无 | idle 时自动 continuation |
| 完成审计 | 无 | 严格的 completion audit + blocked audit |

### Goal 的状态机

定义在 `codex-rs/state/src/model/thread_goal.rs`：

```
                    ┌──────────┐
      创建 ────────►│  Active  │
                    └────┬─────┘
               ┌────┬────┼────┬────────┐
               │    │    │    │        │
           pause  │  complete│   budget
               │  │    │    │   exhausted
          ┌────▼──▼────▼────▼──┐
          │     Terminal       │
          │ (Paused/Blocked/   │
          │  UsageLimited/     │
          │  BudgetLimited/    │
          │  Complete)         │
          └────────────────────┘
```

## 三、核心实现拆解

### 3.1 三个内置 Tool（给 Agent 用的）

定义在 `codex-rs/ext/goal/src/spec.rs`：

```rust
// 1. get_goal — 查询当前目标
create_get_goal_tool()
  // 无参数，返回 status、budget、token usage、remaining budget

// 2. create_goal — 创建新目标  
create_create_goal_tool()
  // objective (必需) + token_budget (可选)
  // 规则：仅当用户明确要求时创建，不能从普通任务中推断

// 3. update_goal — 标记完成或受阻
create_update_goal_tool()
  // status: "complete" | "blocked"（仅此两种！）
  // complete: 客观完成，无遗留工作
  // blocked: 同一阻塞条件连续出现 3 轮以上
```

**关键设计约束**（源码中硬编码的 prompt 规则）：
> "Do not use `blocked` merely because the work is hard, slow, uncertain, incomplete, or would benefit from clarification."
> "Do not mark a goal complete merely because its budget is nearly exhausted."

### 3.2 Extension 生命周期钩子

`GoalExtension` 实现了 Extension API 的多个贡献者接口（`extension.rs`）：

| 钩子 | 作用 |
|------|------|
| `on_thread_start` | 初始化 GoalRuntimeHandle、GoalAccountingState |
| `on_thread_resume` | 恢复暂停/阻塞的目标 |
| `on_thread_idle` | **核心！** 如果 goal 是 active 且 thread idle，自动触发 continuation |
| `on_thread_stop` | 注销 runtime |
| `on_turn_start` | 开始新的一轮，记录 token 使用基线，注入 goal steering prompt |
| `on_tool_finish` | 工具调用完成后更新 token accounting |
| `on_turn_abort` | turn 被中断时停止 goal accounting |
| `on_turn_error` | turn 报错时将 goal 标记为 usage_limited |
| `on_config_changed` | 配置变化时同步 enabled 状态 |

**idle continuation 是最关键的机制**（`extension.rs`）：
```rust
fn on_thread_idle(&self, input: ThreadIdleInput) -> ExtensionFuture {
    Box::pin(async move {
        if let Err(err) = runtime.continue_if_idle().await {
            // 当 agent 完成一轮后处于 idle 状态，
            // 如果 goal 仍为 Active → 自动触发下一轮 continuation
        }
    })
}
```

### 3.3 Steering Prompt 注入机制

这是 Goal 机制的**灵魂所在**——每轮对话开始前，系统自动向 model 注入隐藏的 steering prompt。定义在 `steering.rs` 和模板文件中。

有三种 steering prompt 模板：

**① `continuation.md`（续跑提示）— 最复杂、最重要的模板：**
```markdown
Continue working toward the active thread goal.
<objective>{{ objective }}</objective>

关键规则：
- Goal 跨 turn 持久化，不要缩小目标范围
- 保持完整 objective，不能完成就继续
- 严格的 completion audit：逐条验证需求
- 严格的 blocked audit：同一阻塞条件连续 3 轮
- 从证据出发，不依赖记忆或意图
```

**② `budget_limit.md`（预算耗尽提示）：**
```markdown
The active thread goal has reached its token budget.
总结进展，识别剩余工作，不要开始新工作。
```

**③ `objective_updated.md`（目标被编辑提示）：**
```markdown
The active thread goal objective was edited by the user.
调整当前 turn 以追求更新后的目标。
```

注入方式是通过 `InternalModelContextFragment`，source 标记为 `"goal"`，作为隐藏上下文传递给模型：
```rust
fn goal_context_input_item(prompt: String) -> ResponseItem {
    ContextualUserFragment::into(InternalModelContextFragment::new(
        InternalContextSource::from_static("goal"),
        prompt,
    ))
}
```

### 3.4 Token & 时间 Accounting

`accounting.rs` 实现了精确的 token 和时间统计：

```rust
struct GoalAccountingInner {
    current_turn_id: Option<String>,          // 当前 turn ID
    turns: HashMap<String, GoalTurnAccounting>, // 每轮 token 使用
    wall_clock: GoalWallClockAccounting,       // 挂钟时间统计
    budget_limit_reported_goal_id: Option<String>,
}
```

**核心流程**：
1. `start_turn()` — 记录 turn 开始时的 token 基线
2. `record_token_usage()` — 每次 tool call 后计算 delta
3. `progress_snapshot()` — 生成进度快照（token delta + time delta）
4. `mark_progress_accounted_for_status()` — 将 delta 写入 SQLite

**预算超限检测**（在 `GoalStore` 的 SQL 中）：
```sql
status = CASE
    WHEN status = 'active' AND tokens_used >= token_budget 
    THEN 'budget_limited'
    ELSE status
END
```

### 3.5 持久化存储

SQLite 表结构（`0029_thread_goals.sql`）：
```sql
CREATE TABLE thread_goals (
    thread_id TEXT PRIMARY KEY REFERENCES threads(id) ON DELETE CASCADE,
    goal_id TEXT NOT NULL,
    objective TEXT NOT NULL,
    status TEXT NOT NULL CHECK(status IN (
        'active', 'paused', 'blocked', 'usage_limited', 'budget_limited', 'complete'
    )),
    token_budget INTEGER,
    tokens_used INTEGER NOT NULL DEFAULT 0,
    time_used_seconds INTEGER NOT NULL DEFAULT 0,
    created_at_ms INTEGER NOT NULL,
    updated_at_ms INTEGER NOT NULL
);
```

每个 thread 最多一个 goal（`thread_id` 是主键），通过 `ON CONFLICT DO UPDATE WHERE status = 'complete'` 实现：只有完成的目标才能被替换。

### 3.6 TUI 交互层

**命令解析**（`slash_command.rs`）：
```rust
SlashCommand::Goal => "set or view the goal for a long-running task"
```

**支持的操作**（`goal_display.rs`）：
```
/goal [<objective>|clear|edit|pause|resume]
```

| 子命令 | 行为 |
|--------|------|
| `/goal 做XXX` | 创建或替换目标 |
| `/goal edit` | 打开编辑器修改目标 |
| `/goal pause` | 暂停当前目标 |
| `/goal resume` | 恢复暂停/阻塞的目标 |
| `/goal clear` | 清除当前目标 |
| `/goal`（无参数） | 显示目标状态摘要 |

**目标文件机制**（`goal_files.rs`）：超长 objective 会物化为文件：
```rust
// 当 objective 超过 MAX_THREAD_GOAL_OBJECTIVE_CHARS 时：
// 写入 $CODEX_HOME/attachments/<uuid>/goal-objective.md
// 在 objective 中替换为引用：
// "Read the Codex goal objective file at <path> before continuing."
```

**状态栏显示**（`goal_status.rs`）：
- Active: 显示 `12.5K / 50K`（token 预算模式）或 `2m`（时间模式）
- BudgetLimited: 显示 `63.9K / 50K tokens`（超预算时）
- Complete: 显示最终消耗

## 四、关键运行流程

### 完整生命周期：

```
1. 用户输入: /goal "实现一个完整的 Web 框架"
   └─ TUI → AppServer → GoalService.set_thread_goal()
      └─ SQLite INSERT → GoalRuntime.apply_external_goal_set()
         └─ 状态=Active, tools 可见给 Agent

2. Agent 每轮开始前（on_turn_start）：
   └─ 注入 continuation steering prompt（隐藏）
      └─ 模型看到 objective + 预算 + completion audit 规则

3. Agent 工作（可能调用 create_goal/update_goal/get_goal tools）

4. Agent 完成一轮回复 → on_thread_idle
   └─ runtime.continue_if_idle() 检测 goal 仍为 Active
      └─ 自动触发下一轮 continuation（无需用户介入！）

5. 循环持续直到：
   a. Agent 调用 update_goal(status="complete") → 完成审计通过
   b. 同一阻塞条件连续 3 轮 → update_goal(status="blocked")
   c. tokens_used >= token_budget → 自动变为 budget_limited
   d. 用户手动 /goal pause 或 /goal clear

6. 完成后显示 usage summary：
   "Time: 2m. Tokens: 63.9K/50K."
```

## 五、设计亮点总结

| 设计点 | 说明 |
|--------|------|
| **跨轮持久化** | Goal 不是单轮消息，而是持久在 SQLite 中的 thread-level 状态 |
| **Steering Prompt 注入** | 每轮自动注入隐藏 prompt，让模型"记住"目标 |
| **严格审计规则** | Completion 需要逐条证据验证，Blocked 需要连续 3 轮相同条件 |
| **自动续跑** | idle 时自动 continuation，无需用户反复催促 |
| **Token/时间双维度** | 支持 token_budget 上限，同时统计挂钟时间 |
| **Extension 架构** | 通过 ThreadLifecycle/TurnLifecycle/ToolContributor 钩子无缝集成 |
| **并发安全** | Semaphore 锁保护 goal state，防止 idle continuation 与外部修改冲突 |
| **长目标文件化** | 超长 objective 物化为文件，避免上下文窗口溢出 |

**本质**：`/goal` 是 Codex 实现**自主持续工作模式**的核心机制——它把一个"一次性问答"变成了"有明确目标、有预算约束、有完成验证、能自动续跑的持久任务"。
