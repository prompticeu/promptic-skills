# Agent Gym External Submissions

Use this workflow to benchmark an agent that runs outside Promptic and submit
its results to an Agent Gym leaderboard.

## Contents

- Install and authenticate
- Keep the credential in the trusted runner
- Implement a candidate
- Submit with Python or the CLI
- Link an execution trace
- Run the minimal smoke test

## Install and authenticate

Once the Agent Gym interface is released, install it from PyPI:

```bash
python -m pip install promptic-sdk
```

Create an ordinary API key for the AI Application that owns the benchmark, then
configure the runner:

```bash
export PROMPTIC_ENDPOINT="https://promptic.eu"
export PROMPTIC_API_KEY="ptc_REPLACE_ME"
```

`AgentGymClient` uses the same configuration resolution as the rest of the SDK:
explicit constructor arguments, then `PROMPTIC_API_KEY` and
`PROMPTIC_ENDPOINT`, then saved CLI login/configuration.

Until then, test the implementation from the Python SDK PR branch
`feat/agent-gym-external-submissions`:

```bash
python -m pip install -e /path/to/promptic-python-sdk

# Or run without modifying the current environment:
uv run --with-editable /path/to/promptic-python-sdk python smoke_agent_gym.py
uv run --with-editable /path/to/promptic-python-sdk \
  promptic gym run "<BENCHMARK_UUID>" smoke_agent_gym:candidate \
  --name "external-smoke" --version "dev"
```

Treat the unreleased API as provisional until the SDK PR merges. Check the PR
branch rather than guessing if a signature differs.

## Keep the credential in the trusted runner

The external runner owns `PROMPTIC_API_KEY` and is responsible for
materializing cases, uploading outputs, and finalizing the submission. Only run
trusted callbacks in the same process as `AgentGymClient.submit(...)` or
`promptic gym run`.

Never pass the Promptic API key or authenticated client into generated,
sandboxed, or otherwise untrusted candidate code. Run untrusted code in an
isolated process without Promptic credentials, collect its prediction and
files, then let the trusted runner validate and submit those results.

## Implement a candidate

The high-level client creates a revision-bound submission, downloads every
benchmark case and input file, calls the candidate once per case, uploads
returned output artifacts, resolves optional traces, finalizes all predictions,
and polls submission and leaderboard-scoring status.

```python
from pathlib import Path

from promptic_sdk import (
    AgentGymCase,
    AgentGymCaseResult,
    AgentGymOutputArtifact,
)


def candidate(case: AgentGymCase) -> AgentGymCaseResult:
    input_paths = [item.local_path for item in case.files]
    result = run_local_agent(
        instructions=case.instructions,
        payload=case.input,
        files=input_paths,
    )

    return AgentGymCaseResult.structured(
        {"answer": result.answer, "confidence": result.confidence},
        artifacts=[
            AgentGymOutputArtifact(
                source=Path(result.report_path),
                path="report.pdf",
                mime_type="application/pdf",
            ),
        ],
    )
```

Return `AgentGymCaseResult.structured(...)` for JSON-like predictions,
`.text(...)` for text, or `.artifact(...)` when the file itself is the primary
output. `AgentGymOutputArtifact` accepts any local output file, including PDFs,
HTML, images, and slide decks. Set an appropriate MIME type, for example
`text/html`, `image/png`,
`application/vnd.openxmlformats-officedocument.presentationml.presentation`,
or `application/pdf`.

Output files are submission artifacts, not generic trace artifacts. Return them
through `AgentGymCaseResult` so the high-level runner reserves, uploads,
verifies, and attaches them to the correct prediction.

## Submit with Python

```python
from promptic_sdk import AgentGymClient

with AgentGymClient() as gym:
    result = gym.submit(
        benchmark_id="<BENCHMARK_UUID>",
        candidate=candidate,
        name="my-external-agent",
        version="1.0.0",
        architecture_description="Trusted local runner around my agent.",
        workdir=".promptic-agent-gym",
    )

print(result.run_id)
print(result.status["status"])
if result.status["run"]:
    print(result.status["run"]["scoring_status"])
    print(result.status["run"]["eligibility_status"])
```

By default, `submit()` waits for the submission and leaderboard evaluation.
Pass `wait=False` to return after finalization is queued.

## Submit with the CLI

Expose the candidate as an importable `module:function`, then run:

```bash
promptic gym run "<BENCHMARK_UUID>" my_agent:candidate \
  --name "my-external-agent" \
  --version "1.0.0" \
  --workdir ".promptic-agent-gym" \
  --json
```

Use `--revision-id "<REVISION_UUID>"` to pin a published revision,
`--idempotency-key "<STABLE_RETRY_KEY>"` for retry-safe execution, or
`--no-wait` to return after scoring is queued.

## Link an execution trace

Initialize tracing with the same endpoint and AI Application key. Inside the
candidate, capture the active span's raw 32-hex OpenTelemetry trace ID and
return it in `raw_trace_ids`:

```python
from opentelemetry import trace

import promptic_sdk
from promptic_sdk import AgentGymCaseResult

promptic_sdk.init(service_name="agent-gym-external-runner")
tracer = trace.get_tracer(__name__)


def traced_candidate(case):
    with tracer.start_as_current_span("agent_gym.case") as span:
        prediction = run_local_agent(case.input)
        raw_trace_id = f"{span.get_span_context().trace_id:032x}"
        return AgentGymCaseResult.structured(
            prediction,
            raw_trace_ids=[raw_trace_id],
        )
```

The high-level runner flushes the tracer provider, waits for Promptic to ingest
and resolve each raw OTEL ID, and associates the resolved trace with its
prediction. If tracing is unavailable, omit `raw_trace_ids`.

## Minimal smoke test

Use a benchmark placeholder whose published revision contains one case. This
produces one structured prediction, uploads one HTML artifact, and links one
trace when tracing is configured:

```python
from pathlib import Path

from opentelemetry import trace

import promptic_sdk
from promptic_sdk import (
    AgentGymCase,
    AgentGymCaseResult,
    AgentGymClient,
    AgentGymOutputArtifact,
)

BENCHMARK_ID = "<BENCHMARK_UUID>"

promptic_sdk.init(service_name="agent-gym-smoke")
tracer = trace.get_tracer(__name__)


def candidate(case: AgentGymCase) -> AgentGymCaseResult:
    with tracer.start_as_current_span("agent_gym.smoke_case") as span:
        report = Path("smoke-output") / case.id / "report.html"
        report.parent.mkdir(parents=True, exist_ok=True)
        report.write_text("<h1>Agent Gym smoke test</h1>", encoding="utf-8")
        raw_trace_id = f"{span.get_span_context().trace_id:032x}"
        prediction = {"answer": "smoke-ok", "case_ordinal": case.ordinal}

    return AgentGymCaseResult.structured(
        prediction,
        artifacts=[
            AgentGymOutputArtifact(
                source=report,
                path="report.html",
                mime_type="text/html",
            )
        ],
        raw_trace_ids=[raw_trace_id],
    )


with AgentGymClient() as gym:
    result = gym.submit(
        BENCHMARK_ID,
        candidate,
        name="external-smoke",
        version="dev",
        workdir=".promptic-agent-gym-smoke",
        idempotency_key="<UNIQUE_OR_STABLE_RETRY_KEY>",
    )

print(result.run_id, result.status["status"], result.status["run"])
```

Set `PROMPTIC_API_KEY="ptc_REPLACE_ME"` and `PROMPTIC_ENDPOINT` in the trusted
runner environment before executing the smoke test. Do not place real
credentials in source code.
