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
promptic_sdk.artifact(value: str | bytes | os.PathLike[str], *, mime_type=None, api_key=None, endpoint=None) -> ArtifactReference
```

Use `promptic_sdk.artifact(...)` when custom code needs to attach local files,
bytes, or large text payloads to traces explicitly. The returned reference can be
stored in a span attribute, for example `span.set_attribute("input_file",
ref.ref)`. The SDK prefers direct object-storage upload via Promptic's presign
API and only falls back to server-side base64 upload for compatibility.

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
    train_split_ratio=None,          # held-out eval split in [0.5, 0.9]
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

`start_experiment` raises `PrompticAPIError` with status `402` when platform billing is enabled and the AI Application's organization has no active subscription and payment method, or is blocked by the free-tier limit.

The platform also supports a fourth task type, `toolSelection`, for tool-description optimization (MCP servers and hand-authored tool definitions). It is **not a valid `task_type` for `create_experiment`** — create these experiments with the dedicated `create_tool_selection_experiment(...)` method (see below), and treat `taskType: "toolSelection"` as a read-only value surfaced by `list_experiments(...)` / `get_experiment(...)` for existing ones. Such experiments also surface three optional fields on the `Experiment` record: `systemPrompt` (the fixed system prompt used as context during evaluation, may be `None`), `optimizeSystemPrompt` (boolean — whether the optimizer is also rewriting the system prompt), and `optimizedSystemPrompt` (the best system-prompt variant the optimizer settled on, populated only when the toggle was on).

## Observations

```python
client.list_observations(experiment_id: str) -> ObservationList
client.create_observations(experiment_id: str, observations: list[dict]) -> ObservationList
client.update_observation(experiment_id: str, observation_id: int, **data) -> Observation
client.delete_observation(experiment_id: str, observation_id: int) -> None
```

Observation dict format: `{"variables": dict[str, Any], "expected": str, "split": str (optional, default "eval")}`.

For `toolSelection` experiments returned by the API, `variables` carries the user query (e.g. `{"input": "..."}`) and `expected` is the tool name that should be selected — or the empty string `""` when the query should not trigger any tool.

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
  - `strategy` (string): scalar comparison — `"exact" | "embedding" | "contains" | "judge"`. The `judge` value enables LLM-as-judge per-pair scoring on string fields and surfaces reasoning in the observation-details sheet.
  - `array_strategy` (string): array aggregation — `"exact" | "similarity" | "judge"`. The `judge` value runs a single whole-array LLM call returning F1-compatible counts; arrays exceeding 50 items per side fall back to `similarity` with a warning marker.
  - `judge_instructions` (string, optional): field-specific guidance appended to the built-in *"do these convey the same essential information?"* rubric for this field only. Valid only when this field's `strategy` or `array_strategy` is `judge`; setting it on a non-judged field is rejected. Omit to use the built-in rubric on its own — selecting `judge` never requires instructions.

  Whether a field counts as required is read from the JSON schema's `required` array, not from this dict — `FieldConfig` rejects unknown keys.

There is no evaluator-level `judge_instructions`; guidance lives on each judged field, so different fields can carry different notes.

The `embedding` strategy applies a calibrated cosine-similarity floor (`0.15`, tuned for `text-embedding-3-small`) so unrelated string pairs score `0.0` instead of ~`0.55`. Re-running older experiments may show lower scores on string-heavy schemas with unrelated content.

### `toolSelection` evaluator

The `toolSelection` evaluator is **attached automatically when a tool-selection experiment is created** (by `create_tool_selection_experiment(...)` or the dashboard) — it is not user-creatable via `create_evaluators(...)`. It is fixed to a `[0.0, 1.0]` scale, scores `1.0` when the predicted tool name matches `expected` (case-insensitive) and `0.0` otherwise, and takes no `config` keys.

## Tool-selection optimization

In addition to prompt optimization, Promptic can optimize the **tool descriptions** an LLM sees so a downstream model picks the right tool for a given query. Create these experiments with `create_tool_selection_experiment(...)`, which atomically provisions the experiment, its managed dataset, the tool definitions and canonical cases, the system-prompt settings, and the required `toolSelection` evaluator as one pending experiment; call `start_experiment(...)` to run it. The dashboard component-creation wizard offers the same flow, and is also where you auto-discover tools from an MCP server and review the optimized descriptions (which are not returned through the public API).

- **Tools** (`tools`): a list of tool definitions, each `{"name", "description"}` with an optional `input_schema` (also accepted as `inputSchema`); at least one, tool names must be unique and cannot use a reserved no-tool alias. In the dashboard, definitions can also be imported from an MCP server URL (Bearer-token or OAuth 2.0 auth) or pasted as a JSON array; Anthropic-style (`{name, description, input_schema}`), OpenAI-function-calling-style (`{type, function: {name, description, parameters}}`), and plain (`{name, description}`) shapes are all normalized. `tool_source` records the provenance as `"manual"` (default) or `"mcp"`.
- **Test cases** (`test_cases`): each is `{"query", "expected_tool"}` — the user query plus the tool that should fire, or `""` (or a supported no-tool alias) for "no tool should be called". Each `expected_tool` must match one of the supplied tool names. At least one.
- **Optional system prompt**: pass `system_prompt` to use as fixed context during evaluation. When `optimize_system_prompt=True`, the optimizer rewrites it alongside the tool descriptions and persists the best variant on the experiment as `optimizedSystemPrompt`.
- **Other options**: `target_model` (omit for the platform default), `epochs` (1–5), `train_split_ratio` (held-out eval split in `[0.5, 0.9]`; omit to score on all cases), `name`, and `description`.

Existing tool-selection experiments and their iterations / observations / evaluators also come back through the normal SDK methods with `taskType: "toolSelection"`.

## Iterations

```python
client.list_iterations(experiment_id: str) -> IterationList
client.get_iteration(experiment_id: str, iteration_id: int) -> IterationWithScores
client.get_best_iteration(experiment_id: str) -> IterationWithScores
```

Iterations report two scores: `overallNormalizedScore` (train split, used to guide the search) and `evalNormalizedScore` (held-out eval split, `None` when `trainSplitRatio` is not configured on the experiment). `get_best_iteration` ranks by `evalNormalizedScore` when available, otherwise by `overallNormalizedScore`.

Iterations also expose `avgPredictionLatencyMs` — the mean wall-clock duration (milliseconds) of the target-model prediction calls in that iteration, averaged across train + eval predictions. It excludes retries, rate-limit backoff, and failed attempts, so it reflects real per-call response time. The key is **absent (or `None`)** on iterations completed before per-prediction latency tracking shipped, so use `.get("avgPredictionLatencyMs")` rather than direct subscript when handling legacy iterations. Useful for comparing two equally-scoring iterations on speed.

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
client.get_dataset(component_id: str, dataset_id: str) -> DatasetWithItems
client.delete_dataset(component_id: str, dataset_id: str) -> None
```

## Runs

```python
client.create_run(component_id: str, dataset_id: str, *, name=None, trace_ids=None) -> Run
client.list_runs(component_id: str) -> RunList
client.get_run(component_id: str, run_id: str) -> RunWithTraces
client.delete_run(component_id: str, run_id: str) -> None
```

## Annotations

```python
client.upsert_annotation(component_id: str, run_id: str, trace_db_id: str, *, rating=None, comment=None) -> Annotation
client.list_annotations(component_id: str, run_id: str) -> AnnotationList
client.list_dataset_annotations(component_id: str, dataset_id: str) -> AnnotationList
client.delete_annotation(component_id: str, run_id: str, annotation_id: str) -> None
```

- `rating`: `"positive"` or `"negative"`

## Agent Evaluations

```python
client.create_evaluation(component_id: str, dataset_id: str, *, name=None, run_id=None) -> AgentEvaluation
client.list_evaluations(component_id: str) -> AgentEvaluationList
client.get_evaluation(component_id: str, evaluation_id: str) -> AgentEvaluation
client.wait_for_evaluation(component_id: str, evaluation_id: str, *, max_wait=300, poll_interval=2) -> AgentEvaluation
```

`create_evaluation` raises `PrompticAPIError` with status `402` under the same billing conditions as `start_experiment` (active subscription and payment method required, or free-tier limit) when the evaluation uses platform-managed judges.

`AgentEvaluation` status: `"pending" | "running" | "completed" | "failed"`. The `results` field contains `InsightResult` with `insights` (heuristic findings), `judgeResults` (per-judge results from predefined trajectory critics + custom rubrics), and `meta` (aggregate stats). See the example in `SKILL.md` for iteration patterns.
