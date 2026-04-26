---
name: promptic-trace
description: >-
  Add LLM tracing to Python applications using the Promptic SDK. Use when code imports `promptic_sdk`, the user wants to instrument LLM/agent code with OpenTelemetry-based tracing, capture LLM provider calls (OpenAI, Anthropic, Bedrock, Vertex, Mistral, Cohere), trace LangChain / LangGraph / `create_agent` / deepagents / OpenAI Agents SDK / Claude Agent SDK / Pydantic AI runs, wrap pipeline stages in custom workflow spans (retrieval, normalization, control flow), or send spans to the Promptic platform via OpenTelemetry. Companion skills — `promptic-optimize` (prompt experiments), `promptic-agent-eval` (score agent traces), `promptic-benchmark` (Benchmark Studio submissions).
---

# Promptic Tracing

Instrument LLM applications with the Promptic Python SDK. All traces export
to the Promptic platform via OpenTelemetry; auto-instrumentation covers the
major LLM providers and agent frameworks.

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

Config resolution: explicit args > env vars (`PROMPTIC_API_KEY`,
`PROMPTIC_ENDPOINT`) > `~/.promptic/config.toml`.

## Quickstart

Call `promptic_sdk.init()` once at startup. All LLM calls from installed
providers are auto-instrumented via OpenTelemetry.

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

### `init()` parameters

| Parameter          | Description                                         | Default                  |
|--------------------|-----------------------------------------------------|--------------------------|
| `api_key`          | Promptic API key (falls back to `PROMPTIC_API_KEY`) | —                        |
| `endpoint`         | Platform URL (falls back to `PROMPTIC_ENDPOINT`)    | `https://promptic.eu`    |
| `auto_instrument`  | Auto-detect and instrument LLM client libraries     | `True`                   |
| `service_name`     | OpenTelemetry `service.name` resource attribute      | —                        |

Auto-detected instrumentors: OpenAI, Anthropic, Google Generative AI, Vertex
AI, Bedrock, Mistral, Cohere, LangChain (with LangGraph / `create_agent` /
deepagents), OpenAI Agents SDK, Claude Agent SDK. All emit the official
OpenTelemetry GenAI semantic conventions (`gen_ai.*`).

## Tagging spans

### `ai_component` context manager

Tag spans with an AI Component name. The platform links traces to the
matching component.

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

### `dataset` context manager

Tag spans with a dataset name independently:

```python
with promptic_sdk.ai_component("my-agent"):
    with promptic_sdk.dataset("eval-round-1"):
        agent.run(test_input)
```

## Tracing workflows with custom spans

Most users don't need this. With the right `[extras]` installed,
auto-instrumentation already creates spans for every LLM and tool call.
Reach for custom spans only when you have meaningful **non-LLM** workflow
logic (retrieval, normalization, business rules, control flow) you want
represented in the trace.

When you do need it, wrap your workflow stages in custom OpenTelemetry
spans. Auto-instrumented provider spans automatically nest under whichever
custom span is active.

Recommended pattern:

1. Wrap the whole run in one root workflow span inside `ai_component(...)`.
2. Add a child task span for each meaningful stage of the pipeline.
3. Record the stage's input and output as span attributes so the trace
   reads as a transformation, not just a list of LLM calls.

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

- Use semantic span names (`retrieve_context`, `rerank_results`) instead of
  generic function names when several calls would otherwise collide.
- For large payloads, log a small preview plus a count rather than the full
  object — traces are not meant to store data:

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

Verify with `promptic traces get <trace-id> --json`: the root workflow span
should carry structured input/output, task spans should appear as its
children, and auto-instrumented LLM/tool spans should nest under the task
that triggered them.

## Custom OpenTelemetry instrumentors

Since Promptic uses standard OpenTelemetry, add any OTel-compatible
instrumentor:

```python
import promptic_sdk
from opentelemetry.instrumentation.requests import RequestsInstrumentor

promptic_sdk.init()
RequestsInstrumentor().instrument()  # Spans exported to Promptic
```

## LangGraph / deepagents integration

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

## API client (for trace queries)

Both sync (`PrompticClient`) and async (`AsyncPrompticClient`) clients with
identical method signatures.

```python
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    traces = client.list_traces(limit=10)
    detail = client.get_trace(traces["data"][0]["id"])
    stats = client.get_stats(days_back=30)
```

Constructor args: `api_key`, `access_token`, `workspace_id`, `endpoint`,
`timeout` (default 30s).

For the full method surface (traces, workspace, components), see
[references/api.md](references/api.md).

## CLI

Trace-related commands:

```bash
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

# Components (lightweight reads)
promptic components list            # List AI components
promptic components create <name>   # Create a component
promptic components get <id>        # Get component details
promptic components delete <id>     # Delete a component
```

All commands support `--json` for machine-readable output.

## When to reach for a sibling skill

- Need to **score** an agent run by its trace (loop detection, tool errors,
  judge-based) → `promptic-agent-eval`.
- Need to **tune** a prompt via experiments + iterations → `promptic-optimize`.
- Need to **submit predictions** to a Benchmark Studio dataset (Kaggle-shaped
  external runtime) → `promptic-benchmark`.
