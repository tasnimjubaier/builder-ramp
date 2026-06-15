# TypeScript Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 5
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_ts_for_python_devs:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1–2 days"
    type: standalone
    artifact: "learn.ts — typed functions, interfaces, async/await, error handling, module exports"
    notes: ""

  02_typed_http_client:
    status: not_started
    started: ""
    completed: ""
    estimate: "1 day"
    type: standalone
    artifact: "client.ts — typed HTTP client with zod schemas, retry logic, typed error classes"
    notes: ""

  03_fastify_api:
    status: not_started
    started: ""
    completed: ""
    estimate: "1 day"
    type: standalone
    artifact: "Fastify prompt proxy API — typed routes, JSON Schema validation, error handlers"
    notes: ""

  04_llm_sdk_wrapper:
    status: not_started
    started: ""
    completed: ""
    estimate: "1 day"
    type: standalone
    artifact: "LLMClient class — chat(), stream() AsyncGenerator, usage tracking"
    notes: ""

  05_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    type: additive
    artifact: "agent-sdk — published npm/jsr package wrapping the FastAPI agent API, full README, bun tests"
    notes: ""
```

---

## Prereqs

```yaml
prereqs:
  runtime:       "Bun — curl -fsSL https://bun.sh/install | bash"
  python_ramp:   "stages 01–03 useful (concepts map directly)"
  fastapi_ramp:  "capstone required before stage 05 (the API being wrapped)"
```

---

## Notes

<!-- freeform — add blockers, surprises, things to revisit -->
