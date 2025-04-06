---
title: (原理|实战)Query Rewrite
date: 2023-04-20 22:51:05
weight: 1
tags:
  - RAG
categories: 
  - AIGC
  - RAG
---

<p></p>
<!-- more -->


# Query rewrite
### query rewrite [1][2]
+ [论文](https://arxiv.org/pdf/2305.14283.pdf)**使用LLM重写用户查询**，而不是直接使用原始用户查询进行检索。
因为对于LLM 而言，**原始查询不可能总是最佳检索结果**，可以让LLM重写查询。
+ [Repo](https://github.com/langchain-ai/langchain/blob/master/cookbook/rewrite.ipynb) git
【问题的多样化】
### Transformation-多样性

# Step Back
### Step Back问答回退策略 [3]
Step Back问答回退，首先提示LLM提出一个**关于高级概念或原则的通用后退问题**，并检索有关它们的相关事实，使用此基础来帮助回答用户问题。

### Step-back Prompting [1][2]
+ [论文](https://arxiv.org/pdf/2310.06117.pdf)使用退一步提示，**使用LLM生成"后退"(Step back prompting)问题**。
使用检索时，"后退"问题和原始问题都会被用来进行检索，然后这两个结果都会被用来作为语言模型回复的基础。
+ [Repo](https://github.com/langchain-ai/langchain/blob/master/cookbook/stepback-qa.ipynb) git
【问题的抽象化】
### Transformation-抽象化

# HyDE
### HyDE混合策略[3]
LLM将**问题**转换为回答问题的**假设文档**。**使用嵌入的假设文档检索真实文档**，前提是doc-doc相似性搜索可以产生更多相关匹配。

+ HyDE
At a high level, HyDE is an embedding technique that takes queries, **generates a hypothetical answer**, and then embeds that generated document and uses that as the final example.

### Transformation-具体化

# 参考
1. [知识图谱用于细粒度大模型幻觉评估：兼论Langchain-RAG问答中的问题改写范式 ](https://mp.weixin.qq.com/s?__biz=MzAxMjc3MjkyMg==&mid=2648406156&idx=1&sn=d91a4df105c4fc4c9523f7141bc1c24d) 
    RAG:  rewrite , Step back, fusion  

2. [Query Transformations](https://blog.langchain.dev/query-transformations/)  

3. [一文详看Langchain框架中的RAG多阶段优化策略：从问题转换到查询路由再到生成优化](https://mp.weixin.qq.com/s/pK2BRLrWpEKKIPFhUtGvcg) ***   原理paper，代码示例  
   Multi Query多查询策略， Decomposition问题，RAG-Fusion， Step Back， HyDE混合  
   [rag-from-scratch Repo](https://github.com/langchain-ai/rag-from-scratch) git  
   [RAG(检索增强） 从入门到精通 虚拟文档嵌入（Hyde)](https://www.bilibili.com/video/BV1Vx421U7a4/) V  
   

1xx. [业界总结｜搜索中的Query理解](https://zhuanlan.zhihu.com/p/393914267) ***  

1xx. [智能扩充机器人的“标准问”库之Query生成](https://zhuanlan.zhihu.com/p/149429784)  

[LLM之RAG实战（二十八）| 探索RAG query重写](https://zhuanlan.zhihu.com/p/685981587)


[高级RAG检索中的五种查询重写策略](https://zhuanlan.zhihu.com/p/699118668)

