# Stage 07 — Packaging and Distribution

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — add pyproject.toml and wire the entry point
**Depends on:** Stage 06 (full codebase built, tests passing)

---

## The Story So Far

agentlog works. Models validate, storage persists, the runner executes concurrently, tests pass. But you're still running it with `python main.py`. To use it the way you'd actually use a CLI tool — `agentlog run "hello"`, `agentlog list` — it needs to be a proper Python package: installed, entry point registered, importable from anywhere.

This stage adds `pyproject.toml`, installs agentlog with `pip install -e .`, and wires the CLI entry point so `agentlog version` works from the terminal. Stage 08 then builds out the full CLI on top of this foundation.

---

## Why this stage exists

A Python library is just a folder with the right files. `pyproject.toml` is the single config file that defines everything: name, version, dependencies, CLI entry points, test config. `pip install -e .` installs your package in editable mode — changes take effect immediately. `pip install mini-agenteval` works because someone wrote a `pyproject.toml` and published to PyPI. This stage makes that process demystified and reproducible.

---

## Concepts This Stage Teaches

```yaml
concepts:
  pyproject.toml:
    what: >
      The modern standard config file for Python packages.
      Replaces setup.py, setup.cfg, and requirements.txt for most use cases.
    sections:
      "[build-system]": which build tool to use (hatchling, flit, setuptools)
      "[project]":      name, version, description, dependencies
      "[project.scripts]": CLI entry points — agentlog = "agentlog.cli:app"
      "[tool.pytest.ini_options]": pytest config lives here too
    why: >
      Every Python package you publish, contribute to, or fork has this file.
      Reading it tells you everything: dependencies, entry points, Python version,
      dev tools. One file, all the answers.

  pip install -e . (editable install):
    what: >
      Installs the package so Python reads your source directly —
      changes take effect without reinstalling.
    why: >
      Without -e, every code change requires reinstalling the package.
      With -e, you edit a file and the change is immediately visible everywhere.
      All other ramp capstones use pip install -e . to install their own packages.

  Entry points:
    what: >
      [project.scripts]
      agentlog = "agentlog.cli:app"
      Installs a agentlog command in .venv/bin/ that runs cli.py's app function.
    why: >
      This is how uvicorn, pytest, and typer install their terminal commands.
      After pip install -e ., agentlog runs from anywhere in the terminal.

  Optional dependencies:
    what: >
      [project.optional-dependencies]
      dev = ["pytest>=8.0", "pytest-asyncio>=0.23"]
      Install with: pip install -e ".[dev]"
    why: >
      Users get the core package. Contributors get dev tools.
      Standard pattern for every open source Python project.

  Building and publishing:
    what: >
      python -m build → produces dist/ with .whl and .tar.gz
      twine upload --repository testpypi dist/* → uploads to TestPyPI
    why: >
      This is what makes pip install agentlog work for anyone.
      TestPyPI is the safe sandbox — publish there first.

  __version__:
    what: >
      __version__ = "0.1.0" in agentlog/__init__.py
      importlib.metadata.version("agentlog") reads the installed version.
    why: >
      Users and tools need to know which version is installed.
      LangChain, Pydantic, FastAPI all do this. agentlog does too.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write pyproject.toml"
    feature: "agentlog — packaging: pyproject.toml with name, deps, entry point, and pytest config"
    why: >
      This is the package manifest. Everything pip needs to install agentlog
      correctly — name, version, Python version, dependencies, entry point.
      Write it once; pip install -e . handles everything else.
    detail: >
      Create pyproject.toml in the project root (alongside .venv/):

      [build-system]
      requires = ["hatchling"]
      build-backend = "hatchling.build"

      [project]
      name = "agentlog"
      version = "0.1.0"
      description = "A CLI tool for managing and inspecting AI agent runs"
      readme = "README.md"
      requires-python = ">=3.12"
      dependencies = [
          "pydantic>=2.0",
          "pydantic-settings>=2.0",
          "python-dotenv>=1.0",
          "pyyaml>=6.0",
          "httpx>=0.27",
          "typer>=0.12",
          "rich>=13.0",
      ]

      [project.optional-dependencies]
      dev = [
          "pytest>=8.0",
          "pytest-asyncio>=0.23",
          "pytest-cov>=5.0",
      ]

      [project.scripts]
      agentlog = "agentlog.cli:app"

      [tool.pytest.ini_options]
      asyncio_mode = "auto"

  2:
    task: "Install the package in editable mode"
    feature: "agentlog — packaging: pip install -e .[dev] — editable install so source changes are live immediately"
    why: >
      After this command, agentlog is a real installed package. You can import it
      from anywhere in the venv, and the agentlog terminal command exists.
      Changes to your source code take effect immediately — no reinstall.
    detail: >
      pip install -e ".[dev]"

      Verify:
        pip show agentlog
        python -c "import agentlog; print(agentlog.__version__)"

      Now edit agentlog/utils.py — add any function.
      Import it in a Python REPL immediately. No reinstall. This is editable mode.

  3:
    task: "Add __version__ to __init__.py"
    feature: "agentlog — package metadata: __version__ string in __init__.py for CLI and tooling"
    why: >
      The CLI's agentlog version command needs to print this.
      The pyproject.toml version and __version__ should always match.
    detail: >
      # agentlog/__init__.py

      __version__ = "0.1.0"

      from agentlog.utils import greet

      Verify:
        python -c "import agentlog; print(agentlog.__version__)"
        python -c "from importlib.metadata import version; print(version('agentlog'))"

  4:
    task: "Write agentlog/cli.py — the entry point stub"
    feature: "agentlog — cli.py: Typer app scaffold with agentlog version command and registered entry point"
    why: >
      The full CLI is built in Stage 08. Right now you just need the entry point
      to exist so the agentlog terminal command works. A single version command
      is enough to confirm the wiring is correct.
    detail: >
      # agentlog/cli.py

      import typer
      from rich.console import Console

      app = typer.Typer(name="agentlog", help="Manage AI agent runs from the terminal.")
      console = Console()

      @app.command()
      def version():
          """Show the current version."""
          from agentlog import __version__
          console.print(f"agentlog [cyan]v{__version__}[/cyan]")

      if __name__ == "__main__":
          app()

      Test:
        agentlog version        # uses the entry point — should print "agentlog v0.1.0"
        agentlog --help         # shows the help text from typer

  5:
    task: "Write README.md"
    feature: "agentlog — documentation: README with install instructions and all CLI command signatures"
    why: >
      Every package published to PyPI needs a README. This one is minimal now —
      Stage 08 expands it when all commands are built.
    detail: >
      # agentlog

      A CLI tool for managing and inspecting AI agent runs locally.

      ## Installation

      ```bash
      pip install agentlog
      ```

      ## Usage

      ```bash
      agentlog version
      agentlog run "Write a haiku about Python"
      agentlog list
      agentlog get <run_id>
      ```

      ## Development

      ```bash
      git clone ...
      python -m venv .venv && source .venv/bin/activate
      pip install -e ".[dev]"
      pytest tests/
      ```

  6:
    task: "Build the package"
    feature: "agentlog — packaging: python -m build — produce .whl and .tar.gz distribution artifacts"
    why: >
      python -m build produces the actual artifacts that pip installs and PyPI hosts.
      Opening the .whl file shows you exactly what gets distributed — demystifies
      the entire packaging system.
    detail: >
      pip install build
      python -m build

      This creates:
        dist/
          agentlog-0.1.0-py3-none-any.whl
          agentlog-0.1.0.tar.gz

      The .whl is a zip file. Extract it with unzip and look inside.
      This is exactly what gets uploaded to PyPI and downloaded by pip.

  7:
    task: "Publish to TestPyPI (optional but do it once)"
    feature: "agentlog — packaging: full publish cycle — TestPyPI upload and install from index"
    why: >
      Publishing to TestPyPI and then installing from it confirms the full cycle:
      pyproject.toml → build → upload → pip install → running tool.
      Do this once so the process is not mysterious when you publish a real OSS project.
    detail: >
      Register at test.pypi.org (free, separate from pypi.org)
      pip install twine
      twine upload --repository testpypi dist/*

      Verify in a fresh venv:
        python -m venv /tmp/test_venv
        source /tmp/test_venv/bin/activate
        pip install --index-url https://test.pypi.org/simple/ agentlog
        agentlog version
        deactivate
```

---

## Final Project Structure (end of Stage 07)

```
agentlog/                   ← project root
├── .venv/                 ← virtual environment (gitignored)
├── .env                   ← local config (gitignored)
├── .gitignore
├── pyproject.toml
├── README.md
├── agentlog/
│   ├── __init__.py        ← version + exports
│   ├── cli.py             ← entry point stub (expanded in Stage 08)
│   ├── config.py          ← Settings with BaseSettings
│   ├── constants.py       ← VERSION, APP_NAME
│   ├── errors.py          ← exception hierarchy
│   ├── models.py          ← Run, RunRegistry (Pydantic)
│   ├── runner.py          ← async RunRunner
│   ├── storage.py         ← FileStorage
│   └── utils.py           ← greet, format_list, get_banner
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_models.py
    ├── test_storage.py
    └── test_runner.py
```

---

## Done When

```yaml
done_when:
  - pip install -e ".[dev]" installs cleanly
  - agentlog version runs from the terminal via the entry point
  - agentlog --help shows correct help text
  - python -m build produces dist/ without errors
  - All existing tests still pass after the refactor
  - You can explain pyproject.toml's three main sections in one sentence each
  - You understand the difference between pip install . and pip install -e .
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  pyproject.toml: "Like .csproj — package definition, dependencies, build config"
  pip install -e .: "Like project reference in a solution — edits to source reflect immediately"
  entry_point: "Like a console app's Main() registered as an executable"
  hatchling: "Like MSBuild — the build backend that pyproject.toml delegates to"
  twine upload: "Like dotnet nuget push — publishes to the package registry"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between a wheel (.whl) and a source distribution (.tar.gz)?
    When would a user need the source distribution?
  - What does hatchling do? Could you use setuptools instead?
  - What is semantic versioning? When do you increment major vs minor vs patch?
  - How do you handle dependencies that need different versions for different Python versions?
    (Hint: PEP 508 environment markers)
```
