---
bookHidden: true
---

# 《大模型 Agent 和应用》— Agent Memory 章节参考文献

---

## 一、学术论文

### 1.1 综述

- **[arXiv:2512.13564]** "Memory in the Age of AI Agents" — 全面梳理 Agent Memory 的形态（Forms）、功能（Functions）和动态演化（Dynamics），定义了 Memory 在 Agent 架构中的系统性地位。
- **[arXiv:2603.07670]** Du, P. "Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers" (2026) — 系统梳理自主 LLM Agent 记忆的机制设计、评估方法和新兴前沿方向，聚焦 Agent 在长周期、多轮次场景下如何构建、维护和利用记忆以克服上下文窗口限制。

### 1.2 开创性工作

- **[arXiv:2304.03442]** Park, J.S. et al. "Generative Agents: Interactive Simulacra of Human Behavior" — 提出 Memory-Reflection-Planning 循环，在虚拟小镇中展示 25 个 Agent 的长期记忆、社交互动和自发行为。
- **[arXiv:2310.08560]** Packer, C. et al. "MemGPT: Towards LLMs as Operating Systems" — 提出虚拟上下文管理（Virtual Context Management），将操作系统分层内存思想引入 LLM，实现文档分析和多轮对话中的长期记忆。后演变为 Letta 项目。
- **[arXiv:2303.11366]** Shinn, N. et al. "Reflexion: Language Agents with Verbal Reinforcement Learning" — 提出语言层面的反射式自我修正，Agent 通过记忆失败教训并生成反思来持续改进策略。
- **[arXiv:2309.02427]** CoALA 框架 — 提出认知架构的统一框架，系统化梳理 Agent 中 Learning 与 Acting 的协同机制。

### 1.3 上下文与检索优化

- **[arXiv:2310.06839]** Jiang, H. et al. "LongLLMLingua: Accelerating and Enhancing LLMs in Long Context Scenarios via Prompt Compression" — 通过问题感知的 prompt 压缩，在长上下文场景下降低 token 消耗同时保持关键信息。
- **[arXiv:2312.03815]** AIOS: LLM Agent Operating System — 将 LLM 视为操作系统的核，提供上下文管理、权限控制、资源调度等 OS 级基础设施。

---

## 二、工业实践

### 2.1 MemOS / MemoryOS

- **[arXiv:2505.22101]** Li, Z. et al. "MemOS: An Operating System for Memory-Augmented Generation (MAG) in Large Language Models" (2025) — 首次将 Memory 提升为 LLM 的一等公民资源，统一参数记忆、激活记忆和明文记忆的表示、组织和治理机制。核心抽象为 MemCube，支持异构记忆的追踪、融合和迁移。
- **[arXiv:2507.03724]** Li, Z. et al. "MemOS: A Memory OS for AI System" (2025) — MemOS 的系统级扩展，将记忆视为可管理系统资源，统一 plaintext、activation-based 和 parameter-level 三种记忆的调度与演化，引入 MemCube 封装记忆内容与元数据（provenance、versioning），支持记忆的组成、迁移和融合。

### 2.2 Mem0

- **[arXiv:2504.19413]** Chhikara, P. et al. "Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory" (2025) — Mem0 团队的官方技术论文，详细阐述 Mem0 架构设计、可扩展长期记忆的构建方法、在生产环境中的实践经验和效果评估。
- **官方网站**: https://mem0.ai/
- **文档**: https://docs.mem0.ai/
- **GitHub**: https://github.com/mem0ai/mem0
- Mem0（YC-backed）提供面向 AI 应用的 Memory 基础设施，核心能力包括：（1）基于对话的增量记忆提取与更新（Add → Learn → Retrieve 三步流程）；（2）单层 pass 的分层蒸馏压缩引擎，自动将聊天历史压缩为紧凑记忆，降低 token 消耗和延迟；（3）跨 session 和跨 agent 的记忆持久化；（4）支持多模态记忆（文本、图像、音频）。在 Healthcare、Education、CRM 等领域有落地案例。提供 Python/Node.js SDK 和云端 API。

### 2.3 Letta（原 MemGPT）

- **[arXiv:2310.08560]** Packer, C. et al. "MemGPT: Towards LLMs as Operating Systems" (2023) — 开创性工作，提出虚拟上下文管理，启发了后续 Letta 项目。
- **官方网站**: https://www.letta.com/
- **GitHub**: https://github.com/letta-ai/letta
- Letta 诞生于 UC Berkeley Sky Computing Lab 的 MemGPT 研究，定位为"Memory-first Agent"平台。核心特性：（1）持久化 Agent，每个 Agent 拥有独立经验和人格，可随交互持续进化；（2）后台 Memory Subagent 自动改进 prompt、上下文和技能；（3）Memory Palace 可透明查看和修改 Agent 记忆；（4）记忆可跨模型和跨提供商迁移（port memory across models）；（5）支持在任意设备上运行，通过 letta server 远程控制。

### 2.4 OpenViking

- **GitHub**: https://github.com/volcengine/OpenViking
- **官方网站**: https://openviking.ai/
- 字节跳动（火山引擎）开源的 AI Agent Context Database，专为 Agent（如 OpenClaw）设计。核心设计哲学：**摒弃传统 RAG 的碎片化向量存储模式，采用文件系统范式统一管理 Agent 所需的记忆、资源和技能**。
  - **文件系统管理范式**：基于文件系统层级结构统一管理记忆（Memory）、资源（Resources）和技能（Skills），解决上下文碎片化问题。
  - **L0/L1/L2 三级上下文加载**：按需加载，显著降低 token 消耗。
  - **目录递归检索**：结合目录定位与语义搜索，实现递归式精准上下文获取，突破扁平检索的局限。
  - **可视化检索轨迹**：支持目录检索轨迹可视化，可观测检索路径，解决传统 RAG 的黑盒问题。
  - **自动会话管理**：自动压缩对话内容、资源引用、工具调用等，提取长期记忆，实现上下文自迭代。
  - **原生支持 OpenClaw 等 Agent 平台**，提供 pip install openviking 一键安装。

### 2.5 阿里 ReMe

- 阿里巴巴推出的 Retrieval-enhanced Memory 系统，面向长周期 Agent 的持续记忆管理。核心设计：（1）基于检索增强的记忆存取架构，将交互历史转化为可检索的结构化记忆单元；（2）支持记忆的自动摘要、去重和冲突解决；（3）与通义系列大模型深度集成，在电商客服、智能助手等场景中有大规模应用验证。

### 2.6 Claude Code 的 Memory 实现

- **官方文档**: https://code.claude.com/docs/en/agent-sdk/sessions
- Claude Code（Anthropic）通过 Session 机制实现对话记忆的持久化管理：（1）自动将 Agent 的完整对话历史（prompt、tool calls、tool results、responses）写入磁盘，支持后续恢复；（2）`resume` 可按 session ID 恢复特定历史会话，`fork` 可复制当前会话创建分支用于探索不同方向；（3）`SystemMessage(subtype="compact_boundary")` 标记上下文压缩边界，压缩后仅保留摘要；（4）支持文件级 checkpoint，在重要操作前手动快照文件系统状态；（5）Memory 能力本身需外部实现（Claude Code 不内置长期记忆层），通过 Session + 外部存储组合实现。
- **[Deep Dive: Claude Code Memory Architecture]** Snowan "Deep Dive: Claude Code Memory Architecture" — 深入解析 Claude Code 的 Memory 架构设计，包括 Session 持久化机制、resume/fork 实现原理、compact_boundary 的上下文压缩策略。（URL: https://snowan.gitbook.io/study-notes/ai-blogs/claude-code-memory-architecture）

### 2.7 OpenClaw 的 Memory 实现

- **官方文档**: https://docs.openclaw.ai/
- **GitHub**: https://github.com/openclaw/openclaw
- OpenClaw 作为自托管 AI 网关，提供多层 Memory 架构：（1）短期记忆基于会话上下文窗口，通过 Agent Loop 内维护对话状态；（2）长期记忆采用文件持久化方案（MEMORY.md、USER.md、TOOLS.md、SOUL.md 等结构化 Markdown 文件），每次会话启动时自动加载；（3）`memory/` 目录按日期存储每日笔记（`YYYY-MM-DD.md`），支持原始日志和精炼长期记忆的分离；（4）Heartbeat 机制定期执行记忆维护——将日记中的关键洞察蒸馏到 MEMORY.md，淘汰过时信息；（5）结合向量数据库（如 LanceDB）实现语义记忆检索。整体设计借鉴操作系统的"缓存 → 主存 → 磁盘"分层策略。
- **[InfoQ]** 熊飞宇（MemTensor 创始人 & CEO）"从上下文到经验资产：OpenClaw 热潮下的 Agent 记忆系统工程实践" — 从工程实践视角分析 OpenClaw 的记忆系统设计，探讨如何将对话上下文转化为可沉淀的经验资产。（URL 待补充）
- **[Deep Dive: How OpenClaw's Memory System Works]** Snowan "Deep Dive: How OpenClaw's Memory System Works" — 深入解析 OpenClaw 的记忆系统架构，包括 Markdown 文件分层、Heartbeat 机制、LanceDB 向量检索等核心组件的设计与实现。（URL: https://snowan.gitbook.io/study-notes/ai-blogs/openclaw-memory-system-deep-dive）

### 2.8 Hermes Agent 的 Memory 实现

- **GitHub**: https://github.com/NousResearch/hermes-agent
- **文档**: https://hermes-agent.docs.nousresearch.com/
- Hermes Agent（NousResearch）的 Memory 系统有以下独特设计：（1）基于 SQLite + FTS5 全文索引的轻量级会话存储，无需外部向量数据库（如 Milvus/Pinecone），对轻量级部署友好；（2）集成 [Honcho](https://github.com/plastic-labs/honcho) 用户建模系统，通过辩证交互（主动提问验证和更新用户理解）构建深层用户画像；（3）闭环学习（Closed Learning Loop）系统：Agent 不仅使用工具，还能产出训练数据，用于后续 RL 训练；（4）通过 MEMORY.md / USER.md 等结构化文件实现跨会话记忆，并设有记忆字符上限强制 Agent 学会"什么值得记"；（5）内置 47 个工具分布在 19 个 toolsets 中，可按平台（CLI vs 消息渠道）动态启用不同记忆相关工具。

---

## 三、删除的条目

以下条目已从本列表中移除：

- ~~LangChain Memory 文档~~ — 理由：LangChain Memory 更多是框架级 API 封装（ConversationBufferMemory、ConversationSummaryMemory 等），缺乏独立的架构创新和系统性设计，其能力已被上述工业实践（Mem0、Letta 等）覆盖和超越。
- ~~LlamaIndex~~ — 理由：LlamaIndex 的核心定位是数据索引和 RAG 管道，而非专门的 Agent Memory 系统。其记忆能力本质上是检索增强生成的变体，与 MemOS、Mem0 等 dedicated memory infrastructure 不在同一设计层面。

---

*最后更新: 2026-04-29*
