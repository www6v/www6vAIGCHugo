---
title: (原理|实战)Self-QA *
date: 2023-09-27 12:11:03
weight: 2
tags:
  - Data
categories: 
  - AIGC
  - Data  
---

<p></p>
<!-- more -->


# Self-QA[10]

### 论文

[SELF-QA: Unsupervised Knowledge Guided Language Model Alignment](https://arxiv.org/pdf/2305.11952)

### 思想

知识引导的指令生成Knowledge-Guided Instruction Generation

### 指令生成阶段

- 采用语言模型本身来根据无监督的文本生成指令。这种方法使生成的指令具有领域针对性，并与所提供的无监督文本的内容相关。
    - 非结构化的知识，如网页和书籍数据，直接使用。
    - **结构化数据**，如表格和知识图谱，在被利用之前需要**转换为非结构化文本数据**。如通过使用模板填充槽或将每个数据条目与相应的属性名称连接起来来实现。

### 指令答案生成阶段

- 将**生成的指令问题**让大模型进行预测，**生成答案**

# Self-QA 实战[11]

```python
SYSTEM_PROMPT = """
    你是一个能根据提供的文本内容生成QA对的机器人。以下是你的任务要求：
    1. 生成尽可能多的QA对。
    2. 每个QA对包含一个问题和一个简洁的答案。
    3. 答案必须用简体中文。
    4. 生成的QA对不能重复。
    5. 使用json格式将QA对包裹起来，问题用"question"表示，答案用"answer"表示。
    
    示例格式：
    [
        {
            "question": "...",
            "answer": "..."
        },
        {
            "question": "...",
            "answer": "..."
        }
    ]
    以下是给定的文本内容：
    """
```

# 参考

### Self-QA

10.《第二章 大模型训练与微调研发背后的数据艺术》 LLM大语言模型算法特训 那位科技 ***  
**SELF-INSTRUCT**， Baize， **Evol-instruct**， **Self-QA**， Ultra-chat  

11. [Self-QA：生成自然语言处理训练数据的实用方法](https://blog.csdn.net/m0_56090828/article/details/138361006)  

1xx. [项目实训2024.04.12日志：Self-QA生成问答对](https://blog.csdn.net/lyh20021209/article/details/137696063)  

 [SELF-QA：无监督知识引导语言模型对齐](https://blog.csdn.net/2202_75336422/article/details/140866160)  

[SELF-QA：无监督的知识引导语言模型对齐论文精读](https://zhuanlan.zhihu.com/p/659246821)  

