# API Reference — Prompt Optimization

Method signatures for `PrompticClient` and `AsyncPrompticClient` used by
the `promptic-optimize` skill.

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

Observation dict format:
`{"variables": dict[str, Any], "expected": str, "split": str (optional, default "eval")}`.

## Evaluators

```python
client.list_evaluators(experiment_id: str) -> EvaluatorList
client.create_evaluators(experiment_id: str, evaluators: list[dict]) -> EvaluatorList
client.update_evaluator(experiment_id: str, evaluator_id: str, **data) -> Evaluator
client.delete_evaluator(experiment_id: str, evaluator_id: str) -> None
```

Evaluator dict format:
`{"name": str, "type": "f1"|"judge"|"similarity"|"structuredOutput", "weight": float, "description": str (optional), "config": dict (optional)}`.

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

`DeployedPrompt` fields: `prompt`, `model`, `provider`, `componentId`,
`componentName`, `experimentId`, `iterationId`, `score`, `schemaSnapshot`.
