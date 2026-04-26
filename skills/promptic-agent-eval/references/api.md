# API Reference — Agent Evaluation

Method signatures for `PrompticClient` and `AsyncPrompticClient` used by
the `promptic-agent-eval` skill.

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

`AgentEvaluation` status: `"pending" | "running" | "completed" | "failed"`.
The `results` field contains `InsightResult` with `insights` list and
`meta` object.
