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

## Observations

```python
client.list_observations(experiment_id: str) -> ObservationList
client.create_observations(experiment_id: str, observations: list[dict]) -> ObservationList
client.update_observation(experiment_id: str, observation_id: int, **data) -> Observation
client.delete_observation(experiment_id: str, observation_id: int) -> None
```

Observation dict format: `{"variables": dict[str, Any], "expected": str, "split": str (optional, default "eval")}`.

## Evaluators

```python
client.list_evaluators(experiment_id: str) -> EvaluatorList
client.create_evaluators(experiment_id: str, evaluators: list[dict]) -> EvaluatorList
client.update_evaluator(experiment_id: str, evaluator_id: str, **data) -> Evaluator
client.delete_evaluator(experiment_id: str, evaluator_id: str) -> None
```

Evaluator dict format: `{"name": str, "type": "f1"|"referenceJudge"|"comparisonJudge"|"generalJudge"|"similarity"|"structuredOutput", "weight": float, "description": str (optional), "config": dict (optional), "scaleMin": float (optional), "scaleMax": float (optional)}`.

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

  Whether a field counts as required is read from the JSON schema's `required` array, not from this dict — `FieldConfig` rejects unknown keys.

- `judge_instructions` (string, optional): domain-specific guidance shared by every field configured with `strategy=judge` or `array_strategy=judge`. Appended to the built-in *"do these convey the same essential information?"* rubric — leave unset to use the rubric on its own.

The `embedding` strategy applies a calibrated cosine-similarity floor (`0.15`, tuned for `text-embedding-3-small`) so unrelated string pairs score `0.0` instead of ~`0.55`. Re-running older experiments may show lower scores on string-heavy schemas with unrelated content.

## Iterations

```python
client.list_iterations(experiment_id: str) -> IterationList
client.get_iteration(experiment_id: str, iteration_id: int) -> IterationWithScores
client.get_best_iteration(experiment_id: str) -> IterationWithScores
```

Iterations report two scores: `overallNormalizedScore` (train split, used to guide the search) and `evalNormalizedScore` (held-out eval split, `None` when `trainSplitRatio` is not configured on the experiment). `get_best_iteration` ranks by `evalNormalizedScore` when available, otherwise by `overallNormalizedScore`.

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
