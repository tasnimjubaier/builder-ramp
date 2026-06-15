# Stage 08 — Capstone: agentlog CLI

**Status:** not started
**Estimated time:** 3–4 days
**Type:** synthesis — complete agentlog using all 7 stages
**Depends on:** all previous stages

---

## The Story So Far

You have a fully working project: validated models, persistent storage, a type-safe config layer, an async runner, a test suite, and a packaged entry point. The one missing piece is the CLI itself — the user-facing interface that ties everything together.

This stage builds that. When it's done, agentlog is a real, installable tool:

```bash
agentlog run "Write a haiku about Python"
agentlog list
agentlog list --status done
agentlog get <run_id>
agentlog clear
agentlog config show
agentlog version
```

Every command maps to something you already built. This stage is the wiring.

---

## What changes vs earlier stages

There's one new thing in this stage: a **real LLM call**. The runner's `asyncio.sleep()` placeholder gets replaced with an actual OpenAI API call. Everything else — models, storage, errors, async — is already in place. The LLM call slots in cleanly because the architecture was designed for it.

---

## Build Steps

```yaml
steps:
  1:
    task: "Add Rich and wire the console"
    why: >
      agentlog needs colored terminal output — green for done, red for failed,
      yellow for pending. Rich handles this with zero effort.
      Define a module-level console once; use it everywhere in cli.py.
    detail: >
      pip install rich  (already in pyproject.toml — already installed)

      # agentlog/cli.py — expand from the Stage 07 stub

      import typer
      from rich.console import Console

      app = typer.Typer(name="agentlog", help="Manage AI agent runs from the terminal.")
      console = Console()

      STATUS_COLORS = {
          "pending":   "yellow",
          "running":   "blue",
          "streaming": "cyan",
          "done":      "green",
          "failed":    "red",
      }

      Quick test:
        console.print("[green]Done[/green] [bold]run-001[/bold]")
        console.print(f"[{STATUS_COLORS['failed']}]failed[/{STATUS_COLORS['failed']}]")

  2:
    task: "Build agentlog run"
    why: >
      The primary command. Takes a prompt, creates a Run, executes it via RunRunner,
      saves it, prints the result. This is the full agentlog pipeline in one command.
    detail: >
      import asyncio, uuid
      from agentlog.models import Run
      from agentlog.runner import RunRunner
      from agentlog.storage import FileStorage
      from agentlog.errors import DevToolError

      def get_storage() -> FileStorage:
          return FileStorage(data_dir=".agentlog_data")

      @app.command()
      def run(
          prompt: str = typer.Argument(..., help="The prompt to run"),
          wait: bool = typer.Option(True, "--wait/--no-wait",
                                    help="Wait for result or submit and return immediately"),
      ):
          """Submit a new run."""
          storage = get_storage()
          runner = RunRunner(storage)
          new_run = Run(run_id=str(uuid.uuid4())[:8], prompt=prompt)

          if wait:
              result = asyncio.run(runner.execute(new_run))
              color = STATUS_COLORS.get(result.status, "white")
              console.print(f"[{color}]{result.status}[/{color}] [{result.run_id}]")
              if result.result:
                  console.print(result.result)
              if result.error:
                  console.print(f"[red]Error:[/red] {result.error}")
          else:
              registry = storage.load()
              registry.add(new_run)
              storage.save(registry)
              console.print(f"[yellow]Submitted[/yellow] [{new_run.run_id}]")
              console.print(f"Check status: agentlog get {new_run.run_id}")

      Test:
        agentlog run "hello world"
        agentlog run "hello" --no-wait

  3:
    task: "Build agentlog list"
    why: >
      The most-used command. Lists all runs in a Rich table with colored status.
      Optional --status filter so you can see just what failed, or just what's pending.
    detail: >
      from rich.table import Table

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
          table.add_column("ID",      style="cyan")
          table.add_column("Status",  style="bold")
          table.add_column("Prompt")
          table.add_column("Created")

          for r in runs:
              color = STATUS_COLORS.get(r.status, "white")
              table.add_row(
                  r.run_id,
                  f"[{color}]{r.status}[/{color}]",
                  r.prompt[:60] + ("..." if len(r.prompt) > 60 else ""),
                  r.created_at[:19],
              )

          console.print(table)

      Test:
        agentlog list
        agentlog list --status done
        agentlog list --status failed

  4:
    task: "Build agentlog get and agentlog clear"
    why: >
      get shows the full detail of one run — result, error, token usage.
      clear is the reset command — start fresh. Both need clean error handling:
      get should exit with code 1 on missing run, clear should confirm before deleting.
    detail: >
      @app.command()
      def get(run_id: str = typer.Argument(..., help="Run ID to inspect")):
          """Show details of a specific run."""
          from agentlog.errors import RunNotFoundError
          storage = get_storage()
          registry = storage.load()

          try:
              r = registry.get(run_id)
          except RunNotFoundError:
              console.print(f"[red]Error:[/red] Run '{run_id}' not found")
              raise typer.Exit(code=1)

          color = STATUS_COLORS.get(r.status, "white")
          console.print(f"[bold]ID:[/bold]       {r.run_id}")
          console.print(f"[bold]Status:[/bold]   [{color}]{r.status}[/{color}]")
          console.print(f"[bold]Prompt:[/bold]   {r.prompt}")
          console.print(f"[bold]Created:[/bold]  {r.created_at[:19]}")
          if r.usage.total > 0:
              console.print(f"[bold]Tokens:[/bold]   {r.usage.total} "
                            f"(prompt: {r.usage.prompt_tokens}, "
                            f"completion: {r.usage.completion_tokens})")
          if r.result:
              console.print(f"\n[bold]Result:[/bold]\n{r.result}")
          if r.error:
              console.print(f"\n[bold red]Error:[/bold red]\n{r.error}")

      @app.command()
      def clear(
          force: bool = typer.Option(False, "--force", "-f", help="Skip confirmation"),
      ):
          """Clear all stored runs."""
          if not force:
              confirm = typer.confirm("This will delete all runs. Continue?")
              if not confirm:
                  raise typer.Abort()
          get_storage().clear()
          console.print("[green]Cleared all runs.[/green]")

      Test:
        agentlog get <run_id>
        agentlog get nonexistent    # should exit 1 with clean message
        agentlog clear              # should prompt for confirmation
        agentlog clear --force      # should clear without prompt

  5:
    task: "Build agentlog config show and agentlog version"
    why: >
      config show is for debugging — confirm the right API key and settings are loaded.
      version is for CI and users. Both are one-liners that round out the CLI.
    detail: >
      @app.command()
      def version():
          """Show the current version."""
          from agentlog import __version__
          console.print(f"agentlog [cyan]v{__version__}[/cyan]")

      config_app = typer.Typer(help="Configuration commands.")
      app.add_typer(config_app, name="config")

      @config_app.command(name="show")
      def config_show():
          """Show current configuration."""
          from agentlog.config import settings
          from rich.table import Table

          table = Table(title="Configuration")
          table.add_column("Setting", style="cyan")
          table.add_column("Value")

          table.add_row("app_env",   settings.app_env)
          table.add_row("log_level", settings.log_level)
          table.add_row("max_runs",  str(settings.max_runs))

          key = settings.openai_api_key
          masked = ("***" + key[-4:]) if key and len(key) > 4 else "[not set]"
          table.add_row("openai_api_key", masked)

          console.print(table)

      Test:
        agentlog version
        agentlog config show

  6:
    task: "Add global error handling"
    why: >
      Without this, any unhandled DevToolError prints a raw Python traceback.
      With it, the user sees a clean one-line error message.
      KeyboardInterrupt (Ctrl+C) should exit cleanly, not dump a traceback.
    detail: >
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
        agentlog = "agentlog.cli:main"

      Reinstall: pip install -e .

  7:
    task: "Replace the simulated LLM call with a real OpenAI call"
    why: >
      This is what everything has been building toward. The runner's asyncio.sleep()
      placeholder gets replaced with a real API call. The architecture doesn't change —
      execute() is still async, still returns a Run, still saves to storage.
      The LLM call slots into exactly the place designed for it.
    detail: >
      pip install openai

      # agentlog/runner.py — replace execute()

      from openai import AsyncOpenAI
      from agentlog.config import settings

      class RunRunner:
          def __init__(self, storage: FileStorage) -> None:
              self.storage = storage
              self._client = AsyncOpenAI(api_key=settings.openai_api_key)

          async def execute(self, run: Run) -> Run:
              run.status = "running"
              try:
                  response = await self._client.chat.completions.create(
                      model="gpt-4o-mini",
                      messages=[{"role": "user", "content": run.prompt}],
                  )
                  content = response.choices[0].message.content or ""
                  run.mark_done(content)
                  run.usage.prompt_tokens = response.usage.prompt_tokens
                  run.usage.completion_tokens = response.usage.completion_tokens
              except Exception as e:
                  run.mark_failed(str(e))
              return run

          async def execute_batch(self, runs: list[Run]) -> list[Run]:
              return await asyncio.gather(*[self.execute(run) for run in runs])

      Update .env:
        OPENAI_API_KEY=sk-your-real-key

      Test:
        agentlog run "Write a haiku about Python"
        agentlog get <run_id>   # should show real result and token counts

  8:
    task: "Write tests/test_cli.py"
    why: >
      The CLI is the user-facing layer. Test it with Typer's CliRunner —
      invoke commands programmatically, assert on exit codes and output.
      Mock the runner so tests don't make real API calls.
    detail: >
      from typer.testing import CliRunner
      from unittest.mock import patch, AsyncMock
      from agentlog.cli import app
      from agentlog.models import Run

      runner = CliRunner()

      def test_version():
          result = runner.invoke(app, ["version"])
          assert result.exit_code == 0
          assert "agentlog" in result.output

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

      def test_clear_with_force(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)
          result = runner.invoke(app, ["clear", "--force"])
          assert result.exit_code == 0
          assert "Cleared" in result.output

      async def mock_execute(run: Run) -> Run:
          run.mark_done("mocked result")
          return run

      def test_run_command(tmp_path, monkeypatch):
          monkeypatch.chdir(tmp_path)
          with patch("agentlog.cli.RunRunner") as MockRunner:
              instance = MockRunner.return_value
              instance.execute = AsyncMock(side_effect=mock_execute)
              result = runner.invoke(app, ["run", "hello world"])
          assert result.exit_code == 0
          assert "done" in result.output

      pytest tests/ -v   # all tests should pass

  9:
    task: "Full manual walkthrough"
    why: >
      Tests confirm correctness. This confirms the user experience.
      Do this after all tests pass — it's the graduation test.
    detail: >
      agentlog clear --force

      agentlog run "Write a haiku about Python"
      agentlog run "What is asyncio in one sentence?"
      agentlog run "Explain Pydantic briefly" --no-wait

      agentlog list
      agentlog list --status done
      agentlog list --status pending

      agentlog get <run_id_from_above>
      agentlog get nonexistent

      agentlog config show
      agentlog version
      agentlog --help

      Everything should work, output should be clean and colored,
      data should persist across terminal sessions.
```

---

## Done When

```yaml
done_when:
  - All 6 commands work end-to-end from the terminal
  - agentlog list shows a formatted Rich table with colored status
  - agentlog get <invalid_id> exits with code 1 and a clean message — no traceback
  - agentlog run makes a real OpenAI API call and stores the result
  - Token usage appears in agentlog get output
  - All CLI tests pass with mocked runner
  - README documents all commands

graduation_test: >
  Close your editor. Open a fresh terminal. Activate the venv.
  Run agentlog --help. Read the output.

  Can you explain every command, every file in the project, and every
  design decision you made — without looking at your notes?

  Can you trace a single agentlog run "hello" call through the entire
  codebase: CLI → runner → storage → models → disk → back?

  If yes — you're ready for the agentic ramp.
  LangGraph is async, Pydantic-first, OOP-heavy, and tested with pytest.
  You just built all of that from scratch.
```

---

## Open Questions

```yaml
open_questions:
  - How would you add streaming output to agentlog run?
    (Hint: openai client supports stream=True — returns an async generator)
  - How would you add a agentlog replay <run_id> command that re-runs a stored prompt?
  - How would you add --model flag to agentlog run to choose gpt-4o vs gpt-4o-mini?
  - What would agentlog export look like — all runs to a JSON or CSV file?
    Which layer handles this: CLI, storage, or models?
```
