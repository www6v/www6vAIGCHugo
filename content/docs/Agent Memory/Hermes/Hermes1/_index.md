---
title: Hermes Memory
weight: 10
---



# Hermes Agent Memory 设计与实现深度解析

> "An agent without memory is just a function — stateless, forgetful, and doomed to repeat itself."
>
> —— 改编自 Andrew Ng 对 AI Agent 的论述[^1]

---

假设你是一个软件工程师，每天早上打开终端，对着一个 AI 助手说："帮我看看昨天那个 bug 修好了没？"如果这个助手每次都像第一次见面一样问你"你指的是哪个项目？用的什么语言？昨天发生了什么？"——你会疯掉。

这就是没有记忆的 AI Agent 的困境。大语言模型本质上是**无状态的函数**：给定输入，产生输出，然后一切归零。上下文窗口再大（200K、1M token），也只覆盖当前会话。一旦你关掉终端、开启新 session，或者上下文压缩发生，之前的对话、偏好、环境知识就灰飞烟灭了。

Hermes Agent 的 Memory 系统要解决的，正是这个根本矛盾：**如何让一个无状态的模型，拥有跨会话的、持续增长的、个性化的记忆能力？**

本文将深入剖析 Hermes Agent 的记忆架构——从源码级别拆解三层记忆系统的设计决策、工程取舍和安全机制。文章基于 Hermes Agent 开源代码（NousResearch/hermes-agent，12,000+⭐）的源码分析[^2]。

---

## 三层记忆架构总览

在深入每个组件之前，先建立全局视角。Hermes 的记忆系统不是单一存储，而是**三层架构**，每一层解决不同粒度和不同延迟需求的记忆问题：

| 层次 | 实现 | 容量 | 延迟 | 典型场景 |
|------|------|------|------|----------|
| 内置记忆（Built-in） | `MEMORY.md` / `USER.md` 文件 | ~2,200 + 1,375 字符 | 即时（system prompt 注入） | 用户偏好、环境约定、项目经验 |
| 会话搜索（Session Search） | SQLite FTS5 全文索引 | 无上限 | ~20ms | 历史对话检索、跨 session 上下文回溯 |
| 外部插件（External Provider） | Honcho / Mem0 / Hindsight 等 8 个插件 | 后端决定 | 取决于后端（通常 50-200ms） | 知识图谱、语义搜索、用户建模 |

这三层不是冗余备份，而是**互补的分工**：

- **内置记忆**是"笔记本"——小容量、高优先级、每次对话都携带的精选事实
- **会话搜索**是"档案馆"——完整记录所有历史，按需检索
- **外部插件**是"专业数据库"——语义搜索、知识图谱、用户画像等高级能力

用一个类比：内置记忆是你随身携带的便签，会话搜索是你办公室的文件柜，外部插件是你连接的公司知识库系统。

---

## 一、MemoryStore：内置记忆的实现

### 1.1 双存储设计

Hermes 的记忆存储并非一个单一文件，而是**两个独立的 Markdown 文件**，分别位于 `~/.hermes/memories/` 目录下（使用 `get_hermes_home()` 获取路径，确保 profile 隔离）：

- **MEMORY.md**（默认 2,200 字符上限）：Agent 的"个人笔记"——环境事实（"huggingface.co 连接超时"）、项目约定（"测试用 pytest -n 4"）、经验教训（"不要用 raw.githubusercontent.com"）
- **USER.md**（默认 1,375 字符上限）：用户画像——偏好（"用户要求被称呼为'大风'"）、沟通风格（"回复要简洁"）、习惯（"所有文件保存到 `/home/ubuntu/wei/harnessWork/hermesWorks/`"）

这种分离的设计决策源于**关注点分离**原则：用户画像的变化频率远低于环境事实，将两者分开可以减少不必要的系统提示词更新，同时也便于用户手动编辑 USER.md 来调整 Agent 对自己的认知。

每个记忆条目用 `§`（section sign，Unicode U+00A7）分隔。选择这个分隔符而非常见的 `---` 或 `===` 是因为：`§` 在代码和日常写作中极少出现，极大降低了分隔符冲突的概率。

```
Server connectivity: huggingface.co unreachable (connection refused)...
§
用户要求称呼他为"大风"，AI自称"小伟"。
§
共创四本Hugo开源书：www6vAIGC, www6vAlgo, www6vMLSys, www6vVision
```

### 1.2 Frozen Snapshot 模式：工程上的关键取舍

这是 Hermes Memory 中最值得深入理解的设计决策。

**问题**：如果把记忆文件的内容直接拼接进每次 API 调用的 system prompt，那么每当 Agent 写入一条新记忆（比如 `memory` 工具调用返回后），system prompt 就会改变。这会**破坏 LLM 的 prefix cache**——因为 system prompt 是请求的前缀，任何改动都意味着缓存失效，需要从第一个 token 重新计算。

**解法**：Hermes 采用了 **Frozen Snapshot（冻结快照）** 模式[^3]：

1. Session 启动时，调用 `load_from_disk()` 读取 MEMORY.md 和 USER.md
2. 将内容格式化为系统提示词块，存入 `_system_prompt_snapshot` 字典
3. **此后的整个 session 中，snapshot 内容不再改变**——即使 Agent 通过 `memory` 工具写入了新记忆

```python
# 简化示意（来自 tools/memory_tool.py）
def format_for_system_prompt(self, target: str) -> str:
    # 返回的是冻结的快照，不是实时读取文件
    return self._system_prompt_snapshot.get(target, "")
```

这意味着：你在 session 中间写入的记忆，**不会立即出现在当前 session 的上下文感知中**，而是在下一个 session 启动时生效。

**权衡分析**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Frozen Snapshot（Hermes 选择） | Prefix cache 稳定、token 效率最大化、可预测 | 写入延迟一轮生效 |
| 实时注入（每次写入后更新 prompt） | 即时生效 | 每次写入破坏 cache、token 浪费、可能引发上下文震荡 |
| 工具调用读取（LLM 主动 read） | 按需加载、不占 context | 增加一次工具调用、LLM 可能忘记读取 |

Hermes 选择了效率优先的 Frozen Snapshot，因为它假设：**记忆写入的是长期知识，不需要在当前对话中立即消费**。如果需要即时反馈，Agent 可以通过 `memory` 工具的返回结果（live state）知道自己写入成功。

### 1.3 存储引擎：原子写入与并发安全

MemoryStore 的持久化层有三个关键设计：

**原子写入**：使用 `tempfile.mkstemp()` 创建临时文件 → 写入内容 → `atomic_replace()`（同文件系统内的 rename 操作）。这保证了写入过程中不会出现"半写"状态——要么旧内容完整，要么新内容完整，绝不会出现截断文件。

```python
# 伪代码
fd, tmp_path = tempfile.mkstemp(dir=memory_dir)
os.write(fd, new_content.encode())
os.close(fd)
os.replace(tmp_path, target_path)  # 原子 rename
```

**文件锁**：使用 `fcntl.LOCK_EX`（Unix）或 `msvcrt.locking`（Windows）进行排他锁。锁文件是独立的 `.lock` 文件，而非直接在数据文件上加锁——这避免了某些文件系统上锁与 rename 操作冲突的问题。

**外部漂移检测（External Drift Detection）**：这是 Hermes 的一个独特设计。如果用户手动编辑了 MEMORY.md 文件，或者通过 `patch` 工具修改了文件内容，MemoryStore 在下次写入前会检测"磁盘内容是否能通过自身的解析器正确 round-trip"。如果不能（比如用户用了不同的分隔符、或者某个条目超出了字符限制），则**拒绝写入**并将当前文件备份为 `.bak.<timestamp>`，防止数据丢失。

这个设计解决的问题是：当多个工具（`memory` 工具、`patch` 工具、手动编辑器）可能同时修改同一个文件时，如何保证不会出现"后写入者覆盖先写入者"的数据丢失。

### 1.4 容量管理：为什么是字符限制而非 Token 限制？

MEMORY.md 和 USER.md 的容量以**字符数**而非 token 数限制。这看似不精确（因为不同模型的 token 化方式不同），但有以下优势：

1. **模型无关**：无论你用 Claude（~3 char/token）、GPT-4（~4 char/token）还是本地模型（变化更大），字符数是一致的
2. **计算成本低**：统计字符数是 O(1)，统计 token 数需要调用 tokenizer（需要加载模型、增加启动延迟）
3. **足够精确**：2,200 字符在任何主流模型下大约对应 550-730 token，在 system prompt 中占比可控

去重机制采用 exact duplicate 拒绝——如果新内容与已有条目完全相同，则静默拒绝写入，不消耗容量。

### 1.5 安全扫描：记忆作为攻击面

记忆系统是 Agent 安全的一个重要攻击面。想象这个场景：某个网页内容包含隐藏的 prompt injection 指令，Agent 在浏览网页后试图将其保存为记忆。如果这个注入指令被写入 MEMORY.md 并在下次 session 中注入 system prompt，Agent 的行为就可能在不知不觉中被劫持。

Hermes 的防御层包括：

- **威胁模式扫描**：所有写入和快照加载都会经过 `tools/threat_patterns.py` 的 `"strict"` 级别扫描，检测 prompt injection、凭证外泄、SSH 后门等模式
- **Unicode 不可见字符过滤**：过滤零宽字符、双向文本攻击（BIDI）等不可见 Unicode 字符
- **隔离层**：被标记为受污染的条目在 live state（工具调用返回）中保持原文以便用户检查，但在 snapshot（注入 system prompt 的部分）中被替换为 `[BLOCKED: ...]`

这种"双层呈现"的设计——受污染内容对用户可见但对模型不可见——是一个务实的权衡：既不隐藏问题（用户可以 inspect），也不让问题影响模型行为。

---

## 二、MemoryManager 与插件架构

### 2.1 插件发现机制

Hermes 的插件系统允许开发者替换或增强内置记忆能力。插件发现遵循**双目录扫描**模式：

1. **内置插件**：`plugins/memory/<name>/`（随 Hermes 分发）
2. **用户安装插件**：`$HERMES_HOME/plugins/<name>/`

当名称冲突时，内置插件优先——这防止了用户安装的恶意插件覆盖核心功能。

加载机制采用 Python 的 `importlib.util.spec_from_file_location` 动态导入，为用户插件创建合成父包（`_hermes_user_memory`）以避免 `sys.modules` 命名空间冲突。每个插件通过两种方式之一注册：

- `register(ctx)` 函数模式：返回一个 MemoryProvider 实例
- `MemoryProvider` 子类反射：自动实例化

### 2.2 MemoryManager 协调器

`MemoryManager`（位于 `agent/memory_manager.py`）是整个记忆系统的集成点，承担四个核心职责：

**提供者注册**：内置 provider 始终第一位，**最多允许一个外部 provider**。这不是技术限制，而是设计决策——多个外部 provider 之间会产生语义冲突（哪个 provider 的事实更权威？）且增加延迟。

**工具路由**：通过 `_tool_to_provider` 字典将工具调用路由到正确的 provider。例如，`memory` 工具（add/replace/remove）路由到内置 provider，而 `mem0_search` 路由到 Mem0 provider。

**故障隔离**：每个 provider 的操作被包裹在 try/except 中。一个 provider 的失败（比如 Mem0 API 超时）不会阻塞其他 provider（内置记忆仍然正常工作）。

**生命周期编排**：

```
Turn 开始:  on_turn_start(turn, msg)
预取:       prefetch_all(query) → 合并为 <memory-context> 块注入
API 调用:   LLM 收到冻结的 memory snapshot
Turn 结束:  sync_all(user, assistant) + queue_prefetch_all(query)
Session 结束: on_session_end(messages) + shutdown_all()
```

注意 `queue_prefetch_all` 的设计——在每轮对话结束后，**异步预热**下一轮的检索查询。这利用了对话的连续性：下一轮的话题很可能与当前轮相关，提前检索可以隐藏延迟。

### 2.3 MemoryProvider 抽象基类

MemoryProvider ABC（`agent/memory_provider.py`，296 行）定义了插件必须实现的接口和可选的生命周期钩子。

**必须实现的 5 个方法**：

| 方法 | 用途 | 性能要求 |
|------|------|---------|
| `name` (property) | Provider 标识符 | - |
| `is_available()` | 检查配置/依赖是否就绪 | 无网络调用 |
| `initialize(session_id, **kwargs)` | 连接后端、创建资源、预热 | 可阻塞 |
| `get_tool_schemas()` | 返回 OpenAI function-calling 格式的工具定义 | - |
| `handle_tool_call(tool_name, args)` | 处理工具调用并返回结果 | - |

**可选的 8 个生命周期钩子**：

| 钩子 | 触发时机 | 典型用途 |
|------|---------|---------|
| `system_prompt_block()` | 提示词组装 | 注入 provider 的静态信息 |
| `prefetch(query)` | 每轮 API 调用前 | 根据当前对话召回相关上下文 |
| `queue_prefetch(query)` | 每轮对话后 | 异步预热下一轮检索 |
| `sync_turn(user, asst)` | 每轮对话后 | 异步持久化对话内容 |
| `on_turn_start()` | 每轮开始 | 轮次计数、周期维护 |
| `on_session_end()` | session 退出 | 端到端事实提取 |
| `on_pre_compress()` | 上下文压缩前 | 从即将丢弃的消息中提取洞见 |
| `on_memory_write()` | 内置记忆写入时 | 镜像同步到外部后端 |

这种丰富的生命周期钩子设计让外部 provider 可以在记忆系统的每个关键节点上插入自定义逻辑——比如 Honcho 在 `on_session_end` 时做完整的用户建模提取，Mem0 在 `sync_turn` 时做异步的事实提取和语义索引。

### 2.4 StreamingContextScrubber：流式输出中的记忆保护

Hermes 支持流式输出（streaming response）。在流式模式下，`<memory-context>` 标签可能被 stream delta 截断——比如一个 chunk 包含 `<memory-context`，下一个 chunk 才包含 `>...`。

StreamingContextScrubber 是一个**状态机**，跨 chunk 追踪记忆上下文标签的开启和关闭：

- 当检测到 `<memory-context` 开头的 chunk 时，进入"抑制模式"
- 在抑制模式中的所有 chunk 内容被丢弃（不传递给 UI）
- 当检测到闭合标签 `</memory-context>` 时，退出抑制模式

这防止了记忆上下文（包含用户偏好、项目秘密等敏感信息）泄露到用户可见的终端或消息界面中。

---

## 三、Session Search：会话级全文检索

如果内置记忆是"精选便签"，那么 Session Search 就是"完整档案馆"。

Hermes 将所有历史对话存储在 SQLite 数据库中（`~/.hermes/sessions/`），并通过 FTS5（Full-Text Search version 5）建立全文索引[^4]。FTS5 是 SQLite 的原生全文搜索扩展，支持：

- 词干分析（stemming）：搜索 "run" 会匹配 "running"、"ran"
- 前缀匹配：搜索 "deploy*" 匹配 "deployment"、"deployed"
- 布尔查询：`python NOT java`、`docker AND kubernetes`
- 短语搜索：`"docker networking"`
- OR/AND 默认策略：`elevenlabs OR baseten`（宽泛召回），默认 AND（精确匹配）

搜索流程分为三阶段：

1. **Discovery**：根据关键词匹配 session，返回标题、时间戳、摘要
2. **Navigation**：向前/向后滚动查看更多匹配
3. **Targeting**：精确定位到具体的对话轮次

Session Search 与内置记忆是**互补关系**而非竞争关系：

| 维度 | 内置记忆 | Session Search |
|------|---------|---------------|
| 内容来源 | Agent 主动筛选写入 | 所有对话自动记录 |
| 检索方式 | 自动注入 system prompt | 按需查询 |
| 容量 | 有限（~3,575 字符） | 无上限 |
| 延迟 | 0ms（已在 prompt 中） | ~20ms |
| 适用场景 | 稳定、重要的事实 | 历史回顾、上下文找回 |

这种双轨制设计借鉴了计算机体系结构中的 **Cache-Memory-Disk 三级存储** 思想：最快最小的存储放在最靠近 CPU 的位置（system prompt），最大的存储放在远端（SQLite），按需调取。

---

## 四、记忆在 Agent 生命周期中的完整流转

理解记忆系统最好的方式是追踪一次完整的 Agent 对话周期。以下是基于源码（`agent/agent_init.py`、`agent/conversation_loop.py`）的生命周期分析：

### 启动链

```
Agent 启动
  ├─ 读取 config.yaml → memory.memory_enabled, memory.user_profile_enabled
  ├─ MemoryStore(memory_char_limit, user_char_limit) 初始化
  ├─ load_from_disk() → 读取 MEMORY.md / USER.md
  │   ├─ 解析 § 分隔条目
  │   ├─ threat scan → 污染标记
  │   └─ 构建 _system_prompt_snapshot
  ├─ 加载外部插件（如配置了 memory.provider）
  │   ├─ 扫描 plugins/memory/ 目录
  │   ├─ importlib 动态加载
  │   └─ register(ctx) → MemoryProvider 实例
  ├─ MemoryManager.add_provider(builtin + external)
  ├─ initialize_all(session_id, hermes_home, ...)
  └─ 将 frozen snapshot 注入 system prompt（仅一次）
```

### 每轮对话（Turn）

```
Turn N 开始
  ├─ on_turn_start(turn, msg) → 通知所有 provider
  ├─ prefetch_all(query)
  │   ├─ 内置 provider：无操作（snapshot 已冻结）
  │   └─ 外部 provider：执行语义搜索/知识图谱查询
  ├─ 构建 <memory-context> 块 → 拼接到 user message 前
  ├─ LLM API 调用（system prompt 含 frozen memory snapshot）
  │   ├─ 如果有 tool_calls → dispatch
  │   │   └─ memory 工具 → MemoryStore.add/replace/remove
  │   │       ├─ 写入磁盘（atomic_replace + file lock）
  │   │       ├─ 更新 live state
  │   │       ├─ on_memory_write → 通知外部 provider 镜像
  │   │       └─ 注意：snapshot 不变！
  │   └─ 如果有 text response → 返回给用户
  └─ sync_all(user_msg, assistant_msg) + queue_prefetch_all(next_query)
      ├─ 外部 provider 异步持久化对话内容
      └─ 异步预热下一轮检索
```

### Session 结束

```
Session 结束
  ├─ on_session_end(messages)
  │   └─ 端到端扫描完整对话 → 提取新事实
  ├─ shutdown_all()
  │   └─ 关闭连接、释放资源
  └─ (可选) nudge → 提醒 Agent 审查并写入记忆
```

注意 `nudge_interval`（默认 10 轮）的设计：每 N 轮对话后，Hermes 会在响应末尾添加一个隐式的"记忆审查"提示，鼓励 Agent 评估当前对话是否有值得持久化的信息。这不是强制的——Agent 可以选择不写入——但提供了一个自动化的记忆巩固机制。

---

## 五、外部插件生态对比

Hermes 的记忆插件生态已经发展出 8 个主要实现，各自侧重不同的记忆范式：

| 插件 | 存储后端 | 核心能力 | 依赖 | 成本 |
|------|---------|---------|------|------|
| **Honcho**[^5] | Cloud / 自建 | 跨 session 用户建模、辩证推理、peer card | `honcho-ai` | 按定价 |
| **OpenViking**[^6] | 自建 | 分层检索 L0→L2、6 类自动提取、文件系统浏览 | `openviking` | 免费 |
| **Mem0**[^7] (60,336⭐) | Cloud | LLM 事实提取、语义搜索 + reranking | `mem0ai` | 按定价 |
| **Hindsight**[^8] | Cloud / 本地 | 知识图谱、实体解析、多策略检索、`reflect` 合成 | `hindsight-client` | 免费/按定价 |
| **Holographic**[^9] | 本地 SQLite | FTS5、信任评分、HRR 代数查询 | 无 | 免费 |
| **RetainDB**[^10] | Cloud | 混合搜索（Vector+BM25）、7 种记忆类型、增量压缩 | API key | $20/月 |
| **ByteRover**[^11] | 本地 / Cloud | CLI 知识树、分层检索、SOC2 云同步 | `byterover-cli` | 免费/按定价 |
| **Supermemory**[^12] | Cloud | 语义记忆、用户画像、图 API、session 级 ingest | `supermemory` | 按定价 |

### 开源 vs 商业方案的技术对比

| 维度 | 开源方案（Holographic, OpenViking） | 商业方案（Mem0, Honcho, RetainDB） |
|------|-----------------------------------|-----------------------------------|
| **部署** | 本地运行，无网络依赖 | 需要 API 连接 |
| **数据隐私** | 数据完全留在本地 | 数据发送到第三方 |
| **检索能力** | 基于关键词/FTS（Holographic）或分层规则（OpenViking） | 语义向量搜索 + LLM 提取 |
| **维护成本** | 零运营成本 | API 费用 + 延迟依赖 |
| **可扩展性** | 受限于本地资源 | 云原生弹性扩展 |
| **集成深度** | 通过 MemoryProvider ABC 完全集成 | 同上，但依赖外部 API 可用性 |

**选择建议**：

- 如果你的 Agent 处理敏感数据（代码库、内部文档），选择 **Holographic**（本地 SQLite，零依赖）
- 如果你需要最强的语义理解能力（"上次讨论的那个数据库设计问题"），选择 **Mem0** 或 **Hindsight**
- 如果你追求零成本且不需要语义搜索，**Holographic** 或 **OpenViking** 是最佳起点

---

## 六、Profile 隔离与多 Agent 场景

### 6.1 Profile 级路径隔离

Hermes 支持多 Profile（独立的配置、技能、会话和记忆）。记忆系统通过 `get_hermes_home()` 函数确保路径隔离：

```python
# 不同 profile 的记忆文件位于不同路径
~/.hermes/memories/              # default profile
~/.hermes/profiles/research/memories/  # research profile
```

这意味着切换 profile 时，Agent 的"记忆人格"也随之切换——research profile 可能记得你在做 ML 研究，而 default profile 记得你在做 Web 开发。

### 6.2 子 Agent 的 Memory 隔离

当使用 `delegate_task` 派生子 Agent 时，Hermes 默认使用 `skip_memory=True`——子 Agent **不加载**父 Agent 的记忆文件。这是有意为之的设计：

1. **防止记忆污染**：子 Agent 的任务是执行特定的子任务，不需要携带父 Agent 的完整记忆上下文
2. **节省 token**：记忆快照占用 system prompt 空间，子 Agent 的 token 预算更紧张
3. **安全隔离**：如果子 Agent 使用不同的模型或 provider，记忆中的敏感信息不应泄露

子 Agent 通过 `parent_session_id` 参数与父 Agent 关联，如果需要访问历史上下文，可以通过 `session_search` 工具按需检索。

---

## 七、配置与调优

Memory 系统的行为通过 `config.yaml` 中的 `memory` 节控制：

```yaml
memory:
  memory_enabled: true              # 启用 MEMORY.md
  user_profile_enabled: true        # 启用 USER.md
  memory_char_limit: 2200           # MEMORY.md 字符上限
  user_char_limit: 1375             # USER.md 字符上限
  provider: null                    # 外部插件名（如 "mem0", "honcho"）
  nudge_interval: 10                # 每 N 轮触发记忆审查提示
```

通过 CLI 调整：

```bash
# 增加记忆容量（适合长会话场景）
hermes config set memory.memory_char_limit 4000

# 启用外部插件
hermes config set memory.provider mem0
hermes memory setup                   # 交互式配置向导

# 关闭记忆（适合一次性查询场景）
hermes config set memory.memory_enabled false
```

---

## 八、总结

Hermes Agent 的 Memory 系统展现了一个工程上务实且安全优先的记忆架构设计。回顾核心要点：

### 核心设计原则

1. **三层分工**：内置记忆（精选事实，即时可用）+ 会话搜索（完整档案，按需检索）+ 外部插件（专业数据库，高级能力），借鉴了计算机体系结构中的 Cache-Memory-Disk 思想
2. **Frozen Snapshot**：session 启动时冻结记忆快照，牺牲"即时生效"换取 prefix cache 稳定性和 token 效率——典型的工程权衡
3. **插件 ABC 设计**：通过 MemoryProvider 抽象基类和 8 个生命周期钩子，支持无缝切换记忆后端而不改变 Agent 核心逻辑
4. **安全优先**：威胁模式扫描、Unicode 过滤、双层呈现（live state vs snapshot）三重防护
5. **Profile 隔离**：通过 `get_hermes_home()` 实现多 profile 的记忆隔离，子 Agent 默认 `skip_memory=True` 防止污染

### 架构取舍

| 决策 | 选择 | 代价 |
|------|------|------|
| 记忆注入时机 | 启动时冻结 | 写入延迟一轮生效 |
| 容量单位 | 字符数 | 不同模型 token 换算不精确 |
| 去重策略 | Exact match | 语义重复无法检测 |
| 外部 provider 数量 | 最多 1 个 | 无法多后端协同检索 |
| 子 Agent 记忆 | 默认不加载 | 需要显式传递上下文 |

### 值得关注的方向

- **自动压缩策略**：当前容量超限即拒绝写入，未来可实现自动合并/淘汰策略（类似 LRU + 语义重要性评分）
- **语义去重**：引入轻量 embedding 模型，检测语义重复而非 exact match
- **多后端协同**：允许同时使用多个外部 provider，按检索场景路由
- **记忆质量评估**：定期审查记忆条目，淘汰过时事实

---

## 参考文献

[^1]: Andrew Ng, "AI Agent 设计模式", DeepLearning.AI, https://www.deeplearning.ai/short-courses/ai-agents-in-langgraph/
[^2]: Nous Research, "hermes-agent" (12,000+⭐), https://github.com/NousResearch/hermes-agent
[^3]: Hermes Agent 源码 `tools/memory_tool.py` 中 `_system_prompt_snapshot` 实现
[^4]: SQLite FTS5 文档, https://www.sqlite.org/fts5.html
[^5]: Honcho, https://github.com/inferableinc/honcho
[^6]: OpenViking, https://github.com/OpenViking/openviking
[^7]: Mem0 (60,336⭐), https://github.com/mem0ai/mem0
[^8]: Hindsight, https://github.com/hindsight-ai/hindsight
[^9]: Holographic, https://github.com/holographic-ai/holographic-memory
[^10]: RetainDB, https://github.com/retaindb/retaindb
[^11]: ByteRover, https://github.com/byterover-inc/byterover
[^12]: Supermemory, https://github.com/supermemoryai/supermemory

---

*本文基于 Hermes Agent 开源代码（NousResearch/hermes-agent）的源码分析和官方文档撰写，所有技术细节均可在源码中验证。*
