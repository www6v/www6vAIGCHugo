---
title: Agent Memory 实践
weight: 1
---

# Agent Memory 实践

> 本文是《大模型 Agent 和应用》Agent Memory 系列的**下篇**，聚焦关键技术、工业实践与选型指南。上篇《Agent Memory 原理》可参见 `chapter-part1-principles.md`。

---

## 导读

上篇《Agent Memory 原理》我们讨论了 Agent Memory 的定义、生命周期和五种经典架构模式。本文将进入**实践层面**：

1. 记忆压缩、向量检索、跨模态记忆等**关键技术**是如何工作的？
2. MemOS、Mem0、Letta、OpenViking、Claude Code、OpenClaw、Hermes Agent 等真实系统是怎么做的？
3. 为什么"本地文件系统存储"正成为 Agent Memory 的主流选择？
4. 如何为你的 Agent 选择合适的 Memory 方案？

---

## 4. 关键技术深潜

### 4.1 记忆压缩（Prompt Compression）

#### 4.1.1 为什么需要压缩

token 经济学是驱动记忆压缩的核心力量。如果每次请求都携带完整的记忆，成本会随对话轮数呈 O(n²) 增长。压缩的本质是**用更少的 token 承载相同的信息量**。

#### 4.1.2 LongLLMLingua：问题感知的压缩

LongLLMLingua（arXiv:2310.06839）的核心洞察是：**不是所有文本都同等重要——重要性取决于你要回答什么问题**。

传统摘要压缩的做法是把文本无差别地压缩到固定长度。LongLLMLingua 则根据用户的问题，保留与问题最相关的信息，丢弃无关内容。它通过计算每个 token 相对于问题的"困惑度增益"（perplexity gain）来决定保留哪些 token。

实验数据显示，LongLLMLingua 可以将 40K-100K 的上下文压缩到原来的 1/4 到 1/8，同时保持甚至提升下游任务的性能。

#### 4.1.3 Claude Code 的 compact_boundary 策略

Claude Code 采用了一种不同的压缩策略：**上下文压缩边界**（compact_boundary）。Agent 在对话过程中标记一个边界，边界之前的内容可以被压缩为摘要，边界之后的内容保留原文。

这种策略的优势在于：

- **Agent 自主决定**压缩时机，而不是外部系统强制截断
- 保留了最近对话的完整细节（边界后的原文），同时压缩了历史信息
- 通过 `resume/fork` 操作，可以在压缩后恢复到特定时间点的状态

#### 4.1.4 压缩的代价

压缩不是免费的午餐。信息丢失是不可避免的——关键问题在于丢失的是什么、是否可以接受。量化研究表明：

- 对于问答类任务，压缩到原始长度的 1/4 时，性能损失通常小于 5%
- 对于需要精确引用的任务（如法律分析），任何压缩都可能导致关键细节丢失
- 问题感知压缩（LongLLMLingua）的丢失率显著低于无差别压缩

### 4.2 向量检索与 Embedding

#### 4.2.1 Embedding 的选择

Embedding 模型是向量检索的质量天花板。常见的选择包括：

| 模型 | 维度 | 特点 | 适用场景 |
|------|------|------|---------|
| OpenAI text-embedding-3-large | 3072 | 通用性强，质量高 | 通用记忆检索 |
| text-embedding-3-small | 1536 | 性价比好 | 成本敏感的场景 |
| BGE-M3 | 1024 | 多语言支持好 | 中文记忆检索 |
| Jina Embeddings | 768-1024 | 开源可自托管 | 隐私敏感的场景 |

维度越高，语义区分能力越强，但存储和计算成本也越高。

#### 4.2.2 向量数据库选型

| 数据库 | 部署方式 | 特点 | 适用场景 |
|--------|---------|------|---------|
| FAISS | 本地库 | Meta 开源，性能极高，但无持久化 | 快速原型、嵌入在应用中的检索 |
| Milvus | 服务化 | 功能最全，支持分布式 | 大规模生产环境 |
| Qdrant | 服务化 | Rust 实现，性能好，API 友好 | 中等规模的记忆检索 |
| Chroma | 本地/服务化 | 轻量，开发友好 | 开发和小规模部署 |
| Pinecone | 托管服务 | 零运维，按量计费 | 不想管理基础设施的团队 |
| LanceDB | 嵌入式 | 零依赖，本地持久化 | OpenClaw 等轻量级 Agent |

#### 4.2.3 混合检索

单一检索方式往往不够。混合检索结合了多种方式：

- **BM25 + 向量**：BM25 擅长精确关键词匹配，向量擅长语义理解。两者结合可以覆盖更多查询模式。
- **向量 + 结构化索引**：先用结构化条件过滤（如"只搜索项目 X 的记忆"），再用向量相似度排序。

### 4.3 记忆的元数据与治理

记忆不只是内容本身，还需要元数据来管理：

- **时间戳**：记忆创建和最后访问的时间，用于 TTL 和衰减策略
- **来源**：记忆是从哪个会话、哪个 Agent 产生的，用于审计和隔离
- **可信度标签**：记忆的可信程度（如"用户明确说的"vs "Agent 推测的"）
- **版本控制**：当记忆被更新时，保留历史版本以支持回滚

多用户场景下，记忆隔离至关重要。一个用户的记忆绝不能被另一个用户的 Agent 检索到。这需要在存储层和检索层都实现用户级别的隔离。

隐私与合规方面，GDPR 的"被遗忘权"（Right to be Forgotten）要求系统能够彻底删除特定用户的所有记忆数据。这在向量数据库中尤其具有挑战性——向量嵌入本身无法直接映射回原始文本，删除一个用户的记忆可能需要重新索引整个数据库。

### 4.4 跨模态记忆

记忆不只是文本。Agent 可能需要记住：

- **图像**：用户发送的截图、设计稿、照片
- **音频**：语音指令、会议录音
- **视频**：教程、演示

Mem0（arXiv:2504.19413）已经展示了多模态记忆的实践——通过多模态 Embedding 模型，将文本、图像、音频映射到统一的向量空间，实现跨模态检索。例如，你可以搜索"上次我发的那张系统架构图"，系统通过语义相似度找到对应的图像记忆。

多模态记忆的挑战在于：

- 多模态 Embedding 模型的质量和一致性仍在快速演进中
- 图像和音频的存储成本远高于文本
- 跨模态检索的评估标准不如纯文本检索成熟

---

## 5. 工业实践：真实系统是怎么做的

理论框架和架构模式讨论完了。现在我们来看八个真实系统是如何实现 Agent Memory 的。每个系统都代表了不同的设计哲学和技术路径。

### 5.1 MemOS — 把记忆当操作系统资源

MemOS（arXiv:2505.22101, arXiv:2507.03724）提出了一个雄心勃勃的目标：**将 Memory 提升为 LLM 的一等公民资源**，就像 CPU、内存、磁盘是操作系统的一等公民一样。

#### 三种记忆类型的统一

MemOS 将记忆分为三种类型，并统一管理：

| 类型 | 类比 | 内容 | 特点 |
|------|------|------|------|
| **参数记忆（Parametric Memory）** | 固件/BIOS | 模型参数、Adapter 权重 | 最紧凑，更新成本高 |
| **激活记忆（Activation Memory）** | 缓存/寄存器 | 推理过程中的中间状态 | 快速但短暂 |
| **明文记忆（Plaintext Memory）** | 文件系统中的文件 | 文本、结构化数据 | 可读可编辑，持久化 |

#### MemCube：统一抽象

MemOS 的核心抽象是 **MemCube**——一个封装记忆内容及其元数据（来源 provenance、版本 versioning）的统一容器。所有三种类型的记忆都通过 MemCube 来表示和操作，支持异构记忆的追踪、融合和迁移。

### 5.2 Mem0 — 面向开发者的 Memory 基础设施

Mem0（YC-backed, arXiv:2504.19413）的定位是：**为 AI 应用提供开箱即用的 Memory 基础设施**，让开发者不需要自己搭建记忆系统。

#### Add → Learn → Retrieve 三步流程

1. **Add**：将对话内容发送给 Mem0。
2. **Learn**：Mem0 自动提取增量记忆，更新已有的记忆。使用 LLM 来判断哪些信息是新的、哪些是重复的、哪些是冲突的。
3. **Retrieve**：在需要时检索相关记忆，注入到 Agent 的 prompt 中。

#### 分层蒸馏压缩

Mem0 采用了一种称为"单层 pass 分层蒸馏"（Single-pass Hierarchical Distillation）的压缩引擎。它将冗长的聊天历史在一次处理中压缩为紧凑的结构化记忆，而不是逐轮处理。这降低了延迟和 token 消耗。

#### 跨 Session 和跨 Agent 记忆

Mem0 的独特能力是：记忆不仅跨 session 持久化，还可以跨 agent 共享。同一个用户的记忆可以被多个不同的 Agent 使用。这在 Healthcare、Education、CRM 等场景中特别有用——用户的偏好和历史不需要重复建立。

### 5.3 Letta — 从学术论文到产品

Letta 诞生于 UC Berkeley Sky Computing Lab 的 MemGPT 研究（arXiv:2310.08560），是将学术论文产品化的典型案例。

#### Memory Palace

Letta 的核心特性是 **Memory Palace**（记忆宫殿）——一个可透明查看和修改的 Agent 记忆界面。开发者可以直接看到 Agent "记住"了什么，并且可以手动编辑记忆内容。

#### 跨模型记忆迁移

Letta 支持将记忆从一个模型迁移到另一个模型。这意味着你可以在 GPT-4 上训练的 Agent 记忆，迁移到 Claude 上继续使用。

#### 后台 Memory Subagent

Letta 引入了一个后台的 Memory Subagent，专门负责改进 Agent 的 prompt、上下文和技能。

### 5.4 OpenViking — 专为 Agent 设计的 Context Database

OpenViking（字节跳动/火山引擎开源）的定位非常明确：**专为 AI Agent 设计的 Context Database**。

#### 文件系统范式

OpenViking 的核心创新是采用**文件系统范式**来统一管理 Agent 所需的记忆、资源和技能：

```
/memory/          # 记忆：交互历史、用户偏好、经验教训
/resources/       # 资源：文档、代码片段、参考链接
/skills/          # 技能：工具定义、工作流模板
```

#### L0/L1/L2 三级上下文

- **L0**：全局上下文，每次会话启动时加载
- **L1**：当前子域上下文，切换到相关子域时加载
- **L2**：细粒度上下文，按需检索加载

#### 可视化检索轨迹

OpenViking 支持**目录检索轨迹可视化**，让你可以清晰地看到检索路径。这种可观测性对于调试和优化检索逻辑至关重要。

#### 自动会话管理

自动压缩对话内容、资源引用和工具调用，提取长期记忆，实现上下文的自迭代。

### 5.5 阿里 ReMe — 检索增强的长周期记忆

阿里 ReMe 是 Retrieval-enhanced Memory 系统的代表，面向长周期 Agent 的持续记忆管理。

核心设计包括：

- **基于检索增强的记忆存取架构**：将交互历史转化为可检索的结构化记忆单元
- **自动摘要、去重和冲突解决**：减少记忆冗余，处理记忆冲突
- **与通义系列大模型深度集成**：在电商客服、智能助手等场景中有大规模应用验证

### 5.6 Claude Code 的 Memory — Session 持久化方案

#### 核心机制

| 机制 | 功能 |
|------|------|
| **Session 落盘** | 完整对话历史自动写入磁盘 |
| **resume** | 按 session ID 恢复特定历史会话 |
| **fork** | 复制当前会话创建分支 |
| **compact_boundary** | 标记上下文压缩边界 |
| **checkpoint** | 文件级手动快照 |

#### 局限

Claude Code 不内置长期记忆层。Session 持久化解决了"恢复对话"的问题，但没有解决"从经验中学习"的问题。



### 5.7 OpenClaw 的 Memory — 文件持久化的轻量方案

#### Markdown 文件分层

| 文件 | 角色 | 类比 |
|------|------|------|
| **MEMORY.md** | 长期记忆 | `/etc/profile` — 全局配置 |
| **USER.md** | 用户画像 | `/home/<user>/profile` |
| **TOOLS.md** | 工具文档 | `/usr/share/doc` |
| **SOUL.md** | 人格配置 | Agent 的"人格配置文件" |
| **memory/YYYY-MM-DD.md** | 每日日志 | `/var/log/` |

#### Heartbeat 机制

定期扫描原始日志，提取关键洞察，更新 MEMORY.md，淘汰过时信息——相当于后台 daemon 定期执行垃圾回收和日志轮转。

#### LanceDB 向量检索

对于需要语义检索的场景，OpenClaw 集成了 LanceDB 作为辅助的向量索引层，零依赖、嵌入式、本地持久化。



### 5.8 Hermes Agent 的 Memory — SQLite + 闭环学习

- **SQLite + FTS5** 全文索引的轻量级会话存储，无需外部向量数据库
- **Honcho 用户建模**：通过辩证交互主动提问验证和更新对用户的理解
- **闭环学习系统**：Agent 产出训练数据，用于后续 RL 训练
- **记忆字符上限**：强制 Agent 学会"什么值得记"

---

## 5.9 跨系统洞见：本地文件系统存储已成为 Agent Memory 的主流选择

在分析了八个系统之后，一个模式逐渐浮现：**将 Agent 记忆落盘到本地文件系统，正从一种朴素的实现选择演变为行业共识**。

> **核心洞察**：「Agent Memory 使用本地文件系统存储」并非因为"没有更好的方案"，而是因为文件系统在 Agent 场景下恰好匹配了一系列根本性的设计需求。OpenClaw、Claude Code、OpenViking 三个系统从不同路径得出了相同的设计选择——而非依赖外部数据库或云服务。其中 OpenViking 更是将这一理念产品化，直接以**文件系统范式**作为 Context Database 的核心设计哲学。

### 5.9.1 为什么是本地文件系统？

| 驱动因素 | 说明 |
|---------|------|
| **零依赖部署** | 无需搭建向量数据库、无需配置云服务，开箱即用 |
| **Agent 天然拥有文件读写权限** | 文件系统是 Agent 的"原生接口" |
| **透明可审计** | 直接打开文件查看 Agent 记住了什么 |
| **版本控制友好** | Git 天然支持 Markdown/JSON 的版本管理 |
| **低成本** | 不产生额外云服务费用，适合自托管 |

### 5.9.2 OpenClaw — Markdown 文件分层架构

```
~/.openclaw/workspace/
├── MEMORY.md          # 长期记忆（精炼）
├── USER.md            # 用户画像
├── TOOLS.md           # 工具备忘
├── SOUL.md            # 人格配置
├── memory/
│   ├── 2026-04-29.md  # 今天的原始日志
│   └── 2026-04-28.md  # 昨天的原始日志
```

### 5.9.3 Claude Code — Session 持久化落盘

每个 session 的完整对话历史自动写入磁盘。`resume/fork` 本质上是文件级恢复和分支，`compact_boundary` 标记压缩边界，`checkpoint` 是手动快照。

### 5.9.4 OpenViking — Context Database 的文件系统范式

OpenViking 代表了最"激进"的路径：**直接以文件系统范式作为 Context Database 的设计基础**。记忆、资源、技能按目录层级组织，检索通过目录递归实现，加载通过 L0/L1/L2 三级按需完成。

### 5.9.5 三种系统的对比分析

| 维度 | OpenClaw | Claude Code | OpenViking |
|------|----------|-------------|------------|
| **存储格式** | Markdown 文件 | Session 日志文件 | 文件系统层级 + 内部索引 |
| **零依赖** | ✅ 完全零依赖 | ✅ 完全零依赖 | ⚠️ 需 Embedding/VLM |
| **版本控制** | ✅ Git | ✅ Git | ⚠️ 部分支持 |
| **长期记忆** | ✅ 内置 | ❌ 需外部实现 | ✅ 内置 |
| **语义检索** | ✅ LanceDB | ❌ 无 | ✅ 目录递归 |
| **定位** | 轻量级自托管 | 编码 Agent 会话管理 | 专用 Context Database |

---

## 6. 实践指南：如何为你的 Agent 选择 Memory 方案

### 6.1 需求分析框架

在做技术选型之前，先回答四个问题：

1. **记忆规模预期**：100 条以内→ JSON/Markdown 文件；1 万条左右→ 向量数据库或分层文件系统；100 万条以上→ 分布式向量数据库。

2. **检索延迟要求**：实时（< 100ms）→ 预加载或嵌入式索引；秒级可接受→ 向量数据库网络调用。

3. **一致性需求**：强一致 vs 最终一致。

4. **隐私与合规要求**：敏感数据优先本地文件系统；GDPR 合规需要"被遗忘权"的完整删除。

### 6.2 从简单开始：渐进式 Memory 架构

| 阶段 | 方案 | 何时升级 |
|------|------|---------|
| **阶段一** | 对话历史 + 简单摘要 | 当对话超过上下文窗口时 |
| **阶段二** | 引入向量检索（FAISS/Chroma） | 当简单搜索找不到相关内容时 |
| **阶段三** | 分层记忆 + 压缩（Letta/OpenViking 模式） | 当记忆规模达万级且 token 成本过高时 |
| **阶段四** | 学习型记忆（Reflexion/Hermes 模式） | 当 Agent 需要从失败经验中自动学习时 |

> **关键原则**：每个阶段的升级都应该是由"实际问题"驱动的，而不是"未来可能的需求"。过度设计记忆系统的代价，往往比设计不足更大。

### 6.3 常见陷阱

#### 6.3.1 过度记忆：什么都记

**解决策略**：在写入环节引入价值判断。Mem0 用 LLM 自动判断，OpenClaw 用 Heartbeat 蒸馏，Hermes 用字符上限强制 Agent 做选择。

#### 6.3.2 记忆孤岛：不同 session 的记忆无法打通

**解决策略**：使用跨 session 持久化的记忆系统（如 Mem0、Letta、OpenClaw 的 MEMORY.md）。

#### 6.3.3 僵尸记忆：过时信息污染 Agent 判断

**解决策略**：引入记忆更新机制和 TTL 策略。Mem0 在 Learn 阶段自动处理冲突，OpenClaw 的 Heartbeat 淘汰过时信息。

#### 6.3.4 隐私泄漏

**解决策略**：存储层按用户分目录，检索层过滤不属于当前用户的记忆。

---

## 7. 开放问题与未来趋势

### 7.1 记忆的可解释性

目前的记忆系统大多无法回答"你为什么记得这件事？"。未来的记忆系统需要提供完整的"记忆溯源链"。

### 7.2 记忆的跨 Agent 共享

不同 Agent 对同一条记忆的理解和使用方式是否应该不同？客服 Agent 和技术 Agent 对同一个用户的记忆，关注点可能完全不同。

### 7.3 记忆的主动生成：Agent 能否"做梦"？

Agent 能否在空闲时间主动"做梦"——回顾交互、提取经验教训、整合碎片记忆？OpenClaw 的 Heartbeat 机制已经展现了这一方向的雏形。

### 7.4 记忆与身份的边界

记忆改变 = 人格改变？当记忆的变更累积到一定程度，我们还是同一个 Agent 吗？

### 7.5 长期记忆的安全

记忆系统越强大，安全风险越高。需要多层次的防御：写入层可信度验证、存储层加密和访问控制、检索层完整性校验。

---

## 本文小结

### 核心要点

1. **本地文件系统正成为主流选择**。OpenClaw、Claude Code、OpenViking 三个系统从不同路径得出了相同的设计选择——将记忆落盘到本地文件系统。零依赖部署、透明可审计、版本控制友好是核心驱动力。
2. **没有银弹，只有权衡**。扁平列表简单但不可扩展，分层记忆强大但复杂，检索增强高效但可能丢失语义，学习型记忆最具前瞻性但实现成本最高。选择取决于你的具体场景。
3. **渐进式架构是最稳妥的路径**。从最简单的方案开始，根据实际需求逐步演进——每个阶段的升级都应该由"实际问题"驱动。
4. **遗忘和记忆同等重要**。TTL、重要性衰减、容量上限是三种基本的遗忘策略，没有遗忘机制的记忆系统最终会因为噪声膨胀而失效。

## 参考文献

[1] Jiang, H., et al. (2023). *LongLLMLingua: Accelerating and Enhancing LLMs in Long Context Scenarios via Prompt Compression*. arXiv:2310.06839.（问题感知的 Prompt 压缩：根据问题重要性保留/丢弃 token，可将 40K-100K 上下文压缩到 1/4 到 1/8）

[2] Shinn, N., et al. (2025). *MemOS: A Memory OS for AI Systems*. arXiv:2505.22101.（将 Memory 提升为 LLM 一等公民资源，提出 MemCube 统一抽象，管理参数/激活/明文三种记忆类型）

[3] Shinn, N., et al. (2025). *MemOS: Unified Memory Architecture for Large Language Models*. arXiv:2507.03724.（MemOS 架构的后续演进，异构记忆的追踪、融合和迁移）

[4] Chhikara, A., et al. (2025). *Mem0: Building Production-Ready Long-Term Memory Infrastructure for AI Agents*. arXiv:2504.19413.（面向开发者的 Memory 基础设施：Add → Learn → Retrieve 三步流程，单层 pass 分层蒸馏压缩，跨 Agent 记忆共享）

[5] Shinn, N., & Labash, B. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning*. arXiv:2303.11366.（语言层面的反射式自我修正：失败 → 生成反思 → 存储为记忆 → 下次检索 → 修正策略）

[6] Wang, Y., et al. (2023). *CoALA: A Survey on the Coordinated Learning and Acting of LLM-Based Agents*. arXiv:2309.02427.（LLM Agent 学习框架：闭环学习系统，记忆作为训练数据来源）

[7] Xiong, F. (2025). *从上下文到经验资产：OpenClaw 热潮下的 Agent 记忆系统工程实践*. InfoQ.（OpenClaw Heartbeat 机制与 LanceDB 向量检索的工程实践分析）

[8] Snowan. *Deep Dive: Claude Code Memory Architecture*. GitBook. https://snowan.gitbook.io/study-notes/ai-blogs/claude-code-memory-architecture.（Claude Code 的 Session 落盘、resume/fork、compact_boundary 机制分析）

[9] Snowan. *Deep Dive: How OpenClaw's Memory System Works*. GitBook. https://snowan.gitbook.io/study-notes/ai-blogs/openclaw-memory-system-deep-dive.（OpenClaw Markdown 文件分层架构与 Heartbeat 记忆蒸馏机制分析）

[10] 火山引擎开源社区. *OpenViking: An AI Agent Context Database*. GitHub: volcengine/OpenViking.（专为 AI Agent 设计的 Context Database，采用文件系统范式统一管理记忆/资源/技能，支持 L0/L1/L2 三级上下文和目录递归检索）

---

*本文是《大模型 Agent 和应用》Agent Memory 系列的下篇。上篇《Agent Memory 原理》可参见 `chapter-part1-principles.md`。*
