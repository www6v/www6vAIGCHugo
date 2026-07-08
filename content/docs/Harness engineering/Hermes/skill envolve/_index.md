# Hermes Agent Skill 自演化机制深度调研

## 1. 总览：自演化的核心理念

Hermes Agent 的 skill 自演化建立在 **"Agent 是自身知识库的维护者"** 这一核心理念上。Skill 不是静态文档，而是**可写、可改、可生长、可衰老、可合并**的程序化记忆（procedural memory）。整个系统由以下关键组件协作完成：

```
Agent（AI）                     Curator（策展人）                    持久化层
┌──────────────────┐          ┌────────────────────┐          ┌──────────────────────┐
│  skill_manage()  │ ◄─────── │  maybe_run_curator()│          │  ~/.hermes/skills/   │
│  创建/修改技能    │          │  后台维护/合并/归档  │─────────►│  ├── skill-a/       │
│                  │ ───────► │                    │          │  ├── skill-b/       │
│  skills_list()   │          │  apply_automatic_   │          │  ├── .archive/       │
│  skill_view()    │          │    transitions()    │          │  ├── .usage.json     │
│  ↑               │          │                    │          │  └── .curator_state  │
│  │               │          │  run_llm_review()   │          └──────────────────────┘
│  遥测计数器       │          └────────────────────┘
│  bump_use/view/  │
│  patch()         │
└──────────────────┘
```

## 2. Skill 的生命周期状态机

每个 agent 创建的 skill 都经历以下状态流转（`tools/skill_usage.py`）：

```
                    ┌──────────────────────────────────────────────┐
                    │                                              ▼
  [新创建] ─────► active ──(超过stale_after_days未使用)──► stale
  STATE_ACTIVE      │  ▲                                       │
                    │  │                                       │ (超过archive_after_days)
                    │  │ (被重新使用)                           ▼
                    │  └───────────────────────────── archived (移入 .archive/)
                    │                                              ▲
                    │                                              │
                    └── pinned (用户手动固定，跳过所有自动流转) ───┘
```

**关键时间参数**（可通过 `config.yaml` 的 `curator.*` 调整）：
- `stale_after_days`：默认 **30 天** 未使用标记为 stale
- `archive_after_days`：默认 **90 天** 未使用移入归档
- `interval_hours`：Curator 运行间隔，默认 **7 天**（168 小时）
- `min_idle_hours`：Agent 空闲至少 **2 小时** 才触发 Curator

## 3. 自演化的三层架构

### 3.1 第一层：Agent 自主创建和修改（运行时自演化）

Agent 在执行任务过程中，通过 `skill_manage` tool 自主管理技能。

**`tools/skill_manager_tool.py` 提供 6 种操作：**

| Action | 触发时机 | 行为 |
|--------|----------|------|
| `create` | 完成复杂任务（5+ 次 tool 调用）、克服错误、用户纠正后 | 在 `~/.hermes/skills/` 下创建新目录 + SKILL.md |
| `patch` | 发现 skill 过时、遗漏坑点 | 模糊匹配替换 SKILL.md 中的特定文本 |
| `edit` | 大规模重写 | 完全替换 SKILL.md 内容 |
| `write_file` | 添加参考文档/模板/脚本 | 写入 `references/`, `templates/`, `scripts/`, `assets/` |
| `delete` | skill 不再需要 | 删除整个目录 |
| `remove_file` | 移除支持文件 | 删除指定子文件 |

**关键特性：**
- **原子写入**：所有文件写入使用 `tempfile + os.replace` 确保崩溃安全
- **安全扫描**（可选）：写入后可触发 `skills_guard` 安全扫描，检测危险内容
- **模糊匹配**：patch 使用 fuzzy_match 引擎，容忍空格/缩进差异
- **前置验证**：frontmatter 必须合法，name ≤ 64 字符，description ≤ 1024 字符，总内容 ≤ 100,000 字符
- **Pinned 保护**：用户 pin 的 skill 对 agent 写操作免疫

### 3.2 第二层：Curator 策展人（后台维护）

`agent/curator.py` 是整个自演化系统的核心大脑。

**触发机制**（非 cron 守护进程，而是空闲触发）：

```python
def maybe_run_curator(*, idle_for_seconds, on_summary):
    # 在 session 启动时检查：
    # 1. curator.enabled == True
    # 2. 未暂停
    # 3. 上次运行超过 interval_hours
    # 4. Agent 空闲超过 min_idle_hours
    # 全部满足 → 启动
```

**Curator 执行两步走：**

#### Step 1：自动状态迁移（纯逻辑，无需 LLM）

```python
def apply_automatic_transitions():
    for skill in agent_created_report():
        if pinned: continue
        anchor = last_activity or created_at
        if anchor <= archive_cutoff: archive_skill(name)      # 90天未用 → 归档
        elif anchor <= stale_cutoff: set_state(STALE)         # 30天未用 → 标记过期
        elif anchor > stale_cutoff and state == STALE: set_state(ACTIVE)  # 重新使用 → 恢复
```

#### Step 2：LLM 巩固合并（fork 一个独立 AIAgent）

这是最精妙的部分——**用一个独立的 AI Agent 来整理和合并现有的 skills**：

```python
def run_curator_review():
    # 1. 自动状态迁移（Step 1）
    apply_automatic_transitions()
    
    # 2. 生成候选列表（含使用统计）
    candidate_list = _render_candidate_list()
    # 输出类似：
    # - hermes-config-fix  state=active  pinned=no  activity=12  use=8  patches=3
    # - hermes-dashboard   state=active  pinned=no  activity=5   use=3  patches=1
    
    # 3. Fork 一个独立 AIAgent 执行巩固任务
    review_agent = AIAgent(
        max_iterations=9999,        # 高迭代上限
        platform="curator",         # 标记为 curator 平台
        skip_context_files=True,    # 跳过上下文文件
        skip_memory=True,           # 跳过 memory
    )
    review_agent._memory_nudge_interval = 0  # 禁止递归触发 curator
    review_agent._skill_nudge_interval = 0
    
    # 4. 执行对话
    review_agent.run_conversation(
        user_message=CURATOR_REVIEW_PROMPT + "\n\n" + candidate_list
    )
```

**Curator 的指令（CURATOR_REVIEW_PROMPT）核心原则：**
- **目标**：将技能集合维护为 **"类别级指令库"**，而非"一次会话一个 skill"的碎片
- **合并策略**（三种方式）：
  - **MERGE INTO EXISTING UMBRELLA**：已有宽泛 skill 作为伞，patch 它添加子章节，归档窄 skill
  - **CREATE NEW UMBRELLA**：创建新的类别级 SKILL.md，覆盖共享工作流
  - **DEMOTE TO REFERENCES/TEMPLATES/SCRIPTS**：窄但有价值的内容移入支持文件
- **硬规则**：
  - 不碰 bundled 或 hub-installed skill
  - **绝不删除**，最大破坏动作是归档（可恢复）
  - 不碰 pinned skill
  - 不用使用计数器作为跳过合并的理由——基于**内容重叠**判断

#### 报告和审计

每次 Curator 运行生成完整报告：
```
~/.hermes/logs/curator/YYYYMMDD-HHMMSS/
├── run.json              # 机器可读完整记录
├── REPORT.md             # 人类可读报告
└── cron_rewrites.json    # （如有）Cron job skill 引用重写
```

报告包含：
- 自动迁移计数（stale/archived/reactivated）
- LLM 合并详情（哪些 skill 被合并到哪些 umbrella）
- 修剪列表（哪些 skill 被归档且未合并）
- Cron job 自动重写（当 skill 被合并时，自动更新引用的定时任务）

### 3.3 第三层：使用遥测和生命周期（数据驱动）

`tools/skill_usage.py` 维护 `.usage.json` sidecar 文件：

```json
{
  "hermes-config-fix": {
    "use_count": 8,
    "view_count": 15,
    "patch_count": 3,
    "created_at": "2025-01-15T10:30:00+00:00",
    "last_used_at": "2025-02-10T14:20:00+00:00",
    "last_viewed_at": "2025-02-11T09:15:00+00:00",
    "last_patched_at": "2025-02-08T16:45:00+00:00",
    "state": "active",
    "pinned": false,
    "archived_at": null
  }
}
```

**计数器自动递增点：**
- `bump_use()`：skill 被主动使用时（`/skill-name` 或 `build_skill_invocation_message`）
- `bump_view()`：`skill_view()` 成功返回时
- `bump_patch()`：`skill_manage(patch/edit/write_file/remove_file)` 成功时
- `forget()`：skill 被删除时

**来源判定**：`is_agent_created()` 通过比对 `.bundled_manifest` 和 `.hub/lock.json` 判断 skill 来源。Curator 只操作 agent-created skill。

## 4. Skill 发现和加载机制

### 4.1 渐进式披露（Progressive Disclosure）

Hermes 采用 Anthropic 推荐的渐进式披露架构：

| 层级 | 内容 | 令牌成本 | 触发 |
|------|------|----------|------|
| Tier 1 | name + description（skills_list） | 极低 | 系统 prompt 中的 skills index |
| Tier 2 | SKILL.md 完整内容（skill_view） | 中 | Agent 主动请求查看 |
| Tier 3 | references/templates/scripts/assets | 按需 | Agent 指定 file_path |

### 4.2 系统 Prompt 注入

`agent/prompt_builder.py` 在系统 prompt 中注入 skills index：
- 扫描所有 `~/.hermes/skills/` 目录
- 提取每个 skill 的 `name` + `description`（≤1024 字符）
- 格式化为精简列表，供 Agent 知道有哪些 skill 可用
- 带 prompt 缓存保护，变更时通过 `clear_skills_system_prompt_cache()` 刷新

### 4.3 斜杠命令自动注册

`agent/skill_commands.py` 自动将 skill 注册为 `/skill-name` 命令：
- 扫描 skills 目录，将 skill name 转换为 slug（连字符分隔）
- 用户输入 `/hermes-agent` 时，自动加载对应 SKILL.md 作为 user message
- 同时支持 CLI 和 Gateway（Telegram/Discord 等）

### 4.4 SKILL.md 预处理

`agent/skill_preprocessing.py` 在加载时动态处理 skill 内容：
- **模板变量替换**：`${HERMES_SKILL_DIR}`, `${HERMES_SESSION_ID}` → 实际值
- **内联 shell 执行**：`` !`date +%Y-%m-%d` `` → 命令输出（可选，默认关闭）

## 5. 安全机制

自演化不是无限制的，有多层安全护栏：

| 安全层 | 机制 | 实现 |
|--------|------|------|
| 命名验证 | lowercase, hyphens, ≤64字符 | `_validate_name()` |
| Frontmatter验证 | 必须有name/description, YAML合法 | `_validate_frontmatter()` |
| 大小限制 | SKILL.md ≤100K字符, 支持文件 ≤1MiB | `_validate_content_size()` |
| 路径遍历防护 | 禁止 `..` 逃逸 | `path_security.has_traversal_component()` |
| Prompt注入检测 | 检测"ignore previous instructions"等模式 | `_INJECTION_PATTERNS` |
| 安全扫描（可选） | 检测脚本中的危险操作 | `skills_guard.scan_skill()` |
| Pinned保护 | 用户pin的skill不可修改 | `_pinned_guard()` |
| 来源隔离 | Curator不碰bundled/hub skill | `is_agent_created()` |

## 6. 文件结构全景

```
~/.hermes/
├── config.yaml                          # curator.* 配置
├── skills/
│   ├── hermes-agent/SKILL.md             # bundled skill
│   ├── my-custom-skill/                  # agent-created skill
│   │   ├── SKILL.md
│   │   ├── references/
│   │   ├── templates/
│   │   └── scripts/
│   ├── .archive/                         # 归档目录（可恢复）
│   │   └── old-skill/
│   ├── .usage.json                       # 使用遥测 sidecar
│   ├── .curator_state                    # Curator 调度状态
│   ├── .bundled_manifest                 # bundled skill 清单
│   └── .hub/lock.json                    # hub-installed skill 清单
└── logs/curator/
    └── 20260520-140000/
        ├── run.json                      # 机器可读报告
        └── REPORT.md                     # 人类可读报告
```

## 7. 配置项（config.yaml）

```yaml
curator:
  enabled: true                    # 默认开启
  interval_hours: 168              # 运行间隔：7天
  min_idle_hours: 2                # 最少空闲时间
  stale_after_days: 30             # 过期阈值
  archive_after_days: 90           # 归档阈值

skills:
  guard_agent_created: false       # 安全扫描（默认关闭）
  disabled: []                     # 用户禁用的 skill 列表
  platform_disabled: {}            # 按平台禁用
  template_vars: true              # 启用模板变量替换
  inline_shell: false              # 启用内联 shell 执行
  inline_shell_timeout: 10         # 超时秒数

auxiliary:
  curator:                         # Curator 专用模型配置
    provider: auto                 # 默认用主模型
    model: ""
```

## 8. 自演化闭环总结

```
                    ┌─────────────────────────────────────┐
                    │                                     │
    用户任务 ─────► Agent 执行任务                          │
                    │                                     │
    │               ├─► 成功 → skill_manage(create) 保存经验 │
    │               ├─► 出错 → skill_manage(patch) 修复 skill │
    │               └─► 查看 → bump_view/use 更新遥测        │
    │               │                                     │
    │               ▼                                     │
    │         ~/.hermes/skills/ 技能集合                   │
    │               │                                     │
    │         空闲触发（7天 + 2h空闲）                       │
    │               │                                     │
    │               ▼                                     │
    │         Curator 运行：                                │
    │         1. 自动迁移（active→stale→archived）          │
    │         2. Fork AIAgent 执行 LLM 合并               │
    │         3. 生成报告 + 重写 Cron 引用                  │
    │               │                                     │
    └───────────────┘                                     │
                                                          │
    下一轮任务 ──► 使用优化后的 skill 集合 ─────────────────┘
```

**核心洞见**：Hermes Agent 的 skill 自演化不是简单的"自动更新"，而是一个**生态系统式的自组织过程**——Agent 像园丁一样种植新 skill，Curator 像园丁一样修剪、合并、淘汰，使用数据像土壤养分一样驱动决策。整个系统形成了"经验积累 → 知识沉淀 → 自动整理 → 更好执行"的闭环。

## 9. 关键源码文件索引

| 文件 | 作用 |
|------|------|
| `tools/skill_manager_tool.py` | Agent 自主创建、修改、删除 skill 的工具 |
| `tools/skills_tool.py` | skills_list/skill_view 渐进式披露实现 |
| `tools/skill_usage.py` | 使用遥测、生命周期状态、归档/恢复 |
| `agent/curator.py` | 策展人：自动迁移 + LLM 合并审查 |
| `agent/skill_commands.py` | 斜杠命令扫描与注入 |
| `agent/skill_preprocessing.py` | 模板变量替换与内联 shell 扩展 |
| `agent/skill_utils.py` | 工具函数：解析 frontmatter、平台匹配等 |
| `agent/prompt_builder.py` | 系统 prompt 中 skills index 的构建 |
