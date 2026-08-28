---
name: promptic
description: Integrate the Promptic Python SDK for LLM observability, tracing, prompt optimization, datasets, and Agent Optimization. Use when code imports `promptic_sdk`; when the user asks to trace, optimize, benchmark, or deploy an agent or prompt; or when running an agent externally and submitting predictions, files, or traces for scoring. Also use for OpenTelemetry-based LLM, architecture, workflow, and semantic parent/child tracing, canonical datasets, or programmatic AI Component management.
---

# Promptic Python SDK

SDK and CLI for the [Promptic](https://promptic.eu) platform — LLM tracing,
prompt optimization, and Agent Optimization.

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
export PROMPTIC_API_KEY="ptc_..."
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

### File and media artifacts

Promptic automatically offloads inline base64 media and large file-like content
from auto-instrumented spans into trace artifacts. The span keeps a lightweight
`promptic-artifact://...` reference, and the UI/API/CLI can fetch the bytes on
demand.

Artifact uploads should avoid routing file bytes through the Promptic app server
when the platform supports direct storage uploads. The SDK should prefer the
storage-object flow: request a presigned upload from Promptic, upload bytes
directly to object storage, then register the artifact metadata with Promptic.
Only fall back to server-side `contentBase64` uploads for older servers or
temporary compatibility. Do not ask users or coding agents to manually keep
large base64 payloads in span attributes.

Do not silently read local filesystem paths from span attributes. If the user
wants local file contents in a trace, attach them explicitly:

```python
file_ref = promptic_sdk.artifact("/tmp/report.pdf")
span.set_attribute("retrieval.input_file", file_ref.ref)
```

Use this helper for unsupported custom file payloads. External HTTP(S) URLs can
remain as URLs.

When debugging image/file previews in the UI, distinguish ingestion from browser
rendering. If traces contain `promptic-artifact://...` references and artifact
metadata exists, but images do not render, check that the frontend Content
Security Policy allows the object-storage origin. Self-hosted or custom storage
setups should provide `APP_STORAGE_CSP_ORIGINS` with the browser-facing storage
origin, and production Docker/Next builds must receive that value at build time.

### ai_component context manager

Tag spans with an AI Component name. The platform links traces to the matching component.

```python
with promptic_sdk.ai_component("customer-support-agent"):
    # All LLM calls here are tagged
    ...

```

Parameters:
- `name` (str): AI Component name in the workspace

### Tracing workflows with custom spans

Most users don't need this. With the right `[extras]` installed, auto-instrumentation already creates spans for every LLM and tool call. Reach for custom spans only when you have meaningful **non-LLM** workflow logic (retrieval, normalization, business rules, control flow) you want represented in the trace.

When you do need it, wrap your workflow stages in custom OpenTelemetry spans. Auto-instrumented provider spans automatically nest under whichever custom span is active.

Recommended pattern:

1. Wrap the whole run in one root workflow span inside `ai_component(...)`.
2. Add a child task span for each meaningful stage of the pipeline.
3. Record the stage's input and output as span attributes so the trace reads as a transformation, not just a list of LLM calls.

```python
import json
import promptic_sdk
from opentelemetry import trace

promptic_sdk.init()
tracer = trace.get_tracer(__name__)

with promptic_sdk.ai_component("my-agent"):
    with tracer.start_as_current_span("run_workflow") as root:
        root.set_attribute("traceloop.span.kind", "workflow")
        root.set_attribute("traceloop.entity.input", json.dumps(user_input))

        with tracer.start_as_current_span("retrieve_context") as span:
            span.set_attribute("traceloop.span.kind", "task")
            span.set_attribute("traceloop.entity.input", json.dumps(query))
            context = retrieve(query)
            span.set_attribute("traceloop.entity.output", json.dumps(context))

        with tracer.start_as_current_span("generate_answer") as span:
            span.set_attribute("traceloop.span.kind", "task")
            # Auto-instrumented LLM call nests under this task span
            answer = llm_call(context)

        root.set_attribute("traceloop.entity.output", json.dumps(answer))
```

Span attribute conventions:

- `traceloop.span.kind="workflow"` — the top-level run
- `traceloop.span.kind="task"` — an internal pipeline stage
- `traceloop.entity.input` / `traceloop.entity.output` — JSON-serialized stage payloads
- `gen_ai.*` — reserved for LLM/tool spans; auto-instrumentors emit these

Tips:

- Use semantic span names (`retrieve_context`, `rerank_results`) instead of generic function names when several calls would otherwise collide.
- For large payloads, log a small preview plus a count rather than the full object — traces are not meant to store data:

  ```python
  span.set_attribute(
      "traceloop.entity.output",
      json.dumps({
          "items": items[:5],
          "item_count": len(items),
          "additional_item_count": max(len(items) - 5, 0),
      }),
  )
  ```

Verify with `promptic traces get <trace-id> --json`: the root workflow span should carry structured input/output, task spans should appear as its children, and auto-instrumented LLM/tool spans should nest under the task that triggered them.

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
`gen_ai.operation.name`, `gen_ai.usage.*`) for flat agents and multi-agent
graphs uniformly.

Users who prefer the LangSmith OTel bridge (e.g. for hybrid dual-export to
LangSmith) can opt in by setting `LANGSMITH_TRACING=true` and
`LANGSMITH_OTEL_ENABLED=true` before calling `init()`. Note: the LangSmith
bridge does not emit tool definitions, so tool metadata may be incomplete on
LangSmith-bridged traces.

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

## Agent Optimization: external submissions

Use `AgentGymClient.run_and_submit(...)` or `promptic agent-gym run` when a
complete Agent runs outside Promptic. Promptic supplies one immutable benchmark
version; the trusted runner executes every case and progressively persists its
predictions, generated files, and optional traces before requesting scoring.

Read [references/agent-gym.md](references/agent-gym.md) before implementing this
workflow. It covers the trust boundary, callback contract, resumable sessions,
durable uploads, scoring submission, result inspection, comparison, and
recovery. For exact public signatures, also read the Agent Optimization section
of [references/api.md](references/api.md). Do not add or describe Auto Engineer
or autonomous optimization loops; they are not part of this workflow.

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

# Agent Optimization
promptic agent-gym run <benchmark-id> my_agent:run \
  --name my-agent --version 1.0.0 --architecture architecture.md
promptic agent-gym results <benchmark-id> <run-id>
promptic agent-gym compare-runs <benchmark-id> <baseline-run-id> <candidate-run-id>

# Workspace
promptic workspace info             # Show current workspace details
promptic workspace list             # List accessible workspaces
promptic workspace select <id>      # Select active workspace

# Traces
promptic traces list                # List recent traces
promptic traces get <trace-id>      # Get trace with spans and events
promptic traces artifacts <trace-id> # List trace artifacts
promptic artifacts get <artifact-id> -o file.bin # Download artifact bytes
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

```

## Key Types

Enums (Literal types):
- `ExperimentStatus`: `"pending" | "scheduled" | "running" | "completed" | "failed"`
- `ModelProvider`: `"openai" | "openrouter" | "custom" | "google"`
- `TaskType`: `"classification" | "textGeneration" | "structuredOutput"`
- `EvaluatorType`: `"f1" | "referenceJudge" | "comparisonJudge" | "generalJudge" | "similarity" | "structuredOutput"`
- `OptimizerType`: `"promptic" | "prompticV2" | "miproV2" | "bootstrapFewShot" | "gepa"` — `"promptic"` is the legacy v1 value retained for historical experiments; use `"prompticV2"` for new ones.
