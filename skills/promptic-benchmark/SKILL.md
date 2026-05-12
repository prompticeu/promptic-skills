---
name: promptic-benchmark
description: >-
  Solve a Promptic Benchmark Studio dataset by building a local extractor / agent and pushing predictions to the platform. Use when the user wants to "solve a benchmark", "build an extractor for an IE benchmark", "push a candidate bundle / external run", "iterate on a previous bundle version", or shows a `/dashboard/.../benchmarks/ie/<uuid>` URL. Covers the Phase-2 (User-driven AgentGym) external-runtime contract — fetch dataset target_schema + observations, download PDFs, run a local model, POST predictions, poll the leaderboard, iterate with `parent_version` lineage. **Distinct from `promptic-agent-eval`** — this skill scores prediction *values* against held-out gold, agent-eval scores agent *traces*. Companion skills — `promptic-trace` (instrument the local extractor's LLM calls so trace IDs can be linked on the run), `promptic-agent-eval` (score agent runs by trace), `promptic-optimize` (prompt experiments).
---

# Promptic Benchmark Studio (External Runtime)

Benchmark Studio is a **Kaggle-shaped** track for solving frozen evaluation
datasets. Pick this skill when the user wants to "build an extractor / agent
that maximises score against a benchmark dataset" — typically information
extraction (PDFs → structured JSON) but the contract is task-agnostic.

> **Distinct from `promptic-agent-eval`.** Agent Evaluation scores agent
> runs by their *traces* (loop detection, tool errors, …) and uses the
> `components / datasets / runs / evaluations` tables. Benchmark Studio
> scores *predictions* against held-out gold and uses the
> `benchmark_datasets / benchmark_observations / benchmark_runs /
> candidate_bundles` tables. Don't mix them up — different DB tables,
> different endpoints, different mental model. If the user shows you a
> `/dashboard/.../benchmarks/ie/<uuid>` URL or mentions "IE benchmark" /
> "Benchmark Studio" / "external runtime", you're here.

## One run, one bundle (invariant)

Every new run — worker, external, sandbox — corresponds to **exactly one
bundle**. There's no such thing as a "trigger 5 stock bundles, get one
multi-bundle run with a leaderboard inside it." Multi-bundle runs only
exist as a legacy fallback for rows in the database that pre-date the
current model.

When a user ticks N stock bundles in the dashboard's trigger form, the
backend fans out: `POST /runs` worker-mode creates **N independent runs**
(one per bundle), enqueues N jobs, and returns
`{run_ids: string[], bundle_count: N}`. Each run flows through the
worker independently, lands in the leaderboard as its own row, and has
its own single-bundle run-detail page. Comparing them is the dataset
leaderboard's Compare action — not a leaderboard nested inside one run.

This makes worker mode and external mode symmetric. Whether the bundle
ran on the worker (stock) or externally (user's machine), it produces
one run, one row, one set of predictions.

## What "external runtime" means

Phase-2 of the AgentGym vision: the user's *coding agent* (you) writes a
candidate extractor in the user's repo, runs it locally on the user's LLM
keys, and pushes only the predictions to Promptic. Promptic re-runs its
existing scoring + insight pipeline server-side and returns a leaderboard.
No sandbox involved — the user trusts you because they invoked you.

## Data flow

```
Promptic → you:    target_schema (JSON Schema describing the answer shape)
                   observations[]  ({id, pdf_url, pdf_name})
                   PDF bytes (one fetch per observation, presigned URL)
                   gold values are NEVER returned to non-owner workspaces

you → Promptic:    predictions[]  ({observation_id, value})
                   bundle_identity ({name, version, parent_version, intent,
                                     architecture_description, rationale})
                   token_usage, trace_ids (optional)
```

## Loop

```
1. GET  /api/v1/benchmarks/<dataset_id>                  → target_schema
2. GET  /api/v1/benchmarks/<dataset_id>/observations     → [{id, pdfUrl, ...}]
3. for each obs: fetch(obs.pdfUrl) → run your model → value
4. POST /api/v1/benchmarks/<dataset_id>/runs             → run_id
5. GET  /api/v1/benchmarks/<dataset_id>/runs/<run_id>    (poll until terminal)
6. read aggregates + per-field scores → revise architecture → push v_next
```

## Auth

Two valid sources, picked automatically by the route handler:

1. **API key (recommended for scripts / agents)** — mint one in the
   dashboard under **Settings → API Keys**, expose it as
   `PROMPTIC_API_KEY`, and send it as a bearer token:

   ```python
   import httpx, os
   http = httpx.AsyncClient(
       headers={"Authorization": f"Bearer {os.environ['PROMPTIC_API_KEY']}"}
   )
   ```

2. **Dashboard session cookie** — used by the in-browser benchmark forms
   (create dataset, upload observation, trigger run). No script-side
   action needed; it Just Works for fetches issued from the dashboard.

**Alpha gating.** Benchmark Studio is currently restricted to workspace
admins. Either auth source must resolve to a user with `is_admin=true` or
the routes return 403 `Forbidden`. Ask the workspace owner to grant admin
access (or to mint the API key under their own account) until the feature
opens up.

## Push body shape

```python
{
    "runtime": "external",
    "bundle_identity": {
        "name": "my-extractor",        # required, ≤100 chars
        "version": "0.2.0",            # required, ≤50 chars
        "parent_version": "0.1.0",     # optional — the (name, version) you forked
        "intent": "...",               # optional ≤5000 chars — persistent goal
        "architecture_description": "...",  # optional ≤20000 chars — markdown describing the architecture
        "rationale": "...",            # optional ≤2000 chars — what changed in this version
        "commit_hash": "abc123...",    # optional — git commit pointer
    },
    "predictions": [
        {"observation_id": "<uuid>", "value": {...}},  # value matches target_schema
        ...
    ],
    "trace_ids": ["<uuid>", ...],      # optional — link traces ingested via promptic_sdk
    "token_usage": {                   # optional
        "prompt": 12000, "completion": 1200, "total": 13200,
    },
    "latency_ms": 8500,                # optional
}
```

## Response

```python
{
    "run_id": "<uuid>",
    "bundle": {
        "id": "<uuid>",
        "name": "my-extractor", "version": "0.2.0",
        "parent_bundle_id": "<uuid>",  # null for roots
        "lineage_depth": 1,            # 0 for roots, increments per parent hop
    },
    "status": "queued",
}
```

## Errors to handle

Auth / access errors (apply to every endpoint):

| Status | Body                                          | Cause                                                          |
|--------|-----------------------------------------------|----------------------------------------------------------------|
| 401    | `{"error": "Unauthorized"}`                   | No valid auth (neither `Authorization: Bearer …` nor a logged-in dashboard session) |
| 403    | `{"error": "Forbidden"}`                      | Authenticated user is not a workspace admin (alpha gate)       |
| 404    | `{"error": "<resource> not found"}`           | Resource doesn't exist *or* the caller's workspace can't see it (RLS-gated) |

Push-time validation errors (`POST /runs` external mode):

| Status | `error` code                       | What it means                                                            |
|--------|------------------------------------|--------------------------------------------------------------------------|
| 400    | `observations_not_in_dataset`      | A `prediction.observation_id` doesn't belong to the dataset              |
| 403    | `traces_not_in_workspace`          | A `trace_ids[]` UUID is from a different workspace                       |
| 404    | `parent_version_not_found`         | `parent_version` provided but `(name, parent_version)` doesn't exist     |
| 409    | `reserved_bundle_name`             | Bundle name collides with a stock bundle (e.g. `rag-extract`)            |
| 409    | `bundle_authorship_conflict`       | `(name, version)` already exists and is owned by a different user        |

When a user iterates, you only need to bump `version` (e.g. `0.1.0` →
`0.2.0`) and set `parent_version` to the previous one. The `(name, version)`
upsert is idempotent — pushing the same version twice as the same author is
a no-op (definition gets refreshed but no new run row is created); pushing
as a different user gets the 409.

## Where to read scores in the dashboard

The **dataset page** at `/dashboard/.../benchmarks/ie/<dataset_id>` is the
canonical leaderboard. It shows one row per `(bundle_name, bundle_version)`
ever scored on this dataset, sourced from each bundle's *latest successful
run*. New external pushes appear there automatically once scoring lands.

- **Multi-bundle compare**: tick 2-5 rows on the dataset page → "Compare"
  → side-by-side leaderboard + per-observation diff. URL is shareable:
  `/.../compare?bundles=name@v1,name@v2,…`.
- **Single-bundle deep-dive**: clicking a bundle name (or navigating to
  `/.../bundles/<name>/<version>`) shows the bundle's identity (intent,
  architecture_description, rationale, commit_hash), lineage chain,
  score-over-time sparkline, and every run that scored it on this dataset.
- **Run pages** (`/.../runs/<run_id>`) are demoted to "execution receipts" —
  useful for diagnostics (failed runs, retry, export) but not the canonical
  comparison surface. When a coding agent helps a user iterate, refer to
  bundles and the dataset leaderboard, not individual runs.

## Reserved bundle names

These are owned by Promptic-shipped seed bundles — picking any of them as
your `bundle_identity.name` returns 409:

```
text-extract-single-shot, rag-extract, deep-agent, thinking-deep-agent
```

## Three identity fields, three jobs

The bundle metadata has three free-text fields. Each has a specific job;
fill them in correctly so other readers (humans + agents) can understand
the bundle without running it.

| Field | Purpose | Updates when |
|---|---|---|
| `intent` | The persistent *goal* across the whole lineage chain (e.g. "ESG sustainability extraction with verbatim quotes for narrative fields, exact match for GRI numerics"). Copy through unchanged from parent → child. | Rarely. |
| `architecture_description` | A markdown summary of *what this extractor is*: the architecture, the model, the parsing approach, the prompt strategy. 2-5 sentences is plenty. Lets others read what your bundle does without running it. | When the architecture genuinely changes. |
| `rationale` | A one-line summary of *what changed in this version vs the parent*. Targets the hypothesis you're testing. | Every push. |

Example for `my-extractor@0.3.0`:

```
intent:
  ESG sustainability extraction with verbatim quotes; exact match for GRI numerics.

architecture_description:
  Two-step retrieve + extract:
  1. Chunk the PDF into 512-token windows with 64-token overlap.
  2. Embed each chunk with text-embedding-3-small + cosine-rank against the field
     description; keep top-3.
  3. Fill the schema with one gpt-4.1-nano call per field, scoped to that field's
     retrieved chunks.

rationale:
  v0.3.0 — added per-field retrieval. v0.2.0 saw 0.42 on supply_chain_summary
  because the single-shot prompt didn't isolate the relevant section.
```

## Iteration discipline (important for the agent)

When asked to "iterate" or "improve":

1. Read the existing run's per-field scores and pick the **lowest-scoring
   fields** as your hypothesis target.
2. Form a hypothesis (e.g. "supply_chain_summary scored 0.42 because the
   single-shot prompt sees too much irrelevant content; rerank by section
   headings before generation").
3. Implement the change in the local extractor.
4. Push as `v_next` with:
   - `parent_version=v_current`
   - `rationale="<the hypothesis you just stated>"` — what changed and why
   - `architecture_description` updated **only if the architecture itself
     genuinely changed** (added retrieval, swapped models, restructured
     prompts). If the change was a tweak inside the same architecture
     (different prompt wording, different temperature), copy the parent's
     `architecture_description` through unchanged.
   - `intent` left unchanged — it anchors the whole chain.
5. Wait for scoring. Compare per-field deltas to the parent's scores. If
   the targeted field improved, keep iterating; if not, pop back to the
   parent and try a different hypothesis.

## Reference template

A working end-to-end implementation lives at
`scripts/external_run_real_agent.py` in the Promptic repo (osaka-v2). It
shows: dataset fetch, observation listing, PDF download, single-shot OpenAI
structured-output call, predictions push, leaderboard polling. Read that
file before authoring your own — copy the auth + push pattern verbatim;
swap the extraction logic for whatever architecture you're trying.

## Minimal Python skeleton

```python
import asyncio, httpx, os
from openai import AsyncOpenAI

BASE = os.environ["PROMPTIC_BASE_URL"]
HEADERS = {"Authorization": f"Bearer {os.environ['PROMPTIC_API_KEY']}"}
DATASET = os.environ["DATASET_ID"]

async def main():
    oa = AsyncOpenAI()
    async with httpx.AsyncClient(headers=HEADERS, timeout=120.0) as http:
        ds = (await http.get(f"{BASE}/api/v1/benchmarks/{DATASET}")).json()
        schema = ds["targetSchema"]
        obs = (await http.get(f"{BASE}/api/v1/benchmarks/{DATASET}/observations")).json()["data"]

        predictions = []
        for o in obs:
            pdf = (await http.get(o["pdfUrl"])).content
            text = my_pdf_to_text(pdf)            # your text extraction
            value = await my_extract(oa, text, schema)  # your model call
            predictions.append({"observation_id": o["id"], "value": value})

        r = await http.post(f"{BASE}/api/v1/benchmarks/{DATASET}/runs", json={
            "runtime": "external",
            "bundle_identity": {
                "name": "my-extractor", "version": "0.1.0",
                "intent": "...",                    # persistent goal across the chain
                "architecture_description": "...",  # markdown: what this extractor does
                "rationale": "...",                 # what changed in this version vs parent
            },
            "predictions": predictions,
        })
        r.raise_for_status()
        print(r.json())

asyncio.run(main())
```

## Linking traces to a run (optional, recommended)

If your local extractor uses LLM calls and you want each call's trace
linked to the benchmark run, instrument the extractor with `promptic_sdk`
per the `promptic-trace` skill, collect the resulting trace IDs, and pass
them in `trace_ids[]` on the push:

```python
import promptic_sdk
promptic_sdk.init()

trace_ids = []
with promptic_sdk.ai_component("my-extractor", run="benchmark-v0.1.0") as ctx:
    # … run your pipeline; capture trace IDs from spans …
    pass

# Then include in the push body:
body["trace_ids"] = trace_ids
```

The platform will surface the linked traces from the run's detail page so
you can drill into individual extractions when iterating.

## Planned (not yet shipped)

`PrompticClient` does not yet expose `benchmarks.*`. Once it does, the
above collapses to:

```python
client.benchmarks.push_run(
    dataset_id, bundle_identity={...}, predictions=[...],
)
client.benchmarks.wait_for_run(dataset_id, run_id, timeout=600)
```

Until then, use raw `httpx` against the endpoints listed above.

## API reference

See [references/api.md](references/api.md) for the HTTP endpoint catalogue.
