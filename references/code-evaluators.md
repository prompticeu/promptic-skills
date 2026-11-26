# Python and JavaScript code evaluators

Use `codeFunction` for custom deterministic scoring in **prompt experiments**,
and the `CodeFunction` evaluator (CLI kind `code_function`) in **Agent
Optimization benchmarks**; the two receive different rows (see
[Agent Optimization benchmarks](#agent-optimization-benchmarks)). Do not use a
code function as an agent implementation or as a way to run arbitrary tools.
Prefer built-in exact/contains checks when they already express the requirement.

Requires a platform deployment containing code evaluator support (PRO-201).
The SDK's existing sync/async `create_evaluators()` and `update_evaluator()`
methods accept the configuration dictionary. The `CodeFunctionConfig` type and
CLI source-file flags require the SDK/CLI release that adds code evaluators;
check `promptic evaluators add --help` before assuming those flags are installed.
Never promise that an open PR has already reached the hosted service or PyPI.

## Create via the SDK

```python
from pathlib import Path
from promptic_sdk import PrompticClient

with PrompticClient() as client:
    # Use a pending textGeneration or structuredOutput experiment.
    created = client.create_evaluators(experiment_id, [{
        "name": "Python exact answer",
        "type": "codeFunction",
        "scaleMin": 0,
        "scaleMax": 1,
        "weight": 1,
        "config": {
            "language": "python",  # or "javascript" with a JS source file
            "source": Path("score.py").read_text(encoding="utf-8"),
            "function_name": "score",  # optional; the name is fixed
        },
    }])
    evaluator_id = created["data"][0]["id"]
```

Update a pending evaluator with `client.update_evaluator(experiment_id,
evaluator_id, config={"language": "javascript", "source": js_source})`.
Always send both `language` and `source` when replacing `config`. List with
`client.list_evaluators(experiment_id)`. Sync and async clients use the same
payload; await async calls. The optional `CodeFunctionConfig` TypedDict provides
the config shape but does not execute or validate source locally.

## Create via the CLI

```bash
promptic evaluators add EXPERIMENT_ID --name "Python exact answer" --type codeFunction \
  --language python --source-file score.py --json
promptic evaluators add EXPERIMENT_ID --name "JavaScript exact answer" --type codeFunction \
  --language javascript --source-file score.js --json
```

The CLI reads UTF-8 text and submits it; it never imports or executes the file.
Both flags are required for `codeFunction` and rejected on other types.
TypeScript is not supported. No function-name or per-evaluator timeout option
exists. `--weight`, `--scale-min`, `--scale-max`, and `--disabled` also apply.

## Function contract

Python (`score.py`):

```python
def score(row):
    matches = row["expected"].strip() == row["predicted"].strip()
    return {"score": 1.0 if matches else 0.0, "reason": "Exact comparison"}
```

JavaScript (`score.js`):

```javascript
function score(row) {
  return row.expected.trim() === row.predicted.trim() ? 1 : 0;
}
```

Return a finite numeric raw score, or an object with a finite numeric `score`
and an optional short string `reason`. Booleans, NaN, and Infinity are invalid.
No result `metadata` is supported. Match `scaleMin`/`scaleMax` to raw scores;
weight controls the contribution to the combined score.

| Row key | Meaning |
| --- | --- |
| `id` | Case identifier; UUIDs are strings |
| `input_variables` | Original named input object |
| `expected` | Reference output |
| `predicted` | Model output |

Structured outputs arrive as text, not pre-parsed objects. Use only the
restricted language operations supported by the platform. Do not suggest
imports, external API calls, package installation, file reads/writes,
subprocesses, or accessing environment/provider credentials. Source is limited
to 64 KiB of UTF-8. Each batch has a five-second execution limit. Code runs in
the platform's configured isolated sandbox, not in the SDK process; don't
assume a particular cloud provider from the client. Sandbox usage may consume
credits even when the function fails after execution starts. Keep secrets and
unnecessary personal data out of source and diagnostic strings.

In Python the row is a dictionary: use `row["id"]` and
`row["input_variables"]["question"]`, not `row.id`. JavaScript uses `row.id` and
`row.input_variables.question`. Row contract version 3 contains only the four
fields above. It replaces the old `input` and `variables` names and removes
flattened input and miscellaneous prediction metadata; update existing source explicitly. Do not
silently rewrite historical results or promise backwards-compatible aliases.

Invalid source, a missing `score`, runtime errors, invalid return values, and
timeouts fail the evaluation rather than producing minimum quality scores.
Diagnostics expose safe error codes, exception types and source positions when
available, not arbitrary exception messages or full stacks. Stop and fix the
evaluator; do not optimize prompts in response to an evaluator execution error.

## Test the function first

The matching SDK/CLI and server release adds a sandbox probe accepting one
explicit sample prediction. This is the supported way to calibrate a function
without running a model, saving an evaluator, or creating quality scores.

```python
probe = client.test_code_evaluator(
    {"language": "python", "source": Path("score.py").read_text(encoding="utf-8")},
    {"input_variables": {"question": "Say hello"}, "expected": "Hello", "predicted": "Hello"},
)
state = client.get_code_evaluator_test(probe["id"])
```

Poll `get_code_evaluator_test()` every 2 seconds for at most 90 seconds, stopping
at `completed` or `failed`. If the deadline expires, report the test ID and last
status, return control, and do not submit a replacement probe or imply it was
cancelled. `result` contains `score`/`reason` or safe
`error`, `error_type`, `line`, `column`; `errorMessage` is the readable failure.
Repeat with a deliberately incorrect `predicted` value to check the low-score
path. Keep distinct predictions in distinct probe jobs. `CodeFunctionTestRow`
is exported for typed samples. Async clients use the same methods with `await`.

```bash
promptic evaluators test --language python --source-file score.py --row-file row.json --json
promptic evaluators test-result TEST_ID --json
```

`row.json` has `input_variables`, `expected`, and `predicted`; only `id` is optional.
JavaScript uses `--language javascript`. Files must be UTF-8 and each at most
64 KiB in the CLI; the API request limit is 128 KiB. Tests use sandbox compute
and may consume credits. A client timeout or Ctrl-C does not cancel remote
execution. No secrets belong in samples or source. The dashboard offers the
same **Test function** action with language-specific row help and source-line
error markers. Editing the source clears its displayed result, not the remote job.

## Run and inspect

1. Add representative case inputs and known expected answers. Predictions are
   generated by the target model, not uploaded with dataset cases. To calibrate
   an exact-answer rule, use a small dataset whose expected answer is `OK` and
   run two evaluation-only experiments: a prompt asking for exactly `OK`, and
   a prompt asking for exactly `WRONG`. Inspect the actual predictions and
   confirm the rule rewards the first and rejects the second. Saving source
   alone does not verify it.
2. For a candidate supplied by a coding agent, use evaluation-only mode
   (`hyperparameters={"epochs": 0}` on creation) rather than automatically
   optimizing or deploying it. Use `gpt-5.6-luna` for inexpensive test experiments.
3. Explicitly call `client.start_experiment(experiment_id)` and poll
   `client.get_experiment(experiment_id)` every 5 seconds for at most 10 minutes,
   stopping at `completed`, `failed`, or `canceled`. Use a monotonic deadline,
   as in `references/prompt-evaluation.md`. If the deadline expires, report the
   experiment ID and latest status and return control. Do not restart, cancel,
   or claim completion merely because local polling timed out. Starting executes
   model calls and sandbox work and can consume credits.
4. Call `client.list_iterations(experiment_id)`, then use the **persisted**
   iteration ID (not its number) with
   `client.list_experiment_cases(experiment_id, iteration_id=iteration_id)`.
   Follow `nextCursor` with identical sort/filters to inspect further pages.
   Evaluation-only iteration number is zero; its database ID is not zero.
5. Check each case's `evaluatorScores`, not only the aggregate. Entries include
   `rawScore`, normalized `score`, and `additional.reason` on success. Function
   execution errors fail the run and must not be treated as low quality scores.
   The dashboard Data tab and case details show scores and diagnostic reasons.
   Do not claim live managed sandbox execution or billing was verified from a
   local Docker run or mocked SDK transport tests.

Equivalent result commands:

```bash
promptic experiments start EXPERIMENT_ID
promptic experiments get EXPERIMENT_ID --json
promptic iterations list EXPERIMENT_ID --json
promptic experiments cases EXPERIMENT_ID --iteration-id ITERATION_ID --json
```

## Agent Optimization benchmarks

Benchmarks use the typed `CodeFunction(source, language="python", name=...,
weight=1, required=False, scale_min=0, scale_max=1)` in `evaluators=[...]`, or
a `code_function` entry in `agent.json`; see the Code Function section of
[agent-benchmark.md](agent-benchmark.md) for when to choose it and full
examples. It is repeatable (up to 16 per benchmark, unique names) and has no
`threshold`. Requires `promptic-sdk` 1.9.0 or later: check that
`from promptic_sdk import CodeFunction` works first.

The row differs from experiments and holds native JSON values:

| Row key | Meaning |
| --- | --- |
| `id` | Test case identifier |
| `inputs` | Case inputs, an object keyed by input name |
| `output` | Submitted Agent output, shaped by the Output schema |
| `expected_output` | Expected Output object, or `None`/`null` |
| `expected_behavior` | Expected Behavior text, or `None`/`null` |

```python
def score(row):
    output = row["output"] or {}
    expected = row["expected_output"] or {}
    return 1.0 if output.get("invoice_id") == expected.get("invoice_id") else 0.0
```

Failure handling also differs: a runtime error, invalid return value or timeout
fails only that case (with a safe error code, exception type and line), and the
remaining cases still score. Invalid source or a missing `score` fails every
case of that evaluator. A failed case counts as unscored, not as a low score;
with `required=True` it marks the run ineligible. Fix the function and
re-evaluate existing runs; the submissions do not need to be rerun. When the
Agent has an Output schema, external submissions are rejected at upload
(`prediction_contract_invalid`) unless each successful output is an object that
matches it. Without an Output schema, `output` is any JSON value (object, list,
string, number, or null). Read it defensively either way (for example
`row["output"] or {}`), so an unexpected shape fails only that case.

Probe a benchmark function with the same test endpoint and the Agent
Optimization row contract:

```python
probe = client.test_code_evaluator(
    {"language": "python", "source": Path("checks/line_items.py").read_text(encoding="utf-8")},
    {"inputs": {"invoice_text": "..."}, "output": {"total": 7}, "expected_output": {"total": 7}},
    row_contract="agent_gym",
)
state = client.get_code_evaluator_test(probe["id"])
```

```bash
promptic evaluators test --language python --source-file checks/line_items.py \
  --row-file sample-case.json --row-contract agent_gym --json
```

`AgentGymCodeFunctionTestRow` is exported for typed samples; `expected_output`,
`expected_behavior`, and `id` are optional. Polling, limits and credit use are
the same as for experiment probes above. The dashboard's **Test function**
action prefills the sample from the benchmark's first case.
