# Getting Started Guide

Welcome to LoongFlow! This guide is written for **absolute beginners** — developers who have never used the framework before. By the end of this page, you'll understand what LoongFlow is, how it works, and how to use it.

---

## What is LoongFlow?

LoongFlow is an **AI agent framework** that enables automated, evolutionary problem-solving. Think of it as a system that:

1. **Takes a problem** (math optimization, ML competition, general coding)
2. **Generates an initial solution**
3. **Iteratively improves** that solution through a structured Plan→Execute→Summarize loop
4. **Learns from each attempt** to make better decisions over time

Unlike simple "ask AI and hope for the best" approaches, LoongFlow uses a **scientific method**: plan an experiment, execute it, analyze results, and apply lessons learned. This is the **PES (Plan-Execute-Summarize) paradigm**.

---

## Who Is This For?

- **Researchers** working on math optimization or algorithm design
- **Data scientists** competing in Kaggle or similar ML competitions
- **Developers** who want to build custom AI agents that improve over time
- **Anyone** curious about evolutionary AI systems

---

## Prerequisites

Before getting started, you'll need:

- **Python 3.12 or higher** (strict requirement)
- **An LLM API key** (OpenAI, Google Gemini, DeepSeek, or any OpenAI-compatible provider)
- **`uv` package manager** (recommended) or `conda`
- **Git** for cloning the repository

---

## Installation

### Option 1: Using `uv` (Recommended)

```bash
# Install uv if you don't have it
# See: https://docs.astral.sh/uv/getting-started/installation/

# Clone the repository
git clone https://github.com/baidu-baige/LoongFlow.git
cd LoongFlow

# Create a virtual environment
uv venv .venv --python 3.12
source .venv/bin/activate

# Install LoongFlow in development mode
uv pip install -e .
```

### Option 2: Using `conda`

```bash
# Clone the repository
git clone https://github.com/baidu-baige/LoongFlow.git
cd LoongFlow

# Create conda environment
conda create -n loongflow python=3.12
conda activate loongflow

# Install LoongFlow
pip install -e .
```

---

## Core Concepts

Before running anything, let's understand the key concepts:

### The PES Paradigm

Every iteration of a LoongFlow agent follows three phases:

| Phase | What Happens | Analogy |
|-------|-------------|---------|
| **Plan** | Analyze the current best solution, identify weaknesses, design an improvement strategy | A scientist writing a research proposal |
| **Execute** | Generate new code implementing the plan, run it, measure the result | Running the experiment in a lab |
| **Summarize** | Compare new result to previous best, extract lessons learned, update memory | Writing up the research paper |

### Agents

LoongFlow provides three types of agents:

| Agent | Best For | How It Works |
|-------|----------|-------------|
| **Math Agent** | Algorithm optimization, mathematical problems | Uses LiteLLM models to generate Python code, evaluates with a scoring function |
| **ML Agent** | Kaggle competitions, AutoML tasks | Generates a full ML pipeline (data loading → feature engineering → training → prediction) |
| **General Agent** | Any problem with a clear evaluation metric | Uses Claude Code Agent for flexible code generation |

### Evolution Database

LoongFlow keeps a **database of all solutions** ever generated. This enables:

- **Parent selection**: Choose the best previous solution as a starting point
- **Diversity maintenance**: Multiple "islands" prevent getting stuck
- **Experience reuse**: Learn from past successes and failures

### Workers

Each PES phase is handled by a **Worker** — a pluggable component:

- **Planner Worker**: Generates the improvement plan
- **Executor Worker**: Produces the new solution
- **Summary Worker**: Reflects on results

---

## Your First Task: Circle Packing

Let's run the classic "pack circles in a unit square" problem. This is the simplest way to see LoongFlow in action.

### Step 1: Install Task Dependencies

```bash
uv pip install -r ./agents/math_agent/examples/packing_circle_in_unit_square/requirements.txt
```

### Step 2: Configure Your LLM

Edit the configuration file:

```bash
# Open the config file
nano ./agents/math_agent/examples/packing_circle_in_unit_square/task_config.yaml
```

Set your LLM credentials:

```yaml
llm_config:
  url: "https://generativelanguage.googleapis.com/v1beta/openai"
  api_key: "YOUR_API_KEY_HERE"
  model: "openai/gemini-3-pro-preview"
```

!!! tip "Model Format"
    The model name uses `provider/model-name` format. For example:
    
    - `openai/gpt-4o` — OpenAI GPT-4o
    - `openai/gemini-3-pro-preview` — Google Gemini (via OpenAI-compatible API)
    - `deepseek/deepseek-reasoner` — DeepSeek R1

### Step 3: Run the Task

```bash
# Run in background
./run_math.sh packing_circle_in_unit_square --background

# Watch the logs
tail -f ./agents/math_agent/examples/packing_circle_in_unit_square/run.log
```

### Step 4: Understand What's Happening

As the agent runs, you'll see logs showing:

1. **Initial evaluation**: The starting solution is scored
2. **Planning**: The agent analyzes the current best and plans improvements
3. **Execution**: New code is generated and evaluated
4. **Summary**: Results are compared and the database is updated
5. **Repeat**: The cycle continues, with scores generally improving

### Step 5: Stop and View Results

```bash
# Stop the agent
./run_math.sh stop packing_circle_in_unit_square

# Results are in the output directory
ls ./output/
```

### Step 6: Visualize (Optional)

```bash
python agents/math_agent/visualizer/visualizer.py \
    --port 8888 \
    --checkpoint-path output/database/checkpoints
```

Open `http://localhost:8888` in your browser to see the evolution tree, score progression, and code diffs.

---

## Understanding the File Structure

Each task example follows this structure:

```
agents/math_agent/examples/packing_circle_in_unit_square/
├── task_config.yaml      # LLM and evolution configuration
├── initial_program.py    # Starting solution (seed code)
├── eval_program.py       # Scoring function
└── requirements.txt      # Task-specific Python dependencies
```

### task_config.yaml

Controls how the agent behaves:

```yaml
llm_config:
  url: "https://..."          # LLM API endpoint
  api_key: "..."              # API key
  model: "openai/model-name"  # Model identifier

# Optional: Override evolution settings
evolve:
  max_iterations: 50          # Maximum evolution cycles
  target_score: 2.64          # Stop when this score is reached
  concurrency: 3              # Parallel evolution cycles
```

### initial_program.py

The seed solution that the agent starts from. For circle packing, this might be a simple grid layout:

```python
def solve(n):
    """Pack n circles in a unit square, return list of (x, y, r) tuples."""
    positions = []
    r = 0.5 / n
    for i in range(n):
        x = (i + 0.5) / n
        y = 0.5
        positions.append((x, y, r))
    return positions
```

### eval_program.py

The scoring function that measures solution quality:

```python
def evaluate(code_output):
    """Score the packing solution. Higher is better."""
    # Check validity (no overlaps, all within bounds)
    # Calculate total area covered
    # Return score
    return score
```

---

## Common Configuration Options

### LLM Configuration

```yaml
llm_config:
  url: "https://api.openai.com/v1"      # API endpoint
  api_key: "sk-..."                       # Your API key
  model: "openai/gpt-4o"                 # Model to use
```

**Supported providers:**

| Provider | URL | Model Format |
|----------|-----|-------------|
| OpenAI | `https://api.openai.com/v1` | `openai/gpt-4o` |
| Google Gemini | `https://generativelanguage.googleapis.com/v1beta/openai` | `openai/gemini-3-pro-preview` |
| DeepSeek | `https://api.deepseek.com/v1` | `deepseek/deepseek-reasoner` |
| Local (vLLM) | `http://localhost:8000/v1` | `openai/your-model` |

### Evolution Settings

```yaml
evolve:
  max_iterations: 100         # Maximum number of PES cycles
  target_score: 0.95          # Target score to reach (agent stops here)
  concurrency: 5              # Number of parallel evolution cycles
  database:
    num_islands: 3            # Number of solution islands (diversity)
    population_size: 50       # Max solutions per island
    checkpoint_interval: 10   # Save checkpoint every N iterations
```

---

## Glossary

| Term | Definition |
|------|-----------|
| **PES** | Plan-Execute-Summarize — the core thinking paradigm |
| **Worker** | A pluggable component that handles one PES phase |
| **Evolution Cycle** | One complete Plan→Execute→Summarize iteration |
| **Island** | A sub-population of solutions that evolve semi-independently |
| **Solution** | A generated code artifact with its score and metadata |
| **Checkpoint** | A saved snapshot of the evolution database |
| **Boltzmann Selection** | A probabilistic method for choosing parent solutions |
| **MAP-Elites** | A technique for maintaining diverse solutions across feature dimensions |
| **EvoCoder** | The ML Agent's code generation system with generate-evaluate-retry loops |
| **Solution Pack** | A multi-file solution directory used by the General Agent |
| **ReActAgent** | The Reason-Act-Observe loop agent used internally by planners and summarizers |

---

## Troubleshooting

### "Module not found" errors

Make sure you've installed in development mode and set the PYTHONPATH:

```bash
uv pip install -e .
export PYTHONPATH=$PYTHONPATH:./src
```

### LLM API errors

- Verify your API key is correct in `task_config.yaml`
- Check that the URL matches your provider
- Ensure the model name uses the `provider/model-name` format
- Check your API quota and rate limits

### Agent stops too quickly

- Lower the `target_score` in config (maybe the initial solution already meets it)
- Increase `max_iterations` to allow more evolution cycles
- Check the evaluation function for bugs

### Low scores / no improvement

- Try a better LLM model (Gemini Pro or GPT-4o recommended)
- Increase `concurrency` for more parallel exploration
- Increase `num_islands` for more diversity
- Check that your initial solution is valid

### Process won't stop

Use the stop command for your agent type:

```bash
./run_math.sh stop task_name
./run_ml.sh stop task_name
./run_general.sh stop task_name
```

---

## Next Steps

Now that you understand the basics:

1. **[Architecture Deep Dive](architecture.md)** — Understand the internal code structure
2. **[API Reference](api-reference.md)** — Detailed reference for all public APIs
3. **[End-to-End Workflows](workflow.md)** — See complete data flow diagrams
4. **[Quick Start](quickstart.md)** — More examples and advanced configuration
5. **[Build Your Own Agent](../build_agent/get_started.md)** — Create custom agents

---

## FAQ for Beginners

**Q: How much does it cost to run?**

For a problem like circle packing using Gemini 3 Pro, the total cost is approximately $10. Costs vary by model and iteration count.

**Q: Can I use a local LLM?**

Yes! Any OpenAI-compatible API works. Deploy your model with vLLM or SGLang, then point the config to `http://localhost:8000/v1`.

**Q: Do I need a GPU?**

Not for math tasks. For ML tasks with deep learning models, a GPU is recommended. The ML agent auto-detects GPU availability.

**Q: Can I resume a stopped task?**

Yes. The agent saves checkpoints automatically. Use the `--checkpoint-path` CLI argument to resume from a saved state.

**Q: What problems can LoongFlow solve?**

Any problem where you can:

1. Write an initial solution in code
2. Define a scoring function that evaluates solution quality
3. Express the problem clearly in natural language

The agent will iteratively improve the solution using AI-powered code generation and reflection.
