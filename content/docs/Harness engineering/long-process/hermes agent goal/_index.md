# Hermes Agent `/goal` 的原理机制与实现方式

> 基于 Hermes Agent 开源仓库源码深度分析。核心实现位于 `hermes_cli/goals.py`（~535 行），CLI 集成在 `cli.py`（~6500-7200 行），Gateway 集成在 `gateway/run.py`（~7479-7672 行）。

## 一、整体定位

Hermes Agent 的 `/goal` 是一个**跨轮次持久化目标系统**——它给 Agent 设定一个"站着的目标"（standing goal），然后让 Agent 在多个对话轮次中自动持续工作，直到目标达成、预算耗尽、或用户手动暂停/清除。

文档中的原话：

> It's our take on the **Ralph loop**, directly inspired by Codex CLI 0.128.0's `/goal` by Eric Traut (OpenAI). The core idea — keep a goal alive across turns and don't stop until it's achieved — is theirs. The implementation here is independent and adapted to Hermes' architecture.

**一句话概括**：用户设定目标 → 每轮结束后调用辅助模型（judge）判断是否完成 → 未完成则自动注入 continuation prompt 触发下一轮 → 循环直到 done / budget exhausted / user intervention。

## 二、整体架构

```
┌──────────────────────────────────────────────────────────────────┐
│                        User Interface                             │
│  CLI: _handle_goal_command() + _pending_input.put()              │
│  Gateway: _handle_goal_command() + _enqueue_fifo()               │
└──────────────────────┬───────────────────────────────────────────┘
                       │
            ┌──────────▼──────────┐
            │    GoalManager       │
            │  (hermes_cli/goals.py)│
            │                      │
            │  set/pause/resume/   │
            │  clear/status/       │
            │  evaluate_after_turn │
            │  next_continuation   │
            └──┬──────┬──────┬─────┘
               │      │      │
     ┌─────────▼┐ ┌───▼────┐ │
     │SessionDB │ │ Judge  │ │
     │state_meta│ │(aux    │ │
     │goal:<sid>│ │ model) │ │
     └──────────┘ └────────┘ │
                             │
              ┌──────────────▼──────────────┐
              │  Continuation Prompt → FIFO  │
              │  (_pending_input / adapter)  │
              └─────────────────────────────┘
```

## 三、核心模块拆解

### 3.1 GoalState — 数据模型

定义在 `hermes_cli/goals.py`，是一个可序列化的 dataclass：

```python
@dataclass
class GoalState:
    goal: str                        # 目标文本
    status: str = "active"           # active | paused | done | cleared
    turns_used: int = 0              # 已消耗轮次
    max_turns: int = 20              # 轮次预算上限（可配置）
    created_at: float = 0.0          # 创建时间戳
    last_turn_at: float = 0.0        # 最后一轮时间戳
    last_verdict: Optional[str] = None  # "done" | "continue" | "skipped"
    last_reason: Optional[str] = None   # judge 的判据理由
    paused_reason: Optional[str] = None # 暂停原因
```

**状态机**：

```
              set()                  pause()
  [none] ──────────► active ──────────► paused
                         │                │
                         │                │ resume(reset_budget=True)
                         │                │
                         │                ▼
                    evaluate_after_turn  active
                         │
            ┌────────────┼────────────┐
            │            │            │
        done=true   turns>=max   user clears
            │            │            │
            ▼            ▼            ▼
          done        paused       cleared
```

### 3.2 持久化层 — SessionDB

Goal 状态通过 `SessionDB.state_meta` 表持久化，key 为 `goal:<session_id>`：

```python
def _meta_key(session_id: str) -> str:
    return f"goal:{session_id}"

def save_goal(session_id: str, state: GoalState) -> None:
    db = _get_session_db()
    db.set_meta(_meta_key(session_id), state.to_json())

def load_goal(session_id: str) -> Optional[GoalState]:
    db = _get_session_db()
    raw = db.get_meta(_meta_key(session_id))
    return GoalState.from_json(raw) if raw else None
```

**关键设计**：DB 实例按 `HERMES_HOME` 路径缓存（`_DB_CACHE`），避免每次调用都重新打开 SQLite 文件。

**持久化的意义**：`/resume` 能恢复之前设置的目标——即使关闭终端、第二天回来，goal 状态（active/paused/done）依然完好。

### 3.3 Judge 机制 — 核心判断引擎

这是整个 `/goal` 的**灵魂**。定义在 `hermes_cli/goals.py` 的 `judge_goal()` 函数：

```python
def judge_goal(goal: str, last_response: str, *, timeout=30.0) -> Tuple[str, str]:
    """Ask the auxiliary model whether the goal is satisfied.
    
    Returns (verdict, reason) where verdict is "done", "continue", or "skipped".
    
    Deliberately fail-open: any error returns ("continue", "...")
    so a broken judge doesn't wedge progress — the turn budget is the backstop.
    """
```

**Judge 的 System Prompt**（硬编码在源码中）：

```
You are a strict judge evaluating whether an autonomous agent has 
achieved a user's stated goal. You receive the goal text and the 
agent's most recent response. Your only job is to decide whether 
the goal is fully satisfied based on that response.

A goal is DONE only when:
- The response explicitly confirms the goal was completed, OR
- The response clearly shows the final deliverable was produced, OR
- The response explains the goal is unachievable / blocked / needs 
  user input (treat this as DONE with reason describing the block).

Otherwise the goal is NOT done — CONTINUE.

Reply ONLY with a single JSON object on one line:
{"done": <true|false>, "reason": "<one-sentence rationale>"}
```

**Judge 的 User Prompt**（模板）：

```
Goal:
{goal}

Agent's most recent response:
{response}

Is the goal satisfied?
```

**调用方式**：
```python
client, model = get_text_auxiliary_client("goal_judge")
resp = client.chat.completions.create(
    model=model,
    messages=[
        {"role": "system", "content": JUDGE_SYSTEM_PROMPT},
        {"role": "user", "content": prompt},
    ],
    temperature=0,
    max_tokens=200,
    timeout=timeout,
)
```

**关键设计决策**：

| 设计点 | 说明 |
|--------|------|
| **Fail-open 语义** | Judge 出错（网络故障、JSON 解析失败、模型不可用）时返回 `("continue", "...")`，不会卡住进度。真正的安全网是 turn budget。 |
| **保守判断** | 只有明确完成或确认受阻才标记 done，其他情况一律 continue。这使得 false positive（误判完成）比 false negative（误判未完成）更罕见。 |
| **辅助模型** | 使用独立的 `goal_judge` auxiliary task，可配置为便宜快速的模型（如 gemini-3-flash），每次调用仅 ~200 输出 token。 |
| **截断保护** | goal 文本截断到 2000 字符，response 截断到 4000 字符，避免上下文爆炸。 |
| **JSON 解析容错** | 先尝试整体解析，失败后用正则提取第一个 JSON 对象，支持 markdown code fence 包裹。 |

### 3.4 GoalManager — 编排引擎

`GoalManager` 是 CLI 和 Gateway 共同使用的核心编排类：

```python
class GoalManager:
    def __init__(self, session_id: str, *, default_max_turns=20):
        self.session_id = session_id
        self.default_max_turns = default_max_turns
        self._state = load_goal(session_id)
```

**核心方法**：

| 方法 | 作用 |
|------|------|
| `set(goal)` | 设置新目标，创建 GoalState，status=active，turns_used=0 |
| `pause(reason)` | 暂停，status=paused，记录原因 |
| `resume(reset_budget=True)` | 恢复，status=active，**重置 turns_used=0** |
| `clear()` | 清除，status=cleared |
| `mark_done(reason)` | 标记完成 |
| `status_line()` | 生成状态行（用于 CLI/Gateway 显示） |
| `evaluate_after_turn(last_response)` | **核心入口**：调用 judge、更新状态、返回决策 |
| `next_continuation_prompt()` | 生成 continuation prompt 文本 |

**evaluate_after_turn 的决策逻辑**（最关键的函数）：

```python
def evaluate_after_turn(self, last_response, *, user_initiated=True):
    # 1. 检查是否有 active goal
    if state is None or state.status != "active":
        return {"status": None, "should_continue": False, ...}
    
    # 2. 计轮
    state.turns_used += 1
    state.last_turn_at = time.time()
    
    # 3. 调用 judge
    verdict, reason = judge_goal(state.goal, last_response)
    state.last_verdict = verdict
    state.last_reason = reason
    
    # 4. 分支决策
    if verdict == "done":
        # → 目标达成！
        state.status = "done"
        return {"should_continue": False, "verdict": "done", 
                "message": f"✓ Goal achieved: {reason}"}
    
    if state.turns_used >= state.max_turns:
        # → 预算耗尽！
        state.status = "paused"
        state.paused_reason = f"turn budget exhausted ({state.turns_used}/{state.max_turns})"
        return {"should_continue": False, "verdict": "continue",
                "message": f"⏸ Goal paused — {state.turns_used}/{state.max_turns} turns used."}
    
    # → 继续！
    return {"should_continue": True, "verdict": "continue",
            "continuation_prompt": self.next_continuation_prompt(),
            "message": f"↻ Continuing toward goal ({state.turns_used}/{state.max_turns}): {reason}"}
```

### 3.5 Continuation Prompt — 自动续跑注入

当 judge 判断需要继续时，系统自动生成一个 continuation prompt 并注入到对话队列中：

```python
CONTINUATION_PROMPT_TEMPLATE = (
    "[Continuing toward your standing goal]\n"
    "Goal: {goal}\n\n"
    "Continue working toward this goal. Take the next concrete step. "
    "If you believe the goal is complete, state so explicitly and stop. "
    "If you are blocked and need input from the user, say so clearly and stop."
)
```

**关键设计**：
- continuation prompt 只是一个普通的 user-role 消息
- **不修改 system prompt，不切换 toolset，不触碰对话上下文**
- 因此 prompt caching 完全不受影响——20 轮 goal 运行的 cache 成本等同于 20 轮普通对话

### 3.6 CLI 集成

CLI 端的 Goal 生命周期：

```python
# 1. 初始化（懒加载，缓存）
def _get_goal_manager(self):
    sid = self.session_id
    existing = getattr(self, "_goal_manager", None)
    if existing and existing.session_id == sid:
        return existing
    cfg = load_config()
    max_turns = cfg.get("goals", {}).get("max_turns", 20)
    mgr = GoalManager(session_id=sid, default_max_turns=max_turns)
    self._goal_manager = mgr
    return mgr

# 2. 处理 /goal 命令
def _handle_goal_command(self, cmd):
    parts = cmd.strip().split(None, 1)
    arg = parts[1].strip() if len(parts) > 1 else ""
    mgr = self._get_goal_manager()
    
    # /goal 或 /goal status → 显示状态
    if not arg or arg.lower() == "status":
        _cprint(mgr.status_line())
        return
    
    # /goal pause → 暂停
    if arg.lower() == "pause":
        mgr.pause(reason="user-paused")
        return
    
    # /goal resume → 恢复（重置计数器）
    if arg.lower() == "resume":
        mgr.resume()
        return
    
    # /goal clear/stop/done → 清除
    if arg.lower() in ("clear", "stop", "done"):
        mgr.clear()
        return
    
    # /goal <text> → 设置新目标，并立即触发第一轮
    state = mgr.set(arg)
    self._pending_input.put(state.goal)  # 注入队列，立即开始工作

# 3. 每轮结束后调用的 Hook
def _maybe_continue_goal_after_turn(self):
    mgr = self._get_goal_manager()
    if not mgr or not mgr.is_active():
        return
    
    # 如果用户已输入新消息，跳过判断（用户消息优先）
    if self._pending_input and not self._pending_input.empty():
        return
    
    # 提取 assistant 的最终回复
    last_response = extract_last_assistant_response(self.conversation_history)
    
    # 调用 judge
    decision = mgr.evaluate_after_turn(last_response, user_initiated=True)
    _cprint(decision.get("message", ""))
    
    # 如果需要继续，将 continuation prompt 注入队列
    if decision.get("should_continue"):
        self._pending_input.put(decision["continuation_prompt"])
```

### 3.7 Gateway 集成

Gateway 端（支持 Telegram/Discord/Slack/Matrix/Signal/WhatsApp 等所有平台）的实现逻辑相同，但使用了 adapter-FIFO 队列机制：

```python
async def _handle_goal_command(self, event):
    # ... 解析子命令 ...
    
    # 设置新目标时，构造一个 MessageEvent 并放入 adapter FIFO
    kickoff_event = MessageEvent(
        text=state.goal,
        message_type=MessageType.TEXT,
        source=event.source,
    )
    self._enqueue_fifo(_quick_key, kickoff_event, adapter)

def _post_turn_goal_continuation(self, *, session_entry, source, final_response):
    """Turn 边界后的 Hook — 调用 judge 并可能注入 continuation。"""
    mgr = GoalManager(session_id=sid, default_max_turns=max_turns)
    if not mgr.is_active():
        return
    
    decision = mgr.evaluate_after_turn(final_response, user_initiated=True)
    
    # 将 judge 结果发送给用户
    if decision.get("message"):
        adapter.send_message(source, decision["message"])
    
    # 如果需要继续，通过 adapter FIFO 注入 continuation
    if decision.get("should_continue"):
        cont_event = MessageEvent(text=decision["continuation_prompt"], ...)
        self._enqueue_fifo(_quick_key, cont_event, adapter)
```

**Gateway 的安全设计**：
- Agent 运行中，`/goal status`、`/goal pause`、`/goal clear` 是安全的（只修改控制面状态）
- `/goal <new text>` 在 Agent 运行中被拒绝，提示先 `/stop`，避免旧 continuation 与新目标竞争

### 3.8 用户消息优先机制

这是确保系统不会失控的关键设计：

**CLI 端**：
```python
# 如果 _pending_input 队列中已有用户消息，跳过判断
if self._pending_input and not self._pending_input.empty():
    return  # 用户消息优先
```

**Gateway 端**：
```python
# 通过 adapter FIFO 队列，用户消息天然排在 continuation 前面
self._enqueue_fifo(_quick_key, cont_event, adapter)
```

**效果**：任何用户发送的真实消息都会自动抢占 continuation loop，用户消息执行完后 judge 会再次运行——如果用户消息碰巧完成了目标，judge 会识别并停止循环。

## 四、完整生命周期流程图

```
用户: /goal "修复 tests/hermes_cli/ 所有失败测试"

  ⊙ Goal set (20-turn budget): 修复 tests/hermes_cli/ 所有失败测试
  
  [CLI/Gateway 将 goal 文本注入 _pending_input/FIFO]

  ┌─ Turn 1 ────────────────────────────────────────────────────┐
  │ Hermes: 运行测试...发现 5 个失败，修复了 2 个               │
  │ [轮次结束 Hook 触发]                                         │
  │ judge_goal(goal, last_response) → {"done": false, "reason":  │
  │   "only 2 of 5 failures fixed; 3 remain"}                   │
  │ → decision: should_continue=True                             │
  │ ↻ Continuing toward goal (1/20): only 2 of 5 fixed          │
  │ [continuation prompt 注入队列]                               │
  └──────────────────────────────────────────────────────────────┘

  ┌─ Turn 2 (continuation prompt 触发) ──────────────────────────┐
  │ Hermes: [Continuing toward your standing goal]...            │
  │ 修复了剩余 3 个测试                                           │
  │ [轮次结束 Hook 触发]                                         │
  │ judge_goal(goal, last_response) → {"done": true, "reason":   │
  │   "All 5 tests fixed, response confirms completion"}         │
  │ → decision: should_continue=False, verdict=done              │
  │ ✓ Goal achieved: All 5 tests fixed...                        │
  └──────────────────────────────────────────────────────────────┘

  用户: _ (空闲，等待新输入)
```

**其他终止路径**：

```
路径 B（预算耗尽）：
  ↻ Continuing (18/20) → ↻ (19/20) → ⏸ Goal paused — 20/20 turns used
  用户: /goal resume → 计数器重置为 0 → 继续

路径 C（用户干预）：
  ↻ Continuing (3/20) → 用户发送 "等一下，换个思路"
  → 用户消息抢占，continuation 被丢弃
  → 新轮次结束后 judge 再次运行

路径 D（受阻确认）：
  Hermes: "这个测试需要修改上游 API，我无法访问"
  judge → {"done": true, "reason": "goal is blocked, needs user input"}
  ✓ Goal achieved: goal is blocked, needs user input
```

## 五、配置方式

在 `~/.hermes/config.yaml` 中：

```yaml
goals:
  # 最大续跑轮次，默认 20。调低用于 tighter loops，调高用于长期重构
  max_turns: 20

auxiliary:
  goal_judge:
    # 可选：将 judge 路由到便宜快速的模型
    provider: openrouter
    model: google/gemini-3-flash-preview
```

## 六、设计亮点与权衡

| 设计点 | 说明 |
|--------|------|
| **Fail-open 语义** | Judge 失败 → continue，而非 block。broken judge 不会卡住进度，turn budget 是最终安全网 |
| **零侵入性** | continuation 只是普通 user 消息，不改 system prompt、不切 toolset、不碰 prompt cache |
| **持久化** | `SessionDB.state_meta` 持久化，`/resume` 可跨会话恢复 |
| **用户消息优先** | 真实用户消息天然抢占 continuation loop，不会丢失用户输入 |
| **保守判断策略** | 只有明确完成或确认受阻才 done，使 false positive 比 false negative 更罕见 |
| **双端统一** | CLI（`_pending_input`）和 Gateway（adapter-FIFO）共享同一 `GoalManager`，逻辑一致 |
| **独立辅助模型** | Judge 用独立 `goal_judge` auxiliary task，可配便宜模型，每次 ~200 token |
| **JSON 解析容错** | 多层 fallback：直接解析 → 正则提取 JSON 对象 → 剥离 markdown fence |

## 七、与 Codex `/goal` 的对比

| 维度 | Codex `/goal` | Hermes `/goal` |
|------|---------------|----------------|
| 灵感来源 | 原创（Eric Traut, OpenAI） | 受 Codex 启发的 Ralph loop 变体 |
| Judge 机制 | Agent 自身通过 `update_goal` tool 自我判断 | **独立的辅助模型**（auxiliary client）做判断 |
| 续跑注入 | Steering prompt（隐藏上下文，`InternalModelContextFragment`） | **普通 user 消息**（`_pending_input` / FIFO 队列） |
| 持久化 | SQLite `thread_goals` 表（独立表） | `SessionDB.state_meta` KV 存储（`goal:<session_id>`） |
| 预算维度 | Token budget + 挂钟时间 | **Turn budget**（轮次计数） |
| 完成审计 | 严格 completion audit + blocked audit（需 3 轮相同阻塞） | Judge 单次判断（保守策略，blocked 也视为 done） |
| 架构集成 | Extension API（ThreadLifecycle/TurnLifecycle/ToolContributor） | **GoalManager** 类 + CLI/Gateway 双端 Hook |
| 语言 | Rust | Python |

**本质区别**：Codex 把 goal 判断**内嵌在 Agent 的工具调用中**（Agent 自己调 `update_goal`），而 Hermes 用**独立的外部 Judge 模型**在每轮结束后做判断。这使得 Hermes 的实现更轻量——不需要修改 Agent 的工具集或 system prompt，但代价是 Judge 可能误判（通过保守策略和 turn budget 兜底）。

## 八、总结

Hermes Agent 的 `/goal` 是一个精心设计的**自主持续工作模式**，核心思想可以概括为：

> **设定目标 → 工作 → Judge 判断 → 未完成则自动续跑 → 循环直到 done/budget/intervention**

它的巧妙之处在于：
1. **极简实现**——仅 ~535 行核心代码，零侵入性（不改 system prompt / toolset）
2. **独立 Judge**——用辅助模型做判断，避免 Agent 自我评估的 bias
3. **Fail-open**——Judge 坏了也不卡进度，turn budget 兜底
4. **用户优先**——真实消息天然抢占 continuation loop
5. **跨平台统一**——CLI 和 Gateway 共享同一 GoalManager

它把一个"一次性问答"变成了"有明确目标、有轮次预算、有自动判断、能自主续跑的持久任务"。
