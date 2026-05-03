# OpenClaw 工具调用全流程：从模型输出到结果回传的源码级拆解

> **基于 OpenClaw v2026.4.21 源码深度分析**
> 作者：小伟 | 2026-04-30

---

## 导读

OpenClaw 的工具调用系统是其 Agent 能力的核心。模型输出 `tool_calls` 后，OpenClaw 并不是简单地"找到函数然后调用"，而是经历了一整套严密的流程：工具发现 → 策略过滤 → 参数归一化 → `before_tool_call` 钩子 → 分类分发执行 → 结果截断与媒体提取 → `after_tool_call` 钩子 → 写回 transcript → 下一轮 LLM 调用。

本文将从源码级别完整拆解这一流程。

---

## 1. 总览：工具调用架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OpenClaw Gateway                             │
│                                                                     │
│  ┌─────────┐    ┌──────────────┐    ┌──────────────────────────┐   │
│  │  Model   │───▶│ Tool Policy  │───▶│  Tool Definition         │   │
│  │  Stream  │    │  Pipeline    │    │  Builder                 │   │
│  └─────────┘    └──────────────┘    └──────────────────────────┘   │
│                                    │                                │
│                                    ▼                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Agent Loop (selection-DmkxuIQC.js)              │   │
│  │                                                              │   │
│  │  1. 接收 assistant tool_calls                                │   │
│  │  2. buildEmbeddedAttemptToolRunContext() 构建执行上下文      │   │
│  │  3. before_tool_call 钩子（loop 检测、诊断）                 │   │
│  │  4. dispatchToolExecution() 分类分发执行                     │   │
│  │  5. sanitizeToolResult() 结果清洗 + truncation               │   │
│  │  6. after_tool_call 钩子                                     │   │
│  │  7. 写入 session transcript (JSONL)                          │   │
│  │  8. 触发下一轮 LLM 调用                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                    │                                │
│                                    ▼                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  exec    │  │  read    │  │ message  │  │  channel tools   │   │
│  │  process │  │  write   │  │  send    │  │  MCP / plugin    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 工具发现与注册

### 2.1 工具来源分类

OpenClaw 的工具来自 4 个渠道：

| 来源 | 源码模块 | 说明 |
|---|---|---|
| 内置 Bash 工具 | [bash-tools.exec.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec.ts) / [bash-tools.process.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.process.ts) | `exec` 和 `process`，核心执行能力 |
| OpenClaw 内置工具 | [openclaw-tools.registration.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-tools.registration.ts) | `read`/`write`/`message`/`web_search`/`web_fetch`/`sessions_*`/`cron`/`tts` 等 |
| Channel 插件工具 | [channel-tools.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/channel-tools.ts) | 由 Channel 插件注册（如钉钉的 `send_message`、Discord 的 `reaction`） |
| MCP / Plugin 工具 | [openclaw-plugin-tools.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-plugin-tools.ts) | 由外部插件或 MCP Server 提供 |

### 2.2 工具策略过滤（Tool Policy Pipeline）

工具在暴露给模型之前，必须通过策略管道：

**源码**：[tool-policy-pipeline.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-pipeline.ts)

```typescript
// src/agents/tool-policy-pipeline.ts

// 默认管道步骤：
const DEFAULT_PIPELINE_STEPS = [
  'tool-name-allowlist',     // 名称白名单过滤
  'owner-only-policy',        // 仅 owner 可用工具
  'tool-profile-policy',      // 工具 profile（safe-bin 等）
  'tool-fs-policy',           // 文件系统路径限制
  'sandbox-policy',           // 沙箱策略
];

function applyToolPolicyPipeline(params) {
  for (const step of pipelineSteps) {
    const result = step.evaluate(toolName, args, context);
    if (result.deny) return { allowed: false, reason: result.deny };
  }
  return { allowed: true };
}
```

**策略匹配**（[tool-policy-match.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-match.ts)）：

```typescript
// src/agents/tool-policy-match.ts
function isToolAllowedByPolicies(params) {
  // 1. 检查 allow / deny / alsoAllow 列表
  // 2. 检查 owner-only 策略
  // 3. 检查 tool profile（safeBin / fullAccess 等）
  // 4. 检查 fs workspace 限制
  // 5. 检查 subagent tool policy
}
```

### 2.3 工具定义构建 → 发给模型

**源码**：[pi-tool-definition-adapter.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tool-definition-adapter.ts)

通过 `toClientToolDefinitions()` 将内部工具定义转换为模型 API 需要的 JSON Schema 格式（OpenAI function calling / Anthropic tool_use 等），同时检测工具名冲突。

---

## 3. Agent Loop 中的工具调用入口

### 3.1 LLM 返回 tool_calls

模型流式输出结束后，Agent Loop 判断是否存在 `tool_calls`：

**源码**：[selection-DmkxuIQC.js](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/selection.ts)

```typescript
// src/agents/pi-embedded-runner/selection.ts

// 模型输出完成后判断是否有 tool_calls
const stopReason = attempt.clientToolCall
  ? "tool_calls"
  : attempt.yieldDetected
    ? "end_turn"
    : sessionLastAssistant?.stopReason;

if (stopReason === "tool_calls") {
  // 进入工具执行流程
  await executeToolCalls(ctx, toolCalls);
}
```

### 3.2 构建工具执行上下文

**源码**：[attempt.tool-run-context.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts)

每个 tool call 都会通过 `buildEmbeddedAttemptToolRunContext()` 构建执行上下文：

```typescript
// src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts

function buildEmbeddedAttemptToolRunContext(params) {
  return {
    toolName,
    toolCallId,
    args,
    toolResultFormat: params.toolResultFormat ?? "markdown",
    // 图片理解模型配置
    // 媒体处理配置
    // session 可见性策略
    // sandbox 配置
  };
}
```

---

## 4. before_tool_call 钩子

在工具实际执行前，有一个钩子层：

**源码**：[pi-tools.before-tool-call.runtime.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tools.before-tool-call.runtime.ts)

```typescript
// src/agents/pi-tools.before-tool-call.runtime.ts

const beforeToolCallRuntime = {
  getDiagnosticSessionState,   // 获取诊断状态
  logToolLoopAction,           // 记录 tool loop 动作
  detectToolCallLoop,          // 检测工具调用循环
  recordToolCall,              // 记录 tool call
  recordToolCallOutcome        // 记录 tool call 结果
};
```

### 4.1 Tool Loop 检测

**源码**：[tool-loop-detection.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-loop-detection.ts)

```typescript
// src/agents/tool-loop-detection.ts

const DEFAULT_LOOP_DETECTION_CONFIG = {
  enabled: false,
  historySize: 30,              // 保留最近 30 次调用
  warningThreshold: 10,         // 警告阈值
  criticalThreshold: 20,        // 严重阈值
  globalCircuitBreakerThreshold: 30,  // 全局熔断阈值
  detectors: {
    genericRepeat: true,         // 通用重复检测
    knownPollNoProgress: true,   // 已知轮询无进展
    pingPong: true              // ping-pong 模式检测
  }
};

// 工具调用哈希：toolName + 参数的稳定 JSON 序列化摘要
function hashToolCall(toolName, params) {
  return `${toolName}:${digestStable(params)}`;
}

// 提取未知工具名
function extractUnknownToolName(error) {
  const raw = formatErrorForHash(error).trim();
  const toolName = (raw.match(/unknown tool[:\s]+["']?([a-z0-9_.-]+)["']?/i)
    ?? raw.match(/tool\s+["']?([a-z0-9_.-]+)["']?\s+(?:not found|is not available)/i))?.[1]?.trim();
  return toolName ? toolName.toLowerCase() : void 0;
}
```

**三种检测器**：

| 检测器 | 原理 | 阈值 |
|---|---|---|
| `genericRepeat` | 相同 toolName + 相同参数哈希重复出现 | `warningThreshold` (10) |
| `knownPollNoProgress` | `command_status`/`process poll` 连续无新进展 | `unknownToolThreshold` (10) |
| `pingPong` | 两个工具交替调用无实质进展 | `criticalThreshold` (20) |

---

## 5. 工具参数归一化与校验

### 5.1 参数归一化

**源码**：[attempt.tool-call-normalization.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts)

在分发执行前，对工具参数进行标准化处理：

- 大小写归一化（工具名统一小写）
- 字符串值校验
- 可选参数默认值填充
- 类型转换

### 5.2 参数 Schema 校验

**源码**：[pi-tools.params.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tools.params.ts)

每个工具都有基于 TypeBox 定义的 JSON Schema，入参在执行前会被校验。

---

## 6. 工具分类分发执行

工具根据名称被分类到不同的执行器。以下是核心分类：

### 6.1 Bash 工具（exec + process）

**源码**：[bash-tools.exec.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec.ts) / [bash-tools.process.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.process.ts)

| 工具 | 功能 | 关键参数 |
|---|---|---|
| `exec` | 执行 shell 命令（一次性） | `command`, `timeoutSec`, `background` |
| `process` | 管理长驻进程 | `action` (start/poll/log/write/send-keys/submit/kill) |

```typescript
// src/agents/bash-tools.exec.ts
// exec 工具的 JSON Schema
const execSchema = Type.Object({
  command: Type.String({ description: "Shell command to execute" }),
  timeout: Type.Optional(Type.Number()),
  background: Type.Optional(Type.Boolean()),
  // ... 更多参数
});
```

### 6.2 文件系统工具

**源码**：[openclaw-tools.registration.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-tools.registration.ts)

| 工具 | 功能 |
|---|---|
| `read` | 安全读取工作区文件 |
| `write` | 安全写入工作区文件 |

所有文件操作都经过 `readFileWithinRoot()` / `writeFileWithinRoot()`，限制在工作区范围内。

### 6.3 消息工具

**源码**：[message-tool.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tools/message-tool.ts)

```typescript
// src/agents/tools/message-tool.ts
// message 工具支持多种操作：
// - send: 发送消息
// - thread-reply: 回复线程
// - reaction: 添加表情
// - poll: 发起投票
```

### 6.4 Channel 插件工具

**源码**：[channel-tools.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/channel-tools.ts)

每个 Channel 插件（钉钉、Discord、Telegram 等）可以注册自己的工具：

```typescript
// src/agents/channel-tools.ts
function resolvePluginTools(params) {
  // 列出 Channel 插件注册的所有工具
  // 合并到 Agent 的工具列表中
}
```

### 6.5 MCP 工具

**源码**：[pi-bundle-mcp-tools.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/cli-runner/pi-bundle-mcp-tools.ts)

通过 `materializeBundleMcpToolsForRun()` 将 MCP Server 的工具发现并注册为 OpenClaw 工具。

---

## 7. 工具执行结果处理

工具执行完成后，结果需要经过一系列处理才能写回 transcript。

### 7.1 结果清洗与截断

**源码**：[attempt.tool-run-context.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts)

```typescript
// src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts

// sanitizeToolResult() - 清洗工具结果
function sanitizeToolResult(params) {
  // 1. 去除敏感信息（API keys 等）
  // 2. 限制结果大小
  // 3. 编码规范化
}
```

**结果截断**（[tool-result-truncation.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-truncation.ts)）：

```typescript
// src/agents/pi-embedded-runner/tool-result-truncation.ts

// resolveLiveToolResultMaxChars() - 动态计算最大字符数
function resolveLiveToolResultMaxChars(params) {
  // 根据当前上下文窗口剩余空间动态调整
  // 避免单个工具结果吃掉太多 token
}
```

### 7.2 媒体提取

**源码**：[tool-media-payloads.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/tool-media-payloads.ts)

工具结果中可能包含图片/音频/视频 URL，需要提取并转为 payload：

```typescript
// src/agents/pi-embedded-runner/run/tool-media-payloads.ts

function extractToolResultMediaArtifact(params) {
  // 从工具结果中提取媒体 URL
  // 下载并转 Base64
  // 限制大小（maxImageBytes）
  // 返回 media block 供下一轮 LLM 使用
}
```

### 7.3 结果格式转换

```typescript
// src/agents/pi-embedded-runner/selection.ts
const resolvedToolResultFormat = params.toolResultFormat
  ?? (channelHint
    ? isMarkdownCapableMessageChannel(channelHint) ? "markdown" : "plain"
    : "markdown");
```

根据 Channel 能力决定 tool result 格式：支持 Markdown 的 Channel 用 `markdown`，否则用 `plain`。

### 7.4 Session 结果结果保护

**源码**：[session-tool-result-guard.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-tool-result-guard.ts) / [session-tool-result-guard-wrapper.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/session-tool-result-guard-wrapper.ts)

确保 session 管理工具（`sessions_send` 等）的结果不会泄露敏感信息：

```typescript
// src/agents/session-tool-result-guard.ts
// - 隐藏目标 session 的私密消息内容
// - 仅暴露摘要信息
```

---

## 8. after_tool_call 钩子

**源码**：[selection-DmkxuIQC.js](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/selection.ts)

工具执行完成后：

```typescript
// 跟踪工具执行开始数据
const toolStartData = new Map();

// 工具执行完成后：
function buildToolStartKey(runId, toolCallId) {
  return `${runId}:${toolCallId}`;
}

// after_tool_call 钩子记录：
// - 工具名称
// - 参数摘要
// - 执行结果
// - 耗时
```

---

## 9. 写回 Transcript → 下一轮 LLM

### 9.1 JSONL Transcript 写入

工具结果最终以 `tool_result` 消息格式写入 session 的 JSONL transcript 文件：

```jsonl
// ~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
{"type":"tool_result","tool_call_id":"toolu_xxx","content":"命令输出结果...","timestamp":"2026-04-30T03:15:00Z"}
```

### 9.2 结果回传给模型

写回后，Agent Loop 自动触发下一轮 LLM 调用，将 tool_result 作为 `user` 角色消息传入：

```
assistant: 我来执行 ls 命令 [tool_calls: exec]
  └→ tool_result: "file1.txt  file2.txt  README.md"
       └→ 下一轮 LLM 调用，看到工具结果
```

---

## 10. 完整调用序列图

```
┌────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌────────────┐
│ LLM    │  │ Agent Loop   │  │ Policy+Hook  │  │ Tool       │  │ Transcript │
│        │  │ (selection)  │  │              │  │ Executor   │  │ (JSONL)    │
└───┬────┘  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘  └─────┬──────┘
    │              │                  │                 │               │
    │ tool_calls   │                  │                 │               │
    │─────────────▶│                  │                 │               │
    │              │                  │                 │               │
    │              │ buildToolRunCtx  │                 │               │
    │              │─────────────────▶│                 │               │
    │              │                  │                 │               │
    │              │ before_tool_call │                 │               │
    │              │ (loop detection) │                 │               │
    │              │─────────────────▶│                 │               │
    │              │                  │                 │               │
    │              │ 参数归一化+校验   │                 │               │
    │              │─────────────────▶│                 │               │
    │              │                  │                 │               │
    │              │ dispatchExecute  │                 │               │
    │              │───────────────────────────────────▶│               │
    │              │                  │                 │               │
    │              │                  │                 │ exec/read/    │
    │              │                  │                 │ message/...   │
    │              │                  │                 │               │
    │              │                  │◀────────────────│               │
    │              │                  │   原始结果       │               │
    │              │                  │                 │               │
    │              │ sanitizeResult   │                 │               │
    │              │ truncation       │                 │               │
    │              │ media extract    │                 │               │
    │              │◀─────────────────│                 │               │
    │              │                  │                 │               │
    │              │ after_tool_call  │                 │               │
    │              │─────────────────▶│                 │               │
    │              │                  │                 │               │
    │              │ write tool_result│                 │               │
    │              │────────────────────────────────────────────────────▶│
    │              │                  │                 │               │
    │              │ next LLM call    │                 │               │
    │◀─────────────│                  │                 │               │
    │  (含 tool_result)               │                 │               │
```

---

## 11. 关键设计模式

### 11.1 Tool Call ID 追踪

每个 tool call 都有唯一的 `toolCallId`，贯穿整个生命周期：

```typescript
function buildToolItemId(toolCallId) {
  return `tool:${toolCallId}`;
}
function buildCommandItemId(toolCallId) {
  return `command:${toolCallId}`;
}
function buildPatchItemId(toolCallId) {
  return `patch:${toolCallId}`;
}
```

根据工具类型生成不同前缀的 item ID，用于 transcript 中的追踪。

### 11.2 工具结果媒体处理

```typescript
// 工具结果中的媒体 URL 提取流程：
// 1. extractToolResultMediaArtifact() - 提取媒体
// 2. filterToolResultMediaUrls() - 按工具名过滤
// 3. sanitizeToolResultImages() - 图片大小/格式限制
// 4. 转为 base64 或 URL payload 传给模型
```

### 11.3 消息工具的异步投递

```typescript
// src/agents/tools/message-tool.ts
// 消息工具支持异步投递：
// - 立即返回 "消息已发送"
// - 实际发送在后台完成
// - 支持 target/thread/reply 等参数
```

### 11.4 工具结果上下文保护

```typescript
// src/agents/pi-embedded-runner/tool-result-context-guard.ts
// 安装 tool result context guard：
// - 防止 session 管理工具泄露其他 session 内容
// - 确保 agent-to-agent 通信的隐私隔离
```

---

## 12. 源码模块索引

| 模块 | 功能 | GitHub 链接 |
|---|---|---|
| `selection.ts` | Agent Loop 主循环，工具调用入口 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/selection.ts) |
| `attempt.tool-run-context.ts` | 工具执行上下文构建 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/attempt.tool-run-context.ts) |
| `attempt.tool-call-normalization.ts` | 工具参数归一化 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts) |
| `pi-tools.before-tool-call.runtime.ts` | before_tool_call 钩子 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tools.before-tool-call.runtime.ts) |
| `tool-loop-detection.ts` | 工具循环检测 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-loop-detection.ts) |
| `tool-policy-pipeline.ts` | 工具策略管道 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-pipeline.ts) |
| `tool-policy-match.ts` | 工具策略匹配 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-match.ts) |
| `openclaw-tools.registration.ts` | 内置工具注册 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-tools.registration.ts) |
| `bash-tools.exec.ts` | exec 工具实现 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec.ts) |
| `bash-tools.process.ts` | process 工具实现 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.process.ts) |
| `channel-tools.ts` | Channel 插件工具发现 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/channel-tools.ts) |
| `openclaw-plugin-tools.ts` | 插件工具注册 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-plugin-tools.ts) |
| `tool-result-truncation.ts` | 工具结果截断 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/tool-result-truncation.ts) |
| `tool-media-payloads.ts` | 工具结果媒体提取 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/run/tool-media-payloads.ts) |
| `pi-tool-definition-adapter.ts` | 工具定义转换 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tool-definition-adapter.ts) |
| `session-tool-result-guard.ts` | 工具结果隐私保护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/session-tool-result-guard.ts) |

---

## 13. 参考资料

| 资源 | 链接 |
|---|---|
| OpenClaw 文档 | https://docs.openclaw.ai |
| GitHub 仓库 | https://github.com/openclaw/openclaw |
| DeepWiki（AI 源码导读） | https://deepwiki.com/openclaw/openclaw |
| Agent Loop 概念 | https://docs.openclaw.ai/concepts/agent-loop.md |
| Tool 配置指南 | https://docs.openclaw.ai/config/tools.md |

---

_本文基于 OpenClaw v2026.4.21 源码分析，所有代码引用均来自官方 GitHub 仓库。_
