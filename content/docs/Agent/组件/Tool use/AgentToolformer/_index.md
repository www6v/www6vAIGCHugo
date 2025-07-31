---
title: (Work)[SFT]Toolformer
date: 2023-02-03 22:26:11
weight: 11
tags:
  - Tool
categories:
  - AIGC  
  - Tool
---

<p></p>
<!-- more -->

# 论文
+ 论文地址
 [Toolformer: Language Models Can Teach Themselves to Use Tools](https://arxiv.org/abs/2302.04761)  

+ 开源地址
[Implementation of Toolformer](https://github.com/lucidrains/toolformer-pytorch)  git

# Toolformer[1]
  - 🔑关键词和摘要
    - Keywords: Large-scale PLMs,  Tool Learning
    - xxx
      - 驱动语言模型去使用简单的模型来调用外部的工具
      - Toolformer通过语言模型的方法去决定去调用哪些API，传入哪些参数
      - Tooformer是在自监督层面执行的，只需要对每个API的语言描述
  - ⚙️研究设计和结论
    - 方法   
      - Toolformer调用示例：xxx
      - 关键要素：
        - 模型对工具的使用应该是自监督的，这样可以省去很大的标注开销
        - 模型应该自行地去决定在何时间，用何方法来调用工具
      - **方法概要：**
        - 受到in-context learning的启发，给定少量的人写的关于API的描述，让模型去自行生成潜在API调用的语言建模数据
        - 构建一个自监督的Loss函数，让模型来决定哪些API的调用有助于它的语言建模的预测
      - **方法细节：**
        - xxx
          - 给定一个纯文本数据集，构建出一个带有API调用的数据集，然后在此数据集上做微调
          - 第一步：使用in-context learning来生成大量的潜在可能的API调用
          - 第二步：执行这些API，返回得到结果
          - 第三步：检查返回的结果是否有助于语言模型的预测，过滤掉其他的API
        - API调用采样
          - 给每一个API来撰写提示来鼓励模型使用这些API，例如QA的提示是 xxx
          - 对于文本的每一个位置，如果这个位置是<API>（即API调用的开始）的概率大于一个阈值，则将此位置保留到一个集合I中
          - 对于集合I中的每一个位置，通过模型生成最多m个API调用，并且以</API>结尾（如果生成的调用没有以</API>结尾，直接舍去）
        - API执行
          - 去执行所有的API调用，返回文本序列
        - API过滤
          - 构建自监督的语言模型的loss函数
          - 第一个的含义：进行API的调用，并且使用API结果的Loss
          - 第二个的含义：空字符串的Loss和调用API但不返回结果Loss的最小值
          - 这时我们希望模型使用API并且返回结果对语言建模有帮助，且帮助很明显->前者的loss显著比后者小
        - 微调和推理
          - 在经过如上操作后，就可以得到带有API调用的数据集，然后将模型在上面进行微调
          - 当模型在解码阶段输出"->"符号时，意味着需要调用API了，调用得到返回结果然后拼接上去
    - 实验
      - 模型：GPT-J （67亿参数）
      - 原始数据：CCNet
      - 知识探测任务LAMA
        - Toolformer可以大幅超过之前的方法，甚至是GPT-3等大模型
      - 数学数据集
      - 问答
      - 这里即使是Toolformer也无法超越GPT-3，可见预训练规模可以囊括更多知识
      - 模型规模的影响
      - 模型的参数量到一定规模后才拥有使用工具的能力
  - 📚论文贡献
    - 优点
      - 将语言模型使用外部工具的进行很自然的结合
      - **不需要标注大量数据，使用自监督的方法进行学习**
    - 缺点
      - **工具无法交互，也无法链式使用（每个API调用都是独立的）**
      - 定义的工具尚且有限，扩展工具则需要用模型标注新的数据
      - 随着基础模型zero-shot能力的增强，这种需要构建数据并且fine-tune的做法可能会比较麻烦
  - OpenBMB BMTools: https://github.com/OpenBMB/BMTools



# 参考
1. [清华博士带你搞懂大模型自学工具使用（Toolformer)【论文速读】](https://www.bilibili.com/video/BV18s4y1u7nJ/) V 有思维导图  
1xx. [使LLM善假于物: Toolformer ](https://finisky.github.io/toolformer-summary/)  
1xx. [Prompt Engineering ](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/#external-apis)  
1xx. [Toolformer and Tool Learning（LLMs如何使用工具）](https://nakaizura.blog.csdn.net/article/details/130817902)  
