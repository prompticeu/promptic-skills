---
name: promptic-optimize
description: >-
  Tune prompts via Promptic experiments — create an experiment, attach training observations and evaluators, run an optimizer (`prompticV2`, `miproV2`, `bootstrapFewShot`, `gepa`), inspect iterations and the best-scoring iteration, then deploy the optimized prompt back to an AI Component for runtime use. Use when the user wants to improve / tune / optimize an LLM prompt, run prompt experiments, deploy a tuned prompt, manage observations (training data) for an experiment, configure evaluators (`f1`, `judge`, `similarity`, `structuredOutput`), or fetch the deployed prompt at runtime via `client.get_deployed_prompt(...)`. Companion skills — `promptic-trace` (instrument LLM calls), `promptic-agent-eval` (score agent traces), `promptic-benchmark` (Benchmark Studio external-runtime submissions).
---

# Promptic Prompt Optimization

Tune prompts by running experiments: each experiment defines an initial
prompt, training observations, and evaluators; the optimizer iterates until
it converges; the best iteration can be deployed for runtime use.

## Installation

```bash
pip install promptic-sdk
```

(See `promptic-trace` skill for provider/framework extras and auth.)

## End-to-end example

```python
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    # Create experiment
    exp = client.create_experiment(
        ai_component_id="comp_...",
        target_model="gpt-4.1-nano",
        task_type="classification",  # or "textGeneration", "structuredOutput"
        initial_prompt="Classify the following text into categories.",
        optimizer="prompticV2",      # or "miproV2", "bootstrapFewShot", "gepa"
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

## Key types

Enums (Literal types):

- `ExperimentStatus`: `"pending" | "scheduled" | "running" | "completed" | "failed"`
- `ModelProvider`: `"openai" | "openrouter" | "custom" | "google"`
- `TaskType`: `"classification" | "textGeneration" | "structuredOutput"`
- `EvaluatorType`: `"f1" | "judge" | "similarity" | "structuredOutput"`
- `OptimizerType`: `"promptic" | "prompticV2" | "miproV2" | "bootstrapFewShot" | "gepa"`

## Observation format

```python
{"variables": dict[str, Any], "expected": str, "split": str | None}
```

`split` defaults to `"eval"`. Multi-variable experiments key by variable
name (e.g. `{"customer_message": "...", "context": "..."}`); legacy
single-variable experiments use `{"input": "..."}`.

## Evaluator format

```python
{"name": str, "type": EvaluatorType, "weight": float, "description"?: str, "config"?: dict}
```

- `f1` — classification F1 score
- `similarity` — string / embedding similarity
- `judge` — LLM-as-judge with rubric
- `structuredOutput` — per-field scoring of JSON predictions

## Iterating on an existing experiment

```bash
# Clone an experiment (observations + evaluators), optionally start it
promptic experiments duplicate <exp-id> [--start] [-p PROMPT]

# Clone + seed initial prompt from the source's best iteration
promptic experiments continue <exp-id> [--start]
```

`continue` seeds the new experiment with the previous best prompt — useful
for "warm-start the next pass from where we left off". `duplicate` keeps the
source's initial prompt unless `-p` overrides it.

## CLI

```bash
# Experiments
promptic experiments list           # List experiments
promptic experiments create         # Create experiment (interactive wizard)
promptic experiments get <id>       # Get experiment details
promptic experiments update <id>    # Update a pending experiment
promptic experiments delete <id>    # Delete an experiment
promptic experiments start <id>     # Start optimization
promptic experiments duplicate <id> [--start] [-p PROMPT]
promptic experiments continue <id> [--start]

# Observations (training data)
promptic observations list <exp-id>
promptic observations add <exp-id> --from-file f      # Bulk import (CSV/JSONL/JSON)
promptic observations add <exp-id> -i "..." -e "..."  # Add a single observation
promptic observations delete <exp-id> <obs-id>

# Evaluators
promptic evaluators list <exp-id>
promptic evaluators add <exp-id> -n <name> -t <type>
promptic evaluators delete <exp-id> <eval-id>

# Iterations (results)
promptic iterations list <exp-id>
promptic iterations get <exp-id> <iter-id>
promptic iterations best <exp-id>

# Deployments
promptic deployments status <comp-id>
promptic deployments deploy <comp-id> <exp-id>
promptic deployments prompt <comp-id>
promptic deployments undeploy <comp-id>
```

All commands support `--json` for machine-readable output.

## API reference

For full method signatures (experiments, observations, evaluators,
iterations, deployments), see [references/api.md](references/api.md).
