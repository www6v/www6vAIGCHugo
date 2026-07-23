# 如何提升 RAG 的召回率：方案与实现

> 一套从诊断到实战的系统性指南，覆盖 7 个核心方案，每个方案都包含原理分析 + 代码实现。

## 目录

1. [引言：为什么召回率是 RAG 的生命线](#1-引言为什么召回率是-rag-的生命线)
2. [召回率低的根因诊断](#2-召回率低的根因诊断)
3. [方案一：优化文档切分策略](#3-方案一优化文档切分策略)
4. [方案二：Embedding 模型选型与优化](#4-方案二embedding-模型选型与优化)
5. [方案三：查询重写与增强](#5-方案三查询重写与增强query-transformation)
6. [方案四：混合检索](#6-方案四混合检索hybrid-search)
7. [方案五：元数据过滤与结构化检索](#7-方案五元数据过滤与结构化检索)
8. [方案六：重排序](#8-方案六重排序re-ranking)
9. [方案七：构建评估体系](#9-方案七构建评估体系)
10. [综合实战：端到端高召回率 RAG 管线](#10-综合实战一个端到端的高召回率-rag-管线)
11. [总结与建议路线图](#11-总结与建议路线图)
12. [附录](#附录)

---

# 1. 引言：为什么召回率是 RAG 的生命线

## 1.1 RAG 架构回顾

RAG（Retrieval-Augmented Generation）的核心思想非常直观：**先从知识库中检索相关文档，再把检索结果作为上下文喂给大模型生成答案。**

```
用户 Query → [检索器] → 相关文档 chunks → [LLM] → 最终答案
```

看起来简单，但整个管线的性能取决于最薄弱的一环。实践中最常见的失败模式是：

> **检索器没有找到相关文档 → LLM 拿到的上下文无关 → 生成错误答案**

这就是典型的 **Garbage In, Garbage Out**。

## 1.2 召回率的定义与度量

在信息检索领域，**召回率（Recall）** 的定义是：

> **Recall@K = 检索结果 Top-K 中包含的相关文档数 / 所有相关文档总数**

对于 RAG 场景，我们更关注几个实用指标：

| 指标 | 含义 | 适用场景 |
|------|------|----------|
| **Recall@K** | Top-K 结果中是否命中了答案所在的文档 | 评估检索器覆盖面 |
| **Hit Rate** | 至少有一个相关文档被召回的比例 | 端到端管线评估 |
| **MRR（Mean Reciprocal Rank）** | 第一个相关文档排名的倒数均值 | 评估排序质量 |
| **NDCG@K** | 考虑相关性等级的归一化折损累计增益 | 多级相关性评估 |

## 1.3 召回失败 vs 生成失败

调试 RAG 管线时，一个关键能力是**区分错误来源**：

```
错误答案
├── 召回失败：相关文档根本没被检索到 → 优化检索器
├── 排序失败：相关文档被检索到了但排名太低 → 重排序/调整 K
└── 生成失败：上下文正确但 LLM 回答错误 → Prompt 工程 / 模型替换
```

**本文专注于前两类问题的解决**——如何确保相关文档被找到，并且排在前面。

## 1.4 本文路线图

本文系统性梳理提升 RAG 召回率的 **7 个核心方案**，每个方案都包含：

1. **原理分析**：为什么有效
2. **方案对比**：优缺点和适用场景
3. **代码实现**：可运行的示例代码

这 7 个方案不是互斥的，而是可以**叠加使用**。文章最后一章会给出一个端到端的综合管线，以及各方案的 ROI 对比矩阵。
# 2. 召回率低的根因诊断

在动手优化之前，**先诊断问题出在哪里**。盲目优化某个环节可能事倍功半。以下是导致召回率低的六大根因。

## 2.1 文档切分不当

这是最常见的根因。向量检索的基本单位是 chunk，切分策略直接决定了检索效果。

**常见问题：**

| 问题 | 现象 | 影响 |
|------|------|------|
| Chunk 过大 | 检索到的内容噪音多，稀释了关键信息 | 向量表示不精确，相关性下降 |
| Chunk 过小 | 上下文断裂，关键信息被拆分到多个 chunk | 单一 chunk 无法回答问题 |
| 语义断裂 | 在段落中间硬切，概念被割裂 | 语义 embedding 失真 |

**诊断方法：** 随机抽样检索失败的 query，观察被召回的 chunk 是否包含答案。如果不包含，检查答案所在的原始文本是如何被切分的。

## 2.2 向量化质量不足

Embedding 模型是向量检索的"引擎"，模型选择直接影响语义匹配精度。

**常见问题：**

| 问题 | 说明 |
|------|------|
| **领域不匹配** | 通用 embedding 模型在垂直领域（法律、医疗、代码）表现差 |
| **语言不匹配** | 中文文档用了英文主导的 embedding 模型 |
| **维度不足** | 低维度 embedding（如 256-d）表达能力有限 |
| **模型过时** | 早期的 sentence-transformers 模型已被新模型大幅超越 |

**诊断方法：** 构造一组已知相关的 query-document pair，计算余弦相似度分布。如果正样本的相似度与负样本区分度低，说明 embedding 模型不适合当前领域。

## 2.3 查询表达模糊

用户输入的 query 和文档的语言风格、粒度往往不匹配。

**典型场景：**

```
用户 query: "怎么配环境？"
文档标题: "Ubuntu 22.04 下 CUDA 12.0 + PyTorch 2.0 环境搭建指南"

→ 字面重叠几乎为零，向量相似度也可能偏低
```

**诊断方法：** 分析失败 query 的关键词覆盖率和语义相似度。如果 query 过于简短或模糊，问题大概率在查询端而非文档端。

## 2.4 索引结构单一

**纯向量检索有其固有局限：**

- 对精确关键词匹配不敏感（如产品型号、版本号、代码变量名）
- 对数字、日期等结构化信息不敏感
- 无法利用文档的层次结构（标题层级、表格关系）

**诊断方法：** 测试包含精确关键词的 query。如果用户搜索 "v2.3.1" 而文档中确实出现 "v2.3.1"，但向量检索没有召回，说明纯向量检索存在短板。

## 2.5 元数据缺失或利用不足

很多文档携带丰富的元数据信息（作者、时间、来源、文档类型），但检索时没有利用这些信息来缩小搜索空间或提高精度。

**常见缺失：**

| 元数据类型 | 作用 |
|-----------|------|
| 时间戳 | 过滤过期信息，优先检索最新文档 |
| 文档来源 | 区分内部文档 vs 外部引用 |
| 文档类型 | API 文档 vs 教程 vs 问题解答 |
| 标签/分类 | 限定检索范围 |

## 2.6 评估缺失

**没有评估，就没有优化。** 很多 RAG 管线缺乏系统化的评估机制，导致：

- 无法量化当前召回率水平
- 无法判断某个优化方案是否真正有效
- 上线后性能退化无法及时感知

**诊断方法：** 如果无法回答"当前 RAG 管线的 Recall@5 是多少"这个问题，说明评估体系缺失。

---

**下一步：** 诊断完成后，针对具体根因选择对应的优化方案。以下章节逐一展开。
# 3. 方案一：优化文档切分策略

文档切分是 RAG 管线的第一环，也是最容易被忽视的一环。好的切分策略能让同样的 embedding 模型和检索系统发挥更大的威力。

## 3.1 固定长度切分 vs 语义切分 vs 递归切分

### 固定长度切分（Fixed-Size Chunking）

最简单粗暴的方式：按字符数或 token 数切分。

```python
from langchain.text_splitter import CharacterTextSplitter

text_splitter = CharacterTextSplitter(
    separator="\n",
    chunk_size=500,
    chunk_overlap=50,
)
chunks = text_splitter.split_text(long_text)
```

**优点：** 简单、可控、可复现。
**缺点：** 不考虑语义边界，容易在句子中间或段落中间切断。

### 语义切分（Semantic Chunking）

基于文本的语义连贯性来切分：在语义突变处分割，在语义连贯处合并。

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai.embeddings import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()
semantic_splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",  # 百分位阈值
    breakpoint_threshold_amount=95,          # 95分位作为分割点
)
chunks = semantic_splitter.create_documents([long_text])
```

**原理：** 计算相邻句子的 embedding 相似度，相似度骤降的位置就是潜在的分界点。

**优点：** 语义完整性好，chunk 内部主题一致。
**缺点：** 计算成本高（需要对每个句子做 embedding），chunk 大小不可控。

### 递归切分（Recursive Character Text Splitter）

LangChain 默认的切分器，按优先级顺序尝试多种分隔符。

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""],
    length_function=len,
)
chunks = text_splitter.create_documents([long_text])
```

**分隔符优先级：** `\n\n`（段落）→ `\n`（换行）→ `。！？；，`（句子标点）→ ` `（空格）→ `""`（逐字）。

**优点：** 兼顾语义边界和可控性，中文友好（支持中文标点）。
**缺点：** 仍然不是真正的语义理解。

## 3.2 语义感知切分：基于文档结构

对于结构化文档（Markdown、HTML、PDF），**利用文档的层次结构**是最有效的切分策略。

```python
from langchain_text_splitters import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]

markdown_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on,
)
chunks = markdown_splitter.split_text(markdown_text)
```

**效果：** 每个 chunk 自动携带其标题层级信息，既保持了语义完整性，又为后续元数据过滤提供了基础。

## 3.3 重叠策略（Overlap）的选择

overlap 的作用是**在 chunk 边界保留上下文**，避免关键信息被切割。

| Overlap 比例 | 适用场景 | 影响 |
|-------------|---------|------|
| 0%（无重叠） | 独立段落、独立文档 | 节省存储，但容易漏信息 |
| 10-20% | 一般文档 | 平衡成本和效果 |
| 20-30% | 长句子、复杂逻辑 | 更好的上下文保留 |
| 50%+ | 代码、公式、表格 | 结构信息需要更多重叠 |

**经验法则：** `chunk_overlap = chunk_size × 0.15 ~ 0.2` 是一个好的起点，通过评估调优。

## 3.4 层级索引：Parent-Child / Small-to-Big Retrieval

这是目前**最有效的切分+检索策略之一**。

**核心思想：** 索引时使用小 chunk（语义精确），但检索返回时返回其对应的父文档（上下文完整）。

```
文档切分：
├── Parent chunk（大）: 完整段落（2000 tokens）
    ├── Child chunk 1（小）: 第一个概念（300 tokens）
    ├── Child chunk 2（小）: 第二个概念（300 tokens）
    └── Child chunk 3（小）: 第三个概念（300 tokens）

检索流程：
1. 用 Child chunk 的 embedding 做向量检索 → 命中 Child chunk 2
2. 返回对应的 Parent chunk 作为 LLM 上下文
```

```python
from langchain.storage import InMemoryStore
from langchain.retrievers import ParentDocumentRetriever
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings

# 父级切分（大 chunk）
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)
# 子级切分（小 chunk，用于索引）
child_splitter = RecursiveCharacterTextSplitter(chunk_size=300, chunk_overlap=50)

# 向量存储（存子 chunk 的 embedding）
vectorstore = Chroma(embedding_function=OpenAIEmbeddings())
# 存储父文档
store = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 添加文档
retriever.add_documents([document])

# 检索：用子 chunk 匹配，返回父 chunk
results = retriever.invoke("query")
```

**优势：**
- 子 chunk 小 → 向量表示精确，语义匹配更准
- 返回父 chunk → LLM 获得完整上下文，减少幻觉

## 3.5 切分策略效果对比

| 策略 | Recall@5 | 实现复杂度 | 计算成本 | 推荐度 |
|------|----------|-----------|---------|--------|
| 固定长度 | 基线 | ⭐ | ⭐ | ⭐⭐ |
| 递归切分 | +10-15% | ⭐ | ⭐ | ⭐⭐⭐⭐ |
| 语义切分 | +15-25% | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |
| 结构感知 | +20-30% | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| Parent-Child | +25-40% | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ |

> 💡 **实操建议：** 从递归切分开始，用评估指标确认效果，再按需升级到 Parent-Child。不要一开始就上最复杂的方案。

---

**小结：** 切分策略是 RAG 管线的"基础设施"。好的切分让后续所有优化事半功倍。
# 4. 方案二：Embedding 模型选型与优化

Embedding 模型是向量检索的"引擎"。选对模型，召回率可以立竿见影地提升。

## 4.1 通用 vs 领域专用 Embedding 模型

Embedding 模型的表现高度依赖训练数据的领域分布。

| 类型 | 代表模型 | 优势 | 局限 |
|------|---------|------|------|
| **通用模型** | OpenAI text-embedding-3, BGE, nomic-embed | 多语言、多场景泛化好 | 在垂直领域可能不够精准 |
| **代码专用** | CodeBERT, Jina Code Embedding | 理解变量名、API 调用、语法结构 | 自然语言能力弱 |
| **生物医学** | BioBERT, PubMedBERT | 医学术语、基因名、疾病名 | 通用知识弱 |
| **法律专用** | LegalBERT | 法条引用、法律术语 | 非法律领域表现差 |

**选型原则：** 如果领域数据足够，**领域专用模型 > 通用模型**；如果领域广泛或多领域混合，通用模型更稳妥。

## 4.2 主流 Embedding 模型评测

参考 **MTEB（Massive Text Embedding Benchmark）** 榜单，以下是当前主流模型的表现（综合平均分，越高越好）：

| 模型 | 维度 | 综合得分 | 中文能力 | 特点 |
|------|------|---------|---------|------|
| **BGE-M3** | 1024 | ~65 | ⭐⭐⭐⭐⭐ | 多语言、多粒度（稠密+稀疏+多向量） |
| **GTE-Qwen2-7B** | 3584 | ~72 | ⭐⭐⭐⭐⭐ | 开源最强之一，支持长文本 |
| **text-embedding-3-large** | 3072 | ~67 | ⭐⭐⭐⭐ | 商用最强之一，可调维度 |
| **text-embedding-3-small** | 1536 | ~62 | ⭐⭐⭐ | 性价比高 |
| **nomic-embed-text-v1.5** | 768 | ~58 | ⭐⭐ | 开源、轻量、上下文长度 8192 |
| **jina-embeddings-v3** | 1024 | ~62 | ⭐⭐⭐⭐ | 支持 task-specific 微调 |

> 💡 **中文场景推荐：** BGE-M3（开源免费）或 text-embedding-3-large（商用，API 调用）。

## 4.3 Fine-tuning Embedding 模型

当通用模型在特定领域表现不足时，**微调是最终武器**。

### 微调方法：对比学习（Contrastive Learning）

```python
from sentence_transformers import SentenceTransformer, SentencesDataset, losses
from sentence_transformers.readers import InputExample
from torch.utils.data import DataLoader

# 加载预训练模型
model = SentenceTransformer('BAAI/bge-base-zh-v1.5')

# 准备训练数据：(query, positive_doc, negative_doc) 三元组
train_examples = [
    InputExample(texts=["查询1", "相关文档1", "不相关文档1"]),
    InputExample(texts=["查询2", "相关文档2", "不相关文档2"]),
    # ... 更多三元组
]

train_dataset = SentencesDataset(train_examples, model)
train_dataloader = DataLoader(train_dataset, shuffle=True, batch_size=16)

# 使用 MultipleNegativesRankingLoss
train_loss = losses.MultipleNegativesRankingLoss(model)

# 微调
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=3,
    warmup_steps=100,
    output_path='./finetuned-bge-zh'
)
```

### 训练数据从哪里来？

| 数据来源 | 适用场景 | 数据量需求 |
|---------|---------|-----------|
| 历史搜索日志 | 有用户搜索记录的系统 | 10K+ |
| 人工标注 | 小规模精准数据集 | 1K-5K |
| LLM 合成 | 无标注数据时使用 LLM 生成 | 5K-50K |
| 文档-标题对 | 利用文档自身结构 | 自动获取 |

## 4.4 多语言 Embedding 策略

对于中英混合场景，有两种策略：

### 策略 A：使用多语言统一模型

```python
# BGE-M3 天然支持多语言，query 和 document 用同一模型
from FlagEmbedding import BGEM3FlagModel

model = BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)

# 中文 query 和英文文档都可以 embedding
query_emb = model.encode(['如何配置 CUDA 环境？'])['dense_vecs']
doc_emb = model.encode(['CUDA environment setup guide for Ubuntu'])['dense_vecs']
```

### 策略 B：翻译 + 单语模型

```
中文 Query → [翻译] → 英文 Query → [英文 Embedding] → 英文文档
```

适用场景：英文文档库远大于中文文档库，且翻译质量可控。

## 4.5 实现示例：对比 BGE-M3 与 OpenAI text-embedding-3-small

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# === BGE-M3 ===
from FlagEmbedding import BGEM3FlagModel
bge_model = BGEM3FlagModel('BAAI/bge-m3', use_fp16=True)

# === OpenAI ===
from openai import OpenAI
openai_client = OpenAI()

def get_openai_embedding(text):
    resp = openai_client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return np.array(resp.data[0].embedding)

# === 测试 ===
query = "RAG 中如何提升召回率？"
docs = [
    "本文介绍了多种提升 RAG 检索召回率的方法，包括优化切分策略、选择更好的 embedding 模型等。",
    "今天天气不错，适合出去散步。",
    "Retrieval-Augmented Generation combines retrieval with language model generation.",
]

# BGE-M3
q_bge = bge_model.encode([query])['dense_vecs']
d_bge = bge_model.encode(docs)['dense_vecs']
sim_bge = cosine_similarity(q_bge, d_bge)[0]

# OpenAI
q_openai = get_openai_embedding(query)
d_openai = np.array([get_openai_embedding(d) for d in docs])
sim_openai = cosine_similarity([q_openai], d_openai)[0]

print(f"{'文档':<70} | BGE-M3 | OpenAI")
print("-" * 95)
for i, doc in enumerate(docs):
    print(f"{doc:<70} | {sim_bge[i]:.4f} | {sim_openai[i]:.4f}")
```

> 💡 **实操建议：**
> 1. **先评测再选型**：用自己的数据集测试 2-3 个候选模型
> 2. **关注长文本能力**：如果文档较长，注意模型的 max sequence length
> 3. **成本考量**：API 调用的 embedding 模型按 token 计费，量大时考虑本地部署
> 4. **定期关注 MTEB 榜单**：Embedding 模型迭代很快，新模型不断刷新纪录

---

**小结：** Embedding 模型的选择对召回率有 15-30% 的影响。中文场景首选 BGE-M3，追求极致效果可考虑微调。
# 5. 方案三：查询重写与增强（Query Transformation）

用户的原始 query 往往不是检索器的最优输入。**查询重写是成本最低、效果最显著的优化手段之一。**

## 5.1 查询重写（Query Rewriting）

利用 LLM 将用户 query 改写为更适合检索的形式。

### 方法 A：LLM 改写

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4o", temperature=0)

rewrite_prompt = ChatPromptTemplate.from_template("""
你是一位信息检索专家。请将以下用户问题改写为更适合搜索引擎检索的形式。
要求：
1. 提取关键实体和概念
2. 扩展同义词和相关术语
3. 保持语义不变

用户问题：{query}

改写后的查询：
""")

rewrite_chain = rewrite_prompt | llm

def rewrite_query(query: str) -> str:
    response = rewrite_chain.invoke({"query": query})
    return response.content.strip()

# 示例
original = "怎么配环境？"
rewritten = rewrite_query(original)
print(f"原始: {original}")
print(f"改写: {rewritten}")
# 输出示例: "开发环境配置步骤 CUDA PyTorch 安装教程"
```

### 方法 B：同义词扩展

```python
def expand_with_synonyms(query: str) -> list[str]:
    """生成多个查询变体：原始 + 同义词扩展"""
    variants = [query]

    # 示例：简单同义词映射（实际可用 WordNet / HowNet / LLM）
    synonym_map = {
        "配置": ["设置", "安装", "部署", "环境搭建"],
        "提升": ["提高", "优化", "改善", "增强"],
        "召回率": ["recall", "检索准确率", "命中率"],
    }

    for word, synonyms in synonym_map.items():
        if word in query:
            for syn in synonyms[:2]:  # 取前2个同义词
                variants.append(query.replace(word, syn))

    return list(set(variants))

variants = expand_with_synonyms("如何提升 RAG 召回率")
# ['如何提升 RAG 召回率', '如何提高 RAG 召回率', '如何优化 RAG 召回率']
```

## 5.2 多路查询（Multi-Query Retrieval）

生成多个 query 变体，并行检索，去重合并结果。这是**性价比极高的召回率提升方案**。

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import Chroma

# 假设已有 vectorstore
vectorstore = Chroma(...)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(),
    llm=llm,
    parser_key="line",  # 按行解析生成的多个查询
)

# 一次调用，多路检索
docs = retriever.invoke("RAG 召回率怎么提升？")
```

**原理：** LLM 会生成类似如下的多个查询：
1. "如何提高检索增强生成的检索准确率"
2. "RAG retrieval accuracy improvement methods"
3. "向量数据库召回率低怎么办"

每个查询独立检索，结果去重合并 → **显著扩大召回覆盖范围**。

## 5.3 HyDE（Hypothetical Document Embeddings）

HyDE 的核心洞察：**文档的 embedding 和 query 的 embedding 在语义空间中位置不同，但文档和文档的 embedding 更近。**

```
传统方式: Query embedding ←→ Document embedding（可能存在 gap）
HyDE方式: Query → 生成假设文档 → 假设文档 embedding ←→ Document embedding
```

```python
from langchain.prompts import PromptTemplate
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.chains import LLMChain

# 步骤 1：用 LLM 生成假设性文档
hyde_prompt = PromptTemplate(
    input_variables=["query"],
    template="请根据以下问题，生成一个假设性的参考答案（不需要真实，只需要看起来合理）：\n\n问题：{query}\n\n假设性答案："
)

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)
hyde_chain = LLMChain(llm=llm, prompt=hyde_prompt)

# 步骤 2：用假设文档的 embedding 检索真实文档
def hyde_retrieve(query: str, vectorstore, top_k: int = 5):
    # 生成假设文档
    hypothetical_doc = hyde_chain.run(query=query)

    # 用假设文档做 embedding 检索
    docs = vectorstore.similarity_search(hypothetical_doc, k=top_k)

    return docs

# 示例
query = "Transformer 中 Self-Attention 的计算公式是什么？"
docs = hyde_retrieve(query, vectorstore)
```

**为什么有效：** 假设性文档的文本风格和术语使用与真实答案文档更一致，embedding 相似度更高。

## 5.4 子查询分解（Sub-query Decomposition）

对于复杂查询，**分解为多个子查询**分别检索。

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)

decompose_prompt = ChatPromptTemplate.from_template("""
将以下复杂问题分解为多个独立的子问题，每个子问题可以单独检索回答。

问题：{query}

子问题列表（每行一个）：
""")

decompose_chain = decompose_prompt | llm

def decompose_and_retrieve(query: str, vectorstore):
    # 分解子问题
    result = decompose_chain.invoke({"query": query})
    sub_queries = [line.strip() for line in result.content.split("\n") if line.strip()]

    # 每个子问题独立检索
    all_docs = []
    for sq in sub_queries:
        docs = vectorstore.similarity_search(sq, k=3)
        all_docs.extend(docs)

    # 去重
    unique_docs = list({doc.page_content: doc for doc in all_docs}.values())
    return unique_docs

# 示例
complex_query = "对比 BERT 和 GPT 的架构差异，以及它们各自适合什么场景？"
docs = decompose_and_retrieve(complex_query, vectorstore)
# 分解为:
# 1. "BERT 的架构特点是什么？"
# 2. "GPT 的架构特点是什么？"
# 3. "BERT 和 GPT 的架构差异对比"
# 4. "BERT 适合什么应用场景？"
# 5. "GPT 适合什么应用场景？"
```

### Step-back Prompting

一种特殊的分解策略：先问一个更高层级的"后退"问题获取背景知识，再回答原问题。

```
用户问题: "RAG 中用 BM25 和向量混合检索的 RRF 公式是什么？"

Step-back 问题: "什么是混合检索？" → 获取背景知识
然后: "RRF 公式是什么？" → 精确检索
```

## 5.5 实现示例：Multi-Query + HyDE 组合

```python
class EnhancedRetriever:
    """组合 Multi-Query + HyDE 的增强检索器"""

    def __init__(self, vectorstore, llm):
        self.vectorstore = vectorstore
        self.llm = llm

    def retrieve(self, query: str, top_k: int = 10) -> list:
        all_docs = []

        # 策略 1：原始 query 直接检索
        docs_original = self.vectorstore.similarity_search(query, k=top_k)
        all_docs.extend(docs_original)

        # 策略 2：Multi-Query
        multi_query_prompt = f"""
        Generate 3 different versions of the following search query.
        Each version should use different wording but preserve the original intent.

        Original query: {query}

        Query versions (one per line):
        """
        response = self.llm.invoke(multi_query_prompt)
        variants = [line.strip() for line in response.content.split("\n") if line.strip() and line.strip()[0].isdigit()]

        for variant in variants[:3]:
            docs = self.vectorstore.similarity_search(variant, k=5)
            all_docs.extend(docs)

        # 策略 3：HyDE
        hyde_prompt = f"Generate a hypothetical document that would answer this query:\n{query}"
        hyde_response = self.llm.invoke(hyde_prompt)
        docs_hyde = self.vectorstore.similarity_search(hyde_response.content, k=5)
        all_docs.extend(docs_hyde)

        # 去重
        unique_docs = list({doc.page_content: doc for doc in all_docs}.values())
        return unique_docs[:top_k]
```

> 💡 **实操建议：**
> 1. **Multi-Query 是首选**：成本低（小模型即可），效果提升显著（+15-25% Recall@K）
> 2. **HyDE 适合答案风格明确的场景**：如果问题有标准答案格式，HyDE 效果极佳
> 3. **子查询分解适合复杂问题**：多跳推理、对比类问题
> 4. **注意延迟成本**：每增加一个策略，就多一次 LLM 调用和检索，需权衡延迟和效果

---

**小结：** 查询重写是 RAG 优化的"四两拨千斤"之术。用最小的成本（几次 LLM 调用），撬动显著的召回率提升。
# 6. 方案四：混合检索（Hybrid Search）

纯向量检索不是万能的。**混合检索 = 向量检索 + 关键词检索**，两者互补，是提升召回率最稳健的方案之一。

## 6.1 为什么纯向量检索不够

向量检索擅长**语义匹配**，但在以下场景明显不足：

| 场景 | 向量检索 | 关键词检索（BM25） |
|------|---------|------------------|
| 精确匹配（产品型号、版本号） | 差 | 优秀 |
| 专业术语匹配 | 中等 | 优秀 |
| 语义相近但用词不同 | 优秀 | 差 |
| 缩写/别名匹配 | 中等 | 优秀 |
| 长尾查询 | 差 | 中等 |

**典型失败案例：**
```
Query: "Error code 4042 in v3.2.1"
向量检索: 可能匹配到任何关于"error"的文档
BM25: 精确匹配 "4042" 和 "v3.2.1"，直接定位
```

## 6.2 BM25 + 向量的混合策略

### BM25 简介

BM25（Best Matching 25）是信息检索领域的经典算法，核心思想：

```
Score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) / (f(qi, D) + k1 × (1 - b + b × |D|/avgdl))
```

其中：
- `f(qi, D)`：词 qi 在文档 D 中的频率
- `IDF(qi)`：词 qi 的逆文档频率
- `k1, b`：可调参数（通常 k1=1.2~2.0, b=0.75）
- `|D|/avgdl`：文档长度归一化

### BM25 实现

```python
from rank_bm25 import BM25Okapi
import jieba

# 中文需要分词
corpus = [
    "RAG 是一种结合检索和生成的架构",
    "BM25 是经典的关键词匹配算法",
    "向量检索擅长语义匹配，BM25 擅长精确匹配",
]
tokenized_corpus = [list(jieba.cut(doc)) for doc in corpus]

bm25 = BM25Okapi(tokenized_corpus)

query = "向量检索和 BM25 的区别"
tokenized_query = list(jieba.cut(query))

scores = bm25.get_scores(tokenized_query)
for i, score in enumerate(scores):
    print(f"文档{i}: {score:.4f} - {corpus[i]}")
```

## 6.3 稠密 + 稀疏检索的融合架构

```
用户 Query
├──→ [向量检索] → Top-K1 文档（语义相关）
└──→ [BM25检索] → Top-K2 文档（关键词相关）
         ↓
    [融合排序 RRF]
         ↓
    最终 Top-N 文档
```

```python
from rank_bm25 import BM25Okapi
import jieba
import numpy as np

class HybridRetriever:
    """BM25 + 向量混合检索器"""

    def __init__(self, corpus, vectorstore, top_k=10):
        self.corpus = corpus
        self.vectorstore = vectorstore
        self.top_k = top_k

        # 构建 BM25 索引
        tokenized = [list(jieba.cut(doc)) for doc in corpus]
        self.bm25 = BM25Okapi(tokenized)

    def retrieve(self, query: str) -> list:
        # 1. 向量检索
        vector_docs = self.vectorstore.similarity_search_with_score(query, k=self.top_k)

        # 2. BM25 检索
        tokenized_query = list(jieba.cut(query))
        bm25_scores = self.bm25.get_scores(tokenized_query)

        # 3. RRF 融合
        results = self._rrf_fusion(vector_docs, bm25_scores)
        return results

    def _rrf_fusion(self, vector_docs, bm25_scores, k_rrf=60):
        """Reciprocal Rank Fusion"""
        rrf_scores = {}

        # 向量检索结果的 RRF 贡献
        for rank, (doc, vscore) in enumerate(vector_docs):
            content = doc.page_content
            rrf_scores[content] = rrf_scores.get(content, 0) + 1 / (k_rrf + rank + 1)

        # BM25 结果的 RRF 贡献
        bm25_ranked = sorted(enumerate(bm25_scores), key=lambda x: -x[1])
        for rank, (idx, bscore) in enumerate(bm25_ranked[:self.top_k]):
            if bscore > 0:
                content = self.corpus[idx]
                rrf_scores[content] = rrf_scores.get(content, 0) + 1 / (k_rrf + rank + 1)

        # 按 RRF 分数排序
        sorted_results = sorted(rrf_scores.items(), key=lambda x: -x[1])
        return [(content, score) for content, score in sorted_results[:self.top_k]]
```

## 6.4 RRF — Reciprocal Rank Fusion

RRF 是一种简单而强大的结果融合算法，无需调参（只有一个超参数 k）。

```
RRF(d) = Σ 1 / (k + rank_d)
          r
```

其中：
- `d` 是文档
- `r` 是检索器列表（如向量检索 + BM25）
- `rank_d` 是文档 d 在检索器 r 中的排名
- `k` 是常数，通常取 60（经验值，使排名 1 的贡献约为 1/61）

**为什么 RRF 好？**
- 无需分数归一化（向量相似度 0~1，BM25 分数无上限，无法直接比较）
- 对每个检索器的权重自动平衡
- 实现简单，效果好

### RRF vs 加权融合

| 融合方法 | 优点 | 缺点 |
|---------|------|------|
| **RRF** | 无需归一化、免调参 | 不考虑分数绝对值 |
| **加权求和** | 可以调整各检索器权重 | 需要分数归一化、需调权重 |
| **学习排序（Learning to Rank）** | 理论上最优 | 需要标注数据、复杂度高 |

## 6.5 实现示例：Elasticsearch 混合检索

Elasticsearch 原生支持 BM25 + 向量的混合检索，是生产环境的首选方案。

```python
from elasticsearch import Elasticsearch

es = Elasticsearch("http://localhost:9200")

# 创建索引（同时支持 BM25 和 向量）
index_name = "rag-docs"

es.indices.create(index=index_name, body={
    "mappings": {
        "properties": {
            "content": {
                "type": "text",
                "analyzer": "ik_max_word"  # 中文分词
            },
            "embedding": {
                "type": "dense_vector",
                "dims": 1024,
                "index": True,
                "similarity": "cosine"
            }
        }
    }
})

# 索引文档
def index_document(doc_id, content, embedding):
    es.index(index=index_name, id=doc_id, body={
        "content": content,
        "embedding": embedding
    })

# 混合检索
def hybrid_search(query, query_embedding, top_k=10):
    response = es.search(index=index_name, body={
        "retriever": {
            "rrf": {
                "retrievers": [
                    # BM25 检索
                    {
                        "standard": {
                            "query": {
                                "match": {
                                    "content": {
                                        "query": query,
                                        "analyzer": "ik_max_word"
                                    }
                                }
                            }
                        }
                    },
                    # 向量检索（kNN）
                    {
                        "knn": {
                            "field": "embedding",
                            "query_vector": query_embedding,
                            "k": top_k,
                            "num_candidates": top_k * 2
                        }
                    }
                ],
                "rank_constant": 60,  # RRF 的 k 值
                "window_size": top_k
            }
        },
        "size": top_k
    })

    return [hit["_source"]["content"] for hit in response["hits"]["hits"]]
```

> 💡 **实操建议：**
> 1. **混合检索是必选项**：几乎所有 RAG 生产系统都应该用混合检索，而不是纯向量检索
> 2. **BM25 是轻量级 baseline**：`rank_bm25` 库只需几行代码就能跑起来，零成本验证效果
> 3. **生产环境首选 Elasticsearch**：开箱即用的混合检索 + RRF，中文分词用 IK Analyzer
> 4. **RRF 的 k=60 是经验值**：大多数场景下效果最好，不需要调

---

**小结：** 混合检索不是"锦上添花"，而是"基础设施"。向量检索 + BM25 + RRF 是目前性价比最高的召回率提升方案。
# 7. 方案五：元数据过滤与结构化检索

文档不只是文本。**元数据过滤可以在检索前缩小搜索空间，或在检索后精排结果**，是提升召回率和精度的有效手段。

## 7.1 元数据提取策略

### 自动提取的元数据类型

| 元数据类型 | 提取方式 | 用途 |
|-----------|---------|------|
| **时间戳** | 文件名、文档属性、正文日期 | 过滤过期信息、优先最新内容 |
| **来源/作者** | 文档头部、URL、文件路径 | 区分权威来源 |
| **文档类型** | 文件扩展名、路径、内容模式 | API 文档 vs 教程 vs FAQ |
| **语言** | langid / fasttext 检测 | 多语言场景过滤 |
| **关键词/标签** | TF-IDF / TextRank / LLM 提取 | 分类检索 |
| **层级标题** | 文档结构解析 | Parent-Child 索引关联 |

### 元数据提取实现

```python
import os
import re
from datetime import datetime

def extract_metadata(file_path: str, content: str) -> dict:
    """从文件和文档内容中提取元数据"""
    metadata = {}

    # 文件级元数据
    metadata["source"] = file_path
    metadata["filename"] = os.path.basename(file_path)
    metadata["extension"] = os.path.splitext(file_path)[1]
    metadata["modified_time"] = os.path.getmtime(file_path)

    # 从内容中提取时间（如 "2024-03-15", "March 2024"）
    date_patterns = [
        r'(\d{4}-\d{2}-\d{2})',          # 2024-03-15
        r'(\d{4}/\d{2}/\d{2})',          # 2024/03/15
        r'((?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\w* \d{4})',
    ]
    for pattern in date_patterns:
        match = re.search(pattern, content)
        if match:
            metadata["doc_date"] = match.group(1)
            break

    # 从 Markdown 标题提取主题
    headings = re.findall(r'^#{1,3}\s+(.+)$', content, re.MULTILINE)
    metadata["headings"] = headings

    # 用 LLM 提取标签
    # （此处省略，见 7.1 节进阶）

    return metadata
```

### 用 LLM 自动打标

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import JsonOutputParser

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

tagging_prompt = """
你是一个文档分类专家。请分析以下文档内容，提取以下信息：

1. 主题类别（如：教程、API文档、FAQ、最佳实践等）
2. 技术领域（如：机器学习、Web开发、数据库等）
3. 关键词（3-5个）
4. 难度等级（入门/中级/高级）

文档内容（前1000字）：
{text}

以 JSON 格式输出。
"""

tagging_chain = tagging_prompt | llm | JsonOutputParser()

metadata = tagging_chain.invoke({"text": content[:1000]})
# 输出: {"category": "教程", "domain": "机器学习", "keywords": ["RAG", "检索", "召回率"], "difficulty": "中级"}
```

## 7.2 预过滤 vs 后过滤 vs 联合检索

### 预过滤（Pre-filtering）

在向量检索**之前**应用元数据条件缩小范围。

```python
from langchain_chroma import Chroma

# 只检索特定类型和时间范围的文档
docs = vectorstore.similarity_search(
    query="RAG 优化方法",
    k=10,
    filter={
        "doc_type": "tutorial",
        "doc_date": {"$gte": "2024-01-01"}
    }
)
```

**优点：** 检索范围缩小 → 更精准。
**缺点：** 如果过滤条件太严格，可能过滤掉相关文档。

### 后过滤（Post-filtering）

在向量检索**之后**过滤结果。

```python
# 先检索更多结果
docs = vectorstore.similarity_search(query="RAG 优化方法", k=50)

# 再按元数据过滤
filtered_docs = [
    doc for doc in docs
    if doc.metadata.get("doc_type") in ["tutorial", "guide"]
]

# 取 Top-K
final_docs = filtered_docs[:10]
```

**优点：** 不会错过相关文档。
**缺点：** 可能过滤后结果不足。

### 联合检索（Joint Filtering）

将元数据作为检索的联合条件，同时考虑向量相似度和元数据匹配度。

```python
from langchain.retrievers import SelfQueryRetriever
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings, ChatOpenAI

# 定义元数据字段
metadata_field_info = [
    {"name": "doc_type", "type": "string", "description": "文档类型"},
    {"name": "domain", "type": "string", "description": "技术领域"},
    {"name": "difficulty", "type": "string", "description": "难度等级"},
    {"name": "doc_date", "type": "string", "description": "文档日期"},
]

# SelfQueryRetriever：LLM 自动从 query 中提取过滤条件
retriever = SelfQueryRetriever.from_llm(
    llm=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    vectorstore=vectorstore,
    document_contents="技术文档集合，涵盖 AI、编程、数据库等领域",
    metadata_field_info=metadata_field_info,
)

# LLM 自动解析：
# "找一下最近半年关于 RAG 优化的中级教程"
# → 自动提取 filter: {"difficulty": "中级", "doc_date": ">= 2024-01-01"}
docs = retriever.invoke("找一下最近半年关于 RAG 优化的中级教程")
```

## 7.3 知识图谱增强的结构化检索

对于复杂关联关系的知识，**知识图谱（Knowledge Graph）是元数据过滤的进阶形态**。

```
                    文档A
                   /    \
          属于:AI框架    作者:张三
               |
         包含组件:PyTorch
               |
        相关文档:文档B, 文档C
```

```python
from langchain_community.graphs import NetworkxEntityGraph
import networkx as nx

# 构建简单的知识图谱
graph = NetworkxEntityGraph()
graph.add_node("PyTorch", {"type": "framework"})
graph.add_node("RAG", {"type": "architecture"})
graph.add_node("向量数据库", {"type": "component"})

# 添加关系
graph.add_edge("PyTorch", "RAG", relation="used_in")
graph.add_edge("向量数据库", "RAG", relation="component_of")

# 查询：找出与 RAG 直接相关的所有组件
related = list(graph._graph.neighbors("RAG"))
# ['PyTorch', '向量数据库']

# 用这些相关实体扩展检索查询
expanded_query = "RAG " + " ".join(related)
```

## 7.4 实现示例：ChromaDB 元数据过滤 + 向量检索

```python
import chromadb
from chromadb.utils import embedding_functions

# 初始化
client = chromadb.PersistentClient(path="./chroma_db")
embed_fn = embedding_functions.SentenceTransformerEmbeddingFunction(
    model_name="BAAI/bge-m3"
)
collection = client.get_or_create_collection(
    name="tech-docs",
    embedding_function=embed_fn,
)

# 索引带元数据的文档
collection.add(
    documents=[
        "RAG 是一种结合检索和生成的架构...",
        "PyTorch 2.0 性能优化指南...",
        "如何部署 LLM 到生产环境...",
    ],
    metadatas=[
        {"doc_type": "guide", "domain": "LLM", "year": 2024},
        {"doc_type": "tutorial", "domain": "PyTorch", "year": 2024},
        {"doc_type": "ops", "domain": "LLM", "year": 2023},
    ],
    ids=["doc1", "doc2", "doc3"],
)

# 带元数据过滤的检索
results = collection.query(
    query_texts=["LLM 部署最佳实践"],
    n_results=5,
    where={"year": {"$gte": 2024}},        # 过滤条件
    where_document={"$contains": "部署"},  # 内容关键词过滤
)

for i, (doc, metadata, distance) in enumerate(zip(
    results["documents"][0], results["metadatas"][0], results["distances"][0]
)):
    print(f"[{i}] dist={distance:.4f} type={metadata['doc_type']} - {doc[:80]}...")
```

> 💡 **实操建议：**
> 1. **元数据提取是"免费"的优化**：文档入库时提取一次，后续所有检索都能受益
> 2. **SelfQueryRetriever 是利器**：用户自然语言 query → 自动提取过滤条件，体验极佳
> 3. **时间过滤最常见也最有效**：技术文档的时效性极强，按时间过滤能大幅提升准确率
> 4. **LLM 打标成本可控**：每篇文档只需一次 LLM 调用，且可以异步批量处理

---

**小结：** 元数据不是配角，而是检索管线中的关键参与者。好的元数据策略能让检索从"大海捞针"变成"按图索骥"。
# 8. 方案六：重排序（Re-Ranking）

重排序是召回率和精度之间的桥梁。它的核心思路是：**先用低成本方式召回大量候选（Recall），再用高精度模型精排（Precision）。**

## 8.1 为什么检索后需要重排

向量检索（Bi-Encoder）的优势是速度快，但代价是精度有限。因为它在索引时将文档独立编码为向量，查询时只做向量相似度计算，**无法做 query 和 document 之间的细粒度交互**。

```
两阶段检索流水线：

阶段1（召回）: Query → 向量检索 → Top-50 候选（快速、低精度）
阶段2（精排）: Query + Top-50 → Cross-Encoder → Top-5 最终结果（慢、高精度）
```

**效果对比：**

| 方案 | Recall@10 | Precision@5 | 延迟 |
|------|-----------|-------------|------|
| 纯向量检索 | 基线 | 基线 | <50ms |
| 向量 + Reranker | +5-15% | +20-40% | 200-500ms |

## 8.2 Cross-Encoder vs Bi-Encoder 架构对比

### Bi-Encoder（向量检索用）

```
Query  → Encoder → [1024-d 向量]
Doc    → Encoder → [1024-d 向量]
                    ↓
              余弦相似度计算
```

**特点：** Query 和 Document 独立编码，一次编码永久使用。适合大规模索引和检索。
**局限：** 无法感知 Query-Document 之间的细粒度交互。

### Cross-Encoder（重排序用）

```
[Query] + [Separator] + [Document] → Encoder → 相关性分数
```

**特点：** Query 和 Document 一起编码，Attention 机制可以捕捉细粒度交互。
**局限：** 每对 query-document 都需要独立编码，不适合大规模检索。

**类比理解：**
- Bi-Encoder 像是相亲时先看照片（快速筛选）
- Cross-Encoder 像是见面深聊（精确判断，但耗时）

## 8.3 主流重排序模型

| 模型 | 参数量 | 中文能力 | 特点 |
|------|--------|---------|------|
| **BGE-Reranker-v2-m3** | ~0.5B | ⭐⭐⭐⭐⭐ | 多语言，开源最强之一 |
| **BGE-Reranker-Large** | ~0.5B | ⭐⭐⭐⭐ | 中英双语 |
| **Cohere Rerank** | 商用 API | ⭐⭐⭐ | 免部署，按调用计费 |
| **Jina Reranker** | ~0.3B | ⭐⭐⭐⭐ | 支持多语言 |
| **ColBERT** | ~0.1B | ⭐⭐ | 延迟交互，速度较快 |

## 8.4 两阶段检索流水线设计

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

class CrossEncoderReranker:
    """Cross-Encoder 重排序器"""

    def __init__(self, model_name="BAAI/bge-reranker-v2-m3", device="cuda"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(model_name)
        self.model.to(device)
        self.model.eval()

    def rerank(self, query: str, documents: list[str], top_k: int = 5) -> list:
        """
        对候选文档进行重排序
        :param query: 用户查询
        :param documents: 候选文档列表（通常来自向量检索的 Top-30~Top-50）
        :param top_k: 返回的 top 结果数
        :return: 排序后的 (文档, 分数) 列表
        """
        # 构造 (query, document) 对
        pairs = [[query, doc] for doc in documents]

        # 批量推理
        with torch.no_grad():
            inputs = self.tokenizer(
                pairs,
                padding=True,
                truncation=True,
                max_length=512,
                return_tensors="pt"
            ).to(self.model.device)

            scores = self.model(**inputs, return_dict=True).logits.view(-1).float()

        # 排序
        scored_docs = sorted(
            zip(documents, scores.cpu().tolist()),
            key=lambda x: x[1],
            reverse=True
        )

        return scored_docs[:top_k]
```

## 8.5 完整两阶段流水线

```python
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain.vectorstores import Chroma
from langchain_core.documents import Document

class TwoStageRetriever:
    """两阶段检索：向量召回 + Cross-Encoder 精排"""

    def __init__(self, vectorstore, reranker, recall_k=50, final_k=5):
        self.vectorstore = vectorstore
        self.reranker = reranker
        self.recall_k = recall_k    # 阶段1召回数量
        self.final_k = final_k      # 阶段2精排后返回数量

    def retrieve(self, query: str) -> list:
        # === 阶段1：向量召回（快速、大规模）===
        docs = self.vectorstore.similarity_search(query, k=self.recall_k)
        candidates = [doc.page_content for doc in docs]

        if len(candidates) == 0:
            return []

        # === 阶段2：Cross-Encoder 重排（精确、小规模）===
        ranked = self.reranker.rerank(query, candidates, top_k=self.final_k)

        # 返回结果
        return [Document(page_content=doc, metadata={"rerank_score": score})
                for doc, score in ranked]

# === 使用示例 ===
# 1. 初始化组件
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma(
    collection_name="tech-docs",
    embedding_function=embeddings,
    persist_directory="./chroma_db"
)

reranker = CrossEncoderReranker(model_name="BAAI/bge-reranker-v2-m3")

retriever = TwoStageRetriever(
    vectorstore=vectorstore,
    reranker=reranker,
    recall_k=50,
    final_k=5
)

# 2. 检索
query = "如何提升 RAG 的召回率？"
results = retriever.retrieve(query)

for i, doc in enumerate(results):
    print(f"[{i}] score={doc.metadata['rerank_score']:.4f}")
    print(f"    {doc.page_content[:100]}...\n")
```

## 8.6 延迟与成本分析

重排序的成本主要来自 Cross-Encoder 推理：

| 候选数量 | BGE-Reranker-Large (GPU) | BGE-Reranker-Large (CPU) |
|---------|-------------------------|-------------------------|
| Top-10 → Top-5 | ~20ms | ~200ms |
| Top-30 → Top-5 | ~60ms | ~600ms |
| Top-50 → Top-5 | ~100ms | ~1000ms |

**优化策略：**
1. **batch 推理**：一次性对所有候选文档推理，而非逐条
2. **截断长度**：文档截断到 256-512 tokens，减少计算量
3. **召回数量调优**：recall_k 不是越大越好，通常 20-50 是性价比最高的区间
4. **模型蒸馏**：用大模型蒸馏小模型，保持 90%+ 精度，延迟降低 50%+

```python
# Batch 推理优化
def rerank_batch(self, query: str, documents: list[str], batch_size: int = 32):
    """分批量推理，避免 OOM"""
    all_scores = []
    for i in range(0, len(documents), batch_size):
        batch_docs = documents[i:i + batch_size]
        batch_pairs = [[query, doc] for doc in batch_docs]

        with torch.no_grad():
            inputs = self.tokenizer(batch_pairs, padding=True, truncation=True,
                                    max_length=512, return_tensors="pt").to(self.model.device)
            batch_scores = self.model(**inputs, return_dict=True).logits.view(-1).float()
            all_scores.extend(batch_scores.cpu().tolist())

    return all_scores
```

> 💡 **实操建议：**
> 1. **重排序是召回率提升的"最后一道保险"**：前面的方案确保相关文档被召回，重排确保它们排在前面
> 2. **BGE-Reranker-v2-m3 是中文场景首选**：开源免费，效果对标商用方案
> 3. **recall_k=30~50 是甜点区**：太小可能漏掉，太大延迟增加但收益递减
> 4. **生产环境必须上 GPU**：CPU 推理延迟太高，不适合在线服务

---

**小结：** 重排序是"好钢用在刀刃上"——用少量额外计算，换来精度的大幅提升。两阶段检索（召回 + 精排）是现代 RAG 管线的标配。
# 9. 方案七：构建评估体系

没有评估，就没有优化方向。**评估体系告诉你"现在多差"和"改了多好"。**

## 9.1 评估指标

| 指标 | 公式 | 含义 | 适用场景 |
|------|------|------|---------|
| **Recall@K** | 命中相关文档数 / 总相关文档数 | 检索器覆盖面 | 核心指标 |
| **Precision@K** | 命中相关文档数 / K | 结果纯净度 | 关注噪音时 |
| **NDCG@K** | 归一化折损累计增益 | 考虑相关性等级和排名 | 多级相关性 |
| **MRR** | 1 / 第一个相关文档的排名 | 首个命中排名 | QA 场景 |

```python
import numpy as np

def recall_at_k(retrieved_docs, relevant_docs, k):
    """Recall@K"""
    retrieved_top_k = retrieved_docs[:k]
    hits = len(set(retrieved_top_k) & set(relevant_docs))
    return hits / len(relevant_docs) if relevant_docs else 0

def precision_at_k(retrieved_docs, relevant_docs, k):
    """Precision@K"""
    retrieved_top_k = retrieved_docs[:k]
    hits = len(set(retrieved_top_k) & set(relevant_docs))
    return hits / k

def mrr(retrieved_docs, relevant_docs):
    """Mean Reciprocal Rank（单条 query）"""
    for i, doc in enumerate(retrieved_docs):
        if doc in relevant_docs:
            return 1.0 / (i + 1)
    return 0.0

def ndcg_at_k(retrieved_docs, relevant_docs, k, rel_map):
    """NDCG@K，rel_map 是 {doc_id: relevance_score} 的映射"""
    dcg = 0.0
    for i, doc in enumerate(retrieved_docs[:k]):
        rel = rel_map.get(doc, 0)
        dcg += (2**rel - 1) / np.log2(i + 2)

    # Ideal DCG
    ideal_rels = sorted([rel_map.get(d, 0) for d in retrieved_docs], reverse=True)[:k]
    idcg = sum((2**r - 1) / np.log2(i + 2) for i, r in enumerate(ideal_rels))

    return dcg / idcg if idcg > 0 else 0.0
```

## 9.2 构建评估数据集

评估数据集的质量决定了评估结果的可信度。

### 数据集构建方法

| 方法 | 成本 | 质量 | 适用场景 |
|------|------|------|---------|
| **人工标注** | 高 | 最高 | 小规模精准评估（50-200 条） |
| **LLM 合成** | 中 | 较高 | 中等规模评估（500-2000 条） |
| **历史搜索日志** | 低 | 中 | 已有用户行为数据的系统 |
| **文档-标题对** | 最低 | 基线 | 快速验证，自动获取 |

### 用 LLM 合成评估数据集

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import JsonOutputParser

llm = ChatOpenAI(model="gpt-4o", temperature=0.3)

qa_gen_prompt = """
你是评估数据集构建专家。请根据以下文档内容，生成 3-5 个问答对。

要求：
1. 问题应该是用户真实会问的形式
2. 标注出回答该问题需要的文档编号
3. 问题应覆盖不同难度

文档：
{document}

以 JSON 格式输出：
[
  {{"question": "...", "relevant_doc_ids": [0, 2], "difficulty": "easy"}}
]
"""

qa_gen_chain = qa_gen_prompt | llm | JsonOutputParser()

# 为每个文档生成 QA 对
def build_eval_dataset(documents):
    eval_data = []
    for i, doc in enumerate(documents):
        qa_pairs = qa_gen_chain.invoke({"document": doc})
        for qa in qa_pairs:
            eval_data.append({
                "question": qa["question"],
                "relevant_doc_ids": [i],
                "difficulty": qa.get("difficulty", "medium"),
            })
    return eval_data
```

### 评估数据集格式

```json
[
  {
    "question": "如何提升 RAG 的召回率？",
    "relevant_doc_ids": ["doc_chunk_3", "doc_chunk_7", "doc_chunk_12"],
    "difficulty": "hard"
  },
  {
    "question": "BM25 算法是什么？",
    "relevant_doc_ids": ["doc_chunk_15"],
    "difficulty": "easy"
  }
]
```

## 9.3 自动化评估框架：Ragas

[Ragas](https://github.com/explodinggradients/ragas) 是目前最流行的 RAG 评估框架。

```python
from datasets import Dataset
from ragas import evaluate
from ragas.metrics import (
    context_precision,
    context_recall,
    faithfulness,
    answer_relevancy,
)
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

# 准备评估数据
data_samples = {
    "question": [
        "如何提升 RAG 召回率？",
        "BM25 和向量检索的区别是什么？",
        "什么是 RRF 算法？",
    ],
    "contexts": [
        ["本文介绍了7种提升RAG召回率的方案，包括优化切分、混合检索、重排序等。"],
        ["BM25是关键词匹配算法，向量检索是语义匹配算法。两者互补。"],
        ["RRF（Reciprocal Rank Fusion）是一种结果融合算法，公式为 Σ 1/(k+rank)。"],
    ],
    "answer": [
        "可以通过优化文档切分、选择更好的 embedding 模型、查询重写、混合检索、元数据过滤、重排序等方案提升召回率。",
        "BM25 基于关键词频率和逆文档频率做匹配；向量检索将文本映射为向量后用余弦相似度计算。前者精确匹配好，后者语义理解强。",
        "RRF 是 Reciprocal Rank Fusion 的缩写，用于融合多个检索结果。核心公式是每个检索器中 1/(k+排名) 的累加和。",
    ],
    "ground_truth": [
        "主要方案有：优化文档切分策略、embedding 模型选型、查询重写与增强、混合检索（BM25+向量）、元数据过滤、重排序。",
        "BM25 是传统关键词匹配算法，擅长精确匹配；向量检索将文本转为向量后计算语义相似度，擅长语义理解。两者结合效果最佳。",
        "RRF 即 Reciprocal Rank Fusion，通过累加各检索器中 1/(60+排名) 来融合结果，无需分数归一化。",
    ],
}

dataset = Dataset.from_dict(data_samples)

# 运行评估
llm = ChatOpenAI(model="gpt-4o")
embeddings = OpenAIEmbeddings()

result = evaluate(
    dataset,
    metrics=[context_recall, context_precision, faithfulness, answer_relevancy],
    llm=llm,
    embeddings=embeddings,
)

print(result)
# 输出:
# context_recall:      0.85   ← 上下文是否包含了正确答案
# context_precision:   0.80   ← 上下文中有多少是相关的
# faithfulness:        0.90   ← 答案是否忠实于上下文
# answer_relevancy:    0.88   ← 答案是否切题
```

### Ragas 指标解读

| 指标 | 关注点 | 低分说明 |
|------|--------|---------|
| **context_recall** | 检索到的上下文是否包含答案 | 召回率低，需要优化检索 |
| **context_precision** | 检索到的上下文有多少是相关的 | 噪音多，需要重排序/元数据过滤 |
| **faithfulness** | 答案是否基于上下文 | LLM 幻觉问题 |
| **answer_relevancy** | 答案是否切题 | Prompt 或 LLM 问题 |

## 9.4 在线 A/B 测试与监控

离线评估通过后，需要在线验证效果。

```python
class RagMetricsTracker:
    """RAG 管线指标追踪"""

    def __init__(self):
        self.metrics = {
            "total_queries": 0,
            "latency_p50": [],
            "latency_p99": [],
            "user_feedback_thumbs_up": 0,
            "user_feedback_thumbs_down": 0,
            "retrieval_recall_at_5": [],
        }

    def log_query(self, query, response, latency_ms, user_feedback=None):
        self.metrics["total_queries"] += 1
        self.metrics["latency_p50"].append(latency_ms)

        if user_feedback == "up":
            self.metrics["user_feedback_thumbs_up"] += 1
        elif user_feedback == "down":
            self.metrics["user_feedback_thumbs_down"] += 1

    def get_satisfaction_rate(self):
        total = self.metrics["user_feedback_thumbs_up"] + self.metrics["user_feedback_thumbs_down"]
        if total == 0:
            return None
        return self.metrics["user_feedback_thumbs_up"] / total

    def get_p99_latency(self):
        if not self.metrics["latency_p50"]:
            return None
        sorted_latencies = sorted(self.metrics["latency_p50"])
        idx = int(len(sorted_latencies) * 0.99)
        return sorted_latencies[min(idx, len(sorted_latencies) - 1)]
```

## 9.5 实现示例：完整的评估管线

```python
import json
from typing import List, Dict

class RAGEvaluator:
    """完整的 RAG 评估管线"""

    def __init__(self, retriever, llm_evaluator=None):
        self.retriever = retriever
        self.llm_evaluator = llm_evaluator or ChatOpenAI(model="gpt-4o-mini")

    def evaluate(self, eval_dataset: List[Dict], k: int = 5) -> Dict:
        """
        评估 RAG 管线
        :param eval_dataset: [{"question": "...", "relevant_doc_ids": [...]}]
        :param k: Recall/Precision@K 的 K 值
        """
        results = {
            "recall_at_k": [],
            "precision_at_k": [],
            "mrr": [],
            "latency_ms": [],
        }

        for sample in eval_dataset:
            query = sample["question"]
            relevant_ids = set(sample["relevant_doc_ids"])

            # 检索
            import time
            start = time.time()
            retrieved = self.retriever.retrieve(query, top_k=k)
            latency = (time.time() - start) * 1000

            # 提取检索到的 doc_ids
            retrieved_ids = set(doc.metadata.get("doc_id", "") for doc in retrieved)

            # 计算指标
            hits = len(retrieved_ids & relevant_ids)
            recall = hits / len(relevant_ids) if relevant_ids else 0
            precision = hits / k if k > 0 else 0

            # MRR
            mrr_score = 0
            for i, doc in enumerate(retrieved):
                if doc.metadata.get("doc_id") in relevant_ids:
                    mrr_score = 1.0 / (i + 1)
                    break

            results["recall_at_k"].append(recall)
            results["precision_at_k"].append(precision)
            results["mrr"].append(mrr_score)
            results["latency_ms"].append(latency)

        # 汇总
        summary = {
            "num_queries": len(eval_dataset),
            "mean_recall_at_k": np.mean(results["recall_at_k"]),
            "mean_precision_at_k": np.mean(results["precision_at_k"]),
            "mean_mrr": np.mean(results["mrr"]),
            "p50_latency_ms": np.percentile(results["latency_ms"], 50),
            "p99_latency_ms": np.percentile(results["latency_ms"], 99),
        }

        return summary

# === 使用 ===
evaluator = RAGEvaluator(retriever=TwoStageRetriever(...))
summary = evaluator.evaluate(eval_dataset, k=5)

print(json.dumps(summary, indent=2))
# {
#   "num_queries": 100,
#   "mean_recall_at_5": 0.72,
#   "mean_precision_at_5": 0.64,
#   "mean_mrr": 0.58,
#   "p50_latency_ms": 180,
#   "p99_latency_ms": 450
# }
```

> 💡 **实操建议：**
> 1. **先建数据集，再做优化**：没有评估数据集就开始调参是盲人摸象
> 2. **Ragas 是首选工具**：开箱即用，指标全面，社区活跃
> 3. **关注 context_recall 而非 answer_relevancy**：本文聚焦召回率，context_recall 是直接指标
> 4. **定期回归测试**：每次改动都跑一遍评估，防止性能退化
> 5. **用户反馈是金**：点踩/点赞数据是最真实的评估信号

---

**小结：** 评估体系不是"锦上添花"，而是"导航仪"。它让你知道方向对不对、改进有没有效、性能有没有退化。
# 10. 综合实战：一个端到端的高召回率 RAG 管线

前面介绍了 7 个独立方案，本节将它们串联成一个完整的生产级 RAG 管线。

## 10.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        文档入库管线                              │
│                                                                 │
│  原始文档 → 结构解析 → 智能切分 → 元数据提取 → Embedding → 索引  │
│                (Ch3)        (Ch7)      (Ch4)     (Ch4/6/7)       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        检索管线                                  │
│                                                                 │
│  用户Query → 查询重写 → 混合检索 → 元数据过滤 → 重排序 → Top-K    │
│               (Ch5)      (Ch6)        (Ch7)      (Ch8)          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        评估管线                                  │
│                                                                 │
│  评估数据集 → 离线评估(Ragas) → 在线A/B测试 → 指标监控           │
│               (Ch9)              (Ch9)        (Ch9)             │
└─────────────────────────────────────────────────────────────────┘
```

## 10.2 技术栈选型建议

| 组件 | 推荐方案 | 备选方案 | 说明 |
|------|---------|---------|------|
| **切分** | RecursiveCharacterTextSplitter + Parent-Child | 语义切分 | 生产推荐递归+层级 |
| **Embedding** | BGE-M3 (本地) / text-embedding-3-large (API) | GTE-Qwen2-7B | 中文首选 BGE-M3 |
| **向量存储** | Elasticsearch | Qdrant / Milvus / Chroma | ES 原生支持混合检索 |
| **关键词检索** | BM25 (ES 内置) | rank_bm25 (轻量) | 生产用 ES |
| **重排序** | BGE-Reranker-v2-m3 | Cohere Rerank API | 开源首选 |
| **查询重写** | GPT-4o-mini / Qwen-Plus | Claude Haiku | 小模型够用 |
| **评估** | Ragas | DeepEval / Phoenix | Ragas 最成熟 |
| **框架** | LangChain / LlamaIndex | Haystack | 任选其一 |

## 10.3 完整代码示例

以下是一个整合了所有优化方案的端到端 RAG 管线：

```python
"""
高召回率 RAG 管线 - 完整实现
整合方案：智能切分 + Embedding 优化 + 查询重写 + 混合检索 + 元数据过滤 + 重排序
"""

import os
import json
import re
import time
import jieba
from typing import List, Dict, Optional
from dataclasses import dataclass, field

# === LangChain 组件 ===
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.documents import Document
from langchain.retrievers import ParentDocumentRetriever, SelfQueryRetriever

# === 向量存储 ===
from langchain_chroma import Chroma

# === BM25 ===
from rank_bm25 import BM25Okapi

# === Reranker ===
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

# === 评估 ===
from ragas.metrics import context_recall, context_precision
from ragas import evaluate


# ============================================
# 组件1：文档处理
# ============================================

@dataclass
class DocumentProcessor:
    """文档处理管线：解析 → 切分 → 元数据提取"""

    chunk_size: int = 1000
    chunk_overlap: int = 200
    parent_chunk_size: int = 2000
    enable_parent_child: bool = True

    def process(self, raw_text: str, source: str = "") -> List[Document]:
        """处理原始文本为带元数据的 Document 列表"""

        # 1. 提取元数据
        metadata = self._extract_metadata(raw_text, source)

        # 2. 切分
        if self.enable_parent_child:
            docs = self._parent_child_split(raw_text, metadata)
        else:
            docs = self._recursive_split(raw_text, metadata)

        return docs

    def _extract_metadata(self, text: str, source: str) -> Dict:
        """从文档提取元数据"""
        metadata = {"source": source}

        # 提取标题
        title_match = re.search(r'^#\s+(.+)$', text, re.MULTILINE)
        if title_match:
            metadata["title"] = title_match.group(1).strip()

        # 提取日期
        date_match = re.search(r'(\d{4}-\d{2}-\d{2})', text)
        if date_match:
            metadata["date"] = date_match.group(1)

        # 提取标签
        tags_match = re.findall(r'tags?:\s*\[?(.+?)\]?', text)
        if tags_match:
            metadata["tags"] = [t.strip() for t in tags_match[0].split(",")]

        # 提取层级标题
        metadata["headings"] = re.findall(r'^#{1,3}\s+(.+)$', text, re.MULTILINE)

        return metadata

    def _parent_child_split(self, text: str, metadata: Dict) -> List[Document]:
        """Parent-Child 切分"""
        parent_splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.parent_chunk_size,
            chunk_overlap=self.chunk_overlap,
            separators=["\n\n", "\n", "。", "！", "？", "；", "，", " "],
        )
        child_splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.chunk_size,
            chunk_overlap=self.chunk_overlap // 4,
            separators=["\n\n", "\n", "。", "！", "？", "；", "，", " "],
        )

        parent_docs = parent_splitter.create_documents([text], metadatas=[metadata])
        child_docs = child_splitter.create_documents([text], metadatas=[metadata])

        # 关联父子文档（通过 source + 位置信息）
        for i, child in enumerate(child_docs):
            child.metadata["parent_idx"] = i // 3  # 简化关联

        # 实际检索时返回 parent，索引时用 child
        return child_docs  # 索引用 child

    def _recursive_split(self, text: str, metadata: Dict) -> List[Document]:
        """递归切分"""
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=self.chunk_size,
            chunk_overlap=self.chunk_overlap,
            separators=["\n\n", "\n", "。", "！", "？", "；", "，", " "],
        )
        return splitter.create_documents([text], metadatas=[metadata])


# ============================================
# 组件2：混合检索器
# ============================================

class HybridRetriever:
    """BM25 + 向量混合检索 + RRF 融合"""

    def __init__(self, vectorstore, corpus: List[str], top_k: int = 30):
        self.vectorstore = vectorstore
        self.top_k = top_k
        self.corpus = corpus

        # 构建 BM25 索引
        tokenized_corpus = [list(jieba.cut(doc)) for doc in corpus]
        self.bm25 = BM25Okapi(tokenized_corpus)

    def retrieve(self, query: str) -> List[Document]:
        # 向量检索
        vector_docs = self.vectorstore.similarity_search_with_score(query, k=self.top_k)

        # BM25 检索
        tokenized_query = list(jieba.cut(query))
        bm25_scores = self.bm25.get_scores(tokenized_query)

        # RRF 融合
        return self._rrf_fusion(vector_docs, bm25_scores)

    def _rrf_fusion(self, vector_docs, bm25_scores, k_rrf: int = 60) -> List[Document]:
        rrf_scores = {}

        # 向量结果
        for rank, (doc, vscore) in enumerate(vector_docs):
            content = doc.page_content
            rrf_scores[content] = rrf_scores.get(content, 0) + 1 / (k_rrf + rank + 1)

        # BM25 结果
        bm25_ranked = sorted(enumerate(bm25_scores), key=lambda x: -x[1])
        for rank, (idx, bscore) in enumerate(bm25_ranked[:self.top_k]):
            if bscore > 0:
                content = self.corpus[idx]
                rrf_scores[content] = rrf_scores.get(content, 0) + 1 / (k_rrf + rank + 1)

        sorted_results = sorted(rrf_scores.items(), key=lambda x: -x[1])

        return [Document(page_content=content, metadata={"rrf_score": score})
                for content, score in sorted_results[:self.top_k]]


# ============================================
# 组件3：重排序器
# ============================================

class CrossEncoderReranker:
    """Cross-Encoder 重排序器"""

    def __init__(self, model_name: str = "BAAI/bge-reranker-v2-m3",
                 device: str = None):
        self.device = device or ("cuda" if torch.cuda.is_available() else "cpu")
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(model_name)
        self.model.to(self.device)
        self.model.eval()

    def rerank(self, query: str, documents: List[Document],
               top_k: int = 5) -> List[Document]:
        if not documents:
            return []

        pairs = [[query, doc.page_content] for doc in documents]

        with torch.no_grad():
            inputs = self.tokenizer(
                pairs, padding=True, truncation=True,
                max_length=512, return_tensors="pt"
            ).to(self.device)

            scores = self.model(**inputs, return_dict=True).logits.view(-1).float()

        scored_docs = sorted(
            zip(documents, scores.cpu().tolist()),
            key=lambda x: x[1],
            reverse=True
        )

        return [Document(page_content=doc.page_content,
                         metadata={**doc.metadata, "rerank_score": float(score)})
                for doc, score in scored_docs[:top_k]]


# ============================================
# 组件4：查询重写器
# ============================================

class QueryRewriter:
    """LLM 驱动的查询重写"""

    def __init__(self, llm: ChatOpenAI):
        self.llm = llm

    def multi_query(self, query: str, num_variants: int = 3) -> List[str]:
        """生成多个查询变体"""
        prompt = f"""Generate {num_variants} different versions of this search query.
Each version should use different wording but preserve the original intent.

Original query: {query}

Query versions (one per line, no numbering):"""

        response = self.llm.invoke(prompt)
        variants = [line.strip() for line in response.content.split("\n")
                    if line.strip()]

        return [query] + variants[:num_variants]

    def hyde(self, query: str) -> str:
        """生成假设性文档"""
        prompt = f"Generate a hypothetical document that would answer this query:\n{query}"
        response = self.llm.invoke(prompt)
        return response.content.strip()


# ============================================
# 组件5：端到端 RAG 管线
# ============================================

@dataclass
class HighRecallRAGPipeline:
    """高召回率 RAG 管线"""

    vectorstore: Chroma
    corpus: List[str]
    llm: ChatOpenAI
    reranker: Optional[CrossEncoderReranker] = None
    enable_multi_query: bool = True
    enable_hyde: bool = False
    recall_k: int = 30
    final_k: int = 5

    def __post_init__(self):
        self.hybrid_retriever = HybridRetriever(
            self.vectorstore, self.corpus, top_k=self.recall_k
        )
        self.query_rewriter = QueryRewriter(self.llm)

    def retrieve(self, query: str) -> List[Document]:
        """完整检索流程"""
        all_candidates = []

        # 策略1：原始 query 直接检索
        docs = self.hybrid_retriever.retrieve(query)
        all_candidates.extend(docs)

        # 策略2：Multi-Query
        if self.enable_multi_query:
            variants = self.query_rewriter.multi_query(query)
            for variant in variants[1:]:  # 跳过原始 query
                docs = self.hybrid_retriever.retrieve(variant)
                all_candidates.extend(docs)

        # 策略3：HyDE
        if self.enable_hyde:
            hyde_doc = self.query_rewriter.hyde(query)
            docs = self.vectorstore.similarity_search(hyde_doc, k=self.recall_k)
            all_candidates.extend(docs)

        # 去重
        unique = self._deduplicate(all_candidates)

        # 重排序
        if self.reranker and len(unique) > self.final_k:
            unique = self.reranker.rerank(query, unique, top_k=self.final_k)

        return unique[:self.final_k]

    def _deduplicate(self, docs: List[Document]) -> List[Document]:
        seen = set()
        unique = []
        for doc in docs:
            if doc.page_content not in seen:
                seen.add(doc.page_content)
                unique.append(doc)
        return unique

    def generate(self, query: str) -> Dict:
        """检索 + 生成"""
        start_time = time.time()

        # 检索
        docs = self.retrieve(query)
        retrieval_time = time.time() - start_time

        # 构建上下文
        context = "\n\n".join([
            f"[文档{i+1}]\n{doc.page_content}"
            for i, doc in enumerate(docs)
        ])

        # 生成
        prompt = f"""基于以下参考文档回答问题。如果参考文档中没有相关信息，请直接说明。

参考文档：
{context}

问题：{query}

回答："""

        response = self.llm.invoke(prompt)
        generation_time = time.time() - start_time - retrieval_time

        return {
            "query": query,
            "answer": response.content,
            "sources": [
                {
                    "content": doc.page_content[:200] + "...",
                    "rerank_score": doc.metadata.get("rerank_score", None),
                    "rrf_score": doc.metadata.get("rrf_score", None),
                }
                for doc in docs
            ],
            "retrieval_time_ms": round(retrieval_time * 1000, 1),
            "generation_time_ms": round(generation_time * 1000, 1),
            "total_time_ms": round((time.time() - start_time) * 1000, 1),
        }


# ============================================
# 使用示例
# ============================================

def main():
    """完整使用示例"""

    # 1. 初始化组件
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vectorstore = Chroma(
        collection_name="tech-docs",
        embedding_function=embeddings,
        persist_directory="./chroma_db"
    )

    # 语料库（实际应从文档处理管线获取）
    corpus = [
        "RAG 是一种结合检索和生成的架构，召回率是关键指标。",
        "BM25 是经典的关键词匹配算法，与向量检索互补。",
        "RRF（Reciprocal Rank Fusion）是结果融合算法，公式为 Σ 1/(k+rank)。",
        "Cross-Encoder 重排序可以大幅提升检索精度。",
        "BGE-M3 是当前中文场景最强的开源 embedding 模型。",
    ]

    reranker = CrossEncoderReranker()

    # 2. 创建管线
    pipeline = HighRecallRAGPipeline(
        vectorstore=vectorstore,
        corpus=corpus,
        llm=llm,
        reranker=reranker,
        enable_multi_query=True,
        enable_hyde=False,
        recall_k=30,
        final_k=3,
    )

    # 3. 检索 + 生成
    query = "如何提升 RAG 的召回率？"
    result = pipeline.generate(query)

    print(f"问题: {result['query']}")
    print(f"回答: {result['answer']}")
    print(f"耗时: {result['total_time_ms']}ms (检索 {result['retrieval_time_ms']}ms)")
    print(f"来源数: {len(result['sources'])}")


if __name__ == "__main__":
    main()
```

## 10.4 性能与成本权衡

| 方案组合 | 召回率提升 | 延迟增加 | 成本增加 | 推荐场景 |
|---------|-----------|---------|---------|---------|
| 仅向量检索 | 基线 | <50ms | 低 | 原型验证 |
| + 混合检索 | +15-25% | +20ms | 低 | 生产基线 |
| + Multi-Query | +10-20% | +200ms | 中（LLM调用） | 追求召回 |
| + 重排序 | +5-15% | +100ms | 低（GPU推理） | 追求精度 |
| + HyDE | +5-10% | +500ms | 中（LLM调用） | 复杂问题 |
| 全方案叠加 | +40-70% | +800ms | 中高 | 极致效果 |

> 💡 **推荐实施路径：**
>
> **阶段1（1-2天）：** 优化切分 + 混合检索 → 快速拿到 +20-30% 收益
> **阶段2（3-5天）：** 查询重写 + 重排序 → 再拿 +15-25%
> **阶段3（1周+）：** 评估体系 + Embedding 微调 → 精细化调优
>
> 每个阶段完成后用评估数据验证效果，确认有效再进入下一阶段。

---

**小结：** 高召回率 RAG 管线不是某个"银弹"方案，而是多个方案的有机组合。理解每个方案的原理和权衡，按需组合，才能构建出既高效又可靠的检索系统。
# 11. 总结与建议路线图

## 11.1 各方案效果对比（ROI 矩阵）

| 方案 | 召回率提升 | 实施难度 | 延迟影响 | 成本 | 优先级 |
|------|-----------|---------|---------|------|--------|
| **① 优化切分策略** | +15-40% | ⭐⭐ | 无 | 无 | 🔴 P0 |
| **② Embedding 选型** | +15-30% | ⭐⭐ | 无 | 低 | 🔴 P0 |
| **③ 查询重写** | +10-25% | ⭐⭐ | +200ms | 中 | 🟡 P1 |
| **④ 混合检索** | +15-25% | ⭐⭐⭐ | +20ms | 低 | 🔴 P0 |
| **⑤ 元数据过滤** | +5-15% | ⭐⭐ | 无/低 | 低 | 🟡 P1 |
| **⑥ 重排序** | +5-15% | ⭐⭐⭐ | +100ms | 低(GPU) | 🟡 P1 |
| **⑦ 评估体系** | 间接 | ⭐⭐⭐ | 无 | 低 | 🔴 P0 |

> **说明：** 召回率提升数据来源于社区实践和公开论文，实际效果因场景而异。

## 11.2 推荐实施优先级

```
第1周 ─────────────────────────────────────
✅ 优化文档切分（Recursive + Parent-Child）
✅ 切换到合适的 Embedding 模型（BGE-M3）
✅ 加入 BM25 混合检索 + RRF
✅ 建立评估数据集 + Ragas 管线

第2周 ─────────────────────────────────────
✅ Multi-Query 查询重写
✅ Cross-Encoder 重排序
✅ 元数据提取 + 过滤

第3周 ─────────────────────────────────────
✅ HyDE（针对复杂场景）
✅ Embedding 微调（如有领域数据）
✅ 在线监控 + A/B 测试
✅ 性能调优（延迟/成本优化）
```

## 11.3 未来趋势

### Agentic RAG

传统 RAG 是"一次检索 → 生成"的线性流程，而 **Agentic RAG** 让检索器具备自主决策能力：

```
用户 Query → Agent 判断
              ├── 需要检索？→ 选择检索策略
              ├── 检索结果够吗？→ 不够则换策略 / 调整 query
              ├── 需要多步检索？→ 分解子问题，逐步检索
              └── 够了 → 生成答案
```

代表框架：
- **LangGraph**：基于图的状态机，支持多步检索决策
- **LlamaIndex Agent**：Tool-use 方式的自主检索
- **Self-RAG**：检索后自我评估是否需要更多检索

### 端到端微调（End-to-End RAG Fine-tuning）

将检索器和生成器联合微调，让 LLM 学会"在什么情况下检索、检索什么、如何使用检索结果"。

```python
# 概念性示例（非实际可运行代码）
# 训练数据: (query, ground_truth_context, ground_truth_answer)
# 损失函数: LLM生成loss + 检索相关性loss
# 目标: 让模型内化检索策略
```

### LLM-as-Judge 评估

用强 LLM（GPT-4 / Claude）作为评估器，替代传统指标：

```
Query + Retrieved Context + Generated Answer → LLM Judge
→ "相关度: 4/5, 准确性: 5/5, 完整性: 3/5"
```

优势：更贴近人类判断，能评估复杂质量维度。
劣势：成本高，一致性有待验证。

### 向量数据库的演进

| 趋势 | 说明 |
|------|------|
| **原生混合检索** | Elasticsearch 8.x+、Qdrant、Milvus 都已支持 BM25+向量 |
| **稀疏+稠密联合索引** | BGE-M3 的 multi-vector 特性需要数据库支持 |
| **GPU 加速检索** | cuVS（NVIDIA）提供 GPU 加速的 ANN 搜索 |
| **语义缓存** | 相似 query 直接返回缓存结果，减少检索和 LLM 调用 |

---

**写在最后：**

RAG 召回率优化没有银弹。它是一个**系统工程**——从文档切分到 embedding 选型，从查询重写到混合检索，从重排序到评估体系，每一环都可能成为瓶颈，每一环都可能带来提升。

**最有效的路径是：先诊断，再优化，边优化边评估。** 不要试图一次性上所有方案，而是循序渐进，用数据说话。

希望这篇文章能成为你构建高召回率 RAG 系统的实用指南。

---

## 附录 A：常用工具与库速查表

| 类别 | 工具/库 | 用途 | 链接 |
|------|--------|------|------|
| **切分** | LangChain Text Splitter | 递归/语义切分 | `pip install langchain-text-splitters` |
| **Embedding** | FlagEmbedding (BGE) | 中文 embedding + 重排序 | `pip install FlagEmbedding` |
| **Embedding** | sentence-transformers | 通用 embedding + 微调 | `pip install sentence-transformers` |
| **向量库** | Chroma | 轻量级本地向量存储 | `pip install chromadb` |
| **向量库** | Elasticsearch | 生产级混合检索 | Docker 部署 |
| **向量库** | Qdrant | Rust 高性能向量库 | Docker 部署 |
| **BM25** | rank_bm25 | Python BM25 实现 | `pip install rank-bm25` |
| **BM25** | Jieba | 中文分词 | `pip install jieba` |
| **重排序** | Cross-Encoder | 高精度重排序 | `pip install sentence-transformers` |
| **评估** | Ragas | RAG 评估框架 | `pip install ragas` |
| **框架** | LangChain | RAG 管线编排 | `pip install langchain` |
| **框架** | LlamaIndex | 数据索引框架 | `pip install llama-index` |
| **查询重写** | OpenAI / 通义千问 API | LLM 驱动重写 | API 调用 |
| **可视化** | Ragas Dashboard | 评估结果可视化 | `pip install ragas` |

## 附录 B：参考文献与延伸阅读

1. **RAG 原始论文**: Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks", NeurIPS 2020
2. **HyDE 论文**: Gao et al., "Precise Zero-Shot Dense Retrieval without Relevance Labels", ACL 2023
3. **Self-RAG 论文**: Asai et al., "Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection", ICLR 2024
4. **MTEB 榜单**: Muennighoff et al., "MTEB: Massive Text Embedding Benchmark", EACL 2023
5. **RRF 算法**: Cormack et al., "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning to Rank", SIGIR 2009
6. **BM25 算法**: Robertson et al., "The Probabilistic Relevance Framework: BM25 and Beyond", Foundations and Trends in IR, 2009
7. **Ragas 文档**: https://docs.ragas.io
8. **LangChain RAG 指南**: https://python.langchain.com/docs/rag/
9. **LlamaIndex 文档**: https://docs.llamaindex.ai
10. **BGE Embedding 论文**: Xiao et al., "C-Pack: Packaged Resources To Advance General Chinese Embedding", arXiv 2023
