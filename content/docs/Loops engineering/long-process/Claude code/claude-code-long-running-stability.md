---
title: Claude Code 长程任务稳定性
weight: 1
---



# Claude Code 长程任务稳定性：如何在 50+ 轮迭代中不迷失、不失控、不崩溃

> **摘要**：本文深度解析 Claude Code 如何在长程编程任务（数十轮 API 往返、数小时运行、数十万 token 消耗）中保持稳定。涵盖上下文生命周期管理、5 级压缩流水线、Session 持久化与回滚、Subagent 上下文隔离、错误恢复降级链、CLAUDE.md 双记忆系统、Hooks 预处理降本、Token 预算硬限制、Checkpointing 快照等 9 大稳定机制。所有结论均以 Claude Code 官方文档、逆向可运行源码（[oboard/claude-code-rev](https://github.com/oboard/claude-code-rev)）、学术论文（[VILA-Lab, arXiv:2604.14228](https://github.com/VILA-Lab/Dive-into-Claude-Code)）、递进式教学（[shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)）和 17 篇深度解析（[openedclaude/claude-reviews-claude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview)）为交叉验证依据。

---

## 目录

- [1. 引言：长程任务的"死亡之谷"](#1-引言长程任务的死亡之谷)
- [2. 上下文生命周期管理：从加载到压缩的完整链路](#2-上下文生命周期管理从加载到压缩的完整链路)
- [3. Session 持久化与恢复：断点续传的会话管理](#3-session-持久化与恢复断点续传的会话管理)
- [4. Checkpointing：代码与对话的快照回滚](#4-checkpointing代码与对话的快照回滚)
- [5. Subagent 上下文隔离：分而治之的策略](#5-subagent-上下文隔离分而治之的策略)
- [6. 错误恢复与降级链：当事情出错时](#6-错误恢复与降级链当事情出错时)
- [7. CLAUDE.md 双记忆系统：跨会话的知识持久化](#7-claudemd-双记忆系统跨会话的知识持久化)
- [8. Hooks 预处理：在 Claude 看到之前做减法](#8-hooks-预处理在-claude-看到之前做减法)
- [9. 多 Agent 编排与成本治理：团队级稳定性](#9-多-agent-编排与成本治理团队级稳定性)
- [10. 工程卓越：支撑长程稳定的底层架构](#10-工程卓越支撑长程稳定的底层架构)
- [11. 长程任务最佳实践：从官方文档到社区经验](#11-长程任务最佳实践从官方文档到社区经验)
- [12. 可迁移设计模式：构建你自己的长程稳定 Agent](#12-可迁移设计模式构建你自己的长程稳定-agent)
- [13. 总结与展望](#13-总结与展望)
- [附录](#附录)

---

## 1. 引言：长程任务的"死亡之谷"

### 1.1 什么是长程任务？

在 Claude Code 的实践中，任务的复杂度可以从单行 Bug 修复到整个代码库的架构重构。根据任务的规模，我们可以将其分为以下几个等级：

| 任务类型 | 典型轮次 (Turns) | 典型耗时 | 典型 Token 消耗 | 复杂度描述 |
|---------|---------|---------|----------------|-----------|
| 简单修复 | 1-5 轮 | <1 分钟 | <10K | 单文件、单函数的 Bug 修复 |
| 功能开发 | 10-30 轮 | 5-30 分钟 | 50K-200K | 涉及多个文件的 Feature 实现 |
| 重构迁移 | 30-80 轮 | 30 分钟-2 小时 | 200K-500K+ | 系统性重构、API 迁移、框架升级 |
| 大型代码库探索 | 50-100+ 轮 | 1-4 小时 | 500K-1M+ | 无文档大型项目的理解与改造 |

**长程任务**的定义：**超过 20 轮 API 往返**、**涉及多个文件的系统性变更**、**需要保持上下文连贯性**的任务。这类任务对 AI 编程助手提出了最高级别的稳定性要求。

### 1.2 长程任务的"死亡之谷"

任何使用过 Claude Code 处理复杂任务的开发者都经历过一个令人沮丧的现象：任务开始时 Claude 表现卓越，但随着对话轮次的增加，质量逐渐退化。这种退化不是随机的，而是遵循一个可预测的模式——我们称之为**"死亡之谷"（Valley of Death）**：

| 阶段 | 轮次范围 | 风险等级 | 典型症状 | 根因分析 |
|------|---------|---------|---------|---------|
| **蜜月期** | 1-10 轮 | 低 | Claude 精准执行，效果显著 | 完整上下文、Prompt Cache 命中率高 |
| **疲劳期** | 10-30 轮 | 中 | 开始遗漏指令，需要重复提醒 | 早期指令被压缩、注意力分散 |
| **迷失期** | 30-50 轮 | 高 | 忘记早期目标，产生幻觉 | 上下文截断、关键信息丢失 |
| **崩溃期** | 50+ 轮 | 极高 | 循环执行、上下文溢出、任务失败 | 熵增效应达到临界点 |

这个现象的本质是**上下文熵增**——随着对话的进行，上下文中的有效信息密度不断降低，噪声和冗余不断增加，最终导致模型的理解和执行能力退化。

### 1.3 本文视角：五大参考源交叉验证

本文的每一个技术结论都经过至少两个独立参考源的交叉验证：

| 参考源 | 类型 | 规模 | 贡献维度 | 链接 |
|--------|------|------|---------|------|
| [Claude Code 官方文档](https://code.claude.com/docs/) | 官方文档 | 65+ 页面 | 机制规范、API 定义、最佳实践 | [code.claude.com/docs](https://code.claude.com/docs/) |
| [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) | 逆向工程 | 可运行源码 | 12 步状态机、compaction 源码、错误恢复逻辑 | [GitHub](https://github.com/oboard/claude-code-rev) |
| [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) | 学术论文 | 系统性分析 (arXiv:2604.14228) | 5 级 Compaction 机制、上下文衰减模型 | [GitHub](https://github.com/VILA-Lab/Dive-into-Claude-Code) |
| [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | 教学材料 | 12 阶段递进 | 长程任务最佳实践、上下文管理策略 | [GitHub](https://github.com/shareAI-lab/learn-claude-code) |
| [openedclaude/claude-reviews-claude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) | 深度解析 | 17 篇，8,600+ 行 | 工程实践、可迁移设计模式、错误恢复 | [openedclaude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) |

### 1.4 官方参考文档索引

本文交叉引用 Claude Code 官方文档共 65 篇，按主题分类如下：

#### 核心机制（与长程稳定性直接相关）

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 1 | [Agent Loop (SDK)](https://code.claude.com/docs/en/agent-sdk/agent-loop.md) | Ch2 上下文生命周期、Ch6 错误恢复 |
| 2 | [Checkpointing](https://code.claude.com/docs/en/checkpointing.md) | Ch4 快照回滚 |
| 3 | [Sessions](https://code.claude.com/docs/en/sessions.md) | Ch3 持久化与恢复 |
| 4 | [Sessions (SDK)](https://code.claude.com/docs/en/agent-sdk/sessions.md) | Ch3 持久化与恢复 |
| 5 | [Memory (CLAUDE.md)](https://code.claude.com/docs/en/memory.md) | Ch7 双记忆系统 |
| 6 | [Sub-agents](https://code.claude.com/docs/en/sub-agents.md) | Ch5 上下文隔离 |
| 7 | [Subagents (SDK)](https://code.claude.com/docs/en/agent-sdk/subagents.md) | Ch5 上下文隔离 |
| 8 | [Context Window](https://code.claude.com/docs/en/context-window.md) | Ch2 上下文生命周期 |
| 9 | [Compaction](https://code.claude.com/docs/en/compaction.md) | Ch2 5 级压缩流水线 |
| 10 | [Costs](https://code.claude.com/docs/en/costs.md) | Ch9 成本治理 |
| 11 | [Cost Tracking (SDK)](https://code.claude.com/docs/en/agent-sdk/cost-tracking.md) | Ch9 成本治理 |

#### 扩展与集成

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 12 | [Hooks](https://code.claude.com/docs/en/hooks.md) | Ch8 Hooks 预处理 |
| 13 | [Hooks (SDK)](https://code.claude.com/docs/en/agent-sdk/hooks.md) | Ch8 Hooks 预处理 |
| 14 | [Skills](https://code.claude.com/docs/en/skills.md) | Ch8 Skills 按需加载 |
| 15 | [Skills (SDK)](https://code.claude.com/docs/en/agent-sdk/skills.md) | Ch8 Skills 按需加载 |
| 16 | [Plugins](https://code.claude.com/docs/en/plugins.md) | Ch12 可迁移模式 |
| 17 | [MCP](https://code.claude.com/docs/en/mcp.md) | Ch2 上下文加载 |
| 18 | [Tool Search](https://code.claude.com/docs/en/tool-search.md) | Ch2 上下文加载 |

#### 安全与权限

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 19 | [Permissions](https://code.claude.com/docs/en/permissions.md) | Ch6 错误恢复 |
| 20 | [Permission Modes](https://code.claude.com/docs/en/permission-modes.md) | Ch6 错误恢复 |

#### 多 Agent 编排

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 21 | [Agent Teams](https://code.claude.com/docs/en/agent-teams.md) | Ch9 多 Agent 编排 |
| 22 | [Agent View](https://code.claude.com/docs/en/agent-view.md) | Ch9 多 Agent 编排 |
| 23 | [Agents (Parallel)](https://code.claude.com/docs/en/agents.md) | Ch9 多 Agent 编排 |

#### 最佳实践与工作流

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 24 | [Best Practices](https://code.claude.com/docs/en/best-practices.md) | Ch11 最佳实践 |
| 25 | [Common Workflows](https://code.claude.com/docs/en/common-workflows.md) | Ch11 最佳实践 |
| 26 | [Code Review](https://code.claude.com/docs/en/code-review.md) | Ch11 最佳实践 |
| 27 | [Status Line](https://code.claude.com/docs/en/statusline.md) | Ch11 最佳实践 |
| 28 | [Todo Tracking (SDK)](https://code.claude.com/docs/en/agent-sdk/todo-tracking.md) | Ch11 最佳实践 |

#### 工程与运维

| # | 文档 | 本文引用章节 |
|---|------|------------|
| 29 | [Claude Directory](https://code.claude.com/docs/en/claude-directory.md) | Ch7 CLAUDE.md 双记忆 |
| 30 | [Settings](https://code.claude.com/docs/en/settings.md) | Ch3 Session 管理 |
| 31 | [Environment Variables](https://code.claude.com/docs/en/env-vars.md) | Ch3 Session 管理 |
| 32 | [CLI Reference](https://code.claude.com/docs/en/cli-reference.md) | Ch3 Session 管理 |
| 33 | [Analytics](https://code.claude.com/docs/en/analytics.md) | Ch9 成本治理 |
| 34 | [Structured Outputs (SDK)](https://code.claude.com/docs/en/agent-sdk/structured-outputs.md) | Ch12 可迁移模式 |
| 35 | [File Checkpointing (SDK)](https://code.claude.com/docs/en/agent-sdk/file-checkpointing.md) | Ch4 快照回滚 |

### 1.5 核心命题

> **长程任务稳定性的本质，是对抗上下文熵增。**

Claude Code 的稳定性不是靠某个银弹功能，而是 **9 个相互协作的机制**共同对抗上下文熵增：

1. **上下文压缩（Compaction）** — 控制上下文窗口的信息密度
2. **Session 持久化与恢复** — 保证对话状态的可追溯性和可恢复性
3. **Subagent 上下文隔离** — 防止探索性任务污染主上下文
4. **错误恢复降级链** — 确保工具失败时能自主修复
5. **CLAUDE.md 双记忆系统** — 实现跨会话的知识持久化
6. **Hooks 预处理降载** — 在 Claude 看到之前做减法
7. **Token 预算硬限制** — 防止资源无限消耗
8. **Checkpointing 快照回滚** — 提供代码和对话的安全网
9. **Session 命名与分支管理** — 支持多路径探索而不丢失上下文

下面，我们将逐章深入剖析这些机制的实现原理和工程实践。

---

## 2. 上下文生命周期管理：从加载到压缩的完整链路

### 2.1 上下文窗口全景图

Claude Code 的上下文窗口是一个复杂的分层结构，每一层都有不同的加载时机、生命周期和成本特征。理解这个结构是理解长程稳定性的前提。


{{<mermaid>}}
graph TB
    subgraph "上下文窗口 (Context Window)"
        A["System Prompt<br/>系统提示 (固定)"]
        B["CLAUDE.md (4级)<br/>Managed > User > Project > Local"]
        C["工具描述 (Tool Descriptions)<br/>Built-in + MCP + Custom"]
        D["对话历史<br/>User Messages + Assistant Responses"]
        E["工具结果<br/>Tool Results (动态增长)"]
    end

    subgraph "生命周期特征"
        F["L1: 始终加载<br/>System + Managed CLAUDE.md"]
        G["L2: 会话加载<br/>User + Project CLAUDE.md"]
        H["L3: 渐进增长<br/>对话历史 + 工具结果"]
        I["L4: 动态管理<br/>Compaction 控制"]
    end

    A --> F
    B --> G
    C --> F
    D --> H
    E --> I

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e9
    style D fill:#fff3e0
    style E fill:#fce4ec
{{</mermaid>}}

上下文窗口的组成可以用以下公式表示：

```
Context Window = System Prompt
               + CLAUDE.md (4 级: Managed → User → Project → Local)
               + Tool Descriptions (Built-in + MCP + Custom Tools)
               + Conversation History (Messages)
               + Tool Results
               + Prior Work Summaries (Compaction 产物)
```

其中，**对话历史**和**工具结果**是唯一在会话期间持续增长的部分，也是熵增的主要来源。

### 2.2 上下文加载时序（Session Startup）

当 Claude Code 启动一个新会话时，上下文加载遵循严格的顺序：

{{<mermaid>}}
sequenceDiagram
    participant U as 用户
    participant CC as Claude Code CLI
    participant FS as 文件系统
    participant MCP as MCP 服务器
    participant API as Claude API

    U->>CC: 启动会话
    CC->>FS: 加载 Managed CLAUDE.md (组织级)
    CC->>FS: 加载 User CLAUDE.md (~/.claude/)
    CC->>FS: 加载 Project CLAUDE.md (./CLAUDE.md)
    CC->>FS: 加载 CLAUDE.local.md
    CC->>FS: 加载 .claude/rules/ 规则文件
    CC->>MCP: 查询可用工具列表
    MCP-->>CC: 返回工具描述
    CC->>CC: 组装工具池 (Built-in + MCP + Custom)
    CC->>API: 发送首次请求 (System + Context + Tools)
    API-->>CC: 返回响应
    CC-->>U: 等待用户输入
{{</mermaid>}}

每个加载步骤都有可估算的 token 成本：

| 加载步骤 | Token 估算 | 说明 |
|---------|-----------|------|
| System Prompt | ~2K-5K | 固定的系统提示，包含工具描述和安全约束 |
| Managed CLAUDE.md | 0-10K | 组织级配置，由管理员设置 |
| User CLAUDE.md | 0-5K | 用户级偏好和工具快捷方式 |
| Project CLAUDE.md | 0-10K | 项目架构、编码标准、构建命令 |
| CLAUDE.local.md | 0-3K | 个人沙盒配置 |
| .claude/rules/ | 0-5K | 路径级作用域规则 |
| MCP 工具描述 | 1K-10K | 取决于注册的 MCP 服务器数量 |
| **总计（冷启动）** | **3K-48K** | 实际取决于配置复杂度 |

> **关键洞察**：冷启动 token 消耗在 3K-48K 之间，这意味着对于上下文窗口为 200K 的模型，仅初始加载就可能消耗 2.4%-24% 的窗口。在长程任务中，这个初始负载会进一步压缩可用空间。

### 2.3 5 级压缩流水线（Compaction Pipeline）

Compaction 是 Claude Code 对抗上下文熵增的核心机制。根据 [VILA-Lab 论文 (arXiv:2604.14228)](https://github.com/VILA-Lab/Dive-into-Claude-Code) 的系统性分析和 [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) 的逆向工程验证，Claude Code 实现了 **5 级渐进式压缩流水线**：

| 级别 | 名称 | 触发条件 | 策略 | 代价 | 保留率 |
|------|------|---------|------|------|--------|
| **L1** | `applyToolResultBudget` | 工具结果 > 预算阈值 | 截断超大输出 | 低 | ~95% |
| **L2** | `snipCompact` | Feature-gated 触发 | 裁剪最旧的历史消息 | 中 | ~80% |
| **L3** | `microcompact` | 始终运行 | 缓存感知 + 时间衰减加权 | 低 | ~70% |
| **L4** | `contextCollapse` | Feature-gated 触发 | 折叠冗余上下文片段 | 中 | ~50% |
| **L5** | `autocompact` | 兜底触发（上下文即将溢出） | AI 生成摘要，替换历史 | 高 | ~30% |

{{<mermaid>}}
graph LR
    A["工具执行结果"] --> B{L1: 结果预算检查}
    B -->|结果过大| C["截断输出"]
    B -->|结果正常| D{L2: 历史裁剪}
    D -->|启用| E["移除最旧消息"]
    D -->|未启用| F{L3: 微压缩}
    E --> F
    F --> G["时间衰减 + 缓存感知"]
    G --> H{L4: 上下文折叠}
    H -->|启用| I["合并冗余片段"]
    H -->|未启用| J{L5: 自动压缩}
    I --> J
    J -->|兜底触发| K["AI 摘要生成"]
    J -->|未触发| L["正常继续"]
    K --> L

    style A fill:#e3f2fd
    style C fill:#fff3e0
    style E fill:#fff3e0
    style G fill:#fff3e0
    style I fill:#fff3e0
    style K fill:#fce4ec
    style L fill:#e8f5e9
{{</mermaid>}}

#### L1: applyToolResultBudget — 工具结果预算控制

这是最轻量的压缩层级，在**每次工具执行后**运行。当工具返回的结果超过预设预算时，自动截断输出。

```python
# 伪代码：applyToolResultBudget 的工作原理
def apply_tool_result_budget(tool_result: str, budget: int) -> str:
    """截断超出预算的工具结果"""
    if len(tool_result) <= budget:
        return tool_result

    # 保留开头和结尾，截断中间部分
    head_size = budget // 3
    tail_size = budget // 3
    truncation_marker = f"\n... [{len(tool_result) - head_size - tail_size} chars truncated] ...\n"

    return tool_result[:head_size] + truncation_marker + tool_result[-tail_size:]
```

这个机制的意义在于防止单个工具调用（如 `cat` 一个大型日志文件）瞬间耗尽上下文窗口。

#### L2: snipCompact — 历史裁剪

当 feature flag 启用时，snipCompact 会裁剪最旧的历史消息。这是一个**有损但安全**的操作，因为最旧的对话通常对当前任务的参考价值最低。

#### L3: microcompact — 微压缩

这是**始终运行**的压缩层级，结合了两种策略：

1. **缓存感知（Cache-Aware）**：尽量保留 Prompt Cache 中的内容，避免破坏缓存命中率
2. **时间衰减（Time Decay）**：根据消息的时间距离给予不同的保留权重

VILA-Lab 论文的上下文衰减模型表明，消息的保留价值随时间呈指数衰减：

```
保留权重 w(t) = e^(-λ × Δt)

其中：
  λ = 衰减速率常数（由 Claude Code 内部调优）
  Δt = 消息距当前轮次的时间距离
```

#### L4: contextCollapse — 上下文折叠

当上下文中有大量冗余内容时（如多次读取同一个文件），contextCollapse 会折叠这些重复片段，保留最新版本。

#### L5: autocompact — 自动压缩

这是最后的兜底机制。当上下文即将溢出且前面的压缩层级不足以释放足够空间时，Claude 会**自主生成摘要**，用一段简洁的总结替换大量历史消息。

```
autocompact 的效果：
  输入：50 轮对话历史 (~80K tokens)
  输出：1 段摘要 (~2K tokens)
  压缩率：~97.5%
  信息损失：高（但核心语义保留）
```

> **注意**：Autocompact 是信息损失最大的压缩方式。[官方文档 compaction.md](https://code.claude.com/docs/en/compaction.md) 指出，压缩后的摘要虽然保留了核心语义，但具体的代码片段、参数值和精确指令可能丢失。

### 2.4 Prompt Cache 锁存效应

Claude Code 的上下文加载过程中存在一个重要的优化机制：**Prompt Cache 锁存（Prompt Cache Latching）**。

{{<mermaid>}}
graph LR
    subgraph "首次调用 (Cold)"
        A1["System Prompt + CLAUDE.md<br/>全额计费"]
    end

    subgraph "后续调用 (Cache Hit)"
        A2["System Prompt + CLAUDE.md<br/>~50% 成本"]
    end

    subgraph "Compaction 后 (Cache Miss)"
        A3["System Prompt + 摘要<br/>全额计费<br/>缓存失效"]
    end

    A1 -->|"首次加载后缓存"| A2
    A2 -->|"Compaction 修改上下文"| A3
    A3 -->|"重新缓存"| A2

    style A1 fill:#fce4ec
    style A2 fill:#e8f5e9
    style A3 fill:#fff3e0
{{</mermaid>}}

根据 [官方 costs.md 文档](https://code.claude.com/docs/en/costs.md)：

- System Prompt 和 CLAUDE.md 在前几次调用后进入 Prompt Cache
- 缓存命中后，这些重复内容的成本降低约 **50%**
- 但 Compaction（尤其是 L4 和 L5）会**破坏缓存**，导致下一次调用重新全额计费

这个权衡是 Claude Code 设计中一个经典的**成本-稳定性权衡**：
- 保持稳定（不 Compaction）→ 缓存命中率高 → 成本低 → 但上下文可能溢出
- 积极 Compaction → 避免溢出 → 但缓存失效 → 成本上升

### 2.5 上下文窗口使用率监控

Claude Code 提供了多种方式监控上下文使用情况：

| 方法 | 命令/接口 | 说明 |
|------|-----------|------|
| 状态栏 | `/context` | 显示当前上下文使用率 |
| SDK | `context` 消息类型 | 在 agent stream 中返回 token 使用量 |
| Status Line | 自定义配置 | 在终端状态栏实时显示 |
| 成本报告 | `/usage` | 显示累积 token 和成本 |

```typescript
// SDK 中监控上下文使用
for await (const message of query({ prompt: "Refactor the auth module" })) {
    if (message.type === "context") {
        const usage = message.context;
        console.log(`Tokens: ${usage.input_tokens} input, ${usage.output_tokens} output`);
        console.log(`Cache: ${usage.cache_creation_input_tokens} created, ${usage.cache_read_input_tokens} read`);
    }
}
```

---

## 3. Session 持久化与恢复：断点续传的会话管理

### 3.1 JSONL 会话存储

Claude Code 的会话数据以 **JSONL（JSON Lines）** 格式持久化到磁盘：

```
存储路径: ~/.claude/projects/<project-name>/<session-id>.jsonl
```

JSONL 格式的每一行都是一个独立的 JSON 对象，包含以下类型的记录：

| 记录类型 | 内容 | 说明 |
|---------|------|------|
| `user` | 用户消息 | 用户的输入文本 |
| `assistant` | AI 回复 | Claude 的回复内容 |
| `tool_use` | 工具调用 | 工具名称和参数 |
| `tool_result` | 工具结果 | 工具执行的输出 |
| `metadata` | 元数据 | 时间戳、token 计数等 |

```
session-abc123.jsonl 示例：
{"type":"user","content":"Refactor the auth module","timestamp":"2025-01-15T10:00:00Z"}
{"type":"assistant","content":"I'll start by exploring the auth directory...","timestamp":"2025-01-15T10:00:05Z"}
{"type":"tool_use","name":"Read","input":{"path":"src/auth/login.ts"},"timestamp":"2025-01-15T10:00:06Z"}
{"type":"tool_result","content":"export function login(...)","timestamp":"2025-01-15T10:00:07Z"}
```

**关键设计特性**：

1. **连续写入，无批量 flush** — 每条消息立即写入磁盘，确保即使进程崩溃也不会丢失最近的对话
2. **追加写入，不修改历史** — 只追加新记录，从不修改已有记录，保证数据一致性
3. **项目隔离** — 每个项目的会话存储在不同的子目录中

### 3.2 Session 命名与发现

Claude Code 支持在启动时或会话中对 Session 进行命名，使其更容易识别和恢复：

| 操作 | 命令 | 说明 |
|------|------|------|
| 启动时命名 | `claude -n auth-refactor` | 创建可恢复的命名会话 |
| 会话中重命名 | `/rename auth-refactor` | 动态修改当前会话名称 |
| 恢复命名会话 | `claude --resume auth-refactor` | 直接恢复到指定会话 |
| Session Picker | `/resume` | 交互式浏览和选择历史会话 |

{{<mermaid>}}
graph LR
    A["claude -n auth-refactor"] --> B["创建命名会话"]
    B --> C["对话进行..."]
    C --> D["终端关闭"]
    D --> E["数据持久化到 JSONL"]
    E --> F["claude --resume auth-refactor"]
    F --> G["恢复完整上下文"]

    style B fill:#e1f5fe
    style E fill:#e8f5e9
    style G fill:#c8e6c9
{{</mermaid>}}

### 3.3 Session 分支（Branching）

Session 分支是 Claude Code 长程任务稳定性的关键功能之一。它允许你在不丢失原始对话的前提下，探索不同的实现路径。

{{<mermaid>}}
graph TB
    A["主会话: auth-refactor"] --> B["轮次 1-15: 正常开发"]
    B --> C["决策点: 需要尝试不同方案"]
    C --> D["方案 A: /branch → JWT 方案"]
    C --> E["方案 B: /branch → OAuth 方案"]
    C --> F["主会话: 保持不变"]
    D --> G["在分支上继续 10 轮"]
    E --> H["在分支上继续 10 轮"]
    G --> I["评估: JWT 方案更好"]
    H --> J["评估: OAuth 方案不适合"]

    style A fill:#e1f5fe
    style F fill:#e1f5fe
    style D fill:#fff3e0
    style E fill:#f3e5f5
    style I fill:#e8f5e9
    style J fill:#fce4ec
{{</mermaid>}}

**分支的重要特性**：

| 特性 | 说明 | 影响 |
|------|------|------|
| 对话复制 | `/branch` 创建当前对话的完整副本 | 保留所有上下文 |
| 原会话不变 | 原始会话保持完整，不受分支影响 | 安全探索 |
| 权限不继承 | "allow for this session" 的权限不传递到分支 | 安全隔离 |
| 独立存储 | 分支有独立的 JSONL 文件 | 独立管理生命周期 |

命令行分支：
```bash
# 从上一个会话继续并分支
claude --continue --fork-session
```

### 3.4 会话清理与生命周期

Claude Code 对历史会话有自动清理机制：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| 自动清理周期 | 30 天 | 超过 30 天的会话自动清理 |
| 存储路径 | `~/.claude/projects/` | 可通过 `CLAUDE_CONFIG_DIR` 自定义 |
| 跳过持久化 | 关闭 | `CLAUDE_CODE_SKIP_PROMPT_HISTORY=1` 跳过写入 |
| 非交互模式 | 保存 | `--no-session-persistence` 非交互模式不保存 |

```bash
# 自定义存储路径
export CLAUDE_CONFIG_DIR=/mnt/data/claude-configs

# 不记录历史（隐私模式）
export CLAUDE_CODE_SKIP_PROMPT_HISTORY=1

# 非交互模式不保存会话
claude --print "Generate a report" --no-session-persistence
```

对于长程任务，建议**不要**启用跳过持久化，因为这意味着一旦进程崩溃，所有对话记录都将丢失。

### 3.5 SDK 中的 Session 管理

在 Claude Agent SDK 中，Session 管理通过 `resume` 选项实现：

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// 恢复之前的会话
for await (const message of query({
    prompt: "继续之前的工作",
    options: {
        resume: "previous-session-id",  // 恢复指定会话
    }
})) {
    console.log(message);
}
```

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions

async def resume_session():
    async for message in query(
        prompt="继续之前的工作",
        options=ClaudeAgentOptions(
            resume="previous-session-id"  # 恢复指定会话
        )
    ):
        print(message)

asyncio.run(resume_session())
```

---

## 4. Checkpointing：代码与对话的快照回滚

### 4.1 Checkpoint 机制

Checkpointing 是 Claude Code 提供的**安全网机制**。每次工具执行后，Claude Code 自动创建代码状态的快照，使你可以随时回滚到之前的任何状态。

{{<mermaid>}}
graph LR
    A["工具执行前"] --> B["自动创建 Checkpoint"]
    B --> C["执行工具操作"]
    C --> D["代码发生变化"]
    D --> E["下一个工具执行前"]
    E --> B

    style B fill:#e8f5e9
    style D fill:#fce4ec
{{</mermaid>}}

**核心特性**：

| 特性 | 说明 |
|------|------|
| 自动创建 | 每次工具执行后自动创建，无需手动操作 |
| 文件级差异 | 记录每个文件的变更（类似 git diff） |
| 对话关联 | 每个 Checkpoint 与特定的对话轮次关联 |
| 会话内有效 | Checkpoint 在当前会话内可用 |

### 4.2 Checkpoint vs Git Commit

理解 Checkpoint 与 Git Commit 的区别对于正确使用它们至关重要：

| 特性 | Checkpoint | Git Commit |
|------|-----------|-----------|
| **创建频率** | 每次工具执行后自动 | 手动触发 |
| **粒度** | 文件级差异 | 仓库级快照 |
| **回滚范围** | 代码 + 对话上下文 | 仅代码 |
| **用户感知** | 完全自动，零感知 | 需要手动操作 |
| **持久性** | 会话内有效 | 永久存储（除非 force push） |
| **回滚命令** | `/checkpoint revert <id>` | `git revert` / `git reset` |
| **可分享性** | 仅限本地会话 | 可推送到远程仓库 |

> **最佳实践**：Checkpoint 和 Git Commit 是互补关系，而非替代关系。在长程任务中，Checkpoint 提供细粒度的快速回滚能力，而 Git Commit 提供永久的版本控制。

### 4.3 回滚策略

Claude Code 支持两种回滚模式：

{{<mermaid>}}
graph TB
    A["检测到问题"] --> B{回滚策略选择}
    B --> C["渐进式回滚<br/>Step-by-step revert"]
    B --> D["全量回滚<br/>Full snapshot revert"]

    C --> E["回滚到上一 Checkpoint"]
    E --> F["注入回滚后的上下文"]
    F --> G["Claude 理解变更并继续"]

    D --> H["回滚到指定 Checkpoint"]
    H --> I["恢复完整对话状态"]
    I --> G

    style C fill:#e1f5fe
    style D fill:#f3e5f5
    style G fill:#e8f5e9
{{</mermaid>}}

**渐进式回滚**：回滚到上一个 Checkpoint，只撤销最近的一次工具操作。适用于单步操作出现问题的情况。

**全量回滚**：回滚到任意指定的 Checkpoint，恢复到该时间点的完整代码状态。适用于发现多个操作都有问题的情况。

回滚后，Claude Code 会**自动注入回滚上下文**到对话中，让 Claude 知道代码已经回滚，以及回滚到了哪个状态。这确保 Claude 不会在回滚后继续基于错误的代码状态进行操作。

### 4.4 SDK 中的文件 Checkpointing

在 Claude Agent SDK 中，Checkpointing 通过文件快照实现：

```python
from claude_agent_sdk import query, ClaudeAgentOptions

async def run_with_checkpointing():
    async for message in query(
        prompt="Refactor the database migration scripts",
        options=ClaudeAgentOptions(
            # 启用文件 checkpointing
            # SDK 会自动在每次工具执行后创建快照
        )
    ):
        if message.type == "result":
            print("Task completed with full checkpoint history")
```

SDK 的 checkpointing 确保在长时间运行的任务中，即使中间步骤失败，也可以从最近的快照恢复。

---

## 5. Subagent 上下文隔离：分而治之的策略

### 5.1 为什么需要上下文隔离？

在长程任务中，上下文窗口是最稀缺的资源。以下操作会快速填充上下文：

- **探索性文件阅读**：读取多个文件以理解代码结构
- **代码搜索**：使用 Grep/Glob 搜索模式匹配
- **文档查阅**：阅读外部文档或 API 参考
- **大规模重构**：同时修改数十个文件

如果所有这些操作都在主会话的上下文窗口中进行，窗口会迅速被填满，导致**核心任务逻辑的上下文被挤压**。

Subagent 的核心价值在于：**在独立的上下文窗口中执行探索性任务，仅将结果摘要返回给主会话**。

{{<mermaid>}}
graph TB
    subgraph "主会话 (Main Session)"
        A["核心任务上下文<br/>~200K tokens"]
        A1["System Prompt + CLAUDE.md"]
        A2["对话历史"]
        A3["Subagent 摘要 (仅 1-2K tokens)"]
    end

    subgraph "Subagent (Explore)"
        B["独立上下文窗口<br/>~200K tokens"]
        B1["读取 20 个文件"]
        B2["搜索代码模式"]
        B3["生成摘要报告"]
    end

    A3 -.->|"摘要返回"| A
    A -->|"委派任务"| B

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style A3 fill:#e8f5e9
{{</mermaid>}}

### 5.2 内建 Subagent 体系

Claude Code 内建了多种 Subagent，每种都有不同的模型、工具集和上下文隔离策略：

| Subagent | 模型 | 工具权限 | 用途 | 上下文隔离 | 典型 Token 消耗 |
|---------|------|---------|------|-----------|---------------|
| **Explore** | Haiku | 只读 (Read, Grep, Glob) | 代码搜索、文件发现 | 完全隔离 | ~5K-20K |
| **Plan** | 继承主会话 | 只读 | Plan Mode 调研 | 完全隔离 | ~10K-30K |
| **General-purpose** | 继承主会话 | 全部工具 | 复杂多步任务 | 完全隔离 | ~20K-100K |
| **statusline-setup** | Sonnet | 有限 | 状态栏配置 | 完全隔离 | ~2K-5K |
| **claude-code-guide** | Haiku | 有限 | 功能问答 | 完全隔离 | ~2K-10K |

> **关键洞察**：Explore Subagent 使用 Haiku 模型（成本最低），且仅具有只读工具权限。这意味着大量的探索性文件读取不会消耗主会话的昂贵 Sonnet/Opus token，也不会污染主会话的上下文。

### 5.3 Subagent 上下文成本分析

```
主会话 Token 计算：
  主会话 Token = System Prompt + CLAUDE.md + 对话历史 + Subagent 摘要

Subagent Token 计算：
  Subagent Token = 独立窗口 (System Prompt + CLAUDE.md + 完整探索历史)

成本节省：
  如果直接在主会话中探索 20 个文件: ~50K tokens (Sonnet 价格)
  使用 Explore Subagent: ~10K tokens (Haiku 价格) + ~1K tokens (摘要)
  节省: 30-70% 的主会话 token 消耗
```

### 5.4 自定义 Subagent

Claude Code 支持通过 Markdown + YAML frontmatter 定义自定义 Subagent：

```markdown
---
name: security-reviewer
model: claude-sonnet-4-20250514
description: Reviews code for security vulnerabilities
tools:
  - Read
  - Grep
  - Glob
---

You are a security code reviewer. Analyze the codebase for:
1. SQL injection vulnerabilities
2. XSS (Cross-Site Scripting) risks
3. Exposed credentials or API keys
4. Insecure authentication patterns
5. Missing input validation
```

**5 个作用域**（优先级从高到低）：

| 优先级 | 作用域 | 路径 | 说明 |
|--------|--------|------|------|
| 1 | Managed Settings | 组织级 | 管理员强制配置 |
| 2 | CLI | 命令行参数 | `--agent` 指定 |
| 3 | 项目 | `.claude/agents/` | 团队共享 |
| 4 | 用户 | `~/.claude/agents/` | 个人配置 |
| 5 | Plugin | 插件目录 | 插件提供 |

每个 Subagent 有独立的记忆目录（`~/.claude/agent-memory/`），确保不同 Agent 之间的学习不会互相干扰。

---

## 6. 错误恢复与降级链：当事情出错时

### 6.1 错误分类矩阵

在长程任务中，各种错误不可避免。Claude Code 通过**分类错误并应用不同的恢复策略**来维持任务稳定性：

| 错误类型 | 发生频率 | 严重程度 | 恢复策略 | 自动/手动 |
|---------|---------|---------|---------|----------|
| **API Rate Limit** | 高 | 低 | 指数退避重试 (Exponential Backoff) | 自动 |
| **Token Budget 超支** | 中 | 中 | 触发 Compaction → 警告 → 终止 | 半自动 |
| **工具执行失败** | 中 | 中 | 错误注入上下文，Claude 自主修复 | 自动 |
| **网络超时** | 低 | 中 | 重试 + 降级到备用端点 | 自动 |
| **Prompt Cache Miss** | 低 | 低 | 重新加载（成本增加 ~50%） | 自动 |
| **上下文溢出** | 低 | 高 | 强制 Autocompact | 自动 |
| **权限拒绝** | 中 | 中 | 请求用户授权或跳过操作 | 手动 |

### 6.2 TokenBudget 硬限制

Claude Code 通过 `tokenBudget` 参数控制最大 token 消耗，防止长程任务无限消耗资源：

{{<mermaid>}}
graph LR
    A["任务开始"] --> B["监控 token 消耗"]
    B --> C{是否超过预算?}
    C -->|否| B
    C -->|是 (80%)| D["L1: 触发 Compaction"]
    D --> E["Compaction 后仍然超预算?"]
    E -->|否| B
    E -->|是 (95%)| F["L2: 警告用户"]
    F --> G["用户选择: 继续/终止"]
    G -->|继续| H["L3: 更激进 Compaction"]
    G -->|终止| I["L4: 安全终止"]

    style A fill:#e1f5fe
    style D fill:#fff3e0
    style F fill:#fce4ec
    style I fill:#e8f5e9
{{</mermaid>}}

[openedclaude/claude-reviews-claude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) 的 12 步状态机分析表明，TokenBudget 的实现遵循以下状态流转：

```
IDLE → RUNNING → APPROACHING_LIMIT → COMPACTING → WARNING → TERMINATING → STOPPED
```

### 6.3 工具错误注入与自主修复

这是 Claude Code 错误恢复机制中最精妙的设计之一：**工具失败结果作为普通消息注入上下文**，让 Claude 像处理用户消息一样处理错误。

{{<mermaid>}}
sequenceDiagram
    participant C as Claude Code
    participant T as 工具 (Bash)
    participant M as Claude 模型
    participant U as 用户

    C->>T: 执行 `npm run build`
    T-->>C: 失败: "Error: Cannot find module 'express'"
    C->>M: 注入错误结果到上下文
    M->>M: 分析错误: 缺少 express 依赖
    M->>T: 执行 `npm install express`
    T-->>C: 成功
    M->>U: "已安装缺失的 express 依赖，重试构建"
    C->>T: 执行 `npm run build`
    T-->>C: 成功
{{</mermaid>}}

**自主修复的局限性**：
- 最大重试次数由 `applyToolResultBudget` 控制
- 每次重试都会消耗 token
- 如果修复尝试也失败，Claude 会向用户报告问题并请求指导

### 6.4 StopHooks 与提前终止

Claude Code 的 Hooks 系统支持在任务执行过程中进行拦截和干预（详见第 8 章）。在错误恢复场景中，特别重要的是 **StopHook** 机制：

| Hook 类型 | 触发时机 | 错误恢复用途 |
|-----------|---------|------------|
| **PreToolUse** | 工具执行前 | 拦截危险操作、修改命令参数 |
| **PostToolUse** | 工具执行后 | 修改输出、检测错误模式 |
| **Notification** | 通知发送前 | 过滤噪声通知 |
| **Stop** | 任务停止前 | 清理资源、保存状态 |

StopHook 可以在任务被终止前执行清理操作，确保：
1. 未保存的代码变更被回滚
2. 中间状态被持久化
3. 资源（MCP 连接、临时文件）被正确释放

### 6.5 权限系统的错误恢复

[官方 permissions.md](https://code.claude.com/docs/en/permissions.md) 和 [permission-modes.md](https://code.claude.com/docs/en/permission-modes.md) 文档定义了多层权限控制：

| 权限模式 | 行为 | 适用场景 |
|---------|------|---------|
| **Default** | 危险操作需要确认 | 日常开发 |
| **Accept Edits** | 接受所有编辑，确认危险操作 | 快速迭代 |
| **Danger Mode** | 所有操作自动执行 | CI/CD、自动化任务 |

在长程任务中，权限确认可能成为瓶颈。[auto-mode-config](https://code.claude.com/docs/en/auto-mode-config.md) 允许预先配置特定命令的自动批准，减少人工干预：

```json
// .claude/settings.json
{
    "permissions": {
        "allow": [
            "Bash(npm install *)",
            "Bash(npm run build)",
            "Bash(npm test)"
        ]
    }
}
```

---

## 7. CLAUDE.md 双记忆系统：跨会话的知识持久化

### 7.1 双记忆架构

Claude Code 实现了两种互补的记忆机制，共同支撑跨会话的知识持久化：

| 维度 | CLAUDE.md | Auto Memory |
|------|-----------|-------------|
| **谁写入** | 用户（手动编辑） | Claude（自动学习） |
| **内容类型** | 指令、规则、标准 | 学习到的模式、经验 |
| **作用域** | 项目/用户/组织 | 每个仓库 |
| **加载方式** | 每次会话完整加载 | 每次会话加载（前 200 行或 25KB） |
| **更新频率** | 用户主动修改 | Claude 持续自动更新 |
| **示例内容** | 编码标准、架构决策 | 构建命令、调试经验、用户偏好 |

{{<mermaid>}}
graph TB
    subgraph "CLAUDE.md (用户指令)"
        A["编码标准"]
        B["架构决策"]
        C["工作流规范"]
        D["工具配置"]
    end

    subgraph "Auto Memory (自动学习)"
        E["构建命令"]
        F["调试经验"]
        G["用户偏好"]
        H["常见错误模式"]
    end

    subgraph "Claude Code 会话"
        I["会话 1"]
        J["会话 2"]
        K["会话 3"]
    end

    A --> I
    B --> I
    E --> I
    F --> I

    I -->|"学习经验"| E
    I -->|"学习经验"| F

    E --> J
    F --> J
    A --> J
    B --> J

    style A fill:#e1f5fe
    style B fill:#e1f5fe
    style E fill:#f3e5f5
    style F fill:#f3e5f5
{{</mermaid>}}

> **关键洞察**：[官方 memory.md](https://code.claude.com/docs/en/memory.md) 指出，Auto Memory 是 Claude 自动学习和存储的信息，而 CLAUDE.md 是你想要 Claude 始终记住的信息。这两者的分工类似于人类的"程序性记忆"（CLAUDE.md，有意识的规则）和"陈述性记忆"（Auto Memory，经验积累）。

### 7.2 CLAUDE.md 四级加载

CLAUDE.md 支持四级作用域，从组织级到本地级，优先级依次递增：

| 级别 | 路径 | 作用域 | 示例内容 | 优先级 |
|------|------|--------|---------|--------|
| **L1** | Managed Settings | 组织级 | 公司编码标准、安全策略、合规要求 | 最低（可被覆盖） |
| **L2** | `~/.claude/CLAUDE.md` | 用户级 | 个人编码偏好、常用工具快捷方式 | 中低 |
| **L3** | `./CLAUDE.md` | 项目级 | 项目架构、编码标准、构建命令 | 中高 |
| **L4** | `./CLAUDE.local.md` | 本地级 | 个人沙盒 URL、测试数据路径 | 最高 |

```bash
CLAUDE.md 加载顺序（优先级从低到高）：
  1. Managed Settings (组织管理员设置)
  2. ~/.claude/CLAUDE.md (用户全局配置)
  3. ./CLAUDE.md (项目配置)
  4. ./CLAUDE.local.md (本地覆盖)

后加载的配置会覆盖先加载的同名配置。
```

### 7.3 .claude/rules/ 路径级作用域

除了 CLAUDE.md 的四级加载，Claude Code 还支持 `.claude/rules/` 目录实现**路径级作用域**的规则：

```
.claude/rules/
├── frontend/     # 仅在前端目录生效
│   └── rules.md
├── backend/      # 仅在后端目录生效
│   └── rules.md
└── global.md     # 全局规则
```

这在大型项目中特别有用，因为不同模块可能有完全不同的编码标准和架构约束。

### 7.4 记忆衰减与一致性

[官方 memory.md](https://code.claude.com/docs/en/memory.md) 明确指出：

> CLAUDE.md is treated as context, not as a set of strict rules that Claude always follows.

这意味着：

1. **CLAUDE.md 是上下文而非硬约束** — Claude 会参考 CLAUDE.md 中的指令，但不会将其视为不可违背的规则
2. **文件过大导致遵循度下降** — 当 CLAUDE.md 超过 200 行时，Claude 对其中指令的遵循度会显著下降
3. **上下文压缩的影响** — Compaction 可能丢失 CLAUDE.md 的部分内容

> **最佳实践**：保持 CLAUDE.md 精简（<200 行），将详细指令迁移到 Skills 中按需加载（详见第 8 章）。

---

## 8. Hooks 预处理：在 Claude 看到之前做减法

### 8.1 Hooks 架构

Hooks 是 Claude Code 最强大的扩展机制之一。它们允许你在 Claude **看到**数据之前或之后进行拦截、过滤、修改和增强。

{{<mermaid>}}
graph LR
    A["工具执行"] --> B["PreToolUse Hook"]
    B --> C{"Hook 批准?"}
    C -->|否| D["阻止工具执行"]
    C -->|是| E["执行工具"]
    E --> F["PostToolUse Hook"]
    F --> G["修改输出结果"]
    G --> H["注入 Claude 上下文"]
    H --> I["Claude 处理"]

    style B fill:#e1f5fe
    style F fill:#f3e5f5
    style D fill:#fce4ec
    style I fill:#e8f5e9
{{</mermaid>}}

| Hook 类型 | 触发时机 | 典型用途 | 返回值 |
|-----------|---------|---------|--------|
| **PreToolUse** | 工具执行前 | 修改命令参数、拦截危险操作、注入环境变量 | approve/deny/modify |
| **PostToolUse** | 工具执行后 | 过滤输出、格式化结果、检测错误模式 | modified_output |
| **Notification** | 通知发送前 | 过滤噪声、转发到外部系统 | modified_notification |
| **Stop** | 任务停止前 | 清理资源、保存状态、生成报告 | cleanup_actions |

### 8.2 降本案例：10,000 行日志 → 100 行

Hooks 在降低长程任务 token 消耗方面效果显著。以下是一个经典的 PostToolUse Hook 案例：

```json
// .claude/hooks.json
{
    "hooks": [
        {
            "matcher": "pytest",
            "hook": "PostToolUse",
            "command": "bash -c 'cat - | grep -A 5 -E \"(FAIL|ERROR|error:)\" | head -100'"
        }
    ]
}
```

**效果对比**：

| 场景 | Hook 前 | Hook 后 | 节省 |
|------|---------|---------|------|
| 运行完整测试套件 | ~10,000 行输出 (~50K tokens) | ~100 行输出 (~500 tokens) | **~99%** |
| 构建日志 | ~5,000 行输出 (~25K tokens) | ~50 行输出 (~250 tokens) | **~99%** |
| 依赖安装输出 | ~500 行输出 (~2.5K tokens) | ~10 行输出 (~50 tokens) | **~98%** |

在长程任务中，如果一个任务涉及 50 次测试运行，Hook 前的 token 消耗为 50 × 50K = 2.5M tokens，而 Hook 后仅为 50 × 500 = 25K tokens。这是一个 **100 倍的成本差异**。

### 8.3 将指令从 CLAUDE.md 迁移到 Skills

如第 7 章所述，CLAUDE.md 每次会话加载会消耗大量 token。[官方 skills.md](https://code.claude.com/docs/en/skills.md) 提供了一种更高效的替代方案：

| 对比维度 | CLAUDE.md | Skills |
|---------|-----------|--------|
| **加载方式** | 每次会话自动加载 | 按需加载（Claude 自主决定） |
| **Token 成本** | 每次会话固定消耗 | 仅使用时消耗 |
| **适用内容** | 全局规则、核心指令 | 领域知识、复杂流程、最佳实践 |
| **维护成本** | 容易膨胀 | 模块化，易维护 |

**迁移策略**：

```
Step 1: 分析 CLAUDE.md，识别"不常用"的指令
Step 2: 将这些指令提取为独立的 Skill 文件
Step 3: 在 CLAUDE.md 中保留简短的 Skill 引用
Step 4: 让 Claude 自主决定何时加载 Skill
```

```markdown
<!-- .claude/skills/deploy/SKILL.md -->
---
description: Deploy the application to staging or production. Use when the user asks to deploy, release, or publish the app.
---

# Deployment Skill

## Staging Deployment
1. Run `npm run build`
2. Run `npm test`
3. Execute `./scripts/deploy.sh staging`
4. Verify health check at `/health`

## Production Deployment
1. Require explicit user confirmation
2. Run `npm run build --production`
3. Execute `./scripts/deploy.sh production`
4. Run smoke tests
5. Notify the team
```

```markdown
<!-- CLAUDE.md (精简版) -->
# Project: MyApp

- Framework: React 18 + TypeScript
- Build: `npm run build`
- Test: `npm test`
- For deployment, use the `/deploy` skill
- For database migrations, use the `/migrate` skill
```

通过这种迁移，CLAUDE.md 从原来的 500 行精简到 50 行，每次会话节省约 2K tokens。

### 8.4 SDK 中的 Hooks

在 Claude Agent SDK 中，Hooks 通过编程方式注册：

```typescript
import { query, HookEvent } from "@anthropic-ai/claude-agent-sdk";

// 注册 PreToolUse Hook
for await (const message of query({
    prompt: "Run the test suite",
    options: {
        hooks: {
            preToolUse: async (event: HookEvent) => {
                if (event.toolName === "Bash" && event.input.command.includes("rm -rf")) {
                    return { type: "deny", message: "Dangerous operation blocked" };
                }
                return { type: "approve" };
            }
        }
    }
})) {
    console.log(message);
}
```

```python
from claude_agent_sdk import query, ClaudeAgentOptions

async def pre_tool_use_hook(event):
    if event.tool_name == "Bash" and "rm -rf" in event.input.command:
        return {"type": "deny", "message": "危险操作被阻止"}
    return {"type": "approve"}

async for message in query(
    prompt="Run tests",
    options=ClaudeAgentOptions(
        hooks={"pre_tool_use": pre_tool_use_hook}
    )
):
    print(message)
```

---

## 9. 多 Agent 编排与成本治理：团队级稳定性

### 9.1 Agent Teams

[官方 agent-teams.md](https://code.claude.com/docs/en/agent-teams.md) 描述了 Claude Code 的多 Agent 协作模式：

{{<mermaid>}}
graph TB
    subgraph "Agent Team"
        A["Leader Agent<br/>主 Claude Code 实例"]
        B["Teammate 1<br/>独立的 Claude Code 实例"]
        C["Teammate 2<br/>独立的 Claude Code 实例"]
        D["Teammate N<br/>独立的 Claude Code 实例"]
    end

    A -->|"分配任务"| B
    A -->|"分配任务"| C
    A -->|"分配任务"| D
    B -->|"报告结果"| A
    C -->|"报告结果"| A
    D -->|"报告结果"| A

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e9
    style D fill:#fff3e0
{{</mermaid>}}

**关键特性**：

| 特性 | 说明 | 对稳定性的影响 |
|------|------|---------------|
| 独立上下文 | 每个 teammate 有独立的上下文窗口 | 防止主会话上下文膨胀 |
| 独立 token 预算 | 每个 teammate 有自己的 token 限制 | 控制总体成本 |
| 并行执行 | teammates 可以同时工作 | 缩短总耗时 |
| 结果聚合 | Leader 汇总所有 teammate 的结果 | 保持主会话简洁 |

**成本公式**：
```
总 Token 消耗 = Leader Token + Σ(Teammate Token)

其中：
  Leader Token ≈ 主会话上下文 + 任务分配 + 结果聚合
  Teammate Token ≈ 独立上下文 + 任务执行
```

### 9.2 成本控制策略

| 策略 | 说明 | 效果 | 适用场景 |
|------|------|------|---------|
| **使用 Sonnet 作为 teammate** | teammate 使用 Sonnet 而非 Opus | ~50% 降本 | 大部分代码任务 |
| **保持小团队** | 减少并发 teammate 数量 | 线性降本 | 成本敏感项目 |
| **Spawn Prompt 精简** | 减少 teammate 的初始上下文 | 10-20% 降本 | 任务明确的场景 |
| **任务完成即清理** | 及时终止空闲 teammate | 防止持续浪费 | 长程任务 |
| **Haiku 用于子任务** | 简单任务配置 `model: haiku` | ~80% 降本 | 文档生成、格式检查 |

```json
// .claude/agents/quick-reviewer.md
---
name: quick-reviewer
model: claude-haiku
description: Quick code review for small changes
---

Review the code changes for obvious issues only.
Focus on: syntax errors, missing imports, typos.
Keep the review brief.
```

### 9.3 企业级速率限制建议

[官方 costs.md](https://code.claude.com/docs/en/costs.md) 提供了针对不同团队规模的速率限制建议：

| 团队规模 | TPM/用户 (Tokens Per Minute) | RPM/用户 (Requests Per Minute) |
|---------|---------------------------|---------------------------|
| 1-5 人 | 200K-300K | 5-7 |
| 5-20 人 | 100K-150K | 2.5-3.5 |
| 20-50 人 | 50K-75K | 1.25-1.75 |
| 50-100 人 | 25K-35K | 0.62-0.87 |
| 100-500 人 | 15K-20K | 0.37-0.47 |
| 500+ 人 | 10K-15K | 0.25-0.35 |

[官方 admin-setup.md](https://code.claude.com/docs/en/admin-setup.md) 描述了管理员如何配置和强制执行这些限制。

### 9.4 Agent View 与可观测性

[官方 agent-view.md](https://code.claude.com/docs/en/agent-view.md) 提供了多 Agent 协作的可观测性：

| 维度 | 说明 |
|------|------|
| 实时状态 | 查看每个 teammate 的当前操作和进度 |
| 成本追踪 | 每个 teammate 的独立 token 消耗 |
| 工具使用 | 每个 teammate 使用的工具列表 |
| 决策链 | 跟踪 Leader 如何分配任务 |

```typescript
// SDK 中的成本追踪
for await (const message of query({
    prompt: "Build the feature with 2 teammates",
    options: { maxTurns: 20 }
})) {
    if (message.type === "result" && message.cost) {
        console.log(`Total cost: $${message.cost.total_cost_usd}`);
        console.log(`Input tokens: ${message.cost.input_tokens}`);
        console.log(`Output tokens: ${message.cost.output_tokens}`);
    }
}
```

---

## 10. 工程卓越：支撑长程稳定的底层架构

### 10.1 35 行 Store：零依赖状态管理

[openedclaude/claude-reviews-claude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) 深入分析了 Claude Code 的状态管理架构。令人惊讶的是，Claude Code 没有使用 Redux、Zustand 或任何第三方状态管理库，而是实现了一个仅 35 行的 Store：

```typescript
// 简化的 Store 实现（基于 openedclaude 分析）
type Listener<T> = (state: T) => void;

class Store<T> {
    private state: T;
    private listeners: Set<Listener<T>> = new Set();

    constructor(initialState: T) {
        this.state = initialState;
    }

    getState(): T {
        return this.state;
    }

    setState(next: T | ((prev: T) => T)): void {
        this.state = typeof next === 'function'
            ? (next as (prev: T) => T)(this.state)
            : next;
        this.notify();
    }

    subscribe(listener: Listener<T>): () => void {
        this.listeners.add(listener);
        return () => this.listeners.delete(listener);
    }

    private notify(): void {
        this.listeners.forEach(l => l(this.state));
    }
}
```

**对长程稳定性的意义**：

1. **状态一致**：发布-订阅模式确保所有组件看到相同的最新状态
2. **无内存泄漏**：订阅者返回取消订阅函数，防止悬挂引用
3. **零中间件**：没有 Redux DevTools、thunk、saga 等复杂层，减少故障面
4. **可预测**：纯函数式的状态更新，易于测试和调试

### 10.2 Import-Gap 并行化

Claude Code 启动时面临一个挑战：Node.js 的模块加载是串行的，大量依赖会导致明显的冷启动延迟。

Claude Code 通过 **Import-Gap 并行化** 解决这个问题：

{{<mermaid>}}
graph LR
    A["启动开始"] --> B["串行加载核心模块"]
    B --> C["识别可并行加载的模块"]
    C --> D["并行加载依赖"]
    D --> E["等待所有 Promise 完成"]
    E --> F["所有模块就绪"]
    F --> G["开始处理用户输入"]

    style A fill:#e1f5fe
    style D fill:#e8f5e9
    style G fill:#c8e6c9
{{</mermaid>}}

**时间对比**：
- 串行加载：~3-5 秒
- 并行加载：~1-2 秒
- 加速比：~2-3 倍

对于长程任务，虽然启动时间只发生一次，但在频繁重启（如崩溃恢复）的场景下，这个优化显著减少了总耗时。

### 10.3 Stale-While-Error 渲染

Claude Code 的 UI 架构采用了 **Stale-While-Error** 模式：

```
正常渲染：
  组件 A → 成功 → 显示
  组件 B → 成功 → 显示
  组件 C → 成功 → 显示

组件故障时：
  组件 A → 成功 → 显示 (最新)
  组件 B → 失败 → 显示 (旧缓存) ← 关键：不崩溃
  组件 C → 成功 → 显示 (最新)
```

**对长程任务的意义**：即使部分 UI 组件出现渲染错误，Claude Code 的核心功能（终端交互、工具执行）仍然可用。这防止了"一个组件崩溃导致整个会话丢失"的情况。

### 10.4 叶模块模式（Leaf Module Pattern）

Claude Code 的代码组织采用了**叶模块模式**：

```
传统架构（层间依赖）：
  UI 层 → 业务逻辑层 → 数据层
  ↑ 任何一层故障都会级联影响上层

叶模块模式（扁平化）：
  Store ←→ Hooks ←→ Tools ←→ Renderer
  ↑ 每个模块直接与 Store 通信，不依赖其他模块
```

**降低级联故障风险**：
- 模块之间没有直接的调用链
- 所有通信通过 Store 的发布-订阅机制
- 单个模块故障不会影响其他模块
- 符合 [官方 plugins.md](https://code.claude.com/docs/en/plugins.md) 的扩展架构设计

---

## 11. 长程任务最佳实践：从官方文档到社区经验

### 11.1 "探索 → 计划 → 实现 → 提交"四阶段工作流

[官方 best-practices.md](https://code.claude.com/docs/en/best-practices.md) 和 [common-workflows.md](https://code.claude.com/docs/en/common-workflows.md) 推荐以下四阶段工作流：

| 阶段 | 模式 | 操作 | 说明 |
|------|------|------|------|
| **Explore** | Plan Mode | 阅读文件，理解代码结构 | 使用 Explore Subagent 进行代码发现 |
| **Plan** | Plan Mode | 创建详细的实现计划 | 让 Claude 生成逐步的执行计划 |
| **Implement** | Default Mode | 执行计划，运行测试 | 按计划分步执行，每步验证 |
| **Commit** | Default Mode | 提交变更，创建 PR | 使用 `/commit` 自动生成提交信息 |

{{<mermaid>}}
graph LR
    A["Explore: 理解代码结构"] --> B["Plan: 制定实现计划"]
    B --> C["Implement: 分步执行"]
    C --> D["Commit: 提交变更"]

    A -->|"Explore Subagent"| A
    B -->|"Plan Mode"| B
    C -->|"Checkpoint 保护"| C
    D -->|"Code Review"| D

    style A fill:#e1f5fe
    style B fill:#f3e5f5
    style C fill:#e8f5e9
    style D fill:#fff3e0
{{</mermaid>}}

**为什么这个工作流有效**：

1. **Explore 阶段**使用只读 Subagent，不污染主会话上下文
2. **Plan 阶段**在修改代码之前形成清晰的执行计划，减少返工
3. **Implement 阶段**分步执行，每步都有 Checkpoint 保护
4. **Commit 阶段**使用 [官方 code-review.md](https://code.claude.com/docs/en/code-review.md) 的自动化代码审查

### 11.2 给 Claude 验证能力

[官方 best-practices.md](https://code.claude.com/docs/en/best-practices.md) 强调的**最高杠杆原则**是：

> **给 Claude 验证自己工作的能力。**

这包括：
- **提供测试**：让 Claude 运行测试并查看结果
- **提供预期输出**：告诉 Claude 正确的输出应该是什么
- **提供截图/日志**：让 Claude 自主诊断问题

```
错误示范：
  用户："帮我修复这个 bug"
  Claude：修改代码
  用户："不行，还是有问题"（人工验证）

正确示范：
  用户："帮我修复这个 bug。测试用例在 tests/auth.test.ts，运行 `npm test` 验证。"
  Claude：修改代码 → 运行测试 → 测试通过 → 报告结果
```

这种方式可以**减少 50-80% 的人工干预轮次**，直接降低长程任务的 token 消耗。

### 11.3 上下文管理策略

| 策略 | 命令 | 适用场景 | 效果 |
|------|------|---------|------|
| **清空上下文** | `/clear` | 切换到不相关任务 | 节省后续所有 token |
| **压缩上下文** | `/compact` | 保持任务连续性 | 保留核心上下文 |
| **重命名 + 清空** | `/rename` + `/clear` | 长期任务分阶段 | 可追溯 + 低成本 |
| **使用 Subagent** | 自动/`/agent` | 探索性任务 | 30-70% 主会话 token 节省 |
| **MCP 延迟加载** | 配置控制 | 大量工具场景 | 减少初始上下文 |

[官方 statusline.md](https://code.claude.com/docs/en/statusline.md) 描述了如何在终端状态栏中实时监控上下文使用情况，帮助你在上下文耗尽之前采取行动。

### 11.4 VS Code 和 JetBrains 中的长程任务

在 IDE 集成模式下（[vs-code.md](https://code.claude.com/docs/en/vs-code.md)、[jetbrains.md](https://code.claude.com/docs/en/jetbrains.md)），长程任务的稳定性还受到以下因素影响：

| 因素 | 影响 | 建议 |
|------|------|------|
| IDE 崩溃 | 可能导致 Claude Code 会话丢失 | 启用 Session 持久化 |
| 文件同步延迟 | Claude 可能读取过时的文件 | 在修改后等待保存完成 |
| 终端缓冲区限制 | 输出可能被截断 | 使用 PostToolUse Hook 过滤 |

---

## 12. 可迁移设计模式：构建你自己的长程稳定 Agent

### 12.1 14 条可迁移设计模式

从 Claude Code 的架构和实践中，我们可以提炼出 14 条可迁移的设计模式，适用于任何基于 LLM 的 Agent 系统：

| # | 模式 | 一句话描述 | 适用场景 | Claude Code 实现 |
|---|------|-----------|---------|-----------------|
| 1 | **5 级压缩流水线** | 渐进式上下文管理，从轻到重 | 任何 LLM Agent | Compaction (L1-L5) |
| 2 | **上下文隔离 Subagent** | 分而治之，防止上下文污染 | 探索性任务 | Explore Subagent |
| 3 | **错误注入自主修复** | 工具失败后注入错误信息 | 工具执行 Agent | Tool Result 注入 |
| 4 | **Token 预算硬限制** | 防止无限循环和资源浪费 | 所有 Agent | TokenBudget |
| 5 | **双记忆系统** | 用户指令 + 自动学习 | 跨会话 Agent | CLAUDE.md + Auto Memory |
| 6 | **Hook 预处理** | 在 LLM 看到之前做减法 | 大数据处理 | PreToolUse / PostToolUse |
| 7 | **Session 持久化** | JSONL 连续写入，崩溃恢复 | 所有对话式 Agent | JSONL 存储 |
| 8 | **Checkpointing 快照** | 自动快照 + 回滚 | 编程 Agent | Checkpoint 系统 |
| 9 | **Session 分支** | 尝试不同方案 | 探索性开发 | /branch |
| 10 | **Prompt Cache 锁存** | 重复内容降本 | API 成本优化 | Prompt Cache |
| 11 | **Stale-While-Error** | 局部故障不崩溃 | 复杂 UI | 渲染容错 |
| 12 | **叶模块模式** | 减少级联故障 | 大型代码库 | 扁平化架构 |
| 13 | **按需 Skill 加载** | 替代全局指令 | 领域知识注入 | Skills 系统 |
| 14 | **指数退避重试** | API 限流恢复 | 所有网络请求 | Rate Limit 处理 |

### 12.2 Python 最小实现：5 级 Compaction

以下是一个完整的 5 级压缩流水线的最小 Python 实现：

```python
"""
Claude Code 5 级 Compaction 的最小 Python 实现
适用于任何基于 LLM 的 Agent 系统
"""

import time
import math
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class Message:
    content: str
    role: str  # "user" or "assistant"
    timestamp: float = field(default_factory=time.time)
    token_count: int = 0
    is_tool_result: bool = False


class CompactionPipeline:
    def __init__(
        self,
        max_tokens: int = 200_000,
        tool_result_budget: int = 5_000,
        snip_threshold: float = 0.8,
        decay_rate: float = 0.05,
        compact_threshold: float = 0.9,
    ):
        self.max_tokens = max_tokens
        self.tool_result_budget = tool_result_budget
        self.snip_threshold = snip_threshold
        self.decay_rate = decay_rate
        self.compact_threshold = compact_threshold
        self.messages: list[Message] = []
        self.summary: Optional[str] = None

    def add(self, msg: Message) -> list[Message]:
        """添加消息并自动触发必要的压缩"""
        self.messages.append(msg)
        self._run_pipeline()
        return self.messages

    def _run_pipeline(self):
        """按顺序运行 5 级压缩"""
        current_tokens = self._total_tokens()

        # L1: 工具结果预算控制
        for msg in self.messages:
            if msg.is_tool_result and msg.token_count > self.tool_result_budget:
                msg.content = self._truncate(msg.content, self.tool_result_budget)
                msg.token_count = self._estimate_tokens(msg.content)

        # L2: 历史裁剪（当使用率超过 snip_threshold）
        if current_tokens / self.max_tokens > self.snip_threshold:
            self._snip_compact()

        # L3: 微压缩（始终运行）
        self._micro_compact()

        current_tokens = self._total_tokens()

        # L4: 上下文折叠（当使用率仍然很高）
        if current_tokens / self.max_tokens > self.snip_threshold:
            self._context_collapse()

        current_tokens = self._total_tokens()

        # L5: 自动压缩（兜底）
        if current_tokens / self.max_tokens > self.compact_threshold:
            self._auto_compact()

    def _truncate(self, content: str, budget: int) -> str:
        """L1: 截断工具结果"""
        if len(content) <= budget:
            return content
        head = budget // 3
        tail = budget // 3
        marker = f"\n... [{len(content) - head - tail} chars truncated] ...\n"
        return content[:head] + marker + content[-tail:]

    def _snip_compact(self):
        """L2: 裁剪最旧的历史消息"""
        # 保留最近的 50% 消息
        keep_count = max(len(self.messages) // 2, 5)
        removed = self.messages[:-keep_count]
        # 生成被移除消息的摘要
        if self.summary:
            self.summary += "\n" + self._summarize_messages(removed)
        else:
            self.summary = self._summarize_messages(removed)
        self.messages = self.messages[-keep_count:]

    def _micro_compact(self):
        """L3: 缓存感知 + 时间衰减"""
        # 根据时间衰减计算每条消息的保留权重
        now = time.time()
        weights = []
        for msg in self.messages:
            age = now - msg.timestamp
            weight = math.exp(-self.decay_rate * age)
            weights.append((msg, weight))

        # 保留高权重消息，截断低权重消息的内容
        weights.sort(key=lambda x: x[1])
        for msg, weight in weights[:len(weights)//4]:
            if msg.token_count > 100:
                msg.content = msg.content[:200] + "... [truncated]"
                msg.token_count = self._estimate_tokens(msg.content)

    def _context_collapse(self):
        """L4: 折叠冗余上下文"""
        # 合并同一轮次的多条消息
        collapsed = []
        current_role = None
        current_content = []

        for msg in self.messages:
            if msg.role == current_role:
                current_content.append(msg.content)
            else:
                if current_role:
                    collapsed.append(Message(
                        content="\n".join(current_content),
                        role=current_role,
                        timestamp=min(m.timestamp for m in self.messages if m.role == current_role)
                    ))
                current_role = msg.role
                current_content = [msg.content]

        if current_role:
            collapsed.append(Message(
                content="\n".join(current_content),
                role=current_role
            ))

        self.messages = collapsed

    def _auto_compact(self):
        """L5: AI 生成摘要（兜底）"""
        # 在真实系统中，这里会调用 LLM 生成摘要
        # 此处用简单规则模拟
        if self.messages:
            key_points = []
            for msg in self.messages:
                if "important" in msg.content.lower() or "key" in msg.content.lower():
                    key_points.append(msg.content[:100])

            self.summary = (self.summary or "") + "\nKey points: " + "; ".join(key_points)

            # 仅保留最近 3 条消息
            self.messages = self.messages[-3:]

    def _total_tokens(self) -> int:
        return sum(m.token_count for m in self.messages) + (
            self._estimate_tokens(self.summary) if self.summary else 0
        )

    def _estimate_tokens(self, text: str) -> int:
        """粗略的 token 估算（英文: ~4 chars/token, 中文: ~1.5 chars/token）"""
        chinese_chars = sum(1 for c in text if '\u4e00' <= c <= '\u9fff')
        other_chars = len(text) - chinese_chars
        return chinese_chars // 2 + other_chars // 4

    def _summarize_messages(self, messages: list[Message]) -> str:
        """生成消息摘要"""
        return f"[{len(messages)} messages summarized, covering: {messages[0].content[:50]}...]"
```

### 12.3 TypeScript 最小实现：错误恢复降级链

```typescript
/**
 * Claude Code 错误恢复降级链的 TypeScript 实现
 */

type ErrorType =
  | "rate_limit"
  | "token_budget"
  | "tool_failure"
  | "network_timeout"
  | "context_overflow";

interface RecoveryStrategy {
  type: ErrorType;
  action: (error: Error, context: RecoveryContext) => Promise<RecoveryResult>;
  maxRetries: number;
}

interface RecoveryContext {
  currentTurn: number;
  totalTokens: number;
  tokenBudget: number;
  messageHistory: Message[];
}

interface RecoveryResult {
  success: boolean;
  action: "retry" | "compact" | "warn" | "terminate" | "skip";
  message?: string;
}

class ErrorRecoveryChain {
  private strategies: RecoveryStrategy[] = [
    {
      type: "rate_limit",
      maxRetries: 5,
      action: async (error, context) => {
        // 指数退避重试
        const delay = Math.min(1000 * Math.pow(2, context.currentTurn), 30000);
        await new Promise(resolve => setTimeout(resolve, delay));
        return { success: true, action: "retry" };
      }
    },
    {
      type: "token_budget",
      maxRetries: 3,
      action: async (error, context) => {
        const usage = context.totalTokens / context.tokenBudget;
        if (usage < 0.8) return { success: true, action: "retry" };
        if (usage < 0.95) return { success: true, action: "compact" };
        return { success: false, action: "warn", message: "Token budget nearly exhausted" };
      }
    },
    {
      type: "tool_failure",
      maxRetries: 3,
      action: async (error, context) => {
        // 将错误注入上下文，让 LLM 自主修复
        const errorMessage = `Tool execution failed: ${error.message}`;
        context.messageHistory.push({
          role: "system",
          content: errorMessage
        });
        return { success: true, action: "retry" };
      }
    },
    {
      type: "context_overflow",
      maxRetries: 1,
      action: async (error, context) => {
        return { success: true, action: "compact", message: "Forcing autocompact" };
      }
    }
  ];

  async recover(errorType: ErrorType, error: Error, context: RecoveryContext): Promise<RecoveryResult> {
    const strategy = this.strategies.find(s => s.type === errorType);
    if (!strategy) {
      return { success: false, action: "terminate", message: "No recovery strategy" };
    }
    return strategy.action(error, context);
  }
}
```

### 12.4 从 Claude Code 学到的通用原则

1. **上下文是最稀缺的资源** — 一切设计围绕上下文管理。Claude Code 的 9 大稳定机制本质上都是上下文管理的不同侧面。

2. **零信任** — 每个工具调用都经过多层检查（权限 → Hook → 执行 → 结果过滤 → 注入）。不信任任何外部输入。

3. **循环直到完成** — 不是请求-响应模式，而是迭代执行模式。Claude 会持续尝试直到任务完成或达到预算上限。

4. **可扩展性决定上限** — Skill/Plugin/MCP 让能力无限延伸，而不需要在每次会话中加载所有指令。

5. **分而治之** — Subagent 隔离上下文爆炸。这是 Claude Code 长程稳定性的最关键设计决策之一。

---

## 13. 总结与展望

### 13.1 Claude Code 长程稳定性的核心可借鉴点

通过本文对 Claude Code 长程任务稳定性的深度剖析，我们可以总结出以下核心可借鉴点：

| 可借鉴点 | 核心思想 | 迁移难度 | 影响力 |
|---------|---------|---------|--------|
| **5 级压缩流水线** | 渐进式、从轻到重的上下文管理 | 中 | 极高 |
| **上下文隔离** | 探索性任务在独立窗口中执行 | 中 | 极高 |
| **错误注入修复** | 让 LLM 自主处理工具错误 | 低 | 高 |
| **双记忆系统** | 用户指令与自动学习分离 | 低 | 高 |
| **Hook 预处理** | 在 LLM 看到数据之前做减法 | 低 | 中 |
| **JSONL 持久化** | 简单、可靠、可恢复的状态存储 | 低 | 中 |

### 13.2 通用长程 Agent 设计原则

基于 Claude Code 的架构，我们提出以下通用长程 Agent 设计原则：

1. **上下文预算管理** — 显式跟踪和控制 token 消耗，设置硬性上限
2. **渐进式退化** — 资源不足时优雅降级，而非突然崩溃
3. **状态可恢复** — 所有状态可持久化，支持断点续传
4. **安全网机制** — 自动快照和回滚能力
5. **可观测性** — 实时监控上下文使用率、成本、错误率
6. **模块化扩展** — 按需加载能力，不全局注入

### 13.3 未来趋势：更长上下文、更智能压缩、更稳定编排

Claude Code 的长程稳定性机制代表了当前 LLM Agent 工程的最佳实践。未来，随着以下技术的发展，长程任务的处理能力将进一步提升：

- **更长上下文窗口** — 从 200K 到 1M+ tokens，减少 Compaction 的频率和信息损失
- **更智能压缩** — 基于语义理解的压缩，而非简单的截断和摘要
- **更稳定编排** — 多 Agent 协作的容错和自愈机制
- **更精确成本预测** — 基于任务复杂度的预先成本估算

### 13.4 国产编程工具的启示

Claude Code 的长程稳定性设计对国产编程 AI 编程工具具有以下启示：

1. **必须实现上下文压缩机制** — 长程任务是不可避免的场景，没有压缩就没有稳定性
2. **Session 持久化是基础** — 崩溃恢复是用户对工具信任的前提
3. **Subagent 隔离是趋势** — 随着任务复杂度增加，上下文隔离将成为标配
4. **Hooks 是可扩展性的关键** — 没有 Hook 机制，工具就无法适应不同的工作流
5. **成本透明度不可少** — 用户需要知道长程任务的 token 消耗和成本

---

## 附录

### A. 五大参考源对应章节索引

| 参考源 | 本文引用章节 | 关键贡献 |
|--------|------------|---------|
| [Claude Code 官方文档](https://code.claude.com/docs/) | Ch2-Ch12 | 所有机制的官方规范 |
| [oboard/claude-code-rev](https://github.com/oboard/claude-code-rev) | Ch2.3, Ch6.2, Ch10.1 | 12 步状态机、compaction 源码分析 |
| [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) | Ch2.3, Ch7.4 | 5 级 Compaction、上下文衰减模型 |
| [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) | Ch3, Ch11 | 长程任务最佳实践、上下文管理策略 |
| [openedclaude/claude-reviews-claude](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview) | Ch6, Ch10, Ch12 | 工程架构分析、可迁移设计模式 |

### B. 关键术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 上下文窗口 | Context Window | LLM 单次处理的最大 token 数量 |
| Compaction | Compaction | 上下文压缩，减少上下文占用的 token 数 |
| Session | Session | 一次完整的对话会话，具有持久化能力 |
| Subagent | Subagent | 在独立上下文中运行的子代理 |
| Checkpointing | Checkpointing | 自动创建代码和对话快照 |
| Token Budget | Token Budget | 单次任务的最大 token 消耗限制 |
| Prompt Cache | Prompt Cache | 缓存重复的 prompt 内容以降低 API 成本 |
| CLAUDE.md | CLAUDE.md | Claude Code 的配置文件，用作跨会话记忆 |
| Hooks | Hooks | 在工具执行前后拦截和修改的事件处理器 |
| Skills | Skills | 按需加载的领域知识和操作指南 |
| 熵增 | Entropy Increase | 上下文中有效信息密度随时间降低的现象 |
| Stale-While-Error | Stale-While-Error | 组件故障时显示旧数据而非崩溃的模式 |

### C. 官方文档相关链接

#### 核心机制

- [Agent Loop (SDK)](https://code.claude.com/docs/en/agent-sdk/agent-loop.md)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing.md)
- [Sessions](https://code.claude.com/docs/en/sessions.md)
- [Sessions (SDK)](https://code.claude.com/docs/en/agent-sdk/sessions.md)
- [Memory](https://code.claude.com/docs/en/memory.md)
- [Sub-agents](https://code.claude.com/docs/en/sub-agents.md)
- [Subagents (SDK)](https://code.claude.com/docs/en/agent-sdk/subagents.md)
- [Context Window](https://code.claude.com/docs/en/context-window.md)
- [Compaction](https://code.claude.com/docs/en/compaction.md)
- [Costs](https://code.claude.com/docs/en/costs.md)

#### 扩展与集成

- [Hooks](https://code.claude.com/docs/en/hooks.md)
- [Hooks (SDK)](https://code.claude.com/docs/en/agent-sdk/hooks.md)
- [Skills](https://code.claude.com/docs/en/skills.md)
- [Skills (SDK)](https://code.claude.com/docs/en/agent-sdk/skills.md)
- [Plugins](https://code.claude.com/docs/en/plugins.md)
- [MCP](https://code.claude.com/docs/en/mcp.md)

#### 安全与权限

- [Permissions](https://code.claude.com/docs/en/permissions.md)
- [Permission Modes](https://code.claude.com/docs/en/permission-modes.md)

#### 多 Agent 编排

- [Agent Teams](https://code.claude.com/docs/en/agent-teams.md)
- [Agent View](https://code.claude.com/docs/en/agent-view.md)
- [Agents (Parallel)](https://code.claude.com/docs/en/agents.md)

#### 最佳实践与工作流

- [Best Practices](https://code.claude.com/docs/en/best-practices.md)
- [Common Workflows](https://code.claude.com/docs/en/common-workflows.md)
- [Code Review](https://code.claude.com/docs/en/code-review.md)
- [Status Line](https://code.claude.com/docs/en/statusline.md)
- [Todo Tracking (SDK)](https://code.claude.com/docs/en/agent-sdk/todo-tracking.md)

#### 工程与运维

- [Claude Directory](https://code.claude.com/docs/en/claude-directory.md)
- [Settings](https://code.claude.com/docs/en/settings.md)
- [Environment Variables](https://code.claude.com/docs/en/env-vars.md)
- [CLI Reference](https://code.claude.com/docs/en/cli-reference.md)
- [Analytics](https://code.claude.com/docs/en/analytics.md)
- [Structured Outputs (SDK)](https://code.claude.com/docs/en/agent-sdk/structured-outputs.md)
- [File Checkpointing (SDK)](https://code.claude.com/docs/en/agent-sdk/file-checkpointing.md)

#### Slash Commands (SDK)

- [Slash Commands (SDK)](https://code.claude.com/docs/en/agent-sdk/slash-commands.md)
