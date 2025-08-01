---
title: (List)Agent 产品 平台
date: 2023-03-05 16:48:28
tags:
  - Agent
categories: 
  - AIGC
  - Agent  
---

<p></p>
<!-- more -->




# 应用 
### 分类 [10][11][12]
+ Action agents  
    - Function Call
    - ReACT
    
+ Simulation agents 
    生成式智能体， CAMEL，  Generative Agents
    
+ Automomous Agent
    **AutoGPT**， **BabyAGI**,  **AutoGen**
    **MetaGPT**     ChatDev
    
+ 跨模态Agents
    HuggingGPT

### HuggingGPT 

### BabyAGI  [AIGC]
Plan-and-execute agents 
The **planning** is almost always done **by an LLM**.
The **execution** is usually done by a **separate agent (equipped with tools)**.

### AutoGPT[10]
AutoGPT 的核心逻辑是一个 Prompt Loop，步骤如下

1. AutoGPT 会基于一定策略自动组装 Command Prompt，这些首次会包含用户输入的 Name, Role和Goals 
2. Command Prompt 的目标不是为了拿到最终结果，而是通过 GPT Chat API(Thinking 的过程)返回下一步的 Command (包含name和arguments, 如`browser_website(url = "www.baidu.com")` )
3. 这些 Command 都是可扩展的，每一种命令代表一种外部能力(比如爬虫、Google搜索，也包括GPT的能力)，通过这些 Command 调用返回的 Result 又会成为到 Command Prompt 的组成元素，
4. 回到第 1 步往复循环，直到拿到最终结果结果（状态为“compelete”）

# Platform[20]
### 字节 Coze[21,22]
优势:  有RAG，结构化数据  
劣势:  只能发布到飞书，微信  

### 百度 AppBuilder  

### Dify  

# 参考
1xx. [「Agent」通俗易懂地聊聊AI Agent（附66个开源+44个闭源Agent项目）](https://zhuanlan.zhihu.com/p/664281311)  

1xx. [主流Agent框架及金融Agent-FinRobot：兼看面向实体增强的细粒度实体描述知识库项目 ](https://mp.weixin.qq.com/s/IubrsvB0ypn8KC-_SMPvLQ)  
   Agent-FinRobot  基于autogen 实现  

### xxx
10. [2023年新生代大模型Agents技术,ReAct,Self-Ask,Plan-and-execute,以及AutoGPT, HuggingGPT等应用](https://zhuanlan.zhihu.com/p/642357544) ***  论文+代码  
11. 公开课  
12. 公开课  

### Platform
20. [AgentBuilder 中小企业如何选择：coze、dify、appbuilder、毕晟](https://www.bilibili.com/video/BV1Bm411B79m/) V  
21. [COZE：中小企业均可0门槛创建业务agent](https://www.bilibili.com/video/BV1DA4m1w72p/) V  
22. [利用Coze 实现吴恩达的4种 AI Agent 设计模式](https://mp.weixin.qq.com/s/WDkYZF9-JRhzM467Uf8lUA)  


###  产品 &调研
1xx. [2024 年最完整的 AI Agents 清单来了，涉及 13 个领域，上百个 Agents！ ](https://mp.weixin.qq.com/s/RrymA1NwHzz9Q34wITZo6w)  开源 闭源 ***  
1xx. [全球AI Agent大盘点，大语言模型创业一定要参考的60个AI智能体](https://mp.weixin.qq.com/s/v_BvcMqd-FpwRbOD09MR0A)  
1xx. [大模型时代的APP--2024年 AI Agent行业报告](https://mp.weixin.qq.com/s/KbDOBkmYsTiwkjg2-YoRrQ) ***  

### modelscope-agent  AgentFabric
1xx. [LLM 大模型学习必知必会系列(十)：基于AgentFabric实现交互式智能体应用,Agent实战-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2421616)  

1xx. [modelscope-agent/apps/agentfabric at master · modelscope/modelscope-agent](https://github.com/modelscope/modelscope-agent/tree/master/apps/agentfabric)  

1xx. [Modelscope Agent实操（一）：0代码创建、发布并分享一个专属Agent](https://zhuanlan.zhihu.com/p/669397935)  

1xx. [从agentfabric开始体验魔搭Agent](https://zhuanlan.zhihu.com/p/672235050)  

1xx. [社区供稿 | GLM-4适配ModelScope-Agent最佳实践](https://zhuanlan.zhihu.com/p/679918404)  