---
title: Cloud Code Memory 
weight: 3
---


# Claude Code Memory 深度解析 — 项目记忆的两种路径

> "The bitter lesson is based on the historical observations that 1) AI researchers have often tried to build knowledge into their agents, 2) this always helps in the short term, and is personally satisfying to the researcher, but 3) in the long run it always degrades performance, and 4) eventually the methods based on scaling computation and data win."
>
> — Rich Sutton, *The Bitter Lesson* [7]

## 引言：AI 编程助手为什么需要跨会话记忆？

想象这样一个场景：你早上花了半小时，在 Claude Code 的对话中详细描述了项目的架构约定——"所有 API 响应必须用 Pydantic 模型包装"、"数据库迁移用 Alembic，不要用 Django migrations"、"测试文件放在 `tests/` 下，遵循 `test_<module>.py` 命名"。中午吃完饭回来，开了一个新会话，第一句话就让 Claude 改一个 API endpoint。它反问："请问这个项目的 API 响应格式是什么样的？"

你又要把早上的话再说一遍。

这不是 Claude 不够聪明，而是 LLM 的本质决定的：**每一次 API 调用都是独立的，模型没有记忆**。上一次会话中积累的上下文、决策、约定，随着会话结束全部消散。这是大语言模型的物理定律，不是 bug，是 feature——无状态保证了可重复性和可伸缩性，但也意味着每次对话都是"初见"。

然而，人类程序员不是这样的。你不需要每天早上重新告诉同事项目的编码规范。你的大脑会记住：这个项目的惯例是什么、上次踩了什么坑、哪个模块的代码风格特殊。这种记忆是跨天的、跨周的、甚至跨年的。

Claude Code 的记忆系统就是要解决这个鸿沟。它通过两条互补的路径，让 AI 编程助手从"无状态的对话者"进化为"有经验的协作者"：

- **CLAUDE.md**：你主动编写的持久指令文件——相当于给 Claude 一本"项目宪法"
- **auto memory**：Claude 自动积累的经验文件——相当于 Claude 自己的"工作日志"

更有趣的是，Claude Code 的记忆设计做了一个"反直觉"的选择：**完全基于纯文本 Markdown 文件，不使用向量数据库、不使用嵌入模型、不使用语义搜索**。所有记忆都是可读、可编辑、可版本控制的普通文件 [3]。这个选择背后有深刻的工程考量，我们将在本文中逐一拆解。

### 多源研究视角

本文的分析基于以下参考源的多源三角验证：

| 视角 | 参考源 | 独特贡献 |
|------|--------|---------|
| 官方规范 | [1][2] Anthropic 官方文档 | CLAUDE.md 语法、auto memory 开关、加载规则、故障排查 |
| 源码逆向还原 | [3][4] Dive-into-Claude-Code + claude-code-rev | 4 级层级实现细节、5 层压缩机制、LLM 检索逻辑、真实代码路径 |
| 从零构建教学 | [5] learn-claude-code | 理解为什么需要记忆系统、最小可行实现、学习路径 |
| AI 自述解构 | [6] Claude Reviews Claude | 第一人称架构视角、上下文装配流程、子系统交互 |
| 学术论文级拆解 | [3][7][8] | 设计原则、数据量化、趋势判断 |

---

## 1. 两种记忆路径：CLAUDE.md vs. auto memory

Claude Code 的记忆系统不是一整块单一机制，而是两条并行路径，各自解决不同的问题。

### 1.1 CLAUDE.md：项目宪法

CLAUDE.md 是一个（或一组）Markdown 文件，里面写着你希望 Claude 记住的所有持久指令。它可以包含：

- 编码规范和命名约定
- 架构约束和设计模式
- 项目特有的工具和流程
- 禁止事项和反模式
- 团队约定和协作规则

CLAUDE.md 的内容会在每次 Claude Code 启动时自动加载，并注入到 Claude 的上下文窗口中。它就像一份"宪法"——你制定了规则，Claude 在每次会话中都会遵守（或者说，尽量遵守——我们稍后讨论"概率性遵守"的问题）。

### 1.2 auto memory：工作日志

auto memory 是 Claude 的自动学习机制。在与你的对话过程中，Claude 会识别出有价值的经验——比如你反复纠正的编码习惯、你偏好的测试策略、你对某个模块的特殊处理方式——并自动将这些经验保存到记忆文件中。

auto memory 的核心价值在于**不需要你主动维护**。你不需要停下来写文档，Claude 会在对话的间隙自动积累。但它也有明显的局限：你无法精确控制 Claude 记住了什么、忘记了什么。

### 1.3 两种路径的对比

| 维度 | CLAUDE.md（显式指令） | auto memory（隐式学习） |
|------|---------------------|----------------------|
| **控制权** | 完全在用户手中 | 由 Claude 自主决定 |
| **可审计性** | 100% 透明，随时可查看和编辑 | 可查看但需通过 `/memory` 命令 |
| **适用场景** | 架构约束、编码规范、禁止事项 | 个人偏好、反复纠正的习惯、隐性知识 |
| **维护成本** | 高：需要主动编写和更新 | 低：自动积累，但需定期审查 |
| **可靠性** | 高：内容确定、格式固定 | 中：Claude 可能记错或遗漏 |
| **生命周期** | 随项目长期存在 | 可能随会话积累而膨胀 |
| **类比** | 项目宪法——你写的规则 | 工作日志——Claude 自己记的笔记 |

关键区别在于：**CLAUDE.md 的内容作为 user context 注入（概率性遵守），而非 system prompt（确定性执行）**[3]。这意味着 Claude 会"读到"你写的规则，但它不是一个硬编码的约束——它是一个指导性的上下文。相比之下，system prompt 中的指令是模型行为的硬性约束，优先级更高但用户不可修改。

这个设计选择反映了 Anthropic 的工程哲学：**给用户最大的灵活性，同时保持 AI 的自主判断能力**。你可以通过 CLAUDE.md 告诉 Claude "这个项目用 pytest 而不是 unittest"，但 Claude 仍然可以在特殊情况下做出合理的例外判断。

---

## 2. CLAUDE.md 文件体系：显式指令的四层架构

CLAUDE.md 最容易被误解的地方是：它不是一个文件，而是一个**层级系统**。Claude Code 在启动时会从四个层级读取 CLAUDE.md 文件，按照优先级合并成一个完整的指令集。

### 2.1 四级 CLAUDE.md 层级

源码分析揭示了 Claude Code 的 4 级 CLAUDE.md 层级 [3]：

```
Managed 级 → User 级 → Project 级 → Local 级
  /etc/      ~/.claude/   项目根目录    项目根目录
                           +.claude/    (gitignored)
                           rules/
```

| 层级 | 路径 | 作用域 | 典型内容 |
|------|------|--------|---------|
| **Managed（组织级）** | `/etc/claude/CLAUDE.md` | 全组织所有项目 | 安全策略、合规要求、公司编码标准 |
| **User（用户级）** | `~/.claude/CLAUDE.md` | 该用户的所有项目 | 个人编码偏好、常用工具配置 |
| **Project（项目级）** | `<project>/CLAUDE.md`<br>`<project>/.claude/rules/` | 特定项目 | 项目架构、编码规范、工具链 |
| **Local（本地级）** | `<project>/CLAUDE.local.md` | 个人本地开发环境 | 临时实验配置、个人调试设置 |

**加载顺序与优先级**：Claude Code 从 Managed 级开始读取，依次到 Local 级。越靠近 Local 级的规则优先级越高，可以覆盖上层规则。这种设计类似于 Git 的配置系统（`/etc/gitconfig` → `~/.gitconfig` → `.git/config`），让不同层级的规则可以共存而不冲突。

### 2.2 `.claude/rules/` 目录：路径级规则

项目级 CLAUDE.md 可以通过 `.claude/rules/` 目录进一步细化。这个目录允许你为**项目中的不同路径设置不同的规则**：

```
my-project/
├── CLAUDE.md              # 全局项目规则
└── .claude/
    └── rules/
        ├── frontend.md    # 前端代码规则
        ├── backend.md     # 后端代码规则
        └── tests.md       # 测试代码规则
```

当 Claude 处理 `frontend/` 目录下的文件时，它会自动加载 `frontend.md` 中的规则；处理 `backend/` 时加载 `backend.md`。这种路径级规则对于大型项目尤其有用——前端和后端可能有完全不同的编码规范、测试策略和架构约束。

### 2.3 AGENTS.md 兼容层

Claude Code 还支持读取 `AGENTS.md` 文件，这是与 OpenAI Codex 等工具的约定文件互通层 [1]。如果你的项目同时使用多个 AI 编程助手，AGENTS.md 可以作为一个通用的指令文件，被不同的工具读取和遵守。这种互操作性反映了 AI 编程工具生态的成熟——从各自为战到逐渐形成共识标准。

### 2.4 大团队管理

对于大型组织，CLAUDE.md 支持组织级部署和排除规则：

- **组织级 CLAUDE.md**：通过 Managed 级（`/etc/`）部署全组织统一的规则，如安全策略、合规要求
- **排除规则**：在 `.claude/settings.json` 中可以配置排除特定的 CLAUDE.md 文件，实现精细的规则控制
- **符号链接**：可以通过符号链接在不同项目间共享规则文件，避免重复维护

---

## 3. Auto Memory：隐式学习的机制与边界

如果说 CLAUDE.md 是你给 Claude 写的"教科书"，那 auto memory 就是 Claude 自己记的"笔记本"。

### 3.1 工作原理

auto memory 的运作流程可以概括为三步：

1. **识别**：在对话过程中，Claude 识别出有价值的经验——比如你反复说"不要用 print 调试，用 logging"
2. **提取**：Claude 将这条经验抽象为一条通用的规则或偏好
3. **保存**：将抽象后的规则保存到记忆文件中

这个过程发生在对话的间隙，不会打断你的工作流程。Claude 会在合适的时机自动执行，你甚至可能没有察觉。

### 3.2 存储位置

auto memory 的内容存储在 `.claude/` 目录下的记忆文件中。具体来说：

- 记忆内容保存在 Claude Code 内部管理的记忆文件中（不在用户直接编辑的 CLAUDE.md 中）
- 用户可以通过 `/memory` 命令查看、编辑和删除记忆
- 记忆文件是纯文本格式，完全可检查、可编辑、可版本控制 [3]

### 3.3 记忆检索机制

这是 auto memory 设计中最有趣的部分：**Claude 不使用向量数据库或嵌入模型来检索记忆**。相反，它使用一种"朴素"但有效的方法 [3]：

1. **LLM 扫描文件头部**：Claude 读取每个记忆文件的头部（headers/metadata）
2. **选择相关文件**：基于当前任务的上下文，选择最多 5 个最相关的记忆文件
3. **注入上下文**：将选中文件的内容注入到当前的上下文窗口中

这种方法的优势显而易见：
- **完全可解释**：没有黑盒的向量相似度计算，每一步都是可读的文本匹配
- **可调试**：如果 Claude 使用了错误的记忆，你可以直接查看文件内容找到原因
- **无依赖**：不需要额外的向量数据库服务、不需要嵌入模型推理

当然，这种方法也有局限：它依赖于 Claude 自身的判断能力来选择合适的记忆文件，而不是基于数学上的语义相似度。但随着模型智能的提升，这种"LLM 驱动的检索"正在变得越来越可靠。

### 3.4 启用、禁用与信任轨迹

用户可以完全控制 auto memory 的开关 [1]：
- **启用/禁用**：通过设置可以开启或关闭 auto memory
- **权限边界**：auto memory 只能访问和修改 Claude Code 管理的记忆文件，不能读取用户的其他文件
- **信任轨迹（trust trajectories）**[3]：auto memory 的设计考虑了信任建立的过程——Claude 不会在一开始就积累大量记忆，而是随着交互的深入，逐步建立对项目的理解。这类似于新员工入职的过程：一开始只知道公司名字，随着时间推移，逐渐了解项目细节和团队文化。

---

## 4. 记忆的生命周期：从写入到生效的完整链路

理解 Claude Code 的记忆系统，最关键的是理解它的**完整生命周期**。记忆不是"写进去就完事了"——从写入到生效，经历了多个阶段，每个阶段都有精心的工程处理。

### 4.1 写入阶段

记忆的写入有两种方式：

- **主动写入**：用户编辑 CLAUDE.md 文件，添加或修改指令
- **自动写入**：Claude 在对话过程中自动积累 auto memory

两种方式各有优劣。主动写入精确但费力，自动写入省力但不可控。最佳实践是两者结合：用 CLAUDE.md 管理核心的、稳定的规则，让 auto memory 捕捉细节和偏好。

### 4.2 加载阶段：9 个有序上下文来源

这是 Claude Code 记忆系统最精妙的部分之一。当 Claude Code 启动时，它不是简单地"读取 CLAUDE.md 然后开始工作"——它按照一个精心设计的顺序，从 **9 个不同的来源** 构建上下文窗口 [3]：

```
1. 系统提示（System Prompt）        — Claude 的身份和基本行为
2. 开发者信息（Developer Info）     — 平台级别的配置
3. CLAUDE.md 层级                  — 4 级规则文件的合并
4. 项目文件（Project Files）        — 关键项目文档
5. 记忆文件（Memory Files）         — auto memory 的内容
6. 会话历史（Session History）      — 当前对话的历史
7. 工具结果（Tool Results）         — 之前工具调用的结果
8. 工作区状态（Workspace State）    — 文件系统快照
9. 用户输入（User Input）           — 当前的问题/指令
```

这个加载顺序不是随意的——它反映了 Claude Code 的设计哲学：**确定性最高的信息优先级最高**。系统提示是最确定性的（硬编码），用户输入是最即时的（当前会话），中间的层级则按照稳定性和相关性排列。

### 4.3 生效阶段：与 Prompt Caching 的配合

Claude Code 利用 Anthropic 的 prompt caching 技术来优化记忆加载的性能 [8]。CLAUDE.md 等持久指令文件的内容通常变化频率低，非常适合缓存。当 Claude 启动新会话时，缓存的上下文可以直接复用，显著减少了 token 消耗和延迟。

### 4.4 维护阶段：5 层压缩机制

长时程任务经常超出 Claude 的上下文窗口长度。Claude Code 设计了一个**5 层渐进式压缩机制**（graduated lazy-degradation）来应对这个问题 [3]：

| 层级 | 机制 | 触发条件 | 效果 |
|------|------|---------|------|
| **1. Budget reduction** | 减少分配给非关键内容的 token 预算 | 上下文接近上限 | 最小影响，仅减少详细度 |
| **2. Snip** | 截断旧的工具结果和思考块 | Budget reduction 不够 | 丢失旧的中间数据 |
| **3. Microcompact** | 小范围的上下文摘要 | Snip 不够 | 部分历史被压缩成摘要 |
| **4. Context Collapse** | 读取时重建项目上下文 | Microcompact 不够 | 从 CLAUDE.md 和记忆文件重建 |
| **5. Auto-Compact** | 全模型摘要（最后手段） | 所有其他手段都不够 | 整个会话被压缩成摘要 |

这个 5 层机制的设计精妙之处在于：**它渐进地降级，而不是突然崩溃**。每一层都试图在前一层不够时补充，只有到了第 5 层才会做出不可逆的摘要决策。而且，第 4 层（Context Collapse）利用了 CLAUDE.md 和记忆文件作为"重建源"——这意味着即使上下文窗口被清空，Claude 仍然可以通过重新读取持久记忆文件来恢复项目知识。

这正是持久记忆的价值所在：**它是上下文窗口的"备份盘"**。当临时上下文被压缩或清空时，持久记忆可以确保关键的项目知识不会丢失。

---

## 5. 记忆系统的工程挑战

Claude Code 的记忆系统设计面临五个核心工程挑战，每个都有独特的解决方案。

### 5.1 挑战一：概率性遵守 vs. 确定性执行

CLAUDE.md 的内容作为 user context 注入，这意味着 Claude "读到"了规则，但它不是一个硬编码的约束。Claude 可能会在某些情况下选择忽略 CLAUDE.md 中的规则 [3]。

**何时用 CLAUDE.md，何时用其他方式？**

- 用 CLAUDE.md 管理**指导性规则**（"优先使用 X"、"倾向于 Y"）
- 用 hooks 或权限模式管理**强制性约束**（"绝对不要做 Z"）
- 用代码审查工具验证 Claude 是否遵守了 CLAUDE.md

### 5.2 挑战二：记忆膨胀

随着 auto memory 的不断积累，记忆文件可能变得越来越大，影响 Claude 的性能和响应速度。

**应对策略**：
- 定期审查 auto memory 内容（通过 `/memory` 命令）
- 删除过时或不准确的记忆
- 将稳定、重要的规则从 auto memory 迁移到 CLAUDE.md（从隐式转为显式）
- 利用 5 层压缩机制控制上下文窗口中的记忆内容

### 5.3 挑战三：记忆冲突

4 级 CLAUDE.md 层级可能导致规则冲突。比如，User 级规则说"用 4 空格缩进"，Project 级规则说"用 2 空格缩进"。

**解决机制**：
- 优先级规则：越靠近 Local 级的规则优先级越高
- 冲突检测：Claude 会尝试检测明显冲突的规则，并在遇到冲突时优先使用更具体的规则
- 用户主动管理：通过 `settings.json` 的排除规则，可以禁用特定层级的 CLAUDE.md

### 5.4 挑战四：记忆丢失

`/compact` 命令会压缩上下文窗口，可能导致 CLAUDE.md 中的某些指令被截断。

**保障措施**：
- CLAUDE.md 的内容在每次新会话启动时重新加载，不依赖 `/compact` 之前的上下文
- Context Collapse 机制会从持久记忆文件重建项目上下文
- 重要规则应放在 CLAUDE.md 的靠前位置（加载时优先保留）

### 5.5 挑战五：安全边界

auto memory 是否会泄露敏感信息？如果 Claude 记住了某个 API 密钥或内部架构细节，这些信息是否会被不当地传播？

**安全设计**：
- auto memory 只能访问 Claude Code 管理的记忆文件，不能读取用户的其他文件
- 记忆文件存储在 `.claude/` 目录下，受项目版本控制管理（可以选择 gitignore）
- Claude 的记忆积累受权限模式约束，不能自主访问受保护的资源

---

## 6. 与其他工具的记忆机制横向对比

Claude Code 不是唯一一个有记忆系统的 AI 编程助手。让我们横向对比主流工具的记忆设计。

### 6.1 对比表格

| 工具 | 记忆机制 | 存储格式 | 向量搜索 | 自动学习 | 层级结构 | 开源 |
|------|---------|---------|---------|---------|---------|------|
| **Claude Code** | CLAUDE.md + auto memory | 纯文本 Markdown | ❌ | ✅ | 4 级层级 | ❌ |
| **Cursor** | .cursor/rules + .cursorindex | 纯文本 + 索引文件 | ❌ | ❌ | 单层 | ❌ |
| **GitHub Copilot** | .github/copilot-instructions.md | 纯文本 Markdown | ❌ | ❌ | 单层 | ❌ |
| **OpenAI Codex** | AGENTS.md + SOUL.md + IDENTITY.md | 纯文本 Markdown | ❌ | ❌ | 多文件 | ✅* |
| **Cline** | CLINE.md + custom instructions | 纯文本 Markdown | ❌ | ❌ | 单层 | ✅ |
| **Aider** | .aider.input.history + .aider.conf | 纯文本 + YAML | ❌ | ❌ | 单层 | ✅ |

\* OpenAI Codex 的 AGENTS.md 规范是开放的，但 Codex 本体不开源。

### 6.2 Claude Code 的独特选择

Claude Code 在记忆系统设计上有两个突出的独特选择：

**选择一：纯文件，无向量 DB。** 大多数 AI 项目的记忆系统都会引入向量数据库来实现语义搜索。Claude Code 反其道而行——完全基于纯文本文件，用 LLM 自身的能力来检索记忆。这个选择看似"落后"，实则深思熟虑：
- 可调试性：每个决策都是可读的文本，不需要解释向量相似度
- 零依赖：不需要额外的数据库服务
- 版本控制友好：纯文本文件天然适合 Git 管理
- 成本效益：不需要嵌入模型的推理成本

**选择二：4 级层级结构。** 大多数工具只有一层记忆文件（如 `.cursor/rules`）。Claude Code 的 4 级层级（Managed → User → Project → Local）借鉴了 Unix 配置文件的传统（`/etc/` → `~/.` → 项目级 → 本地级），让不同范围的规则可以自然地共存。

---

## 7. 最佳实践：如何为你的项目设计记忆策略

### 7.1 小团队/个人项目：轻量起步

```
my-project/
├── CLAUDE.md              # 核心项目规则
└── .claude/
    └── settings.json      # auto memory 开关
```

CLAUDE.md 模板（建议不超过 50 行）：

```markdown
# My Project

## 技术栈
- Python 3.12 + FastAPI
- PostgreSQL + SQLAlchemy
- pytest 测试

## 编码规范
- 类型注解是强制的
- 所有公共函数必须有 docstring
- 使用 ruff 格式化

## 架构约定
- API 响应统一用 Pydantic 模型
- 数据库迁移用 Alembic
- 错误处理用自定义异常类

## 禁止事项
- 不要在代码中硬编码密钥
- 不要用 print 调试，用 logging
```

### 7.2 中型团队：路径级规则

```
my-project/
├── CLAUDE.md              # 全局项目规则
└── .claude/
    └── rules/
        ├── frontend.md    # React + TypeScript 规则
        ├── backend.md     # FastAPI + SQLAlchemy 规则
        └── tests.md       # 测试编写规范
```

关键原则：
- 全局 CLAUDE.md 只放**所有代码都适用**的规则
- 具体技术栈的规则放到对应的路径级文件中
- 定期审查 auto memory，将稳定的经验迁移到 CLAUDE.md

### 7.3 大型组织：Managed 级部署

```
/etc/claude/CLAUDE.md      # 组织级安全策略
├── 禁止在代码中存储密钥
├── 必须使用公司批准的依赖库
└── 所有 API 必须经过认证

~/.claude/CLAUDE.md        # 用户级偏好
├── 编辑器配置偏好
└── 个人编码习惯

project/CLAUDE.md          # 项目级规则
├── 技术栈和架构
└── 项目特定约定
```

关键原则：
- Managed 级规则应最小化——只放安全策略和合规要求
- 排除规则（`.claude/settings.json`）用于管理不需要加载的 CLAUDE.md
- 定期审计各层级的规则一致性

### 7.4 记忆治理四原则

1. **定期审查**：每月至少审查一次 auto memory 和 CLAUDE.md
2. **清理过期记忆**：删除不再适用的规则和过时的偏好
3. **避免记忆污染**：不要让 auto memory 积累太多低质量的记忆
4. **保持精简**：CLAUDE.md 应该在 50-200 行之间，超过这个范围考虑拆分

### 7.5 反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|---------|
| 百科全书式 CLAUDE.md | 文件过长，加载慢，Claude 难以关注重点 | 只写核心规则，详细内容放到单独文件并用 import 引用 |
| auto memory 不受控增长 | 积累大量过时或冲突的记忆 | 定期审查，将稳定的经验迁移到 CLAUDE.md |
| 层级冲突不解决 | 4 级规则相互矛盾，Claude 无所适从 | 明确每层级的职责边界，避免重复定义 |
| 把密钥写进 CLAUDE.md | 安全风险，可能提交到版本控制 | 使用环境变量或密钥管理服务 |

---

## 本章总结

### 核心要点

1. **Claude Code 的记忆系统由两条互补路径组成**：显式的 CLAUDE.md（项目宪法）和隐式的 auto memory（工作日志），前者精确可控，后者自动积累但需定期审查。

2. **4 级 CLAUDE.md 层级是核心设计**：Managed（`/etc/`）→ User（`~/.claude/`）→ Project（项目根目录）→ Local（gitignored），借鉴 Unix 配置传统，让不同范围的规则自然共存。

3. **基于纯文本文件的记忆设计是深思熟虑的选择**：不用向量 DB、不用嵌入模型，完全基于可读可编辑的 Markdown 文件，确保可调试性、零依赖和版本控制友好。

4. **CLAUDE.md 作为 user context 注入，是概率性遵守而非确定性执行**：Claude 会"读到"规则但不保证严格遵守，需要配合 hooks 和权限模式管理强制性约束。

5. **5 层渐进式压缩机制保障记忆的可靠性**：从 Budget reduction 到 Auto-Compact，逐层降级，只有到最后一层才会做出不可逆的摘要决策。

6. **LLM 驱动的记忆检索替代了传统的向量搜索**：Claude 通过扫描记忆文件头部选择最多 5 个相关文件，随着模型智能提升，这种方法正在变得越来越可靠。

7. **记忆治理需要主动管理**：定期审查、清理过期记忆、避免记忆污染、保持精简——记忆不是"设置后就忘了"，而是需要持续维护的系统。

### 读者行动建议

- **如果你刚开始使用 Claude Code**：先写一个 50 行以内的 CLAUDE.md，覆盖技术栈、编码规范和禁止事项
- **如果你已经用了一段时间**：运行 `/memory` 检查 auto memory 积累的内容，将稳定的经验迁移到 CLAUDE.md
- **如果你在管理团队**：建立 CLAUDE.md 模板和审查流程，确保规则的一致性和及时更新
- **如果你在评估 AI 编程工具**：关注各工具的记忆设计差异——文件存储 vs. 向量搜索、显式 vs. 隐式、单层 vs. 层级

### 延伸阅读

- [1] [Anthropic 官方文档 — How Claude remembers your project](https://code.claude.com/docs/en/memory)
- [2] [Anthropic 官方文档 — Explore the .claude directory](https://code.claude.com/docs/en/claude-directory)
- [3] [VILA-Lab — Dive into Claude Code](https://github.com/VILA-Lab/Dive-into-Claude-Code)（源码级架构分析论文）
- [5] [shareAI-lab — learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)（从零构建 agent harness）
- [7] [Anthropic — Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [8] [Anthropic — Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)
