---
bookHidden: true
---


# Deep Research 文章大纲

## 1. 引言：什么是 Deep Research？
- 1.1 定义与演进（从 Web Search → Agentic Search → Deep Research）
- 1.2 与传统搜索/问答的本质区别（单次检索 vs 多轮自主探索）
- 1.3 为什么 Deep Research 是大模型应用的重要分水岭

## 2. 技术架构深度解析
- **2.1 整体架构分层**
  - **感知层（Perception）**：Web 搜索、页面爬取、API 调用、文档解析
    - 搜索引擎接入（Google/Bing/SearXNG 等）
    - 网页内容提取（Readability / Jina Reader / Firecrawl）
    - 多源数据整合（学术论文、新闻、代码仓库、私有知识库）
  - **认知层（Cognition）**：信息理解、推理、知识整合
    - LLM 作为推理引擎（推理模型 vs 通用模型的选择）
    - 信息抽取与结构化（实体、关系、时间线）
    - 知识图谱构建与利用
  - **决策层（Decision）**：策略制定、路径规划、迭代控制
    - 任务分解策略（自上而下 vs 自下而上）
    - 搜索路径优化（广度优先 vs 深度优先 vs 自适应）
    - 资源预算分配（token 预算、时间预算、API 调用限制）
  - **执行层（Action）**：工具调用、搜索执行、报告输出
    - 工具调用框架（Function Calling / MCP / ReAct）
    - 报告生成与引用标注
    - 人机交互（中间状态反馈、方向修正）

- **2.2 核心组件详解**
  - **Planner（规划器）**：任务理解与意图分析、多粒度任务分解、动态规划调整
  - **Searcher（搜索器）**：查询生成与优化、多源检索策略、去重与结果排序
  - **Reader（阅读器）**：页面内容提取与清洗、关键信息定位与摘要、多格式处理
  - **Critic（评估器）**：信息可信度评估、相关性打分、矛盾信息识别与消解
  - **Writer（报告生成器）**：结构化报告模板、引用溯源与证据链、可视化呈现

- **2.3 关键算法模式**
  - ReAct、Plan-and-Solve、Iterative Refinement、Self-Reflection、Tree of Thoughts、Agentic RAG

- **2.4 Agent 架构范式**
  - Single-Agent、Multi-Agent、Hierarchical Multi-Agent

- **2.5 工作流示意**
  ```
  用户提问 → [Planner] 任务拆解 → [Searcher] 多源检索 → [Reader] 内容提取 →
  [Critic] 信息评估 → 信息不足？→ 返回 Planner 迭代 ──→ 信息充足？→
  [Writer] 报告生成 → 输出交付（含引用）
  ```

## 3. 技术挑战与局限
- **3.1 信息幻觉与事实准确性**
  - LLM 幻觉本质原因、搜索结果噪声、交叉验证局限性
  - 解决方案：引用约束、事实核查、置信度标注

- **3.2 搜索深度与 Token 成本的平衡**
  - 搜索深度指数增长 vs token 消耗
  - 信息边际效用递减
  - 成本控制：早停机制、信息饱和度检测、分层预算

- **3.3 长窗口上下文管理**
  - 海量搜索结果超出上下文窗口的挑战
  - 上下文压缩技术（ReSum 方法）
  - 检索式记忆 vs 全量上下文

- **3.4 信息时效性与实时性**
  - 训练数据截止时间限制、Web 搜索实时性
  - 动态更新结果的整合、持续学习机制缺失

- **3.5 多模态信息处理**
  - 当前以文本为主、图表理解瓶颈、视频/音频信息抽取、多模态推理未解难题

- **3.6 评估与基准测试**
  - 缺乏统一评估标准（GAIA、WebArena 等局限性）
  - 研究质量衡量：完整性、准确性、深度、可读性
  - 人工评估 vs 自动评估

- **3.7 安全与伦理**
  - 信息来源偏见与公平性、版权与知识产权、敏感信息合规、滥用风险

## 4. 开源项目深度解析
> 数据来源：[awesome-deepResearch](https://github.com/www6v/awesome-deepResearch)，按 GitHub Stars 降序（完整榜单见参考资料）

- **4.1 分类分析**
  - **End-to-end Agent 类**：deer-flow、gpt-researcher、deep-research、DeepResearch、open_deep_research
    - 特点：开箱即用，一键部署，适合快速上手
    - 典型架构：搜索 → 阅读 → 推理 → 报告
  - **多 Agent 框架类**：claude-cookbooks、PraisonAI
    - 特点：角色分工明确，可扩展性强
    - 典型架构：Planner → Researcher → Writer → Critic 协同
  - **本地/隐私类**：local-deep-researcher、deep-searcher
    - 特点：数据不出本地，适合企业私有数据场景
    - 技术路线：本地 LLM + 本地检索
  - **Tool-Augmented 类**：gemini-langgraph-quickstart
    - 特点：工具增强推理，侧重教学与快速启动
    - 技术路线：Gemini + LangGraph 生态

- **4.2 技术路线对比**
  - **单 Agent vs 多 Agent**
    - 单 Agent：实现简单、成本低、调试方便，但能力受限于单个模型
    - 多 Agent：专业化分工、可扩展，但通信开销大、部署复杂
    - 趋势：从单 Agent 向 Multi-Agent / Hierarchical 演进
  - **Prompting vs SFT vs RL**
    - Prompting：零成本、快速验证，但上限受限于基座模型
    - SFT：稳定提升、可控性强，需要高质量标注数据
    - RL（尤其是 GRPO）：当前主流范式，可突破基座能力上限
  - **开源模型 vs 闭源 API**
    - 开源（Qwen、Llama）：可控、成本低，需要 GPU 资源
    - 闭源（GPT-o1、Claude）：开箱即用，但受限于 API 调用与数据隐私

- **4.3 如何选择**
  - **按场景推荐**
    - 学术研究：gpt-researcher、open_deep_research
    - 商业分析：deer-flow、PraisonAI
    - 个人学习：deep-research、local-deep-researcher
    - 企业私有数据：deep-searcher、local-deep-researcher
  - **按资源选择**
    - 有 GPU：DeepResearch（Qwen）、ZeroSearch
    - 仅 API：gpt-researcher、open_deep_research
    - 零代码：gemini-langgraph-quickstart

## 5. 关键论文解读
> 来自 [awesome-deepResearch](https://github.com/www6v/awesome-deepResearch) Works 部分（完整列表见参考资料）

- **5.1 技术趋势分析**
  - **RL 主导优化范式**
    - GRPO 成为 Deep Research 领域最流行的 RL 算法（ReSearch、ZeroSearch、ReSum 均采用）
    - PPO、Reinforce 等作为补充
    - RL 的核心价值：让模型学会"何时搜索、搜索什么、如何整合"
  - **Qwen 系列模型的统治地位**
    - 10 篇论文中有 8 篇使用 Qwen 作为基座模型
    - Qwen3-30B-A3B 系列成为 Deep Research 事实上的基准模型
    - 原因：MoE 架构的推理效率 + 优秀的中文能力 + 开源生态
  - **合成数据（Synthetic Data）重要性提升**
    - WebSailor-V2 展示合成数据 + 可扩展 RL 的训练范式
    - ZeroSearch 极端方案：完全不需要真实搜索，用模拟数据训练搜索能力

- **5.2 架构演进**
  - **Single-Agent → Multi-Agent**
    - 当前 10 篇论文中 9 篇采用 Single-Agent，仅 WebResearcher 采用 Multi-Agent
    - Single-Agent 仍是主流：实现简单、训练成本低
    - Multi-Agent 代表未来方向：WebResearcher 展示多角色协作释放无界推理能力
  - **阿里系论文的体系化贡献**
    - 5 篇论文形成完整体系：技术报告（DeepResearch）→ 上下文压缩（ReSum）→ 动态大纲（WebWeaver）→ 多 Agent 推理（WebResearcher）→ 合成数据（WebSailor-V2）
    - 覆盖 Deep Research 的全链路优化

- **5.3 关键突破点**
  - **ZeroSearch**（2025/05）：无需真实搜索即可训练搜索能力
    - 核心思想：用 LLM 模拟搜索结果，通过 RL 训练搜索策略
    - 意义：降低训练成本，突破搜索 API 限制
  - **WebWeaver**（2025/09）：动态大纲驱动的开放研究
    - 核心思想：研究过程中动态生成和修改大纲
    - 意义：从线性研究进化为非线性、自适应研究
  - **ReSum**（2025/09）：长程搜索中的上下文压缩
    - 核心思想：通过上下文摘要技术突破长程搜索的窗口限制
    - 意义：解决"越搜越忘"的长程 Agent 核心痛点
  - **Search-R1**（2025/03）：搜索与推理的 RL 统一
    - 核心思想：将搜索工具调用内化为推理过程的一部分
    - 意义：端到端训练，无需启发式规则

## 6. 未来展望
- 6.1 从"研究"到"行动"的闭环
- 6.2 开源生态与闭源产品的竞争格局
- 6.3 多模态 Deep Research 的兴起
- 6.4 监管与伦理

## 7. 结语

---

## 📚 参考资料

**参考仓库**：[awesome-deepResearch](https://github.com/www6v/awesome-deepResearch) — deepResearch 开源项目、论文、综述一站式资源索引

### 开源项目 Top 10 完整榜单

> 数据来源：[awesome-deepResearch](https://github.com/www6v/awesome-deepResearch)，按 GitHub Stars 降序

| # | 项目 | ⭐ Stars | 类型 | 核心特点 |
|---|------|---------|------|---------|
| 1 | [bytedance/deer-flow](https://github.com/bytedance/deer-flow) | 64.5k | End-to-end Agent | 字节跳动开源，长程 SuperAgent（研究+代码+创作），沙箱+子 Agent |
| 2 | [anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks) | 42.1k | 多 Agent 模式 | Anthropic 官方多 Agent 研究系统最佳实践 |
| 3 | [assafelovic/gpt-researcher](https://github.com/assafelovic/gpt-researcher) | 26.8k | End-to-end Agent | 最早的开源 Deep Research Agent，支持任意 LLM |
| 4 | [dzhng/deep-research](https://github.com/dzhng/deep-research) | 18.8k | End-to-end Agent | 迭代式网页研究（搜索+爬取+LLM 合成），Aomni 开源版 |
| 5 | [Alibaba-NLP/DeepResearch](https://github.com/Alibaba-NLP/DeepResearch) | 18.8k | End-to-end Agent | 阿里通义开源，SFT+RL 训练，配套 5 篇技术论文 |
| 6 | [google-gemini/gemini-fullstack-langgraph-quickstart](https://github.com/google-gemini/gemini-fullstack-langgraph-quickstart) | 18.1k | Tool-Augmented | Google Gemini + LangGraph 全栈 Agent 快速启动 |
| 7 | [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research) | 11.3k | End-to-end Agent | LangChain 官方 Deep Research 技术栈 |
| 8 | [langchain-ai/local-deep-researcher](https://github.com/langchain-ai/local-deep-researcher) | 9.1k | 本地/隐私 | 完全本地网页研究与报告生成 |
| 9 | [zilliztech/deep-searcher](https://github.com/zilliztech/deep-searcher) | 7.8k | 本地/隐私 | 面向私有数据的 Deep Search，Python 实现 |
| 10 | [MervinPraison/PraisonAI](https://github.com/MervinPraison/PraisonAI) | 7.0k | Multi-Agent 框架 | 生产级多 Agent 栈，内置 Deep Research 流程 |

### 关键论文 Top 10 完整列表

> 数据来源：[awesome-deepResearch](https://github.com/www6v/awesome-deepResearch) Works 部分

| # | 论文 | 日期 | 基座模型 | 优化方法 | Agent 架构 |
|---|------|------|---------|---------|-----------|
| 1 | [Tongyi DeepResearch Technical Report](https://arxiv.org/abs/2510.24701) | 2025/10 | Qwen3-30B-A3B | SFT + RL | Single-Agent |
| 2 | [ReSum](https://arxiv.org/abs/2509.13313) — Context Summarization for Long-Horizon Search | 2025/09 | Qwen3-30B-A3B-Thinking | GRPO + SFT | Single-Agent |
| 3 | [WebWeaver](https://arxiv.org/abs/2509.13312) — Dynamic Outlines for Open-Ended Research | 2025/09 | Qwen3-30B-A3B | SFT | Single-Agent |
| 4 | [WebResearcher](https://arxiv.org/abs/2509.13309) — Unleashing Unbounded Reasoning | 2025/09 | Qwen3-30B-A3B | RFT + RL | Multi-Agent |
| 5 | [WebSailor-V2](https://arxiv.org/abs/2509.13305) — Synthetic Data + Scalable RL | 2025/09 | Qwen3-30B-A3B | SFT + RL | Single-Agent |
| 6 | [Search-R1](https://arxiv.org/pdf/2503.09516) — Reason + Search with RL | 2025/03 | Qwen2.5-7B | RL(PPO, GRPO) | Single-Agent |
| 7 | [Open Deep Search](https://arxiv.org/pdf/2503.20201) — Open-source Reasoning Agents | 2025/03 | Llama3.1-70B / Deepseek-R1 | Prompting | Single-Agent |
| 8 | [WebThinker](https://arxiv.org/abs/2504.21776) — LLMs with Deep Research Capability | 2025/04 | GPT-o1 / Deepseek-R1 | RL(DPO) | Single-Agent |
| 9 | [ReSearch](https://arxiv.org/pdf/2503.19470) — Reasoning with Search via RL | 2025/03 | Qwen2.5-7B/32B | RL(GRPO) | Single-Agent |
| 10 | [ZeroSearch](https://arxiv.org/pdf/2505.04588) — Incentivize Search without Searching | 2025/05 | Qwen2.5/Llama3.2 | RL(Reinforce/GRPO/PPO) | Single-Agent |
