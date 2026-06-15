# Python Ramp — Learning Sequence

This is the prerequisite ramp. Every other ramp — agentic, FastAPI, RAG, TypeScript, MCP — assumes the skills built here. LeetCode Python gets you syntax and algorithms. This ramp gets you professional Python: the kind used in frameworks, libraries, and production systems.

The ramp is additive. One codebase grows through all 8 stages. The capstone is a real, installable CLI tool that uses every layer.

---

## Structure

```yaml
stages:
  01_project_structure:    "Python as a professional language — envs, imports, modules"
  02_classes_and_oop:      "Classes, dataclasses, dunder methods — framework-style OOP"
  03_type_hints_pydantic:  "Type system + Pydantic — the foundation of FastAPI and LangGraph"
  04_files_config_env:     "pathlib, os.environ, dotenv, json/yaml — real-world operations"
  05_errors_and_testing:   "Custom exceptions, pytest — prerequisite for every capstone"
  06_async_python:         "async/await, asyncio — prerequisite for FastAPI and LangGraph"
  07_packaging:            "pyproject.toml, pip install -e, publishing — OSS prerequisite"
  08_capstone:             "Typer CLI that uses all 7 layers — installable, publishable"
```

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
  - You can set up a Python project with a virtual env and install dependencies
  - You can write a class with type hints, __init__, properties, and dunder methods
  - You can define and validate data with Pydantic BaseModel
  - You can read/write files, load .env config, and handle paths with pathlib
  - You can write pytest tests that pass and fail correctly
  - You understand async def and can write a simple async function
  - You can package a Python project and install it locally with pip install -e .
  - You have a working CLI tool as the capstone artifact
```

---

## Approach

```yaml
rules:
  - Additive — one codebase, one folder, grows through all stages
  - Every stage produces runnable code — not just reading exercises
  - You come from C#/.NET — the concepts are familiar, only the syntax is new
  - No data science, no numpy, no ML — that's a different track
  - Use Python 3.12 throughout
```
