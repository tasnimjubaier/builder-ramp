# Stage 08 — Capstone: devtool CLI

**Status:** not started
**Estimated time:** 3–4 days
**Type:** additive — complete devtool/cli.py using all 7 stages
**Depends on:** all previous stages

This is the synthesis. Every concept from every stage — classes, Pydantic, config, file I/O, error handling, async, packaging — gets wired together into a real, installable CLI tool. After this stage you have one artifact that demonstrates professional Python, not a collection of scripts.

---

## What You're Building

Complete `devtool` — a CLI for managing AI agent runs locally:

```bash
devtool run "Write a haiku about Python"      # submit a run, print result
devtool run "Summarize this" --async          # submit and return immediately (async mode)
devtool list                                   # list all runs with status
devtool get <run_id>                          # show details of a specific run
devtool clear                                 # clear all stored runs
devtool version                               # show version
devtool config show                           # print current config
```

Everything persists to disk via FileStorage. Async runs use asyncio. Output is formatted with Rich (colored terminal output). Tests cover all commands.

---

## Final Project Structure

```
devtool/
├── devtool/
│   ├── __init__.py
│   ├── cli.py          ← all commands live here (expanded from Stage 07 stub)
│   ├── config.py
│   ├── constants.py
│   ├── errors.py
│   ├── models.py
│   ├── runner.py
│   ├── storage.py
│   └── utils.py
├── tests/
│   ├── conftest.py
│   ├── test_cli.py     ← NEW: CLI integration tests
│   ├── test_models.py
│   ├── test_runner.py
│   └── test_storage.py
├── .env
├── .gitignore
├── pyproject.toml
└── README.md
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Add Rich for terminal output"
    detail: >
      pip install rich
      Add "rich>=13.0" to pyproject.toml dependencies.

      Rich gives you colored output, tables, and progress bars.
      It's used in Typer, FastAPI dev server, and most modern Python CLIs.

      Quick test:
        from rich.console import Console
        console = Console()
        console.print("[green]Hello[/green] [bold]world[/bold]!")

  2:
    task: "Build devtool run command"
    detail: >
      # devtool/cli.py

      import asyncio
      import typer
      from rich.console import Console
      from rich.table import Table
      from devtool.models import Run, RunRegistry
      from devtool.runner import RunRunner
      from devtool.storage import FileStorage
      from devtool.config import settings
      from devtool.errors import DevToolError
      import uuid

      app = typer.Typer(name="devtool", help="Manage AI agent runs from the terminal.")
      console = Console()

      def get_storage() -> FileStorage:
          return FileStorage(data_dir=".devtool_data")

      @app.command()
      def run(
          prompt: str = typer.Argument(..., help="The prompt to run"),
          wait: bool = typer.Option(True, "--wait/--no-wait",
                                    help="Wait for result or return immediately"),
      ):
          """Submit a new run."""
          storage = get_storage()
          runner = RunRunner(storage)
          new_run = Run(run_id=str(uuid.uuid4())[:8], prompt=prompt)

          if wait:
              result = asyncio.run(runner.execute(new_run))
              console.print(f"[green]Done[/green] [{result.run_id}]")
              console.print(result.result)
          else:
              # Fire and register, return immediately
              registry = storage.load()
              registry.add(new_run)
              storage.save(registry)
              console.print(f"[yellow]Submitted[/yellow] [{new_run.run_id}]")
              console.print(f"Check status: devtool get {new_run.run_id}")

  3:
    task: "Build devtool list command"
    detail: >
      @app.command(name="list")
      def list_runs(
          status: str | None = typer.Option(None, help="Filter by status"),
      ):
          """List all runs."""
          storage = get_storage()
          registry = storage.load()
          runs = registry.all()

          if status:
              runs = [r for r in runs if r.status == status]

          if not runs:
              console.print("[dim]No runs found.[/dim]")
              return

          table = Table(title="Runs")
          table.add_column("ID", style="cyan")
          table.add_column("Status", style="bold")
          table.add_column("Prompt")
          table.add_column("Created")

          STATUS_COLORS = {
              "pending": "yellow",
              "running": "blue",
              "done": "green",
              "failed": "red",
          }

          for r in runs:
              color = STATUS_COLORS.get(r.status, "white")
              table.add_row(
                  r.run_id,
                  f"[{color}]{r.status}[/{color}]",
                  r.prompt[:50] + ("..." if len(r.prompt) > 50 else ""),
                  r.created_at[:19],
              )

          console.print(table)

  4:
    task: "Build devtool get and devtool clear"
    detail: >
      @app.command()
      def get(run_id: str = typer.Argument(..., help="Run ID to inspect")):
          """Show details of a specific run."""
          from devtool.errors import RunNotFoundError
          storage = get_storage()
          registry = storage.load()

          try:
              r = registry.get(run_id)
          except RunNotFoundError:
              console.print(f"[red]Error:[/red] Run '{run_id}' not found")
              raise typer.Exit(code=1)

          console.print(f"[bold]Run ID:[/bold]  {r.run_id}")
          console.print(f"[bold]Status:[/bold]  {r.status}")
          console.print(f"[bold]Prompt:[/bold]  {r.prompt}")
          console.print(f"[bold]Created:[/bold] {r.created_at}")
          if r.result:
              console.print(f"\n[bold]Result:[/bold]\n{r.result}")
          if r.error:
              console.print(f"\n[bold red]Error:[/bold red]\n{r.error}")

      @app.command()
      def clear(
          force: bool = typer.Option(False, "--force", "-f",
                                      help="Skip confirmation"),
      ):
          """Clear all stored runs."""
          if not force:
              confirm = typer.confirm("This will delete all runs. Continue?")
              if not confirm:
                  raise typer.Abort()

          storage = get_storage()
          storage.clear()
          console.print("[green]Cleared all runs.[/green]")

  5:
    task: "Build devtool config show and devtool version"
    detail: >
      @app.command()
      def version():
          """Show the current version."""
          from devtool import __version__
          console.print(f"devtool [cyan]v{__version__}[/cyan]")

      config_app = typer.Typer(help="Configuration commands.")
      app.add_typer(config_app, name="config")

      @config_app.command(name="show")
      def config_show():
          """Show current configuration."""
          table = Table(title="Configuration")
          table.add_column("Setting", style="cyan")
          table.add_column("Value")

          table.add_row("app_env", settings.app_env)
          table.add_row("log_level", settings.log_level)
          table.add_row("max_runs", str(settings.max_runs))
          table.add_row("openai_api_key", "***" + settings.openai_api_key[-4:]
                                           if settings.openai_api_key else "[not set]")

          console.print(table)

  6:
    task: "Add global error handling"
    detail: >
      Wrap the app with a global exception handler so users see clean error messages,
      not stack traces:

      def main():
          try:
              app()
          except DevToolError as e:
              console.print(f"[red]Error:[/red] {e}")
              raise SystemExit(1)
          except KeyboardInterrupt:
              console.print("\n[dim]Interrupted.[/dim]")
              raise SystemExit(0)

      if __name__ == "__main__":
          main()

      Update pyproject.toml entry point:
        devtool = "devtool.cli:main"

  7:
    task: "Write tests/test_cli.py"
    detail: >
      from typer.testing import CliRunner
      from devtool.cli import app

      runner = CliRunner()

      def test_version():
          result = runner.invoke(app, ["version"])
          assert result.exit_code == 0
          assert "devtool" in result.output

      def test_run_command(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)   # isolate storage to tmp dir
          result = runner.invoke(app, ["run", "hello world"])
          assert result.exit_code == 0
          assert "Done" in result.output

      def test_list_empty(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)
          result = runner.invoke(app, ["list"])
          assert result.exit_code == 0
          assert "No runs found" in result.output

      def test_get_not_found(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)
          result = runner.invoke(app, ["get", "nonexistent"])
          assert result.exit_code == 1
          assert "not found" in result.output

      def test_run_then_list(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)
          runner_instance = CliRunner()
          runner_instance.invoke(app, ["run", "test prompt"])
          result = runner_instance.invoke(app, ["list"])
          assert "test prompt" in result.output

      pytest tests/ -v   # all tests should pass

  8:
    task: "Update README.md"
    detail: >
      Full README with:
        - What devtool is (2 sentences)
        - Installation: pip install devtool
        - All commands with examples
        - Development setup
        - Running tests: pytest tests/ --cov=devtool

      This README is the face of the package on PyPI and GitHub.
```

---

## Manual Test Walkthrough

Once everything is built, do this by hand:

```bash
# Start fresh
devtool clear --force

# Submit runs
devtool run "Write a haiku about Python"
devtool run "What is asyncio?" --no-wait
devtool run "Explain Pydantic in one sentence"

# Inspect
devtool list
devtool list --status done
devtool get <run_id_from_above>

# Config
devtool config show

# Version
devtool version
```

Everything should work, look clean in the terminal, and persist across terminal sessions.

---

## Done When

```yaml
done_when:
  - All 6 commands work end-to-end from the terminal
  - devtool list shows a formatted Rich table
  - devtool get <invalid_id> exits with code 1 and a clean error message
  - All CLI tests pass
  - README is complete and accurate
  - pip install -e . still works cleanly

graduation_test: >
  Close your editor. Open a fresh terminal. Activate the venv.
  Run devtool --help. Read the output.
  Can you explain every command, every file in the project, and every design decision
  without looking at your notes?

  If yes — you're ready for the FastAPI ramp.
  The skills you just built (Pydantic models, async, pytest, packaging, error handling)
  are exactly what FastAPI, LangGraph, and every other ramp assumes.
```
