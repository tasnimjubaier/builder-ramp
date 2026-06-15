# Stage 07 — Packaging and Distribution

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — add pyproject.toml and package structure to devtool/
**Depends on:** Stage 06 (full codebase built, tests passing)

A Python library is just a folder with the right files. pyproject.toml is the single config file that defines everything: package name, version, dependencies, scripts, metadata. `pip install -e .` installs your package locally in editable mode — changes to your code take effect immediately without reinstalling. `pip install mini-agenteval` works because someone set up a pyproject.toml and published to PyPI. This stage makes that process demystified and reproducible.

---

## What You're Building

Add `pyproject.toml` to devtool and wire everything up:
- `pip install -e .` installs devtool as a real package
- `devtool` command runs from the terminal (CLI entry point)
- Package is ready to publish to TestPyPI

---

## Concepts This Stage Teaches

```yaml
concepts:
  pyproject.toml:
    what: >
      The modern standard config file for Python packages. Replaces setup.py, setup.cfg,
      and requirements.txt for most use cases. One file for everything.
    sections:
      "[build-system]": which build tool to use (hatchling, flit, setuptools)
      "[project]":      package name, version, description, dependencies
      "[project.scripts]": CLI entry points
      "[tool.pytest.ini_options]": pytest configuration
    why: >
      Every Python package you publish, contribute to, or fork will have this file.
      Reading it tells you everything about the project — dependencies, entry points,
      Python version requirements, dev tools.

  pip install -e . (editable install):
    what: >
      Installs the package in "editable" mode — Python finds your source code directly
      instead of copying it into site-packages. Changes take effect without reinstalling.
    why: >
      This is how you develop a library. Without -e, every code change requires reinstalling.
      With -e, you edit a file and the change is immediately visible to anything that imports it.
      All other ramp capstones use pip install -e . to install their own packages.

  Entry points (scripts):
    what: >
      [project.scripts]
      devtool = "devtool.cli:main"
      Installs a devtool command in the venv's bin/ that runs devtool/cli.py's main() function.
    why: >
      This is how uvicorn, pytest, and typer CLI tools install their commands.
      After pip install -e ., you can run devtool from anywhere in the terminal.

  Dependencies in pyproject.toml:
    what: >
      [project]
      dependencies = ["pydantic>=2.0", "pydantic-settings>=2.0", "python-dotenv"]
      Optional groups: [project.optional-dependencies] → dev = ["pytest", "pytest-asyncio"]
    why: >
      pip install -e ".[dev]" installs the package plus dev dependencies.
      This is the standard for open source projects — users get the core package,
      contributors get the dev tools.

  Building and publishing:
    what: >
      python -m build produces dist/ with .whl and .tar.gz files.
      twine upload --repository testpypi dist/* uploads to TestPyPI.
    why: >
      This is what makes pip install your-package work for anyone in the world.
      TestPyPI is a safe sandbox — you publish there first to verify everything works.

  __version__ and version management:
    what: >
      Convention: define __version__ = "0.1.0" in devtool/__init__.py.
      pyproject.toml's version = "0.1.0" should match.
      importlib.metadata.version("devtool") reads the installed package version.
    why: >
      Users and tools need to know which version is installed.
      This is the standard pattern — LangChain, Pydantic, FastAPI all do this.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write pyproject.toml"
    detail: >
      Create pyproject.toml in the project root (devtool/ folder, alongside .venv/):

      [build-system]
      requires = ["hatchling"]
      build-backend = "hatchling.build"

      [project]
      name = "devtool"
      version = "0.1.0"
      description = "A CLI tool for managing AI agent runs"
      readme = "README.md"
      requires-python = ">=3.12"
      dependencies = [
          "pydantic>=2.0",
          "pydantic-settings>=2.0",
          "python-dotenv>=1.0",
          "pyyaml>=6.0",
          "httpx>=0.27",
          "typer>=0.12",
      ]

      [project.optional-dependencies]
      dev = [
          "pytest>=8.0",
          "pytest-asyncio>=0.23",
          "pytest-cov>=5.0",
      ]

      [project.scripts]
      devtool = "devtool.cli:app"

      [tool.pytest.ini_options]
      asyncio_mode = "auto"

  2:
    task: "Install the package in editable mode"
    detail: >
      # With your venv active, from the project root:
      pip install -e ".[dev]"

      Verify:
        pip show devtool         # shows package info
        python -c "import devtool; print(devtool.__version__)"

      Now open devtool/utils.py and add a new function.
      Import it immediately — no reinstall needed. This is editable mode.

  3:
    task: "Add __version__ to __init__.py"
    detail: >
      # devtool/__init__.py

      __version__ = "0.1.0"

      from devtool.utils import greet

      # Verify from terminal:
      python -c "import devtool; print(devtool.__version__)"
      python -c "from importlib.metadata import version; print(version('devtool'))"

  4:
    task: "Write devtool/cli.py (placeholder for capstone)"
    detail: >
      # devtool/cli.py
      # Full CLI is built in Stage 08. This is the entry point stub.

      import typer

      app = typer.Typer(name="devtool", help="Manage AI agent runs from the terminal.")

      @app.command()
      def version():
          """Show the current version."""
          from devtool import __version__
          typer.echo(f"devtool v{__version__}")

      if __name__ == "__main__":
          app()

      Install Typer first: pip install typer (already in pyproject.toml, so it's already installed)
      Test: devtool version   ← uses the entry point installed by pip install -e .

  5:
    task: "Write README.md"
    detail: >
      # devtool

      A CLI tool for managing AI agent runs.

      ## Installation

      ```bash
      pip install devtool
      ```

      ## Usage

      ```bash
      devtool version
      ```

      ## Development

      ```bash
      git clone ...
      python -m venv .venv && source .venv/bin/activate
      pip install -e ".[dev]"
      pytest tests/
      ```

      Every package needs a README. This one is minimal — you'll expand it in Stage 08.

  6:
    task: "Build the package"
    detail: >
      pip install build
      python -m build

      This creates:
        dist/
          devtool-0.1.0-py3-none-any.whl
          devtool-0.1.0.tar.gz

      The .whl is what pip installs. The .tar.gz is the source distribution.
      Open the .whl file — it's a zip. Extract it and look inside.
      This is exactly what gets uploaded to PyPI and downloaded by users.

  7:
    task: "Publish to TestPyPI (optional but recommended)"
    detail: >
      Register at test.pypi.org (free, separate from pypi.org)
      pip install twine
      twine upload --repository testpypi dist/*

      Then install from TestPyPI in a fresh venv to confirm it works:
        python -m venv /tmp/test_venv
        source /tmp/test_venv/bin/activate
        pip install --index-url https://test.pypi.org/simple/ devtool
        devtool version
        deactivate
```

---

## Final Project Structure

After this stage, the devtool project looks like:

```
devtool/                   ← project root
├── .venv/                 ← virtual environment (gitignored)
├── .env                   ← local config (gitignored)
├── .gitignore
├── pyproject.toml         ← package definition
├── README.md
├── devtool/               ← the package
│   ├── __init__.py        ← version + exports
│   ├── cli.py             ← CLI entry point (expanded in Stage 08)
│   ├── config.py          ← Settings with BaseSettings
│   ├── constants.py       ← VERSION, APP_NAME
│   ├── errors.py          ← custom exception hierarchy
│   ├── models.py          ← Run, RunRegistry (Pydantic)
│   ├── runner.py          ← async RunRunner
│   ├── storage.py         ← FileStorage
│   └── utils.py           ← greet, format_list, get_banner
└── tests/
    ├── __init__.py
    ├── conftest.py        ← shared fixtures
    ├── test_models.py
    ├── test_storage.py
    └── test_runner.py
```

---

## Done When

```yaml
done_when:
  - pip install -e ".[dev]" works without errors
  - devtool version runs from the terminal via the entry point
  - python -m build produces dist/ successfully
  - You can explain pyproject.toml's three main sections in one sentence each
  - You understand the difference between pip install . and pip install -e .
  - All existing tests still pass after the refactor
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between a wheel (.whl) and a source distribution (.tar.gz)?
    When would a user need the source distribution?
  - What does hatchling do? Could you use setuptools instead? What's the difference?
  - How do you handle a dependency that has different versions for different Python versions?
    (Hint: look up environment markers in PEP 508)
  - What is semantic versioning (semver)? How does it apply to Python packages?
    When should you increment major vs minor vs patch version?
```
