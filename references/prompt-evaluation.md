# Evaluate locally authored prompt candidates

Use this workflow when a coding agent writes prompts locally and Promptic should
execute and score them. Promptic supplies the target-model calls, configured
evaluators, saved predictions, scores, latency and usage. This differs from Agent
Optimization, where external code executes the Agent and submits its outputs.

Requires a server release with evaluation-only support. `evaluation_only=True`,
the CLI flags, and `list_experiment_cases` require the corresponding SDK release.
Older clients can configure `hyperparameters={"epochs": 0, "numFewShots": 8}`;
they cannot retrieve case-result pages from servers without that endpoint.

## Reuse a configured evaluation setup

Keep one configured reference experiment with representative cases, evaluators,
target model, output schema, input mapping and model settings. Duplicate it for
each candidate so the reference and previous results remain available. A duplicate
copies cases and evaluators; it is not an immutable benchmark revision. Keep the
setup unchanged during a comparison and record the reference ID and local Git
commit in the candidate's name or description.

```python
import time
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    candidate = client.duplicate_experiment(
        reference_experiment_id,
        initial_prompt_override="Classify the request: {input}",
    )
    candidate = client.update_experiment(
        candidate["id"],
        name="Candidate abc123",
        description=f"Reference: {reference_experiment_id}; Git commit: abc123",
        hyperparameters={
            "numFewShots": 8,
            **candidate["hyperparameters"],
            "epochs": 0,
        },
    )
    client.start_experiment(candidate["id"])

    deadline = time.monotonic() + 300
    while True:
        run = client.get_experiment(candidate["id"])
        if run["experimentStatus"] in {"completed", "failed", "canceled"}:
            break
        if time.monotonic() >= deadline:
            raise TimeoutError(f"Run {candidate['id']} is still {run['experimentStatus']}")
        time.sleep(2)
    if run["experimentStatus"] != "completed":
        raise RuntimeError(run.get("errorMessage") or run["experimentStatus"])

    iteration = client.get_best_iteration(candidate["id"])
    assert iteration["iterationNumber"] == 0
    page = client.list_experiment_cases(
        candidate["id"], iteration_id=iteration["id"],
        split="all", sort="score", limit=25,
    )
    for case in page["data"]:
        print(case["variables"], case["expected"], case.get("prediction"),
              case.get("score"), case.get("evaluatorScores"),
              case.get("predictionAdditional"))
```

The iteration **number** is `0`; the persisted iteration **ID** is a separate
positive integer. Pass the ID to `get_iteration` and `list_experiment_cases`.
A zero score is a real result. Check prediction diagnostics before treating a
failure or missing prediction as an ordinary incorrect answer. Report failed,
canceled or still-running status rather than claiming a completed evaluation.
A polling timeout does not cancel remote execution.

Case sorting applies across the selected dataset before pagination. Follow
`nextCursor` with the same iteration, split and sort when more cases are needed.
`total` covers the whole selected split, not just the returned page. Use the
saved iteration's aggregate scores for run comparisons; do not average a partial
page and present it as the full-dataset score. With a train/eval split, compare
the same split across candidates. Reserve an untouched holdout if the agent is
repeatedly optimizing against feedback.

## Create a new evaluation setup

```python
experiment = client.create_experiment(
    component_id, "gpt-5.6-luna",
    task_type="classification",
    initial_prompt="Classify the request: {input}",
    evaluation_only=True,
)
# Add cases to experiment["datasetId"] and configure evaluators before starting.
```

For a chat template, pass `prompt_format="multi_message"` and ordered
`initial_prompt_messages=[{"role": "system", "content": ...}, ...]` at creation.
For an existing pending candidate, update `promptFormat` and
`initialPromptMessages`. Do not use the text-only duplicate override to replace
a multi-message template. Preserve roles, order, input placeholders and the
output schema. Evaluation-only uses the normal task prediction contract; it
does not execute arbitrary local inference code.

`evaluation_only=True` conflicts with an explicitly nonzero `epochs` budget.
Raw `epochs=0` remains supported. For tool selection, use
`create_tool_selection_experiment(..., evaluation_only=True)` or `epochs=0`.
These runs still incur applicable inference, evaluator and platform usage
charges; they skip the successful optimization fee.

Both sync and async clients support these arguments. Await the async calls and
use `asyncio.sleep` for bounded polling in async code.

## CLI

```bash
promptic experiments duplicate REFERENCE_ID --initial-prompt 'New candidate {input}' --evaluate-only --json
promptic experiments start CANDIDATE_ID --json
promptic experiments get CANDIDATE_ID --json
promptic iterations list CANDIDATE_ID --json
promptic experiments cases CANDIDATE_ID --iteration-id ITERATION_ID --sort score --limit 25 --json
# Fetch another page only when needed, retaining the same sort and split:
promptic experiments cases CANDIDATE_ID --iteration-id ITERATION_ID --sort score --cursor CURSOR --json
```

`duplicate --evaluate-only --start` combines candidate creation, evaluation-only
configuration and explicit scheduling. If configuration fails, the clone may
remain pending; inspect existing candidates before retrying. For new setups,
use `experiments create --evaluate-only` with `--initial-prompt` or a `--messages`
JSON file. `experiments update ID --epochs 0` preserves other hyperparameters.
Tool-selection creation accepts `--epochs 0` or `--evaluate-only`.

Authentication must target the correct endpoint and AI Application. API keys are
bound to an AI Application; selecting another application does not retarget the
key. Use a matching key or login credentials with an explicit AI Application.
Keep the normal start endpoint's billing and scheduling checks. Preparation
does not itself authorize paid runs or deployment; retain authorization already
provided by the user. Do not deploy evaluation candidates unless requested.
