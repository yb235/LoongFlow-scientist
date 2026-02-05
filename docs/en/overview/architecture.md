# Architecture Deep Dive

This page provides a comprehensive, detailed guide to LoongFlow's internal architecture. It is written for first-time users, contributors, and developers who need to understand **how the code is organized**, **how the components relate to each other**, and **how data flows** through the system.

---

## Repository Structure

```
LoongFlow/
├── src/loongflow/               # Core framework (installable package)
│   ├── agentsdk/                # Low-level building blocks
│   │   ├── logger/              # Structured logging utilities
│   │   ├── memory/              # Memory systems (Evolution + Grade)
│   │   ├── message/             # Message & Element abstractions
│   │   ├── models/              # LLM model wrappers
│   │   ├── token/               # Token counting
│   │   └── tools/               # Tool framework (BaseTool, Toolkit)
│   └── framework/               # High-level agent frameworks
│       ├── base/                # AgentBase abstract class
│       ├── pes/                 # PESAgent (Plan-Execute-Summarize)
│       ├── react/               # ReActAgent (Reason-Act-Observe)
│       └── claude_code/         # Claude Code Agent integration
├── agents/                      # Domain-specific agent implementations
│   ├── general_agent/           # General-purpose evolutionary agent
│   ├── math_agent/              # Math/algorithm optimization agent
│   └── ml_agent/                # Machine learning competition agent
├── tests/                       # Unit and integration tests
├── docs/                        # Documentation (this site)
├── run_general.sh               # Shell launcher for General Agent
├── run_math.sh                  # Shell launcher for Math Agent
├── run_ml.sh                    # Shell launcher for ML Agent
└── pyproject.toml               # Python project configuration
```

---

## Layered Architecture

LoongFlow follows a **three-layer architecture**:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent Implementations                       │
│         (general_agent, math_agent, ml_agent)                   │
│   Domain-specific planners, executors, summarizers, evaluators  │
├─────────────────────────────────────────────────────────────────┤
│                     Framework Layer                              │
│              (PESAgent, ReActAgent, AgentBase)                   │
│   Orchestration, lifecycle, concurrency, worker registration    │
├─────────────────────────────────────────────────────────────────┤
│                     Agent SDK                                   │
│      (Message, Tools, Models, Memory, Logger, Token)            │
│   Foundational building blocks shared by all agents             │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 1: Agent SDK (`src/loongflow/agentsdk/`)

The SDK provides the **foundational primitives** that all agents use:

| Module | Purpose |
|--------|---------|
| **message/** | Unified `Message` and `Element` abstractions for all data exchange |
| **tools/** | `BaseTool`, `FunctionTool`, `Toolkit` — the tool registration and execution framework |
| **models/** | `BaseLLMModel`, `LiteLLMModel` — LLM provider abstraction supporting OpenAI, Gemini, DeepSeek, etc. |
| **memory/** | Two memory systems: `EvolveMemory` (for PES agents) and `GradeMemory` (for ReAct agents) |
| **logger/** | Structured logging with trace IDs for debugging |
| **token/** | Token counting for cost tracking |

### Layer 2: Framework (`src/loongflow/framework/`)

The framework layer provides **agent orchestration patterns**:

| Module | Purpose |
|--------|---------|
| **base/** | `AgentBase` — abstract async agent with hooks, lifecycle, and error handling |
| **pes/** | `PESAgent` — the core Plan-Execute-Summarize evolutionary agent with concurrent cycles |
| **react/** | `ReActAgent` — the Reason-Act-Observe loop agent for interactive tool-using tasks |
| **claude_code/** | `ClaudeCodeAgent` — integration with Anthropic's Claude for code generation |

### Layer 3: Agent Implementations (`agents/`)

Concrete domain-specific agents that implement the framework's abstract interfaces:

| Agent | Use Case | Framework |
|-------|----------|-----------|
| **general_agent/** | General-purpose problem solving | PESAgent + Claude Code |
| **math_agent/** | Mathematical optimization, algorithm design | PESAgent + LiteLLM |
| **ml_agent/** | ML competitions (Kaggle), AutoML | PESAgent + EvoCoder |

---

## Class Hierarchy

```
                    AgentBase (ABC)
                    ├── PESAgent
                    └── ReactAgentBase
                        └── ReActAgent

                    Worker (ABC)
                    ├── Planner workers
                    ├── Executor workers
                    └── Summary workers

                    BaseLLMModel (ABC)
                    └── LiteLLMModel

                    EvolveMemory (ABC)
                    ├── InMemoryEvolveMemory
                    ├── RedisEvolveMemory
                    └── BoltzmannMemory

                    BaseTool (ABC)
                    ├── FunctionTool
                    ├── ShellTool
                    ├── ReadTool
                    ├── WriteTool
                    ├── LsTool
                    ├── ExecuteCodeTool
                    ├── TodoReadTool
                    ├── TodoWriteTool
                    └── AgentTool

                    BaseElement (ABC)
                    ├── ContentElement
                    ├── ToolCallElement
                    ├── ToolOutputElement
                    ├── ThinkElement
                    └── EvolveResultElement
```

---

## The PES Agent in Detail

The **PESAgent** is the centerpiece of LoongFlow. It orchestrates an evolutionary loop where each iteration follows three phases:

### Phase 1: Plan

The **Planner** worker receives the current context (task description, database of past solutions) and produces a strategic plan. It:

- Samples a parent solution from the evolution database
- Analyzes the parent's strengths and weaknesses
- Retrieves relevant past experience from memory
- Generates a detailed improvement blueprint

### Phase 2: Execute

The **Executor** worker takes the plan and produces a new candidate solution. It:

- Parses the plan from the planner's output message
- Clones the parent solution (copy-on-write pattern)
- Generates or modifies code based on the plan
- Evaluates the new solution using the evaluator
- Returns the solution path and evaluation results

### Phase 3: Summarize

The **Summary** worker reflects on the execution results. It:

- Gathers evidence: plan, solution, evaluation, parent comparison
- Assesses the outcome: IMPROVEMENT, REGRESSION, or STALE
- Extracts reusable insights
- Records the new solution in the database with a calculated weight

### Concurrency Model

The PESAgent runs **multiple evolution cycles concurrently**:

```python
# Simplified view of PESAgent.run()
while not stop_event.is_set():
    # Start up to max_workers concurrent cycles
    while len(running_tasks) < max_workers:
        task = asyncio.create_task(self._evolution_cycle(iteration_id))
        running_tasks.add(task)

    # Wait for any cycle to complete
    done, pending = await asyncio.wait(running_tasks, return_when=FIRST_COMPLETED)

    for task in done:
        # Check if target score reached
        if target_score_reached:
            stop_event.set()
        # Save checkpoints periodically
        await self._handle_cycle_completion_and_checkpoint(iteration_id)
```

---

## The ReAct Agent in Detail

The **ReActAgent** implements a Reason-Act-Observe loop for interactive tasks:

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Reason  │────▶│   Act    │────▶│ Finalize │────▶│ Observe  │
│ (LLM)   │     │ (Tools)  │     │ (Check)  │     │ (Update) │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     ▲                                                    │
     └────────────────────────────────────────────────────┘
                    (loop up to max_steps)
```

### Components

The ReActAgent uses four pluggable components (defined as Protocols):

| Component | Role |
|-----------|------|
| **Reasoner** | Calls the LLM to generate thoughts and tool calls |
| **Actor** | Executes tool calls (parallel or sequential) |
| **Observer** | Processes tool outputs and updates context |
| **Finalizer** | Detects completion conditions, summarizes on max steps exceeded |

---

## Memory Systems

### Evolution Memory (for PES Agents)

Used by PESAgent to store and retrieve solutions across iterations:

- **InMemoryEvolveMemory**: Fast, in-process storage
- **RedisEvolveMemory**: Distributed storage for multi-process setups
- **BoltzmannMemory**: Temperature-based probabilistic selection

Key features:

- **Multi-Island**: Solutions are organized into "islands" to preserve diversity
- **MAP-Elites**: Feature-based binning to maintain diverse solution niches
- **Adaptive Boltzmann Selection**: Balances exploration vs. exploitation
- **Checkpointing**: Persistent snapshots for fault tolerance

### Grade Memory (for ReAct Agents)

A three-tier memory system for managing conversation history:

| Tier | Description | Purpose |
|------|-------------|---------|
| **STM** (Short-Term) | Recent messages | Immediate context |
| **MTM** (Medium-Term) | Compressed summaries | Conversation continuity |
| **LTM** (Long-Term) | Knowledge base | Cross-session learning |

---

## Message Protocol

All inter-component communication uses the unified **Message** class:

```python
class Message(BaseModel):
    id: uuid.UUID                    # Unique identifier
    timestamp: datetime              # Creation time
    trace_id: str                    # Execution trace ID
    conversation_id: str             # Conversation context
    role: Role                       # SYSTEM, USER, ASSISTANT, TOOL
    sender: str                      # Component identifier
    metadata: Dict[str, Any]         # Flexible metadata
    content: List[Element]           # Payload (ContentElement, ToolCallElement, etc.)
```

Messages carry different types of `Element` payloads:

| Element | Use Case |
|---------|----------|
| `ContentElement` | Text, JSON, images |
| `ToolCallElement` | Tool invocation requests |
| `ToolOutputElement` | Tool execution results |
| `ThinkElement` | Internal reasoning traces |
| `EvolveResultElement` | Evolution final results |

---

## Tool System

Tools are registered with a `Toolkit` and made available to agents:

```python
# Define a tool
class MyTool(BaseTool):
    def __init__(self):
        super().__init__(name="my_tool", description="Does something useful")

    def get_declaration(self) -> FunctionDeclarationDict:
        return {"name": self.name, "description": self.description, "parameters": {...}}

    async def arun(self, *, args, tool_context=None) -> ToolResponse:
        # Tool logic here
        return ToolResponse(...)

# Register and use
toolkit = Toolkit()
toolkit.register_tool(MyTool())
```

Built-in tools include:

| Tool | Purpose |
|------|---------|
| `ShellTool` | Execute shell commands |
| `ReadTool` | Read file contents |
| `WriteTool` | Write/create files |
| `LsTool` | List directory contents |
| `ExecuteCodeTool` | Execute Python code safely |
| `TodoReadTool` | Read structured task lists |
| `TodoWriteTool` | Update structured task lists |
| `AgentTool` | Delegate to sub-agents |

---

## Configuration System

LoongFlow uses YAML-based configuration with Pydantic validation:

```
EvolveChainConfig (Root)
├── workspace_path: str
├── logger: LoggerConfig
├── llm_config: LLMConfig              # Global fallback
├── planners: Dict[str, config]         # Planner configurations
├── executors: Dict[str, config]        # Executor configurations
├── summarizers: Dict[str, config]      # Summarizer configurations
└── evolve: EvolveConfig
    ├── task: str                       # Task description
    ├── initial_code: str               # Seed solution
    ├── initial_score: float            # Baseline score
    ├── max_iterations: int             # Iteration limit
    ├── target_score: float             # Goal score
    ├── concurrency: int                # Max parallel workers
    ├── database: DatabaseConfig
    │   ├── num_islands: int
    │   ├── population_size: int
    │   └── checkpoint_interval: int
    └── evaluator: EvaluatorConfig
        ├── evaluate_code: str
        ├── workspace_path: str
        └── timeout: int
```

Each task provides a `task_config.yaml` with at minimum:

```yaml
llm_config:
  url: "https://api.example.com/v1"
  api_key: "your-api-key"
  model: "openai/gemini-3-pro-preview"
```

---

## Evaluation System

Each agent type has its own evaluator that runs in a **subprocess** for safety:

| Agent | Evaluator | How It Works |
|-------|-----------|--------------|
| **General** | `GeneralEvaluator` | Self-evaluation via LLM or custom evaluation script |
| **Math** | `LoongFlowEvaluator` | Runs `eval_program.py` in subprocess, returns numeric score |
| **ML** | `MLEvaluator` | Executes ML pipeline, scores predictions against ground truth |

The subprocess isolation ensures that:

- Buggy generated code cannot crash the agent
- Timeouts prevent infinite loops
- Resource limits protect the host system

---

## Worker Registration

Workers (Planners, Executors, Summarizers) are registered using a global registry:

```python
from loongflow.framework.pes import PESAgent, Worker

class MyPlanner(Worker):
    async def run(self, context, message):
        # Planning logic
        return Message.from_text("plan details")

class MyExecutor(Worker):
    async def run(self, context, message):
        # Execution logic
        return Message.from_text("solution")

class MySummary(Worker):
    async def run(self, context, message):
        # Summary logic
        return Message.from_text("reflection")

# Register workers with PESAgent
agent = PESAgent(config=config)
agent.register_planner_worker("my_planner", MyPlanner)
agent.register_executor_worker("my_executor", MyExecutor)
agent.register_summary_worker("my_summary", MySummary)
```

---

## LLM Integration

LoongFlow supports multiple LLM providers through `LiteLLMModel`:

```python
from loongflow.agentsdk.models import LiteLLMModel

model = LiteLLMModel(
    model_name="openai/gpt-4",
    base_url="https://api.openai.com/v1",
    api_key="sk-..."
)

# Generate completions
async for response in model.generate(request):
    print(response.content)
```

Supported providers include:

- **OpenAI** (GPT-4, GPT-4o, etc.)
- **Google** (Gemini Pro, Gemini Flash)
- **DeepSeek** (DeepSeek-R1)
- **Anthropic** (Claude, via Claude Agent SDK)
- Any **OpenAI-compatible** API (vLLM, SGLang, etc.)

The model name uses the format `provider/model-name` (e.g., `openai/gemini-3-pro-preview`).
