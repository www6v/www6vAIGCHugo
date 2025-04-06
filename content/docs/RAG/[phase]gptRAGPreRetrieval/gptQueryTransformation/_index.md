---
title: (原理|实战)Query Transformation
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


# Query Transformation
### Multi Query多查询策略[3]
该方法**从多个角度重写用户问题**，为每个重写的问题检索文档，返回所有查询的唯一文档。

### Decomposition问题分解策略[3]
+ Answer recursively迭代式回答
  在问题分解的基础上，逐步迭代出答案，**将上一步问题的答案，与下一步骤的答案进行拼接**，送入大模型进行问答

+ Answer individually
  也可以**让每个subquery分别进行处理**，然后得到答案，然后再拼接成一个QA pairspprompt最终形成答案。



# 参考
1. xxx

2. xxx

3. [一文详看Langchain框架中的RAG多阶段优化策略：从问题转换到查询路由再到生成优化](https://mp.weixin.qq.com/s/pK2BRLrWpEKKIPFhUtGvcg) ***   原理paper，代码示例  
   [Multi Query多查询策略， Decomposition问题]， RAG-Fusion， Step Back， HyDE混合  
   [rag-from-scratch Repo](https://github.com/langchain-ai/rag-from-scratch) git  
   

