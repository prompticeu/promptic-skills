# Agent Optimization: external submissions

Use this workflow when a complete Agent runs in the user's environment and
Promptic should evaluate different implementations against the same benchmark.
Call each implementation a **variant**. Promptic owns the immutable benchmark,
evaluation engine, scored results, and leaderboard; the external runner owns
execution and submits the resulting evidence.

This reference covers configuring, executing, evaluating, and iterating on
externally run Agent variants.

## Choose the interface

- Use `AgentGymClient.run_and_submit(...)` for a trusted Python callback. It is
  the shortest correct path.
- Use `promptic agent-gym run` when the callback is importable as
  `module:function` and a CLI workflow is more convenient.
- Use `start_submission(...)` and the resumable session API when execution is
  isolated, distributed, or must survive process restarts.

## Follow the optimization loop

Agent Optimization is an evidence-driven iteration loop:

1. Define the benchmark: the Agent goal, input/output contract, representative
   cases, and evaluators. Publish it as an immutable version.
2. Implement a baseline Agent architecture. Record a stable variant name and
   version plus its source revision and a concise architecture description.
3. Run the variant against every frozen case, upload its predictions and
   evidence, submit it for scoring, and wait for evaluation to finish.
4. Inspect the overall results and the weakest or failed cases. Review each
   evaluator's score and reasoning, generated files, latency, and token usage.
5. Open linked traces when the result alone does not explain a failure. Use
   them to inspect tool selection, call order, retries, intermediate failures,
   and slow steps.
6. Form a concrete hypothesis and change the Agent architecture—for example
   its instructions, model, tools, retrieval, control flow, validation, or
   retry policy. Avoid changing several unrelated variables at once.
7. Submit the changed implementation as a new child variant, recording its
   parent, rationale, and intended behavioral effect.
8. Compare the variants on the same benchmark version. Keep the change only
   when the case-level evidence supports the aggregate improvement, then
   repeat from step 4.

Keep the benchmark fixed while comparing architectures. If the goal, cases,
schemas, or evaluators change, publish a new benchmark version and establish a
new baseline instead of treating its scores as directly comparable.

## Authenticate the trusted runner

Install the released SDK and provide an AI Application-scoped credential:

```bash
python -m pip install promptic-sdk
export PROMPTIC_ENDPOINT="https://promptic.eu"
export PROMPTIC_API_KEY="ptc_..."
export BENCHMARK_ID="<benchmark-uuid>"
```

Interactive local workflows may use `promptic login` instead. The SDK resolves
explicit constructor arguments first, then environment variables, then saved
CLI configuration.

Keep Promptic and model-provider credentials in the trusted coordinator. Never
give them to generated, sandboxed, or otherwise untrusted candidate code. Run
untrusted code without platform credentials, collect its outputs, and let the
trusted coordinator upload those outputs through a submission session.

## Optionally configure the Agent externally

The dashboard is not required. A coding agent or CI job can create the Agent,
its input/output contract, evaluators, cases, and files before publishing an
immutable benchmark version:

```bash
promptic agent-gym create agent.json
promptic agent-gym status <benchmark-id>
promptic agent-gym publish-draft <benchmark-id>
```

The Python equivalent uses `gym.benchmarks.create(...)`, `agent.cases.add(...)`,
and `agent.publish()`. Keep this part brief unless the user explicitly asks to
author the benchmark; the core skill workflow is external execution and
submission.

## Implement the callback

The callback receives one materialized `AgentGymCase`. Input-file fields are
available as local paths. Return a terminal `AgentGymCaseResult` with structured
or text output, generated files, optional trace IDs, and useful execution
metrics.

```python
from pathlib import Path

from promptic_sdk import AgentGymCase, AgentGymCaseResult, AgentGymOutputArtifact


def run(case: AgentGymCase) -> AgentGymCaseResult:
    result = run_local_agent(case.input)
    return AgentGymCaseResult.succeeded(
        {"answer": result.answer, "confidence": result.confidence},
        artifacts=[
            AgentGymOutputArtifact(
                source=Path(result.report_path),
                field_path="report",
                path="report.pdf",
                mime_type="application/pdf",
            )
        ],
    )
```

Use `.succeeded(...)` for structured output, return a string for plain text,
and use `.artifact(...)` when generated files are the primary result. Use
`.failed(...)` for a terminal case failure instead of dropping the case.
Generated files are prediction artifacts, not generic trace artifacts; return
them through the case result so the SDK verifies and attaches them to the
correct prediction. Each artifact's `field_path` must name its Output-schema
field.

If tracing matters, return raw 32-character OpenTelemetry trace IDs through
`raw_trace_ids`. The runner resolves them after ingestion and links the traces
to the prediction. Omit them when tracing is not part of evaluation.

## Run and submit

### Python

```python
import os

from promptic_sdk import AgentGymClient
from my_agent import run

with AgentGymClient() as gym:
    result = gym.run_and_submit(
        benchmark_id=os.environ["BENCHMARK_ID"],
        executor=run,
        name="invoice-agent",
        version="1.2.0",
        architecture_description=(
            "Extract the document, validate required fields, and generate a "
            "review report before returning the structured result."
        ),
        repository_url="https://github.com/acme/invoice-agent",
        commit_hash="<git-commit>",
    )

print(result.run_id)
```

Give each variant a stable name and version. Include the repository URL, commit
hash, and a useful architecture description when available so a winning run is
reproducible. For a child variant, provide parent identity plus a rationale and
the intended behavioral effect.

### CLI

```bash
promptic agent-gym run "$BENCHMARK_ID" my_agent:run \
  --name invoice-agent \
  --version 1.2.0 \
  --architecture architecture.md
```

The high-level workflow materializes the frozen cases, executes the callback,
uploads generated files, and persists each completed prediction immediately.
Only after every frozen case has a terminal prediction does it submit the
session for scoring. A missing prediction therefore produces an explicit
incomplete-submission error instead of silently scoring partial coverage.

## Use a resumable session for isolated execution

```python
with AgentGymClient() as gym:
    submission = gym.start_submission(
        os.environ["BENCHMARK_ID"],
        idempotency_key="invoice-agent-1.2.0-session",
        variant_identity={
            "name": "invoice-agent",
            "version": "1.2.0",
            "architecture_description": "Extract, validate, and report.",
        },
    )

    materialized = submission.materialize_manifest("./agent-inputs")
    for case in materialized.manifest["data"]:
        result = execute_isolated(case, materialized.files)
        submission.add_prediction(case["dataset_case_id"], result)

    submission.finalize(
        idempotency_key="invoice-agent-1.2.0-submit",
    )
    status = submission.wait(max_wait=600, poll_interval=2)
```

`add_prediction(...)` persists the prediction immediately; it is not merely an
in-memory staging operation. Repeating an upload safely replaces that case's
canonical prediction while the session is open. Lower-level batch uploads are
limited to 500 cases per request. Keep the submission ID and use
`resume_submission(benchmark_id, submission_id)` after a process restart.

Finalization verifies exact frozen-case coverage, closes further prediction
writes, and requests scoring for the existing run. Reuse stable idempotency
keys when retrying the same session creation or scoring submission; using the
same key for different content returns a conflict.

## Inspect and compare results

```bash
promptic agent-gym results "$BENCHMARK_ID" "$RUN_ID"
promptic agent-gym case-results "$BENCHMARK_ID" "$RUN_ID" --sort score --limit 5
promptic agent-gym case-result "$BENCHMARK_ID" "$RUN_ID" 42
promptic agent-gym artifact-download \
  "$BENCHMARK_ID" "$RUN_ID" 42 0 --output review/report.pdf
promptic agent-gym compare-runs \
  "$BENCHMARK_ID" "$BASELINE_RUN_ID" "$CANDIDATE_RUN_ID"
```

The SDK equivalents are `get_run_results()`, `list_case_results()`,
`iter_case_results()`, `get_case_result()`, `download_prediction_artifact()`,
and `compare_runs()`. Review weak and failed cases, judge reasoning, generated
files, trace evidence, latency, and token usage; do not choose a variant from
the mean score alone. Paired comparison requires compatible runs from the same
immutable benchmark and evaluator setup.

## Monitor and recover

```bash
promptic agent-gym submission-status "$BENCHMARK_ID" "$SUBMISSION_ID"
promptic agent-gym submission-wait "$BENCHMARK_ID" "$SUBMISSION_ID" --max-wait 600
promptic agent-gym retry-scoring "$BENCHMARK_ID" "$RUN_ID"
promptic agent-gym submission-cancel "$BENCHMARK_ID" "$SUBMISSION_ID" --yes
```

- Retry scoring only after a recoverable scoring-dispatch failure. It reuses
  the existing immutable predictions and does not rerun the Agent or create a
  replacement leaderboard entry.
- Rerunning the Agent creates a new empty submission session.
- Cancel only an unsubmitted session that should accept no more predictions.
- If required evaluator evidence is missing or evaluation fails, inspect the
  run's eligibility state rather than treating a partial score as official.

## Completion checklist

- The variant identity describes the implementation that actually ran.
- Every frozen case has one terminal prediction, including explicit failures.
- Generated files are returned as output artifacts and trace IDs are linked
  only when useful.
- The session was submitted for scoring and reached a terminal state.
- The run and weakest cases were inspected; related variants were compared
  only when their benchmark and evaluator fingerprints are compatible.
