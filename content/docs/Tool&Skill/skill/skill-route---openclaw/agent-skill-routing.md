# Agent Skill 路由：如何在大量 Skill 下保证命中率

> 从"全量注入 prompt"到"分层发现 + 多路召回 + 反馈闭环"

---

## 一、问题的本质

Agent 面对大量 Skill 时，核心矛盾是一个无法回避的 trade-off：

```
Skill 描述越长越准
    → Context Window 越大
    → LLM 注意力越分散
    → 命中率越低

Skill 描述越短越省
    → 语义区分度越低
    → LLM 越容易选错
    → 命中率也越低
```

这个矛盾在 MCP 工具选择、Agent 技能路由、Plugin 发现等场景中普遍存在。当 Skill 数量从十几个增长到几百上千时，**"把所有 Skill 描述塞进 prompt"的做法必然失效**。

## 二、分层 Skill 发现架构

核心原则：**不要把所有 Skill 一次性塞给 Agent，而是分层递进。**

```
                    用户请求
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
    全局索引层       分类/领域层      精确匹配层
    (~10 个候选)    (~5 个候选)      (1 个目标)
        │               │               │
        ▼               ▼               ▼
    语义召回         领域过滤         精确描述
    (embedding      (标签/类型       (完整 schema
     粗排)          /命名空间)        + 示例)
```

### 2.1 第一层：全局索引（粗筛）

- 每个 Skill 只有**一行摘要**（~50 tokens）
- 通过 embedding 语义召回 Top-K（比如 10 个）
- 这一步的目的不是选对，而是**不漏**

### 2.2 第二层：分类/领域过滤

- Skill 按领域分组：`database.*`、`git.*`、`deploy.*`
- 根据请求关键词做 namespace 过滤
- 比如请求"查数据库慢查询"，只加载 `database/` 下的 Skill

### 2.3 第三层：精确匹配（细选）

- 对 Top-K 候选加载**完整 Skill 描述**（参数、示例、边界条件）
- LLM 在这个小集合中做最终选择

**效果**：把 1000 个 Skill 的问题，从"一次选 1000"变成"100 粗筛 → 10 精选 → 1 命中"。

## 三、Skill 描述的质量是关键

再好的路由机制，也救不了糟糕的 Skill 描述。高质量 Skill 描述应包含：

```yaml
skill: optimize-slow-query
namespace: database.mysql
description: 分析和优化 MySQL 慢查询，生成 EXPLAIN 报告和优化建议
keywords: [慢查询, slow query, EXPLAIN, 性能优化, 索引]
triggers:
  - "查询很慢"
  - "优化 SQL"
  - "explain"
  - "慢查询分析"
inputs:
  - query: "SQL 语句"
  - table_name: "表名（可选）"
output: "EXPLAIN 分析 + 优化建议"
examples:
  - input: "SELECT * FROM orders WHERE status = 'pending'"
    output: "分析此查询的全表扫描问题，建议添加索引..."
```

### 3.1 关键字段说明

| 字段 | 作用 | 示例 |
|------|------|------|
| **namespace** | 命名空间，用于领域过滤 | `database.mysql` |
| **description** | 一行摘要，用于全局索引 | "分析和优化 MySQL 慢查询" |
| **keywords** | 同义词扩展，解决"用户说法不一" | `[慢查询, slow query, EXPLAIN]` |
| **triggers** | 触发短语，用于精确匹配 | `"查询很慢"`, `"优化 SQL"` |
| **examples** | Few-shot 示例，提升 LLM 理解 | 输入输出对 |

## 四、检索策略组合

单一语义检索不够，需要**多路召回 + RRF 融合**：

| 策略 | 适用场景 | 召回率 | 精确率 |
|------|---------|--------|--------|
| **关键词精确匹配** | 用户明确提到 Skill 名或触发词 | 低 | 极高 |
| **Embedding 语义召回** | 用户描述模糊但意图明确 | 高 | 中 |
| **BM25 关键词检索** | 用户提到技术术语 | 中 | 中高 |
| **领域/namespace 过滤** | 请求有明确领域指向 | — | 提升精确率 |
| **历史使用频率加权** | 常用 Skill 优先 | — | 提升热门场景命中率 |

### 4.1 RRF 融合公式

多路召回结果用 RRF（Reciprocal Rank Fusion）融合，比简单加权平均更鲁棒：

```
score(skill) = Σ (1 / (rank_i + k))
```

其中 `rank_i` 是某路检索中的排名，`k` 是平滑常数（通常取 60）。排名的稳定性远高于绝对分数。

## 五、运行时优化

### 5.1 热路径缓存

```
用户请求 → 计算请求 fingerprint → 查缓存 → 命中则直接返回 Skill
```

相同或相似请求直接走缓存，避免重复检索。Fingerprint 可以用请求的 embedding + 用户 ID 生成。

### 5.2 Skill 按需加载（Lazy Loading）

```python
class SkillRouter:
    def __init__(self):
        self.index = load_skill_index()      # 只加载索引
        self.loaded_skills = {}               # 缓存已加载的完整 Skill
    
    def route(self, query):
        # Step 1: 语义粗筛
        candidates = self.semantic_search(query, top_k=10)
        
        # Step 2: 按需加载完整描述
        for skill in candidates:
            if skill.id not in self.loaded_skills:
                self.loaded_skills[skill.id] = self.load_full_skill(skill.id)
        
        # Step 3: LLM 在 Top-K 中做最终选择
        return self.llm_select(candidates)
```

**关键原则**：启动时只加载索引，完整 Skill 描述按需加载。

### 5.3 反馈闭环

```
Agent 选择 Skill → 执行结果 → 用户反馈(显式/隐式) → 更新 Skill 权重
```

| 反馈类型 | 信号 | 动作 |
|---------|------|------|
| 显式负面 | "这不是我想要的" | 降低该 Skill 在此类请求下的权重 |
| 显式正面 | "对的，就是这个" | 提升权重 |
| 隐式正面 | 执行成功 + 用户采纳 | 隐式正向信号，轻微提升 |
| 隐式负面 | 执行失败 / 用户重试 | 隐式负向信号，降低权重 |

## 六、不同方案的命中率对比

| 方案 | 10 个 Skill | 100 个 Skill | 1000 个 Skill |
|------|------------|-------------|--------------|
| 全量注入 prompt | 95%+ | 60% | <30% |
| 语义召回 Top-20 | 95%+ | 80% | 50% |
| 分层架构（索引 → 领域 → 精确） | 95%+ | 90%+ | 80%+ |
| 分层 + RRF + 反馈 | 95%+ | 95%+ | 85%+ |

**关键结论**：

> 超过 50 个 Skill 后，全量注入 prompt 的命中率会断崖式下跌。必须用分层架构。

## 七、与现有记忆方案的关联

### 7.1 TencentDB Agent Memory 的 Skill 管理

TencentDB Agent Memory 通过 `/v3/skill/*` API 管理 Skill，支持：
- 版本管理
- 资源文件绑定
- 搜索路由
- 对话驱动提取（从对话中自动提取 Skill）
- 四级可见性（private / team / restricted / agent）

这种 Skill 管理方式天然适合分层路由：先按可见性过滤，再按 namespace 过滤，最后语义召回。

### 7.2 OpenViking 的 Skill 组织

OpenViking 用 `viking://user/{user_id}/skills/` 组织 Skill，每个 Skill 有 L0/L1/L2 三层：

```
viking://user/alice/skills/
├── code_review/
│   ├── code_review.abstract    # L0: "代码审查工作流"
│   ├── code_review.overview    # L1: "审查流程：1. 检查测试覆盖率..."
│   └── code_review.md          # L2: [完整工作流定义]
├── optimize-slow-query/
│   ├── optimize-slow-query.abstract
│   ├── optimize-slow-query.overview
│   └── optimize-slow-query.md
└── ...
```

**检索时**：先扫 L0 摘要粗筛 → 加载 Top-K L1 概览精选 → 按需加载 L2 全文。这与本文的分层路由思路完全一致。

### 7.3 Mem0 的技能经验提取

Mem0 从对话中提取工具经验，写入结构化路径：

```
viking://user/{user_id}/memories/tools/
```

这些提取的经验本质上就是"动态生成的 Skill"——随着使用不断积累的实践经验。

## 八、工程实践 Checklist

- [ ] 每个 Skill 有唯一的 namespace（`domain.subdomain.skill_name`）
- [ ] 每个 Skill 有 ~50 tokens 的一行摘要（用于全局索引）
- [ ] 每个 Skill 有关键词列表（keywords）和触发短语（triggers）
- [ ] 每个 Skill 有至少一个输入输出示例（examples）
- [ ] 实现三层路由：全局索引 → 领域过滤 → 精确匹配
- [ ] 多路召回 + RRF 融合（至少语义 + BM25 两路）
- [ ] 热路径缓存（相同请求直接返回）
- [ ] Skill 按需加载（Lazy Loading，不启动时全量加载）
- [ ] 反馈闭环（显式 + 隐式信号更新权重）
- [ ] 监控：记录每次 Skill 选择的命中率、用户反馈

---

*2026-08-11*
