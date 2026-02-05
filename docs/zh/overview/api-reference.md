# API 参考手册

本页面记录了 LoongFlow 框架的公共 API。涵盖开发者在构建智能体或扩展框架时需要的所有类、方法和接口。

---

## 框架 API

### AgentBase

所有智能体的抽象基类。提供生命周期管理、钩子和错误处理。

**模块：** `loongflow.framework.base`

```python
from loongflow.framework.base import AgentBase
```

| 方法 | 签名 | 描述 |
|------|------|------|
| `run` | `async def run(*args, **kwargs) -> Message` | **抽象方法。** 主智能体逻辑 |
| `interrupt` | `async def interrupt() -> None` | 优雅停止智能体 |
| `register_hook` | `def register_hook(hook_type: str, hook_fn: Callable) -> None` | 注册前/后钩子 |

### PESAgent

规划-执行-总结进化智能体。管理并发进化循环。

**模块：** `loongflow.framework.pes`

```python
from loongflow.framework.pes import PESAgent

agent = PESAgent(config=config, checkpoint_path=checkpoint_path)
agent.register_planner_worker("planner", MyPlannerWorker)
agent.register_executor_worker("executor", MyExecutorWorker)
agent.register_summary_worker("summary", MySummaryWorker)
result = await agent()
```

### ReActAgent

推理-行动-观察循环智能体。

**模块：** `loongflow.framework.react`

```python
from loongflow.framework.react import ReActAgent

agent = ReActAgent.create_default(
    model=model, sys_prompt="...", toolkit=toolkit, max_steps=15
)
result = await agent(Message.from_text("..."))
```

---

## Agent SDK API

### Message

通用数据容器，用于所有组件间通信。

```python
from loongflow.agentsdk.message import Message, Role

msg = Message.from_text("Hello", sender="user", role=Role.USER)
```

### 工具系统

```python
from loongflow.agentsdk.tools import Toolkit, ShellTool, ReadTool

toolkit = Toolkit()
toolkit.register_tool(ShellTool())
toolkit.register_tool(ReadTool())
```

### LLM 模型

```python
from loongflow.agentsdk.models import LiteLLMModel

model = LiteLLMModel(
    model_name="openai/gemini-3-pro-preview",
    base_url="https://...",
    api_key="..."
)
```

---

更多详细内容，请参考 [英文版 API 参考](../../en/overview/api-reference.md)。
