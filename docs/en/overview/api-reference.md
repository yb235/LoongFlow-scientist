# API Reference

This page documents the public API of the LoongFlow framework. It covers all classes, methods, and interfaces that developers need when building agents or extending the framework.

---

## Framework APIs

### AgentBase

The abstract base class for all agents. Provides lifecycle management, hooks, and error handling.

**Module:** `loongflow.framework.base`

```python
from loongflow.framework.base import AgentBase
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `__call__` | `async def __call__(*args, **kwargs) -> Message` | Invokes the agent. Delegates to `run()` with error handling. |
| `run` | `async def run(*args, **kwargs) -> Message` | **Abstract.** Main agent logic. Must be implemented by subclasses. |
| `interrupt` | `async def interrupt() -> None` | Signals the agent to stop gracefully. |
| `interrupt_impl` | `async def interrupt_impl() -> None` | **Abstract.** Custom interruption logic for subclasses. |
| `register_hook` | `def register_hook(hook_type: str, hook_fn: Callable) -> None` | Registers a pre/post hook on a method. |
| `remove_hook` | `def remove_hook(hook_type: str, hook_fn: Callable) -> None` | Removes a previously registered hook. |
| `handle_error` | `async def handle_error(error: Exception) -> dict` | Default error handler. Override for custom behavior. |
| `is_running` | `@property def is_running() -> bool` | Whether the agent is currently executing. |
| `interrupted` | `@property def interrupted() -> bool` | Whether interruption has been requested. |

---

### PESAgent

The Plan-Execute-Summarize evolutionary agent. Manages concurrent evolution cycles.

**Module:** `loongflow.framework.pes`

```python
from loongflow.framework.pes import PESAgent
```

**Constructor:**

```python
PESAgent(
    config: EvolveChainConfig,
    database: Optional[EvolveDatabase] = None,
    evaluator: Optional[Evaluator] = None,
    finalizer: Optional[Finalizer] = None,
    checkpoint_path: Optional[Path] = None,
)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `config` | `EvolveChainConfig` | Complete configuration for the evolution process |
| `database` | `Optional[EvolveDatabase]` | Custom database instance. If `None`, one is created from config |
| `evaluator` | `Optional[Evaluator]` | Custom evaluator. If `None`, default `LoongFlowEvaluator` is used |
| `finalizer` | `Optional[Finalizer]` | Custom finalizer. If `None`, default `LoongFlowFinalizer` is used |
| `checkpoint_path` | `Optional[Path]` | Path to load a saved checkpoint to resume evolution |

**Key Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_planner_worker` | `def register_planner_worker(name: str, worker_class: type[Worker]) -> None` | Registers a Planner worker class by name |
| `register_executor_worker` | `def register_executor_worker(name: str, worker_class: type[Worker]) -> None` | Registers an Executor worker class by name |
| `register_summary_worker` | `def register_summary_worker(name: str, worker_class: type[Worker]) -> None` | Registers a Summary worker class by name |
| `run` | `async def run() -> Message` | Starts the evolution process. Returns when target score is reached or max iterations completed. |

**Usage Example:**

```python
from loongflow.framework.pes import PESAgent, Worker

agent = PESAgent(config=config, checkpoint_path=checkpoint_path)
agent.register_planner_worker("planner", MyPlannerWorker)
agent.register_executor_worker("executor", MyExecutorWorker)
agent.register_summary_worker("summary", MySummaryWorker)

result = await agent()  # Run the agent
```

---

### Worker

The abstract interface for all PES phase workers (Planners, Executors, Summarizers).

**Module:** `loongflow.framework.pes`

```python
from loongflow.framework.pes import Worker
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `run` | `async def run(context: Any, message: Message \| None) -> Message` | **Abstract.** Executes the worker's phase logic. |

**Parameters:**

- `context` — A `Context` object containing task description, iteration count, workspace paths, island ID, etc.
- `message` — The output from the previous phase (`None` for planners in the first iteration).

---

### ReActAgent

The Reason-Act-Observe loop agent for interactive, tool-using tasks.

**Module:** `loongflow.framework.react`

```python
from loongflow.framework.react import ReActAgent, AgentContext
```

**Constructor:**

```python
ReActAgent(
    context: AgentContext,
    reasoner: Reasoner,
    actor: Actor,
    observer: Observer,
    finalizer: Finalizer,
    name: str = "ReAct",
)
```

**Factory Method:**

```python
@classmethod
def create_default(
    cls,
    model: BaseLLMModel,
    sys_prompt: str,
    output_format: Type[BaseModel] | None = None,
    toolkit: Toolkit | None = None,
    parallel_tool_run: bool = False,
    max_steps: int = 10,
    hint_message: Message = None,
) -> ReActAgent
```

Creates a ReActAgent with standard default components. This is the recommended way to create a ReActAgent.

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | `BaseLLMModel` | The LLM model to use for reasoning |
| `sys_prompt` | `str` | System prompt defining agent behavior |
| `output_format` | `Type[BaseModel] \| None` | Optional structured output format |
| `toolkit` | `Toolkit \| None` | Tools available to the agent |
| `parallel_tool_run` | `bool` | Whether to run tools in parallel |
| `max_steps` | `int` | Maximum reasoning steps before forced summarization |
| `hint_message` | `Message` | Optional hint message added to context |

**Key Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `run` | `async def run(initial_messages: Message \| List[Message], **kwargs) -> Message` | Starts the ReAct loop with initial messages |
| `register_interrupt` | `def register_interrupt(handler: Callable) -> None` | Registers an interrupt handler |

**Usage Example:**

```python
from loongflow.framework.react import ReActAgent
from loongflow.agentsdk.tools import Toolkit, ShellTool, ReadTool
from loongflow.agentsdk.models import LiteLLMModel

# Create model
model = LiteLLMModel(
    model_name="openai/gpt-4",
    base_url="https://api.openai.com/v1",
    api_key="sk-..."
)

# Create toolkit
toolkit = Toolkit()
toolkit.register_tool(ShellTool())
toolkit.register_tool(ReadTool())

# Create agent
agent = ReActAgent.create_default(
    model=model,
    sys_prompt="You are a helpful coding assistant.",
    toolkit=toolkit,
    max_steps=15
)

# Run
result = await agent(Message.from_text("Write a Python hello world script"))
```

---

## Agent SDK APIs

### Message

The universal data container for all inter-component communication.

**Module:** `loongflow.agentsdk.message`

```python
from loongflow.agentsdk.message import Message, Role
```

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | `uuid.UUID` | Auto-generated unique identifier |
| `timestamp` | `datetime` | Creation timestamp |
| `trace_id` | `str` | Execution trace ID for logging |
| `conversation_id` | `str` | Conversation context ID |
| `role` | `Role` | One of: `SYSTEM`, `USER`, `ASSISTANT`, `TOOL` |
| `sender` | `str` | Name of the sending component |
| `metadata` | `Dict[str, Any]` | Flexible metadata dictionary |
| `content` | `List[Element]` | List of content elements |

**Factory Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_text` | `Message.from_text(data: str, sender: str = "", role: Role = Role.USER, mime_type: MimeType = MimeType.TEXT_PLAIN) -> Message` | Create from plain text |
| `from_content` | `Message.from_content(data: Any, mime_type: MimeType, sender: str = "", role: Role = Role.USER) -> Message` | Create from typed content |
| `from_tool_call` | `Message.from_tool_call(target: str, arguments: Dict, sender: str = "", role: Role = Role.ASSISTANT) -> Message` | Create a tool invocation |
| `from_tool_output` | `Message.from_tool_output(call_id: uuid.UUID, tool_name: str, status: ToolStatus, result: List[ContentElement], sender: str) -> Message` | Create a tool result |
| `from_think` | `Message.from_think(content: Any, sender: str = "", role: Role = Role.ASSISTANT) -> Message` | Create an internal thought |
| `from_elements` | `Message.from_elements(elements: List[Element], sender: str = "", role: Role = Role.ASSISTANT) -> Message` | Create from element list |

**Instance Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `get_elements` | `def get_elements(element_cls: Type[ElementT]) -> List[ElementT]` | Filter content by element type |
| `to_dict` | `def to_dict(**kwargs) -> Dict[str, Any]` | Serialize to dictionary |
| `from_dict` | `@classmethod Message.from_dict(data: Dict) -> Message` | Deserialize from dictionary |

---

### Elements

Content elements that Messages carry as payload.

**Module:** `loongflow.agentsdk.message`

```python
from loongflow.agentsdk.message import (
    ContentElement, ToolCallElement, ToolOutputElement,
    ThinkElement, MimeType, ToolStatus
)
```

| Class | Purpose | Key Fields |
|-------|---------|------------|
| `ContentElement` | Text, JSON, or media content | `data: Any`, `mime_type: MimeType` |
| `ToolCallElement` | Tool invocation request | `call_id: uuid.UUID`, `target: str`, `arguments: Dict` |
| `ToolOutputElement` | Tool execution result | `call_id: uuid.UUID`, `tool_name: str`, `status: ToolStatus`, `result: List[ContentElement]` |
| `ThinkElement` | Internal reasoning trace | `content: Any` |
| `EvolveResultElement` | Evolution final result | `content: Any` |

**Enums:**

- `MimeType`: `TEXT_PLAIN`, `APPLICATION_JSON`, `IMAGE_JPEG`, `IMAGE_PNG`, `AUDIO_MPEG`, `VIDEO_MP4`
- `ToolStatus`: `SUCCESS`, `ERROR`, `IN_PROGRESS`
- `Role`: `SYSTEM`, `USER`, `ASSISTANT`, `TOOL`

---

### BaseTool

Abstract base class for all tools.

**Module:** `loongflow.agentsdk.tools`

```python
from loongflow.agentsdk.tools import BaseTool
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `__init__` | `def __init__(*, name: str, description: str) -> None` | Initialize with name and description |
| `get_declaration` | `def get_declaration() -> Optional[FunctionDeclarationDict]` | **Abstract.** Returns JSON Schema for the LLM |
| `arun` | `async def arun(*, args: Dict[str, Any], tool_context: Optional[ToolContext] = None) -> ToolResponse` | **Abstract.** Async tool execution |
| `run` | `def run(*, args: Dict[str, Any], tool_context: Optional[ToolContext] = None) -> ToolResponse` | **Abstract.** Sync tool execution |

---

### FunctionTool

A tool implementation that wraps a Python function.

**Module:** `loongflow.agentsdk.tools`

```python
from loongflow.agentsdk.tools import FunctionTool
```

FunctionTool automatically generates JSON Schema declarations from Python type hints. This is the easiest way to create custom tools.

---

### Toolkit

Manages registration, retrieval, and execution of tools.

**Module:** `loongflow.agentsdk.tools`

```python
from loongflow.agentsdk.tools import Toolkit
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_tool` | `def register_tool(tool: FunctionTool, *, auths: Optional[list] = None) -> None` | Register a tool |
| `unregister_tool` | `def unregister_tool(name: str) -> None` | Remove a tool by name |
| `get` | `def get(name: str) -> Optional[FunctionTool]` | Get tool by name |
| `list_tools` | `def list_tools() -> List[str]` | List all tool names |
| `run` | `def run(name: str, *, args: Dict, tool_context: Optional[ToolContext] = None) -> ToolResponse` | Run a tool synchronously |
| `arun` | `async def arun(name: str, *, args: Dict, tool_context: Optional[ToolContext] = None) -> ToolResponse` | Run a tool asynchronously |
| `get_declarations` | `def get_declarations() -> List[dict]` | Get all tool JSON schemas |

---

### BaseLLMModel

Abstract base class for LLM model wrappers.

**Module:** `loongflow.agentsdk.models`

```python
from loongflow.agentsdk.models import BaseLLMModel
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `__init__` | `def __init__(model_name: str, base_url: str, api_key: str) -> None` | Initialize with model config |
| `generate` | `async def generate(request: CompletionRequest, stream: bool = False) -> AsyncGenerator[CompletionResponse, None]` | **Abstract.** Generate LLM completions |

### LiteLLMModel

Concrete LLM model implementation using the LiteLLM library. Supports 100+ LLM providers.

**Module:** `loongflow.agentsdk.models`

```python
from loongflow.agentsdk.models import LiteLLMModel
```

```python
model = LiteLLMModel(
    model_name="openai/gemini-3-pro-preview",
    base_url="https://generativelanguage.googleapis.com/v1beta/openai",
    api_key="your-api-key"
)
```

---

### EvolveMemory

Abstract base class for evolutionary memory systems.

**Module:** `loongflow.agentsdk.memory`

```python
from loongflow.agentsdk.memory.evolution import EvolveMemory, Solution
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_solution` | `async def add_solution(*args, **kwargs) -> str` | Add a new solution to the population |
| `get_solutions` | `def get_solutions(*args, **kwargs) -> list[Solution]` | Retrieve solutions by IDs |
| `list_solutions` | `def list_solutions(*args, **kwargs) -> list[Solution]` | List solutions by creation time |
| `get_best_solutions` | `def get_best_solutions(*args, **kwargs) -> list[Solution]` | Get top solutions by score |
| `sample` | `def sample(*args, **kwargs) -> Solution` | Sample a parent solution (Boltzmann selection) |
| `save_checkpoint` | `async def save_checkpoint(*args, **kwargs) -> None` | Save memory state to disk |
| `load_checkpoint` | `def load_checkpoint(*args, **kwargs) -> None` | Restore from checkpoint |
| `memory_status` | `def memory_status(*args, **kwargs) -> dict` | Return memory statistics |
| `update_solution` | `async def update_solution(*args, **kwargs) -> None` | Update solution properties |

### Solution

A dataclass representing a single solution in the evolutionary population.

```python
from loongflow.agentsdk.memory.evolution import Solution

solution = Solution(
    id="unique-id",
    code="def solve(): ...",
    score=0.95,
    evaluation={"accuracy": 0.95, "speed": 1.2},
    metadata={"island_id": 0, "iteration": 5}
)
```

---

### GradeMemory

Three-tier memory system for ReAct agents.

**Module:** `loongflow.agentsdk.memory.grade`

```python
from loongflow.agentsdk.memory.grade import GradeMemory
```

| Method | Signature | Description |
|--------|-----------|-------------|
| `add` | `async def add(messages: Message \| List[Message]) -> None` | Add messages to memory |
| `remove` | `async def remove(message_id: uuid.UUID) -> bool` | Remove a message |
| `get_memory` | `async def get_memory() -> List[Message]` | Retrieve conversation context |
| `create_default` | `@classmethod create_default(model: BaseLLMModel) -> GradeMemory` | Factory with defaults |

---

### Logger

Structured logging with trace IDs.

**Module:** `loongflow.agentsdk.logger`

```python
from loongflow.agentsdk.logger import get_logger

logger = get_logger("my_component")
logger.info("Processing iteration %d", iteration_id)
```

---

## Built-in Tools Reference

| Tool Class | Name | Description |
|------------|------|-------------|
| `ShellTool` | `shell` | Execute shell commands and return output |
| `ReadTool` | `read` | Read file contents from the filesystem |
| `WriteTool` | `write` | Write or create files on the filesystem |
| `LsTool` | `ls` | List directory contents |
| `ExecuteCodeTool` | `execute_code` | Execute Python code in a sandbox |
| `TodoReadTool` | `todo_read` | Read structured task/todo lists |
| `TodoWriteTool` | `todo_write` | Create/update structured task lists |
| `AgentTool` | `agent` | Delegate tasks to sub-agents |
| `FunctionTool` | *(custom)* | Wraps any Python function as a tool |
