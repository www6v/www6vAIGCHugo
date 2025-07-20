---
title: (原理)Instruct Tuning 
date: 2023-01-06 19:09:20
weight: 1
tags:
  - Instruct-Tuning
categories: 
  - AIGC
  - Instruct-Tuning  
---

<p></p>
<!-- more -->

# In Context Learning ( ICL ) 上下文学习
![ICL](./images/ICL.webp)

+ **in context learning**，大意是在**prompt learning的基础上，将少量有标签样本融入prompt**。
+ 上图的ICL模型可以理解成**有监督、无训练**的**小样本学习**。
+ 但**并非所有ICL都不训练**。比如下图右上角的**FLAN**就是用instruction tuning**训练参数**的。

![ICL-tech](./images/ICL-tech.webp)
+ **FLAN**，**既属于 in context learning，也属于 instruction learning**


# Instruction Learning [1]
###  Instruct Tuning-
    FLANv1, FLANv2
### instructGPT

### chatGPT 


# Instruction Tuning
![instructTuning](./images/instructTuning.webp)

+ 对于已有的预训练模型，继续在多项任务（B、C、D等）上做训练，在其他任务（A）上做预测。**虽然依然没见过任务A，但是根据对B、C、D等的训练，对A的效果有所提升；** [1]

+ **Instruct Tuning 本质上也是Prompt Tuning** [2]


+ 研究了缩放对指令微调的影响 [3]
    与微调指令的任务数量有关，**任务数量越多效果越好**
    与模型的大小有关，**模型越大效果越好**

+ Prompt vs. Instruction Tuning  [4]
	Prompt是去激发语言模型的**补全能力**，比如给出上半句生成下半句、或者做完形填空，都还是像在做language model任务.
	而Instruction Tuning则是激发语言模型的**理解能力**，通过给出更明显的指令/指示，让模型去理解并做出正确的action
	**Prompt tuning**都是针对**一个任务**的，比如做个情感分析任务的prompt tuning，精调完的模型只能用于情感分析任务，而经过**Instruction Tuning多任务**精调后，可以用于其他任务的zero-shot

+ Instruction Tuning 指令微调  [4]
  - Self Instruction
    + Alpaca = LLaMA + Intruction Tuning [2]
    
# Limitation of instruction finetuning [2]
![limitation](./images/limitation.JPG)
问题1.  开放性问题
问题2.  看图


# 参考

1. [各种tuning的简单逻辑解释](https://zhuanlan.zhihu.com/p/619406727)

2. [第九课：Instruct Tuning](https://www.bilibili.com/video/BV1cm4y1e7Cc/) *** V

3. [FLANv2：大模型指令微调必看论文](https://zhuanlan.zhihu.com/p/646136859) 

4. [Instruction Tuning｜谷歌Quoc V.Le团队提出又一精调范式](https://zhuanlan.zhihu.com/p/408166011)

1xx. [June 2023, A Stage Review of Instruction Tuning](https://yaofu.notion.site/June-2023-A-Stage-Review-of-Instruction-Tuning-f59dbfc36e2d4e12a33443bd6b2012c2)

1xx. [【LLM系列之FLAN-T5/PaLM】Scaling Instruction-Finetuned Language Models](https://zhuanlan.zhihu.com/p/629461665)

1xx. [如何优化大模型的In-Context Learning效果？](https://zhuanlan.zhihu.com/p/597036814)

1xx. [Instruction Tuning（FLAN、instructGPT、chatGPT）](https://nakaizura.blog.csdn.net/article/details/128265846)

