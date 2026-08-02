# Harness 工程深潜：从 Agent Loop 到生产级架构的实战拆解——OpenClaw vs Claude Code vs Hermes Agent

> 写在前面的话：本文不是概念科普，而是从源码和工程实践出发，拆解"Harness"到底是什么、怎么实现的、有哪些坑。以 **OpenClaw**、**Claude Code** 和 **Hermes Agent** 三个典型实现为案例，穿插通用设计模式。

---

## 一、什么是 Harness

在大模型应用开发中，"Harness"（线束/支架）指的是**围绕 LLM API 调用构建的工程基础设施层**。它把裸模型变成可用的 Agent 系统。

如果把 LLM 比作引擎，Harness 就是底盘、传动系统、仪表板和安全气囊的总和。没有它，引擎再好也跑不起来。

一个完整的 Harness 至少包含以下模块：

```
┌─────────────────────────────────────────────┐
│                  Harness                     │
│  ┌──────────┐  ┌─────────┐  ┌────────────┐  │
│  │Agent Loop│  │Tool     │  │Context     │  │
│  │(ReAct)   │  │Dispatch │  │Management  │  │
│  └────┬─────┘  └────┬────┘  └─────┬──────┘  │
│       │              │              │         │
│  ┌────┴──────────────┴──────────────┴──────┐ │
│  │          Session & State Layer          │ │
│  └─────────────────────────────────────────┘ │
│  ┌──────────┐  ┌─────────┐  ┌────────────┐  │
│  │Security &│  │Multi-   │  │Observ-     │  │
│  │Sandbox   │  │Agent    │  │ability     │  │
│  │          │  │Routing  │  │            │  │
│  └──────────┘  └─────────┘  └────────────┘  │
└─────────────────────────────────────────────┘
         │
         ▼
   ┌───────────┐
   │  LLM API  │
   └───────────┘
```

每个框都不是概念——都是要实打实写代码的子系统。下面逐个拆解。

---

## 二、Agent Loop：ReAct 范式的工程化

### 2.1 理论原型

Agent Loop 的核心是 ReAct 范式（Reason + Act），源自 Yao et al. 的论文：

> **ReAct: Synergizing Reasoning and Acting in Language Models** (2022)
> https://arxiv.org/abs/2210.03629

核心思想：模型不只输出文本，还输出"思考→行动→观察"的循环。每次行动后，把观察结果喂回模型，让它决定下一步。

### 2.2 Claude Code 的实现

Claude Code Agent SDK 的实现非常清晰，文档直接给出了伪代码级别的说明（来源：[How the agent loop works](https://code.claude.com/docs/en/agent-sdk/agent-loop.md)）：

```
1. Receive prompt → SDK yield SystemMessage(init)
2. Evaluate and respond → AssistantMessage(text + tool_calls)
3. Execute tools → 每个 tool call 执行，结果返回给 Claude
4. Repeat → 步骤 2-3 循环，直到 no tool calls
5. Return result → ResultMessage(最终文本 + cost + session_id)
```

**Know-how 1：消息类型的状态机设计**

Claude Code 定义了 5 种核心消息类型驱动整个循环：

| 类型 | 触发时机 | 作用 |
|------|---------|------|
| `SystemMessage` | 会话开始/压缩边界 | 元数据、生命周期信号 |
| `AssistantMessage` | 每轮 Claude 响应后 | 携带 tool calls + 文本 |
| `UserMessage` | 每次工具执行后 | 工具结果回灌 |
| `StreamEvent` | 实时流式输出 | 前端进度展示 |
| `ResultMessage` | 循环结束 | 最终结果 + 费用 |

这种设计的工程价值在于：**消息即状态**。不需要额外的状态机变量，整个循环的状态通过消息类型自然表达。

**Know-how 2：Parallel vs Sequential Tool 执行**

Claude Code 对工具做了读写分类：
- **只读工具**（Read, Glob, Grep）→ 并发执行
- **写操作工具**（Edit, Write, Bash）→ 串行执行

这是防止竞态条件的关键设计。自定义工具默认串行，需要显式标记 `readOnly` 才能并发。

### 2.3 Hermes Agent 的实现

Hermes Agent 的 Agent Loop 核心在 `run_agent.py` 中的 `AIAgent.run_conversation()`（约 10,700 行）：

```
User input → prompt_builder.build_system_prompt()
            → runtime_provider.resolve_runtime_provider()
            → API call (chat_completions / codex_responses / anthropic_messages)
            → tool_calls? → model_tools.handle_function_call() → loop
            → final response → display → save to SessionDB
```

**Know-how 11：Provider Runtime Resolution 抽象层**

Hermes Agent 通过 `runtime_provider.py` 抽象了 3 种 API 模式：
- `chat_completions`（OpenAI 兼容接口）
- `codex_responses`（OpenAI CodeX 格式）
- `anthropic_messages`（Anthropic 原生格式）

这意味着同一个 Agent Loop 可以跑在任何 LLM 上——不需要为每个 provider 写单独的调用逻辑。只需在 `runtime_provider.py` 里注册新 provider 的凭证和 API mode 即可。

**Know-how 12：SQLite + FTS5 全文检索作为会话存储**

不同于 OpenClaw 的文件系统和 Claude Code 的 session_id 跟踪，Hermes Agent 用 SQLite 存储会话状态，并启用 FTS5 全文索引。这带来了两个关键能力：
- **跨会话搜索**：用自然语言搜索过去的对话，然后让 LLM 对搜索结果做摘要
- **结构化查询**：按时间、用户、主题过滤会话历史

### 2.4 OpenClaw 的实现

OpenClaw 的 Agent Loop 藏在 `dispatch-D3I9DBNN.js` 和 `pi-embedded-runner` 模块中。核心入口是 `withReplyDispatcher(params)`：

```typescript
async function withReplyDispatcher(params) {
  try {
    return await params.run();  // 实际的 Agent Loop
  } finally {
    params.dispatcher.markComplete();
    await params.dispatcher.waitForIdle();
    await params.onSettled?.();
  }
}
```

**Know-how 3：入站去重（Inbound Dedupe）**

OpenClaw 在 Agent Loop 外层做了精细的入站去重：

```typescript
function buildInboundDedupeKey(ctx) {
  return [
    provider,      // 渠道来源
    accountId,     // 账号
    sessionScope,  // 会话作用域
    peerId,        // 发送方
    threadId,      // 线程
    messageId      // 消息ID
  ].filter(Boolean).join("|");
}
```

三种去重状态：`claimed`（新消息）/ `duplicate`（重复）/ `inflight`（处理中）。TTL 默认 20 分钟，最大 5000 条。

这在多消息渠道场景下是必需的——网络重试、用户重复发送、多端同时在线都会导致重复消息。

---

## 三、Tool Dispatch：工具调用的工程实现

### 3.1 工具定义层

OpenClaw 定义了 `AgentHarness` 接口，包含：
- `AgentHarnessAttemptParams` / `AgentHarnessAttemptResult` —— 单次尝试的参数和结果
- `AgentHarnessCompactParams` / `AgentHarnessCompactResult` —— 上下文压缩
- `AgentHarnessResetParams` —— 会话重置
- `AnyAgentTool` —— 通用工具抽象
- `NormalizedUsage` —— 标准化用量统计

这些类型定义了工具调用的完整生命周期。

### 3.2 Hermes Agent 的工具系统

Hermes Agent 有 47 个内置工具，组织在 19 个 toolsets 中：

| 分类 | 工具 | 说明 |
|------|------|------|
| 文件操作 | read_file, write_file, patch, search_files | 文件读写修改和搜索 |
| Web | web_search, web_extract | 网页搜索和内容提取 |
| 浏览器 | 10 个 browser 工具 | 浏览器自动化（5 种后端） |
| 代码执行 | execute_code | 沙箱代码执行 |
| 终端 | 终端编排（6 种后端） | Shell 命令、Git 操作 |
| 委托 | delegate | 子 Agent 委托调用 |
| MCP | mcp_tool (~2,200 行) | MCP 服务器动态接入 |
| 记忆 | memory (add/replace/remove) | 跨会话记忆管理 |
| 视觉 | vision | 图像理解 |
| 编排 | skills, /model, /personality | 技能管理和模型切换 |

**Know-how 13：Toolsets 分组 + 平台预设**

Hermes Agent 不是简单地启用/禁用单个工具，而是通过 `toolsets.py` 定义工具分组，并为不同平台（CLI vs 消息渠道）预设不同的 toolset 组合。比如 CLI 模式下启用全部 47 个工具，而在 WhatsApp 模式下可能只启用核心文件操作和记忆工具。

### 3.3 Claude Code 的工具分类

Claude Code 的工具分为 6 大类（来源：[Agent SDK 文档](https://code.claude.com/docs/en/agent-sdk/agent-loop.md#built-in-tools)）：

| 分类 | 工具 | 说明 |
|------|------|------|
| 文件操作 | Read, Edit, Write | 文件读写修改 |
| 搜索 | Glob, Grep | 文件查找、正则搜索 |
| 执行 | Bash | Shell 命令、Git 操作 |
| 网络 | WebSearch, WebFetch | 网页搜索和抓取 |
| 发现 | ToolSearch | 按需发现工具 |
| 编排 | Agent, Skill, AskUserQuestion, TodoWrite | 子 Agent、技能、询问用户 |

**Know-how 14：ToolSearch 的按需加载**

当工具数量超过上下文窗口承载能力时（比如 MCP 服务器提供了 100+ 工具），Claude Code 使用 `ToolSearch` 做动态发现和加载——不是一次性把所有工具描述塞进 prompt，而是按需查找和加载。

这是**工具搜索（Tool Search）** 模式的核心：用一次额外的 LLM 调用来决定"当前需要哪些工具"，而不是用巨大的工具列表污染上下文。

---

## 四、Context Management：上下文窗口管理

### 4.1 Compaction（压缩）

Agent 会话越长，上下文窗口越大，直到触及上限。三种解决方案：

**方案 A：Compaction（摘要压缩）**

OpenClaw 通过 `AgentHarnessCompactParams` 触发压缩——让模型对已有对话生成摘要，替换原始消息。

Claude Code 用 `SystemMessage(subtype="compact_boundary")` 标记压缩边界，压缩后上下文窗口被清理，只保留摘要。

**方案 B：Session Fork（分支）**

Claude Code 支持 `fork`——复制当前会话历史创建新会话，原会话不变。用于"试试另一个方向但不丢失当前进度"的场景。

**方案 C：ContextEngine 可插拔（Hermes Agent）**

Hermes Agent 定义了 `ContextEngine` 抽象基类（`context_engine.py`），默认实现是 `context_compressor.py`（有损摘要压缩）。同时支持 Anthropic 的 prompt caching 来保留部分上下文不被清除。
这种设计的价值在于：如果你有更好的压缩策略（比如语义分块、关键决策点保留），只需实现 `ContextEngine` 接口并替换即可。

### 4.2 Know-how 5：上下文管理的成本权衡

| 策略 | Token 成本 | 信息丢失 | 适用场景 |
|------|-----------|---------|---------|
| 全量保留 | 最高 | 无 | 短会话（<50轮） |
| Compaction | 中（压缩轮额外消耗） | 部分细节丢失 | 长会话 |
| Fork | 双倍存储 | 无 | 分支探索 |
| Truncate | 最低 | 最老的消息丢失 | 成本敏感场景 |

实战建议：
- **默认用 Compaction**，在摘要中保留关键决策和代码变更
- **重要操作前手动 checkpoint**（Claude Code 支持文件级 checkpoint）
- **预算敏感的系统设置 `max_turns` 和 `max_budget_usd`**

---

## 五、Session & State：会话管理

### 5.1 OpenClaw 的会话模型

OpenClaw 的会话是**多维的**：
- `sessionKey` 编码了 agentId + channel + peer 信息
- 支持 **主会话**（main session）和 **子 Agent 会话**（subagent session）
- 通过 `isSubagentSessionKey` 区分
- 会话数据持久化在文件系统中

```typescript
// OpenClaw 的会话写入锁，防止并发写冲突
import { acquireSessionWriteLock } from "../agents/session-write-lock.js";
```

**Know-how 6：Session Write Lock**

在个人助手场景中，可能存在多个入口同时操作同一会话（比如 webhook + cron + 用户消息）。OpenClaw 用文件系统锁保证会话写入的原子性——这不是并发编程教科书里的理论，是真金白银踩坑出来的。

### 5.2 Claude Code 的会话模型

Claude Code Agent SDK 的会话管理（来源：[Sessions 文档](https://code.claude.com/docs/en/agent-sdk/sessions.md)）：

```python
# Python: ClaudeSDKClient 自动跟踪会话
async with ClaudeSDKClient(options=options) as client:
    await client.query("第一个任务")
    async for message in client.receive_response(): ...
    await client.query("第二个任务")  # 自动继续同一会话
    async for message in client.receive_response(): ...
```

三种恢复方式：
- **Continue**：找当前目录最近的会话（无需跟踪 ID）
- **Resume**：指定 session ID（多用户/多会话场景）
- **Fork**：复制历史创建新会话

**Know-how 7：Session vs Checkpoint 的分离**

Claude Code 明确区分：
- **Session**：保存对话历史（prompt、tool calls、responses）
- **Checkpoint**：保存文件系统的快照

这是关键设计——对话历史和文件状态是正交的两个维度。回滚文件不等于回滚对话。

### 5.3 Hermes Agent 的会话模型

Hermes Agent 用 **SQLite + FTS5** 存储会话：
- 所有对话历史持久化到 SQLite 数据库
- 启用 FTS5 全文索引，支持自然语言搜索跨会话内容
- 搜索结果可以被 LLM 做摘要，实现跨会话的知识召回
- 路径管理通过 `HERMES_HOME` 常量统一（`hermes_constants.py`）

**Know-how 15：FTS5 全文检索作为 Agent 的"长期记忆索引"**

Hermes Agent 的 FTS5 搜索 + LLM 摘要，实际上构成了一个简单但有效的 RAG 系统——不依赖外部向量数据库，直接用 SQLite 的内置全文索引。这对轻量级部署非常友好（不需要额外的 Milvus/Pinecone 等）。
配合 MEMORY.md/USER.md 的冻结快照记忆，形成两层记忆体系：
- **短期/冻结记忆**：MEMORY.md/USER.md（系统提示注入）
- **长期/可检索记忆**：SQLite + FTS5（按需搜索）

---

## 六、Security & Sandbox：安全隔离

### 6.1 OpenClaw 的安全模型

```
默认：主会话工具在宿主机运行（完整访问权限）
非主会话：自动进入沙箱
沙箱后端：Docker（默认）/ SSH / OpenShell
```

OpenClaw 沙箱默认允许：
- `bash`, `process`, `read`, `write`, `edit`
- `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`

默认拒绝：
- `browser`, `canvas`, `nodes`, `cron`, `discord`, `gateway`

**Know-how 8：沙箱默认策略是"按需放权"而非"全开再收"**

这是一种最小权限原则的实践。默认给子会话最小权限集，而不是给所有权限再靠 deny 规则回收。

### 6.3 Hermes Agent 的安全模型

Hermes Agent 的安全控制是多层组合：
- **命令审批**：危险命令需要用户确认（`approval.py`）
- **DM 配对授权**：消息渠道的 DM 需要配对验证（`pairing.py`）
- **容器隔离**：Docker/SSH/Singularity 等后端提供执行隔离
- **环境变量隔离**：通过 `env_passthrough.py` 控制沙箱内可见的环境变量

### 6.2 Claude Code 的工具权限

Claude Code 用三层控制（来源：[Permissions 文档](https://code.claude.com/docs/en/agent-sdk/permissions.md)）：

```
allowed_tools    → 自动批准
disallowed_tools → 绝对禁止
permission_mode  → 其他工具的行为（默认询问）
```

支持细粒度规则：
```
"Bash(npm *)"    → 只允许 npm 命令
"Read(/src/**)"  → 只允许读 src 目录
```

---

## 七、Multi-Agent Routing：多 Agent 编排

### 7.1 OpenClaw 的多 Agent 架构

OpenClaw 通过 `acp`（Agent Communication Protocol）和 `acpx` 实现多 Agent 编排：

- `sessions_spawn`：创建隔离子 Agent 会话
- `sessions_send`：向已有会话发消息
- `sessions_yield`：让出控制权，等待子 Agent 结果
- `subagents(action=list|steer|kill)`：子 Agent 管理

支持两种运行时：
- `runtime="subagent"`：OpenClaw 内置的子 Agent
- `runtime="acp"`：外部 ACP 兼容的 Agent（如 Claude Code、Codex）

**Know-how 9：ACP 协议的桥接价值**

OpenClaw 本身不生产 Agent 能力，它通过 ACP 协议桥接多个编码 Agent（Claude Code、Codex 等），同时自己也提供 subagent 能力。这种"编排而非实现"的设计让 OpenClaw 成为**Agent 的 Agent**。

### 7.2 Hermes Agent 的子 Agent

Hermes Agent 通过 **Delegate 工具**实现子 Agent 调用：
- **隔离上下文**：子 Agent 有自己的上下文窗口，不污染主会话
- **Python RPC 脚本**：可以编写 Python 脚本直接调用子 Agent 的工具，实现"零上下文成本"的多步操作——即不需要把中间结果塞回主对话历史，直接在脚本层面传递数据。
这种设计适合"批处理"场景：比如同时处理 10 个文件的分析任务，可以并行分发给 10 个子 Agent，最后只汇总结果。

### 7.3 Claude Code 的子 Agent

Claude Code 通过 `Agent` 工具实现子 Agent 调用：
```python
# Agent 工具可以隔离上下文、并行执行、专业化分工
```

支持：
- **上下文隔离**：子 Agent 有自己的上下文窗口
- **并行执行**：多个子 Agent 可以同时运行
- **专业化**：给不同子 Agent 不同的 system prompt 和工具集

---

## 八、Observability：可观测性

### 8.1 Claude Code 的方案

- **OpenTelemetry 导出**：trace、metric、event（来源：[Observability 文档](https://code.claude.com/docs/en/agent-sdk/observability.md)）
- **Analytics Dashboard**：团队使用指标

### 8.2 OpenClaw 的方案

- **Diagnostic Events**：结构化诊断事件
- **Subsystem Logger**：子系统级日志
- **Session Transcript Events**：会话转录事件

### 8.3 Hermes Agent 的方案

- **`/usage`**：查看 token 用量统计
- **`/insights [--days N]`**：按天数窗口查看使用洞察
- **会话转录**：所有对话持久化到 SQLite，可随时回溯

---

## 九、Hermes Agent 的独特设计：闭环学习系统

Hermes Agent 最突出的特点是**闭环学习（Closed Learning Loop）**，这是 OpenClaw 和 Claude Code 都没有的。

### 9.1 学习循环的四个阶段

```
┌─────────────────────────────────────────────┐
│           Hermes 学习循环                    │
│                                             │
│  ① 自主技能创建                              │
│     完成复杂任务后，Agent 自动将经验          │
│     提炼为可复用的 Skill 文件                 │
│         │                                   │
│  ② 使用中自改进                              │
│     每次使用 Skill 时，Agent 可根据           │
│     实际效果自动优化 Skill 内容              │
│         │                                   │
│  ③ 记忆持久化提示                            │
│     Agent 会定期被提示将重要信息             │
│     写入 MEMORY.md（有字符限额）             │
│         │                                   │
│  ④ 会话搜索与总结                            │
│     FTS5 搜索过去的对话，LLM 做摘要          │
│     实现跨会话的知识召回                      │
└─────────────────────────────────────────────┘
```

### 9.2 Honcho 辩证用户建模

Hermes Agent 集成了 [Honcho](https://github.com/plastic-labs/honcho) 的用户建模系统。这不是简单的"记住用户偏好"，而是通过**辩证交互**——在对话中主动向用户提问来验证和更新对用户的理解——构建更深层的用户画像。

### 9.3 记忆系统的工程设计细节

| 设计点 | 实现 |
|--------|------|
| 存储格式 | MEMORY.md (2,200 chars) + USER.md (1,375 chars) |
| 注入方式 | 会话启动时冻结快照注入 system prompt |
| 更新方式 | 会话中修改立即持久化，但 system prompt 下次生效（保护 prefix cache） |
| 更新匹配 | substring 匹配，不需要完整原文 |
| 满载策略 | 自动合并/替换旧条目释放空间 |

这个设计的关键 insight 是：**内存限额不是限制，而是特性**。通过强制字符上限，Agent 必须学会"什么值得记，什么不值得记"——这恰恰是人类记忆的工作方式。

### 9.4 研究向能力：训练数据生成

Hermes Agent 不仅是一个使用工具，还能**产出训练数据**：
- **Batch 轨迹生成**：批量运行任务，保存完整的 tool-calling 轨迹
- **Atropos RL 环境**：强化学习训练环境（`environments/` 目录）
- **轨迹压缩**：将长轨迹压缩为训练友好的格式，用于训练下一代 tool-calling 模型

这是开源 Agent 中少见的"自举"能力——用 Agent 产生的数据训练更好的 Agent。

---

## 十、架构对比总结

下表用 🏛️ 标记 **Harness 核心功能**（三者都具备，是构建一个合格 Harness 的基石），用 ⭐ 标记 **差异化功能**（部分或单一实现，是产品的竞争力所在）。

| 维度 | OpenClaw | Claude Code | Hermes Agent | 类型 |
|------|---------|-------------|-------------|------|
| Agent Loop | 嵌入式 Pi Runner (dispatch) | Agent SDK query() + 5 种消息类型状态机 | AIAgent.run_conversation() (~10,700 行) | 🏛️ |
| 工具调度 | 动态发现（MCP + 内置） | 6 大类工具集 | 47 工具 + 19 toolsets + MCP | 🏛️ |
| 工具执行 | 宿主/沙箱分离 | 读并发/写串行 | Tool Registry + Toolsets 分组 | 🏛️ |
| 上下文管理 | Compaction + Session Reset | Compaction + Fork + Checkpoint | ContextEngine 可插拔 + Prompt Caching | 🏛️ |
| 会话存储 | 文件系统 + sessionKey + 写入锁 | session_id + 文件 checkpoint | SQLite + FTS5 + HERMES_HOME | 🏛️ |
| 安全控制 | 沙箱 + 最小权限默认 | 工具级 allow/deny + 路径规则 | 命令审批 + DM 配对 + 容器隔离 | 🏛️ |
| 多 Agent | ACP 桥接 + subagent | Agent 工具 + 上下文隔离 | Delegate 工具 + Python RPC 脚本 | 🏛️ |
| 消息渠道 | 20+（微信/Discord/Telegram 等） | Terminal/IDE/Desktop/Browser | 18 平台适配器（Telegram/Discord/微信 等） | 🏛️ |
| 技能系统 | Skill 文件（SKILL.md） | Skill 工具 | 自主创建 + 自改进 + agentskills.io | 🏛️ |
| 可观测性 | 诊断事件 + 子系统日志 | OpenTelemetry + Analytics | /usage + /insights + 会话转录 | 🏛️ |
| | | | | |
| 模型绑定 | 多模型（Gateway 配置） | **仅 Claude** | 多模型（200+ providers） | ⭐ |
| 记忆系统 | MEMORY.md + daily notes (文件) | **无内置** | MEMORY.md + USER.md + FTS5 搜索 | ⭐ |
| 终端后端 | Docker/SSH/OpenShell | **本地终端** | 6 种后端 + serverless 休眠 | ⭐ |
| 入站去重 | **channel+peer+messageId** | **无** | **无明确文档** | ⭐ |
| 定时任务 | 内置 cron + 渠道投递 | **无内置** | 内置 Cron Scheduler | ⭐ |
| IDE 集成 | ACP 协议 | **IDE 原生集成** | ACP Adapter | ⭐ |
| 研究能力 | **无** | **无** | Batch 轨迹生成 + Atropos RL | ⭐ |
| 迁移路径 | - | - | hermes claw migrate | ⭐ |
| 出品方 | 社区开源 | Anthropic | Nous Research | - |
| 定位 | 个人 AI 助手 Gateway | 编码 Agent | 自进化 AI Agent | - |
| 语言 | TypeScript/Node.js | Python SDK | Python | - |
| GitHub Stars | 社区项目 | 不开源 | ~80,000+ | - |

### Harness 核心功能 vs 差异化功能

从上表可以看出一个清晰的结论：**10 个维度是三者都具备的核心功能**，这意味着这些不是某个产品的"加分项"，而是构建一个合格 Harness 的**必备条件**：

1. **Agent Loop** —— ReAct 循环是所有 Harness 的心脏，没有循环就没有 Agent
2. **工具调度** —— 纯 LLM 不是 Agent，能调用工具才是
3. **工具执行** —— 读写分类、安全隔离，工具必须有正确的执行策略
4. **上下文管理** —— 会话越长上下文越大，不管理必崩窗口上限
5. **会话存储** —— 跨会话连续性是 Agent 区别于单次 API 调用的本质特征
6. **安全控制** —— 沙箱、权限、审批，没安全就不可生产化
7. **多 Agent** —— 任务分解、并行执行、上下文隔离，复杂任务必须拆分
8. **消息渠道** —— 人机交互的入口，无论终端/IM/IDE
9. **技能系统** —— 将经验沉淀为可复用能力，这是"变聪明"的工程基础
10. **可观测性** —— token 用量、会话转录、诊断日志，没有就无法调试和优化

而**差异化功能**才是三个产品拉开距离的地方：

- **模型绑定**：Claude Code 锁定自家模型（封闭），OpenClaw 和 Hermes Agent 多模型（开放）
- **记忆系统**：Claude Code 无内置记忆（需外部实现），另外两者都有
- **定时任务**：Claude Code 无 cron，OpenClaw 和 Hermes Agent 都有
- **研究能力**：只有 Hermes Agent 有 Batch 轨迹生成和 RL 训练环境（面向研究的独特定位）

**给 Harness 开发者的启示**：先补齐 10 个核心功能，再在差异化功能中找到自己的护城河。

### 核心设计哲学差异

---

## 十一、实战 Know-how 汇总

### 给 Harness 开发者的 13 条建议

1. **消息即状态**：用消息类型驱动循环，别搞隐式状态机
2. **去重是刚需**：任何多入口系统都必须有入站去重
3. **读写工具分类执行**：只读可并发，写操作必串行
4. **按需加载工具**：超过 50 个工具就用 ToolSearch
5. **Compaction 是长会话的命脉**：别等窗口满了才想起来
6. **Session 和文件状态分离**：回滚文件≠回滚对话
7. **会话写入必须加锁**：多入口并发是迟早的事
8. **最小权限默认**：按需放权，别反过来
9. **编排 > 实现**：能做桥接就别重复造 Agent
10. **预算上限是安全网**：生产环境必设 `max_turns` + `max_budget_usd`
11. **Provider 抽象层是关键**：用 3 种 API mode 适配所有 LLM，避免 per-provider 分支逻辑（Hermes Agent 的 `runtime_provider.py`）
12. **记忆限额是特性不是限制**：强制字符上限逼 Agent 学会"什么值得记"，这正是人类记忆的工作方式（Hermes Agent 的 MEMORY.md/USER.md）
13. **闭环学习让 Agent 越用越聪明**：自主技能创建 + 使用中自改进 + 跨会话搜索 + 用户建模，四个阶段缺一不可

---

## 参考资料

1. **ReAct 论文**: Yao, S. et al. "ReAct: Synergizing Reasoning and Acting in Language Models" (2022) — https://arxiv.org/abs/2210.03629
2. **Claude Code 官方文档**: https://code.claude.com/docs/
   - How the agent loop works: https://code.claude.com/docs/en/agent-sdk/agent-loop.md
   - Sessions: https://code.claude.com/docs/en/agent-sdk/sessions.md
   - Permissions: https://code.claude.com/docs/en/agent-sdk/permissions.md
3. **OpenClaw 源码**: https://github.com/openclaw/openclaw
   - Dispatch: `src/auto-reply/dispatch-dispatcher.ts`
   - Agent Harness: `src/agents/harness/`
   - Embedded Runner: `src/agents/pi-embedded-runner/`
4. **OpenClaw 官方文档**: https://docs.openclaw.ai/
   - Architecture: https://docs.openclaw.ai/concepts/architecture
   - Security: https://docs.openclaw.ai/gateway/security
   - Sandboxing: https://docs.openclaw.ai/gateway/sandboxing
5. **Hermes Agent 源码**: https://github.com/NousResearch/hermes-agent
   - Agent Loop: `run_agent.py` (AIAgent.run_conversation)
   - Architecture: https://hermes-agent.nousresearch.com/docs/developer-guide/architecture
   - Memory: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory
   - Tools: https://hermes-agent.nousresearch.com/docs/user-guide/features/tools
   - Skills: https://hermes-agent.nousresearch.com/docs/user-guide/features/skills
   - Security: https://hermes-agent.nousresearch.com/docs/user-guide/security
6. **MCP (Model Context Protocol)**: https://github.com/modelcontextprotocol/specification
7. **Tool Use 论文**: "Tool Learning with Foundation Models" (2023) — https://arxiv.org/abs/2304.08354
8. **Honcho 用户建模**: https://github.com/plastic-labs/honcho

---

*作者：伟哥 | IT 20年 | 大模型 Infra / Agent / 算法 / 多模态方向*
