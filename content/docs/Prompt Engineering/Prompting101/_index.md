---
title: (原理) 谷歌 Prompting guide 101 *
date: 2024-11-10 15:48:39
weight: 1 
---



# 四个核心要素
### **1. Persona（角色）**

**定义**：使用者身份/职业定位，决定AI的回应视角

**关键作用**：

- 帮助AI理解用户的专业背景和需求层次
- 使输出内容更符合特定角色的思维方式和表达习惯

**应用示例**：

- 行政支持："You are a program manager in [industry]"（项目管理者身份）
- 市场营销："I am a PR manager at [company name]"（公关经理身份）
- 技术岗位："As the CIO at [company]"（首席信息官身份）

**最佳实践**：

- 使用明确职称（如"executive assistant"）
- 补充行业属性（如"in the personal care industry"）
- 需保持角色一致性（同一会话中不宜切换角色）

---

### **2. Task（任务）**

**定义**：需要AI完成的具体工作指令

**关键作用**：

- 明确操作类型（生成/总结/分析/优化等）
- 界定输出内容的范围与深度

**应用示例**：

- 内容生成："Draft an executive summary email"（起草邮件）
- 数据分析："Identify trends in customer feedback"（趋势分析）
- 流程优化："Create a budget tracker for business travel"（创建模板）

**最佳实践**：

- 必须包含动词（如generate/summarize/analyze）
- 明确操作对象（邮件/表格/报告等）
- 分层递进（主任务→子任务）

---

### **3. Context（上下文）**

**定义**：任务相关的背景信息与约束条件

**关键作用**：

- 提供必要的业务场景信息
- 确保输出内容的准确性和相关性

**应用示例**：

- 行业背景："newly formed team of content marketers"（团队构成）
- 数据来源："based on @[Product Launch Notes]"（引用文件）
- 限制条件："within 10-minute walk of the hotel"（地理限制）

**最佳实践**：

- 使用占位符动态引用（[industry]/[date]等）
- 关联Workspace文件（@file语法调用云端文档）
- 说明特殊要求（如合规/隐私限制）

---

### **4. Format（格式）**

**定义**：期望的输出结构与呈现方式

**关键作用**：

- 规范内容组织逻辑
- 提高信息可读性和实用性

**应用示例**：

- 结构化数据："Put it in a table format"（表格格式）
- 视觉呈现："Create an image of a trade show booth"（图像生成）
- 文本规范："Limit to bullet points"（要点列表）

**最佳实践**：

- 明确格式类型（邮件/报告/图表等）
- 指定内容层级（标题/段落/项目符号）
- 设置量化指标（如"3 different icebreaker activities"）

---

# **综合应用案例**

**场景**：行政助理制定差旅行程

**提示词结构**：

`"I am an executive assistant. Create a 2-day business trip itinerary in [location] during [dates]. Include breakfast/dinner options within 10-minute walk of [hotel], and one entertainment option. Put it in a table."`

**要素拆解**：

- **Persona**：Executive assistant（角色定位）
- **Task**：Create itinerary（核心任务）
- **Context**：Business trip + location/time constraints（场景限制）
- **Format**：Table with specified columns（表格呈现）

---

# **优化建议**

1. **迭代优化**：通过"Make this a power prompt"指令让AI优化原始提示
2. **动态引用**：使用@file语法调用云端文档增强上下文关联性
3. **分层提示**：复杂任务分解为"主提示→细化提示→格式调整"多轮对话
4. **语气控制**：通过"formal/casual/upbeat"等形容词调整输出语调

文档强调，有效提示的平均长度约21词（含关键上下文），而用户初始尝试往往不足9词，建议通过这四个要素的系统应用提升提示质量。



# 参考
1. [Prompting Guide 101](https://services.google.com/fh/files/misc/gemini-for-google-workspace-prompting-guide-101.pdf)
   更偏向各行各业的提示案例和实践，
   围绕 Persona（角色）、Task（任务）、Context（上下文）、Format（格式）四要素展开：
   

【**腾讯元宝 生成**
根据文档内容，围绕"Persona（角色）、Task（任务）、Context（上下文）、Format（格式）"四个核心要素的解析如下：】