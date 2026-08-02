# Hermes Agent 远程接入指南：API Server 与 Webhook

> 本文档详细介绍如何通过 **API Server** 和 **Webhook** 两种方式，让远程应用程序连接到 Hermes Agent。

---

## 一、API Server — OpenAI 兼容的 HTTP API

### 1.1 概述

API Server 是 Hermes 提供的 **OpenAI 兼容 HTTP 接口**。任何支持 OpenAI API 格式的客户端（Open WebUI、LobeChat、LibreChat、自定义程序等）都可以直接对接。

本质上，它把 Hermes Agent（具备完整工具调用能力）包装成了一个 OpenAI-compatible endpoint。

### 1.2 启动方式

```bash
# 1. 配置 API Server
hermes gateway setup          # 选择 "API Server" 平台
# 会要求设置：
#   - 监听端口（默认 8642）
#   - API Server Key（用于鉴权）

# 2. 前台运行（测试用）
hermes gateway run

# 3. 安装为后台服务（生产用）
hermes gateway install
hermes gateway start

# 4. 检查状态
hermes gateway status
```

配置会写入 `~/.hermes/config.yaml` 的 `api_server` 段。

### 1.3 鉴权

所有 API 请求需要在 Header 中携带 API Key：

```
Authorization: Bearer <API_SERVER_KEY>
```

API Key 在配置时设定，保存在 `~/.hermes/.env` 或 `config.yaml` 中。

### 1.4 完整端点列表

| 方法 | 端点 | 说明 |
|------|------|------|
| `POST` | `/v1/chat/completions` | OpenAI 对话接口（支持 tool calling） |
| `POST` | `/v1/responses` | OpenAI Responses API 格式（有状态） |
| `GET` | `/v1/responses/{id}` | 获取已存储的响应 |
| `DELETE` | `/v1/responses/{id}` | 删除已存储的响应 |
| `GET` | `/v1/models` | 列出可用模型 |
| `GET` | `/v1/capabilities` | 机器可读的 API 能力描述 |
| `GET` | `/api/sessions` | 列出所有会话 |
| `POST` | `/api/sessions` | 创建空会话 |
| `GET/PATCH/DELETE` | `/api/sessions/{id}` | 读取/更新/删除会话 |
| `GET` | `/api/sessions/{id}/messages` | 读取会话消息历史 |
| `POST` | `/api/sessions/{id}/fork` | 分支会话 |
| `POST` | `/api/sessions/{id}/chat` | 向会话发送消息（普通） |
| `POST` | `/api/sessions/{id}/chat/stream` | 向会话发送消息（流式） |
| `POST` | `/v1/runs` | 启动一个异步运行（立即返回 run_id） |
| `GET` | `/v1/runs/{id}` | 查询运行状态 |
| `GET` | `/v1/runs/{id}/events` | SSE 流式获取结构化事件 |
| `POST` | `/v1/runs/{id}/approval` | 处理待审批的命令 |
| `POST` | `/v1/runs/{id}/stop` | 中断正在运行的 Agent |
| `GET` | `/health` | 健康检查 |
| `GET` | `/health/detailed` | 详细状态（用于仪表盘） |

### 1.5 使用示例

#### 示例 1：curl 调用聊天（OpenAI 兼容格式）

```bash
curl -X POST http://localhost:8642/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "帮我分析服务器上最近10条Nginx错误日志"}
    ],
    "stream": false
  }'
```

#### 示例 2：保持会话连续性

通过 `X-Hermes-Session-Id` 头保持同一会话上下文：

```bash
# 第一次请求
curl -X POST http://localhost:8642/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-Hermes-Session-Id: my-project-001" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "检查 /var/log 目录的磁盘使用情况"}
    ]
  }'

# 后续请求 — 同一会话，保持上下文
curl -X POST http://localhost:8642/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "X-Hermes-Session-Id: my-project-001" \
  -d '{
    "model": "hermes-agent",
    "messages": [
      {"role": "user", "content": "基于上面的结果，清理最占空间的目录"}
    ]
  }'
```

#### 示例 3：Python 客户端（使用 OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8642/v1",
    api_key="YOUR_API_KEY"
)

response = client.chat.completions.create(
    model="hermes-agent",
    messages=[{"role": "user", "content": "列出当前系统运行的所有进程"}],
)

print(response.choices[0].message.content)
```

#### 示例 4：异步 Run（适合长时间任务）

Hermes Agent 可能执行多步工具调用（几分钟），使用 Run API 避免 HTTP 超时：

```bash
# 启动运行 — 立即返回 run_id
curl -X POST http://localhost:8642/v1/runs \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "message": "分析项目代码库，生成架构报告保存到 ~/report.md"
  }'
# 返回: {"run_id": "abc123", "status": "running"}

# 轮询状态
curl http://localhost:8642/v1/runs/abc123 \
  -H "Authorization: Bearer YOUR_API_KEY"
# 返回: {"run_id": "abc123", "status": "completed", "response": "..."}

# 或通过 SSE 实时流式获取事件
curl http://localhost:8642/v1/runs/abc123/events \
  -H "Authorization: Bearer YOUR_API_KEY"
```

#### 示例 5：处理命令审批

当 Agent 需要执行危险命令（如 `rm -rf`）时，会进入审批状态：

```bash
# 查看待审批的命令
curl http://localhost:8642/v1/runs/abc123 \
  -H "Authorization: Bearer YOUR_API_KEY"
# 返回: {"status": "pending_approval", "approval_request": "..."}

# 批准
curl -X POST http://localhost:8642/v1/runs/abc123/approval \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"action": "approve"}'

# 拒绝
curl -X POST http://localhost:8642/v1/runs/abc123/approval \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"action": "deny"}'
```

### 1.6 关键特性

| 特性 | 说明 |
|------|------|
| **Tool Calling** | 完整支持，Agent 可以调用终端、文件、搜索等工具 |
| **流式输出** | 支持 SSE 流式响应（`stream: true`） |
| **会话持久化** | 通过 `X-Hermes-Session-Id` 保持上下文 |
| **多模态** | 支持图片输入（image_url / data URI） |
| **异步 Run** | 长时间任务不阻塞 HTTP 连接 |
| **命令审批** | 远程审批/拒绝 Agent 的危险操作 |
| **前端兼容** | 任何 OpenAI 兼容 UI 可直接对接 |

### 1.7 对接 Open WebUI（示例）

1. 启动 Hermes API Server
2. 在 Open WebUI 中添加连接：
   - URL: `http://YOUR_SERVER_IP:8642/v1`
   - API Key: `YOUR_API_KEY`
3. 选择模型 `hermes-agent`
4. 即可在 Web 界面中与 Hermes 交互

---

## 二、Webhook — 事件驱动的 HTTP 回调

### 2.1 概述

Webhook 是 **事件驱动** 的接入方式。外部系统通过 HTTP POST 触发 Hermes 执行，Hermes 处理后可将结果回传到指定平台（钉钉、Telegram、GitHub Issue 评论等），或直接返回 HTTP 响应。

**适用场景：**
- CI/CD 流水线触发代码审查
- 监控告警自动分析日志
- GitHub/GitLab 事件自动响应
- 定时任务/外部系统通知触发
- 多 Agent 之间互相通信

### 2.2 启动方式

```bash
# 1. 配置 Webhook
hermes gateway setup          # 选择 "Webhook" 平台
# 会要求设置：
#   - 监听端口（默认 8644）
#   - HMAC Secret（签名验证密钥）

# 2. 运行
hermes gateway run            # 前台
hermes gateway install        # 后台服务
hermes gateway start
```

### 2.3 配置格式

Webhook 路由配置在 `~/.hermes/config.yaml` 中：

```yaml
webhook:
  host: "0.0.0.0"        # 监听地址（0.0.0.0 = 公开，127.0.0.1 = 仅本地）
  port: 8644              # 监听端口
  secret: "YOUR_SECRET"   # 全局 HMAC 密钥（推荐）
  rate_limit: 30          # 每分钟最大请求数
  max_body_bytes: 1048576 # 请求体大小限制（1MB）
  routes:
    ci-review:            # 路由名称
      events: ["push", "pull_request"]  # 接受的事件类型（可选）
      secret: "CI_SECRET"               # 路由级 HMAC 密钥
      prompt: "收到CI事件: {{payload}}，请审查代码变更"  # 提示词模板
      skills: ["github-code-review"]    # 可选：预加载技能
      deliver: "dingtalk"               # 结果投递目标
      deliver_extra:                    # 投递额外参数
        chat_id: "YOUR_CHAT_ID"
    log-analysis:
      events: []
      secret: "LOG_SECRET"
      prompt: "分析以下日志并给出建议: {{payload}}"
      deliver: "log"                    # 仅记录日志，不投递
```

### 2.4 动态路由（运行时创建）

除了在 config.yaml 中静态配置，还可以通过命令动态创建路由：

```bash
# 创建一个 webhook 订阅
hermes webhook subscribe my-service
# 会生成 /webhooks/my-service 端点

# 查看所有订阅
hermes webhook list

# 测试
hermes webhook test my-service

# 删除
hermes webhook remove my-service
```

### 2.5 使用示例

#### 示例 1：curl 触发 Webhook

```bash
curl -X POST http://YOUR_SERVER:8644/webhooks/ci-review \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=<HMAC_SIGNATURE>" \
  -d '{
    "action": "opened",
    "pull_request": {
      "number": 42,
      "title": "Fix authentication bug",
      "head": {"sha": "abc123"}
    }
  }'
```

**签名计算（以 GitHub 为例）：**
```bash
# Python 示例
import hmac, hashlib, json

secret = b"CI_SECRET"
body = json.dumps(payload).encode()
signature = hmac.new(secret, body, hashlib.sha256).hexdigest()
# Header: X-Hub-Signature-256: sha256=<signature>
```

**测试模式（跳过签名验证）：**
```yaml
routes:
  test-route:
    secret: "INSECURE_NO_AUTH"    # 仅用于本地测试！
    prompt: "测试消息: {{payload}}"
    deliver: "log"
```
> ⚠️ `INSECURE_NO_AUTH` 仅在 `host: "127.0.0.1"` 时允许，绑定 `0.0.0.0` 会被拒绝启动。

#### 示例 2：GitHub Webhook 自动代码审查

```yaml
webhook:
  routes:
    github-pr:
      secret: "GITHUB_WEBHOOK_SECRET"
      prompt: |
        GitHub PR 事件收到：
        仓库: {{payload.repository.full_name}}
        PR #{{payload.pull_request.number}}: {{payload.pull_request.title}}
        请审查此 PR 的代码变更，重点关注安全性和性能问题。
      skills: ["github-code-review"]
      deliver: "github_comment"
      deliver_extra:
        repo: "{{payload.repository.full_name}}"
        pr_number: "{{payload.pull_request.number}}"
```

当 GitHub 推送 PR 事件到 `http://YOUR_SERVER:8644/webhooks/github-pr` 时，Hermes 会自动审查代码并将评论发布到 PR 上。

#### 示例 3：监控告警 → 自动分析 → 推送到钉钉

```yaml
webhook:
  routes:
    alert-handler:
      secret: "ALERT_SECRET"
      prompt: |
        收到监控告警，请分析原因并给出修复建议：
        {{payload}}
      deliver: "dingtalk"
      deliver_extra:
        chat_id: "YOUR_DINGTALK_CHAT_ID"
```

远程监控系统 POST 告警到 Webhook → Hermes 分析 → 结果推送到钉钉群。

#### 示例 4：纯通知模式（不经过 Agent，零 LLM 成本）

```yaml
webhook:
  routes:
    deploy-notify:
      secret: "DEPLOY_SECRET"
      prompt: "🚀 部署完成：{{payload.environment}} - {{payload.status}}"
      deliver_only: true          # 跳过 Agent，直接投递
      deliver: "dingtalk"
      deliver_extra:
        chat_id: "YOUR_DINGTALK_CHAT_ID"
```

`deliver_only: true` 时，渲染后的 prompt 直接作为消息投递，**不经过 LLM**，适合快速通知。

### 2.6 安全特性

| 安全机制 | 说明 |
|----------|------|
| **HMAC 签名** | 每个路由必须配置 secret，验证请求来源 |
| **速率限制** | 每路由固定窗口限流（默认 30 次/分钟） |
| **幂等性缓存** | 防止 webhook 重试导致重复执行（TTL 1小时） |
| **请求体大小限制** | 默认 1MB，防止大请求攻击 |
| **启动时验证** | 无 secret 的路由会拒绝启动 |
| **INSECURE_NO_AUTH 限制** | 仅在 loopback 地址上允许 |

### 2.7 可用的投递目标（deliver）

| 投递目标 | 说明 |
|----------|------|
| `log` | 仅记录到日志（默认） |
| `dingtalk` | 推送到钉钉 |
| `telegram` | 推送到 Telegram |
| `discord` | 推送到 Discord |
| `slack` | 推送到 Slack |
| `github_comment` | 发布为 GitHub PR/Issue 评论 |
| `email` | 发送邮件 |
| 其他 IM 平台 | 所有已配置的 Hermes 平台 |

---

## 三、API Server vs Webhook 对比

| 维度 | API Server | Webhook |
|------|-----------|---------|
| **通信模式** | 请求-响应（同步/异步） | 事件驱动（POST 触发） |
| **协议兼容** | OpenAI API 格式 | 自定义 JSON POST |
| **会话管理** | 支持（Session ID） | 每次独立触发 |
| **流式输出** | ✅ SSE 流式 | ❌ 结果投递到其他平台 |
| **长时间任务** | ✅ Run API（异步） | ✅ 后台执行，完成后投递 |
| **命令审批** | ✅ 远程审批 | ❌ 需配置 approvals.mode |
| **签名验证** | Bearer Token | HMAC Signature |
| **适合场景** | 聊天界面、API 集成 | 事件触发、自动化流水线 |
| **默认端口** | 8642 | 8644 |
| **LLM 成本** | 每次调用都有 | deliver_only 模式可零成本 |

---

## 四、快速选择指南

**选 API Server 如果：**
- 你要构建聊天界面（Web/App）
- 需要会话上下文保持
- 需要流式输出
- 你的客户端已经支持 OpenAI API 格式
- 需要远程审批 Agent 的命令执行

**选 Webhook 如果：**
- 你的应用是事件驱动的（CI/CD、监控、通知）
- 需要第三方服务（GitHub、JIRA、Stripe）触发 Agent
- 需要结果自动投递到钉钉/Telegram 等 IM
- 需要零 LLM 成本的快速通知（deliver_only）
- 多个 Agent 之间互相通信

---

*生成时间：2026-07-12 | Hermes Agent 版本：基于当前安装版本*
