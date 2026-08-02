# 优秀开源 Agent Sandbox 项目（按 GitHub Star 排序）

> 数据获取时间：2026-07-13

## 什么是 Agent Sandbox？

Agent Sandbox（智能体沙箱）是为 AI Agent 提供安全代码执行环境的开源项目。它们通过隔离机制（容器、虚拟机、WebAssembly、系统调用过滤等）确保 AI 生成的代码可以在受控环境中安全运行，防止恶意代码逃逸或破坏宿主系统。

---

## 综合排行榜

| # | Stars | 语言 | 项目 | 简介 |
|---|-------|------|------|------|
| 1 | 72,197 | Go | [daytonaio/daytona](https://github.com/daytonaio/daytona) | 安全弹性的 AI 代码执行基础设施，支持按需创建隔离环境 |
| 2 | 56,575 | TypeScript | [appwrite/appwrite](https://github.com/appwrite/appwrite) | 全栈云平台，内置沙箱化代码执行引擎（含 Code Interpreter 功能） |
| 3 | 19,601 | HTML | [trycua/cua](https://github.com/trycua/cua) | 开源 Computer-Use Agent 基础设施，含沙箱、SDK 和基准测试 |
| 4 | 12,955 | Python | [e2b-dev/E2B](https://github.com/e2b-dev/E2B) | 企业级 Agent 安全执行环境，支持真实工具链和桌面环境 |
| 5 | 11,974 | Python | [opensandbox-group/OpenSandbox](https://github.com/opensandbox-group/OpenSandbox) | 安全、快速、可扩展的 AI Agent 沙箱运行时 |
| 6 | 4,309 | HTML | [judge0/judge0](https://github.com/judge0/judge0) | 高性能沙箱化在线代码执行系统，支持人和 AI |
| 7 | 3,142 | Go | [kubernetes-sigs/agent-sandbox](https://github.com/kubernetes-sigs/agent-sandbox) | K8s 原生 Agent 沙箱，管理隔离的有状态单例工作负载 |
| 8 | 2,753 | JavaScript | [engineer-man/piston](https://github.com/engineer-man/piston) | 高性能通用代码执行引擎，支持多语言沙箱运行 |
| 9 | 2,130 | Python | [e2b-dev/open-computer-use](https://github.com/e2b-dev/open-computer-use) | 基于开源 LLM + E2B Desktop Sandbox 的 Computer Use 方案 |
| 10 | 1,478 | TypeScript | [rivet-dev/sandbox-agent](https://github.com/rivet-dev/sandbox-agent) | 在沙箱中运行编码 Agent（Claude Code、Codex 等），通过 HTTP 控制 |
| 11 | 1,432 | Python | [e2b-dev/desktop](https://github.com/e2b-dev/desktop) | E2B 桌面沙箱，带 GUI 桌面环境的 LLM 沙箱 |
| 12 | 1,092 | Python | [vndee/llm-sandbox](https://github.com/vndee/llm-sandbox) | 轻量级可移植 LLM 沙箱运行时（Code Interpreter）Python 库 |
| 13 | 830 | Python | [agentscope-ai/agentscope-runtime](https://github.com/agentscope-ai/agentscope-runtime) | 生产级 Agent 运行时框架，含安全工具沙箱、Agent-as-a-Service API |
| 14 | 829 | Go | [abshkbh/arrakis](https://github.com/abshkbh/arrakis) | 可定制自托管的 AI Agent 代码执行和 Computer Use 沙箱方案 |
| 15 | 588 | Python | [mensfeld/code-on-incus](https://github.com/mensfeld/code-on-incus) | 为每个 AI Agent 提供独立 Incus 虚拟机，含主动防御机制 |
| 16 | 554 | TypeScript | [provos/ironcurtain](https://github.com/provos/ironcurtain) | 自治 AI Agent 安全运行时，用自然语言"宪法"定义策略 |
| 17 | 546 | Python | [SWE-agent/SWE-ReX](https://github.com/SWE-agent/SWE-ReX) | 沙箱化代码执行（本地/云端），大规模并行，SWE-agent 底层引擎 |
| 18 | 493 | Python | [modal-labs/modal-client](https://github.com/modal-labs/modal-client) | Modal 云沙箱 SDK，serverless 代码执行环境 |
| 19 | 467 | Rust | [Th0rgal/sandboxed.sh](https://github.com/Th0rgal/sandboxed.sh) | 链上 AI Agent 安全运行时，Wasm 隔离沙箱 |
| 20 | 426 | C | [jamesstringer90/appsandbox](https://github.com/jamesstringer90/appsandbox) | 为开发或 Computer Use 模型创建完整沙箱化虚拟机 |
| 21 | 293 | Rust | [capsulerun/capsule](https://github.com/capsulerun/capsule) | 基于 WebAssembly 的 AI Agent 任务安全沙箱运行时 |
| 22 | 267 | Rust | [portofcontext/pctx](https://github.com/portofcontext/pctx) | Agent 工具调用执行层，自动转换 Agent 工具和 MCP Server 为代码执行 |
| 23 | 256 | Go | [GreyhavenHQ/greywall](https://github.com/GreyhavenHQ/greywall) | 无容器、默认拒绝的 AI 编码 Agent 沙箱，内核级文件系统/网络/syscall 隔离 |
| 24 | 253 | Go | [FootprintAI/Containarium](https://github.com/FootprintAI/Containarium) | 开源 Agent 运行时，SSH 原生隔离、eBPF 出口策略、K8s+LXC 后端、GPU 透传 |
| 25 | 166 | Go | [agent-sandbox/agent-sandbox](https://github.com/agent-sandbox/agent-sandbox) | E2B 兼容的企业级 Agent 沙箱，支持安全代码执行 |
| 26 | 70 | - | [pocketenv-io/pocketenv](https://github.com/pocketenv-io/pocketenv) | Agent 和人类的通用沙箱运行时 |
| 27 | 63 | Python | [ClickHouse/code-interpreter](https://github.com/ClickHouse/code-interpreter) | 沙箱化代码执行 API，支撑 LibreChat 的 Code Interpreter |
| 28 | 63 | TypeScript | [typper-io/ai-code-sandbox](https://github.com/typper-io/ai-code-sandbox) | 基于 Docker 的安全 Python 沙箱，安全运行 LLM 输出 |

---

## 分类详解

### 第一梯队：核心基础设施（⭐ > 10,000）

#### 1. Daytona (72,197⭐)
- **定位**：安全弹性的 AI 代码执行基础设施
- **核心能力**：按需创建隔离沙箱，支持弹性扩缩容
- **适用场景**：多 Agent 并发执行、云端部署
- **语言**：Go

#### 2. Appwrite (56,575⭐)
- **定位**：全栈后端云平台，内置沙箱代码执行
- **核心能力**：Auth、Database、Storage + Code Interpreter
- **适用场景**：需要完整后端 + AI 代码执行的场景
- **语言**：TypeScript
- **注意**：不只是沙箱，是完整云平台

#### 3. CUA (19,601⭐)
- **定位**：Computer-Use Agent 开源基础设施
- **核心能力**：沙箱 + SDK + 基准测试，专为桌面操控 Agent 设计
- **适用场景**：训练和评估 Computer Use Agent
- **语言**：多语言（HTML 为主的项目结构）

#### 4. E2B (12,955⭐)
- **定位**：企业级 Agent 安全执行环境
- **核心能力**：真实工具链、桌面环境（含 GUI）、多语言支持
- **适用场景**：企业级 Agent 部署、Computer Use
- **语言**：Python
- **生态**：E2B 生态下有多个子项目（desktop、surf、open-computer-use）

#### 5. OpenSandbox (11,974⭐)
- **定位**：安全、快速、可扩展的 AI Agent 沙箱运行时
- **核心能力**：专注 Agent 代码执行的安全隔离
- **适用场景**：AI Agent 代码执行环境
- **语言**：Python

### 第二梯队：成熟代码执行引擎（⭐ 1,000–10,000）

#### 6. Judge0 (4,309⭐)
- **定位**：高性能沙箱化代码执行系统
- **核心能力**：支持 60+ 编程语言，快速、可扩展
- **适用场景**：在线编程平台、AI Code Interpreter
- **语言**：Go (基于 Docker + isolate)

#### 7. Kubernetes Agent-Sandbox (3,142⭐)
- **定位**：K8s 原生 Agent 沙箱管理
- **核心能力**：隔离、有状态、单例工作负载管理
- **适用场景**：Kubernetes 集群内的 Agent 隔离执行
- **语言**：Go
- **特点**：Kubernetes SIGs 官方项目

#### 8. Piston (2,753⭐)
- **定位**：高性能通用代码执行引擎
- **核心能力**：多语言代码执行，广泛使用
- **适用场景**：Discord bot、在线编程平台、AI 代码执行
- **语言**：JavaScript

### 第三梯队：新兴/专用方案（⭐ < 1,000）

#### 10. Rivet Sandbox-Agent (1,478⭐)
- **定位**：在沙箱中运行编码 Agent
- **核心能力**：支持 Claude Code、Codex、OpenCode、Amp，HTTP 控制
- **适用场景**：云端编码 Agent 部署

#### 12. LLM-Sandbox (1,092⭐)
- **定位**：轻量级可移植 LLM 沙箱运行时
- **核心能力**：Python 库，即插即用
- **适用场景**：快速集成 Code Interpreter 到应用

#### 14. Arrakis (829⭐)
- **定位**：可定制自托管沙箱方案
- **核心能力**：代码执行 + Computer Use，自托管
- **适用场景**：私有化部署的 Agent 沙箱

#### 17. SWE-ReX (546⭐)
- **定位**：SWE-agent 底层沙箱引擎
- **核心能力**：大规模并行、本地/云端执行
- **适用场景**：软件工程的 Agent 代码执行

#### 19. Sandboxed.sh (467⭐)
- **定位**：链上 AI Agent 安全运行时
- **核心能力**：Wasm 隔离、加密密钥、链上交互
- **适用场景**：区块链/DeFi Agent

#### 21. Capsule (293⭐)
- **定位**：WebAssembly AI Agent 沙箱
- **核心能力**：Wasm 隔离，安全运行不可信代码
- **适用场景**：轻量级沙箱执行

#### 23. Greywall (256⭐)
- **定位**：无容器内核级沙箱
- **核心能力**：内核级文件系统/网络/syscall 隔离，无需容器
- **适用场景**：轻量高性能 Agent 沙箱

#### 24. Containarium (253⭐)
- **定位**：SSH 原生 Agent 运行时
- **核心能力**：eBPF 出口策略、K8s+LXC 后端、GPU 透传
- **适用场景**：需要 GPU 的 Agent 任务

---

## 技术路线对比

| 隔离方式 | 代表项目 | 优点 | 缺点 |
|---------|---------|------|------|
| **容器/Docker** | Judge0, Piston, OpenSandbox | 成熟、易用 | 隔离性有限，可能有逃逸风险 |
| **虚拟机/Incus** | Code-on-Incus, AppSandbox | 强隔离 | 资源开销大，启动慢 |
| **WebAssembly** | Sandboxed.sh, Capsule | 轻量、安全、跨平台 | 语言支持受限 |
| **内核级隔离** | Greywall, Containarium | 极强隔离，无容器开销 | 需要内核支持，部署复杂 |
| **K8s 原生** | kubernetes-sigs/agent-sandbox | 原生编排，可扩展 | 依赖 K8s 集群 |
| **混合/平台** | E2B, Daytona, Modal | 全托管，功能丰富 | 依赖第三方服务 |

---

## 选型建议

| 场景 | 推荐 | 理由 |
|------|------|------|
| 快速集成 Code Interpreter | **LLM-Sandbox** | 轻量 Python 库，几行代码即可用 |
| 企业级生产部署 | **E2B** 或 **Daytona** | 成熟生态，企业级安全 |
| 大规模多语言执行 | **Judge0** 或 **Piston** | 60+ 语言，久经考验 |
| Computer Use Agent | **E2B Desktop** 或 **CUA** | 专为桌面操控设计 |
| 自托管/私有化 | **Arrakis** 或 **OpenSandbox** | 完全自托管，数据不出域 |
| K8s 环境 | **kubernetes-sigs/agent-sandbox** | K8s 原生，无缝集成 |
| 极致安全 | **Greywall** 或 **Containarium** | 内核级/eBPF 隔离 |
| 区块链 Agent | **Sandboxed.sh** | Wasm 隔离 + 链上交互 |
| 编码 Agent 部署 | **Rivet Sandbox-Agent** | 专为编码 Agent 优化 |
| 轻量无容器 | **Capsule** | Wasm 运行时，极低开销 |
