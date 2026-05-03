---
title: (原理&实战)技术全景
weight: 1
---


# Deep Research：从搜索到自主研究的技术全景

> 本文是 [《大模型 Agent 和应用》](https://www6v.github.io/www6vAIGC/) 书籍的一个章节草稿。

---

## 1. 引言：什么是 Deep Research？

2025 年初，OpenAI 推出 Deep Research 功能，一句话让大模型替你读完数百个网页、写出一份带引用的研究报告。这看似只是搜索的"升级版"，但实际上它代表了一条截然不同的技术路线。要理解这一点，我们需要回溯信息获取方式的演进脉络。

### 1.1 定义与演进

信息获取经历了三个阶段的范式跃迁：

| 阶段 | 代表技术 | 核心能力 | 用户负担 |
|---|---|---|---|
| **Web Search** | Google/Bing | 关键词匹配 → 返回链接列表 | 用户需要逐一点击、阅读、筛选、整合 |
| **Agentic Search** | Perplexity、ChatGPT Search | 单次检索 + LLM 摘要 | 用户仍需判断是否需要进一步探索 |
| **Deep Research** | OpenAI Deep Research、各类开源 Agent | 多轮自主检索 → 推理 → 报告生成 | 用户只给一个问题，等待一份报告 |

关键区别在于 **自主性（Autonomy）** 和 **迭代深度（Iterative Depth）**。传统搜索中，人是大脑——模型只是加速了检索这一步；Deep Research 中，模型同时承担了大脑和执行者的角色：它自己决定"还要搜什么"，自己判断"信息够不够了"，自己把碎片拼成完整的知识图谱。

> 参考: [From Web Search towards Agentic Deep Research](https://arxiv.org/abs/2506.18959)
> 参考: [Deep Research Agents: A Systematic Examination And Roadmap](https://arxiv.org/abs/2506.18096)

### 1.2 与传统搜索/问答的本质区别

如果把传统搜索问答比作"问路"——你问一个方向，别人指一条路——那 Deep Research 就是"派一个研究员替你出差"。具体差异体现在三个维度：

**检索策略：单次 vs 多轮**

传统 RAG 或搜索问答通常遵循"一问一答"模式：用户提问 → 检索一次 → 生成回答。即使有 ReAct 循环，也往往在 2-3 轮内结束。Deep Research 的检索轮次通常在 10-50 轮之间，且每一轮搜索的查询词是由前一轮的发现动态生成的——前一步找到了某个关键概念，下一步就围绕这个概念深入。

**信息整合：摘要 vs 知识建构**

传统问答的"整合"大多是简单拼接：把检索到的几段文字压缩成一段摘要。Deep Research 需要的是**知识建构**——识别不同来源之间的矛盾，建立实体之间的关联，形成结构化的理解。这要求模型具备推理能力，而非仅仅是摘要能力。

**输出形态：回答 vs 报告**

传统问答的输出是"一段话"；Deep Research 的输出是一份"研究报告"，包含结构化章节、引用溯源、证据链，甚至可视化图表。这不是长度的差异，而是**信息组织方式**的根本不同。

### 1.3 为什么 Deep Research 是大模型应用的重要分水岭

Deep Research 的出现标志着一个转折点：**大模型从"对话工具"进化为"自主工作单元"**。

在此之前，LLM 应用的典型模式是"人在回路"（Human-in-the-loop）——用户发一条消息，模型回一条消息，节奏由人控制。Deep Research 打破了这个模式：用户启动任务后，模型在数十分钟内自主运行，期间可能发起上百次工具调用，最终交付一个完整的成果。

这意味着 LLM 应用的核心挑战从"如何让模型回答更好"转向了**"如何让模型自主工作得更可靠"**——包括任务规划、资源管理、事实核查、长程上下文管理等一系列全新问题。这些问题不是微调一下 prompt 就能解决的，它需要系统级的架构设计。

> 参考: [A Comprehensive Survey of Deep Research: Systems, Methodologies, and Applications](https://arxiv.org/abs/2506.12594)

---

## 2. 技术架构深度解析

要构建一个 Deep Research 系统，不能只靠一个 prompt 和 search API。它是一个复杂的多层系统，每一层都有明确的责任边界。

### 2.1 整体架构分层

我们可以将 Deep Research 系统拆解为四个层次：感知层、认知层、决策层和执行层。这种分层不是物理部署的划分，而是**逻辑职责**的抽象。

```
┌─────────────────────────────────────────────────┐
│                  用户 / 外部世界                   │
├─────────────────────────────────────────────────┤
│  感知层 (Perception)                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ Web搜索  │ │ 页面爬取  │ │ API调用  │ ...     │
│  └──────────┘ └──────────┘ └──────────┘         │
├─────────────────────────────────────────────────┤
│  认知层 (Cognition)                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ 信息理解  │ │ 推理引擎  │ │ 知识整合  │         │
│  └──────────┘ └──────────┘ └──────────┘         │
├─────────────────────────────────────────────────┤
│  决策层 (Decision)                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ 任务分解  │ │ 路径规划  │ │ 预算分配  │         │
│  └──────────┘ └──────────┘ └──────────┘         │
├─────────────────────────────────────────────────┤
│  执行层 (Action)                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ 工具调用  │ │ 报告生成  │ │ 人机交互  │         │
│  └──────────┘ └──────────┘ └──────────┘         │
└─────────────────────────────────────────────────┘
```
`[Mermaid 图]`

> 说明：上图展示了 Deep Research 系统的四层逻辑架构，自下而上构成完整的感知→认知→决策→执行闭环。

#### 感知层（Perception）：信息输入的管道

感知层负责从外部世界获取原始数据。它是系统与互联网的接口，决定了"能看到什么"。

**搜索引擎接入**是感知层最基础的能力。主流方案包括 Google Custom Search API、Bing Web Search API，以及开源替代方案 SearXNG。不同搜索引擎的覆盖范围和 API 特性差异显著：

| 搜索引擎 | 覆盖范围 | API 特性 | 适用场景 |
|---|---|---|---|
| Google Custom Search | 最广 | 100 次/天免费配额，付费扩展 | 通用搜索 |
| Bing Web Search | 广 | 按调用量计费，支持安全搜索 | 商业搜索 |
| SearXNG | 依赖实例 | 开源、无限制、需自建 | 隐私场景 |
| Tavily | AI 优化 | 专为 Agent 设计，返回结构化结果 | Agent 场景 |
| Brave Search | 中等 | 免费 2000 次/月 | 轻量场景 |

> 参考: [Tavily Search API](https://docs.tavily.com/)
> 参考: [SearXNG](https://docs.searxng.org/)

**网页内容提取**是从原始 HTML 中获取可读文本的关键步骤。直接解析 HTML 会引入大量噪声（导航栏、广告、脚本）。主流提取方案有：

- **Readability**：Mozilla 开源的页面可读性算法，提取正文、标题、作者等结构化信息。
- **Jina Reader**：将任意 URL 转换为 LLM 友好的 Markdown 格式，支持深层爬取。
- **Firecrawl**：完整的爬取管道，支持页面提取、网站地图遍历、结构化数据提取。

> 参考: [Jina Reader](https://jina.ai/reader/)
> 参考: [Firecrawl](https://www.firecrawl.dev/)

**多源数据整合**则意味着 Deep Research 不限于 Web 搜索。学术论文（ArXiv、Semantic Scholar）、新闻源（NewsAPI）、代码仓库（GitHub API）、以及企业私有知识库（通过 RAG 接入）都可以成为感知层的数据源。优秀的 Deep Research 系统应当具备**多源适配**能力，根据任务类型自动选择最合适的检索渠道。

#### 认知层（Cognition）：信息加工的大脑

如果说感知层是眼睛和耳朵，认知层就是大脑。它负责将原始信息转化为结构化知识。

**LLM 作为推理引擎**是认知层的核心。在 Deep Research 场景中，模型选择存在一个重要的权衡：

| 模型类型 | 推理能力 | 成本 | 延迟 | 典型代表 |
|---|---|---|---|---|
| 推理模型 | 极强 | 高 | 慢（秒级） | o1、o3、Claude Sonnet |
| 通用模型 | 中等 | 低 | 快（毫秒级） | GPT-4o、Qwen-Max |
| 轻量模型 | 基础 | 极低 | 极快 | Qwen-7B、Llama-3-8B |

实践中，多数 Deep Research 系统采用**混合策略**：用通用/轻量模型处理简单的信息提取和摘要任务，在关键的推理节点（如矛盾消解、策略调整）切换到推理模型。Tongyi DeepResearch 的技术报告就详细阐述了这种分层模型选择策略。

> 参考: [Tongyi DeepResearch Technical Report](https://arxiv.org/abs/2510.24701)

**信息抽取与结构化**要求模型从非结构化文本中识别实体（人名、组织、技术术语）、关系（A 是 B 的子类、A 导致了 B）、时间线（事件发生的先后顺序）。这一步的质量直接影响后续的知识整合效果。

**知识图谱构建与利用**是更高级的认知能力。当多轮检索积累了大量信息后，系统需要构建一个临时知识图谱来组织这些碎片——哪些实体已被覆盖？哪些关系尚未建立？哪些信息存在矛盾？知识图谱不仅用于组织信息，还能指导后续的搜索方向：图谱中的"空洞"就是下一步需要搜索的目标。

#### 决策层（Decision）：策略制定的指挥中心

决策层负责回答三个核心问题：**做什么？怎么做？做到什么程度？**

**任务分解策略**决定了如何将用户的自然语言问题拆解为可执行的子任务。两种典型路线：

- **自上而下（Top-down）**：先建立整体框架，再逐层细化。适合结构化问题（如"比较 A 和 B 的优缺点"）。Planner 首先识别问题的维度，然后为每个维度生成子查询。
- **自下而上（Bottom-up）**：先广泛收集信息，再逐步归纳模式。适合开放性问题（如"2024 年 AI 领域有哪些重大突破"）。Searcher 先做广度搜索，Critic 再识别关键模式。

**搜索路径优化**决定了探索的策略。类比图搜索算法：

- **广度优先（BFS 式）**：先覆盖主题的多个方面，每个方向浅尝辄止，再逐步深入。适合探索未知领域。
- **深度优先（DFS 式）**：沿着一条线索持续深入，直到穷尽或达到阈值。适合验证性研究。
- **自适应（A* 式）**：根据当前信息动态调整搜索方向，在广度和深度之间切换。这是 Deep Research 最理想但最难实现的模式，需要 Critic 实时评估信息饱和度并反馈给 Planner。

**资源预算分配**是 Deep Research 特有的挑战。系统需要在有限的 token 预算、时间预算和 API 调用次数内做出最优决策。这本质上是一个**多约束优化问题**：

- **Token 预算**：决定了上下文窗口中可容纳的信息量，直接影响推理质量。
- **时间预算**：用户通常期望在几分钟内获得结果，系统需要在速度和深度之间权衡。
- **API 调用限制**：搜索引擎和 LLM API 都有速率限制，系统需要合理分配调用配额。

#### 执行层（Action）：工具调用与结果交付

执行层是将决策层指令转化为实际行动的"手脚"。

**工具调用框架**是执行层的技术基础。当前主流方案有三种：

| 方案 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| Function Calling | 模型输出结构化 JSON，框架路由到对应函数 | 简单直接、生态成熟 | 调用深度受限 |
| MCP（Model Context Protocol） | 标准化协议，工具独立于模型 | 工具可复用、跨模型兼容 | 生态尚在早期 |
| ReAct（Reason + Act） | 模型交替输出推理和行动 | 灵活、支持复杂链式调用 | 需要精细的 prompt 设计 |

> 参考: [MCP Protocol](https://modelcontextprotocol.io/)

**报告生成与引用标注**是 Deep Research 的最终交付环节。一份合格的报告不仅是信息的堆砌，还需要：

1. **结构化组织**：按主题/维度分章节，逻辑递进。
2. **引用溯源**：每个观点都需要标注来源，读者可以追溯到原始信息。
3. **证据链**：重要结论需要多条独立来源支撑，形成交叉验证。

**人机交互**则贯穿整个执行过程。优秀的 Deep Research 系统会在关键节点向用户反馈中间状态（如"已搜索 15 个来源，正在分析..."），并允许用户在方向偏离时进行干预和修正。这种**人在回路**（Human-in-the-loop）的设计在长程任务中尤为重要。

### 2.2 核心组件详解

在四层架构的基础上，我们可以进一步抽象出五个核心组件。这些组件在实际系统中可能由同一个 LLM 实例的不同 prompt 角色来实现，也可能部署为独立的微服务。

```
                    ┌──────────┐
                    │ Planner  │ ← 任务理解与规划
                    └────┬─────┘
                         │ 分解为子任务
              ┌──────────▼──────────┐
              │      Searcher      │ ← 查询生成与多源检索
              └──────────┬─────────┘
                         │ 检索结果
              ┌──────────▼─────────┐
              │       Reader       │ ← 内容提取与信息摘要
              └──────────┬─────────┘
                         │ 结构化信息
              ┌──────────▼─────────┐
              │       Critic       │ ← 质量评估与矛盾消解
              └──────┬───────┬─────┘
                     │       │
              信息不足│       │信息充足
                     ▼       ▼
              ┌──────────┐ ┌──────────┐
              │ 返回迭代  │ │  Writer  │ ← 报告生成
              └──────────┘ └──────────┘
```
`[Mermaid 图]`

> 说明：五组件协同工作流。Critic 作为质量关卡，决定是继续迭代还是进入报告生成。

#### Planner（规划器）

Planner 是整个系统的"大脑皮层"，负责理解用户意图并制定研究计划。

**任务理解与意图分析**是 Planner 的第一步。用户的问题可能是模糊的（"帮我了解一下 RAG"），也可能是精确的（"比较 2024-2025 年间发布的三个开源 RAG 框架的吞吐量和延迟"）。Planner 需要识别问题的类型、所需的深度、以及隐含的约束条件。

**多粒度任务分解**意味着 Planner 不仅要拆解问题，还要在合适的粒度上进行拆解。太粗（"去搜一下 RAG"）会导致搜索方向模糊；太细（"分别搜索 RAG 的召回率、延迟、吞吐量..."）则会限制探索的灵活性。理想的分解应该是**层次化**的：顶层是研究维度，每个维度下是可执行的搜索查询。

**动态规划调整**是 Planner 区别于简单任务分解器的关键。在研究过程中，Planner 需要接收 Critic 的反馈，动态调整研究计划——发现了一个未曾预料到的关键概念，就应该将其纳入研究范围；某个方向的搜索结果质量持续不佳，就应该考虑放弃或换一种搜索策略。

#### Searcher（搜索器）

Searcher 负责将 Planner 生成的查询转化为具体的检索操作。

**查询生成与优化**不仅仅是把 Planner 的查询直接发给搜索引擎。优秀的 Searcher 会：

- 将自然语言查询转化为搜索引擎友好的关键词组合
- 生成多个变体查询，以覆盖不同的搜索角度
- 根据前一轮搜索结果调整后续查询策略（如某关键词效果不佳，尝试同义词或相关概念）

**多源检索策略**要求 Searcher 根据查询类型自动选择最合适的检索源。技术类查询可能更适合搜索 GitHub 和 ArXiv；商业类查询可能需要 NewsAPI 和行业报告数据库。

**去重与结果排序**是 Searcher 容易被忽视但至关重要的职责。多轮检索会产生大量重叠结果，去重不仅是 URL 级别的简单比较，还需要语义级别的去重——两个不同 URL 可能包含几乎相同的内容。结果排序则决定了哪些页面值得被 Reader 进一步处理。

#### Reader（阅读器）

Reader 将搜索结果页面转化为结构化信息。

**页面内容提取与清洗**使用 Readability、Jina Reader 或 Firecrawl 等工具，去除导航、广告、脚本等噪声，提取核心正文内容。

**关键信息定位与摘要**要求 Reader 在清洗后的文本中找到与当前研究问题相关的段落，并生成简洁摘要。这不是简单的文本截断，而是有选择性的信息提取——保留与研究问题直接相关的内容，丢弃无关信息。

**多格式处理**意味着 Reader 不仅要处理 HTML 页面，还要能解析 PDF（学术论文）、Markdown（技术文档）、JSON（API 响应）等多种格式。

#### Critic（评估器）

Critic 是 Deep Research 系统中负责质量控制的"守门员"。

**信息可信度评估**需要 Critic 判断每个信息来源的可靠性。学术论文通常比博客文章更可信；官方文档比第三方解读更可信；多个独立来源的一致性信息比单一来源更可信。

**相关性打分**衡量当前收集的信息与用户问题的关联程度。Critic 需要识别哪些信息是直接回答问题的，哪些是背景知识，哪些是完全无关的噪声。

**矛盾信息识别与消解**是 Critic 最核心的能力。当不同来源对同一事实给出不同描述时，Critic 需要：

1. 识别矛盾
2. 评估各方来源的可信度
3. 标注不确定性（而非强行选择一个答案）

这种能力对于避免 Deep Research 系统产生"看似权威实则错误"的报告至关重要。

#### Writer（报告生成器）

Writer 将 Critic 验证过的信息整合为最终的研究报告。

**结构化报告模板**要求报告有清晰的组织结构：摘要、背景、主体分析、结论、引用。不同主题的报告可能需要不同的模板，Writer 需要根据研究内容自动选择合适的结构。

**引用溯源与证据链**是 Writer 区别于普通文本生成的关键特征。报告中的每个重要观点都需要标注来源，读者可以追溯到原始信息。对于关键结论，还需要展示证据链——这个结论是基于哪些来源的哪些信息推导出来的。

**可视化呈现**则是报告的加分项。表格对比、流程图、时间线等可视化元素可以大幅提升报告的可读性。在 Deep Research 场景中，这些可视化可以由 LLM 生成 Mermaid 代码或 Python 图表脚本来实现。

### 2.3 关键算法模式

Deep Research 系统的底层算法模式决定了 Agent 的行为方式。以下是几种核心模式：

| 模式 | 核心思想 | 适用场景 | 局限性 |
|---|---|---|---|
| **ReAct** | Reason + Act 交替循环：先推理再行动，根据行动结果继续推理 | 需要多步工具调用的任务 | 深层推理时容易偏离主题 |
| **Plan-and-Solve** | 先制定完整计划，再逐步执行 | 结构化、可预测的任务 | 对突发发现缺乏灵活性 |
| **Iterative Refinement** | 生成 → 评估 → 修改的循环迭代 | 需要渐进式优化的任务 | 迭代次数难以确定 |
| **Self-Reflection** | Agent 评估自己的输出，识别不足并改进 | 需要高质量输出的场景 | 可能陷入自我确认偏差 |
| **Tree of Thoughts** | 维护多个推理分支，选择最优路径 | 需要探索多种可能性的任务 | 计算开销大 |
| **Agentic RAG** | 将 RAG 嵌入 Agent 循环中，检索与推理深度融合 | 知识密集型研究任务 | 实现复杂度高 |

**ReAct** 是最基础的 Agent 模式，也是大多数 Deep Research 系统的起点。它的核心循环是：

```
思考 → 行动 → 观察 → 思考 → 行动 → 观察 → ... → 最终回答
```

在 Deep Research 场景中，ReAct 的"行动"通常是搜索操作，"观察"是搜索结果。但单纯的 ReAct 模式存在一个致命缺陷：它容易在深层搜索中"迷失方向"——Agent 可能因为搜索结果中的某个有趣细节而偏离原始问题。

**Plan-and-Solve** 通过在 ReAct 之前增加一个规划阶段来缓解这个问题。Planner 先生成完整的研究计划，Searcher 再按计划执行。但这又带来了另一个问题：计划是静态的，而研究过程是动态的。

**Iterative Refinement** 和 **Self-Reflection** 则引入了反馈循环。Agent 生成初步报告后，自我评估报告质量，然后有针对性地进行补充搜索。这种模式在实践中效果显著，但需要设计合理的评估标准。

**Tree of Thoughts** 维护多个并行的研究路径，定期评估哪条路径最有价值。这种模式特别适合探索性研究，但计算开销很大，需要合理的剪枝策略。

**Agentic RAG** 是 RAG 与 Agent 的深度融合。传统 RAG 中，检索是一次性的、被动的；Agentic RAG 中，检索是主动的、多轮的、推理驱动的。检索不再只是"找相关文档"，而是"根据当前推理状态决定需要找什么信息"。

> 参考: [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)
> 参考: [Tree of Thoughts: Deliberate Problem Solving with LLMs](https://arxiv.org/abs/2305.10601)

### 2.4 Agent 架构范式

Deep Research 系统的 Agent 架构经历了从单一到多元的演进：

| 架构 | 描述 | 代表项目 | 优势 | 劣势 |
|---|---|---|---|---|
| **Single-Agent** | 一个 Agent 承担所有角色（Planner、Searcher、Reader、Writer） | gpt-researcher、deep-research | 实现简单、调试方便、延迟低 | 能力受限于单个模型、状态管理复杂 |
| **Multi-Agent** | 多个专用 Agent 各司其职，通过消息传递协作 | claude-cookbooks、PraisonAI | 专业化、可扩展、可并行 | 通信开销大、部署复杂、调试困难 |
| **Hierarchical Multi-Agent** | 分层管理：一个主 Agent 协调多个子 Agent | deer-flow | 兼顾灵活性和可控性 | 架构设计复杂、需要精心设计协调机制 |

**Single-Agent 架构**是当前最主流的选择。它的实现方式是让同一个 LLM 实例在不同阶段扮演不同角色（通过切换 system prompt）。这种架构的优势在于简单——不需要维护 Agent 之间的通信协议，不需要处理分布式状态。但它的上限也很明显：一个模型很难在所有角色上都达到最优。

**Multi-Agent 架构**将不同职责分配给不同的 Agent 实例。每个 Agent 可以有自己的 system prompt、甚至使用不同的模型（比如 Planner 用推理模型，Reader 用轻量模型）。这种架构的优势在于专业化和可扩展性——你可以为特定任务增加新的 Agent 角色。但通信开销和部署复杂度是主要挑战。

**Hierarchical Multi-Agent** 是 Multi-Agent 的升级版，引入了管理层和执行层的分离。一个主 Agent（Coordinator）负责任务分解和进度管理，多个子 Agent（Worker）负责具体执行。这种架构在 deer-flow 中得到了充分体现。

```
                    ┌──────────────┐
                    │ Coordinator  │ ← 任务协调与进度管理
                    └──────┬───────┘
                           │ 分发子任务
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  Researcher   │ │   Analyst    │ │   Writer     │
   │  (搜索+阅读)   │ │  (分析+评估)  │ │  (报告生成)   │
   └──────────────┘ └──────────────┘ └──────────────┘
```
`[Mermaid 图]`

> 说明：分层多 Agent 架构，Coordinator 负责任务分配和结果整合，Worker Agent 各司其职。

### 2.5 工作流示意

将上述组件和模式组合起来，一个典型的 Deep Research 工作流如下：

```
用户提问
    │
    ▼
┌─────────┐     任务分解      ┌─────────┐
│ Planner │ ───────────────→ │ Searcher│
└─────────┘                  └────┬────┘
                                  │ 原始搜索结果
                                  ▼
                            ┌─────────┐
                            │ Reader  │
                            └────┬────┘
                                 │ 结构化信息
                                 ▼
                            ┌─────────┐
                      ┌─────│ Critic  │
                      │     └────┬────┘
                      │          │
                 信息不足    信息充足/饱和
                      │          │
                      ▼          ▼
               ┌──────────┐ ┌──────────┐
               │ 返回Planner│ │  Writer  │
               │ 补充搜索  │ │ 报告生成  │
               └──────────┘ └────┬─────┘
                                 │
                                 ▼
                         输出交付（含引用）
```
`[Mermaid 图]`

> 说明：Deep Research 的迭代工作流。Critic 是关键的质量关卡，决定是否需要补充搜索。

> 参考: [Tongyi DeepResearch Technical Report](https://arxiv.org/abs/2510.24701)
> 参考: [gpt-researcher](https://github.com/assafelovic/gpt-researcher)
> 参考: [deer-flow](https://github.com/bytedance/deer-flow)

---

## 3. 技术挑战与局限

Deep Research 虽然强大，但它远非完美。这一章我们将直面当前的技术瓶颈——不是泛泛而谈，而是深入到具体问题和可能的解决方向。

### 3.1 信息幻觉与事实准确性

**幻觉**（Hallucination）是 LLM 的老问题，但在 Deep Research 场景中被放大到了一个新的量级。

**LLM 幻觉的本质原因**在于模型的生成机制：LLM 本质上是一个概率分布采样器，它生成的是"最可能的下一个 token"，而非"事实正确的下一个 token"。当模型在检索结果中找不到答案时，它倾向于"编造"一个看似合理的答案——这不是模型的"恶意"，而是其训练目标的必然结果。

**搜索结果噪声**加剧了幻觉问题。搜索引擎返回的结果中包含大量低质量内容——SEO 优化的博客、未经验证的观点、甚至虚假信息。当 Reader 将这些噪声信息提取后，Writer 可能将其当作事实写入报告。

**交叉验证的局限性**在于：即使系统从多个来源获取信息，如果这些来源都引用了同一个错误源头，交叉验证也无法发现问题。这就是所谓的"信息回音室"效应。

**解决方案方向**：

| 方案 | 原理 | 效果 |
|---|---|---|
| **引用约束** | 要求模型生成的每个观点都必须有引用来源，无引用的观点视为无效 | 显著降低"凭空捏造"的概率 |
| **事实核查** | 报告生成后，用独立的核查 Agent 验证关键事实 | 可以捕捉部分幻觉，但增加延迟 |
| **置信度标注** | 为报告中的每个结论标注置信度（高/中/低），低置信度结论需要人工复核 | 提升透明度，但依赖模型的自我评估能力 |

> 参考: [Deep Research Agents: A Systematic Examination And Roadmap](https://arxiv.org/abs/2506.18096)

### 3.2 搜索深度与 Token 成本的平衡

这是 Deep Research 最现实的经济问题。

**搜索深度指数增长 vs Token 消耗**。假设每一轮搜索平均获取 5 个页面，每个页面提取后约 2000 token，那么：

- 3 轮搜索：15 个页面 ≈ 30,000 token
- 10 轮搜索：50 个页面 ≈ 100,000 token
- 30 轮搜索：150 个页面 ≈ 300,000 token

再加上 Planner、Critic、Writer 的推理开销，一次完整的 Deep Research 任务消耗数十万甚至上百万 token 并不罕见。以 GPT-4o 的 API 价格计算，单次报告成本可能达到数美元——这对于个人用户来说是不可持续的。

**信息边际效用递减**是一个经济学概念，在 Deep Research 中同样适用。前几轮搜索通常能获取核心信息，后续搜索的增量价值迅速递减。第 10 轮搜索可能只带来 5% 的新信息，但却消耗了与前几轮相同的 token 成本。

**成本控制策略**：

| 策略 | 原理 | 权衡 |
|---|---|---|
| **早停机制** | 当连续 N 轮搜索的新信息量低于阈值时，自动停止搜索 | 可能错过重要信息 |
| **信息饱和度检测** | Critic 评估当前信息对问题的覆盖度，达到阈值即停止 | 需要精确的覆盖度评估模型 |
| **分层预算** | 为核心子问题分配更多预算，边缘问题分配较少预算 | 需要准确判断"核心"与"边缘" |

> 参考: [ReSum: Context Summarization for Long-horizon Search Agents](https://arxiv.org/abs/2509.13313)

### 3.3 长窗口上下文管理

Deep Research 的核心挑战之一是：**随着搜索轮次的增加，积累的上下文量远超单个模型的上下文窗口**。

**海量搜索结果超出上下文窗口**是一个硬约束。即使是 128K 或 200K token 的上下文窗口，在 30 轮搜索后也可能不够用。更糟糕的是，简单的"截断"策略会丢失早期搜索中获得的关键信息——这就是所谓的"越搜越忘"现象。

**上下文压缩技术**是解决这一问题的关键。ReSum（Recursive Summarization）提出了一种递归摘要方法：当上下文超出阈值时，将早期搜索结果压缩为结构化摘要，保留核心信息的同时大幅减少 token 消耗。实验表明，ReSum 可以将长程搜索的 token 消耗降低 40-60%，同时保持信息完整性。

**检索式记忆 vs 全量上下文**是两种不同的策略选择：

| 策略 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| **全量上下文** | 将所有搜索结果保留在 prompt 中 | 信息完整、推理质量高 | 窗口限制、成本高昂 |
| **检索式记忆** | 将历史搜索结果存入向量数据库，按需检索召回 | 突破窗口限制、成本低 | 可能遗漏相关信息 |

ReSum 的核心贡献在于找到了一条中间路径：它不是简单地压缩或检索，而是**有结构地压缩**——按照主题/维度组织信息摘要，使得压缩后的上下文仍然保留足够的结构信息供推理使用。

> 参考: [ReSum: Context Summarization for Long-horizon Search Agents](https://arxiv.org/abs/2509.13313)

### 3.4 信息时效性与实时性

**训练数据截止时间限制**是 LLM 的固有缺陷。即使模型拥有强大的推理能力，它的"知识"截止于训练数据的最后更新日期。对于"2025 年 3 月发布的最新技术趋势"这类问题，模型的先验知识可能完全空白。

**Web 搜索的实时性**在一定程度上弥补了这个缺陷——搜索引擎可以返回最新内容。但这带来了新的问题：搜索结果中的最新信息可能与模型训练数据中的旧信息产生冲突，模型需要判断哪个更可信。

**动态更新结果的整合**意味着 Deep Research 系统需要在研究过程中处理"新出现的信息"。比如，在一个持续 30 分钟的研究过程中，可能有一条新的新闻发布——系统是否应该将其纳入？如何评估其对已有结论的影响？

**持续学习机制的缺失**是当前 Deep Research 系统的一个根本性局限。模型无法从一次研究任务中"学习"并改进下一次任务的表现。每次研究都是"从零开始"，这在长期使用的场景中是一个效率瓶颈。

### 3.5 多模态信息处理

当前的 Deep Research 系统**以文本处理为主**，这既是技术现状也是能力限制。

**图表理解瓶颈**在于：许多重要信息以图表形式存在于网页或论文中（比如性能对比图、架构图、统计图表）。当前的多模态 LLM 虽然能理解简单的图表，但对于复杂的技术图表（如系统架构图、性能基准图）的理解能力仍然有限。

**视频/音频信息抽取**是一个更难的挑战。许多技术信息以视频形式存在（如 YouTube 技术演讲、产品演示视频），当前的 Deep Research 系统无法有效处理这些内容。即使通过自动语音识别（ASR）提取文本，也会丢失视觉信息。

**多模态推理**是终极挑战：当不同模态的信息需要整合时（比如一段文字描述 + 一张架构图 + 一个性能图表），模型需要跨模态推理来形成统一的理解。这在当前仍然是一个开放的研究问题。

> 参考: [A Comprehensive Survey of Deep Research: Systems, Methodologies, and Applications](https://arxiv.org/abs/2506.12594)

### 3.6 评估与基准测试

Deep Research 缺乏统一的评估标准，这是制约其发展的重要因素。

**现有基准的局限性**：

| 基准 | 设计目标 | 对 Deep Research 的适用性 | 局限 |
|---|---|---|---|
| **GAIA** | 通用 Agent 能力评估 | 部分适用 | 侧重单步推理，不评估长程研究能力 |
| **WebArena** | Web 环境中的 Agent 交互 | 有限 | 侧重 UI 交互，不评估信息整合质量 |
| **SWE-bench** | 软件工程任务 | 不适用 | 完全针对代码任务 |

**研究质量衡量**需要多个维度：

- **完整性**：是否覆盖了问题的所有重要方面？
- **准确性**：报告中的事实是否正确？
- **深度**：是否超越了表面信息，触及核心原理？
- **可读性**：报告是否结构清晰、易于理解？

**人工评估 vs 自动评估**的困境在于：自动评估（如 LLM-as-a-judge）效率高但可靠性存疑；人工评估可靠但成本高昂、难以规模化。当前学术界的主流做法是"混合评估"——用自动评估做初步筛选，用人工评估做最终验证。

> 参考: [GAIA Benchmark](https://arxiv.org/abs/2311.12983)
> 参考: [WebArena](https://arxiv.org/abs/2307.13854)

### 3.7 安全与伦理

**信息来源偏见与公平性**：搜索引擎的排名算法本身就存在偏见（SEO 优化、商业推广），Deep Research 系统如果直接采信搜索结果，可能会放大这些偏见。

**版权与知识产权**：Deep Research 系统抓取和整合大量网页内容，这些内容的版权归属和使用限制是一个灰色地带。生成的报告是否可以商用？引用了多少原始内容？这些问题目前缺乏明确的法律框架。

**敏感信息合规**：在涉及医疗、法律、金融等领域时，Deep Research 生成的报告如果包含错误或过时信息，可能导致严重后果。系统需要具备领域敏感性和合规检查能力。

**滥用风险**：Deep Research 系统可能被用于自动生成虚假信息、恶意竞争情报收集等场景。系统设计者需要考虑滥用防护机制。

---

## 4. 开源项目深度解析

> 数据来源：[awesome-deepResearch](https://github.com/www6v/awesome-deepResearch)，按 GitHub Stars 降序。

开源生态是 Deep Research 技术发展的重要推动力。以下分析基于 GitHub 社区的活跃度、技术贡献和社区反馈。

### 4.1 分类分析

Deep Research 开源项目可以根据架构和定位分为四大类：

#### End-to-end Agent 类

这类项目提供"开箱即用"的 Deep Research 体验，用户只需部署即可使用。

| 项目 | Stars | 核心特点 | 技术栈 |
|---|---|---|---|
| **[deer-flow](https://github.com/bytedance/deer-flow)** | 64.5k | 字节跳动出品，分层多 Agent 架构，支持多种搜索后端 | LangGraph + 多模型 |
| **[gpt-researcher](https://github.com/assafelovic/gpt-researcher)** | 26.8k | 最早的开源 Deep Research 项目之一，社区活跃 | LangChain + OpenAI |
| **[deep-research](https://github.com/dzhng/deep-research)** | 18.8k | 轻量级 TypeScript 实现，快速上手 | TypeScript + OpenAI |
| **[DeepResearch](https://github.com/Alibaba-NLP/DeepResearch)** | 18.8k | 阿里通义出品，配套技术报告，支持多种训练策略 | Qwen + 自研框架 |
| **[open_deep_research](https://github.com/langchain-ai/open_deep_research)** | 11.3k | LangChain 官方实现，与 LangChain 生态深度集成 | LangChain |

**典型架构**：搜索 → 阅读 → 推理 → 报告。这类项目的共同特点是实现了完整的 Deep Research 闭环，但各自的侧重点不同：deer-flow 强在架构设计（分层多 Agent），gpt-researcher 强在社区生态（丰富的插件和集成），DeepResearch 强在学术支撑（配套论文和训练方法）。

#### 多 Agent 框架类

这类项目提供的是"框架"而非"产品"，用户可以基于框架自定义 Agent 的角色和协作方式。

| 项目 | Stars | 核心特点 |
|---|---|---|
| **[claude-cookbooks](https://github.com/anthropics/claude-cookbooks)** | 42.1k | Anthropic 官方 Cookbook，包含多种 Multi-Agent 研究模式的参考实现 |
| **[PraisonAI](https://github.com/MervinPraison/PraisonAI)** | 7.0k | 低代码 Multi-Agent 框架，支持 AutoGen、CrewAI 等多种后端 |

**典型架构**：Planner → Researcher → Writer → Critic 协同。这类项目的优势在于可扩展性——用户可以增加新的 Agent 角色（如 FactChecker、DataAnalyst），也可以调整 Agent 之间的协作方式。

#### 本地/隐私类

这类项目强调"数据不出本地"，适合企业私有数据场景。

| 项目 | Stars | 核心特点 |
|---|---|---|
| **[local-deep-researcher](https://github.com/langchain-ai/local-deep-researcher)** | 9.1k | 本地 LLM + 本地检索，完全离线运行 |
| **[deep-searcher](https://github.com/zilliztech/deep-searcher)** | 7.8k | 基于 Milvus 向量数据库的深度搜索，支持私有知识库检索 |

**技术路线**：本地 LLM（如 Qwen-7B、Llama-3-8B）+ 本地检索（如 Milvus、Chroma 向量数据库）。这类项目的核心价值在于**数据隐私**——研究过程中的所有数据都保留在本地，不经过第三方 API。

#### Tool-Augmented 类

| 项目 | Stars | 核心特点 |
|---|---|---|
| **[gemini-fullstack-langgraph-quickstart](https://github.com/google-gemini/gemini-fullstack-langgraph-quickstart)** | 18.1k | Google 官方教学项目，展示 Gemini + LangGraph 的 Deep Research 实现 |

**技术路线**：Gemini + LangGraph 生态。这类项目的重点不是"产品化"，而是"教学示范"——帮助开发者理解如何用工具增强 LLM 的推理能力。

### 4.2 技术路线对比

#### 单 Agent vs 多 Agent

| 维度 | Single-Agent | Multi-Agent |
|---|---|---|
| **实现复杂度** | 低：一个 prompt 流程 | 高：需要设计 Agent 间通信协议 |
| **调试难度** | 低：线性流程，容易定位问题 | 高：分布式状态，问题可能在任何 Agent |
| **推理上限** | 受限于单个模型的能力 | 可通过专业分工突破单模型上限 |
| **资源消耗** | 低：一次模型调用链 | 高：多个 Agent 并发调用 |
| **扩展性** | 有限：增加功能需要修改核心流程 | 强：增加新 Agent 即可扩展能力 |
| **适用场景** | 个人使用、快速验证 | 企业级应用、复杂研究任务 |

**趋势**：从 Single-Agent 向 Multi-Agent / Hierarchical 演进。早期项目（如 gpt-researcher、deep-research）都采用 Single-Agent 架构，而较新的项目（如 deer-flow）已经转向分层多 Agent 架构。这反映了行业对 Deep Research 复杂性认识的深化。

#### Prompting vs SFT vs RL

| 方法 | 开发成本 | 性能上限 | 数据需求 | 代表方案 |
|---|---|---|---|---|
| **Prompting** | 极低 | 受限于基座模型 | 无 | 多数开源项目 |
| **SFT** | 中等 | 稳定提升 | 高质量标注数据 | DeepResearch 基线 |
| **RL（GRPO）** | 高 | 可突破基座上限 | 合成数据 + 奖励模型 | ReSearch、ZeroSearch、ReSum |

**Prompting** 是最快速的验证方式。通过精心设计的 system prompt 和 few-shot 示例，可以让通用 LLM 表现出 Deep Research 能力。但 Prompting 的上限完全取决于基座模型的固有能力的。

**SFT（监督微调）** 通过在 Deep Research 任务的标注数据上微调，可以显著提升模型在特定任务上的表现。但 SFT 需要大量高质量的标注数据——即完整的"问题→搜索过程→报告"示例对。

**RL（强化学习）** 尤其是 **GRPO（Group Relative Policy Optimization）**，已成为 Deep Research 领域最主流的优化范式。RL 的核心价值在于：它让模型学会"何时搜索、搜索什么、如何整合"这些策略性决策，而不仅仅是"如何生成文本"。GRPO 相比 PPO 的优势在于不需要额外的价值模型（Value Model），降低了训练复杂度。

#### 开源模型 vs 闭源 API

| 维度 | 开源模型（Qwen、Llama） | 闭源 API（GPT-o1、Claude） |
|---|---|---|
| **可控性** | 完全可控，可微调 | 黑盒，不可修改 |
| **成本** | GPU 成本高，但边际成本低 | 按 token 计费，长期使用成本高 |
| **性能** | 取决于具体模型，Qwen3 系列已接近 GPT-4 | 当前最强（o1、Claude 3.5） |
| **数据隐私** | 数据完全本地 | 数据发送到云端 |
| **部署难度** | 需要 GPU 基础设施和运维能力 | API 调用，即开即用 |

### 4.3 如何选择

**按场景推荐**：

| 场景 | 推荐项目 | 理由 |
|---|---|---|
| **学术研究** | gpt-researcher、open_deep_research | 支持多源学术检索，引用格式规范 |
| **商业分析** | deer-flow、PraisonAI | 分层架构适合复杂分析任务 |
| **个人学习** | deep-research、local-deep-researcher | 轻量级、快速上手 |
| **企业私有数据** | deep-searcher、local-deep-researcher | 数据不出本地，支持私有知识库 |

**按资源选择**：

| 资源条件 | 推荐方案 | 说明 |
|---|---|---|
| **有 GPU（单卡/多卡）** | DeepResearch（Qwen）、ZeroSearch | 可以用开源模型本地运行，支持微调 |
| **仅 API（无 GPU）** | gpt-researcher、open_deep_research | 基于 OpenAI/Claude API，无需本地 GPU |
| **零代码（无开发能力）** | gemini-langgraph-quickstart | Google 官方教学项目，配置即用 |

> 参考: [awesome-deepResearch](https://github.com/www6v/awesome-deepResearch)

---

## 5. 关键论文解读

> 以下论文均来自 [awesome-deepResearch](https://github.com/www6v/awesome-deepResearch) 的 Works 部分。

Deep Research 作为一个独立的研究方向，在 2025 年迎来了论文爆发。本节挑选最具代表性的论文进行深度解读。

### 5.1 技术趋势分析

#### RL 主导优化范式

纵观 2025 年 Deep Research 领域的论文，一个显著趋势是：**强化学习（RL）已成为优化 Deep Research Agent 能力的主流范式**。

| 论文 | RL 算法 | 优化目标 |
|---|---|---|
| ReSearch | GRPO | 搜索策略优化 |
| ZeroSearch | GRPO | 零搜索成本训练 |
| ReSum | GRPO | 上下文压缩策略 |
| Search-R1 | PPO | 搜索与推理端到端统一 |

**GRPO 成为最受欢迎的选择**，原因在于它不需要额外的价值模型（Value Model），只需对同一 prompt 生成多组输出、比较它们的奖励即可计算优势函数。这在 Deep Research 场景中尤为重要——因为训练一个可靠的价值模型本身就非常困难。

**RL 的核心价值**在于让模型学会策略性决策：何时应该继续搜索？何时应该停止？应该搜索什么方向？这些决策无法通过简单的 SFT 获得，因为它们涉及的是"做选择"而非"生成文本"。RL 通过奖励信号，让模型在试错中学习到最优策略。

> 参考: [ReSearch](https://arxiv.org/pdf/2503.19470)
> 参考: [ZeroSearch](https://arxiv.org/pdf/2505.04588)
> 参考: [ReSum](https://arxiv.org/abs/2509.13313)
> 参考: [Search-R1](https://arxiv.org/pdf/2503.09516)

#### Qwen 系列模型的统治地位

在 Deep Research 领域的 10 篇代表性论文中，有 **8 篇使用 Qwen 系列作为基座模型**。这一现象并非偶然：

| 因素 | 说明 |
|---|---|
| **MoE 架构的推理效率** | Qwen3-30B-A3B 等 MoE 模型在推理时只激活部分参数，在保持高推理能力的同时降低了计算开销 |
| **优秀的中文能力** | 对于中文 Deep Research 场景，Qwen 的中文理解能力显著优于 Llama |
| **开源生态** | Qwen 系列完全开源（模型权重、训练方法、工具链），降低了研究和部署门槛 |
| **推理工具调用能力** | Qwen 系列在 function calling 和 tool use 方面表现优异 |

Qwen3-30B-A3B 系列已成为 Deep Research 事实上的基准模型。这一趋势反映了 Deep Research 研究的一个重要特征：**它需要的不仅是"聪明"的模型，更是"会用工具"的模型**。

#### 合成数据（Synthetic Data）重要性提升

当真实标注数据稀缺时，合成数据成为突破瓶颈的关键。

**WebSailor-V2** 展示了合成数据与可扩展 RL 的训练范式：通过 LLM 自动生成高质量的 Deep Research 训练数据（包括搜索轨迹、信息整合过程、最终报告），然后用这些数据训练搜索 Agent。这种方法的关键在于合成数据的质量控制——需要设计合理的验证机制来确保合成数据的可靠性。

**ZeroSearch** 则将合成数据推向了极端：完全不需要真实搜索 API，用 LLM 模拟搜索结果来训练搜索策略。这种方法的意义在于它突破了搜索 API 的成本限制和速率限制，使得大规模训练成为可能。

> 参考: [WebSailor-V2](https://arxiv.org/abs/2509.13305)
> 参考: [ZeroSearch](https://arxiv.org/pdf/2505.04588)

### 5.2 架构演进

#### Single-Agent → Multi-Agent

在分析的 10 篇关键论文中，**9 篇采用 Single-Agent 架构，仅 WebResearcher 一篇采用 Multi-Agent 架构**。这反映了当前的研究现状：

- **Single-Agent 仍是主流**：实现简单、训练成本低、实验可重复性高。对于学术研究来说，Single-Agent 降低了实验的复杂度，使得研究者可以专注于核心算法的改进。
- **Multi-Agent 代表未来方向**：WebResearcher 论文展示了多角色协作（Planner、Searcher、Evaluator、Writer 各司其职）如何释放无界推理能力。虽然实现复杂，但它的性能上限更高，特别是在复杂研究任务中。

> 参考: [WebResearcher](https://arxiv.org/abs/2509.13309)

#### 阿里系论文的体系化贡献

一个值得注意的现象是，Deep Research 领域最具体系性的研究成果来自阿里团队。**5 篇论文形成了一个完整的研究体系**：

| 论文 | 核心贡献 | 在全链路中的位置 |
|---|---|---|
| **DeepResearch**（技术报告） | 完整的系统设计与训练方法 | 全局架构 |
| **ReSum** | 上下文压缩技术 | 长程搜索的上下文管理 |
| **WebWeaver** | 动态大纲驱动研究 | 研究路径规划 |
| **WebResearcher** | Multi-Agent 推理 | 多角色协作 |
| **WebSailor-V2** | 合成数据训练范式 | 训练数据获取 |

这 5 篇论文覆盖了 Deep Research 的全链路优化：从系统架构（DeepResearch）到上下文管理（ReSum）、路径规划（WebWeaver）、协作模式（WebResearcher）、训练数据（WebSailor-V2）。这种体系化的研究方式在 AI 领域并不多见，它反映了工业界研究团队的工程化思维。

> 参考: [DeepResearch](https://arxiv.org/abs/2510.24701)
> 参考: [ReSum](https://arxiv.org/abs/2509.13313)
> 参考: [WebWeaver](https://arxiv.org/abs/2509.13312)
> 参考: [WebResearcher](https://arxiv.org/abs/2509.13309)
> 参考: [WebSailor-V2](https://arxiv.org/abs/2509.13305)

### 5.3 关键突破点

#### ZeroSearch（2025/05）：无需真实搜索即可训练搜索能力

**核心思想**：用 LLM 模拟搜索结果，通过 RL 训练搜索策略。

传统搜索 Agent 的训练需要大量真实的搜索 API 调用——这既昂贵又受限于 API 速率。ZeroSearch 的创新在于：它训练一个"搜索模拟器"（Search Simulator），这个模拟器可以根据搜索查询生成逼真的搜索结果。然后，用这些模拟结果来训练搜索 Agent 的策略。

这个方案的关键在于模拟器的质量。ZeroSearch 通过精心设计的合成数据生成流程，确保模拟结果足够接近真实搜索结果，使得在模拟环境中训练的策略可以迁移到真实搜索场景。

**意义**：大幅降低训练成本，突破搜索 API 限制，使得 Deep Research 的训练可以在没有搜索 API 配额的情况下进行。

> 参考: [ZeroSearch](https://arxiv.org/pdf/2505.04588)

#### WebWeaver（2025/09）：动态大纲驱动的开放研究

**核心思想**：研究过程中动态生成和修改大纲。

传统的 Deep Research 系统要么使用固定大纲（Plan-and-Solve），要么完全没有大纲（纯 ReAct）。WebWeaver 提出了一种**动态大纲**（Dynamic Outline）机制：

1. 研究开始时，Planner 生成一个初步大纲。
2. 随着研究的深入，新发现的概念和关系被添加到大纲中。
3. 如果某个大纲分支的信息已经饱和，则标记为"已完成"。
4. 如果发现某个大纲方向不正确，则回退并重新规划。

这种机制使得研究过程从"线性"进化为"非线性、自适应"——系统可以像人类研究者一样，在研究过程中调整研究方向。

**意义**：解决了传统 Deep Research 系统"一旦开始就难以调整方向"的问题。

> 参考: [WebWeaver](https://arxiv.org/abs/2509.13312)

#### ReSum（2025/09）：长程搜索中的上下文压缩

**核心思想**：通过上下文摘要技术突破长程搜索的窗口限制。

在长程搜索中，Agent 可能执行数十轮搜索，累积的信息量远超上下文窗口。ReSum 提出了一种递归摘要（Recursive Summarization）方法：

1. 当上下文接近窗口限制时，将早期搜索结果压缩为结构化摘要。
2. 摘要按照主题/维度组织，保留关键实体、关系和结论。
3. 压缩后的摘要与新的搜索结果一起作为下一轮的上下文。

实验表明，ReSum 可以将长程搜索的 token 消耗降低 40-60%，同时保持信息完整性和推理质量。

**意义**：解决了"越搜越忘"的长程 Agent 核心痛点，使得 Deep Research 可以扩展到更深的研究轮次。

> 参考: [ReSum](https://arxiv.org/abs/2509.13313)

#### Search-R1（2025/03）：搜索与推理的 RL 统一

**核心思想**：将搜索工具调用内化为推理过程的一部分。

传统的搜索 Agent 中，搜索是一个"外部工具调用"——模型决定搜索，执行搜索，然后处理搜索结果。Search-R1 通过强化学习将搜索行为**内化**为模型推理的一部分，使得模型可以端到端地学习"何时搜索、搜索什么"的策略。

具体来说，Search-R1 使用 PPO 算法训练模型，奖励信号基于最终答案的质量（而非中间步骤的正确性）。这使得模型学会在需要时搜索、在不需要时跳过搜索，而不是机械地"每步都搜"。

**意义**：端到端训练消除了对启发式规则的依赖，使得搜索策略可以自适应地优化。

> 参考: [Search-R1](https://arxiv.org/pdf/2503.09516)

---

## 6. 未来展望

Deep Research 技术仍处于早期阶段。基于当前的技术趋势和研究方向，我们可以预见以下几个重要发展。

### 6.1 从"研究"到"行动"的闭环

当前的 Deep Research 系统的输出是"研究报告"——一份静态的文档。但研究的终极目的是**行动**。

未来的 Deep Research 系统将不仅告诉你"应该做什么"，还会**直接帮你做**。比如：

- 研究完"如何部署一个 RAG 系统"后，直接生成部署脚本并执行
- 研究完"某个竞品的功能对比"后，自动生成产品需求文档
- 研究完"某个技术方案的可行性"后，直接启动 PoC 项目

这需要 Deep Research 系统具备**行动能力**（Action Capability）——不仅是信息的整合者，更是任务的执行者。这也带来了新的安全挑战：一个可以"行动"的研究 Agent 需要严格的权限控制和人工审批机制。

### 6.2 开源生态与闭源产品的竞争格局

Deep Research 领域正在形成两条并行路线：

**闭源路线**（OpenAI Deep Research、Google Gemini、Anthropic Claude）的优势在于性能和体验。闭源产品可以使用最强大的模型（o3、Claude Opus），拥有最优的工程实现，并且可以直接集成到用户的日常工具中。

**开源路线**（deer-flow、gpt-researcher、DeepResearch 等）的优势在于可控性和定制化。企业可以针对特定场景微调模型，可以确保数据隐私，可以深度定制系统行为。

两者的竞争将推动整个领域的发展：闭源产品设定性能上限，开源项目降低使用门槛。最终，**混合模式**（核心推理用闭源模型 + 搜索/提取用开源组件）可能成为企业的最优选择。

### 6.3 多模态 Deep Research 的兴起

当前的 Deep Research 几乎完全依赖文本信息。但真实世界的知识远不止文本：

- **图表**：论文和报告中的性能对比图、架构图、统计图表包含了大量量化信息
- **视频**：技术演讲、产品演示、教程视频是重要的知识来源
- **代码**：GitHub 仓库中的源码和 Issue 讨论是技术理解的直接来源
- **音频**：播客、会议录音等音频内容也包含重要信息

多模态 Deep Research 系统将能够整合这些不同模态的信息，形成更全面的研究结论。这需要多模态 LLM 的突破，也需要跨模态检索和对齐技术的进步。

### 6.4 监管与伦理

随着 Deep Research 系统的能力不断增强，监管和伦理问题将日益凸显：

- **信息溯源**：如何确保报告中的每个观点都可以追溯到可靠的来源？
- **版权合规**：自动抓取和整合网页内容是否侵犯版权？
- **数据隐私**：研究过程中处理的敏感信息如何保护？
- **偏见控制**：如何避免系统放大搜索引擎和训练数据中的偏见？
- **透明度**：用户是否知道报告是如何生成的？置信度如何？

这些问题没有简单的技术答案，需要技术界、法律界和监管机构的协作。但有一点是确定的：**缺乏伦理框架的 Deep Research 系统，能力越强，风险越大**。

---

## 7. 结语

Deep Research 代表了大模型应用的一个重要方向：从"被动回答"到"主动研究"，从"对话工具"到"自主工作单元"。它不仅是搜索技术的升级，更是 AI Agent 能力的一次质变。

回顾本文的论述，我们可以总结出 Deep Research 的几个核心要点：

1. **架构决定上限**：四层架构（感知、认知、决策、执行）和五组件（Planner、Searcher、Reader、Critic、Writer）提供了理解 Deep Research 系统的通用框架。
2. **RL 是关键使能技术**：GRPO 等强化学习算法让模型学会策略性决策，突破了 Prompting 和 SFT 的能力上限。
3. **开源生态正在加速**：从 Single-Agent 到 Multi-Agent，从 Prompting 到 RL 训练，开源项目的迭代速度令人瞩目。
4. **挑战依然严峻**：幻觉控制、成本优化、长程上下文管理、多模态处理——每一个都是需要持续攻坚的技术难题。

Deep Research 的终极愿景，是让每一个人都拥有一个"随叫随到的研究员"——一个能理解你的问题、自主探索答案、并以清晰的方式呈现的 AI 伙伴。这个愿景尚未实现，但路径已经清晰。

---

## 参考文献

### 论文

| # | 论文 | 链接 |
|---|---|---|
| 1 | Tongyi DeepResearch Technical Report | https://arxiv.org/abs/2510.24701 |
| 2 | ReSum: Context Summarization for Long-horizon Search Agents | https://arxiv.org/abs/2509.13313 |
| 3 | WebWeaver: Dynamic Outline-Driven Open-Ended Research | https://arxiv.org/abs/2509.13312 |
| 4 | WebResearcher: Multi-Agent Reasoning for Deep Research | https://arxiv.org/abs/2509.13309 |
| 5 | WebSailor-V2: Scalable Training with Synthetic Data | https://arxiv.org/abs/2509.13305 |
| 6 | Search-R1: End-to-End RL for Search and Reasoning | https://arxiv.org/pdf/2503.09516 |
| 7 | Open Deep Search | https://arxiv.org/pdf/2503.20201 |
| 8 | WebThinker | https://arxiv.org/abs/2504.21776 |
| 9 | ReSearch: GRPO-Optimized Search Strategy | https://arxiv.org/pdf/2503.19470 |
| 10 | ZeroSearch: Train Search Without Real Search | https://arxiv.org/pdf/2505.04588 |

### 综述论文

| # | 论文 | 链接 |
|---|---|---|
| 1 | From Web Search towards Agentic Deep Research | https://arxiv.org/abs/2506.18959 |
| 2 | Deep Research Agents: A Systematic Examination And Roadmap | https://arxiv.org/abs/2506.18096 |
| 3 | A Comprehensive Survey of Deep Research: Systems, Methodologies, and Applications | https://arxiv.org/abs/2506.12594 |

### 其他论文

| # | 论文 | 链接 |
|---|---|---|
| 1 | ReAct: Synergizing Reasoning and Acting in Language Models | https://arxiv.org/abs/2210.03629 |
| 2 | Tree of Thoughts: Deliberate Problem Solving with LLMs | https://arxiv.org/abs/2305.10601 |
| 3 | GAIA: A Benchmark for General AI Assistants | https://arxiv.org/abs/2311.12983 |
| 4 | WebArena: A Realistic Web Environment for Building Autonomous Agents | https://arxiv.org/abs/2307.13854 |

### 开源项目

| # | 项目 | Stars | 链接 |
|---|---|---|---|
| 1 | bytedance/deer-flow | 64.5k | https://github.com/bytedance/deer-flow |
| 2 | anthropics/claude-cookbooks | 42.1k | https://github.com/anthropics/claude-cookbooks |
| 3 | assafelovic/gpt-researcher | 26.8k | https://github.com/assafelovic/gpt-researcher |
| 4 | dzhng/deep-research | 18.8k | https://github.com/dzhng/deep-research |
| 5 | Alibaba-NLP/DeepResearch | 18.8k | https://github.com/Alibaba-NLP/DeepResearch |
| 6 | google-gemini/gemini-fullstack-langgraph-quickstart | 18.1k | https://github.com/google-gemini/gemini-fullstack-langgraph-quickstart |
| 7 | langchain-ai/open_deep_research | 11.3k | https://github.com/langchain-ai/open_deep_research |
| 8 | langchain-ai/local-deep-researcher | 9.1k | https://github.com/langchain-ai/local-deep-researcher |
| 9 | zilliztech/deep-searcher | 7.8k | https://github.com/zilliztech/deep-searcher |
| 10 | MervinPraison/PraisonAI | 7.0k | https://github.com/MervinPraison/PraisonAI |

### 工具与服务

| # | 工具 | 链接 |
|---|---|---|
| 1 | MCP (Model Context Protocol) | https://modelcontextprotocol.io/ |
| 2 | Jina Reader | https://jina.ai/reader/ |
| 3 | Firecrawl | https://www.firecrawl.dev/ |
| 4 | SearXNG | https://docs.searxng.org/ |
| 5 | Tavily Search | https://docs.tavily.com/ |

---

*本文草稿，待确认后方可定稿。*
