---
name: promptic-agent-eval
description: >-
  Score agent performance by analysing the agent's traces (loop detection, tool errors, unused tools, judge rubric, structured output) using Promptic's Agent Evaluation pipeline. Use when the user wants to evaluate an agent run, create or run an evaluation against a dataset, list / fetch insights, group runs into datasets for comparison, annotate individual traces, or programmatically trigger / wait for evaluations through `PrompticClient` or the `promptic` CLI. Also use when the user mentions agent insights, evaluation runs, agent runs, or wants to compare agent versions side-by-side. Companion skills — `promptic-trace` (instrument the agent first), `promptic-optimize` (prompt tuning via experiments), `promptic-benchmark` (Benchmark Studio external-runtime submissions — different track, scores predictions vs gold rather than scoring traces).
---

# Promptic Agent Evaluation

Score agent runs by analysing their *traces* (loops, tool errors, unused
tools, judge-based rubrics, structured output). Uses the platform's
`components / datasets / runs / evaluations` tables.

> **Distinct from `promptic-benchmark`.** This skill scores agent traces
> (the *how* of a run). Benchmark Studio scores prediction values against
> held-out gold (the *what*). Different DB tables, different endpoints.
> If the user shows you a `/dashboard/.../benchmarks/ie/<uuid>` URL, switch
> to the `promptic-benchmark` skill.

## Prerequisites

The agent must already emit traces to Promptic. If it doesn't, instrument
it first per the **`promptic-trace`** skill (`promptic_sdk.init()` +
`ai_component(...)` context manager). Tracing is what produces the data
this skill scores.

## Installation

```bash
pip install promptic-sdk
```

(See `promptic-trace` skill for provider/framework extras and auth.)

## Workflow

### Step 1 — Run the agent with tracing

Instrument the agent with dataset and run tagging — traces are
auto-collected:

```python
import promptic_sdk

promptic_sdk.init()

with promptic_sdk.ai_component("my-agent", dataset="eval-set", run="v2-improved"):
    for query in test_queries:
        agent.run(query)
```

The `dataset` and `run` kwargs auto-create those entities if they don't
exist; subsequent calls reuse them.

### Step 2 — Trigger evaluation

**Option A — CLI (recommended for agentic / scripted workflows):**

```bash
# Find component / dataset / run IDs
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

`AgentEvaluation` status: `"pending" | "running" | "completed" | "failed"`.
`results` contains an `InsightResult` with `insights` list and `meta` object.

## CLI

```bash
# Datasets
promptic datasets create --component <id> --name <n>  # Create dataset
promptic datasets list --component <id>          # List datasets
promptic datasets get <ds-id> --component <id>   # Get dataset with items
promptic datasets delete <ds-id> --component <id>  # Delete dataset

# Runs (group of traces for comparison)
promptic runs create --component <id> --dataset <ds-id>  # Create run
promptic runs list --component <id>              # List runs
promptic runs get <run-id> --component <id>      # Get run with traces
promptic runs delete <run-id> --component <id>   # Delete run

# Annotations (per-trace ratings + comments)
promptic annotations create --component <id> --run <r> --trace <t>  # Annotate trace
promptic annotations list --component <id> --run <r>    # List by run
promptic annotations list --component <id> --dataset <d>  # List by dataset
promptic annotations delete <ann-id> --component <id> --run <r>  # Delete

# Evaluations
promptic evaluations run <comp-id> --dataset <ds-id> --run <run-id>  # Run evaluation (--run required)
promptic evaluations list --component <id>       # List evaluations
promptic evaluations get <eval-id> --component <id>     # Get results
```

All commands support `--json` for machine-readable output.

## API reference

For full method signatures (datasets, runs, annotations, evaluations), see
[references/api.md](references/api.md).
