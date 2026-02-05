# End-to-End Workflows

This page explains **how LoongFlow works from start to finish**, covering the complete data flow for each agent type. If you're a first-time user, this is the best place to understand what happens when you run a LoongFlow agent.

---

## Overview: The PES Loop

Every LoongFlow evolutionary agent follows the same high-level loop:

```
┌─────────────────────────────────────────────────────────────────┐
│                     PESAgent.run()                              │
│                                                                 │
│   1. Load/evaluate initial solution                             │
│   2. Check if initial score meets target → early exit           │
│   3. Start concurrent evolution cycles                          │
│                                                                 │
│   ┌─────────────────────────────────────┐                       │
│   │        Evolution Cycle              │ ← runs N in parallel  │
│   │                                     │                       │
│   │   ┌──────────┐                      │                       │
│   │   │  PLAN    │ Sample parent,       │                       │
│   │   │          │ generate strategy    │                       │
│   │   └────┬─────┘                      │                       │
│   │        │ Message (plan)             │                       │
│   │   ┌────▼─────┐                      │                       │
│   │   │ EXECUTE  │ Generate solution,   │                       │
│   │   │          │ evaluate score       │                       │
│   │   └────┬─────┘                      │                       │
│   │        │ Message (solution+score)   │                       │
│   │   ┌────▼─────┐                      │                       │
│   │   │ SUMMARY  │ Reflect, record     │                       │
│   │   │          │ to database          │                       │
│   │   └──────────┘                      │                       │
│   └─────────────────────────────────────┘                       │
│                                                                 │
│   4. Repeat until target score reached or max iterations        │
│   5. Generate final report via Finalizer                        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Workflow 1: Math Agent

The Math Agent solves mathematical optimization and algorithm design problems. It's the simplest way to understand the PES loop.

### What Happens When You Run `./run_math.sh packing_circle_in_unit_square`

**Step 1: Shell Script Setup**

```bash
# run_math.sh does the following:
export PYTHONPATH=$PYTHONPATH:./src
python agents/math_agent/math_evolve_agent.py \
    --config agents/math_agent/examples/packing_circle_in_unit_square/task_config.yaml \
    --eval-file agents/math_agent/examples/packing_circle_in_unit_square/eval_program.py \
    --initial-file agents/math_agent/examples/packing_circle_in_unit_square/initial_program.py
```

**Step 2: Agent Initialization** (`math_evolve_agent.py`)

1. Parse CLI arguments and load `task_config.yaml`
2. Read `initial_program.py` as the seed solution
3. Create `EvolveChainConfig` with all settings
4. Instantiate `PESAgent` with the config
5. Register workers:
   - `EvolvePlanAgent` as the planner
   - `EvolveExecuteAgentChat`, `EvolveExecuteAgentReact`, `EvolveExecuteAgentFuse` as executors
   - `EvolveSummaryAgent` as the summarizer
6. Call `await agent()` to start

**Step 3: Initial Evaluation**

- The `initial_program.py` is evaluated using `eval_program.py`
- If the initial score meets the target, the agent exits immediately
- Otherwise, the initial solution is stored in the evolution database

**Step 4: Evolution Cycles** (repeated concurrently)

```
Iteration 1:
├── PLAN (EvolvePlanAgent)
│   ├── Create ReActAgent with database query tools
│   ├── Sample parent solution from database
│   ├── Analyze parent's strengths/weaknesses
│   ├── Generate 3 candidate improvement plans
│   └── Select the most robust plan → write to best_plan.md
│
├── EXECUTE (EvolveExecuteAgentChat/React/Fuse)
│   ├── Parse plan from planner's message
│   ├── Generate new Python code based on plan
│   ├── Run eval_program.py on new code → get score
│   └── Return code + score
│
└── SUMMARY (EvolveSummaryAgent)
    ├── Compare child score vs parent score
    ├── Classify: IMPROVEMENT / REGRESSION / STALE
    ├── Generate reflection using ReActAgent
    ├── Calculate weight: parent_weight + (3 * score_diff * step_size) + 3 * child_score
    └── Store new solution in database
```

**Step 5: Termination**

When target score is reached or max iterations completed:

1. All running cycles are gracefully cancelled
2. The Finalizer generates a report with the best solution
3. Final checkpoint is saved
4. Agent returns a `Message` with `EvolveResultElement`

### Data Flow Diagram (Math Agent)

```
task_config.yaml ─┐
eval_program.py ──┤
initial_program.py┘
        │
        ▼
  MathPESAgent
        │
        ▼
  ┌── Planner ──────────────────────────────┐
  │  Input: Context (task, parent solution)  │
  │  Tools: GetMemoryStatus, GetSolutions,   │
  │         GetBestSolutions                 │
  │  Output: Message(plan.md path)           │
  └──────────────┬──────────────────────────┘
                 │
  ┌── Executor ──▼──────────────────────────┐
  │  Input: Message(plan + parent code)      │
  │  Action: Generate new code via LLM       │
  │  Evaluate: Run eval_program.py           │
  │  Output: Message(code path + score)      │
  └──────────────┬──────────────────────────┘
                 │
  ┌── Summary ───▼──────────────────────────┐
  │  Input: Message(code + score + parent)   │
  │  Action: Assess, reflect, calculate wt   │
  │  Output: Solution stored in database     │
  └─────────────────────────────────────────┘
```

---

## Workflow 2: ML Agent

The ML Agent automates machine learning competitions (e.g., Kaggle). It uses a more complex pipeline with the **EvoCoder** code generation system.

### What Happens When You Run `./run_ml.sh run ml_example`

**Step 1: Environment Setup**

```bash
# run_ml.sh does the following:
# 1. Activate conda environment "loongflow_ml"
# 2. Set PYTHONPATH
# 3. Run:
python3 -u agents/ml_agent/ml_evolve_agent.py \
    --config task_config.yaml \
    --task-data-path ./public \
    --task-file ./public/description.md \
    --eval-file ./eval_program.py
```

**Step 2: Agent Initialization** (`ml_evolve_agent.py`)

1. Load task configuration and description
2. Collect hardware metadata (GPU availability)
3. Scan task directory for data structure
4. Create `PESAgent` with ML-specific workers
5. Register: `MLPlannerAgent`, `MLExecutorAgent`, `MLSummaryAgent`

**Step 3: Evolution Cycle — Planning Phase**

The ML Planner uses a ReActAgent with three specialized tools:

| Tool | Purpose |
|------|---------|
| `eda_tool` | Run Exploratory Data Analysis via EvoCoder |
| `strategic_analysis_tool` | Analyze top solutions in database |
| `ensemble_tool` | Suggest model ensemble strategies |

The planner autonomously decides whether to run EDA, then generates an ML-specific plan covering:

- Feature engineering strategies
- Model selection and hyperparameter ranges
- Ensemble approaches
- Cross-validation strategy

**Step 4: Evolution Cycle — Execution Phase**

The ML Executor generates code for a **multi-stage pipeline**:

```
Stage 1: LOAD_DATA
   └── Generate data loading code via EvoCoder

Stage 2: CROSS_VALIDATION
   └── Generate CV setup code

Stage 3: CREATE_FEATURES
   └── Generate feature engineering code

Stage 4: TRAIN_AND_PREDICT
   └── Generate model training code

Stage 5: ENSEMBLE
   └── Generate ensemble code

Stage 6: WORKFLOW
   └── Generate final pipeline that chains all stages
```

Each stage uses **EvoCoder** — a generate-evaluate-retry loop:

```
for round in range(max_rounds):
    1. Send context to LLM (task, stage description, previous errors)
    2. Extract generated code from response
    3. Validate code (syntax check, import check, execution test)
    4. If SUCCESS → save code and move to next stage
    5. If ERROR → append error message to context, retry
```

**Step 5: Evaluation**

The `MLEvaluator` runs the complete pipeline in a subprocess:

1. Dynamically loads the generated code module
2. Executes the full ML pipeline
3. Calls the user-provided `eval_program.py`
4. Returns score, metrics, and prediction statistics

**Step 6: Summary**

Similar to Math Agent — assess, reflect, and record in database.

---

## Workflow 3: General Agent

The General Agent is the most flexible, using Claude Code Agent for both planning and execution.

### What Happens When You Run `./run_general.sh hello_world`

**Step 1: Shell Script Setup**

```bash
python agents/general_agent/general_evolve_agent.py \
    --config agents/general_agent/examples/hello_world/task_config.yaml
```

**Step 2: Solution Pack Protocol**

The General Agent uses **solution packs** — directories containing multi-file solutions:

```
solution_pack/
├── index.json          # Manifest with file inventory
├── main.py             # Primary code file
├── utils.py            # Supporting modules
├── config.yaml         # Configuration
└── README.md           # Documentation
```

The `ManifestGenerator` uses an LLM to analyze the directory and create `index.json`:

```json
{
  "version": "1.0",
  "entrypoint": "main.py",
  "description": "Solution for the optimization problem",
  "files": [
    {"path": "main.py", "type": "code", "description": "Main solver"},
    {"path": "utils.py", "type": "code", "description": "Utility functions"}
  ],
  "metadata": {}
}
```

**Step 3: Planning with Claude Code Agent**

The planner:

1. Samples a parent solution from the database
2. Loads the solution pack context (file tree + manifest)
3. Creates a Claude Code Agent with database query tools
4. The Claude agent analyzes the parent and writes an improvement plan

**Step 4: Execution with Claude Code Agent**

The executor:

1. Clones the parent solution files (copy-on-write)
2. Creates a Claude Code Agent with file read/write tools
3. The Claude agent executes the plan, modifying code files
4. Ensures a manifest (`index.json`) is generated
5. Evaluates the new solution

**Step 5: Two Evaluation Modes**

| Mode | How It Works |
|------|-------------|
| **Self-Evaluation** | Agent evaluates its own output via system prompt criteria |
| **Custom Script** | User provides `eval_program.py` that runs in subprocess |

---

## Key Concepts for First-Time Users

### 1. Everything is a Message

All communication between components uses `Message` objects. When the Planner produces a plan, it's a `Message`. When the Executor returns results, it's a `Message`. This provides a uniform interface.

### 2. Workers are Pluggable

You can swap out any Planner, Executor, or Summarizer by implementing the `Worker` interface:

```python
class Worker(ABC):
    async def run(self, context, message) -> Message:
        ...
```

### 3. Evolution Database Tracks Everything

The database stores every solution ever generated, with:

- Score and evaluation metrics
- Parent-child relationships (evolutionary tree)
- Island assignments (for diversity)
- Sample weights (for selection probability)

### 4. Checkpointing for Resilience

The PESAgent automatically saves checkpoints at configurable intervals. If the process crashes, you can resume from the last checkpoint:

```python
agent = PESAgent(config=config, checkpoint_path="output/database/checkpoints/iter_50.json")
```

### 5. Multi-Island Evolution

Solutions are distributed across multiple "islands" to prevent premature convergence:

- Each island maintains its own population
- Solutions occasionally migrate between islands
- Different islands may explore different strategies

### 6. Adaptive Boltzmann Selection

When sampling parent solutions, LoongFlow uses Boltzmann selection:

- Higher-scoring solutions are more likely to be selected
- Temperature parameter controls exploration vs. exploitation
- Prevents the population from getting stuck on local optima

---

## Running Your First Task: Step by Step

### Prerequisites

```bash
# 1. Clone the repository
git clone https://github.com/baidu-baige/LoongFlow.git
cd LoongFlow

# 2. Set up Python environment
uv venv .venv --python 3.12
source .venv/bin/activate
uv pip install -e .
```

### Running a Math Task

```bash
# 3. Install task dependencies
uv pip install -r ./agents/math_agent/examples/packing_circle_in_unit_square/requirements.txt

# 4. Configure your LLM (edit the config file)
# Set url, api_key, and model in:
#   agents/math_agent/examples/packing_circle_in_unit_square/task_config.yaml

# 5. Run the task
./run_math.sh packing_circle_in_unit_square --background

# 6. Monitor progress
tail -f ./agents/math_agent/examples/packing_circle_in_unit_square/run.log

# 7. Stop when satisfied
./run_math.sh stop packing_circle_in_unit_square
```

### Running an ML Task

```bash
# 3. Initialize ML environment (requires conda/mamba)
./run_ml.sh init

# 4. Configure your LLM
# Edit: agents/ml_agent/examples/ml_example/task_config.yaml

# 5. Run the task
./run_ml.sh run ml_example --background

# 6. Monitor progress
tail -f ./agents/ml_agent/examples/ml_example/agent.log

# 7. Stop when satisfied
./run_ml.sh stop ml_example
```

### Understanding the Output

After running, you'll find results in the `output/` directory:

```
output/
├── database/
│   ├── checkpoints/         # Periodic snapshots
│   │   ├── iter_10.json
│   │   └── iter_20.json
│   └── solutions/           # Generated code files
├── planner/                 # Plans from each iteration
├── executor/                # Execution artifacts
└── summarizer/              # Reflection summaries
```

### Visualization

For math tasks, you can visualize the evolution:

```bash
python agents/math_agent/visualizer/visualizer.py \
    --port 8888 \
    --checkpoint-path output/database/checkpoints
```

This launches a web interface at `http://localhost:8888` showing:

- 🌳 Evolution tree with parent-child relationships
- 📈 Score progression across generations
- 🔍 Code diff viewer
- 📊 Island distribution map
