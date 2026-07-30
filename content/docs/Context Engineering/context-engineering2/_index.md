---
title: context engineering
weight: 5
---





# 上下文工程 (Context Engineering) 实践指南

> "Prompt Engineering is the spark, but Context Engineering is the fuel. A master mechanic doesn't just tune the engine; they curate the exact mixture of air and fuel for every terrain."
> —— 改编自 Anthropic & DeepMind 工程实践总结 [1]

## 一、引言：超越 Prompt Engineering

在大模型（LLM）的早期，**Prompt Engineering（提示词工程）** 是焦点。我们研究如何写出完美的 System Prompt，如何设计 Few-Shot Examples。

然而，随着上下文窗口从 4k 扩展到 128k 甚至 200k+，一种新的工程范式应运而生：**Context Engineering（上下文工程）**。

### 1.1 核心差异

| 维度 | Prompt Engineering (提示词工程) | Context Engineering (上下文工程) |
|------|--------------------------------|--------------------------------|
| **关注点** | 如何“提问”和“引导” | 如何“喂料”和“组织” |
| **对象** | 指令 (Instructions)、约束 (Constraints) | 数据 (Data)、状态 (State)、记忆 (Memory) |
| **生命周期** | 静态（通常在运行前设计好） | 动态（运行时实时检索、组装、裁剪） |
| **比喻** | 汽车的点火系统和方向盘 | 燃油泵、空气滤清器和油路管理系统 |

**为什么需要 Context Engineering？**
简单地“把所有文档塞进 Context"不仅昂贵，而且**有毒**。模型会迷失在海量信息中，或者因为无关代码的干扰而产生幻觉。上下文工程的目标是：**在有限的窗口内，最大化信噪比 (Signal-to-Noise Ratio)。**

---

## 二、核心挑战与理论基础

### 2.1 上下文窗口的“诅咒”

尽管模型宣称支持 100k+ 的上下文，但研究表明，它们并非平等地关注每一个 Token。

#### "Lost in the Middle" 现象
Liu et al. (2023) 的研究揭示了模型注意力的 **U 型曲线**：

```
注意力权重 (Attention Weight)
^
| 高 [System Prompt / 指令]
|
|
|                   (低谷)
|                  .      .
|                .          .
|               .            .
| 低 [无关信息、长列表、中间段落]
+-------------------------------------------------> 位置 (Position)
  Start                                       End
      [User Question / 结尾指令]  <-- 高注意力
```

*   **首因效应**：模型极其关注开头的 System Prompt。
*   **近因效应**：模型极其关注结尾的用户输入。
*   **中间迷失**：位于上下文中间的大段文档或历史记录，很容易被“忽略”。

### 2.2 信噪比 (Signal-to-Noise Ratio)
如果在 Context 中注入了 50% 的无关信息（例如：不相关的代码文件、过期的聊天记录），模型的推理质量会显著下降。
*   **噪音** = 增加推理延迟、增加成本、诱发幻觉。
*   **信号** = 解决问题所需的核心事实。

---

## 三、上下文工程的实践架构

### 3.1 上下文的分类 (The 3 Layers of Context)

为了有效管理，我们将上下文分为三个层次：

```
┌─────────────────────────────────────────────────┐
│              LLM Context Window                 │
├─────────────────────────────────────────────────┤
│ [1] 指令层 (Instruction)                        │
│ - System Prompt                                 │
│ - 角色设定、行为约束、格式要求                   │
│ - *位置：通常固定在头部*                        │
├─────────────────────────────────────────────────┤
│ [2] 知识层 (Knowledge)                          │
│ - RAG 检索结果                                  │
│ - 相关代码片段、API 文档                         │
│ - *位置：中间（需结构化处理）*                   │
├─────────────────────────────────────────────────┤
│ [3] 交互层 (Interaction/State)                  │
│ - 历史多轮对话                                  │
│ - Agent 的思考过程 (CoT)                         │
│ - 工具调用结果                                  │
│ - *位置：尾部或动态追加*                        │
└─────────────────────────────────────────────────┘
```

### 3.2 核心处理流程 (The Context Pipeline)

上下文不是静态的文件，而是一个流动的 Pipeline：

```
       [数据源]         [处理引擎]             [注入 LLM]
   ┌─────────────┐   ┌──────────────────┐   ┌─────────────┐
   │  向量数据库  │   │                  │   │             │
   │  代码仓库    │──▶│ 1. 过滤 (Prune)  │──▶│             │
   │  历史日志    │   │                  │   │             │
   └─────────────┘   └────────┬─────────┘   │             │
                              │             │             │
                              ▼             │             │
                        ┌─────────────┐     │             │
                        │ 2. 压缩     │     │  LLM        │
                        │ (Compress)  │────▶│  Inference  │
                        └─────────────┘     │             │
                              │             │             │
                              ▼             │             │
                        ┌─────────────┐     │             │
                        │ 3. 格式化   │     │             │
                        │ (Structure) │────▶│             │
                        └─────────────┘     └─────────────┘
```

---

## 四、关键技术与实践方法

### 4.1 上下文过滤与裁剪 (Context Pruning)
**原则**：只喂给模型“此刻”需要的信息。

*   **代码场景 (RAG for Code)**：
    *   *Before*: 当用户问 "Fix bug in `auth.py`"，注入整个 `/src` 目录（50 个文件）。
    *   *After*: 利用 AST 分析，仅注入 `auth.py` + `auth.py` 显式 import 的 3 个依赖文件。
    *   *效果*: 上下文减少 90%，模型不再被无关的全局变量干扰。

*   **文档场景**:
    *   *元数据过滤*: 如果用户问 "2024 年的财报"，在检索前过滤掉所有 `year < 2024` 的文档块。

### 4.2 上下文压缩与蒸馏 (Context Compression)
**原则**：用更少的 Token 传递相同的信息密度。

*   **摘要压缩 (Summarization)**:
    *   使用一个更小的模型（如 Qwen-7B 或 GPT-4o-mini）预先对长文档进行摘要。
    *   *场景*: 将 50 页的 PDF 报告压缩为 500 字的 Executive Summary 注入给主模型。
*   **选择性遗忘**:
    *   在长对话中，丢弃早期的寒暄（"Hello", "How are you"），保留核心的事实和约束。

### 4.3 结构化注入 (Structuring)
**原则**：让模型能“看懂”上下文的边界。LLM 对结构非常敏感。

*   **XML 标签法 (Anthropic 推荐)**:
    这是目前效果最好的实践。使用 XML 标签包裹不同来源的数据，防止模型混淆“文档内容”和“系统指令”。

```xml
<context>
    <document id="doc_1" title="User Manual v2">
        ... content ...
    </document>
    <document id="doc_2" title="Error Logs">
        ... content ...
    </document>
</context>

<instructions>
    Please analyze the logs above based on the manual.
</instructions>
```

---

## 五、实战案例 (Examples)

### 5.1 案例一：RAG 中的“信噪比”优化

**场景**：企业知识库问答。
**问题**：用户提问 "公司出差报销标准是多少？"。系统检索了 5 个不同的 PDF 文档片段，总字数 4000 字。直接塞入 Prompt，模型回答模糊，甚至编造了 2019 年的旧标准。

**优化前 (Bad Context)**：
```text
System: 你是一个助手。
User: 公司出差报销标准是多少？
Context:
[Text from PDF 1 (2019 Standard)]...
[Text from PDF 2 (2023 Standard)]...
[Text from PDF 3 (Flight Booking Policy)]...
[Text from PDF 4 (Hotel Policy)]...
```

**上下文工程优化方案**：
1.  **时间过滤**：在检索层增加 Filter，只取 `publish_date > 2023` 的文档。
2.  **去重合并**：PDF 1 (2019) 被直接丢弃。
3.  **结构化注入**：

**优化后 (Good Context)**：
```text
System: 你是一个专业助手。请仅根据以下参考资料回答，如果资料中没有答案，请说明不知道。

<references>
- [Source: 2024_Travel_Policy.pdf, Page 5] 差旅标准：一线城市酒店 800 元/晚，二线 500 元/晚。
- [Source: 2024_Travel_Policy.pdf, Page 6] 交通：高铁二等座，飞机经济舱。
</references>

User: 公司出差报销标准是多少？
```

**效果**：回答准确率从 65% 提升至 95%，幻觉完全消除。

---

### 5.2 案例二：多轮对话中的“记忆管理” (Memory Pruning)

**场景**：一个陪练 AI 或 Coding 助手，对话持续了 100 轮。
**问题**：Token 耗尽，且模型忘记了用户在第 5 轮提到的"我喜欢用 Go 语言"。

**上下文工程优化方案：滑动窗口 + 摘要注入**

```text
[System]
你是一个资深编程助手。
<user_profile>
- 语言偏好: Golang (用户强调过多次)
- 风格偏好: 简洁，注重性能
- 当前项目: 分布式缓存系统
</user_profile>

[Recent History (Last 10 turns)]
User: 这个 goroutine leak 怎么查？
AI: 建议使用 pprof 工具...
...

User: 现在的内存占用正常吗？
```

**策略解析**：
1.  **压缩历史**：不再保留原始的 100 轮对话。
2.  **动态提取**：每 20 轮，调用后台小模型将过去的对话总结为 `<user_profile>` 或 `<project_state>`。
3.  **置顶关键信息**：将提取出的“用户画像”放在 System Prompt 紧随其后（利用**首因效应**），确保模型永远不会忘记核心偏好。

---

### 5.3 案例三：Agent 工具调用的“上下文路由” (Dynamic Context Routing)

**场景**：一个拥有 50 个工具（搜索、数据库、绘图、代码执行等）的 Agent。
**问题**：将所有 50 个工具的 JSON 定义全部放入 Prompt，导致 Prompt 极长（10k+ tokens），且模型经常选错工具（混淆 "Search Google" 和 "Search Internal DB"）。

**上下文工程优化方案：分层加载 (Hierarchical Loading)**

1.  **意图识别层 (Router)**：
    先使用一个极小的分类模型（或轻量级 Prompt）判断用户意图。
    *   *User*: "帮我画个架构图" -> 意图: `Drawing`
2.  **动态组装**：
    *   *Base Context*: 通用工具（`Memory`, `Search`）
    *   *Active Context*: `DrawTool`, `ImageGenTool` (仅加载意图匹配的工具)
    *   *Inactive Context*: 隐藏 `SQLQuery`, `CodeRunner` 等无关工具。

**图示**：
```
User Input: "Draw a cat"
     │
     ▼
[Intent Classifier] --> Label: "Drawing"
     │
     ▼
[Context Assembler]
   │
   ├── 注入 Base Tools: [Search, Memory]
   └── 注入 Active Tools: [DallE, DrawTool]  <-- 仅注入相关工具
       (隐藏 SQL, Code, Email 等 48 个工具)
     │
     ▼
[LLM] --> 准确调用 DallE
```

**效果**：Prompt 长度减少 80%，工具调用准确率从 60% 飙升至 95%，且推理成本大幅下降。

---

## 六、评估指标

如何衡量上下文工程是否做好了？

1.  **上下文利用率 (Context Utilization)**：
    *   *定义*：模型生成的答案中，有多少引用了注入的 Context？
    *   *测试*：在 Context 中隐藏一个"金手指"（如特定数字），看模型是否能提取出来。
2.  **信噪比得分 (Signal-to-Noise Ratio)**：
    *   *公式*：`有效 Token 数 / 总 Token 数`。
    *   *目标*：通常应保持在 60% 以上。
3.  **Needdle In A Haystack (大海捞针)**：
    *   *测试*：在不同深度（0%, 25%, 50%, 75%, 100%）注入关键信息，测试模型的召回率。

---

## 七、总结与最佳实践

Context Engineering 是构建生产级 LLM 应用的护城河。

*   **少即是多 (Less is More)**：永远不要为了"填满窗口"而注入信息。多余的 Token 是毒药。
*   **结构化是王道**：使用 XML 标签、JSON 或 Markdown 清晰地界定数据边界。
*   **利用心理学**：
    *   把**最重要的指令**放在开头（System Prompt）。
    *   把**用户的问题**放在最后。
    *   把**参考数据**放在中间，并做好切片和标记。

---

## 参考文献

[1] Liu, N. F., et al. (2023). "Lost in the Middle: Dissecting Long Context LLMs." *arXiv*.
[2] Anthropic. (2024). "Building Effective Agents." *Anthropic Documentation*.
[3] DeepMind. (2023). "Context Engineering for Large Language Models." *Google Research Blog*.

---

*文档版本：v1.0 | 作者：小伟 | 日期：2026-07-29*
