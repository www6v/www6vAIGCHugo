# Hermes Agent Loop 中止机制全解析

## 一、核心数据结构：中断信号是线程安全的

整个中断机制的核心在 `tools/interrupt.py`，它实现了一个**线程级别的中断信号表**：

```python
# tools/interrupt.py
_interrupted_threads: set[int] = set()   # 被标记中断的线程 ID 集合
_lock = threading.Lock()

def set_interrupt(active: bool, thread_id: int | None = None):
    """将某个线程标记为中断/清除中断"""
    tid = thread_id or threading.current_thread().ident
    with _lock:
        if active:
            _interrupted_threads.add(tid)
        else:
            _interrupted_threads.discard(tid)

def is_interrupted() -> bool:
    """当前线程是否被标记中断"""
    return threading.current_thread().ident in _interrupted_threads
```

**关键设计**：按线程隔离。Gateway 中多个 agent 会话并发运行在同一进程中，中断一个会话不会影响其他会话。

## 二、Agent 侧的中断接口：`interrupt()` 方法

`AIAgent` 类（`run_agent.py`）暴露了两个关键方法：

```python
# run_agent.py:2237
def interrupt(self, message: str = None):
    """请求中断当前工具调用循环"""
    self._interrupt_requested = True     # 标志位
    self._interrupt_message = message    # 附带的新消息
    # 向当前执行线程发送中断信号
    _set_interrupt(True, self._execution_thread_id)
    # 传播给所有子 agent（delegate_task 创建的子任务）
    for child in self._running_children:
        child.interrupt(message)
```

`clear_interrupt()` 做相反操作：清除标志、清除线程信号、清理子 agent。

## 三、主循环中的中断检查点

主循环在 `agent/conversation_loop.py:run_conversation()` 中，有多层中断检测：

```python
# 1. 循环入口检测
while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) \
        or agent._budget_grace_call:
    
    # ★ 每次循环迭代开始都检查
    if agent._interrupt_requested:
        interrupted = True
        _turn_exit_reason = "interrupted_by_user"
        break
    
    api_call_count += 1
    
    # ... 调用 LLM、执行工具调用、处理结果 ...
    
    # 2. 工具调用结果处理中也检查（约 1569 行）
    if agent._interrupt_requested:
        break
    
    # 3. 资源清理前检查（约 2706 行）
    if agent._interrupt_requested:
        break
```

## 四、API 调用级别的中断（最精妙的部分）

API 调用在 `agent/chat_completion_helpers.py` 中实现了**可中断的后台线程模式**：

```python
def interruptible_api_call(agent, api_kwargs):
    """在后台线程中执行 API 调用，允许主循环检测中断"""
    result = {"response": None, "error": None}
    
    def _call():
        """实际 API 调用在子线程中运行"""
        try:
            client = _set_request_client(create_client(...))
            result["response"] = client.chat.completions.create(**api_kwargs)
        except Exception as e:
            result["error"] = e
    
    # ★ 启动后台线程
    worker = threading.Thread(target=_call)
    worker.start()
    
    # ★ 主线程轮询中断信号
    while worker.is_alive():
        if agent._interrupt_requested:
            # 关闭 HTTP 连接，让后台线程的 recv() 收到 EPIPE
            _close_request_client_once("interrupt_abort")
            break
        worker.join(timeout=0.5)  # 每 0.5 秒检查一次
    
    return result
```

**设计要点**：
- API 调用在独立线程中执行
- 主线程每 0.5 秒检查一次 `_interrupt_requested`
- 中断时通过关闭 HTTP socket 让后台线程的 `recv()` 阻塞提前返回
- 避免了直接 `Thread._stop()` 导致的资源泄漏（FD 回收、SSL BIO 写坏等）

流式调用（`interruptible_streaming_api_call`）同理，在 SSE 事件循环中也会检查中断信号。

## 五、工具执行中的中断

工具在执行长操作时（如 `terminal` 后台进程、网络请求），通过 `is_interrupted()` 主动检测：

```python
# 任何工具内部都可以这样检测
from tools.interrupt import is_interrupted

if is_interrupted():
    return json.dumps({"output": "[interrupted]", "returncode": 130})
```

## 六、触发中断的用户入口

| 触发方式 | 代码路径 | 说明 |
|---------|---------|------|
| **CLI 中 Ctrl+C** | `cli.py` 信号处理器 → `agent.interrupt()` | 键盘中断 |
| **`/new` 或 `/reset`** | `cli.py` / `gateway/run.py` | 新会话会先 interrupt 当前 agent |
| **`/stop`** | gateway 的 Level-2 handler | 中止后台进程 + interrupt agent |
| **Gateway 收到新消息** | `gateway/run.py:18617` | 用户发来新消息 → interrupt 当前 turn → 处理新消息 |
| **超时自动中断** | `gateway/run.py:18858` | 超时后自动 interrupt + 清理 |
| **`delegate_task` 子任务** | `run_agent.py:2299` | 父 agent 中断时传播给所有子 agent |

## 七、中断后的处理流程

```
中断触发
  ↓
agent._interrupt_requested = True
  ↓
_set_interrupt(True, execution_thread_id)
  ↓
传播给所有子 agent
  ↓
主循环在下一个检查点 break
  ↓
记录 _turn_exit_reason = "interrupted_by_user"
  ↓
执行资源清理（VM、浏览器、终端进程）
  ↓
clear_interrupt() 清理状态
  ↓
如果附带新消息 → 作为下一条 user message 继续
如果 /new → 开启全新会话
```

## 八、与普通 `/steer` 的区别

Hermes 还提供了 `/steer` 命令（注入消息但**不中断**）：

```python
def steer(self, text: str):
    """注入消息到下一个 tool result 中，不打断当前执行"""
    self._steer_message = text
    # 注意：这里不调用 interrupt()
```

`/steer` 适用于“想给建议但不打断当前工具调用”的场景，而 `/new` 和新消息触发的是**软中断（soft interrupt）**，会优雅地中止当前循环。

## 总结

Hermes 的中止机制是一个**线程安全的标志位 + 多检查点轮询 + HTTP 连接优雅关闭**的组合方案，保证了即使 API 调用卡住或工具执行耗时，也能在数秒内安全退出，且不影响其他并发会话。
