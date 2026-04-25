---
name: promptic
description: Integrate the Promptic Python SDK for LLM observability, tracing, prompt optimization, and agent evaluation. Use when code imports `promptic_sdk`, user asks to add Promptic tracing, optimize prompts, evaluate agents, deploy prompts, or integrate with the Promptic platform. Also use when setting up OpenTelemetry-based LLM tracing, creating evaluation datasets, or managing AI components programmatically.
---

# Promptic Python SDK

SDK and CLI for the [Promptic](https://promptic.eu) platform — LLM tracing, prompt optimization, and agent evaluation.

## Installation

```bash
pip install promptic-sdk
```

Install extras for auto-instrumentation:

```bash
# LLM providers
pip install promptic-sdk[openai]         # OpenAI
pip install promptic-sdk[anthropic]      # Anthropic
pip install promptic-sdk[bedrock]        # AWS Bedrock
pip install promptic-sdk[vertexai]       # Google Vertex AI
pip install promptic-sdk[mistralai]      # Mistral

# Agent frameworks
pip install promptic-sdk[langchain]      # LangChain / LangGraph / create_agent / deepagents
pip install promptic-sdk[openai-agents]  # OpenAI Agents SDK
pip install promptic-sdk[claude-agent]   # Claude Agent SDK

pip install promptic-sdk[all]            # Everything above
```

Pydantic AI ships its own OpenTelemetry emitter — enable with
`Agent(..., instrument=True)`, no extras needed.

## Authentication

```bash
# Browser login (local dev)
promptic login

# CI/CD
export PROMPTIC_API_KEY="pk_..."
```

Config resolution: explicit args > env vars (`PROMPTIC_API_KEY`, `PROMPTIC_ENDPOINT`) > `~/.promptic/config.toml`.

## Tracing

Call `promptic_sdk.init()` once at startup. All LLM calls from installed providers are auto-instrumented via OpenTelemetry.

```python
import promptic_sdk
from openai import OpenAI

promptic_sdk.init()
client = OpenAI()

with promptic_sdk.ai_component("my-agent"):
    response = client.chat.completions.create(
        model="gpt-4.1-nano",
        messages=[{"role": "user", "content": "Hello!"}],
    )
```

### init() parameters

| Parameter          | Description                                         | Default                  |
|--------------------|-----------------------------------------------------|--------------------------|
| `api_key`          | Promptic API key (falls back to `PROMPTIC_API_KEY`) | —                        |
| `endpoint`         | Platform URL (falls back to `PROMPTIC_ENDPOINT`)    | `https://promptic.eu`    |
| `auto_instrument`  | Auto-detect and instrument LLM client libraries     | `True`                   |
| `service_name`     | OpenTelemetry `service.name` resource attribute      | —                        |

Auto-detected instrumentors: OpenAI, Anthropic, Google Generative AI, Vertex AI,
Bedrock, Mistral, Cohere, LangChain (with LangGraph / `create_agent` / deepagents),
OpenAI Agents SDK, Claude Agent SDK. All emit the official OpenTelemetry GenAI
semantic conventions (`gen_ai.*`).

### ai_component context manager

Tag spans with an AI Component name. The platform links traces to the matching component.

```python
with promptic_sdk.ai_component("customer-support-agent"):
    # All LLM calls here are tagged
    ...

# With dataset and run tagging for evaluation:
with promptic_sdk.ai_component("my-agent", dataset="eval-set", run="v1-baseline"):
    agent.run(test_input)
```

Parameters:
- `name` (str): AI Component name in the workspace
- `dataset` (str, optional): Dataset name — traces auto-added to this dataset (created if needed)
- `run` (str, optional): Run name — groups traces within a dataset for comparison

### dataset context manager

Tag spans with a dataset name independently:

```python
with promptic_sdk.ai_component("my-agent"):
    with promptic_sdk.dataset("eval-round-1"):
        agent.run(test_input)
```

### Custom OpenTelemetry instrumentors

Since Promptic uses standard OpenTelemetry, add any OTel-compatible instrumentor:

```python
import promptic_sdk
from opentelemetry.instrumentation.requests import RequestsInstrumentor

promptic_sdk.init()
RequestsInstrumentor().instrument()  # Spans exported to Promptic
```

### LangGraph / deepagents integration

`pip install promptic-sdk[langchain]` installs OpenLLMetry's
`opentelemetry-instrumentation-langchain` (≥0.60), which covers LangChain
chains, LangGraph (`create_agent`), and deepagents with subagents. Emits the
official OpenTelemetry GenAI semantic conventions (`gen_ai.tool.definitions`,
`gen_ai.operation.name`, `gen_ai.usage.*`), so agent-evaluation insights
(loops, tool errors, unused tools) work for flat agents and multi-agent
graphs uniformly.

Users who prefer the LangSmith OTel bridge (e.g. for hybrid dual-export to
LangSmith) can opt in by setting `LANGSMITH_TRACING=true` and
`LANGSMITH_OTEL_ENABLED=true` before calling `init()`. Note: the LangSmith
bridge does not emit tool definitions, so the "unused tools" insight will
not fire on LangSmith-bridged traces.

## API Client

Both sync (`PrompticClient`) and async (`AsyncPrompticClient`) clients with identical method signatures.

```python
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    traces = client.list_traces(limit=10)
```

```python
from promptic_sdk import AsyncPrompticClient

async with AsyncPrompticClient() as client:
    traces = await client.list_traces(limit=10)
```

Constructor args: `api_key`, `access_token`, `workspace_id`, `endpoint`, `timeout` (default 30s).

### API reference

For detailed method signatures and parameters, see [references/api.md](references/api.md).

## Agent Evaluation Workflow

Evaluate agent performance using datasets, runs, and evaluations.

### Step 1: Run agent with tracing

Instrument the agent with dataset and run tagging — traces are auto-collected:

```python
import promptic_sdk

promptic_sdk.init()

with promptic_sdk.ai_component("my-agent", dataset="eval-set", run="v2-improved"):
    for query in test_queries:
        agent.run(query)
```

### Step 2: Trigger evaluation

**Option A — CLI (recommended for agentic workflows):**

```bash
# Find the component and dataset IDs
promptic components list --json
promptic datasets list --component <comp-id> --json
promptic runs list --component <comp-id> --json

# Run evaluation (waits for completion by default)
promptic evaluations run <comp-id> --dataset <ds-id> --run <run-id> --name "v2-eval"

# Or don't wait and check later
promptic evaluations run <comp-id> --dataset <ds-id> --run <run-id> --no-wait
promptic evaluations get <eval-id> --component <comp-id>
```

**Option B — Python API:**

```python
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    components = client.list_components()
    comp_id = components["data"][0]["id"]

    datasets = client.list_datasets(comp_id)
    ds_id = datasets["data"][0]["id"]

    evaluation = client.create_evaluation(comp_id, ds_id, name="v2-eval")
    result = client.wait_for_evaluation(comp_id, evaluation["id"])

    for insight in result["results"]["insights"]:
        print(f"[{insight['severity']}] {insight['title']}: {insight['description']}")
```

## Prompt Optimization Workflow

Optimize prompts via experiments:

```python
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    # Create experiment
    exp = client.create_experiment(
        ai_component_id="comp_...",
        target_model="gpt-4.1-nano",
        task_type="classification",  # or "textGeneration", "structuredOutput"
        initial_prompt="Classify the following text into categories.",
        optimizer="prompticV2",      # or "miproV2", "bootstrapFewShot"
    )

    # Add training observations
    client.create_observations(exp["id"], [
        {"variables": {"message": "Great product!"}, "expected": "positive"},
        {"variables": {"message": "Terrible service"}, "expected": "negative"},
    ])

    # Add evaluators
    client.create_evaluators(exp["id"], [
        {"name": "accuracy", "type": "f1", "weight": 1.0},
    ])

    # Start optimization
    client.start_experiment(exp["id"])

    # After completion, deploy the best prompt
    best = client.get_best_iteration(exp["id"])
    client.deploy("comp_...", exp["id"])

    # Fetch deployed prompt at runtime
    prompt = client.get_deployed_prompt("comp_...")
    print(prompt["prompt"])
```

## CLI

The `promptic` CLI mirrors the API client. All commands support `--json` for JSON output.

```
# Auth
promptic login                      # Browser auth (device flow)
promptic logout                     # Clear saved credentials
promptic configure                  # Save API key & endpoint (CI/CD)

# Workspace
promptic workspace info             # Show current workspace details
promptic workspace list             # List accessible workspaces
promptic workspace select <id>      # Select active workspace

# Traces
promptic traces list                # List recent traces
promptic traces get <trace-id>      # Get trace with spans and events
promptic traces stats               # Aggregated tracing stats

# Components
promptic components list            # List AI components
promptic components create <name>   # Create a component
promptic components get <id>        # Get component details
promptic components delete <id>     # Delete a component

# Experiments
promptic experiments list           # List experiments
promptic experiments create         # Create experiment (interactive wizard)
promptic experiments get <id>       # Get experiment details
promptic experiments update <id>    # Update a pending experiment
promptic experiments delete <id>    # Delete an experiment
promptic experiments start <id>     # Start optimization
promptic experiments duplicate <id> [--start] [-p PROMPT]    # Clone experiment (observations + evaluators)
promptic experiments continue <id> [--start]                 # Clone, seed initial prompt from source's best iteration

# Observations (training data)
promptic observations list <exp-id>              # List observations
promptic observations add <exp-id> --from-file f # Bulk import (CSV/JSONL/JSON)
promptic observations add <exp-id> -i "..." -e "..." # Add single observation
promptic observations delete <exp-id> <obs-id>   # Delete an observation

# Evaluators
promptic evaluators list <exp-id>                # List evaluators
promptic evaluators add <exp-id> -n <name> -t <type>  # Add evaluator
promptic evaluators delete <exp-id> <eval-id>    # Delete an evaluator

# Iterations
promptic iterations list <exp-id>   # List iterations
promptic iterations get <exp-id> <iter-id>  # Get iteration with scores
promptic iterations best <exp-id>   # Get best-scoring iteration

# Deployments
promptic deployments status <comp-id>            # Show active deployment
promptic deployments deploy <comp-id> <exp-id>   # Deploy experiment
promptic deployments prompt <comp-id>            # Show deployed prompt
promptic deployments undeploy <comp-id>          # Remove deployment

# Datasets
promptic datasets create --component <id> --name <n>  # Create dataset
promptic datasets list --component <id>          # List datasets
promptic datasets get <ds-id> --component <id>   # Get dataset with items
promptic datasets delete <ds-id> --component <id>  # Delete dataset

# Runs
promptic runs create --component <id> --dataset <ds-id>  # Create run
promptic runs list --component <id>              # List runs
promptic runs get <run-id> --component <id>      # Get run with traces
promptic runs delete <run-id> --component <id>   # Delete run

# Annotations
promptic annotations create --component <id> --run <r> --trace <t>  # Annotate trace
promptic annotations list --component <id> --run <r>    # List by run
promptic annotations list --component <id> --dataset <d>  # List by dataset
promptic annotations delete <ann-id> --component <id> --run <r>  # Delete

# Evaluations
promptic evaluations run <comp-id> --dataset <ds-id> --run <run-id>  # Run evaluation (--run required)
promptic evaluations list --component <id>       # List evaluations
promptic evaluations get <eval-id> --component <id>     # Get results
```

## Key Types

Enums (Literal types):
- `ExperimentStatus`: `"pending" | "scheduled" | "running" | "completed" | "failed"`
- `ModelProvider`: `"openai" | "openrouter" | "custom" | "google"`
- `TaskType`: `"classification" | "textGeneration" | "structuredOutput"`
- `EvaluatorType`: `"f1" | "judge" | "similarity" | "structuredOutput"`
- `OptimizerType`: `"promptic" | "prompticV2" | "miproV2" | "bootstrapFewShot" | "gepa"`
