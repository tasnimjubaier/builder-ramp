# Stage 01 — Python as a Professional Language

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive start — this folder grows through all 8 stages
**Depends on:** basic Python syntax (you have this from LeetCode)

LeetCode Python runs in a single function in a single file. Real Python projects have structure: a virtual environment that isolates dependencies, a package that can be imported, modules that separate concerns, and a clear boundary between "my code" and "installed libraries." This stage builds that foundation.

---

## What You're Building

A project folder called `devtool/` that will grow through all 8 stages. By the end of this stage it has:
- A virtual environment
- One Python package (`devtool/`)
- Two modules inside it
- A working import chain you can run from the terminal

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
      devtool/utils.py is a module. devtool/ (with __init__.py) is a package.
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
      Absolute: from devtool.utils import format_message   (full path from project root)
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
mkdir devtool
cd devtool

# Create virtual environment
python3.12 -m venv .venv
source .venv/bin/activate

# Confirm
which python   # should show .venv/bin/python
python --version   # should show 3.12.x
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create the project structure"
    detail: >
      devtool/               ← project root
      ├── .venv/             ← virtual environment (git-ignored)
      ├── devtool/           ← the Python package
      │   ├── __init__.py    ← makes this a package
      │   ├── models.py      ← data structures (grows in Stage 02-03)
      │   └── utils.py       ← utility functions
      └── main.py            ← entry point script

      Create all files now, even if empty.
      touch devtool/__init__.py devtool/models.py devtool/utils.py main.py

  2:
    task: "Write something in utils.py"
    detail: >
      # devtool/utils.py

      def greet(name: str) -> str:
          return f"Hello, {name}!"

      def format_list(items: list) -> str:
          if not items:
              return "(empty)"
          return "\n".join(f"  - {item}" for item in items)

      Note the type hints — name: str, -> str. Not required yet but start the habit now.

  3:
    task: "Write something in __init__.py"
    detail: >
      # devtool/__init__.py

      # This file marks devtool/ as a package.
      # You can import things here to make them available at the package level.

      from devtool.utils import greet

      # Now both of these work:
      #   from devtool.utils import greet      (module-level import)
      #   from devtool import greet            (package-level import via __init__)

  4:
    task: "Write main.py with the __main__ guard"
    detail: >
      # main.py

      from devtool.utils import greet, format_list
      from devtool import greet as greet_short   # same function, imported differently

      def run():
          print(greet("world"))
          print(greet_short("world"))
          print(format_list(["agentic ramp", "fastapi ramp", "rag ramp"]))

      if __name__ == "__main__":
          run()

  5:
    task: "Run it"
    detail: >
      # From the project root (devtool/)
      python main.py

      You should see:
        Hello, world!
        Hello, world!
          - agentic ramp
          - fastapi ramp
          - rag ramp

  6:
    task: "Add a second module and import between modules"
    detail: >
      Create devtool/constants.py:
        VERSION = "0.1.0"
        APP_NAME = "devtool"

      Import it in utils.py:
        from devtool.constants import APP_NAME

        def get_banner() -> str:
            return f"=== {APP_NAME} ==="

      Import get_banner in main.py and call it.
      This exercises the inter-module import pattern you'll use constantly.

  7:
    task: "Break imports deliberately"
    detail: >
      - Delete __init__.py and try to run main.py. What error? Restore it.
      - Change "from devtool.utils import greet" to "from utils import greet".
        Run from the project root. What error? Run from inside devtool/. Different result?
        This teaches you why absolute imports and running from the project root matter.
      - Add print("utils loaded") at the top of utils.py (outside any function).
        Import utils from main.py. Does it print? When?
        This teaches the side-effect-at-import problem.
```

---

## .gitignore

Create this now — you'll need it throughout:

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
```

---

## Done When

```yaml
done_when:
  - python main.py runs and prints correct output
  - You can explain the difference between a module and a package in one sentence
  - You understand what __init__.py does and what happens without it
  - You can add a new module, add a function to it, and import it correctly
  - You know what "ModuleNotFoundError" means and how to fix it
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
    (You'll answer this properly in Stage 07 — note it for now)
  - What does from devtool import * do? Why is it usually a bad idea?
  - What happens if two modules import each other (circular imports)?
    Can you trigger this deliberately and read the error?
  - What is __all__ in __init__.py and when would you use it?
```
