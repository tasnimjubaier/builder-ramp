# Stage 01 — Python as a Professional Language

**Status:** not started
**Estimated time:** 1–2 days
**Type:** foundation — this folder grows through all 8 stages
**Depends on:** basic Python syntax (you have this from LeetCode)

---

## The Project

You're going to build **`agentlog`** — a local CLI that runs prompts against an LLM, logs every run, and lets you list, inspect, and replay them from the terminal. By Stage 08 it's a real installable tool you'd actually use while building agents.

This stage doesn't touch LLMs yet. It builds the house the project will live in. Every other stage adds a floor.

By the end of this stage you have a working project skeleton with a virtual environment, a package you can import, and a confirmed import chain. Small, but real.

---

## Why this stage exists

LeetCode Python runs in a single function in a single file. The judge handles everything else. Real Python projects have structure: a virtual environment that isolates dependencies, a package that can be imported, modules that separate concerns, and a clear boundary between "my code" and "installed libraries."

When you eventually add LLM calls, Pydantic models, async runners, and a CLI — they all need to live somewhere that imports cleanly. That somewhere is what you're building now.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Virtual environment:
    what: >
      An isolated Python installation just for this project. pip install inside a venv
      only affects this project — not your system Python or other projects.
    why: >
      Without a venv, every project shares the same packages. Version conflicts break
      things in ways that are hard to debug. Every professional Python project starts with a venv.
    commands:
      create: "python -m venv .venv"
      activate_mac: "source .venv/bin/activate"
      deactivate: "deactivate"
      confirm: "which python  # should show .venv/bin/python"

  Package vs module:
    what: >
      A module is a single .py file.
      A package is a folder containing an __init__.py file — it can be imported.
      agentlog/utils.py is a module. agentlog/ (with __init__.py) is a package.
    why: >
      Every library you install (langchain, fastapi, pydantic) is a package.
      When you write from langchain_core.messages import HumanMessage, you're
      importing from a package. Understanding this makes imports stop being magic.

  __init__.py:
    what: >
      An empty (or non-empty) file that marks a folder as a Python package.
      Without it, Python won't treat the folder as importable.
    why: >
      This is the single most common "why isn't my import working" cause.
      Once you understand __init__.py, import errors become obvious.

  Absolute vs relative imports:
    what: >
      Absolute: from agentlog.utils import format_message   (full path from project root)
      Relative: from .utils import format_message           (relative to current file)
    why: >
      Absolute imports are preferred — they work regardless of where you run the script from.
      Relative imports break when you move files. Use absolute by default.

  if __name__ == "__main__":
    what: >
      Code inside this block only runs when the file is executed directly (python main.py),
      not when it's imported by another module.
    why: >
      Without this guard, importing a module that has side effects (printing, connecting to a DB)
      runs those side effects at import time. Every script that can also be imported needs this.

  sys.path and PYTHONPATH:
    what: >
      Python searches for modules in a list of directories (sys.path).
      Running python from the project root usually adds the project root to sys.path.
      PYTHONPATH is an env var that adds directories to sys.path.
    why: >
      "ModuleNotFoundError" is almost always a sys.path problem.
      Understanding sys.path means you can diagnose import errors immediately.
```

---

## Project Setup

```bash
mkdir agentlog
cd agentlog

python3.12 -m venv .venv
source .venv/bin/activate

which python   # should show .venv/bin/python
python --version   # should show 3.12.x
```

---

## Build Steps

The goal of this stage: get to `python main.py` printing real output, with a clean import chain across multiple modules. Every step adds one piece.

```yaml
steps:
  1:
    task: "Create the skeleton"
    feature: "agentlog — project scaffold: folder structure, package init, virtual environment"
    why: >
      agentlog needs a home. The package folder is where all future modules live —
      models, config, runner, storage, CLI. Create the structure now so imports work
      from day one.
    detail: >
      agentlog/               ← project root
      ├── .venv/             ← virtual environment (git-ignored)
      ├── agentlog/           ← the Python package
      │   ├── __init__.py    ← makes this a package
      │   ├── models.py      ← data structures (grows in Stage 02-03)
      │   └── utils.py       ← utility functions
      └── main.py            ← entry point script

      touch agentlog/__init__.py agentlog/models.py agentlog/utils.py main.py

  2:
    task: "Write something in utils.py"
    feature: "agentlog — utils module: format_list() and greet() utility functions"
    why: >
      agentlog needs to display run summaries in the terminal. A format_list() utility
      will be used in every stage that prints output. Write it now so the import chain
      has something real to exercise.
    detail: >
      # agentlog/utils.py

      def greet(name: str) -> str:
          return f"Hello, {name}!"

      def format_list(items: list) -> str:
          if not items:
              return "(empty)"
          return "\n".join(f"  - {item}" for item in items)

      Note the type hints — name: str, -> str. Not enforced yet but start the habit now.
      Every function in this project will have them.

  3:
    task: "Wire __init__.py"
    feature: "agentlog — package public API: top-level exports via __init__.py"
    why: >
      You want to write `from agentlog import greet` in other modules, not
      `from agentlog.utils import greet`. __init__.py is where you control
      what the package exposes at the top level.
    detail: >
      # agentlog/__init__.py

      from agentlog.utils import greet

      # Now both of these work:
      #   from agentlog.utils import greet      (module-level import)
      #   from agentlog import greet            (package-level import via __init__)

  4:
    task: "Write main.py with the __main__ guard"
    feature: "agentlog — entry point: main.py with __main__ guard and import chain verification"
    why: >
      main.py is the entry point for the whole project right now. Later it becomes
      a CLI, but for this stage it's a simple script that confirms imports work.
      The __main__ guard ensures this file can also be imported safely.
    detail: >
      # main.py

      from agentlog.utils import greet, format_list
      from agentlog import greet as greet_short

      def run():
          print(greet("agentlog"))
          print(greet_short("agentlog"))
          print(format_list(["run a prompt", "log it", "inspect it", "replay it"]))

      if __name__ == "__main__":
          run()

  5:
    task: "Run it"
    feature: "agentlog — smoke test: confirm venv, package import, and output are working"
    detail: >
      python main.py

      You should see:
        Hello, agentlog!
        Hello, agentlog!
          - run a prompt
          - log it
          - inspect it
          - replay it

      This confirms: venv is active, package is importable, import chain works.

  6:
    task: "Add a constants module"
    feature: "agentlog — constants module: VERSION, APP_NAME, and get_banner() utility"
    why: >
      agentlog will need a version number and an app name in multiple places —
      the CLI header, the storage file, the package metadata. One module,
      imported everywhere. Add it now and practice cross-module imports.
    detail: >
      # agentlog/constants.py
      VERSION = "0.1.0"
      APP_NAME = "agentlog"

      # agentlog/utils.py — add:
      from agentlog.constants import APP_NAME

      def get_banner() -> str:
          return f"=== {APP_NAME} ==="

      # main.py — add:
      from agentlog.utils import get_banner
      print(get_banner())

  7:
    task: "Break imports deliberately"
    feature: "agentlog — import system: understand ModuleNotFoundError, absolute imports, side effects"
    why: >
      You'll hit import errors constantly while building the rest of this project.
      Trigger them on purpose now so you recognize them on sight.
    detail: >
      - Delete __init__.py and run main.py. What error? Restore it.
      - Change "from agentlog.utils import greet" to "from utils import greet".
        Run from project root. What error? Run from inside agentlog/. Different result?
        This shows why absolute imports and running from the project root matter.
      - Add print("utils loaded") at the top of utils.py (outside any function).
        Import utils from main.py. Does it print? When does it run?
        This is the side-effect-at-import problem you'll avoid in all future modules.
```

---

## .gitignore

```
.venv/
__pycache__/
*.pyc
*.pyo
.env
.DS_Store
dist/
build/
*.egg-info/
.agentlog_data/
```

---

## Done When

```yaml
done_when:
  - python main.py runs and prints correct output
  - You can explain the difference between a module and a package in one sentence
  - You understand what __init__.py does and what happens without it
  - You can add a new module, write a function in it, and import it correctly
  - You know what ModuleNotFoundError means and how to fix it
  - You can explain why if __name__ == "__main__" exists
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  virtual_env: "Like a project-scoped NuGet packages folder — not global"
  package: "Like a C# namespace + assembly. The folder IS the namespace."
  __init__.py: "Like AssemblyInfo.cs — marks the boundary of the package"
  import: "Like using MyNamespace.MyClass — but you import the name, not declare a using"
  if __name__ == "__main__": "Like static void Main() — but opt-in per file"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between pip install and pip install -e .?
    (You'll answer this in Stage 07)
  - What does from agentlog import * do? Why is it usually a bad idea?
  - What happens if two modules import each other (circular imports)?
    Trigger it deliberately and read the error.
  - What is __all__ in __init__.py and when would you use it?
```
