# Author an Agent benchmark

Use this reference when the user needs to create or revise the benchmark used
by Agent Optimization. A benchmark is the stable test contract shared by every
variant: the Agent goal, input and output contracts, representative cases, and
evaluation plan.

Design the benchmark before implementing variants. During an optimization
cycle, keep its published version fixed so a score change reflects the Agent
architecture rather than a changed test.

## Define the Agent contract

Write the **goal** as the outcome the Agent must accomplish and the important
constraints on a successful result. Do not prescribe one implementation when
different architectures should be allowed to solve the task differently.

The **Input schema** is the Agent's request contract. It defines the named data
and files every variant receives. The **Output schema** is its result contract.
It defines structured values and generated-file fields that a successful
variant may return. Both are JSON object schemas.

Use `"x-promptic-type": "file"` on a schema field containing files. Declare a
generated artifact in the Output schema and return it with the same field path;
do not treat output files as unrelated attachments.

Only add an Output schema when the result has a meaningful stable structure.
Do not invent fields merely to make a subjective task look deterministic.

## Create representative cases

Cases should cover normal requests, difficult edge cases, and failures that
matter to the user. Include enough variation to distinguish architectures,
not only examples the current implementation already handles.

Each case can contain:

- `input`: the concrete values and files supplied to the Agent;
- `output`: the expected structured values or files, when there is a useful
  canonical answer; and
- `expected_behavior`: case-specific prose describing what a successful Agent
  should do when exact expected output is insufficient.

Use `expected_behavior` for case-specific requirements such as which ambiguity
to resolve, what must be cited, which constraint matters, or what safe behavior
is expected. It is especially useful when several outputs can be valid. Keep
the shared scoring policy in the evaluator instructions instead of copying the
same rubric into every case. Expected output and expected behavior can coexist.

Never expose expected output or expected behavior to the Agent callback. They
are evaluation evidence, not runtime input.

Neither field creates a score by itself. The configured evaluators determine
which expected values and behaviors they inspect and how those become scores.
Likewise, a schema validates the contract; it is not an evaluation plan.

## Choose evaluators

Use the simplest evaluator that measures the intended outcome reliably.

| Evaluator | Use it when | Required setup |
| --- | --- | --- |
| `ClassificationF1` | One or more Output-schema enum fields contain categorical labels and class-level precision/recall matters. | List the enum field paths. Cases need expected values for them. |
| `FieldLevelJudge` | The Output schema has fields that should be compared independently. | Configure each included field with `exact`, `contains`, `embedding`, or `judge`; configure arrays with `exact`, `similarity`, or `judge`. Add field-specific judge instructions only for judged fields. |
| `VerifierAgent` | Success requires the most flexible, holistic inspection of outputs, generated files, case requirements, or traces. | Define explicit metrics, instructions for each metric, allowed evidence, tools, and an investigation budget. |
| `TrajectoryJudge` | Tool selection, execution order, retries, or other trace behavior is itself part of correctness. | Supply a focused rubric and ensure variants submit trace IDs. |

Practical rules:

- Prefer deterministic field scoring where equality or containment really
  expresses correctness.
- Use semantic or judge scoring only for fields where exact comparison would
  reject valid answers.
- Use `VerifierAgent` for artifact-heavy or open-ended work because it can
  investigate the selected evidence as a whole. Its metrics should name the
  distinct qualities the user actually cares about, such as correctness and
  completeness.
- Use `TrajectoryJudge` only when the path matters, not merely because traces
  happen to exist. Prefer outcome evaluation when different workflows may be
  equally valid.
- Combine evaluators when they measure distinct dimensions. Avoid scoring the
  same property twice under different names.

Weights control contribution to the combined score. A threshold expresses the
minimum acceptable normalized score for an evaluator or verifier metric. Leave
thresholds unset until there is a defensible acceptance boundary; do not choose
one from gut feeling. Calibrate evaluators against a small set of manually
reviewed cases before trusting leaderboard order.

Common typed configurations look like this:

```python
from promptic_sdk import (
    ClassificationF1,
    FieldLevelJudge,
    FieldScoring,
    TrajectoryJudge,
)

classification = ClassificationF1(field_paths=("currency",))

structured = FieldLevelJudge(
    fields={
        "invoice_number": FieldScoring("exact", weight=2),
        "summary": FieldScoring(
            "judge",
            judge_instructions="Check factual equivalence, not writing style.",
        ),
        "line_items": FieldScoring("exact", array_strategy="similarity"),
    }
)

trajectory = TrajectoryJudge(
    rubric=(
        "Use the document tool before answering, and retry only after a "
        "transient tool failure."
    )
)
```

Pass one or more of these objects in `evaluators=[...]`. For
`FieldLevelJudge`, omitted Output-schema fields do not receive an explicit
override; use `include=False` when a listed field is intentionally excluded.
Use dotted field paths for nested fields.

## Configure a benchmark with Python

```python
from pathlib import Path

from promptic_sdk import (
    AgentGymClient,
    BenchmarkCase,
    BenchmarkFile,
    MetricBinding,
    VerifierAgent,
    VerifierEvidence,
    VerifierMetric,
)

with AgentGymClient() as gym:
    benchmark = gym.benchmarks.create(
        name="Invoice extraction Agent",
        goal=(
            "Extract the requested invoice fields and produce a review report "
            "that cites unresolved ambiguities."
        ),
        input_schema={
            "type": "object",
            "properties": {
                "document": {"type": "array", "x-promptic-type": "file"},
            },
            "required": ["document"],
        },
        output_schema={
            "type": "object",
            "properties": {
                "invoice_number": {"type": "string"},
                "currency": {"type": "string", "enum": ["EUR", "USD", "GBP"]},
                "report": {"type": "array", "x-promptic-type": "file"},
            },
            "required": ["invoice_number", "currency", "report"],
        },
        evaluators=[
            VerifierAgent(
                instructions="Inspect the selected evidence and score each metric.",
                metrics=(
                    VerifierMetric(
                        "correctness",
                        "Correctness",
                        "Check extracted values against the document.",
                    ),
                    VerifierMetric(
                        "report_quality",
                        "Report quality",
                        "Check that the report explains and cites ambiguities.",
                    ),
                ),
                metric_bindings={
                    "correctness": MetricBinding(weight=2),
                    "report_quality": MetricBinding(weight=1),
                },
                evidence=VerifierEvidence(
                    case_inputs=True,
                    expected_behavior=True,
                    expected_output=True,
                    reference_files=True,
                    submitted_output=True,
                    submitted_artifacts=True,
                    execution_trace=False,
                ),
            )
        ],
    )

    benchmark.cases.add(
        title="Invoice with ambiguous identifier",
        input={"document": [BenchmarkFile(Path("fixtures/invoice.pdf"))]},
        output={
            "invoice_number": "INV-1042",
            "currency": "EUR",
            "report": [BenchmarkFile(Path("fixtures/expected-report.pdf"))],
        },
        expected_behavior=(
            "Use the invoice identifier, not the nearby purchase-order number, "
            "and explain the ambiguity in the report."
        ),
    )
    benchmark.publish()
```

The equivalent CLI workflow stores the same fields in `agent.json`, then runs:

```bash
promptic agent-gym apply agent.json
promptic agent-gym status <benchmark-id>
```

`apply` publishes the first version when the configuration contains cases.
Later authoring changes remain in a draft; inspect them with `promptic
agent-gym draft <benchmark-id>` and publish them with `promptic agent-gym
publish-draft <benchmark-id>` only after the revised contract is coherent.

## Review the benchmark before optimizing

- Every Input-schema field is data the Agent genuinely receives.
- Every submitted structured value or artifact has a matching Output-schema
  field.
- Cases contain expected output where a canonical answer exists and
  case-specific expected behavior where it does not tell the whole story.
- Each evaluator has the evidence it needs and measures a distinct property.
- Judge instructions describe observable success, not the implementation the
  author happens to prefer.
- A human has reviewed representative evaluator results for obvious false
  positives, false negatives, and rubric ambiguity.
- The benchmark is published before the baseline variant is run.
