# Python Ramp — Progress

```yaml
status: in_progress
started: ""
completed: ""
total_stages: 8
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_project_structure:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "agentlog/ folder with package, modules, working import chain"
    notes: ""

  02_classes_and_oop:
    status: not_started
    started: ""
    completed: ""
    estimate: "2 days"
    artifact: "agentlog/models.py — Run, RunRegistry, StreamingRun"
    notes: ""

  03_type_hints_pydantic:
    status: not_started
    started: ""
    completed: ""
    estimate: "2 days"
    artifact: "models.py rewritten as Pydantic BaseModel with validation + JSON schema"
    notes: ""

  04_files_config_env:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "pathlib file ops, .env loading, json/yaml config round-trip"
    notes: ""

  05_errors_and_testing:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "custom exceptions + pytest test suite"
    notes: ""

  06_async_python:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "async run execution, asyncio task patterns"
    notes: ""

  07_packaging:
    status: not_started
    started: ""
    completed: ""
    estimate: "1 day"
    artifact: "pyproject.toml, pip install -e . working"
    notes: ""

  08_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "installable Typer CLI using all 7 layers"
    notes: ""
```

---

## Unlocks

```yaml
unlocks:
  agentic_ramp:   "stages 01–06 required"
  fastapi_ramp:   "stages 01–06 required"
  rag_ramp:       "stages 01–05 required"
  mcp_ramp:       "stages 01–06 required"
```

---

## Notes

<!-- freeform — add blockers, surprises, things to revisit -->
