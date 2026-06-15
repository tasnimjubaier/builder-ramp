# Python Ramp — Learning Sequence

This is the prerequisite ramp. Every other ramp — agentic, FastAPI, RAG, TypeScript, MCP — assumes the skills built here. LeetCode Python gets you syntax and algorithms. This ramp gets you professional Python: the kind used in frameworks, libraries, and production systems.

---

## The Project

You build **one thing across all 8 stages**: `agentlog` — a local CLI that runs prompts against an LLM, logs every run, and lets you list, inspect, and replay them from the terminal.

```bash
agentlog run "Write a haiku about Python"
agentlog list
agentlog list --status failed
agentlog get <run_id>
agentlog clear
agentlog config show
agentlog version
```

Each stage adds one layer. The LLM call comes last — in Stage 08. Everything before it builds the foundation that makes the LLM call clean and testable.

---

## Structure

```yaml
stages:
  01_project_structure:
    builds: "agentlog/ skeleton — venv, package, import chain"
    concept: "Virtual envs, packages, modules, imports"

  02_classes_and_oop:
    builds: "agentlog/models.py — Run, RunRegistry, StreamingRun"
    concept: "Classes, dataclasses, dunder methods, inheritance, properties"

  03_type_hints_pydantic:
    builds: "Rewrite models.py with Pydantic — validation, serialization, schema"
    concept: "Type hints, BaseModel, Field validators, nested models, model_dump()"

  04_files_config_env:
    builds: "agentlog/config.py + agentlog/storage.py — runs persist to disk"
    concept: "pathlib, BaseSettings, json, yaml, context managers"

  05_errors_and_testing:
    builds: "agentlog/errors.py + tests/ — explicit failures, full test suite"
    concept: "Exception hierarchy, pytest fixtures, parametrize, tmp_path, coverage"

  06_async_python:
    builds: "agentlog/runner.py — async RunRunner, concurrent batch execution"
    concept: "Event loop, async/await, asyncio.gather(), blocking vs non-blocking"

  07_packaging:
    builds: "pyproject.toml — installable package, agentlog terminal command"
    concept: "pyproject.toml, editable install, entry points, build + publish"

  08_capstone:
    builds: "agentlog/cli.py — full CLI wired to everything, real OpenAI calls"
    concept: "Synthesis — all 7 layers working together"
```

---

## The thread

Every step is motivated by the project, not by the concept list.

- Stage 01: agentlog needs a home → structure
- Stage 02: runs need to be represented → classes
- Stage 03: runs need validation → Pydantic
- Stage 04: runs need to persist → files and config
- Stage 05: things break; you need to know when → errors and tests
- Stage 06: LLM calls are slow; run them in parallel → async
- Stage 07: you want to install it properly → packaging
- Stage 08: wire everything into a tool you actually use → capstone

---

## What This Unlocks

```yaml
unlocks:
  agentic_ramp:    "Stages 01–06 all required — LangGraph is async, OOP-heavy, type-annotated"
  fastapi_ramp:    "Stages 01–06 all required — FastAPI is async, Pydantic-first"
  rag_ramp:        "Stages 01–05 required — sync Python, file ops, testing"
  typescript_ramp: "Stages 01–03 useful — concepts map directly"
  mcp_ramp:        "Stages 01–06 required — MCP SDK is async Python"
```

---

## Completion Criteria

```yaml
done_when:
  - agentlog run "hello" makes a real OpenAI call and stores the result
  - agentlog list shows all runs in a Rich table with colored status
  - pytest tests/ passes with >80% coverage on core modules
  - pip install -e . works cleanly
  - You can trace a single agentlog run call through the entire codebase
    from CLI → runner → storage → models → disk → back
  - You can read a LangGraph or FastAPI source file and understand every class, decorator, and pattern
```

---

## Rules

```yaml
rules:
  - One codebase, one project — grows through all 8 stages
  - Every stage produces runnable code — not just reading exercises
  - You come from C#/.NET — concepts are familiar, only syntax is new
  - No data science, no numpy, no ML — different track
  - Python 3.12 throughout
```
