---
title: (原理|实战)RAG Fusion
date: 2023-05-14 18:23:41
weight: 2
tags:
  - RAG
categories: 
  - AIGC
  - RAG  
---

<p></p>
<!-- more -->


### RAG-Fusion多查询结果融合策略
将多个召回查询的结果进行**合并**[3]

其思想在于通过生成多个用户查询和重新排序结果来解决RAG固有的约束；利用倒数排序融合（RRF）和自定义向量评分加权，生成全面准确的结果。[2]

### 代码
[RAG Fusion](https://github.com/langchain-ai/langchain/blob/master/cookbook/rag_fusion.ipynb) git 

# 参考
2. [Forget RAG, the Future is RAG-Fusion](https://towardsdatascience.com/forget-rag-the-future-is-rag-fusion-1147298d8ad1)  失效
   [使用RAG-Fusion和RRF让RAG在意图搜索方面更进一步](https://mp.weixin.qq.com/s/N7HgjsqgCVf2i-xy05qZtA)
 
   [再谈大模型RAG问答中的三个现实问题：兼看RAG-Fusion多query融合策略、回答引文生成策略及相关数据集概述](https://mp.weixin.qq.com/s/NFjn8pUsQaSx85nhBphORA)    
   二、基于大模型生成能力自动生成引文

3. [一文详看Langchain框架中的RAG多阶段优化策略：从问题转换到查询路由再到生成优化](https://mp.weixin.qq.com/s/pK2BRLrWpEKKIPFhUtGvcg) ***   原理paper，代码示例  
   Multi Query多查询策略， Decomposition问题， [RAG-Fusion]， Step Back， HyDE混合  
   [rag-from-scratch Repo](https://github.com/langchain-ai/rag-from-scratch) git  
      
    