# Promptic Skills

AI agent skills for integrating with the [Promptic](https://promptic.eu)
platform. Each skill is a focused capability that loads only when its
trigger pattern matches the user's task.

## Install

```bash
npx skills add prompticeu/promptic-skills
```

## Skills

| Skill | Triggers on | What it covers |
|-------|-------------|----------------|
| **`promptic-trace`** | "add tracing to my LLM app", `import promptic_sdk`, instrumenting OpenAI / Anthropic / LangChain / LangGraph / agents | Auto-instrumentation, `init()` + `ai_component()`, custom workflow spans, custom OTel instrumentors, LangGraph / deepagents support |
| **`promptic-optimize`** | "tune / improve / optimize a prompt", running experiments, deploying a prompt | Experiments → observations → evaluators → iterations → deployments. Supports `prompticV2`, `miproV2`, `bootstrapFewShot`, `gepa` optimizers |
| **`promptic-agent-eval`** | "evaluate / score / grade an agent run", insights on traces, judge-based scoring | Components → datasets → runs → evaluations. Scores agent traces (loop detection, tool errors, unused tools, judge rubrics) |
| **`promptic-benchmark`** | "solve a Promptic Benchmark Studio dataset", "build an extractor for an IE benchmark", "push a candidate bundle / external run", `/dashboard/.../benchmarks/ie/<uuid>` URL | Phase-2 (User-driven AgentGym) external runtime — fetch dataset, run model locally, push predictions, iterate with `parent_version` lineage. **Alpha-gated to workspace admins.** |

## Picking the right skill

```
Want to instrument an LLM app and see traces?               → promptic-trace
Want to tune a prompt to improve a metric?                  → promptic-optimize
Want to score an agent's traces (loops, tool errors, …)?    → promptic-agent-eval
Want to solve a frozen benchmark by submitting predictions? → promptic-benchmark
```

The four skills are independent — install whichever you need. They share
the same `promptic-sdk` Python package and the same auth surface
(`promptic login` / `PROMPTIC_API_KEY`), so installing one and adding
others later is friction-free.

## What are skills?

Skills are reusable capabilities for AI coding agents (Claude Code,
Cursor, Windsurf, etc.). They provide procedural knowledge that helps
agents accomplish specific tasks more effectively.

Learn more at [skills.sh](https://skills.sh).

## Repo layout

```
promptic-skills/
└── skills/
    ├── promptic-trace/
    │   ├── SKILL.md
    │   └── references/api.md
    ├── promptic-optimize/
    │   ├── SKILL.md
    │   └── references/api.md
    ├── promptic-agent-eval/
    │   ├── SKILL.md
    │   └── references/api.md
    └── promptic-benchmark/
        ├── SKILL.md
        └── references/api.md
```

## License

MIT
