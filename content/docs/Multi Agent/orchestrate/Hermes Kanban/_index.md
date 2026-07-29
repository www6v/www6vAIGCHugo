# Hermes Agent Kanban 深度解析：原理、实现与竞品分析

> "A task board is not a UI — it's the shared memory of a multi-agent system."

Hermes Agent 的 Kanban 不是一个简单的任务管理界面，而是一个**基于 SQLite 的多 Agent 协调内核**。它通过原子任务声明、DAG 依赖门控、进程级隔离和结构化交接，让多个 Agent 实例（Profile）在一台机器上安全协作。本文从源码出发，深入解析其设计原理、实现细节，并与 Claude Code 的 Dynamic Workflow 进行系统性对比。

---

## §1 引言：为什么是 Kanban？

假设你需要一个团队完成以下工作：

1. 研究员 A 分析 vLLM、SGLang、TensorRT-LLM 的性能对比
2. 研究员 B 分析三者的许可证和社区活跃度
3. 分析师 C 综合 A 和 B 的发现，给出推荐
4. 写作者 D 把推荐写成决策备忘录

用 `delegate_task` 可以并行 A 和 B，但 C 必须等 A 和 B 都完成。如果 A 失败了需要重试怎么办？如果 C 发现 A 的输出不完整需要打回怎么办？如果整个流程需要跨越数小时甚至数天，父 Agent 的对话上下文早就溢出了怎么办？

这正是 Kanban 要解决的问题：**跨 Agent 的持久化、可重试、可审计的协作**。

---

## §2 架构全景

Hermes Kanban 的核心架构可以概括为四层：

```
┌──────────────────────────────────────────────────┐
│                  人类交互层                        │
│  CLI (hermes kanban ...)  /  Dashboard  /  Gateway │
├──────────────────────────────────────────────────┤
│              Kanban Tools (7 个)                  │
│  kanban_show / create / complete / block /        │
│  comment / heartbeat / link                        │
├──────────────────────────────────────────────────┤
│              数据库层 (SQLite)                     │
│  tasks / task_links / task_runs /                 │
│  task_comments / task_events / kanban_notify_subs │
├──────────────────────────────────────────────────┤
│              调度器 (Dispatcher)                   │
│  dispatch_once(): 回收 → 检测崩溃 → 依赖推进 → 派生│
│  _default_spawn(): hermes -p <profile> chat -q    │
└──────────────────────────────────────────────────┘
```

### 关键文件映射

| 文件 | 职责 | 行数 |
|------|------|------|
| `hermes_cli/kanban_db.py` | 数据库 Schema、CRUD、调度器、工作上下文构建 | ~2765 |
| `hermes_cli/kanban.py` | CLI 子命令（15 个 verb）、参数解析、输出格式化 | ~1393 |
| `tools/kanban_tools.py` | Agent 工具注册（7 个）、环境门控 | ~726 |

---

## §3 数据库设计：5 张表的协调原语

Kanban 的数据模型极度精简——只有 5 张核心表，却承载了整个多 Agent 协调语义。

### 3.1 核心 Schema

```sql
-- 任务表：状态机 + 声明锁 + 工作区
CREATE TABLE tasks (
    id                   TEXT PRIMARY KEY,       -- t_ + 4 hex bytes (~4.3B 空间)
    title                TEXT NOT NULL,
    body                 TEXT,
    assignee             TEXT,                   -- Profile 名称
    status               TEXT NOT NULL,           -- triage|todo|ready|running|blocked|done|archived
    priority             INTEGER DEFAULT 0,
    workspace_kind       TEXT DEFAULT 'scratch',  -- scratch | worktree | dir
    workspace_path       TEXT,
    claim_lock           TEXT,                    -- host:pid 格式的声明锁
    claim_expires        INTEGER,                 -- 过期时间戳 (默认 15 分钟)
    tenant               TEXT,                    -- 租户隔离
    current_run_id       INTEGER,                 -- 指向 task_runs 的当前活跃行
    max_runtime_seconds  INTEGER,                 -- 每任务运行时限
    spawn_failures       INTEGER DEFAULT 0,       -- 连续派生失败计数
    last_spawn_error     TEXT,
    worker_pid           INTEGER,                 -- 当前 Worker 进程 PID
    last_heartbeat_at    INTEGER,                 -- 最后心跳时间
    skills               TEXT                     -- JSON: 强制加载的技能列表
);

-- 依赖图：DAG 边
CREATE TABLE task_links (
    parent_id  TEXT NOT NULL,
    child_id   TEXT NOT NULL,
    PRIMARY KEY (parent_id, child_id)
);

-- 尝试历史：每次执行都是独立的 run
CREATE TABLE task_runs (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id             TEXT NOT NULL,
    profile             TEXT,
    status              TEXT NOT NULL,     -- running|done|blocked|crashed|timed_out|failed|reclaimed
    claim_lock          TEXT,
    claim_expires       INTEGER,
    worker_pid          INTEGER,
    max_runtime_seconds INTEGER,
    last_heartbeat_at   INTEGER,
    started_at          INTEGER NOT NULL,
    ended_at            INTEGER,
    outcome             TEXT,              -- completed|blocked|crashed|timed_out|spawn_failed|gave_up|reclaimed
    summary             TEXT,              -- 人类可读的交接摘要
    metadata            TEXT,              -- JSON: 结构化事实
    error               TEXT
);

-- 评论线程：跨 Agent 通信
CREATE TABLE task_comments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL, author TEXT NOT NULL,
    body TEXT NOT NULL, created_at INTEGER NOT NULL
);

-- 事件日志：状态变迁的审计追踪
CREATE TABLE task_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id TEXT NOT NULL, run_id INTEGER,
    kind TEXT NOT NULL, payload TEXT, created_at INTEGER NOT NULL
);
```

### 3.2 设计权衡

| 设计决策 | 选择 | 理由 | 代价 |
|----------|------|------|------|
| 数据库 | SQLite (WAL 模式) | 零配置、单文件、内置并发控制 | 不支持多机分布式部署 |
| 并发控制 | `BEGIN IMMEDIATE` + CAS | SQLite 的 WAL 锁天然序列化写者 | 写吞吐量受限 |
| ID 生成 | `secrets.token_hex(4)` | 短 ID，人类可读 | 生日悖论冲突率（4.3B 空间可接受） |
| 状态字段 | 单列 `status` | 简单直观 | 需要额外的 `claim_lock`/`claim_expires` 辅助列 |

**为什么不用 PostgreSQL/Redis？** 因为 Kanban 的设计假设是**单机多 Profile** 场景——所有 Agent 实例运行在同一台机器上，通过 Profile 隔离。SQLite 的 WAL 模式 + `BEGIN IMMEDIATE` 已经提供了足够的并发保障：

```python
# kanban_db.py: write_txn()
@contextlib.contextmanager
def write_txn(conn: sqlite3.Connection):
    conn.execute("BEGIN IMMEDIATE")  # 立即获取写锁
    try:
        yield conn
    except Exception:
        conn.execute("ROLLBACK")
        raise
    else:
        conn.execute("COMMIT")
```

`BEGIN IMMEDIATE` 确保事务开始时立即获取写锁——如果有另一个写者在持有锁，当前事务会阻塞等待而非立即失败。这与 Redis 的乐观锁（WATCH/MULTI/EXEC）不同，后者在冲突时需要客户端重试。

### 3.3 幂等性设计

```python
# kanban_db.py: create_task()
if idempotency_key:
    row = conn.execute(
        "SELECT id FROM tasks WHERE idempotency_key = ? AND status != 'archived' "
        "ORDER BY created_at DESC LIMIT 1",
        (idempotency_key,),
    ).fetchone()
    if row:
        return row["id"]  # 返回已存在的任务 ID
```

这对于 Webhook 驱动的自动化场景至关重要——重试的 Webhook 不会创建重复任务。

---

## §4 状态机：7 种状态的精确语义

Kanban 任务的生命周期不是简单的 TODO → DONE，而是一个精心设计的 7 状态机：

```
                    ┌─────────┐
                    │ triage  │ ◄── 待整理的需求
                    └────┬────┘
                         │ 人工推进
                    ┌────▼────┐
               ┌─── │  todo   │ ◄── 有未完成的前置依赖
               │    └────┬────┘
               │         │ 所有 parents done
               │    ┌────▼────┐
               │    │  ready  │ ◄── 可被调度器认领
               │    └────┬────┘
               │         │ claim_task() CAS
               │    ┌────▼────┐
               ├───►│ running │ ──┐
               │    └────┬────┘   │
               │         │        │
          ┌────▼────┐ ┌──▼───┐ ┌──▼─────┐
          │  done   │ │blocked│ │crashed │
          └────┬────┘ └───┬───┘ └───┬────┘
               │           │         │
               │      unblock│  reclaim│
               │           │         │
               │    ┌──────▼─────────▼────┐
               │    │     ready           │
               │    └─────────────────────┘
               │
          ┌────▼────┐
          │archived │ ◄── 生命周期终结
          └─────────┘
```

### 4.1 状态转移的原子性

每个状态转移都在 `write_txn` 中通过 CAS（Compare-And-Swap）保证原子性：

```python
# kanban_db.py: claim_task()
cur = conn.execute(
    """
    UPDATE tasks
       SET status        = 'running',
           claim_lock    = ?,       -- "hostname:pid"
           claim_expires = ?,       -- now + 900 (15分钟)
           started_at    = COALESCE(started_at, ?)
     WHERE id = ?
       AND status = 'ready'         -- 前置条件：必须是 ready
       AND claim_lock IS NULL       -- 前置条件：没有其他人在 claim
    """,
    (lock, expires, now, task_id),
)
if cur.rowcount != 1:
    return None  # 被其他人抢先了，返回 None
```

**为什么需要 `claim_lock IS NULL` 这个条件？** 因为 `status = 'ready'` 可能被多个并发调度器同时看到。SQLite 的 WAL 锁会序列化写操作，但第一个成功写的事务会把 `claim_lock` 设为非 NULL，后续事务的 `UPDATE` 就会匹配 0 行——这就是 CAS 的本质。

### 4.2 `task_runs` 表：从"任务"到"尝试"的范式转换

v1 版本的 Kanban 没有 `task_runs` 表——任务状态直接写在 `tasks` 表上。后来引入了 `task_runs`，这是一个重要的范式转变：

| 维度 | 仅有 tasks 表 | 引入 task_runs 后 |
|------|-------------|-----------------|
| 重试语义 | 无法区分"第一次失败"和"重试中" | 每次尝试都有独立行 |
| 交接数据 | summary 覆盖写入 | 每次尝试有自己的 summary/metadata |
| 调试 | 只知道"失败了" | 知道第几次失败、什么 outcome、什么 error |
| 统计 | 无法计算平均执行时间 | 每个 run 有 started_at/ended_at |

**当前运行指针**（`tasks.current_run_id`）是一个巧妙的设计——它把任务表和运行表关联起来，让读取当前状态只需查一行，同时保留完整的历史记录。

---

## §5 调度器：四步调度循环

调度器是 Kanban 的心脏，嵌入在 Gateway 进程中运行（旧版是独立 daemon，已废弃）。每次 tick 执行四步：

```python
# kanban_db.py: dispatch_once()
def dispatch_once(conn, *, spawn_fn=None, ...) -> DispatchResult:
    result = DispatchResult()

    # Step 1: 回收过期的声明锁（TTL 超时）
    result.reclaimed = release_stale_claims(conn)

    # Step 2: 检测崩溃的 Worker（PID 不再存活）
    result.crashed = detect_crashed_workers(conn)

    # Step 3: 提升依赖已满足的任务 (todo → ready)
    result.promoted = recompute_ready(conn)

    # Step 4: 派生 ready 任务的 Worker 进程
    for row in ready_rows:
        claimed = claim_task(conn, row["id"], ttl_seconds=ttl_seconds)
        workspace = resolve_workspace(claimed)
        pid = spawn_fn(claimed, str(workspace))  # → hermes -p <profile> chat -q
        _set_worker_pid(conn, claimed.id, pid)
```

### 5.1 Step 1: 声明锁回收

```python
# 找到所有 claim_expires < now 的 running 任务
stale = conn.execute(
    "SELECT id, claim_lock FROM tasks "
    "WHERE status = 'running' AND claim_expires < ?",
    (now,),
).fetchall()
for row in stale:
    # 回滚到 ready，关闭当前 run
    conn.execute(
        "UPDATE tasks SET status = 'ready', claim_lock = NULL, "
        "claim_expires = NULL, worker_pid = NULL WHERE id = ?",
        (row["id"],),
    )
    _end_run(conn, row["id"], outcome="reclaimed", ...)
```

默认 TTL 是 **15 分钟**。Worker 可以通过 `heartbeat_claim()` 续期。

### 5.2 Step 2: 崩溃检测

```python
def _pid_alive(pid: Optional[int]) -> bool:
    try:
        os.kill(int(pid), 0)  # signal 0 = 检查进程是否存在
    except ProcessLookupError:
        return False
    except PermissionError:
        return True  # 存在但无权 signal
    # Linux 额外检查 zombie 状态
    if sys.platform == "linux":
        with open(f"/proc/{pid}/status", "r") as f:
            for line in f:
                if line.startswith("State:") and "Z" in line:
                    return False  # zombie = 已退出但未 reap
    return True
```

**Zombie 处理是一个精妙细节**。`os.kill(pid, 0)` 对 zombie 进程返回成功（因为进程表项还在），但 zombie 实际上已经死了。Hermes 通过检查 `/proc/<pid>/status` 的 `State:` 行来识别 zombie。

### 5.3 Step 3: 依赖推进

```python
def recompute_ready(conn: sqlite3.Connection) -> int:
    promoted = 0
    with write_txn(conn):
        for row in conn.execute("SELECT id FROM tasks WHERE status = 'todo'"):
            task_id = row["id"]
            parents = conn.execute(
                "SELECT t.status FROM tasks t JOIN task_links l "
                "ON l.parent_id = t.id WHERE l.child_id = ?",
                (task_id,),
            ).fetchall()
            if all(p["status"] == "done" for p in parents):
                conn.execute(
                    "UPDATE tasks SET status = 'ready' "
                    "WHERE id = ? AND status = 'todo'",
                    (task_id,),
                )
                promoted += 1
    return promoted
```

这个算法的时间复杂度是 **O(T × P)**，其中 T 是 todo 任务数，P 是平均父任务数。在典型场景下（几十个任务，每个任务 1-3 个父任务），这个开销几乎可以忽略。

### 5.4 Step 4: 派生 Worker

```python
def _default_spawn(task: Task, workspace: str) -> Optional[int]:
    cmd = [
        "hermes",
        "-p", task.assignee,
        "--skills", "kanban-worker",  # 自动加载 worker 技能
        "chat",
        "-q", f"work kanban task {task.id}",
    ]
    env = {
        "HERMES_KANBAN_TASK": task.id,
        "HERMES_KANBAN_WORKSPACE": workspace,
        "HERMES_PROFILE": task.assignee,
    }
    proc = subprocess.Popen(
        cmd,
        cwd=workspace,
        stdin=subprocess.DEVNULL,
        stdout=log_f,          # 输出重定向到日志
        stderr=subprocess.STDOUT,
        env=env,
        start_new_session=True, # 独立进程组
    )
    return proc.pid
```

**关键设计决策**：
- `start_new_session=True` 创建新的进程组——父进程退出后 Worker 不会收到 SIGHUP
- 输出重定向到 `$HERMES_HOME/kanban/logs/<task_id>.log`——单代轮换，2MB 上限
- `stdin=DEVNULL`——Worker 不接收终端输入（fire-and-forget 模式）

### 5.5 派生失败熔断

```python
def _record_spawn_failure(conn, task_id, error, failure_limit=5):
    failures = row["spawn_failures"] + 1
    if failures >= failure_limit:
        # 超过限制 → 自动 block
        conn.execute(
            "UPDATE tasks SET status = 'blocked', "
            "spawn_failures = ?, last_spawn_error = ? WHERE id = ?",
            (failures, error[:500], task_id),
        )
        _append_event(conn, task_id, "gave_up", {...})
    else:
        # 重置为 ready，下次 tick 重试
        conn.execute(
            "UPDATE tasks SET status = 'ready', spawn_failures = ?",
            (failures, task_id),
        )
```

默认 **5 次连续失败后自动 block**，防止调度器对无法修复的任务无限重试。

---

## §6 Kanban Tools：Agent 的结构化交接面

Kanban 工具**只在被调度器派生的 Worker 进程中可用**。普通 `hermes chat` 会话看不到这些工具。

### 6.1 环境门控

```python
# kanban_tools.py
def _check_kanban_mode() -> bool:
    return bool(os.environ.get("HERMES_KANBAN_TASK"))
```

只有环境变量 `HERMES_KANBAN_TASK` 存在时（由 `_default_spawn` 设置），工具才会注册到模型的工具 schema 中。这是**最小权限原则**的体现——普通会话不需要也不应该看到 Kanban 工具。

### 6.2 七种工具

| 工具 | 用途 | 必需参数 |
|------|------|---------|
| `kanban_show` | 读取任务完整状态 | `task_id`（可选，默认取 env） |
| `kanban_create` | 创建子任务（Orchestrator 用） | `title`, `assignee` |
| `kanban_complete` | 标记任务完成 | `summary` 或 `result` |
| `kanban_block` | 阻塞任务等待人工输入 | `reason` |
| `kanban_comment` | 添加评论到线程 | `body` |
| `kanban_heartbeat` | 发送存活信号 | 无 |
| `kanban_link` | 添加依赖边 | `parent_id`, `child_id` |

### 6.3 结构化交接

`kanban_complete` 的设计特别值得注意——它区分了人类可读的 `summary` 和机器可读的 `metadata`：

```python
kanban_complete(
    summary="shipped rate limiter — token bucket, keys on user_id with IP fallback, 14 tests pass",
    metadata={
        "changed_files": ["rate_limiter.py", "tests/test_rate_limiter.py"],
        "tests_run": 14,
        "tests_passed": 14,
        "decisions": ["user_id primary, IP fallback for unauthenticated requests"],
    },
)
```

下游 Agent 通过 `build_worker_context()` 读取这些交接数据：

```python
# 父任务的 run.summary 和 run.metadata 会被注入到子任务的 prompt 中
## Parent task results
### t_a3f2b1c7
shipped rate limiter — token bucket...
_metadata_: `{"changed_files": [...], "tests_run": 14, ...}`
```

---

## §7 依赖图：DAG 构建与循环检测

### 7.1 创建时的依赖检查

```python
def create_task(conn, *, title, parents=(), ...):
    with write_txn(conn):
        # 检查父任务是否存在
        missing = _find_missing_parents(conn, parents)
        if missing:
            raise ValueError(f"unknown parent task(s): {', '.join(missing)}")

        # 判断初始状态
        if parents:
            rows = conn.execute(
                "SELECT status FROM tasks WHERE id IN (...) ", parents
            ).fetchall()
            if any(r["status"] != "done" for r in rows):
                initial_status = "todo"  # 有未完成的父任务
            else:
                initial_status = "ready"  # 所有父任务都已完成
        else:
            initial_status = "ready"
```

### 7.2 循环检测（BFS）

```python
def _would_cycle(conn, parent_id, child_id) -> bool:
    """如果 parent_id 已经是 child_id 的后代，则形成环"""
    seen = set()
    stack = [child_id]
    while stack:
        node = stack.pop()
        if node == parent_id:
            return True
        if node in seen:
            continue
        seen.add(node)
        rows = conn.execute(
            "SELECT child_id FROM task_links WHERE parent_id = ?", (node,)
        ).fetchall()
        stack.extend(r["child_id"] for r in rows)
    return False
```

这是一个标准的 **BFS 可达性检查**，时间复杂度 O(V+E)。在典型场景下（每个任务 1-3 个依赖），这个检查几乎瞬时完成。

### 7.3 后置链接的自动降级

```python
def link_tasks(conn, parent_id, child_id):
    # 检查循环...
    conn.execute("INSERT OR IGNORE INTO task_links ...", (parent_id, child_id))

    # 如果父任务未完成，子任务从 ready 降级到 todo
    parent_status = conn.execute(
        "SELECT status FROM tasks WHERE id = ?", (parent_id,)
    ).fetchone()["status"]
    if parent_status != "done":
        conn.execute(
            "UPDATE tasks SET status = 'todo' WHERE id = ? AND status = 'ready'",
            (child_id,),
        )
```

这个设计允许**动态构建 DAG**——你可以在任务已经 `ready` 后再添加依赖，系统会自动将其降级为 `todo`。

---

## §8 Worker 上下文构建：Prompt 的预算管理

`build_worker_context()` 是 Kanban 最精巧的部分之一——它把任务状态、历史、父任务结果、评论等组装成 Worker Agent 的 prompt，同时有严格的字节上限：

```python
_CTX_MAX_PRIOR_ATTEMPTS = 10      # 最多显示 10 次先前尝试
_CTX_MAX_COMMENTS       = 30      # 最多显示 30 条评论
_CTX_MAX_FIELD_BYTES    = 4 * 1024   # 每个字段 4KB
_CTX_MAX_BODY_BYTES     = 8 * 1024   # task.body 8KB
_CTX_MAX_COMMENT_BYTES  = 2 * 1024   # 每条评论 2KB
```

**上下文组装顺序**：

1. 任务标题 + 元数据（assignee、status、workspace）
2. task.body（需求描述）
3. 本任务的先前尝试（最近的 N 次完整展示，更早的折叠为一行标记）
4. 已完成父任务的交接结果（优先取 run.summary/metadata，回退到 task.result）
5. 该 assignee 的跨任务角色历史（最近 5 次完成的 run）
6. 评论线程（最近的 N 条完整展示，更早的折叠）

```python
# 跨任务角色历史：给 Worker 隐式的连续性
## Recent work by @reviewer
- t_x1y2z3 — Review PR #456 (2026-06-08): found 2 SQL injection issues
- t_a4b5c6 — Review PR #789 (2026-06-07): approved with minor suggestions
- t_d7e8f9 — Review auth module (2026-06-06): blocked on missing CSRF
```

这种设计让 Agent **无需显式记忆配置**就能获得工作连续性——它能看到自己（或同 profile 的 Agent）最近做了什么。

---

## §9 工作区隔离：三种 Workspace 模式

| 模式 | 路径 | 生命周期 | 适用场景 |
|------|------|---------|---------|
| `scratch` | `$HERMES_HOME/kanban/workspaces/<id>/` | 任务归档时 GC | 研究、分析、一次性脚本 |
| `dir:<path>` | 用户指定的绝对路径 | 持久化 | 需要跨任务共享数据的场景 |
| `worktree` | `.worktrees/<id>/` 或指定路径 | 随 git 生命周期 | 并行代码开发 |

**安全约束**：`dir:` 和 `worktree` 模式强制要求**绝对路径**——相对路径被拒绝，防止 confused-deputy 攻击（`../../../tmp/attacker` 相对于调度器的 CWD 解析）。

---

## §10 通知系统：Gateway 集成的 Human-in-the-Loop

```sql
CREATE TABLE kanban_notify_subs (
    task_id       TEXT NOT NULL,
    platform      TEXT NOT NULL,          -- telegram, discord, dingtalk...
    chat_id       TEXT NOT NULL,
    thread_id     TEXT NOT NULL DEFAULT '',
    user_id       TEXT,
    last_event_id INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (task_id, platform, chat_id, thread_id)
);
```

Gateway 的 kanban-notifier watcher 轮询 `task_events` 表，将 `completed`/`blocked` 事件推送到原始的请求者——实现了**跨平台的人工介入闭环**。

---

## §11 竞品对比：Hermes Kanban vs Claude Code Dynamic Workflow

### 11.1 Claude Code 多任务体系概览

Claude Code（131,154⭐）提供了四种多任务处理方式，Dynamic Workflow 是其中最复杂的一种：

| 方式 | 抽象级别 | 适合场景 |
|------|---------|---------|
| **Subagents** | 单个子任务 | 并行执行独立子任务（代码审查、研究） |
| **Agent View** | 集中监控面板 | 从一个屏幕管理多个 Claude Code 会话 |
| **Agent Teams** | 团队协调 | 多个 Claude Code 实例作为"团队"协作，共享任务、跨 Agent 消息传递、集中管理 |
| **Dynamic Workflows** | 脚本化编排 | 大规模子 Agent 编排——Claude 编写脚本，用户可以重复运行 |

**Dynamic Workflow 的核心特征**（来自 Claude Code 官方文档）：

> "Dynamic workflows orchestrate many subagents from a script Claude writes and you can rerun. Use them for codebase audits, large migrations, and cross-checked research."

关键机制：
1. **Claude 编写脚本**：用户描述目标，Claude 生成编排脚本
2. **脚本派生子 Agent**：脚本通过 Claude Agent SDK 或 subagent API 派生多个子 Agent
3. **可重复运行**：脚本可以保存并在未来重新执行
4. **跨检查研究**：多个子 Agent 独立分析同一问题，结果交叉验证

### 11.2 深度对比

| 维度 | Hermes Kanban | Claude Code Dynamic Workflow |
|------|--------------|------------------------------|
| **协调范式** | 任务板 + DAG 调度器 | Claude 编写的编排脚本 |
| **核心抽象** | Task（有状态、有依赖、有历史） | Script（一次性执行的编排逻辑） |
| **持久化** | SQLite（永久，5 张表） | 脚本文件（可重复运行，无内置状态存储） |
| **依赖管理** | DAG + 自动门控（`parents=[]`） | 脚本内手动编排（由 Claude 在脚本中定义顺序） |
| **任务隔离** | Profile + Workspace + 独立进程 | Subagent（共享上下文，SDK 级隔离） |
| **重试机制** | 自动重试 + 熔断（5 次失败自动 block） | 取决于脚本实现 |
| **崩溃恢复** | PID 检测 + TTL 过期自动回收 | 取决于脚本实现 |
| **人工介入** | `kanban_block()` + Gateway 实时推送 | Claude 请求输入 + 终端交互 |
| **审计追踪** | `task_events` 永久事件日志 | 终端输出/日志文件 |
| **运行时** | 进程级（`subprocess.Popen`） | SDK 级（Python/TypeScript 进程内） |
| **动态 DAG** | 运行时创建任务、添加依赖 | 脚本运行时可动态派生，但无显式依赖图 |
| **结构化交接** | `summary`（人类）+ `metadata`（机器） | 取决于脚本约定 |
| **跨平台通知** | Telegram/Discord/DingTalk 实时推送 | 无内置跨平台通知 |
| **运行时长** | 小时/天（进程隔离） | 分钟（SDK 进程内，受上下文限制） |
| **编程接口** | CLI + 7 个工具调用 | Agent SDK (Python/TypeScript) |
| **开源** | MIT | Claude Code 闭源（SDK 开源） |

### 11.3 架构差异的根本原因

**Hermes Kanban 的设计哲学**：

```
任务板 = 共享记忆体（Shared Memory）
调度器 = 自动协调内核（Coordination Kernel）
Profile = Agent 身份（Agent Identity）
```

Kanban 把**任务状态**和**Agent 执行**完全解耦——调度器不知道 Agent 在做什么，只关心任务的状态变迁。Agent 通过 `kanban_complete` 报告完成，通过 `kanban_show` 读取上下文。这种设计使得：

- 一个 Agent 崩溃不影响其他 Agent
- 调度器重启后可以从数据库恢复状态
- 任务历史永久保存

**Claude Dynamic Workflow 的设计哲学**：

```
脚本 = 编排逻辑（Orchestration Logic）
Subagent = 执行单元（Execution Unit）
SDK = 运行环境（Runtime Environment）
```

Dynamic Workflow 把编排逻辑交给**脚本**——Claude 编写一个 Python/TypeScript 脚本来派生和管理子 Agent。这更接近**编程式编排**（Programmatic Orchestration），而非声明式任务板。

### 11.4 各自的优势场景

**Hermes Kanban 更适合**：

| 场景 | 为什么 |
|------|--------|
| 需要任务持久化和审计 | SQLite 永久存储 + 事件日志 |
| 需要自动重试和崩溃恢复 | PID 检测 + TTL + 熔断 |
| 需要动态依赖图 | 运行时创建任务、DAG 自动推进 |
| 需要人工介入 | `kanban_block` + 跨平台实时通知 |
| 需要跨天/跨周的长周期工作 | 进程隔离 + 数据库持久化 |

**Claude Dynamic Workflow 更适合**：

| 场景 | 为什么 |
|------|--------|
| 一次性大规模代码审计 | Claude 编写脚本，运行一次即可 |
| 需要快速原型的编排 | 自然语言描述 → Claude 生成脚本 |
| 需要精细的 SDK 级控制 | Python/TypeScript SDK 完全可编程 |
| 在 Claude Code 生态内闭环 | 与 CLAUDE.md、Skills、MCP 无缝集成 |
| 需要可重复运行的编排脚本 | 脚本可保存、版本控制、重复执行 |

### 11.5 对比总结

```
                    持久化 ◄────────────────────► 临时性
                       │                              │
Hermes Kanban ─────────┤                              ├──── Dynamic Workflow
(任务板 = 数据库)       │                              │ (脚本 = 代码)
                       │                              │
                    声明式 ◄────────────────────► 编程式
                 (定义任务 + 依赖)                (编写编排逻辑)
                       │                              │
                    自动协调                         手动编排
                  (调度器驱动)                      (脚本驱动)
```

Hermes Kanban 是**声明式**的——你定义任务和依赖，调度器自动协调。Claude Dynamic Workflow 是**编程式**的——你（或 Claude）编写脚本，脚本负责一切。

---

## §12 总结

### 核心要点

- **Hermes Kanban 是一个 SQLite-backed 的多 Agent 协调内核**，不是 UI 层
- **5 张表承载所有协调语义**：tasks（状态机）、task_links（DAG）、task_runs（尝试历史）、task_comments（通信）、task_events（审计）
- **四步调度循环**：回收过期锁 → 检测崩溃 → 依赖推进 → 派生 Worker
- **CAS 原子性**：通过 `BEGIN IMMEDIATE` + `WHERE status = 'ready' AND claim_lock IS NULL` 保证并发安全
- **进程级隔离**：每个 Worker 是独立的 `hermes -p <profile>` 子进程，崩溃不影响其他任务
- **结构化交接**：`summary`（人类可读）+ `metadata`（机器可读）区分
- **动态 DAG**：运行时可创建任务、添加依赖，系统自动处理状态推进
- **派生熔断**：5 次连续失败后自动 block，防止无限重试

### Hermes Kanban 的独特优势

1. **持久化即默认**——SQLite 永久存储，不是可选插件
2. **进程隔离**——每个任务独立进程，崩溃自动重试
3. **完整的审计追踪**——`task_events` 记录每个状态变迁
4. **声明式 DAG**——定义任务和依赖，调度器自动协调
5. **跨平台人工介入**——通过 Gateway 推送通知到 Telegram/Discord/DingTalk

### 局限

1. **单机设计**——SQLite 不支持多机分布式部署（但可以通过 NFS 共享文件系统解决）
2. **轮询调度**——Dispatcher 每 60 秒 tick 一次，不是事件驱动（低延迟场景不够理想）
3. **无内置负载均衡**——所有任务在同一台机器上派生

### 行动建议

- **需要持久化、审计、自动重试的长周期多 Agent 协作** → Hermes Kanban
- **需要快速原型、一次性大规模编排、SDK 级精细控制** → Claude Dynamic Workflow
- **混合使用**：Claude 编写编排脚本（Dynamic Workflow）+ Hermes Kanban 作为持久化任务板

### 延伸阅读

- Hermes Kanban 设计文档: `docs/hermes-kanban-v1-spec.pdf`
- Kanban 教程: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial
- `kanban-orchestrator` 技能: `~/.hermes/skills/devops/kanban-orchestrator/SKILL.md`
- `kanban-worker` 技能: `~/.hermes/skills/devops/kanban-worker/SKILL.md`
- Claude Code Dynamic Workflow: https://code.claude.com/docs/en/dynamic_workflow
- Claude Code Subagents: https://code.claude.com/docs/en/subagents
- Claude Agent SDK: https://code.claude.com/docs/en/agent-sdk
