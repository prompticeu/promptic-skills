# API Reference

Complete method signatures for `PrompticClient` and `AsyncPrompticClient`. Both clients share identical signatures — async methods are prefixed with `await`.

## Traces

```python
client.list_traces(*, limit=50, offset=0, status=None, start_after=None, start_before=None) -> TraceList
client.get_trace(trace_id: str) -> Trace
client.list_trace_artifacts(trace_id: str) -> TraceArtifactList
client.get_artifact(artifact_id: str) -> TraceArtifact
client.get_artifact_content(artifact_id: str) -> bytes
client.download_artifact(artifact_id: str, path: str | os.PathLike[str]) -> None
client.get_stats(*, days_back=30) -> TracingStats
```

- `status`: `"ok"` or `"error"`
- `start_after` / `start_before`: ISO timestamp strings

`list_traces()` and `get_stats()` do not expose the telemetry-derived `service`
(OpenTelemetry `service.name`) and `environment` (`deployment.environment.name`)
filters. Use the `service` and `environment` query parameters on the REST API to
filter by them programmatically.

## Artifacts

```python
promptic_sdk.artifact(value: str | bytes | os.PathLike[str], *, name=None, mime_type=None, api_key=None, endpoint=None) -> ArtifactReference
```

Use `promptic_sdk.artifact(...)` when custom code needs to attach local files,
bytes, or large text payloads to traces explicitly. The returned reference can be
stored in a span attribute, for example `span.set_attribute("input_file",
ref.ref)`. `name` is stored on the artifact and used as the default download
filename; for local files it defaults to the file's base name, so pass it
explicitly for bytes or text content, and read it back from `ref.name`. The SDK
prefers direct object-storage upload via Promptic's presign API and only falls
back to server-side base64 upload for compatibility.

## Agent Optimization

The user-facing feature is Agent Optimization. The Python SDK retains the
`AgentGymClient` name and the CLI uses the `agent-gym` command group.

```python
AgentGymClient(
    endpoint: str | None = None,
    timeout: float = 30.0,
    api_key: str | None = None,
    access_token: str | None = None,
    ai_application_id: str | None = None,
)

client.download_dataset(
    benchmark_id: str,
    destination: str | os.PathLike[str],
    *,
    revision_id: str | None = None,
    page_size: int = 100,
    overwrite: bool = False,
) -> DownloadedBenchmarkDataset

client.run_and_submit(
    benchmark_id: str,
    executor: Candidate,
    *,
    name: str,
    version: str,
    architecture_description: str,
    repository_url=None,
    commit_hash=None,
    revision_id=None,
    variant_identity=None,
    metadata=None,
    workdir=None,
    idempotency_key=None,
    capture_exceptions=True,
    wait=True,
    max_wait=600,
    poll_interval=2,
    trace_max_wait=30,
    trace_poll_interval=0.5,
    trace_cases=False,
    trace_policy="best_effort",
    input_model=None,
) -> AgentGymRunResult
```

`AsyncAgentGymClient.run_and_submit(...)` has the same arguments and awaits an
async or synchronous candidate callback.

Resumable sessions:

```python
session = client.start_submission(
    benchmark_id,
    *,
    idempotency_key: str,
    variant_identity: VariantIdentity,
    revision_id=None,
    ttl_seconds=86400,
)
session.get_manifest(page_size=100)
session.materialize_manifest(destination)
session.add_prediction(case_id, result)
session.finalize(*, idempotency_key, metadata=None, trace_policy="best_effort")
session.status()
session.wait(max_wait=600, poll_interval=2)
session.cancel()
session.retry_scoring()

client.resume_submission(benchmark_id, submission_id)
```

`add_prediction(...)` uploads and persists the case result immediately.
`finalize(...)` verifies exact frozen-case coverage, closes prediction writes,
and requests scoring. Lower-level `upload_predictions(...)` accepts 1-500
predictions and safely replaces included cases while the session remains open.

Candidate inputs and results:

```python
AgentGymCase(
    id: int,
    ordinal: int,
    input: dict[str, Any],
    task: dict[str, Any],
)

AgentGymOutputArtifact(
    source: pathlib.Path,
    field_path: str,
    path: str | None = None,
    mime_type: str | None = None,
    role: str = "output",
)

AgentGymCaseResult.succeeded(value, *, artifacts=(), raw_trace_ids=())
AgentGymCaseResult.artifact(
    *artifacts,
    output=None,
    raw_trace_ids=(),
)
AgentGymCaseResult.failed(
    *,
    error_code: str,
    error: str,
    error_category=None,
    retryable=False,
    diagnostics=None,
)
```

`AgentGymRunResult` fields: `submission_id`, `revision_id`, `run_id`,
`variant_id`, and `status`. `status["run"]` includes scoring and eligibility
state when a leaderboard run exists.

Verifier authoring uses
`VerifierMetric(key, name, instructions, weight=1, threshold=None)`,
`EvidencePolicy`, and `InvestigationBudget(max_steps=20)`. The SDK translates
inline metric scoring into the API's separate binding map. Verifier metrics
aggregate independently, so `VerifierAgent` has no evaluator-level weight.
The lower-level `MetricBinding` mapping remains available for compatibility but
must not configure a metric that already has non-default inline scoring.
`ExpectedBehaviorJudge` accepts an optional model and a `behavior_compliance`
metric binding.

Result inspection and recovery:

```python
client.get_run_results(benchmark_id, run_id)
client.list_case_results(benchmark_id, run_id, *, sort="score", limit=100, cursor=None)
client.iter_case_results(benchmark_id, run_id, *, page_size=100, sort="score")
client.get_case_result(benchmark_id, run_id, case_id)
client.download_prediction_artifact(artifact, destination, *, overwrite=False)
client.compare_runs(benchmark_id, *, parent_run_id, candidate_run_id)
client.get_submission_status(benchmark_id, submission_id)
client.wait_for_submission(benchmark_id, submission_id, *, max_wait=600, poll_interval=2)
client.retry_scoring(benchmark_id, run_id)
client.cancel_submission(benchmark_id, submission_id)
```

`get_run_results()` includes `score_status_counts`. Verifier evaluator results
include `source_evaluator_id` and `metric_key`; per-field result maps declare
their `mean_per_field_scores_basis` as `succeeded_only`.

Use `references/agent-gym.md` for the executable workflow and trust boundary.
Use the session API when the trusted runner must isolate untrusted execution,
resume after a restart, or control protocol steps directly.

## AI Application

```python
client.get_ai_application() -> AIApplication
```

`get_workspace()` / `Workspace` remain as deprecated aliases. AI Application scope
is set with the `ai_application_id` argument (or the `PROMPTIC_AI_APPLICATION_ID`
env var), which the SDK sends as the `X-AI-Application-Id` header; the legacy
`workspace_id` / `PROMPTIC_WORKSPACE_ID` still work. Component and artifact
responses expose the scope as `aiApplicationId`.

## Components

```python
client.list_components() -> ComponentList
client.create_component(name: str, *, description=None) -> ComponentCreated
client.get_component(component_id: str) -> Component
client.delete_component(component_id: str) -> None
```

## Experiments

```python
client.list_experiments(*, component_id=None, status=None, limit=50, offset=0) -> ExperimentList
client.create_experiment(
    ai_component_id: str,
    target_model: str,
    *,
    task_type="classification",      # "classification" | "textGeneration" | "structuredOutput"
    initial_prompt=None,
    name=None,
    description=None,
    provider="openai",               # "openai" | "openrouter" | "custom" | "google"
    optimizer="prompticV2",          # "prompticV2" | "miproV2" | "bootstrapFewShot" | "gepa"
    hyperparameters=None,            # {"epochs": int, "trainSplitRatio": float, "numFewShots": int, "enableCot": bool}
    initial_prediction_model_schema=None,
) -> Experiment
client.create_tool_selection_experiment(
    ai_component_id: str,
    *,
    tools: list[dict],               # [{"name", "description", "input_schema"?}]
    test_cases: list[dict],          # [{"query", "expected_tool"}]  ("" expected_tool = no tool)
    target_model=None,               # omit to use the platform default
    tool_source="manual",            # "manual" | "mcp"
    system_prompt=None,
    optimize_system_prompt=False,
    epochs=None,
    train_split_ratio=None,          # training fraction in [0.5, 0.9]
    name=None,
    description=None,
) -> Experiment
client.get_experiment(experiment_id: str) -> Experiment
client.update_experiment(experiment_id: str, **updates) -> Experiment
client.delete_experiment(experiment_id: str) -> None
client.start_experiment(experiment_id: str) -> ExperimentStarted
client.duplicate_experiment(
    experiment_id: str,
    *,
    continue_from_optimized=False,   # True = seed new experiment from source's best optimized prompt
    initial_prompt_override=None,    # Or override the initial prompt with custom text
) -> Experiment                       # Includes ``modelUnavailable`` flag when source's model is gone
```

Each `Experiment` exposes its `aiComponentId` and a dedicated `datasetId`.
Training and eval data are supplied as dataset cases on that dataset (see
[Datasets](#datasets)) — there is no separate experiment-scoped observation
resource.

`start_experiment` raises `PrompticAPIError` with status `402` when platform billing is enabled and the AI Application's organization has no active subscription and payment method, or is blocked by the free-tier limit.

The platform also supports `toolSelection` for optimizing tool descriptions
from MCP or manually supplied definitions. Do not pass it as `task_type` to
`create_experiment(...)`; use `create_tool_selection_experiment(...)` instead.
Returned experiments may include `systemPrompt` and `optimizeSystemPrompt`.

## Evaluators

```python
client.list_evaluators(experiment_id: str) -> EvaluatorList
client.create_evaluators(experiment_id: str, evaluators: list[dict]) -> EvaluatorList
client.update_evaluator(experiment_id: str, evaluator_id: str, **data) -> Evaluator
client.delete_evaluator(experiment_id: str, evaluator_id: str) -> None
```

Evaluator dict format: `{"name": str, "type": "f1"|"referenceJudge"|"comparisonJudge"|"generalJudge"|"similarity"|"structuredOutput", "weight": float, "description": str (optional), "config": dict (optional), "scaleMin": float (optional), "scaleMax": float (optional)}`.

Reading evaluators back via `list_evaluators(...)` for a tool-selection experiment also surfaces the `"toolSelection"` evaluator type, but it is not a value to pass into `create_evaluators(...)` — see `toolSelection` evaluator below.

### Judge evaluator configs

Promptic supports three judge evaluator types. Choose the variant that matches
how the judge should score the prediction against the expected output.

All three judge types accept `scaleMin` / `scaleMax` and require a `config`:

- `referenceJudge` — `config.instructions` (string): rubric text. The judge
  scores predicted and expected outputs independently against the rubric
  (caching the expected-side judgment) and rewards predictions that match or
  exceed the expected score. Best for intrinsic quality rubrics.
- `comparisonJudge` — `config.instructions` (string): rubric text. The judge
  sees predicted and expected together in one prompt and scores how they
  compare. Best for rubrics that relate the two outputs (e.g. structural
  match).
- `generalJudge` — `config.messages` (list of `{role, content}`): full
  multi-message prompt. `role` is `system` / `user` / `assistant`. `content`
  may reference `{input}`, `{expected}`, `{predicted}`, or any dataset
  column name; unknown `{tokens}` are left as-is so misreferences are
  visible in the rendered prompt.

```python
client.create_evaluators(exp_id, [
    {
        "name": "quality",
        "type": "referenceJudge",
        "weight": 1.0,
        "scaleMin": 1,
        "scaleMax": 5,
        "config": {
            "instructions": (
                "Score the answer's factual accuracy. "
                "5 = fully accurate and well-supported; "
                "1 = incorrect or unsupported."
            ),
        },
    },
])
```

### `structuredOutput` evaluator config

Supported `config` keys for the `structuredOutput` type:

- `schema_definition` (dict): JSON schema describing the prediction shape. Drives default per-field scoring — strings → embedding similarity, enums/booleans/integers → exact, numbers → tolerance, nested objects → recursive, arrays → content-aligned soft F1 (not positional).
- `fields` (dict, optional): per-field overrides keyed by dotted JSON path. Each entry accepts:
  - `include` (bool, default `true`)
  - `weight` (float, default `1.0`)
  - `strategy` (string): scalar comparison — `"exact" | "embedding" | "contains" | "judge"`. The `judge` value enables LLM-as-judge per-pair scoring on string fields and surfaces reasoning in the case-details sheet.
  - `array_strategy` (string): array aggregation — `"exact" | "similarity" | "judge"`. The `judge` value runs a single whole-array LLM call returning F1-compatible counts; arrays exceeding 50 items per side fall back to `similarity` with a warning marker.
  - `judge_instructions` (string, optional): field-specific guidance appended to the built-in *"do these convey the same essential information?"* rubric for this field only. Valid only when this field's `strategy` or `array_strategy` is `judge`; setting it on a non-judged field is rejected. Omit to use the built-in rubric on its own — selecting `judge` never requires instructions.

  Whether a field counts as required is read from the JSON schema's `required` array, not from this dict — `FieldConfig` rejects unknown keys.

There is no evaluator-level `judge_instructions`; guidance lives on each judged field, so different fields can carry different notes.

The `embedding` strategy applies a calibrated cosine-similarity floor (`0.15`, tuned for `text-embedding-3-small`) so unrelated string pairs score `0.0` instead of ~`0.55`. Re-running older experiments may show lower scores on string-heavy schemas with unrelated content.

### `toolSelection` evaluator

The `toolSelection` evaluator is **attached automatically when a tool-selection experiment is created** (by `create_tool_selection_experiment(...)` or the dashboard) — it is not user-creatable via `create_evaluators(...)`. It is fixed to a `[0.0, 1.0]` scale, scores `1.0` when the predicted tool name matches `expected` (case-insensitive) and `0.0` otherwise, and takes no `config` keys.

## Tool-selection optimization

In addition to prompt optimization, Promptic can optimize the **tool descriptions** an LLM sees so a downstream model picks the right tool for a given query. Create these experiments with `create_tool_selection_experiment(...)`, which atomically provisions the experiment, its managed dataset, the tool definitions and canonical cases, the system-prompt settings, and the required `toolSelection` evaluator as one pending experiment; call `start_experiment(...)` to run it. The dashboard component-creation wizard offers the same flow and can auto-discover tools from an MCP server.

- **Tools** (`tools`): a list of tool definitions, each `{"name", "description"}` with an optional `input_schema` (also accepted as `inputSchema`); at least one, tool names must be unique and cannot use a reserved no-tool alias. In the dashboard, definitions can also be imported from an MCP server URL (Bearer-token or OAuth 2.0 auth) or pasted as a JSON array; Anthropic-style (`{name, description, input_schema}`), OpenAI-function-calling-style (`{type, function: {name, description, parameters}}`), and plain (`{name, description}`) shapes are all normalized. `tool_source` records the provenance as `"manual"` (default) or `"mcp"`.
- **Test cases** (`test_cases`): each is `{"query", "expected_tool"}` — the user query plus the supplied tool name that should fire. Use `""` as the canonical value for "no tool should be called". The API also accepts `"none"`, `"no tool"`, `"no-tool"`, `"no_tool"`, `"no tool call"`, `"no-tool-call"`, `"no_tool_call"`, `"no tools"`, `"no_tools"`, `"n/a"`, `"na"`, `"-"`, and `"__NO_TOOL__"`; all normalize to the same no-tool expectation. At least one case is required.
- **Optional system prompt**: pass `system_prompt` to use as fixed context during evaluation. When `optimize_system_prompt=True`, the optimizer rewrites it alongside the tool descriptions. Read the best variant from `get_best_iteration(...)['selectionSystemPrompt']`; optimized descriptions are returned by the same iteration under `toolDescriptions`.
- **Other options**: `target_model` (omit for the platform default), `epochs` (1–5), `train_split_ratio` (fraction assigned to training in `[0.5, 0.9]`; the remaining cases form the held-out evaluation split; omit to train and score on all cases), `name`, and `description`.

Existing tool-selection experiments, dataset cases, iterations, and evaluators
come back through the normal SDK methods with `taskType: "toolSelection"`.

## Iterations

```python
client.list_iterations(experiment_id: str) -> IterationList
client.get_iteration(experiment_id: str, iteration_id: int) -> IterationWithScores
client.get_best_iteration(experiment_id: str) -> IterationWithScores
```

Iterations report two scores: `overallNormalizedScore` (train split, used to guide the search) and `evalNormalizedScore` (held-out eval split, `None` when `trainSplitRatio` is not configured on the experiment). `get_best_iteration` ranks by `evalNormalizedScore` when available, otherwise by `overallNormalizedScore`.

Iterations also expose `avgPredictionLatencyMs` — the mean wall-clock duration (milliseconds) of the target-model prediction calls in that iteration, averaged across train + eval predictions. It excludes retries, rate-limit backoff, and failed attempts, so it reflects real per-call response time. The key is **absent (or `None`)** on iterations completed before per-prediction latency tracking shipped, so use `.get("avgPredictionLatencyMs")` rather than direct subscript when handling legacy iterations. Useful for comparing two equally-scoring iterations on speed.

Tool-selection iterations may additionally expose `toolDescriptions`, keyed by
tool name, and `selectionSystemPrompt`, the system prompt used for that
iteration. Both fields can be absent or `None` for historical iterations and
other task types. `promptic iterations get` and `promptic iterations best`
display these outputs; `--json` returns the complete response.

## Deployments

```python
client.get_deployment(component_id: str) -> Deployment | None
client.deploy(component_id: str, experiment_id: str) -> DeploymentCreated
client.undeploy(component_id: str) -> None
client.get_deployed_prompt(component_id: str) -> DeployedPrompt | None
```

`DeployedPrompt` fields: `prompt`, `model`, `provider`, `componentId`, `componentName`, `experimentId`, `iterationId`, `score`, `schemaSnapshot`.

## Datasets

```python
client.create_dataset(component_id: str, name: str, *, description=None, trace_ids=None) -> Dataset
client.list_datasets(component_id: str) -> DatasetList
client.get_dataset(component_id: str, dataset_id: str) -> DatasetWithCases
client.delete_dataset(component_id: str, dataset_id: str) -> None
client.list_dataset_cases(component_id: str, dataset_id: str) -> DatasetCaseList
client.create_dataset_cases(component_id: str, dataset_id: str, cases: DatasetCaseInput | list[DatasetCaseInput]) -> DatasetCaseList
client.get_dataset_case(component_id: str, dataset_id: str, case_id: int) -> DatasetCase
client.update_dataset_case(component_id: str, dataset_id: str, case_id: int, updates: DatasetCaseInput) -> DatasetCase
client.delete_dataset_case(component_id: str, dataset_id: str, case_id: int) -> None
```

`Dataset` reports a `caseCount`; `DatasetWithCases` includes the full `cases` list.
