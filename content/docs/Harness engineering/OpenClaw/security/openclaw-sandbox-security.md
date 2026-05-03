# OpenClaw 安全执行环境：沙箱、审批、策略管道与深度防御体系

> **基于 OpenClaw v2026.4.21 源码深度分析**
> 作者：小伟 | 2026-04-30

---

## 导读

LLM Agent 最危险的操作是什么？**执行不受控的命令**。OpenClaw 面对这个问题的答案不是"信任模型"，而是构建了一套多层纵深防御体系（Defense in Depth）：从 Docker 沙箱隔离 → exec 安全策略 → 命令审批 → 文件系统保护 → SSRF 防护 → 工具策略管道，每一层都独立有效，合在一起构成完整的安全执行环境。

本文将从源码级别完整拆解这套体系。

---

## 1. 总览：纵深防御架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OpenClaw 安全执行环境                                 │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  第1层：沙箱隔离（Sandbox Isolation）                               │ │
│  │  Docker Container: readOnlyRoot, capDrop ALL, network none        │ │
│  │  SSH Backend: 严格 Host Key 校验 + 独立 workspaceRoot             │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────────┐ │
│  │  第2层：Exec 安全策略（Exec Safety）                                │ │
│  │  Shell 元字符检测 + 安全可执行值判断                               │ │
│  │  Safe Bin Profile（仅允许白名单命令）                               │ │
│  └─────────────────────────────┬─────────────────────────────────────┘ │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────────┐ │
│  │  第3层：命令审批（Exec Approval）                                   │ │
│  │  命令 → 用户确认 → 允许/拒绝/allowlist                             │ │
│  │  iOS Push / Channel 推送 / Web 界面                               │ │
│  └─────────────────────────────┬─────────────────────────────────────┘ │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────────┐ │
│  │  第4层：文件系统安全（Filesystem Safety）                           │ │
│  │  readFileWithinRoot / writeFileWithinRoot                         │ │
│  │  Path Alias Guards / Hardlink Guards                              │ │
│  │  Path Safety（穿越/符号链接/越界检测）                               │ │
│  └─────────────────────────────┬─────────────────────────────────────┘ │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────────┐ │
│  │  第5层：网络与 SSRF 防护（Network & SSRF）                          │ │
│  │  Sandbox network=none + SSRF Policy                               │ │
│  │  Fetch Guard（内网 IP 拦截 + 端口限制）                              │ │
│  └─────────────────────────────┬─────────────────────────────────────┘ │
│                                │                                       │
│  ┌─────────────────────────────▼─────────────────────────────────────┐ │
│  │  第6层：工具策略管道（Tool Policy Pipeline）                        │ │
│  │  Allowlist / Denylist / Owner-Only / Profile / FS Policy          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 第1层：沙箱隔离（Sandbox Isolation）

### 2.1 沙箱配置解析

**源码**：[sandbox/config.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/config.ts)

```typescript
// src/agents/sandbox/config.ts

// 沙箱模式
const SANDBOX_MODES = ['off', 'local', 'remote'];

// 后端类型
const SANDBOX_BACKENDS = ['docker', 'ssh'];

// 默认 Docker 配置
function resolveSandboxDockerConfig(params) {
  return {
    image: 'openclaw-sandbox:bookworm-slim',    // 默认沙箱镜像
    containerPrefix: 'openclaw-sbx-',           // 容器名前缀
    workdir: '/workspace',                       // 工作目录
    readOnlyRoot: true,                          // 只读根文件系统
    tmpfs: ['/tmp', '/var/tmp', '/run'],         // 临时文件系统
    network: 'none',                             // 默认无网络
    capDrop: ['ALL'],                            // 丢弃所有 Linux 能力
    // 可选的 seccomp / apparmor profile
    seccompProfile: ...,
    apparmorProfile: ...,
    // 资源限制
    pidsLimit: ...,
    memory: ...,
    memorySwap: ...,
    cpus: ...,
  };
}
```

**关键安全参数解读**：

| 参数 | 默认值 | 安全意义 |
|---|---|---|
| `readOnlyRoot` | `true` | 根文件系统只读，防止持久化恶意文件 |
| `network` | `none` | 无网络，无法发起出站连接（防数据外泄） |
| `capDrop: ['ALL']` | 丢弃全部 | 移除所有 Linux capabilities（防提权） |
| `tmpfs` | `/tmp,/var/tmp,/run` | 内存文件系统，重启即清空 |

### 2.2 三种沙箱作用域

**源码**：[sandbox/config.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/config.ts)

```typescript
// src/agents/sandbox/config.ts
function resolveSandboxScope(params) {
  if (params.scope) return params.scope;
  if (typeof params.perSession === "boolean")
    return params.perSession ? "session" : "shared";
  return "agent";  // 默认按 agent 隔离
}
```

| 作用域 | 说明 | 适用场景 |
|---|---|---|
| `session` | 每个会话独立沙箱 | 最高隔离级别，多用户环境 |
| `agent` | 每个 Agent 共享沙箱 | 默认级别，平衡安全与性能 |
| `shared` | 所有 Agent 共享沙箱 | 单用户开发环境 |

### 2.3 Docker 沙箱后端

**源码**：[sandbox/docker-backend.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/docker-backend.ts)

```typescript
// src/agents/sandbox/docker-backend.ts

async function createDockerSandboxBackend(params) {
  return createDockerSandboxBackendHandle({
    containerName: await ensureSandboxContainer({
      sessionKey, workspaceDir, agentWorkspaceDir, cfg
    }),
    workdir: params.cfg.docker.workdir,
    env: params.cfg.docker.env,
    image: params.cfg.docker.image
  });
}

function createDockerSandboxBackendHandle(params) {
  return {
    id: "docker",
    runtimeId: params.containerName,
    capabilities: { browser: true },

    // 构建 exec 规格：通过 docker exec 在容器内执行
    async buildExecSpec({ command, workdir, env, usePty }) {
      return {
        argv: ["docker", ...buildDockerExecArgs({
          containerName: params.containerName,
          command,
          workdir: workdir ?? params.workdir,
          env,
          tty: usePty
        })],
        env: process.env,
        stdinMode: usePty ? "pipe-open" : "pipe-closed"
      };
    }
  };
}
```

**核心思路**：所有命令都通过 `docker exec` 在隔离容器内执行，而不是直接在宿主机上执行。

### 2.4 SSH 沙箱后端

**源码**：[sandbox/config.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/config.ts)

```typescript
// src/agents/sandbox/config.ts
function resolveSandboxSshConfig(params) {
  return {
    target: ...,
    command: 'ssh',                        // SSH 命令
    workspaceRoot: '/tmp/openclaw-sandboxes',  // 远程工作目录
    strictHostKeyChecking: true,           // 严格 Host Key 校验
    updateHostKeys: true,                  // 自动更新 Host Keys
    identityFile: ...,                     // SSH 私钥
    certificateFile: ...,                  // SSH 证书
    knownHostsFile: ...,                   // 已知主机文件
    identityData: ...,                     // 私钥数据（可加密存储）
    certificateData: ...,                  // 证书数据
    knownHostsData: ...                    // 已知主机数据
  };
}
```

**安全要点**：SSH 后端支持完整的证书/密钥管理，`strictHostKeyChecking: true` 防止中间人攻击。

### 2.5 沙箱浏览器安全

**源码**：[sandbox/config.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/config.ts)

```typescript
// src/agents/sandbox/config.ts
function resolveSandboxBrowserConfig(params) {
  return {
    enabled: false,                          // 默认禁用
    image: 'openclaw-sandbox-browser:bookworm-slim',
    network: 'openclaw-sandbox-browser',      // 独立网络
    cdpPort: 9222,                            // Chrome DevTools Protocol
    vncPort: 5900,                            // VNC 端口
    noVncPort: 6080,                          // noVNC Web 端口
    headless: false,
    enableNoVnc: true,                        // 启用 noVNC
    allowHostControl: false,                  // 默认不允许宿主机控制
    autoStart: true,
    autoStartTimeoutMs: 12000                 // 启动超时 12s
  };
}
```

**浏览器沙箱安全**：
- 浏览器运行在独立网络命名空间（`openclaw-sandbox-browser`）
- noVNC 提供 Web 访问，但不直接暴露 CDP 端口
- `allowHostControl: false` 默认禁止浏览器控制宿主机

### 2.6 沙箱文件系统桥接

**源码**：[sandbox/fs-bridge.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/fs-bridge.ts) / [sandbox/fs-bridge-path-safety.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/fs-bridge-path-safety.ts)

沙箱容器内的文件操作需要通过 **FS Bridge** 安全地映射到宿主机文件系统。桥接过程中进行路径安全校验，防止路径穿越攻击。

### 2.7 沙箱验证

**源码**：[sandbox/validate-sandbox-security.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/validate-sandbox-security.ts)

在沙箱创建后运行安全验证，确保隔离生效。

### 2.8 沙箱修剪（Prune）

**源码**：[sandbox/prune.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/prune.ts)

```typescript
// src/agents/sandbox/prune.ts
function resolveSandboxPruneConfig(params) {
  return {
    idleHours: 24,    // 空闲 24 小时后自动清理
    maxAgeDays: 7     // 最大存活 7 天
  };
}
```

防止僵尸沙箱容器持续消耗资源。

---

## 3. 第2层：Exec 安全策略

### 3.1 安全可执行值判断

**源码**：[infra/exec-safety.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-safety.ts)

```typescript
// src/infra/exec-safety.ts

// Shell 元字符：可能触发命令注入
const SHELL_METACHARS = /[;&|`$<>]/;

// 控制字符：可能导致命令拼接
const CONTROL_CHARS = /[\r\n]/;

// 引号字符：可能逃逸参数
const QUOTE_CHARS = /["']/;

// 安全名称模式
const BARE_NAME_PATTERN = /^[A-Za-z0-9._+-]+$/;

function isLikelyPath(value) {
  if (value.startsWith(".") || value.startsWith("~")) return true;
  if (value.includes("/") || value.includes("\\")) return true;
  return /^[A-Za-z]:[\\/]/.test(value);
}

function isSafeExecutableValue(value) {
  if (!value) return false;
  const trimmed = value.trim();
  if (!trimmed) return false;
  if (trimmed.includes("\0")) return false;           // 拒绝 null 字符
  if (CONTROL_CHARS.test(trimmed)) return false;      // 拒绝换行
  if (SHELL_METACHARS.test(trimmed)) return false;    // 拒绝 shell 元字符
  if (QUOTE_CHARS.test(trimmed)) return false;        // 拒绝引号
  if (isLikelyPath(trimmed)) return true;             // 路径认为是安全的
  if (trimmed.startsWith("-")) return false;          // 拒绝 flag 参数
  return BARE_NAME_PATTERN.test(trimmed);
}
```

**防护目标**：防止通过参数注入恶意 shell 命令。

### 3.2 Safe Bin Profile

**源码**：[infra/exec-safe-bin-runtime-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-safe-bin-runtime-policy.ts) / [infra/exec-safe-bin-policy-profiles.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-safe-bin-policy-profiles.ts)

Safe Bin Profile 是 OpenClaw 的命令白名单机制：

```typescript
// src/infra/exec-safe-bin-runtime-policy.ts
// 定义哪些命令可以在"安全模式"下直接执行而无需审批
// 常见安全命令：ls, cat, grep, find, head, tail, wc 等

// src/infra/exec-safe-bin-policy-profiles.ts
// 定义不同的安全策略 profile（strict / moderate / permissive）
```

### 3.3 Exec Host 选择

**源码**：[infra/exec-host.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-host.ts)

命令可以在多个宿主上执行：

| 执行宿主 | 说明 | 安全性 |
|---|---|---|
| `gateway` | 在 Gateway 进程内执行 | 最低（宿主机） |
| `node` | 在 Node 节点上执行 | 中（可远程隔离） |
| `sandbox` | 在 Docker/SSH 沙箱内执行 | 最高 |

**源码**：[bash-tools.exec-host-gateway.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec-host-gateway.ts) / [bash-tools.exec-host-node.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec-host-node.ts)

---

## 4. 第3层：命令审批（Exec Approval）

### 4.1 审批流程

当模型尝试执行不在 allowlist 中的命令时，触发审批流程：

```
模型请求: exec("rm -rf /tmp/data")
         │
         ▼
  ┌──────────────────┐
  │ 命令是否在       │──Yes──▶ 直接执行
  │ allowlist 中？   │
  └────────┬─────────┘
           │ No
           ▼
  ┌──────────────────┐
  │ 生成审批请求      │
  │ exec-approval    │
  │ -request.ts      │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 推送审批通知      │
  │ 到用户终端        │
  │ (Channel/Push)   │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │ 用户决策          │
  │ ✅ 允许 / ❌ 拒绝  │
  │ 🔁 加入 allowlist │
  └────────┬─────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
   执行        拒绝并反馈
```

### 4.2 审批请求构建

**源码**：[bash-tools.exec-approval-request.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec-approval-request.ts)

```typescript
// src/agents/bash-tools.exec-approval-request.ts
// 构建命令审批请求：
// - 命令原文
// - 安全分析（shell 元字符、可疑模式）
// - 命令显示优化（高亮关键部分）
```

### 4.3 审批转发

**源码**：[infra/exec-approval-forwarder.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approval-forwarder.ts)

```typescript
// src/infra/exec-approval-forwarder.ts
// 将审批请求转发到用户的各种渠道：
// - 钉钉消息
// - 钉钉 iOS Push 通知
// - Web 界面
// - 其他 Channel
```

### 4.4 审批管理

**源码**：[gateway/exec-approval-manager.ts](https://github.com/openclaw/openclaw/blob/main/src/gateway/exec-approval-manager.ts)

管理所有待审批的命令请求，支持超时自动拒绝。

### 4.5 Allowlist 管理

**源码**：[infra/exec-approvals-allowlist.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approvals-allowlist.ts)

```typescript
// src/infra/exec-approvals-allowlist.ts
// 用户可以将常用命令加入 allowlist，避免重复审批
// 支持通配符模式
```

### 4.6 审批分析

**源码**：[infra/exec-approvals-analysis.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approvals-analysis.ts)

对命令进行安全分析，识别可疑模式：

```typescript
// src/infra/exec-approvals-analysis.ts
// 检测：
// - 删除命令（rm -rf）
// - 网络命令（curl, wget）
// - 系统修改（chmod, chown）
// - 敏感路径访问（/etc/shadow）
// - 命令注入模式
```

---

## 5. 第4层：文件系统安全

### 5.1 安全文件操作

**源码**：[infra/fs-safe.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/fs-safe.ts)

```typescript
// src/infra/fs-safe.ts

// 所有文件操作都必须在工作区根目录内
async function readFileWithinRoot(rootDir, filePath) { ... }
async function writeFileWithinRoot(rootDir, filePath, content) { ... }
async function appendFileWithinRoot(rootDir, filePath, content) { ... }
async function openFileWithinRoot(rootDir, filePath, flags) { ... }

// 路径校验：拒绝 ../ 穿越、符号链接逃逸
class SafeOpenError extends Error { ... }
```

**核心原则**：Agent 只能读写工作区（workspace）内的文件，无法访问系统文件。

### 5.2 路径安全校验

**源码**：[infra/path-safety.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/path-safety.ts)

```typescript
// src/infra/path-safety.ts
// 路径安全检测：
// - 拒绝 ../ 路径穿越
// - 拒绝绝对路径逃逸
// - 符号链接解析后必须在根目录内
```

### 5.3 路径别名保护

**源码**：[infra/path-alias-guards.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/path-alias-guards.ts)

```typescript
// src/infra/path-alias-guards.ts

// PATH_ALIAS_POLICIES - 路径别名策略
// 防止通过 @alias 等别名机制绕过路径限制
```

### 5.4 硬链接保护

**源码**：[infra/hardlink-guards.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/hardlink-guards.ts)

防止通过硬链接绕过文件访问限制。

### 5.5 工作区守卫

**源码**：[openclaw-tools.nodes-workspace-guard.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/openclaw-tools.nodes-workspace-guard.ts)

Node 节点上的工具执行同样受工作区限制。

### 5.6 安装安全路径

**源码**：[infra/install-safe-path.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/install-safe-path.ts)

插件/技能安装时的路径安全校验，防止安装到系统目录。

---

## 6. 第5层：网络与 SSRF 防护

### 6.1 SSRF 防护

**源码**：[infra/net/ssrf.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/net/ssrf.ts)

```typescript
// src/infra/net/ssrf.ts
// SSRF（Server-Side Request Forgery）防护：
// - 拦截内网 IP 地址（10.x, 172.16-31.x, 192.168.x, 127.x）
// - 拦截 DNS 重绑定攻击
// - 拦截 IPv6 本地地址
```

### 6.2 Fetch Guard

**源码**：[infra/net/fetch-guard.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/net/fetch-guard.ts)

```typescript
// src/infra/net/fetch-guard.ts
// web_fetch 工具的安全网关：
// - SSRF 检查
// - URL 方案检查（仅 http/https）
// - 重定向限制
// - 响应大小限制
```

### 6.3 SSRF 策略

**源码**：[plugin-sdk/ssrf-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/plugin-sdk/ssrf-policy.ts)

插件 SDK 层面的 SSRF 策略定义。

### 6.4 Web Guarded Fetch

**源码**：[tools/web-guarded-fetch.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tools/web-guarded-fetch.ts)

```typescript
// src/agents/tools/web-guarded-fetch.ts
// web_fetch 工具的安全封装：
// - 自动应用 SSRF 防护
// - 自动限制响应大小
// - 自动处理重定向
```

---

## 7. 第6层：工具策略管道（Tool Policy Pipeline）

### 7.1 策略管道

**源码**：[tool-policy-pipeline.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-pipeline.ts)

```typescript
// src/agents/tool-policy-pipeline.ts

// 默认管道步骤：
const DEFAULT_PIPELINE_STEPS = [
  'tool-name-allowlist',     // 工具名称白名单
  'owner-only-policy',        // 仅 owner 可执行
  'tool-profile-policy',      // 工具 profile
  'tool-fs-policy',           // 文件系统限制
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

### 7.2 策略匹配

**源码**：[tool-policy-match.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-match.ts)

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

### 7.3 沙箱工具策略

**源码**：[sandbox-tool-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox-tool-policy.ts) / [sandbox/tool-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/tool-policy.ts)

```typescript
// src/agents/sandbox-tool-policy.ts
// 沙箱环境下的工具策略：
// - 沙箱中允许的工具列表
// - 沙箱中禁止的操作
// - exec 在沙箱中的特殊处理
```

### 7.4 有效工具策略

**源码**：[pi-embedded-runner/effective-tool-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/effective-tool-policy.ts)

```typescript
// src/agents/pi-embedded-runner/effective-tool-policy.ts
// 计算最终生效的工具策略：
// - 合并多层策略配置
// - 处理策略冲突
// - 生成最终 allow/deny 列表
```

### 7.5 消息 Provider 策略

**源码**：[pi-tools.message-provider-policy.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-tools.message-provider-policy.ts)

消息类工具的策略控制，防止向未授权渠道发送消息。

---

## 8. 第7层：运行时安全

### 8.1 运行时守卫

**源码**：[infra/runtime-guard.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/runtime-guard.ts)

```typescript
// src/infra/runtime-guard.ts
// 运行时安全检查：
// - 进程权限检查
// - 文件权限检查
// - 环境变量安全检查
```

### 8.2 主机环境安全

**源码**：[infra/host-env-security.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/host-env-security.ts)

```typescript
// src/infra/host-env-security.ts
// 主机环境变量安全策略：
// - host-env-security-policy.json - 定义敏感环境变量
// - 防止泄露 API keys、数据库密码等
```

### 8.3 环境变量清理

**源码**：[sandbox/sanitize-env-vars.ts](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/sanitize-env-vars.ts)

```typescript
// src/agents/sandbox/sanitize-env-vars.ts
// 传入沙箱的环境变量需要过滤：
// - 移除敏感变量（AWS_SECRET_KEY, DATABASE_URL 等）
// - 仅传递必要的 LANG、PATH 等基础变量
```

### 8.4 安全打开文件

**源码**：[infra/safe-open-sync.ts](https://github.com/openclaw/openclaw/blob/main/src/infra/safe-open-sync.ts)

同步文件打开操作的安全封装。

---

## 9. 第8层：安全审计

### 9.1 安全审计系统

**源码**：[security/audit.ts](https://github.com/openclaw/openclaw/blob/main/src/security/audit.ts)

OpenClaw 内置安全审计系统，支持多维度的安全检查：

```typescript
// src/security/audit.ts
// 审计维度：
// - Gateway 配置安全
// - 文件系统安全
// - 插件信任度
// - 技能扫描
// - 深度代码安全
```

### 9.2 各审计模块

| 模块 | 功能 | GitHub 链接 |
|---|---|---|
| `audit-gateway-config` | Gateway 配置审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-gateway-config.ts) |
| `audit-fs` | 文件系统审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-fs.ts) |
| `audit-plugins-trust` | 插件信任审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts) |
| `audit-deep-code-safety` | 深度代码安全 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-deep-code-safety.ts) |
| `skill-scanner` | 技能扫描 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/skill-scanner.ts) |
| `dangerous-config-flags` | 危险配置标志 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/dangerous-config-flags.ts) |
| `dangerous-tools` | 危险工具检测 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/dangerous-tools.ts) |

### 9.3 Doctor 安全命令

**源码**：[commands/doctor-security.ts](https://github.com/openclaw/openclaw/blob/main/src/commands/doctor-security.ts) / [commands/doctor-sandbox.ts](https://github.com/openclaw/openclaw/blob/main/src/commands/doctor-sandbox.ts)

```bash
# 检查安全配置
openclaw doctor security

# 检查沙箱状态
openclaw doctor sandbox
```

---

## 10. 安全执行完整序列图

```
用户/模型请求: exec("curl https://api.example.com/data")
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 工具策略管道（Tool Policy Pipeline）                      │
│  - isToolAllowedByPolicies()                                     │
│  - 检查 allowlist/denylist                                      │
│  - 检查 owner-only                                              │
│  - 检查 tool profile                                             │
│  └─▶ 通过                                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: before_tool_call 钩子                                    │
│  - detectToolCallLoop() 检测循环调用                             │
│  - recordToolCall() 记录调用                                    │
│  └─▶ 通过                                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Exec Host 选择                                           │
│  - 如果有沙箱 → 选择 sandbox 后端                                │
│  - 否则 → gateway 或 node                                       │
│  └─▶ sandbox (docker)                                           │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: 命令安全检查                                              │
│  - isSafeExecutableValue() 检查 shell 元字符                     │
│  - exec-safe-bin 检查命令是否在白名单                            │
│  └─▶ 不在白名单 → 触发审批                                        │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Exec Approval（命令审批）                                 │
│  - exec-approval-request.ts 构建审批请求                          │
│  - exec-approval-forwarder.ts 推送通知到用户                      │
│  - exec-approvals-analysis.ts 分析命令风险                        │
│  - exec-approvals-allowlist.ts 检查 allowlist                     │
│  └─▶ 用户批准 ✅                                                  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: Docker 沙箱执行                                           │
│  - docker exec openclaw-sbx-<id> sh -c "curl ..."               │
│  - 容器: readOnlyRoot=true, network=none, capDrop=ALL           │
│  - buildExecSpec() 构建 docker exec 命令                          │
│  - buildDockerExecArgs() 构建参数                                │
│  └─▶ 命令执行完成                                                 │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 7: SSRF 检查（如果涉及网络访问）                               │
│  - ssrf.ts 检查目标 IP 是否内网                                   │
│  - fetch-guard.ts 检查 URL 合法性                                 │
│  └─▶ （本例中网络被 sandbox network=none 阻断）                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 8: 结果处理                                                  │
│  - sanitizeToolResult() 清洗结果                                 │
│  - 过滤敏感信息                                                   │
│  - 截断过长结果                                                   │
│  └─▶ 返回结果给模型                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 11. 关键安全设计模式

### 11.1 默认安全（Secure by Default）

所有安全选项默认都是最严格的：

```typescript
// 沙箱默认配置
readOnlyRoot: true          // 根文件系统只读
network: 'none'             // 无网络
capDrop: ['ALL']            // 丢弃所有能力
strictHostKeyChecking: true // 严格 Host Key 校验
allowHostControl: false     // 不允许宿主机控制
enabled: false              // 浏览器默认禁用
```

### 11.2 纵深防御（Defense in Depth）

每一层安全机制都独立有效：

```
即使绕过了沙箱（几乎不可能），还有：
├── 文件系统保护（readFileWithinRoot）
├── 工具策略管道（denylist）
├── SSRF 防护
└── 安全审计
```

### 11.3 最小权限原则（Least Privilege）

```typescript
// 沙箱容器只挂载必要目录
binds: [
  `${workspaceDir}:/workspace`,        // 工作区
  `${agentWorkspaceDir}:/agent`        // Agent 工作区
]
// 不挂载系统目录、不共享宿主机网络
```

### 11.4 失败安全（Fail-Safe）

```typescript
// 任何安全检测失败时，默认拒绝
if (result.deny) return { allowed: false, reason: result.deny };
// 而不是 allow on error
```

### 11.5 路径别名保护

```typescript
// PATH_ALIAS_POLICIES - 防止通过别名绕过路径限制
// 例如 @root 只能解析到 workspace 根目录
```

### 11.6 环境变量隔离

```typescript
// 沙箱内只接收清理后的环境变量
// 移除 AWS_SECRET_KEY, DATABASE_URL 等敏感变量
```

---

## 12. 配置示例

### 12.1 开启沙箱

```json5
// openclaw.json
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "local",              // local | remote
        "backend": "docker",          // docker | ssh
        "scope": "session",           // session | agent | shared
        "workspaceAccess": "readwrite",
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "readOnlyRoot": true,
          "network": "none",
          "capDrop": ["ALL"],
          "memory": "1g",
          "cpus": 2
        }
      }
    }
  }
}
```

### 12.2 配置 SSH 沙箱

```json5
{
  "sandbox": {
    "mode": "remote",
    "backend": "ssh",
    "ssh": {
      "target": "sandbox-host.example.com",
      "workspaceRoot": "/tmp/openclaw-sandboxes",
      "strictHostKeyChecking": true,
      "identityFile": "~/.ssh/openclaw-sandbox"
    }
  }
}
```

### 12.3 配置 Exec Approval

```json5
{
  "tools": {
    "exec": {
      "approval": {
        "mode": "prompt",            // prompt | auto-allow | block
        "allowlist": ["ls", "cat", "grep"],
        "notifyOnExit": true
      }
    }
  }
}
```

---

## 13. 源码模块索引

| 模块 | 功能 | GitHub 链接 |
|---|---|---|
| **沙箱核心** | | |
| `sandbox/config.ts` | 沙箱配置解析 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/config.ts) |
| `sandbox/docker-backend.ts` | Docker 沙箱后端 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/docker-backend.ts) |
| `sandbox/ssh-backend.ts` | SSH 沙箱后端 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/ssh-backend.ts) |
| `sandbox/browser.ts` | 沙箱浏览器 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/browser.ts) |
| `sandbox/registry.ts` | 沙箱注册表 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/registry.ts) |
| `sandbox/prune.ts` | 沙箱修剪 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/prune.ts) |
| `sandbox/fs-bridge.ts` | 文件系统桥接 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/fs-bridge.ts) |
| `sandbox/fs-bridge-path-safety.ts` | 桥接路径安全 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/fs-bridge-path-safety.ts) |
| `sandbox/workspace-mounts.ts` | 工作区挂载 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/workspace-mounts.ts) |
| `sandbox/sanitize-env-vars.ts` | 环境变量清理 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/sanitize-env-vars.ts) |
| `sandbox/validate-sandbox-security.ts` | 沙箱安全验证 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/validate-sandbox-security.ts) |
| `sandbox/network-mode.ts` | 网络模式 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox/network-mode.ts) |
| **Exec 安全** | | |
| `infra/exec-safety.ts` | Exec 安全检查 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-safety.ts) |
| `infra/exec-safe-bin-runtime-policy.ts` | Safe Bin 策略 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-safe-bin-runtime-policy.ts) |
| `infra/exec-host.ts` | Exec Host 选择 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-host.ts) |
| **命令审批** | | |
| `bash-tools.exec-approval-request.ts` | 审批请求构建 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec-approval-request.ts) |
| `bash-tools.exec-approval-followup.ts` | 审批跟进 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/bash-tools.exec-approval-followup.ts) |
| `infra/exec-approvals.ts` | 审批核心逻辑 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approvals.ts) |
| `infra/exec-approvals-allowlist.ts` | 审批白名单 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approvals-allowlist.ts) |
| `infra/exec-approvals-analysis.ts` | 审批安全分析 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approvals-analysis.ts) |
| `infra/exec-approval-forwarder.ts` | 审批转发 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/exec-approval-forwarder.ts) |
| `gateway/exec-approval-manager.ts` | 审批管理器 | [查看](https://github.com/openclaw/openclaw/blob/main/src/gateway/exec-approval-manager.ts) |
| `gateway/exec-approval-ios-push.ts` | iOS Push 通知 | [查看](https://github.com/openclaw/openclaw/blob/main/src/gateway/exec-approval-ios-push.ts) |
| **文件系统安全** | | |
| `infra/fs-safe.ts` | 安全文件操作 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/fs-safe.ts) |
| `infra/path-safety.ts` | 路径安全 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/path-safety.ts) |
| `infra/path-alias-guards.ts` | 路径别名保护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/path-alias-guards.ts) |
| `infra/hardlink-guards.ts` | 硬链接保护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/hardlink-guards.ts) |
| `infra/install-safe-path.ts` | 安装路径安全 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/install-safe-path.ts) |
| **网络/SSRF** | | |
| `infra/net/ssrf.ts` | SSRF 防护 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/net/ssrf.ts) |
| `infra/net/fetch-guard.ts` | Fetch 网关 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/net/fetch-guard.ts) |
| `plugin-sdk/ssrf-policy.ts` | SSRF 策略 | [查看](https://github.com/openclaw/openclaw/blob/main/src/plugin-sdk/ssrf-policy.ts) |
| **工具策略** | | |
| `tool-policy-pipeline.ts` | 策略管道 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-pipeline.ts) |
| `tool-policy-match.ts` | 策略匹配 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/tool-policy-match.ts) |
| `sandbox-tool-policy.ts` | 沙箱工具策略 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/sandbox-tool-policy.ts) |
| `pi-embedded-runner/effective-tool-policy.ts` | 有效工具策略 | [查看](https://github.com/openclaw/openclaw/blob/main/src/agents/pi-embedded-runner/effective-tool-policy.ts) |
| **运行时安全** | | |
| `infra/runtime-guard.ts` | 运行时守卫 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/runtime-guard.ts) |
| `infra/host-env-security.ts` | 主机环境安全 | [查看](https://github.com/openclaw/openclaw/blob/main/src/infra/host-env-security.ts) |
| **安全审计** | | |
| `security/audit.ts` | 安全审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit.ts) |
| `security/audit-fs.ts` | 文件系统审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-fs.ts) |
| `security/audit-gateway-config.ts` | 配置审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-gateway-config.ts) |
| `security/audit-plugins-trust.ts` | 插件审计 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/audit-plugins-trust.ts) |
| `security/skill-scanner.ts` | 技能扫描 | [查看](https://github.com/openclaw/openclaw/blob/main/src/security/skill-scanner.ts) |

---

## 14. 参考资料

| 资源 | 链接 |
|---|---|
| OpenClaw 文档 | https://docs.openclaw.ai |
| GitHub 仓库 | https://github.com/openclaw/openclaw |
| DeepWiki（AI 源码导读） | https://deepwiki.com/openclaw/openclaw |
| Agent Runtimes | https://docs.openclaw.ai/concepts/agent-runtimes.md |
| Sandbox 配置 | https://docs.openclaw.ai/config/sandbox.md |
| Exec Approval | https://docs.openclaw.ai/gateway/exec-approval.md |
| Security CLI | https://docs.openclaw.ai/cli/security.md |

---

_本文基于 OpenClaw v2026.4.21 源码分析，所有代码引用均来自官方 GitHub 仓库。_
