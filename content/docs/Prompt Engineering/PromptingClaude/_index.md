---
title: (Anthropic)[Claude]Prompting Engineering  * 
weight: 1
# bookFlatSection: false
# bookCollapseSection: false
---



# Beginner[1]

### Prompt generator [10]

[Prompt generator](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prompt-generator)

[提示生成器](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/prompt-generator)



### Be clear and direct

[Be clear and direct](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/be-clear-and-direct)

[清晰直接](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/be-clear-and-direct)



### Give Claude a role (system prompts) *** 

[Give Claude a role (system prompts)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/system-prompts)

[给 Claude 一个角色（系统提示）](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/system-prompts)



{{% details "example" close %}} 

您是一家世界500强科技公司的总法律顾问。我们正在考虑将这份软件许可协议用于我们的核心数据基础设施：
<contract>
{{CONTRACT}}
</contract>

分析其潜在风险，重点关注赔偿、责任和知识产权所有权。请给出您的专业意见。

{{% /details %}}

#  Intermediate[1]

### Use XML tags ***

[Use XML tags](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/use-xml-tags)

[使用 XML 标签](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/use-xml-tags)



{{% details "example" close %}}

你是AcmeCorp的财务分析师。为我们的投资者生成Q2财务报告。

AcmeCorp是一家B2B SaaS公司。我们的投资者重视透明度和可行的见解。

使用这些数据生成报告：<data>{{SPREADSHEET_DATA}}</data>

<instructions>

1. 包括以下部分：收入增长、利润率、现金流。
2. 突出优势和需要改进的领域。

</instructions>

使用简洁专业的语气。遵循这个结构：
<formatting_example>{{Q1_REPORT}}</formatting_example>

 {{% /details %}}



### Let Claude think (chain of thought) *** 

[Let Claude think (chain of thought)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-of-thought)

[让 Claude 思考（思维链）](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/chain-of-thought)



{{% details "example" close %}} 

你是一名财务顾问。一位客户想投资10,000美元。他们可以在两个选项中选择：A）一支历史年回报率为12%但波动的股票，或 B）一支保证年回报率6%的债券。客户需要在5年内用这笔钱作为房子的首付。你推荐哪个选项？【请逐步思考。】

{{% /details %}}



### Use examples (multishot) *** 

[Use examples (multishot)](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting)

[使用示例（多样本）](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/multishot-prompting)



{{% details "example" close %}} 

我们的客服团队被非结构化反馈淹没了。你的任务是为我们的产品和工程团队分析反馈并对问题进行分类。使用这些类别：UI/UX、性能、功能请求、集成、定价和其他。同时评估情感（积极/中性/消极）和优先级（高/中/低）。这里有一个示例：

<example>
输入：新仪表板一团糟！加载需要很长时间，而且我找不到导出按钮。请尽快修复这个问题！
类别：UI/UX、性能
情感：消极
优先级：高</example>

现在，分析这个反馈：{{FEEDBACK}}

{{% /details %}}



### Prefill Claude’s response

[Prefill Claude’s response](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/prefill-claudes-response)

[预填充 Claude 的响应](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/prefill-claudes-response)



{{% details "example" close %}} 

#### 示例 1：控制输出格式并跳过前言

{{% /details %}}

# Advanced[1]

### Chain complex prompts *** 

[Chain complex prompts](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/chain-prompts)

[链接复杂提示](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/chain-prompts)



## 如何链式提示

1. **识别子任务**：将任务分解为不同的、连续的步骤。
2. **使用XML构建清晰的交接**：使用XML标签在提示之间传递输出。
3. **设定单一任务目标**：每个子任务应该有一个明确的单一目标。
4. **迭代**：根据Claude的表现改进子任务。



{{% details "example" close %}} 

事例：分析法律合同

{{% /details %}}



### Long context tips

[Long context tips](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/long-context-tips)

[长上下文提示](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/long-context-tips)

{{% details "example" close %}}

``` xml
```

<documents>
  <document index="1">
    <source>annual_report_2023.pdf</source>
    <document_content>
      {{ANNUAL_REPORT}}
    </document_content>
  </document>
  <document index="2">
    <source>competitor_analysis_q2.xlsx</source>
    <document_content>
      {{COMPETITOR_ANALYSIS}}
    </document_content>
  </document>
</documents>

分析年度报告和竞争对手分析。识别战略优势并推荐第三季度重点关注领域。

```xml

```

{{% /details %}}

# 参考

1. Anthropic提示工程指南：Prompt Engineering Overview

https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview

https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/overview



+ Anthropic交互式提示工程教程：Prompt Engineering Interactive Tutorial

  https://github.com/anthropics/prompt-eng-interactive-tutorial

​	[Anthropic's Prompt Engineering Interactive Tutorial](https://github.com/anthropics/courses/tree/master/prompt_engineering_interactive_tutorial)

​	[Prompt Engineering Interactive Tutorial](https://docs.google.com/spreadsheets/d/1jIxjzUWG-6xBVIa2ay6yDpLyeuOh_hR_ZB75a47KX_E/edit?gid=869808629#gid=869808629)

+ https://blog.zhexuan.org/archives/Anthropic-Prompt.html

### prompt-generator

10. [自动生成首版提示词模板](https://docs.anthropic.com/zh-CN/docs/build-with-claude/prompt-engineering/prompt-generator)

​       https://console.anthropic.com/dashboard

