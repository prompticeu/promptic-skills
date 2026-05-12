# API Reference — Benchmark Studio (external runtime)

Not yet exposed on `PrompticClient` — call the HTTP endpoints directly with
`Authorization: Bearer <PROMPTIC_API_KEY>`. See SKILL.md for the loop,
body shape, and error codes.

> **Alpha gate.** Benchmark Studio is restricted to workspace admins. A
> valid non-admin API key returns 403 `Forbidden` until the gate is lifted.

## Endpoints

```
GET  /api/v1/benchmarks/<dataset_id>
GET  /api/v1/benchmarks/<dataset_id>/observations
POST /api/v1/benchmarks/<dataset_id>/runs
GET  /api/v1/benchmarks/<dataset_id>/runs/<run_id>
```

## `POST /runs` request body

The body uses a `runtime` discriminator:

```python
# external mode (this skill)
{
  "runtime": "external",
  "bundle_identity": {
    "name": str, "version": str,                      # required
    "parent_version": str | None,                     # optional, fork ancestor
    "intent": str | None,                             # optional, persistent goal across lineage
    "architecture_description": str | None,           # optional, markdown describing the architecture
    "rationale": str | None,                          # optional, what changed in this version
    "commit_hash": str | None,                        # optional, git pointer
  },
  "predictions": [{"observation_id": uuid, "value": dict}, ...],
  "trace_ids": [uuid, ...] | None,                    # optional
  "token_usage": {"prompt": int, "completion": int, "total": int} | None,
  "latency_ms": int | None,
}

# worker mode (Promptic-shipped stock bundles, triggered from UI)
{
  "runtime": "worker" | None,
  "bundleRefs": [{"name": str, "version": str}, ...],
  "concurrency": int | None,
}
```

## `POST /runs` (external) response

```python
{
  "run_id": uuid,
  "bundle": {
    "id": uuid, "name": str, "version": str,
    "parent_bundle_id": uuid | None,
    "lineage_depth": int,                              # 0 for roots
  },
  "status": "queued",
}
```

## Auth / access errors (every endpoint)

| Status | Body                                                       | Cause                                                          |
|--------|------------------------------------------------------------|----------------------------------------------------------------|
| 401    | `{"error": "Unauthorized"}`                                | No valid auth (neither `Authorization: Bearer …` nor a logged-in dashboard session) |
| 403    | `{"error": "Forbidden"}`                                   | Authenticated user is not a workspace admin (alpha gate)       |
| 404    | `{"error": "<resource> not found"}`                        | Resource doesn't exist or caller's workspace can't see it (RLS) |

## Push-time validation errors (`POST /runs` external mode)

| Status | `error` code                  | Cause                                                                |
|--------|-------------------------------|----------------------------------------------------------------------|
| 400    | `observations_not_in_dataset` | A `prediction.observation_id` doesn't belong to the dataset          |
| 403    | `traces_not_in_workspace`     | A `trace_ids[]` UUID is from a different workspace                   |
| 404    | `parent_version_not_found`    | `parent_version` provided but `(name, parent_version)` doesn't exist |
| 409    | `reserved_bundle_name`        | Bundle name collides with a stock bundle                             |
| 409    | `bundle_authorship_conflict`  | `(name, version)` already exists and is owned by a different user    |

## Reserved bundle names

```
text-extract-single-shot
rag-extract
deep-agent
thinking-deep-agent
```
