# Hermes Agent 多 Agent 交互：6 种类型、原理与实现

> "The unit of intelligence is not a model — it's a system of models working together."

Hermes Agent 的多 Agent 交互可以分为 **6 种类型**，按**耦合度从紧到松、时间跨度从秒到天**排列：

---

## 类型一：`delegate_task` — 同步子代理委派

**本质**：父 Agent 在当前对话轮次内同步派生一个或多个子 Agent，等待结果后继续。

**机制**：

```
父 Agent (当前 turn)
  ├── delegate_task(goal="A", toolsets=['terminal','file'])  → 子 Agent A (孤立会话)
  ├── delegate_task(goal="B", toolsets=['web'])             → 子 Agent B (孤立会话)  ← 并行
  └── delegate_task(goal="C", toolsets=['terminal'])        → 子 Agent C (孤立会话)  ← 并行
  ↓ 汇总结果，继续执行
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| **同步阻塞** | 父 Agent 等待所有子 Agent 完成后才继续 |
| **上下文隔离** | 每个子 Agent 有独立对话、独立终端会话，无父 Agent 的聊天历史 |
| **工具子集** | 通过 `toolsets` 参数限制子 Agent 可用工具（最小权限原则） |
| **角色分层** | `leaf`（不能继续委派）vs `orchestrator`（可以继续派生子 Agent） |
| **并发上限** | 默认 3 个并行，可通过 `delegation.max_concurrent_children` 调整 |
| **深度限制** | 嵌套委派受 `delegation.max_spawn_depth`（默认 2）限制 |

**实现位置**：

- **代码层**：`delegate_task` 工具函数
- **配置**：`config.yaml` 中的 `delegation` 段（model、provider、max_iterations=50、reasoning_effort）
- 每个子 Agent 使用独立会话，共享同一进程但不同对话上下文

**适用场景**：需要几分钟完成的并行子任务（代码审查、研究、数据处理）

**不适用**：

- 需要用户交互的任务（子 Agent 不能调用 `clarify`）
- 超长时间任务（父 Agent 被中断则子 Agent 被取消）
- 后台持久任务（应改用 `cronjob` 或 `terminal(background=True)`）

---

## 类型二：子代理驱动开发（Subagent-Driven Development）— 两阶段审查流水线

**本质**：`delegate_task` 的**结构化工作流变体**，每个任务经历「实现者 → 规范审查 → 质量审查」三阶段。

**机制**：

```
Plan (已分解的任务列表)
  ↓
┌─────────────────────────────────────────┐
│ Task N: "Create User model"              │
│                                          │
│  1. delegate_task(goal="实现", tools=['terminal','file'])  │
│     → 实现者 Agent: 写代码 + 测试 + 提交   │
│                                          │
│  2. delegate_task(goal="规范审查", tools=['file'])         │
│     → 审查者 A: 是否满足 spec？PASS/FAIL   │
│     如果 FAIL → 修复 → 重新审查            │
│                                          │
│  3. delegate_task(goal="质量审查", tools=['file'])         │
│     → 审查者 B: 代码质量？APPROVED/CHANGES │
│     如果有问题 → 修复 → 重新审查           │
│                                          │
│  4. 标记任务完成                           │
└─────────────────────────────────────────┘
  ↓
全部任务完成 → 最终集成审查
```

**关键设计**：

| 设计原则 | 说明 |
|----------|------|
| **全新子 Agent** | 每个任务用全新子 Agent，不积累上下文污染 |
| **两级审查分离** | 规范审查（有没有漏做/多做）先于质量审查（做得好不好） |
| **任务粒度** | 每个任务 2-5 分钟的专注工作量 |

**实现**：`subagent-driven-development` 技能定义了完整的流程规范，集成 `writing-plans`、`test-driven-development`、`requesting-code-review` 等技能。

---

## 类型三：Hermes 独立进程实例 — 完全隔离的自主 Agent

**本质**：通过 shell 启动一个**全新的 Hermes 进程**，拥有独立的会话、工具、环境。

**机制**：

| 模式 | 命令 | 隔离程度 |
|------|------|---------|
| **一次性** | `hermes chat -q '任务描述'` | 独立进程，独立会话 |
| **交互式 (tmux)** | `tmux new-session -d -s agent1 'hermes'` | 独立进程 + 伪终端 |
| **工作树模式** | `hermes -w` | 独立进程 + 独立 git worktree |

```
父进程 (主控 Agent)
  ├── terminal("tmux new-session -d -s backend 'hermes'")
  │     → Agent A: 完全独立的 Hermes 实例 (后端开发)
  ├── terminal("tmux new-session -d -s frontend 'hermes'")
  │     → Agent B: 完全独立的 Hermes 实例 (前端开发)
  └── 通过 tmux capture-pane 读取进度，tmux send-keys 传递上下文
```

**与 `delegate_task` 的核心区别**：

| | `delegate_task` | 独立进程 |
|-|-----------------|---------|
| 隔离级别 | 独立对话，共享进程 | 完全独立进程 |
| 持续时间 | 分钟级（受父轮次限制） | 小时/天级 |
| 工具访问 | 父 Agent 授权的子集 | 完整工具集 |
| 交互性 | 无（不能 clarify） | 有（PTY 模式） |
| 中断影响 | 父中断→子取消 | 不受影响 |

**适用场景**：长时间自主任务、需要完整工具访问、需要并行编辑不同 git worktree

---

## 类型四：Kanban 编排 — 专家角色路由系统

**本质**：一个编排 Agent（Orchestrator）负责任务分解和路由，多个专家 Agent（Specialist）各自领取任务，通过**依赖图**自动编排执行顺序。

**机制**：

```
编排者 (Orchestrator)
  ↓ 分解任务
┌──────────────────────────────────────────────┐
│ Kanban Board (SQLite 持久化)                   │
│                                              │
│ T1 [researcher]  研究成本    ✓ done ─┐        │
│ T2 [researcher]  研究性能    ✓ done ─┤        │
│                                      ↓        │
│ T3 [analyst]     综合建议    ready ──→ done   │
│                                      ↓        │
│ T4 [writer]      撰写备忘录  todo ────→ ready  │
│                                              │
│ parents=[T1,T2] → 自动门控，父任务全部 done 后 │
│ 子任务自动 promote 到 ready                    │
└──────────────────────────────────────────────┘
```

**关键机制**：

| 机制 | 说明 |
|------|------|
| **依赖门控** | `parents=[t1, t2]` 创建任务间的 DAG 依赖关系 |
| **自动状态推进** | 父任务全部 `done` → 子任务自动从 `todo` → `ready` |
| **专家角色约定** | `researcher`、`analyst`、`writer`、`reviewer`、`backend-eng`、`frontend-eng`、`ops`、`pm` |
| **人工介入** | 任何任务可以 `kanban_block()` 等待人工输入 |
| **审计追踪** | 所有行持久化在 SQLite 中 |

**编排者规则（反诱惑规则）**：

- **不自己做工作**——只分解、路由、总结
- **没有合适的专家？问用户**——不要自己动手

**实现位置**：`kanban-orchestrator` + `kanban-worker` 技能，SQLite 存储

---

## 类型五：Ralph Loop（自主循环 Agent）— 迭代式长周期任务

**本质**：Agent **反复运行**直到任务真正完成，而非一次 LLM 调用结束就停止。

**机制**：

```
┌─────────────────────────────────────────┐
│          自主 Agent 循环 (外层)            │
│  ┌──────────────────────────────────┐   │
│  │  Agent 内循环: LLM ↔ 工具 ↔ ...   │   │
│  └──────────────────────────────────┘   │
│                 ↓                        │
│  verifyCompletion: "任务真的完成了吗？"     │
│                 ↓                        │
│      No → 更新上下文 → 下一轮迭代          │
│      Yes → 返回最终结果                   │
└─────────────────────────────────────────┘
```

**两种实现方式**：

| 方式 | 适用时间尺度 | 实现 |
|------|------------|------|
| **Cronjob 模式** | 小时/天 | `cronjob(action="create", schedule="every 5m")` 作为外层循环 |
| **Delegate 模式** | 分钟 | 父循环中反复调用 `delegate_task` |

**核心设计模式**：

| 设计模式 | 说明 |
|----------|------|
| **每次迭代用全新上下文** | 避免上下文窗口退化 |
| **状态文件持久化** | JSON 文件记录里程碑进度 |
| **质量门控** | 测试/编译通过才能标记里程碑完成 |
| **完成信号检测** | 输出标记、状态检查、共识检测 |
| **知识积累** | 每轮迭代追加 learnings 到 progress log |
| **安全限制** | 最大迭代次数、最大花费、超时退出 |

**与 `delegate_task` 的区别**：Ralph Loop 是**循环式**的——反复派生、验证、重试，直到任务完成；`delegate_task` 是**一次性**的。

**实现位置**：`autonomous-agent-loop` 技能，集成 `cronjob`、`file`、`terminal`、`memory` 工具

---

## 类型六：Cronjob + Webhook — 事件驱动与定时调度

**本质**：通过**时间触发**或**事件触发**自主运行 Agent，无需人工启动。

**Cronjob 机制**：

```python
cronjob(
    action="create",
    schedule="every 2h",           # 调度表达式
    prompt="检查 API 状态，写报告",   # 自包含任务描述
    skills=["some-skill"],         # 预加载技能
    script="collect_data.py",      # 可选：前置数据收集脚本
    context_from=["upstream-job"], # 可选：注入上游 job 输出
    deliver="origin",              # 结果送回当前对话
)
```

**Webhook 机制**：

```
外部系统 POST → /webhooks/<name> → 触发 Agent 执行
```

**关键特性**：

| 特性 | 说明 |
|------|------|
| **全新会话** | Cronjob 运行在无当前聊天上下文的全新会话中 |
| **前置脚本** | 支持 `script` 参数，stdout 注入到 prompt |
| **链式注入** | 支持 `context_from` 注入上游 Job 输出 |
| **重复限制** | 支持 `repeat` 限制重复次数 |
| **工具限制** | 支持 `enabled_toolsets` 限制可用工具集 |

---

## 全景对比

| 类型 | 耦合度 | 时间跨度 | 上下文关系 | 典型用途 |
|------|--------|---------|-----------|---------|
| `delegate_task` | 紧耦合 | 秒~分钟 | 父→子单向传参 | 并行子任务、代码审查 |
| 子代理驱动开发 | 紧耦合 | 分钟 | 严格流水线 | 按规范实现 + 两阶段审查 |
| 独立进程 | 松耦合 | 分钟~天 | 完全隔离 | 长期自主任务、git worktree |
| Kanban 编排 | 中耦合 | 小时~天 | DAG 依赖图 | 多专家协作、审计追踪 |
| Ralph Loop | 自循环 | 小时~天 | 迭代接力棒 | 大型任务的反复迭代完成 |
| Cron/Webhook | 解耦 | 定时/事件 | 无上下文 | 监控、报告、事件响应 |

---

## 总结

- Hermes Agent 提供 **6 种多 Agent 交互模式**，覆盖从秒级并行到天级自主循环的完整谱系
- **核心设计原则**：上下文隔离、最小权限工具集、质量门控、状态持久化
- **选择建议**：短时间任务用 `delegate_task`，长时间任务用独立进程或 Cronjob，复杂协作用 Kanban，迭代任务用 Ralph Loop
- **关键配置**：`config.yaml` 中的 `delegation` 段控制并发和深度，`cronjob` 控制定时调度，Kanban 使用 SQLite 持久化状态
