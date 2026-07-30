
---
title: Cloud Code Memory 
weight: 2
---

# Claude Code Memory 机制详解

> **摘要**：Claude Code 是 Anthropic 推出的 Agentic 编程工具。与所有基于 LLM 的工具一样，它面临一个根本挑战——每次会话从空白的 Context Window 开始。Claude Code 通过一套精心设计的双轨记忆系统（CLAUDE.md + Auto Memory）配合路径级规则（.claude/rules/）来解决跨会话知识传递问题。本文将从官方文档出发，结合 VILA-Lab 的源码级架构分析、OpenClaude 的 16 篇深度解析、以及逆向源码项目 claude-code-rev，系统拆解 Claude Code 的记忆机制。

---

## 1. 引言

### 1.1 为什么 AI 编程工具需要"记忆"

所有基于大语言模型（LLM）的编程工具都面临同一个架构难题：**模型本身是无状态的**。每次 API 调用都是一次全新的对话，模型不记得上一轮交互中你告诉它的构建命令、代码规范、或者刚刚纠正过的错误。

对于 Chat 场景，这或许可以接受——用户可以在同一轮对话中逐步构建上下文。但对于编程工具，尤其是需要跨多天、跨多次会话使用的工具，"失忆"是致命的用户体验。开发者不可能每次启动工具都重新解释一遍项目架构。

因此，**记忆系统不是锦上添花，而是 Agentic 编程工具的基础设施**。

### 1.2 Agent 范式：Model 是大脑，Harness 是身体

shareAI-lab 在 [learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 中提出了一个清晰的 Agent 范式：

```
Agency（感知、推理、行动的能力）来自模型训练，而非外部代码编排。
但一个可用的 Agent 产品需要两样东西：Model + Harness。

Model = 大脑（智能来源）
Harness = 身体（感知、行动、记忆、约束）
```

Harness 的公式可以写为：

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions
```

其中 **Memory（记忆）属于 Knowledge 层**——它负责让 Agent 在每次启动时拥有关于项目、用户偏好和历史经验的背景知识。Claude Code 的记忆系统，本质上就是 Harness 工程中 Knowledge 管理的具体实现。

### 1.3 Claude Code 的记忆设计哲学

Claude Code 的记忆设计有几个核心特点：

1. **文件即持久化**：所有记忆都以 Markdown 或 JSON 文件存储在文件系统中，用户可读、可编辑、可版本控制
2. **上下文即记忆**：记忆内容在会话启动时注入 Context Window，作为 System Prompt 的一部分被模型读取
3. **分层作用域**：从组织级到项目级再到本地级，不同作用域的记忆在不同粒度上生效
4. **按需加载**：不是所有记忆都在启动时加载，部分记忆只在访问特定路径时才触发

### 1.4 本文覆盖范围

本文聚焦 Claude Code 的记忆机制，涵盖：

- CLAUDE.md 文件系统的完整工作机制
- Auto Memory 自动学习机制
- .claude/rules/ 路径级规则
- Context Window 中的记忆加载时序与 Token 消耗
- Monorepo 场景下的记忆管理策略
- Subagent 的独立记忆机制
- 源码层面的实现分析
- Harness 工程视角的记忆设计启示

---

## 2. Claude Code 记忆体系总览

### 2.1 核心问题：每次会话从空白 Context Window 开始

Claude Code 的每次会话（Session）都拥有独立的 Context Window。当你在终端中运行 `claude` 时，模型对你的项目一无所知——它不知道构建命令是什么、不知道代码规范、甚至不知道这是一个 Git 仓库还是裸目录。

这与 ChatGPT 或 Claude.ai 的网页版不同：在网页版中，你可以在同一轮对话中持续积累上下文。而 Claude Code 的每次启动都是一次"冷启动"。

### 2.2 两大记忆机制

为了解决这个问题，Claude Code 设计了两套互补的记忆系统：

| 维度 | CLAUDE.md 文件 | Auto Memory |
|------|---------------|-------------|
| **谁写的** | 你（开发者） | Claude（AI 自身） |
| **写什么** | 指令和规则 | 学习到的经验和模式 |
| **作用域** | 项目 / 用户 / 组织 | 每个仓库（跨 worktree 共享） |
| **加载方式** | 每次会话全量加载 | 前 200 行或 25KB |
| **适用场景** | 编码标准、工作流、项目架构 | 构建命令、调试经验、用户偏好 |

> 来源：[Claude Code 官方文档 — Memory](https://code.claude.com/docs/en/memory)

两者互补而非替代：CLAUDE.md 用于你主动告诉 Claude "应该怎么做"，Auto Memory 用于 Claude 自己发现"什么方式效果更好"。

### 2.3 记忆的本质：是 Context 而非强制配置

这是理解 Claude Code 记忆系统最关键的一点：

> **Claude 将记忆内容视为上下文（context），而非强制执行的配置（enforced configuration）。**

这意味着：

- 记忆内容是**建议性的**，Claude 会参考它们，但不保证每次都严格遵守
- 如果你需要**强制阻止**某个操作（无论 Claude 怎么想），应该使用 [PreToolUse Hook](https://code.claude.com/docs/en/hooks-guide) 而非写入记忆文件
- 指令越**具体、简洁、结构化**，Claude 遵循的可靠性越高

### 2.4 记忆系统的三层架构

综合官方文档和 OpenClaude 的架构解析，Claude Code 的记忆系统可以分为三层：

```
┌─────────────────────────────────────────────┐
│  第一层：持久指令层（CLAUDE.md 家族）         │
│  开发者主动编写，每次会话启动时加载            │
│  作用域：Managed → User → Project → Local    │
├─────────────────────────────────────────────┤
│  第二层：自动学习层（Auto Memory / MEMORY.md） │
│  Claude 自动积累，每次会话加载前 200 行/25KB  │
│  来源：用户纠正、构建命令发现、调试经验        │
├─────────────────────────────────────────────┤
│  第三层：路径规则层（.claude/rules/）          │
│  按需触发，仅当 Claude 访问匹配的文件路径时加载 │
│  解决 CLAUDE.md 膨胀问题                     │
└─────────────────────────────────────────────┘
```

---

## 3. CLAUDE.md：持久指令系统

### 3.1 什么是 CLAUDE.md

CLAUDE.md 是 Claude Code 的**持久指令文件**。它是一个纯文本 Markdown 文件，Claude 在每次会话启动时读取它，从而获得关于项目、用户偏好或组织策略的持久上下文。

你可以把它理解为 Claude Code 的"入职手册"——新成员（每次启动的 Claude）通过阅读这份手册快速了解项目的全貌和规则。

### 3.2 四种作用域与加载优先级

CLAUDE.md 文件可以存放在不同位置，每个位置对应不同的作用域。官方文档按加载优先级（从最宽到最窄）列出了四个层级：

| 作用域 | 位置 | 目的 | 共享范围 |
|--------|------|------|----------|
| **Managed Policy** | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`<br>Linux/WSL: `/etc/claude-code/CLAUDE.md`<br>Windows: `C:\Program Files\ClaudeCode\CLAUDE.md` | 组织级指令（IT/DevOps 管理） | 组织内所有用户 |
| **User Instructions** | `~/.claude/CLAUDE.md` | 个人偏好，适用于所有项目 | 仅你自己 |
| **Project Instructions** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 项目级指令，团队共享 | 团队成员（通过版本控制） |
| **Local Instructions** | `./CLAUDE.local.md` | 个人项目级偏好，应加入 `.gitignore` | 仅你自己（当前项目） |

**加载顺序**决定了上下文中内容的排列：宽作用域的指令先加载，窄作用域的后加载。这意味着项目级指令出现在用户级指令之后，更接近工作目录的指令后加载——这样 Claude 在上下文窗口中"最后看到"的（通常是最近相关的）指令具有更高的注意力权重。

### 3.3 CLAUDE.md 的加载机制

#### 目录树扫描

VILA-Lab 在 [Dive into Claude Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) 的源码分析中指出，CLAUDE.md 的加载机制是 Claude Code 启动引导（Bootstrap）流程中的关键一环：

> Claude Code 从当前工作目录开始，**逐级向上扫描目录树**，检查每个目录中是否存在 `CLAUDE.md` 和 `CLAUDE.local.md` 文件。

以在 `foo/bar/` 目录下启动为例：

```
加载顺序（从文件系统根目录到工作目录）：
  /CLAUDE.md          ← 如果存在
  /foo/CLAUDE.md      ← 如果存在
  /foo/bar/CLAUDE.md  ← 如果存在

  同级目录内：
  CLAUDE.md → CLAUDE.local.md（后加载）
```

#### 内容拼接而非覆盖

所有发现的 CLAUDE.md 文件内容是**拼接（concatenate）**在一起的，而不是互相覆盖。这意味着：

- 父目录的指令不会被子目录的指令完全替换
- 两者同时存在于 Context Window 中
- 如果出现规则冲突，Claude 可能任意选择遵循哪一条

#### 子目录文件的按需加载

除了目录树向上扫描，Claude Code 还会发现**子目录**下的 CLAUDE.md 文件。但这些文件**不会在启动时加载**，而是在 Claude 首次读取该子目录中的文件时才加载。这一设计避免了大型代码库中大量无关指令被提前加载到 Context Window 中。

#### HTML 注释的 Token 优化技巧

Claude Code 在注入 Context Window 之前，会**剥离 CLAUDE.md 中的块级 HTML 注释**（`<!-- ... -->`）。这意味着：

- 你可以在 CLAUDE.md 中用 HTML 注释给人看备注（比如"这个规则是因为 2024 年 Q3 的 XX 事件加的"）
- 这些注释**不消耗 Context Window 的 Token**
- 代码块内的注释会被保留

### 3.4 如何编写高效的 CLAUDE.md

官方文档给出了四条核心原则：

#### 大小控制

**目标：每个 CLAUDE.md 文件控制在 200 行以内。**

原因：CLAUDE.md 文件在每次会话启动时都会加载到 Context Window 中，占用 Token 预算。文件越长，Claude 遵循指令的一致性越低（上下文稀释效应）。

如果你的 CLAUDE.md 越来越长，应考虑：

- 将多步骤流程移到 [Skill](https://code.claude.com/docs/en/skills) 中
- 将特定路径的规则移到 `.claude/rules/` 中
- 使用 `@import` 拆分内容

#### 结构

使用 Markdown 标题和列表来组织指令。Claude 扫描结构的方式与人类读者相同：有组织的段落比密集的文字更容易遵循。

#### 具体性

指令应该具体到可以验证：

```
✅ "使用 2 空格缩进"
❌ "格式化代码"

✅ "提交前运行 npm test"
❌ "测试你的代码"

✅ "API handler 放在 src/api/handlers/"
❌ "保持文件组织有序"
```

#### 一致性

定期检查 CLAUDE.md、子目录中的嵌套 CLAUDE.md、以及 `.claude/rules/` 中的规则文件，移除过时的或冲突的指令。

### 3.5 @import 语法与文件引用

CLAUDE.md 支持使用 `@path/to/import` 语法引入额外文件：

```markdown
参见 @README 获取项目概览，@package.json 获取可用的 npm 命令。

# 额外指令
- git 工作流 @docs/git-instructions.md
```

- 支持**相对路径**（相对于包含 import 的文件）和**绝对路径**
- 支持**递归导入**，最大深度为 4 层
- 被导入的文件在启动时会被展开并加载到上下文中

与 AGENTS.md 的兼容方案：

```markdown
# CLAUDE.md
@AGENTS.md

## Claude Code 特有指令
- 对 src/billing/ 的变更使用 plan mode
```

或者直接创建 symlink：

```bash
ln -s AGENTS.md CLAUDE.md
```

### 3.6 /init 命令：自动生成 CLAUDE.md

Claude Code 提供了 `/init` 命令来自动生成 CLAUDE.md：

- Claude 会分析你的代码库，自动发现构建命令、测试指令和项目约定
- 如果 CLAUDE.md 已存在，`/init` 会提出改进建议而非覆盖
- 设置环境变量 `CLAUDE_CODE_NEW_INIT=1` 可启用交互式多阶段工作流，Claude 会询问要设置哪些 artifact（CLAUDE.md 文件、Skills、Hooks），然后通过子代理探索代码库，在写入任何文件之前展示可审查的提案

### 3.7 CLAUDE.md 在 System Prompt 中的位置

根据 OpenClaude 的架构解析，CLAUDE.md 在 System Prompt 分层构建中的位置如下：

```
System Prompt 构建顺序（按 Token 优先级排序）：
  1. 核心身份与规则    ← 我是谁，能/不能做什么
  2. 工具描述（42+）   ← ~10K+ tokens 的能力定义
  3. Git 状态与项目上下文 ← 当前分支、最近提交
  4. CLAUDE.md 内容    ← 项目级指令
  5. 用户/企业规则      ← 偏好、组织策略
  6. 动态附件          ← 技能发现、记忆注入、MCP 资源
```

CLAUDE.md 处于中间位置，意味着它既在核心规则之后（不会与身份冲突），又在用户提示之前（作为背景知识影响 Claude 的理解）。

---

## 4. Auto Memory：AI 自动学习机制

### 4.1 什么是 Auto Memory

Auto Memory 是 Claude Code 的**自动学习系统**。与 CLAUDE.md 由开发者手动编写不同，Auto Memory 由 Claude 自己在交互过程中自动积累——它基于用户的纠正、偏好发现和调试经验来编写笔记。

### 4.2 工作原理

Claude 在以下场景中会积累 Auto Memory：

- **用户纠正**：当 Claude 犯了错误，用户纠正了它，Claude 会记下这个教训
- **偏好发现**：当 Claude 注意到用户反复以某种方式操作（比如偏好某种测试框架），它会记录下来
- **构建命令发现**：当 Claude 通过分析代码库发现了构建命令或测试指令，它会自动记录
- **调试经验**：当 Claude 成功解决了一个问题，它会记下解决路径

这些笔记被写入仓库级存储（共享于同一仓库的不同 worktree 之间），并在下次会话启动时加载。

### 4.3 加载限制

Auto Memory 的加载有严格限制：

> **每次会话加载前 200 行或 25KB（取先达到的那个）**

这个设计非常关键：

1. 防止 Auto Memory 无限制膨胀导致 Context Window 被占满
2. 迫使 Claude 优先积累最重要的信息（较新的笔记会排在前面）
3. 200 行的限制确保了即使是超大 Auto Memory 也不会对 Token 预算造成灾难性影响

### 4.4 适用场景

| 场景 | 推荐方式 |
|------|----------|
| 编码标准、项目架构 | CLAUDE.md（手动编写，确保准确性） |
| 构建命令发现 | Auto Memory（Claude 自动发现更准确） |
| 调试经验积累 | Auto Memory（Claude 自己记录的调试路径更有用） |
| 个人偏好捕捉 | 两者皆可，但 Auto Memory 更自然 |
| 组织级安全策略 | CLAUDE.md（Managed Policy 作用域） |

### 4.5 Auto Memory vs CLAUDE.md：选择策略

```
什么时候用 CLAUDE.md？
  → 你想主动告诉 Claude 规则
  → 规则需要团队成员共享
  → 内容是关于"应该怎么做"的

什么时候让 Auto Memory 自动学习？
  → 你想让 Claude 从你的行为中学习
  → 内容是关于"什么方式效果更好"的
  → 你不想花时间手动维护笔记
```

### 4.6 /memory 命令

Claude Code 提供了 `/memory` 命令来手动管理 Auto Memory：

- 查看当前 Auto Memory 的内容
- 手动添加或修改记忆条目
- 清理过时或错误的记忆

这是连接手动指令和自动学习的桥梁——你可以审查 Claude 自动积累的内容，并在必要时进行干预。

---

## 5. .claude/rules/：路径级规则系统

### 5.1 为什么需要路径级规则

当项目规模增长时，一个根目录的 CLAUDE.md 文件往往会面临两种困境：

1. **膨胀**：为了覆盖每个子系统的约定，文件变得越来越大，消耗大量 Token
2. **泛化**：为了保持文件精简，规则写得太笼统，无法真正指导 Claude 的行为

`.claude/rules/` 的设计解决了这个问题——它允许将规则**按文件路径范围**进行作用域划分，只有当 Claude 访问匹配的文件时才加载对应规则。

### 5.2 规则文件的结构与语法

规则文件是存放在 `.claude/rules/` 目录下的 Markdown 文件。每个规则文件通过 `paths` 模式定义其适用范围：

```yaml
---
paths: "src/api/**"
---

# API 层代码规范
- 所有路由文件必须导出 Express router
- 禁止在路由 handler 中编写原始 SQL 字符串
- 数据库查询统一使用 Knex
```

### 5.3 触发机制

规则文件的加载是**事件驱动**的：

1. Claude 在会话中首次读取某个文件
2. 系统检查该文件路径是否匹配 `.claude/rules/` 中任何规则的 `paths` 模式
3. 如果匹配，将该规则文件的内容注入到 Context Window 中
4. 在终端中显示 "Loaded .claude/rules/xxx.md" 的提示（但不显示规则内容）

这意味着路径级规则的 Token 消耗只在相关代码被访问时才发生，实现了真正的"按需加载"。

### 5.4 典型用例

- **API 层规范**：`paths: "src/api/**"` 匹配所有 API 相关文件
- **测试文件约定**：`paths: "*.test.ts"` 匹配所有测试文件
- **数据库层规范**：`paths: "src/db/**"` 匹配所有数据库相关文件

### 5.5 与 CLAUDE.md 的配合策略

```
CLAUDE.md（根目录）          ← 全局规则：编码标准、项目结构、通用工作流
.claude/rules/api.md         ← 仅当访问 API 文件时加载
.claude/rules/testing.md     ← 仅当访问测试文件时加载
.claude/rules/database.md    ← 仅当访问数据库文件时加载
```

这种分层策略确保了：Claude 始终看到全局规则，只在需要时加载特定规则，最大限度地节省 Token 预算。

---

## 6. Context Window 中的记忆加载

### 6.1 Context Window 启动流程时序

Claude Code 官方文档提供了一个交互式 Context Window 模拟器，揭示了会话启动时各组件的加载时序和 Token 消耗估算：

| 加载阶段 | 内容 | 估算 Token | 可见性 |
|----------|------|-----------|--------|
| System Prompt | 核心指令、工具使用、响应格式 | ~4,200 | 隐藏 |
| Auto Memory | Claude 的笔记（前 200 行/25KB） | ~680 | 隐藏 |
| 环境信息 | 工作目录、平台、Shell、Git 状态 | ~280 | 隐藏 |
| MCP 工具 | MCP 工具名称列表（Schema 延迟加载） | ~120 | 隐藏 |
| Skill 描述 | 可用 Skill 的单行描述 | ~450 | 隐藏 |
| User CLAUDE.md | `~/.claude/CLAUDE.md` | ~320 | 隐藏 |
| Project CLAUDE.md | 项目级 CLAUDE.md | ~1,800 | 隐藏 |
| **用户提示** | 你的输入 | 视内容而定 | 可见 |

启动阶段总计约 **~7,850 tokens** 被预加载到 Context Window 中。

### 6.2 各阶段 Token 消耗估算

以一个典型的 Web 项目会话为例，Claude 在解决一个问题过程中的完整 Token 消耗流：

```
启动阶段：~7,850 tokens（固定开销）
  ↓
文件读取：Read auth.ts → ~2,400 tokens
文件读取：Read tokens.ts → ~1,100 tokens
路径规则：api-conventions.md → ~380 tokens
文件读取：Read middleware.ts → ~1,800 tokens
文件读取：Read auth.test.ts → ~1,600 tokens
路径规则：testing.md → ~290 tokens
Grep 结果："refreshToken" → ~600 tokens
  ↓
Claude 分析：~800 tokens
代码编辑：~400 tokens
测试输出：~1,200 tokens
  ↓
总消耗：~18,420 tokens
```

在这个例子中，**文件读取占据了最大的 Token 开销**（约 7,500 tokens），启动阶段的记忆相关开销约 7,850 tokens。两者相加，**在 Claude 开始实际工作之前，已经有约 15,700 tokens 被消耗掉了**。

### 6.3 五层压缩系统对记忆的影响

Claude Code 在每次模型调用之前，会运行**五层压缩（compaction）**管线（按成本从低到高排序）：

```
Layer 1: Budget Reduction    ← 最便宜的压缩策略
Layer 2: Snip                ← 截断最旧的对话轮次
Layer 3: Microcompact        ← 清理过期的工具输出（时间衰减）
Layer 4: Context Collapse    ← 折叠冗余上下文
Layer 5: Auto-Compact        ← AI 生成对话摘要（带电路断路器）
```

在这个压缩管线中：

- **启动阶段加载的记忆内容**（System Prompt、CLAUDE.md、Auto Memory）在 `/compact` 之后会**被重新注入**
- **用户实际写入的记忆**（CLAUDE.md 文件）不会被压缩系统修改
- **Auto Memory 的加载限制**（200 行/25KB）本身就是一种压缩策略

### 6.4 /compact 命令后的记忆保留策略

当用户手动运行 `/compact` 时：

- System Prompt、CLAUDE.md、Auto Memory 等启动阶段内容**不受影响**，会在压缩后重新注入
- Skill 描述列表**不会被重新注入**（只有实际调用的 Skill 会被保留）
- 对话历史被压缩为 AI 生成的摘要

### 6.5 记忆对 Token 预算的影响与优化

对于 200K Context Window（较老模型）或 1M Context Window（Claude 4.6 系列），记忆开销的占比不同：

| Context 大小 | 启动记忆开销 | 占比 |
|-------------|-------------|------|
| 200,000 | ~7,850 | ~3.9% |
| 1,000,000 | ~7,850 | ~0.8% |

在 200K 窗口中，记忆开销占比较明显。优化策略：

1. 将 CLAUDE.md 控制在 200 行以内
2. 使用 `.claude/rules/` 替代部分 CLAUDE.md 内容
3. 在大型项目中利用子目录 CLAUDE.md 的按需加载特性
4. 使用 `claudeMdExcludes` 排除无关包的 CLAUDE.md

---

## 7. 大型代码库与 Monorepo 中的记忆管理

### 7.1 问题：单体 CLAUDE.md 的膨胀

在拥有百万行代码的大型代码库或多包 Monorepo 中，一个根目录的 CLAUDE.md 文件会面临严重问题：

- 为了覆盖每个包的约定，文件迅速膨胀
- 大量与当前任务无关的指令被加载到 Context Window 中
- Token 浪费导致 Claude 性能下降

### 7.2 按目录分层 CLAUDE.md

Claude Code 的推荐做法是**按目录分层**：

```
monorepo/
  CLAUDE.md                     ← 仓库级：编码标准、提交约定、项目布局
  packages/
    api/
      CLAUDE.md                 ← API 包：Node.js/Express/PostgreSQL 规范
      src/
    web/
      CLAUDE.md                 ← 前端包：React/Vite/TailwindCSS 规范
      src/
    shared/
      CLAUDE.md                 ← 共享包：TypeScript 工具库规范
      src/
```

当你从 `packages/api/` 启动 Claude 时，它加载：

- `packages/api/CLAUDE.md`（API 专属指令）
- 根目录 `CLAUDE.md`（仓库级规则）
- **不加载** `packages/web/CLAUDE.md`（前端指令不在上下文中）

### 7.3 claudeMdExcludes 排除无关包

对于大型 Monorepo，Claude Code 提供了 `claudeMdExcludes` 配置项来排除不相关团队的 CLAUDE.md 文件：

```json
// .claude/settings.json
{
  "claudeMdExcludes": [
    "packages/unrelated-team/**"
  ]
}
```

这确保了你永远不会因为目录树扫描而加载无关团队的指令。

### 7.4 多 Worktree 场景的记忆共享

当你在多个 Git Worktree 中工作时：

- `CLAUDE.local.md` 只存在于创建它的那个 Worktree 中
- 要在所有 Worktree 中共享个人指令，可以导入一个来自 Home 目录的文件：

```markdown
# 个人偏好
- @~/.claude/my-project-instructions.md
```

Auto Memory 是**跨 Worktree 共享**的——它存储在仓库级别，而非 Worktree 级别。

### 7.5 记忆系统的可扩展性：Plugin 化

当分层 CLAUDE.md 也无法满足规模需求时，Claude Code 提供了 Plugin 系统作为终极解决方案：

- 将通用约定打包为内部 Plugin
- 团队成员通过内部 Marketplace 安装
- 每个 Package 可以拥有独立的 Skills、Hooks、和 MCP 服务器
- Plugin 的指令不占用 CLAUDE.md 的 Context 空间

---

## 8. Subagent 的记忆机制

### 8.1 Subagent 的独立 Context Window

Claude Code 的 Subagent（子代理）运行在**独立的 Context Window** 中。这意味着：

- 每个 Subagent 拥有独立的对话历史
- 主会话的对话内容**不会**传递给 Subagent
- Subagent 可以路由到不同的模型（如 Haiku 用于快速探索）

### 8.2 子代理的启动加载

根据官方文档，Subagent 启动时加载的内容与主会话有所不同：

| 内容 | 主会话 | Subagent |
|------|--------|----------|
| System Prompt | 完整版（~4,200 tokens） | 更短的版本 |
| CLAUDE.md | 加载 | 加载（自身副本） |
| Auto Memory | 加载（主会话的） | **不继承**主会话的 Auto Memory |
| MCP 工具 | 加载 | 加载（相同设置） |
| Skill 描述 | 加载 | 加载 |

**关键区别**：Subagent **不继承**主会话的 Auto Memory，而是从零开始。但如果自定义 Subagent 配置了 `memory` 前置元数据，它会加载自己独立的 MEMORY.md。

### 8.3 持久记忆目录（agent-memory/）

Claude Code 允许为 Subagent 配置持久记忆：

- 选择 **User Scope**：Subagent 获得一个 `~/.claude/agent-memory/` 目录
- Subagent 可以在跨会话中积累洞察（如代码库模式、重复出现的问题）
- 选择 **None**：Subagent 不持久化任何学习成果

这对于**代码审查 Agent**等场景特别有用——Agent 可以在多次使用中积累关于项目代码模式的知识。

### 8.4 主会话与 Subagent 记忆的隔离

```
主会话
  ├── CLAUDE.md（项目级）
  ├── Auto Memory（主会话的）
  └── 对话历史

Subagent（独立）
  ├── CLAUDE.md（相同的副本）
  ├── Auto Memory（无，或自定义的）
  └── 独立对话历史
```

这种隔离确保了 Subagent 的探索不会污染主会话的上下文，是"Context 管理"的最佳实践。

### 8.5 典型场景：Explore / Plan 子代理

Claude Code 内置的 Explore 和 Plan 子代理有一个特别的设计：

> **Explore 和 Plan 会跳过你的 CLAUDE.md 文件和父会话的 Git 状态**，以保持研究的快速和廉价。

这意味着：

- 当你需要快速搜索代码库时，Explore Agent 不会加载 CLAUDE.md 的指令开销
- 研究速度更快、Token 消耗更低
- 代价是 Explore 不了解项目的自定义约定

---

## 9. 源码视角：记忆系统的实现

### 9.1 记忆模块的源码结构

基于 [claude-code-rev](https://github.com/oboard/claude-code-rev) 逆向还原的源码树和 [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) 的架构分析（v2.1.88，~1,884 个文件，~512K 行代码），Claude Code 的记忆相关代码分布在以下模块中：

- **启动引导模块**（`bootstrap/`）：负责启动时的 CLAUDE.md 扫描和加载
- **上下文组装模块**（`context/`）：负责将记忆内容注入 Context Window
- **压缩模块**（`compact/`）：负责五层压缩管线对记忆内容的处理
- **会话持久化模块**（`session/`）：负责 JSONL 追加存储和 Auto Memory 的读写

### 9.2 CLAUDE.md 解析与加载流程

根据 OpenClaude 的 [EP16: 基础设施与配置](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/16-infrastructure-config) 分析，CLAUDE.md 的加载流程如下：

```
1. 目录树扫描
   └── 从工作目录开始，逐级向上到文件系统根目录
   └── 检查每个目录中的 CLAUDE.md 和 CLAUDE.local.md

2. 文件读取
   └── 按扫描顺序读取所有发现的文件

3. HTML 注释剥离
   └── 使用正则表达式移除块级 HTML 注释
   └── 保留代码块内的注释

4. Context 注入
   └── 拼接所有文件内容
   └── 在 System Prompt 构建的第 4 阶段注入
```

### 9.3 Auto Memory 的存储与加载

Auto Memory 以文件形式存储在仓库级别（`<repo>/.claude/MEMORY.md` 或类似路径）。加载时的截断逻辑：

```typescript
// 伪代码：Auto Memory 加载截断
function loadAutoMemory(path: string): string {
  const content = readFileSync(path, 'utf-8');
  const lines = content.split('\n');

  // 前 200 行或 25KB 限制
  let result = '';
  let lineCount = 0;
  for (const line of lines) {
    if (lineCount >= 200 || result.length >= 25 * 1024) {
      break;
    }
    result += line + '\n';
    lineCount++;
  }
  return result;
}
```

### 9.4 Context 压缩系统对记忆的处理

VILA-Lab 论文中详细分析了 5 层压缩管线：

| 层级 | 策略 | 对记忆的影响 |
|------|------|-------------|
| Layer 0 | API 侧自动清除 | 不触及启动记忆 |
| Layer 1 | 微压缩（时间衰减清理旧结果） | 清理过期的工具输出 |
| Layer 2 | 截断（丢弃最旧轮次） | 不触及启动记忆 |
| Layer 3 | 自动压缩（AI 生成摘要） | 压缩对话历史，保留启动记忆 |
| Layer 4 | 紧急压缩（API 返回 prompt-too-long 时触发） | 激进压缩，但启动记忆仍优先保留 |

**启动记忆（System Prompt、CLAUDE.md、Auto Memory）在所有压缩层级中都享有最高优先级**——它们不会被压缩或截断，而是在每次压缩后重新注入。

### 9.5 会话持久化（JSONL 追加存储）

根据 OpenClaude 的 [EP09: 会话持久化](https://openedclaude.github.io/claude-reviews-claude/zh-CN/chapters/09-session-persistence) 分析：

- Claude Code 使用 **JSONL（JSON Lines）格式**追加存储每个会话的完整对话记录
- 支持**恢复（Resume）**、**分叉（Branch）**和**搜索（Search）**操作
- `/resume` 命令可以回到之前的会话
- `/branch` 命令可以从之前的会话创建分支
- 跨会话的记忆通过 Auto Memory 和 CLAUDE.md 实现，而非通过 JSONL 对话记录

---

## 10. Harness 工程视角：Memory 在 Agent 系统中的定位

### 10.1 Harness 的完整公式

shareAI-lab 在 learn-claude-code 中提出了 Harness 的完整定义：

```
Harness = Tools + Knowledge + Observation + Action Interfaces + Permissions

其中：
  Tools       → 文件 I/O、Shell、网络、数据库、浏览器
  Knowledge   → 产品文档、领域参考、API 规范、风格指南  ← Memory 在此层
  Observation → Git diff、错误日志、浏览器状态、传感器数据
  Action      → CLI 命令、API 调用、UI 交互
  Permissions → 沙箱隔离、审批工作流、信任边界
```

Claude Code 的记忆系统（CLAUDE.md + Auto Memory + .claude/rules/）就是 **Knowledge 层**的具体实现。

### 10.2 "Load on demand, not upfront" 原则

Harness 工程的一条核心原则是：

> **按需加载，而非预先加载。**

Claude Code 的记忆系统充分体现了这一原则：

- CLAUDE.md 只在启动时加载（一次性的必要开销）
- 子目录 CLAUDE.md 只在访问该目录时才加载
- `.claude/rules/` 只在匹配文件被读取时才加载
- MCP 工具的完整 Schema 默认延迟加载（按需搜索）

这确保了 Context Window 这个"最稀缺的资源"不被无关信息占满。

### 10.3 Context 管理是 Harness 工程师的核心职责

shareAI-lab 指出，Harness 工程师的核心职责之一就是 **Context 管理**：

> Subagent isolation prevents noise leakage. Context compaction prevents history from drowning the present. Task systems let goals persist beyond a single conversation.

Claude Code 的记忆系统完美展示了这一职责的具体实践：

- **Subagent 隔离**：防止探索结果污染主会话上下文
- **Context 压缩**：防止历史对话淹没当前任务
- **任务系统**：让目标在单次对话之外持续存在

### 10.4 从 Claude Code 看其他 Agent 框架的记忆设计启示

Claude Code 的记忆设计对其他 Agent 框架有以下启示：

1. **文件优先于数据库**：将记忆存储为用户可读的文件，而非嵌入式数据库，提高了透明度和可调试性
2. **分层作用域**：从宽到窄的作用域层次结构，确保指令在合适的粒度上生效
3. **按需加载**：不是所有记忆都在启动时加载，按需触发可以大幅降低 Token 开销
4. **自动 + 手动双轨**：既有手动编写的指令，也有自动积累的学习，两者互补
5. **硬截断保护**：Auto Memory 的 200 行/25KB 限制防止了记忆系统失控

---

## 11. 最佳实践

### 11.1 记忆文件的组织结构建议

基于官方文档和社区分析，推荐以下组织结构：

```
项目根目录/
  CLAUDE.md                    ← 项目级指令（< 200 行）
  CLAUDE.local.md              ← 个人偏好（加入 .gitignore）
  .claude/
    CLAUDE.md                  ← 项目级指令的替代位置
    rules/
      api-conventions.md       ← API 层规则（paths: "src/api/**"）
      testing.md               ← 测试规则（paths: "*.test.ts"）
      database.md              ← 数据库规则（paths: "src/db/**"）
    settings.json              ← 项目级设置
    settings.local.json        ← 个人项目设置
```

### 11.2 避免常见陷阱

#### 规则冲突

当多个 CLAUDE.md 文件或规则文件中存在冲突指令时，Claude 可能任意选择遵循哪一条。定期审查并解决冲突。

#### 文件过大导致 Token 浪费

- 每个 CLAUDE.md 文件控制在 200 行以内
- 将多步骤流程移到 Skill 中
- 将路径特定规则移到 `.claude/rules/` 中
- 使用 `claudeMdExcludes` 排除无关包

#### 过时信息未及时清理

Claude Code 自动创建配置文件的带时间戳备份，并保留最近的 5 个备份。但 CLAUDE.md 本身**不会自动清理过时信息**。建议：

- 在 Pull Request 中将 CLAUDE.md 变更视为文档变更进行审查
- 在大模型版本更新后重新审视规则（某些规则可能是为绕过旧模型限制而写的）
- 使用 Stop Hook 在会话结束时提出 CLAUDE.md 更新建议

### 11.3 团队协作中的记忆管理

在团队环境中，记忆管理需要协调：

| 文件 | 作用域 | 共享方式 | 维护者 |
|------|--------|----------|--------|
| `CLAUDE.md`（根目录） | 项目级 | Git 提交 | 项目维护者 |
| `CLAUDE.md`（子目录） | 包/子系统级 | Git 提交 | 各包负责人 |
| `.claude/rules/` | 路径级 | Git 提交 | 领域专家 |
| `CLAUDE.local.md` | 个人项目级 | `.gitignore` | 个人 |
| `~/.claude/CLAUDE.md` | 用户级 | 不共享 | 个人 |

### 11.4 何时用 Hook 替代 Memory

当需要**强制执行**某个规则（无论 Claude 怎么想都必须遵守）时，使用 Hook 而非记忆：

```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": {
      "Bash(rm -rf *)": "deny"
    }
  }
}
```

Hook 是**强制性的**，记忆是**建议性的**。这是一个根本性的区别。

**选择决策树**：

```
需要 Claude 遵守某个规则？
  ├── 必须强制执行，无论 Claude 怎么想 → 使用 Hook（PreToolUse）
  └── 作为背景知识，希望 Claude 参考 → 使用 CLAUDE.md
      ├── 只在特定路径下生效 → 使用 .claude/rules/
      ├── 只在特定文件类型下生效 → 使用路径模式
      └── 全局生效 → 直接写入 CLAUDE.md
```

### 11.5 定期审查与更新策略

VILA-Lab 论文中提到的一个实用策略：

> **Revisit after major model releases**: instructions that worked around an older model's limitation may become overhead once a newer model handles the case on its own.

建议在以下时机审查记忆文件：

1. 大模型版本升级后（如从 Sonnet 3.5 升级到 Sonnet 4.0）
2. 项目架构重大变更后
3. 团队新增成员后
4. 每季度例行审查

---

## 12. 总结

### 12.1 Claude Code 记忆体系的核心设计

Claude Code 的记忆系统围绕三个核心原则构建：

1. **文件即持久化**：所有记忆都以用户可读的文件形式存在，支持版本控制和协作
2. **上下文即记忆**：记忆在会话启动时注入 Context Window，作为模型理解的背景知识
3. **按需加载**：通过目录层级和路径规则实现按需加载，最大化 Token 利用效率

这套系统的设计体现了 Claude Code 的核心哲学：LLM 是大脑，Harness 是身体。记忆系统是 Harness 中 Knowledge 层的关键实现，它让无状态的 LLM 在编程场景中表现得"有记忆"。

### 12.2 与其他 AI 编程工具的记忆方案对比

| 工具 | 记忆机制 | 特点 |
|------|----------|------|
| **Claude Code** | CLAUDE.md + Auto Memory + .claude/rules/ | 文件驱动、分层作用域、按需加载、双轨制 |
| **Cursor** | .cursorrules + .cursor/memory | 规则文件为主，缺乏自动学习机制 |
| **GitHub Copilot** | 系统提示 + 代码上下文 | 依赖代码上下文，无用户可编辑的记忆文件 |
| **OpenClaw** | AGENTS.md + MEMORY.md + memory/ | 类似 Claude Code 的设计，使用 AGENTS.md 而非 CLAUDE.md |

Claude Code 的独特之处在于：

- **双轨制**（手动 + 自动）的记忆积累
- **四层作用域**（Managed → User → Project → Local）的精细控制
- **路径级规则**的按需加载机制

### 12.3 未来演进方向

基于当前的架构和社区分析，Claude Code 记忆系统的未来演进方向可能包括：

1. **向量检索增强**：将 Auto Memory 从简单的行限制扩展为基于向量检索的相关性排序
2. **跨项目知识共享**：允许用户级别的记忆在不同项目之间迁移学习
3. **团队级 Auto Memory**：团队共享的自动学习机制，而非仅限于个人仓库
4. **智能规则推荐**：基于代码变更模式自动推荐 CLAUDE.md 更新
5. **记忆效能分析**：提供记忆文件的 Token 消耗和遵循率分析报告

---

## 附录

### A. CLAUDE.md 模板示例

```markdown
# 项目概览

这是一个基于 Node.js + TypeScript 的 REST API 项目。

## 构建与测试

- 安装依赖：`npm install`
- 运行开发服务器：`npm run dev`（端口 3001）
- 运行测试：`npm test`（Vitest）
- 数据库迁移：`npm run migrate`

## 代码规范

- 使用 2 空格缩进
- TypeScript strict mode
- API handlers 放在 `src/api/handlers/`
- 数据库查询使用 Knex，禁止原始 SQL 字符串

## 常用工作流

- 新功能开发：创建分支 → 实现 → 测试 → PR
- 数据库变更：编写 migration → 运行 `npm run migrate` → 更新 schema

## 环境配置

- 复制 `.env.example` 到 `.env` 并填写配置
```

### B. 常用命令速查表

| 命令 | 功能 | 说明 |
|------|------|------|
| `/init` | 自动生成 CLAUDE.md | 分析代码库，生成构建命令、测试指令等 |
| `/memory` | 管理 Auto Memory | 查看、编辑、清理自动积累的记忆 |
| `/compact` | 压缩对话历史 | 保留启动记忆，压缩对话内容 |
| `/context` | 查看 Context Window 状态 | 显示当前 Token 使用情况 |
| `/config` | 打开设置界面 | 管理项目和个人配置 |

### C. 设置文件路径速查

| 文件 | 作用域 | 位置 | 共享 |
|------|--------|------|------|
| 用户设置 | 用户级 | `~/.claude/settings.json` | 仅自己 |
| 项目设置 | 项目级 | `.claude/settings.json` | 团队（Git） |
| 本地设置 | 本地级 | `.claude/settings.local.json` | 仅自己（Gitignored） |
| Managed 设置 | 组织级 | `/etc/claude-code/managed-settings.json` | 组织内所有用户 |

### D. Claude Code 记忆相关关键源码文件列表

> 基于 [claude-code-rev](https://github.com/oboard/claude-code-rev) 逆向源码和 [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code) 架构分析

| 模块 | 功能 | 相关文件 |
|------|------|----------|
| 启动引导 | CLAUDE.md 扫描与加载 | `bootstrap/state.ts`, `context/claudeMd.ts` |
| 上下文组装 | 记忆注入 Context Window | `context/assembly.ts`, `context/systemPrompt.ts` |
| 压缩系统 | 五层压缩管线 | `compact/microcompact.ts`, `compact/autoCompact.ts` |
| Auto Memory | 自动学习存储与加载 | `memory/autoMemory.ts` |
| 会话持久化 | JSONL 存储与恢复 | `session/persistence.ts`, `session/jsonl.ts` |
| 路径规则 | .claude/rules/ 触发 | `rules/pathMatcher.ts`, `rules/loader.ts` |

---

**参考来源**：

1. [Claude Code 官方文档 — Memory](https://code.claude.com/docs/en/memory)
2. [Claude Code 官方文档 — Context Window](https://code.claude.com/docs/en/context-window)
3. [Claude Code 官方文档 — Settings](https://code.claude.com/docs/en/settings)
4. [Claude Code 官方文档 — Large Codebases](https://code.claude.com/docs/en/large-codebases)
5. [Claude Code 官方文档 — Sub-agents](https://code.claude.com/docs/en/sub-agents)
6. [Dive into Claude Code (VILA-Lab)](https://github.com/VILA-Lab/Dive-into-Claude-Code) — [论文 arXiv:2604.14228](https://arxiv.org/abs/2604.14228)
7. [Claude Code Reviews Claude (OpenClaude)](https://openedclaude.github.io/claude-reviews-claude/zh-CN/overview)
8. [learn-claude-code (shareAI-lab)](https://github.com/shareAI-lab/learn-claude-code)
9. [claude-code-rev (oboard)](https://github.com/oboard/claude-code-rev)
