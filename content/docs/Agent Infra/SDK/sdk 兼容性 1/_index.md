# AI SDK 向后兼容性保障方案

> **版本**: v0.1 | **风格**: DDIA 式 — 每个方案均有对应工业实践依据

---

## 一、AI SDK 向后兼容的核心挑战

传统 SDK 的兼容性问题是接口变更，而 AI SDK 面临**四层兼容性**，每一层的变更模式都不同：

| 层级 | 兼容性挑战 | 典型场景 | 传统方案是否适用 |
|------|-----------|----------|-----------------|
| **L1: Provider API 层** | 上游模型 API 变更（参数废弃、响应格式变更、端点迁移） | OpenAI gpt-3.5-turbo 废弃 model 参数、Anthropic messages API v1→v2 | 部分适用 |
| **L2: 模型能力层** | 同一 SDK 调用不同模型版本，能力/输出格式差异 | GPT-4 vs GPT-4o 的 tool calling 支持度不同 | **不适用** |
| **L3: 模型行为层** | 同模型版本，行为随时间漂移（prompt injection 修复、输出风格变化） | GPT-3.5 2023 版 vs 2024 版回答风格差异 | **不适用** |
| **L4: 应用接口层** | SDK 向下游暴露的 Python/JS API 变更 | LangChain 0.1→0.2 的 Breaking Change | 适用 |

**核心洞察**: AI SDK 的兼容性不仅是"接口不变"，更是"语义不变"。即使接口签名不变，模型行为漂移也会导致下游应用崩溃。

---

## 二、分层兼容性架构

```mermaid
graph TB
    subgraph "L4: 应用层（SDK 公共 API）"
        APP[用户代码: client.chat.completions.create(...)]
    end

    subgraph "L3: 语义适配层"
        SEM[语义适配: Prompt Template 版本化<br/>Output Schema 校验<br/>Capability Negotiation]
    end

    subgraph "L2: 模型路由层"
        RT[模型路由: Provider Adapter<br/>参数翻译: 旧格式→新格式<br/>Fallback Chain]
    end

    subgraph "L1: Provider SDK 层"
        P1[OpenAI SDK v1.40]
        P2[Anthropic SDK v0.25]
        P3[Google AI SDK v1.0]
    end

    APP --> SEM
    SEM --> RT
    RT --> P1
    RT --> P2
    RT --> P3

    style APP fill:#4a90d9,color:#fff
    style SEM fill:#50c878,color:#fff
    style RT fill:#ffd93d
    style P1 fill:#ff6b6b,color:#fff
    style P2 fill:#ff6b6b,color:#fff
    style P3 fill:#ff6b6b,color:#fff
```

**设计原则**: 每层只关心自己的兼容性责任，通过明确定义的边界向下传递兼容请求。

---

## 三、L4 应用接口层兼容性方案

### 3.1 语义化版本 + 明确生命周期

```
AI SDK 版本号: major.minor.patch

Breaking Change (major++):
  - 删除/重命名公共方法
  - 改变返回值类型结构
  - 移除已废弃参数

Feature Add (minor++):
  - 新增可选参数
  - 新增模型支持
  - 新增辅助功能

Bug Fix (patch++):
  - 修复 Provider API 适配 bug
  - 修复输出解析错误
```

**生命周期策略**：

| 阶段 | 时长 | 支持范围 | 用户动作 |
|------|------|----------|----------|
| **Active** | 最新 major | 全量支持 + Bug 修复 + 新功能 | 推荐升级 |
| **Maintenance** | 12 个月 | Bug 修复 + 安全补丁 | 计划迁移 |
| **EOL** | — | 无 | 必须迁移 |

> **工业实践**: OpenAI Python SDK 遵循此策略，v0.x → v1.x 的 Breaking Change 提供了 6 个月并行维护期。

### 3.2 废弃注解 + 自动迁移工具

```python
# SDK 侧：废弃注解
import warnings
import functools

def deprecated(
    replacement: str,
    remove_in_version: str,
    migration_guide: str = None
):
    """
    废弃装饰器：运行时警告 + IDE 静态提示
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            msg = (
                f"⚠️ {func.__name__} is deprecated and will be removed in v{remove_in_version}. "
                f"Use {replacement} instead."
            )
            if migration_guide:
                msg += f"\nMigration guide: {migration_guide}"
            warnings.warn(msg, DeprecationWarning, stacklevel=2)
            return func(*args, **kwargs)
        return wrapper
    return decorator

# 使用示例
class LegacyChatClient:
    @deprecated(
        replacement="client.chat.completions.create()",
        remove_in_version="2.0.0",
        migration_guide="https://docs.example.com/migrate/chat-v2"
    )
    def chat_completion(self, model: str, messages: list, **kwargs):
        return self.chat.completions.create(model=model, messages=messages, **kwargs)
```

**自动迁移工具（CLI）**：

```bash
# 扫描代码库，自动修复可迁移的废弃 API
ai-sdk-migrate --from openai==0.28 --to openai==1.40 ./src/

# 输出示例:
# ✅ Migrated: api.ChatCompletion.create() → client.chat.completions.create()
# ✅ Migrated: engine="text-davinci-003" → model="gpt-3.5-turbo"
# ⚠️ Manual review needed: Custom retry logic (line 45)
# Summary: 127 files, 342 fixes, 3 manual reviews needed
```

> **工业实践**: OpenAI 提供了 `openai-migrate` CLI 工具，自动将 v0.x 代码迁移到 v1.x。

### 3.3 扩展点设计：可选参数永远向后兼容

```python
# ✅ 正确：新增可选参数，不破坏现有调用
def chat_completion(
    model: str,
    messages: list[dict],
    temperature: float = 1.0,       # 已有参数
    max_tokens: int | None = None,  # 已有参数
    # --- 新增参数，全部可选 ---
    tools: list[dict] | None = None,         # v1.20 新增
    tool_choice: str | None = None,           # v1.20 新增
    response_format: dict | None = None,      # v1.30 新增
    seed: int | None = None,                  # v1.35 新增
    **kwargs  # 兜底：未知参数透传，不报错
) -> ChatCompletion:
    ...

# 用户代码不需要任何修改即可继续工作
client.chat_completion(model="gpt-4", messages=[...])  # 始终可用
```

**关键规则**：
- 新增参数必须有默认值
- 删除参数前，先标记 deprecated → 给过渡期 → 再删除
- 用 `**kwargs` 兜底未知参数，避免 SDK 升级后立即崩溃

---

## 四、L2 模型路由层兼容性方案

### 4.1 Provider 适配层（统一接口，多后端）

```python
from abc import ABC, abstractmethod
from typing import Generic, TypeVar

T = TypeVar("T")

class ModelProvider(ABC):
    """统一模型接口：屏蔽 OpenAI/Anthropic/Google 差异"""

    @abstractmethod
    def chat_completion(
        self,
        model: str,
        messages: list[dict],
        temperature: float = 1.0,
        max_tokens: int | None = None,
        tools: list[dict] | None = None,
        **kwargs
    ) -> dict: ...

class OpenAIProvider(ModelProvider):
    """OpenAI 适配器"""
    def chat_completion(self, model, messages, **kwargs):
        return openai_client.chat.completions.create(
            model=model, messages=messages, **kwargs
        )

class AnthropicProvider(ModelProvider):
    """Anthropic 适配器：参数翻译"""
    def chat_completion(self, model, messages, **kwargs):
        # 将 OpenAI 格式消息翻译为 Anthropic 格式
        anthropic_messages = self._convert_messages(messages)
        # 将 max_tokens 从 kwargs 移到顶层参数（Anthropic 要求）
        max_tokens = kwargs.pop("max_tokens", 4096)
        return anthropic_client.messages.create(
            model=model, messages=anthropic_messages,
            max_tokens=max_tokens, **kwargs
        )

    def _convert_messages(self, messages: list[dict]) -> list[dict]:
        """OpenAI role→content 格式转 Anthropic 格式"""
        converted = []
        for msg in messages:
            if msg["role"] == "system":
                # Anthropic 不支持 system role，放到 prompt 前缀
                converted.append({
                    "role": "user",
                    "content": f"[System]\n{msg['content']}"
                })
            else:
                converted.append(msg)
        return converted

class GoogleProvider(ModelProvider):
    """Google Gemini 适配器"""
    def chat_completion(self, model, messages, **kwargs):
        # Gemini 参数映射
        generation_config = {
            "temperature": kwargs.pop("temperature", 1.0),
            "max_output_tokens": kwargs.pop("max_tokens", 2048),
        }
        contents = self._convert_messages(messages)
        return gemini_client.generate_content(
            contents=contents,
            generation_config=generation_config,
            **kwargs
        )
```

**用户侧完全无感知**：

```python
# 同一套代码，切换模型自动适配底层 API
from ai_sdk import ChatClient

client = ChatClient(provider="openai", model="gpt-4o")
# 或
client = ChatClient(provider="anthropic", model="claude-3-sonnet")
# 或
client = ChatClient(provider="google", model="gemini-1.5-pro")

# 调用方式完全一致
response = client.chat_completion(
    messages=[{"role": "user", "content": "Hello"}],
    temperature=0.7,
    max_tokens=1024
)
```

> **工业实践**: LiteLLM、LangChain 的 ChatModel 抽象、Vercel AI SDK 均采用此模式。

### 4.2 参数翻译矩阵

```python
# 参数兼容性映射表
PARAM_MAPPING = {
    # OpenAI → Anthropic
    "max_tokens": {
        "openai": "max_tokens",
        "anthropic": "max_tokens",       # 同名但 Anthropic 必须传
        "google": "max_output_tokens",
    },
    "temperature": {
        "openai": "temperature",
        "anthropic": "temperature",
        "google": "temperature",
    },
    "top_p": {
        "openai": "top_p",
        "anthropic": "top_p",
        "google": "top_p",
    },
    "stop": {
        "openai": "stop",
        "anthropic": "stop_sequences",
        "google": "stop_sequences",
    },
    "tools": {
        "openai": "tools",
        "anthropic": "tools",            # 格式需转换
        "google": "tools",               # 格式需转换
    },
}

def translate_params(params: dict, source: str, target: str) -> dict:
    """将参数从一种 SDK 格式翻译为另一种"""
    translated = {}
    for key, value in params.items():
        if key in PARAM_MAPPING:
            target_key = PARAM_MAPPING[key].get(target, key)
            translated[target_key] = value
        else:
            translated[key] = value  # 未知参数原样传递
    return translated
```

### 4.3 Capability Negotiation（能力协商）

```python
from dataclasses import dataclass, field

@dataclass
class ModelCapability:
    """模型能力声明"""
    tool_calling: bool = False
    function_calling: bool = False
    json_mode: bool = False
    vision: bool = False
    audio: bool = False
    max_context: int = 4096
    max_output: int = 4096
    structured_output: bool = False

# 能力注册表
CAPABILITY_REGISTRY = {
    "gpt-4o": ModelCapability(
        tool_calling=True, json_mode=True, vision=True,
        max_context=128000, max_output=16384, structured_output=True
    ),
    "gpt-3.5-turbo": ModelCapability(
        tool_calling=True, json_mode=True,
        max_context=16385, max_output=4096
    ),
    "claude-3-sonnet": ModelCapability(
        tool_calling=True, json_mode=False, vision=True,
        max_context=200000, max_output=4096
    ),
    "gemini-1.5-pro": ModelCapability(
        tool_calling=True, json_mode=False, vision=True, audio=True,
        max_context=1000000, max_output=8192
    ),
}

def check_capability(model: str, required: list[str]) -> bool:
    """检查模型是否支持所需能力"""
    cap = CAPABILITY_REGISTRY.get(model)
    if not cap:
        raise ValueError(f"Unknown model: {model}")
    for req in required:
        if not getattr(cap, req, False):
            return False
    return True

# 使用示例
if not check_capability("gpt-3.5-turbo", ["vision"]):
    # 自动 fallback 到支持 vision 的模型
    model = auto_fallback("gpt-3.5-turbo", required=["vision"])
    # → "gpt-4o"
```

**能力协商的关键价值**: 当用户代码使用了某模型不支持的功能时，**在调用前报错**（而非静默失败或返回意外结果），并提供自动 fallback。

---

## 五、L3 语义适配层兼容性方案

### 5.1 Prompt Template 版本化

```python
# 问题：模型行为漂移导致同一 prompt 输出不一致
# 解决：Prompt Template 版本化 + 模型适配

from dataclasses import dataclass

@dataclass
class PromptTemplate:
    """带版本的 Prompt 模板"""
    id: str
    version: str          # 语义版本号
    template: str
    compatible_models: list[str]   # 验证过的模型列表
    fallback_template: str | None  # 兼容低能力模型的简化版

PROMPT_TEMPLATES = {
    "code-review": {
        "v2.0": PromptTemplate(
            id="code-review", version="v2.0",
            template="""You are a senior code reviewer. Analyze the following code:

Rules:
1. Identify security vulnerabilities (OWASP Top 10)
2. Suggest performance optimizations
3. Check for anti-patterns

Code:
```{language}
{code}
```

Respond in JSON format with keys: vulnerabilities, optimizations, patterns.""",
            compatible_models=["gpt-4o", "claude-3-opus", "gpt-4-turbo"],
            fallback_template="""Review this {language} code for issues:

{code}

List any problems found."""
        ),
        "v1.0": PromptTemplate(
            id="code-review", version="v1.0",
            template="Review this code: {code}",
            compatible_models=["gpt-3.5-turbo"],
            fallback_template=None
        ),
    }
}

def get_template(template_id: str, model: str) -> PromptTemplate:
    """获取与模型兼容的最新模板版本"""
    templates = PROMPT_TEMPLATES[template_id]
    # 按版本倒序，找到第一个兼容当前模型的
    for version in sorted(templates.keys(), reverse=True):
        tmpl = templates[version]
        if model in tmpl.compatible_models:
            return tmpl
    # 都不兼容，返回最近的 fallback
    latest = templates[sorted(templates.keys())[-1]]
    return PromptTemplate(
        id=template_id, version="fallback",
        template=latest.fallback_template or latest.template,
        compatible_models=[model],
        fallback_template=None
    )

# 使用
tmpl = get_template("code-review", "gpt-3.5-turbo")
# → 返回 v1.0（因为 v2.0 不兼容 gpt-3.5-turbo）
```

### 5.2 Output Schema 校验 + 自动修复

```python
import json
from pydantic import BaseModel, ValidationError

class CodeReviewResponse(BaseModel):
    """期望的响应结构"""
    vulnerabilities: list[str]
    optimizations: list[str]
    patterns: list[str]

def parse_response(raw: str, schema: type[BaseModel]) -> BaseModel:
    """解析 LLM 输出，失败时自动修复"""
    # 尝试直接解析
    try:
        data = json.loads(raw)
        return schema.model_validate(data)
    except (json.JSONDecodeError, ValidationError) as e:
        # 自动修复策略 1：提取 JSON 代码块
        import re
        json_match = re.search(r'```(?:json)?\s*\n(.*?)\n```', raw, re.DOTALL)
        if json_match:
            try:
                data = json.loads(json_match.group(1))
                return schema.model_validate(data)
            except (json.JSONDecodeError, ValidationError):
                pass

        # 自动修复策略 2：用 LLM 修复格式
        fix_prompt = f"""The following text should be JSON matching this schema:
{schema.model_json_schema()}

Original text:
{raw}

Extract and return ONLY valid JSON. No markdown, no explanation."""

        # 调用 LLM 修复（可用更便宜的模型）
        fix_response = cheap_client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": fix_prompt}],
            temperature=0
        )
        try:
            data = json.loads(fix_response.choices[0].message.content)
            return schema.model_validate(data)
        except (json.JSONDecodeError, ValidationError):
            raise ValueError(f"Cannot parse LLM response into {schema.__name__}: {e}")

# 使用示例
raw_response = llm_call(prompt)
parsed = parse_response(raw_response, CodeReviewResponse)
print(parsed.vulnerabilities)  # 保证是 list[str]
```

> **工业实践**: Instructor 库（github.com/instructor-ai/instructor）采用此模式，OpenAI 的 `response_format` 参数也是类似思路。

### 5.3 模型版本锁定 + 渐进迁移

```python
# 问题：provider 静默更新模型（gpt-3.5-turbo 指向不同快照）
# 解决：支持模型快照版本锁定

MODEL_SNAPSHOTS = {
    "gpt-3.5-turbo": {
        "current": "gpt-3.5-turbo-0125",     # 当前推荐
        "snapshots": {
            "2023-03": "gpt-3.5-turbo-0301",
            "2023-06": "gpt-3.5-turbo-0613",
            "2023-11": "gpt-3.5-turbo-1106",
            "2024-01": "gpt-3.5-turbo-0125",
        },
        "eol_snapshots": ["gpt-3.5-turbo-0301"],  # 已下线
    },
    "gpt-4": {
        "current": "gpt-4-0125-preview",
        "snapshots": {
            "2023-03": "gpt-4-0314",
            "2023-06": "gpt-4-0613",
            "2023-11": "gpt-4-1106-preview",
            "2024-01": "gpt-4-0125-preview",
        },
    },
}

class ModelResolver:
    """模型版本解析器"""
    def __init__(self, pin_snapshots: bool = True):
        self.pin_snapshots = pin_snapshots

    def resolve(self, model_alias: str, preferred_date: str | None = None) -> str:
        """将模型别名解析为具体快照版本"""
        if model_alias not in MODEL_SNAPSHOTS:
            return model_alias  # 非标准模型名，原样返回

        info = MODEL_SNAPSHOTS[model_alias]

        if preferred_date:
            # 用户指定了快照日期，精确锁定
            snapshot = info["snapshots"].get(preferred_date)
            if not snapshot:
                raise ValueError(f"No snapshot for {model_alias} at {preferred_date}")
            if snapshot in info.get("eol_snapshots", []):
                raise ValueError(f"Snapshot {snapshot} is EOL. Use {info['current']} instead.")
            return snapshot

        if self.pin_snapshots:
            # 默认锁定当前推荐版本
            return info["current"]

        # 不锁定，使用浮动别名（provider 默认行为）
        return model_alias

# 使用
resolver = ModelResolver(pin_snapshots=True)
actual_model = resolver.resolve("gpt-3.5-turbo")
# → "gpt-3.5-turbo-0125"（锁定当前快照）

# 需要回溯兼容时：
actual_model = resolver.resolve("gpt-3.5-turbo", preferred_date="2023-06")
# → "gpt-3.5-turbo-0613"（锁定历史快照）
```

---

## 六、L1 Provider API 层兼容性方案

### 6.1 Fallback Chain（降级链）

```python
import asyncio
from typing import Sequence

class FallbackChain:
    """多级降级：主模型失败时自动切换备选"""

    def __init__(self, models: Sequence[dict]):
        """
        models: [
            {"provider": "openai", "model": "gpt-4o", "priority": 1},
            {"provider": "anthropic", "model": "claude-3-sonnet", "priority": 2},
            {"provider": "openai", "model": "gpt-4o-mini", "priority": 3},
        ]
        """
        self.models = sorted(models, key=lambda m: m["priority"])

    async def chat_completion(self, **kwargs) -> dict:
        """按优先级尝试，失败自动降级"""
        last_error = None
        for model_spec in self.models:
            provider = get_provider(model_spec["provider"])
            try:
                return await provider.chat_completion(
                    model=model_spec["model"], **kwargs
                )
            except Exception as e:
                last_error = e
                # 记录降级事件
                logger.warning(
                    f"Fallback: {model_spec['provider']}/{model_spec['model']} failed: {e}"
                )
                continue

        raise RuntimeError(
            f"All models in fallback chain failed. Last error: {last_error}"
        )

# 使用
chain = FallbackChain([
    {"provider": "openai", "model": "gpt-4o", "priority": 1},
    {"provider": "anthropic", "model": "claude-3-sonnet", "priority": 2},
    {"provider": "openai", "model": "gpt-4o-mini", "priority": 3},
])

response = await chain.chat_completion(
    messages=[{"role": "user", "content": "Hello"}],
    temperature=0.7
)
```

> **工业实践**: LiteLLM 内置 fallback 机制；OpenRouter 支持多 provider 自动路由。

### 6.2 Provider API 版本适配层

```python
# 当 provider API 发生 Breaking Change 时，SDK 内部做兼容

class OpenAICompat:
    """OpenAI SDK 兼容层：屏蔽 v0.x → v1.x 差异"""

    def __init__(self, client):
        self.client = client
        self._is_v1 = hasattr(client, "chat")  # v1.x 有 chat 属性

    def create_completion(self, **kwargs):
        """统一的 completion 创建接口"""
        if self._is_v1:
            # v1.x: client.chat.completions.create()
            return self.client.chat.completions.create(**kwargs)
        else:
            # v0.x: client.ChatCompletion.create()
            return self.client.ChatCompletion.create(**kwargs)

    def parse_stream(self, response):
        """统一的流式响应解析"""
        if self._is_v1:
            # v1.x: chunk.choices[0].delta.content
            for chunk in response:
                delta = chunk.choices[0].delta
                if delta.content:
                    yield delta.content
        else:
            # v0.x: chunk.choices[0].get("delta", {}).get("content")
            for chunk in response:
                delta = chunk.choices[0].get("delta", {})
                content = delta.get("content")
                if content:
                    yield content
```

---

## 七、兼容性测试策略

### 7.1 兼容性矩阵测试

```yaml
# ci/compat-matrix.yaml
# 每次 PR 必须通过兼容性矩阵测试

test_matrix:
  # SDK 版本组合
  sdk_versions:
    - openai: "1.40.0"
    - openai: "1.30.0"   # 前一个 minor
    - anthropic: "0.25.0"
    - anthropic: "0.20.0"

  # 模型覆盖
  models:
    - openai/gpt-4o
    - openai/gpt-4o-mini
    - openai/gpt-3.5-turbo
    - anthropic/claude-3-sonnet
    - anthropic/claude-3-haiku

  # 功能覆盖
  features:
    - basic_chat           # 基础对话
    - tool_calling         # 工具调用
    - json_mode            # JSON 模式
    - streaming            # 流式输出
    - vision               # 多模态
    - structured_output    # 结构化输出

  # 断言
  assertions:
    - response_schema_valid    # 响应结构正确
    - no_unexpected_errors     # 无意外错误
    - backward_compatible_api  # 旧 API 调用仍可工作
```

### 7.2 回归测试：Golden Response 比对

```python
import json
from pathlib import Path

class GoldenTest:
    """
    Golden Response 测试：用已知正确的输出作为基准，
    验证 SDK 升级后输出结构不变
    """
    GOLDEN_DIR = Path(__file__).parent / "golden_responses"

    def __init__(self, test_name: str):
        self.test_name = test_name
        self.golden_file = self.GOLDEN_DIR / f"{test_name}.json"

    def assert_matches(self, actual_response: dict):
        """断言实际响应与 Golden 文件结构一致"""
        if not self.golden_file.exists():
            # 首次运行，生成 Golden 文件
            self.golden_file.write_text(
                json.dumps(self._strip_dynamic_fields(actual_response), indent=2)
            )
            return

        expected = json.loads(self.golden_file.read_text())
        actual = self._strip_dynamic_fields(actual_response)

        # 结构比对（忽略时间戳、ID 等动态字段）
        assert actual == expected, (
            f"Response structure changed!\n"
            f"Expected: {json.dumps(expected, indent=2)}\n"
            f"Actual:   {json.dumps(actual, indent=2)}\n"
            f"Run 'pytest --update-golden' to accept new structure"
        )

    def _strip_dynamic_fields(self, data: dict) -> dict:
        """移除动态字段（id、created、usage 等），只保留结构"""
        strip_keys = {"id", "created", "usage", "model", "system_fingerprint"}
        return {k: v for k, v in data.items() if k not in strip_keys}

# 使用
def test_chat_completion_structure():
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": "Say hi"}]
    ).model_dump()

    gt = GoldenTest("basic_chat")
    gt.assert_matches(response)
```

---

## 八、完整方案总结

| 兼容性层级 | 挑战 | 核心方案 | 辅助手段 |
|-----------|------|---------|---------|
| **L4 应用接口** | SDK 公共 API Breaking Change | 语义化版本 + 废弃注解 + 迁移 CLI | 可选参数、**kwargs 兜底 |
| **L3 语义适配** | 模型行为漂移、prompt 失效 | Prompt 模板版本化 + Output Schema 校验 | 自动修复、能力协商 |
| **L2 模型路由** | Provider API 差异、参数不兼容 | 统一接口 + 参数翻译矩阵 + Capability Registry | Fallback Chain、模型快照锁定 |
| **L1 Provider** | 上游 SDK Breaking Change | 内部适配层（如 OpenAICompat） | Golden Response 回归测试 |

### 设计哲学

```
传统 SDK 兼容性 = "接口不变"
AI SDK 兼容性   = "接口不变 + 语义不变 + 行为可预期"

关键公式：
向后兼容性 = 版本控制 + 能力协商 + 自动降级 + 输出校验
```

**一句话总结**: AI SDK 不能只靠版本号管理兼容性，必须建立 **能力声明（Capability）→ 适配层（Adapter）→ 校验层（Validator）→ 降级链（Fallback）** 四层防护，才能让下游应用在模型快速迭代中保持稳定。
