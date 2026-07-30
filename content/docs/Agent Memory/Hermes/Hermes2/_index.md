---
title: Hermes Memory
weight: 20
---



# Hermes Agent Memory：设计与实现

> **摘要**：Hermes Agent 是 NousResearch 开源的 AI Agent 框架（GitHub ⭐210K+），其核心理念是 "The agent that grows with you"。Memory 作为其核心子系统，让 Agent 能够跨会话保持记忆，并随着使用不断进化。本文将从架构设计、插件化体系、内置 Memory 实现、第三方生态、社区增强方案等维度，系统梳理 Hermes Agent Memory 的设计与实现。

---

## 一、概述

### 1.1 什么是 Hermes Agent

Hermes Agent 是由 NousResearch 开源的 AI Agent 框架，在 GitHub 上已获得 210K+ 星标。它的核心理念可以用一句话概括：

> **The agent that grows with you.**

与传统的 ChatGPT 式对话系统不同，Hermes Agent 不是一个无状态的问答机器——它是一个能记住你、了解你、随着交互不断进化的智能体。而实现这一愿景的核心，正是 **Memory 子系统**。

### 1.2 Memory 在 Hermes 中的位置

在 Hermes Agent 中，Memory 不是一个独立的微服务，也不是硬编码在后端逻辑里的一个模块。它采用的是一种 **可插拔的 Provider 体系**：

- 通过 `MemoryManager` 进行统一编排和生命周期管理
- 通过 `MemoryProvider` 抽象接口定义标准化的行为契约
- 内置 Provider（文件系统）+ 第三方插件 Provider（Hindsight、Honcho、Mem0、SuperMemory 等）

这种设计的核心思想是：**Memory 应该是 Agent 的一等公民，但实现方式可以多样化**。

### 1.3 设计目标

Hermes Memory 的设计围绕以下几个核心目标展开：

1. **跨会话持久记忆**：Agent 关闭再打开，依然"记得"之前的交互
2. **插件化架构**：一个外部 Provider 原则，通过标准化接口接入任意后端
3. **本地优先**：默认使用 SQLite/文件存储，支持云端扩展
4. **工具 schema 安全注入**：Memory 工具不会污染或破坏 LLM API 请求

---

## 二、核心架构：MemoryManager + MemoryProvider

### 2.1 MemoryManager（编排中枢）

`MemoryManager` 是整个 Memory 子系统的中枢神经系统。在 `run_agent.py` 中，它被实例化为单一组件，替换了原先散落在各处的后端代码。

#### 生命周期管理

MemoryManager 通过四个核心方法管理所有 Provider 的生命周期：

```python
# 启动时：初始化所有 Provider
memory_manager.initialize_all(session_id, hermes_home, platform, ...)

# 每轮 API 调用前：预取记忆
memory_manager.prefetch_all(query)

# 每轮结束后：同步写入
memory_manager.sync_turn(user_msg, assistant_msg)

# 关闭时：优雅退出
memory_manager.shutdown_all()
```

#### 单 Provider 约束

Hermes Agent 有一个重要的设计约束：**只允许激活一个外部 Provider**。这不是技术限制，而是架构选择——防止工具 schema 膨胀和多个 Provider 之间的冲突。如果同时激活 Hindsight 和 Mem0，工具列表会急剧膨胀，超出模型的上下文预算。

#### 背景执行

所有的 prefetch 和 sync 操作都通过 `ThreadPoolExecutor` 异步执行，带有超时排水机制。这意味着 Memory 操作不会阻塞 Agent 的主循环，保证了交互的流畅性。

### 2.2 MemoryProvider 抽象接口

`MemoryProvider` 是一个 Python 抽象基类（ABC），定义了 Memory 插件必须实现的行为契约：

#### 核心生命周期方法

| 方法 | 作用 | 是否有网络调用 |
|------|------|----------------|
| `is_available()` | 配置就绪检查 | ❌ 无 |
| `initialize(session_id, **kwargs)` | 会话初始化（创建资源/连接） | 可能有 |
| `system_prompt_block()` | 注入 system prompt 的静态文本 | ❌ 无 |
| `prefetch(query)` | 快速召回，返回待注入的上下文 | 可能有 |
| `sync_turn(user_msg, assistant_msg)` | 异步写入当前轮次的交互 | 可能有 |
| `get_tool_schemas()` | 暴露给模型的工具 schema | ❌ 无 |
| `handle_tool_call()` | 工具调用分发处理 | 可能有 |
| `shutdown()` | 清理退出 | ❌ 无 |

#### 可选钩子方法

除了核心方法，Provider 还可以实现一系列可选钩子：

- `on_turn_start()` — 每轮开始前的 tick
- `on_session_end()` — 会话结束时提取摘要
- `on_memory_write()` — 镜像内置 memory 工具的写入事件
- `on_delegation()` — 子任务完成时的观察钩子
- `backup_paths()` — 注册备份路径

这些钩子让 Provider 能够在 Agent 生命周期的关键时刻介入，实现更丰富的行为。

### 2.3 Schema 标准化机制

在 Hermes Agent 中，工具 schema 的格式一致性至关重要。不同 LLM Provider（如 OpenAI、DeepSeek、Anthropic）对 tool 格式的要求各不相同，而 Memory Provider 返回的 schema 可能存在嵌套格式不一致的问题。

`normalize_tool_schema()` 函数负责处理嵌套的 OpenAI tool 格式，确保 schema 符合目标 LLM 的要求。如果不做标准化，格式错误的 schema 会导致整个 API 请求被拒绝——特别是像 DeepSeek 这样严格的 Provider。

`inject_memory_provider_tools()` 则将处理后的工具 schema 安全地注入到 Agent 的工具面中，不会破坏已有工具的结构。

---

## 三、内置 Memory 系统（Built-in Provider）

### 3.1 双文件持久化设计

Hermes Agent 的内置 Memory 使用两个 Markdown 文件实现持久化：

#### MEMORY.md — Agent 的个人笔记

这是 Agent 的"私人笔记本"，记录：

- 环境事实（工具特性、API 行为、系统约束）
- 项目约定（命名规范、目录结构、工作流程）
- 学到的经验（踩过的坑、最佳实践）

#### USER.md — Agent 对用户的了解

这是 Agent 的"用户画像"，记录：

- 用户的偏好（语言风格、技术深度、反馈方式）
- 沟通风格（直接还是委婉、详细还是简洁）
- 工作习惯（时间偏好、工具偏好）
- 期望和边界

### 3.2 Frozen Snapshot 模式

这是一个关键的性能优化设计：

```
会话启动时：
  MEMORY.md → 读取 → 冻结快照 → 注入 system prompt
  USER.md   → 读取 → 冻结快照 → 注入 system prompt

会话期间：
  工具写入 → 更新磁盘文件 ✅
  system prompt 保持不变 ✅（保证前缀缓存命中）

下次会话启动时：
  重新读取磁盘 → 刷新快照
```

**为什么这样做？** 因为 system prompt 的前缀缓存在 LLM API 调用中可以显著减少延迟和成本。如果 system prompt 每轮都变，缓存就无法命中。Frozen Snapshot 模式让 system prompt 在会话期间保持稳定，同时允许磁盘文件实时更新。

### 3.3 Memory Tool（`memory` 工具）

Hermes Agent 提供了一个统一的 `memory` 工具，通过 `action` 参数区分操作：

| Action | 作用 |
|--------|------|
| `add` | 添加新条目 |
| `replace` | 替换已有条目（通过短唯一子串匹配定位） |
| `remove` | 删除条目 |

#### § 分隔符

条目之间使用 `§`（section sign）分隔。这个选择很巧妙——它是一个 Unicode 字符，在正常文本中极少出现，因此不会与条目内容冲突，同时支持多行条目。

#### 字符级限制

Memory 文件使用字符级（而非 token 级）的容量控制。这是模型无关的设计——不同模型的 token 计数方式不同，但字符数是固定的。当文件接近容量上限时，Agent 会自动触发压缩或清理。

#### 原子写入 + 文件锁

写入操作使用原子写入（写入临时文件 + rename）和文件锁（fcntl / msvcrt），确保并发安全。即使在多线程或多进程场景下，也不会出现数据损坏。

### 3.4 安全机制

#### 内容扫描

写入时，`threat_patterns.py` 会检测潜在的注入和信息泄露模式。例如，如果写入内容包含类似系统指令的文本（"忽略之前的指令"、"你现在是一个不同的角色"等），会被标记并拒绝。

#### 偏移检测（Drift Detection）

这是一个精巧的防御机制：如果磁盘上的文件内容与内存中的解析器状态不一致，写入会被拒绝。这防止了外部篡改导致的不一致状态。

#### 严格模式

当 Memory 条目会被注入到 system prompt 中时，使用最严格的安全扫描级别。这是因为 system prompt 中的内容会直接影响 LLM 的行为。

### 3.5 Memory Write 镜像机制

`notify_memory_tool_write()` 实现了一个重要的桥接功能：内置 Memory 工具的写入事件会自动镜像给外部 Provider。

```python
# 内置工具写入 → 自动镜像给外部 Provider
memory_manager.notify_memory_tool_write(
    action="add",
    content="用户偏好使用中文交流",
    metadata=build_metadata(source="memory_tool")
)
```

镜像的约束：
- 只镜像**成功的**写入（失败的不会传播）
- 只镜像**非暂存的**写入（add / replace / remove，而非草稿操作）
- 支持批量操作（`operations` 列表）
- 元数据传递：`build_metadata` 回调注入来源信息

这让外部 Provider 能够在不直接拦截工具调用的情况下，感知到 Memory 的变化。

---

## 四、插件化 Memory Provider 生态

### 4.1 插件注册机制

第三方 Memory Provider 通过标准化的插件机制接入：

```
hermes/
  plugins/
    memory/
      hindsight/
        plugin.yaml      # 插件配置
        __init__.py      # 入口
      honcho/
        plugin.yaml
        __init__.py
      mem0/
        plugin.yaml
        __init__.py
```

`plugin.yaml` 定义插件的元数据和配置 schema。用户通过 `memory.provider` 配置键激活特定的 Provider。

### 4.2 Hindsight（Knowledge Graph + Entity Resolution）

Hindsight 是 Hermes Agent 生态中最成熟的第三方 Memory Provider 之一：

- **知识图谱**：将记忆组织为实体和关系的图谱，支持语义推理
- **实体解析**：自动识别和合并同一实体的多个提及
- **多策略检索**：向量搜索 + 关键词搜索 + 图谱遍历

#### 部署模式

| 模式 | 说明 |
|------|------|
| 云端模式 | 需要 `HINDSIGHT_API_KEY` 和 `HINDSIGHT_BANK_ID`，使用 Hindsight 云服务 |
| 本地模式 | 嵌入式守护进程，支持空闲超时和健康检查 |

#### Retain 机制

Hindsight 支持 Retain 机制——保留观察到的会话转录，用于后续的分析和记忆提取。这对于需要完整上下文理解的场景非常重要。

### 4.3 其他 Provider

| Provider | 特点 |
|----------|------|
| **Honcho** | 结构化 Memory，支持 peer-to-peer 通信 |
| **Mem0** | 会话 Memory + 事实提取，LLM 驱动 |
| **SuperMemory** | SaaS Memory 服务，云端托管 |
| **Byterover** | 字节级 Memory，细粒度控制 |
| **OpenViking** | Viking 生态集成，支持多模态 |
| **RetainDB** | 保留数据库，专注于记忆持久化 |

### 4.4 Mnemosyne（第三方，Hermes-first）

Mnemosyne 是一个值得单独关注的第三方 Memory 系统：

- **零依赖、亚毫秒级**：使用 `sqlite-vec` + FTS5 混合搜索
- **BEAM 架构**：
  - Working Memory：短期工作记忆
  - Episodic Memory：情景记忆
  - TripleStore：时序知识图谱
- **二进制向量（MIB）**：384-dim float32 → 48 字节，32 倍压缩
- **混合评分**：50% 向量相似度 + 30% FTS5 文本匹配 + 20% 重要性权重
- **MCP 内置服务器 + Python SDK**：标准化集成

---

## 五、Memory OS（社区增强方案）

Memory OS 是社区在 Hermes Agent 基础上构建的增强方案，提出了 **7 层 Memory 架构**：

### 5.1 7 层 Memory 架构

| 层级 | 名称 | 实现 | 说明 |
|------|------|------|------|
| L1 | Workspace | MEMORY.md / USER.md / CREATIVE.md | 每次注入 system prompt |
| L2 | Sessions | SQLite + FTS5 | 全量会话历史全文搜索 |
| L3 | Structured Facts | SQLite + HRR + FTS5 + Trust Scoring | 持久事实 + 实体解析 + 信任评分 |
| L4 | Fabric（跨会话） | Icarus Plugin（重度 fork） | LLM 驱动的会话提取 + 多源注入 |
| L5 | Vector DB | Qdrant（4096d Cosine + BM25 sparse） | 4 级回退 + 衰减扫描 + 语义去重 |
| L6 | LLM Wiki | 自动策展 vault | 持续摄入 Qdrant |
| L7 | Ground Truth | SOUL.md / rulebook.md | 确保注入的记忆被 Agent 真正使用 |

### 5.2 关键创新：Ground Truth 层级

L7 Ground Truth 解决了一个经常被忽视的问题：**Memory-Zero Behavior**——记忆被注入 system prompt，但 Agent 并不使用它。

Ground Truth 通过显式指令（写入 SOUL.md / rulebook.md），要求 Agent 将注入的记忆视为权威来源。这意味着 Agent 不会重复调用 API 去"验证"已经注入的内容，而是直接信任并使用这些记忆。

### 5.3 精确注入策略

Memory OS 的注入策略非常精细：

```
pre_llm_call 阶段：
  1. 从四源召回（Fabric + Qdrant + Sessions + Facts）
  2. 相关性门控（低于阈值的条目丢弃）
  3. 每会话去重（避免重复注入）
  4. 社交亲近过滤（优先注入与当前用户相关的记忆）
```

目标是：**零填充、零 hose**——LLM 只获得它需要的内容，不多不少。

---

## 六、Workspace 集成

### 6.1 Hermes Workspace 的 Memory 模块

Hermes Workspace 提供了一个可视化的 Memory 管理界面：

- 浏览、搜索、编辑 Agent 记忆
- Markdown 实时编辑器
- 与 2000+ Skills 目录集成
- MCP 全功能页面（catalog + marketplace + sources）

这让用户（和 Agent）能够直观地管理 Memory 内容，而不需要直接编辑文件。

### 6.2 Memory 文件注入时机

| 时机 | 操作 |
|------|------|
| 系统启动时 | MEMORY.md、USER.md 冻结快照加载 |
| 会话期间 | 工具写入 → 磁盘更新 → 下轮生效 |
| 会话结束时 | 提取摘要 → 更新长期记忆 |

---

## 七、Context Engine 与 Memory 的协同

### 7.1 Context Engine 架构

Context Engine 负责管理单会话内的上下文预算。当对话接近模型的 token 限制时，它会触发压缩：

```yaml
# config.yaml
context:
  engine: "compressor"  # 或 "lcm" 等第三方引擎
```

可插拔设计：支持内置的 `ContextCompressor` 或第三方压缩引擎（如 LCM）。

### 7.2 压缩策略

| 方法 | 作用 |
|------|------|
| `should_compress()` | 检查是否触发压缩（token 预算阈值） |
| `compress()` | 执行压缩（摘要、DAG 构建等） |
| `protect_first_n` | 保留前 N 条消息（通常是 system prompt） |
| `protect_last_n` | 保留最后 N 条消息（最近的上下文） |

### 7.3 Memory 与 Context 的关系

```
Memory：跨会话的持久记忆
Context Engine：单会话内的上下文预算

协同方式：
  Memory 的记忆通过 system prompt 注入（受 protect_first_n 保护）
  Context Engine 管理对话历史的压缩（不触碰 Memory 注入的部分）
```

这两者是正交的——一个管"跨会话"，一个管"会话内"。它们的协同确保了 Agent 既有长期记忆，又不会在单会话中超出 token 限制。

---

## 八、数据流与生命周期

### 8.1 启动流程

```
Agent Init
  ↓
MemoryManager 实例化
  ↓
注册 Provider（Built-in + 最多 1 个 External）
  ↓
initialize_all(session_id, hermes_home, platform, ...)
  ↓
system_prompt_block() 拼接各 Provider 的静态文本
  ↓
MEMORY.md / USER.md 冻结快照加载
  ↓
Agent 就绪，等待用户输入
```

### 8.2 单轮流程

```
用户输入消息
  ↓
prefetch_all() 背景召回（异步）
  ↓
system prompt 拼接（含 Memory 上下文）
  ↓
LLM API 调用
  ↓
sync_turn(user_msg, assistant_msg) 异步写入
  ↓
queue_prefetch_all() 预取下一轮
  ↓
等待下一轮用户输入
```

注意：prefetch 和 sync 都是异步的，不会阻塞主循环。

### 8.3 关闭流程

```
Agent Shutdown 信号
  ↓
shutdown_all()
  ↓
drain sync executor（有界超时，等待未完成写入）
  ↓
reverse order 关闭 Provider（先关闭外部，再关闭内置）
  ↓
清理完成
```

---

## 九、关键设计模式总结

### 9.1 插件化 Provider 模式

通过 ABC 抽象 + 注册机制，Hermes Agent 支持任意 Memory 后端接入。新增一个 Provider 只需要：
1. 继承 `MemoryProvider` 抽象基类
2. 实现核心生命周期方法
3. 编写 `plugin.yaml` 配置
4. 放入 `plugins/memory/<name>/` 目录

### 9.2 Frozen Snapshot 模式

System prompt 稳定 vs 磁盘实时更新。这个模式解决了两个问题：
- **性能**：前缀缓存命中，减少 API 延迟和成本
- **一致性**：会话期间 Memory 上下文不变，避免行为跳变

### 9.3 单 Provider 约束

只允许激活一个外部 Provider，防止：
- 工具 schema 膨胀（超出模型上下文预算）
- Provider 之间的行为冲突
- 调试复杂度爆炸

### 9.4 写入镜像机制

内置工具的写入 → 自动镜像给外部 Provider。这让外部 Provider 能够在不拦截工具调用的情况下，感知到 Memory 的变化，实现解耦。

### 9.5 安全扫描 + 偏移检测

- **内容扫描**：检测注入和信息泄露模式
- **偏移检测**：磁盘内容与解析器不一致时拒绝写入
- **严格模式**：system prompt 注入内容使用最严格级别

这三道防线确保 Memory 系统的安全性和一致性。

---

## 十、总结与展望

### 10.1 Hermes Memory 的核心优势

| 维度 | 说明 |
|------|------|
| **插件化** | ABC 抽象 + 注册机制，支持任意后端 |
| **本地优先** | 默认文件存储，零依赖启动 |
| **安全可控** | 内容扫描 + 偏移检测 + 严格模式 |
| **生态丰富** | Hindsight、Mem0、Honcho、Mnemosyne 等 |
| **社区驱动** | Memory OS 7 层架构、Ground Truth 等创新 |

### 10.2 与同类方案对比

| 方案 | 本地优先 | 插件化 | 知识图谱 | 社区生态 |
|------|----------|--------|----------|----------|
| **Hermes Memory** | ✅ | ✅ | ✅（Hindsight）| ✅ |
| **Mem0** | ❌ | 部分 | ❌ | ✅ |
| **Letta** | ✅ | 部分 | ❌ | 中 |
| **Zep** | ❌ | ❌ | ✅ | 中 |
| **Honcho** | ✅ | ✅ | ❌ | 小 |
| **SuperMemory** | ❌ | ❌ | ❌ | 小 |

Hermes Memory 的优势在于：**零依赖启动 + 完全插件化 + 丰富的第三方生态**。你可以在几分钟内用文件存储跑起来，也可以接入知识图谱、向量数据库、SaaS 服务。

### 10.3 未来方向

1. **更智能的记忆压缩与提取**：LLM 驱动的自动摘要和关键信息提取
2. **多 Agent 共享记忆**：多个 Agent 实例之间的 Memory 同步和共享
3. **向量搜索与知识图谱的深度融合**：Hybrid Search + Graph RAG
4. **Memory 可观测性**：记忆命中率、注入效果、遗忘曲线等指标
5. **跨平台 Memory 同步**：手机、桌面、Web 之间的 Memory 实时同步

---

## 附录：关键源码文件索引

| 文件 | 作用 |
|------|------|
| `hermes/memory/manager.py` | MemoryManager 实现 |
| `hermes/memory/provider.py` | MemoryProvider 抽象基类 |
| `hermes/memory/builtin.py` | 内置 Provider（文件存储） |
| `hermes/memory/security.py` | 安全扫描和偏移检测 |
| `hermes/memory/tools.py` | Memory 工具定义 |
| `hermes/context/engine.py` | Context Engine 实现 |
| `plugins/memory/hindsight/` | Hindsight 插件 |
| `plugins/memory/mem0/` | Mem0 插件 |

---

*本文基于 Hermes Agent 开源代码分析撰写，源码地址：https://github.com/nousresearch/hermes-agent*
