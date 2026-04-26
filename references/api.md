# API Reference

Complete method signatures for `PrompticClient` and `AsyncPrompticClient`. Both clients share identical signatures — async methods are prefixed with `await`.

## Traces

```python
client.list_traces(*, limit=50, offset=0, status=None, start_after=None, start_before=None) -> TraceList
client.get_trace(trace_id: str) -> Trace
client.get_stats(*, days_back=30) -> TracingStats
```

- `status`: `"ok"` or `"error"`
- `start_after` / `start_before`: ISO timestamp strings

## Workspace

```python
client.get_workspace() -> Workspace
```

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
    optimizer="prompticV2",          # "promptic" | "prompticV2" | "miproV2" | "bootstrapFewShot" | "gepa"
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

Evaluator dict format: `{"name": str, "type": "f1"|"judge"|"similarity"|"structuredOutput", "weight": float, "description": str (optional), "config": dict (optional)}`.

### `structuredOutput` evaluator config

Supported `config` keys for the `structuredOutput` type:

- `schema_definition` (dict): JSON schema describing the prediction shape. Drives default per-field scoring — strings → embedding similarity, enums/booleans/integers → exact, numbers → tolerance, nested objects → recursive, arrays → content-aligned soft F1 (not positional).
- `fields` (dict, optional): per-field overrides keyed by dotted JSON path. Each entry accepts:
  - `include` (bool, default `true`)
  - `weight` (float, default `1.0`)
  - `strategy` (string): scalar comparison — `"exact" | "embedding" | "contains" | "judge"`. The `judge` value enables LLM-as-judge per-pair scoring on string fields and surfaces reasoning in the observation-details sheet.
  - `array_strategy` (string): array aggregation — `"exact" | "similarity" | "judge"`. The `judge` value runs a single whole-array LLM call returning F1-compatible counts; arrays exceeding 50 items per side fall back to `similarity` with a warning marker.
  - `required` (bool): defaults from the schema's `required` arrays.
- `judge_instructions` (string, optional): domain-specific guidance shared by every field configured with `strategy=judge` or `array_strategy=judge`. Appended to the built-in *"do these convey the same essential information?"* rubric — leave unset to use the rubric on its own.

The `embedding` strategy applies a calibrated cosine-similarity floor (`0.15`, tuned for `text-embedding-3-small`) so unrelated string pairs score `0.0` instead of ~`0.55`. Re-running older experiments may show lower scores on string-heavy schemas with unrelated content.

## Iterations

```python
client.list_iterations(experiment_id: str) -> IterationList
client.get_iteration(experiment_id: str, iteration_id: int) -> IterationWithScores
client.get_best_iteration(experiment_id: str) -> IterationWithScores
```

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

`AgentEvaluation` status: `"pending" | "running" | "completed" | "failed"`. The `results` field contains `InsightResult` with `insights` list and `meta` object.
