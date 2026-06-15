# Stage 04 — Files, Config, and Environment

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — add agentlog/config.py and agentlog/storage.py
**Depends on:** Stage 03 (Pydantic Run and RunRegistry with dump_all/load_all)

---

## The Story So Far

You have validated Run models and a RunRegistry that can serialize itself to a list of dicts. But every time you restart the Python process, everything is gone. agentlog needs persistence — runs saved to disk so you can come back to them. And it needs configuration — an OpenAI API key, environment settings — loaded from the environment without hardcoding.

This stage builds those two pieces: `agentlog/config.py` loads settings from a `.env` file, and `agentlog/storage.py` saves and loads RunRegistry data as JSON. By the end, runs survive restarts.

---

## Why this stage exists

Every real Python program reads configuration from somewhere and persists data somewhere. These aren't advanced features — they're the minimum for a tool that actually works. This stage builds those skills using the tools professional Python code actually uses: `pathlib` for paths, `pydantic-settings` for type-safe config, and `json` for storage. You'll use all three in every ramp that follows.

---

## Concepts This Stage Teaches

```yaml
concepts:
  pathlib.Path:
    what: >
      An object-oriented interface to file paths.
      Path(".agentlog_data") / "runs.json" creates a path.
      p.exists(), p.read_text(), p.write_text(), p.mkdir(parents=True, exist_ok=True).
    why: >
      The old way (os.path.join, open()) is fragile and verbose. pathlib is the modern
      standard. Every path operation in agentlog uses it.
    csharp_equivalent: "Path.Combine() + File.ReadAllText() + Directory.CreateDirectory()"

  os.environ:
    what: >
      A dict-like object. os.environ["KEY"] raises KeyError if missing.
      os.environ.get("KEY", "default") returns a default safely.
    why: >
      Environment variables are how secrets and config reach a running process
      without being hardcoded. Every cloud deployment and Docker container uses this.

  python-dotenv:
    what: >
      load_dotenv() reads a .env file and injects its contents as environment variables.
      Must be called before any os.environ reads.
    why: >
      In local dev you don't want to export vars manually every session.
      .env stores them. load_dotenv() loads them. .gitignore keeps them out of git.

  pydantic-settings BaseSettings:
    what: >
      A Pydantic model that reads fields from environment variables automatically.
      class Settings(BaseSettings): openai_api_key: str reads OPENAI_API_KEY from env.
    why: >
      Combines Pydantic validation with environment variable loading.
      Type-safe config with zero extra code. The production pattern used in
      FastAPI, LangGraph, and every serious Python service.

  json module:
    what: >
      json.dumps(obj) → JSON string
      json.loads(string) → Python object
      json.dump(obj, file) → write to file object
      json.load(file) → read from file object
    why: >
      JSON is the universal data exchange format. Pydantic's model_dump_json() uses it.
      FileStorage uses it. HTTP APIs return it. You need to handle it directly
      when working outside of Pydantic's serialization layer.

  Context managers (with statement):
    what: >
      with open("file.txt") as f: f.read()
      The with block guarantees f.close() is called even if an exception occurs.
    why: >
      Every file operation should use with. Without it, file handles leak on exceptions.
      Context managers appear everywhere — database connections, locks, HTTP sessions.

  yaml (PyYAML):
    what: >
      yaml.safe_load(string) → Python dict/list
      yaml.dump(obj) → YAML string
    why: >
      Config files are often YAML — docker-compose.yml, GitHub Actions, LangGraph config.
      You need to read and write them.
```

---

## Setup

```bash
pip install python-dotenv pydantic-settings pyyaml
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create a .env file with agentlog's config"
    why: >
      agentlog will eventually make real OpenAI API calls. The API key lives here —
      never in source code, never in git. Set up the pattern now so it's in place
      when the runner makes its first live call in Stage 06.
    detail: >
      # agentlog/.env
      OPENAI_API_KEY=sk-fake-key-for-now
      APP_ENV=development
      LOG_LEVEL=INFO
      MAX_RUNS=100

      Add to .gitignore (already there from Stage 01):
        .env

  2:
    task: "Write agentlog/config.py with BaseSettings"
    why: >
      agentlog needs its API key and settings wherever it runs — in the CLI, in the runner,
      in tests. A module-level settings singleton loads once on import and is available
      everywhere. Type-safe via Pydantic: MAX_RUNS arrives as int, not "100".
    detail: >
      # agentlog/config.py

      from pydantic_settings import BaseSettings
      from pydantic import Field

      class Settings(BaseSettings):
          openai_api_key: str = Field(description="OpenAI API key")
          app_env: str = "development"
          log_level: str = "INFO"
          max_runs: int = 100

          model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

      # Module-level singleton — created once on import, shared everywhere
      settings = Settings()

      Test in main.py:
        from agentlog.config import settings
        print(settings.app_env)         # "development"
        print(settings.max_runs)        # 100 (int, not string — Pydantic coerced it)
        print(settings.openai_api_key)  # "sk-fake-key-for-now"

  3:
    task: "Write agentlog/storage.py — persist runs to disk"
    why: >
      RunRegistry.dump_all() already returns a list of dicts. FileStorage just needs
      to write that list to a JSON file and read it back. This is the persistence layer
      agentlog was missing — after this step, runs survive process restarts.
    detail: >
      # agentlog/storage.py

      import json
      from pathlib import Path
      from agentlog.models import Run, RunRegistry

      class FileStorage:
          def __init__(self, data_dir: str | Path = ".agentlog_data") -> None:
              self.data_dir = Path(data_dir)
              self.data_dir.mkdir(parents=True, exist_ok=True)
              self._runs_file = self.data_dir / "runs.json"

          def save(self, registry: RunRegistry) -> None:
              data = registry.dump_all()
              with open(self._runs_file, "w") as f:
                  json.dump(data, f, indent=2)

          def load(self) -> RunRegistry:
              registry = RunRegistry()
              if not self._runs_file.exists():
                  return registry
              raw = json.loads(self._runs_file.read_text())
              registry.load_all(raw)
              return registry

          def clear(self) -> None:
              if self._runs_file.exists():
                  self._runs_file.unlink()

          @property
          def size_bytes(self) -> int:
              if not self._runs_file.exists():
                  return 0
              return self._runs_file.stat().st_size

  4:
    task: "Test the full persistence round-trip"
    why: >
      This is the first time agentlog's data actually survives a restart.
      Run this, check the file on disk, restart the process, load it back —
      confirm the data is intact. This is Stage 05's test suite in manual form.
    detail: >
      from agentlog.storage import FileStorage
      from agentlog.models import Run, RunRegistry

      storage = FileStorage()

      # Build a registry with some state
      registry = RunRegistry()
      registry.add(Run(run_id="r1", prompt="first run"))
      registry.add(Run(run_id="r2", prompt="second run"))
      registry.get("r1").mark_done("result one")
      registry.get("r2").mark_failed("timeout")

      # Save to disk
      storage.save(registry)
      print(f"Saved {storage.size_bytes} bytes to disk")

      # Load back — simulates a fresh process start
      loaded = storage.load()
      print(len(loaded))                   # 2
      print(loaded.get("r1").status)       # "done"
      print(loaded.get("r1").result)       # "result one"
      print(loaded.get("r2").status)       # "failed"

      # Open .agentlog_data/runs.json in your editor and read it.
      # This is exactly what's stored. No magic.

  5:
    task: "Add YAML config support to FileStorage"
    why: >
      agentlog will eventually support per-project config — model name, temperature,
      custom tags. YAML is the natural format for that. Add it to FileStorage now
      while the pattern is fresh.
    detail: >
      Add to FileStorage:

          def save_config(self, config: dict) -> None:
              import yaml
              config_file = self.data_dir / "config.yaml"
              config_file.write_text(yaml.dump(config, default_flow_style=False))

          def load_config(self) -> dict:
              import yaml
              config_file = self.data_dir / "config.yaml"
              if not config_file.exists():
                  return {}
              return yaml.safe_load(config_file.read_text()) or {}

      Test:
        storage.save_config({"model": "gpt-4o-mini", "temperature": 0})
        config = storage.load_config()
        print(config["model"])   # "gpt-4o-mini"

  6:
    task: "Load a real env var from the OS"
    why: >
      The .env file is for local development. In production (Docker, CI, cloud),
      env vars are injected by the platform — no .env file. Confirm agentlog
      reads both correctly, so you understand the full deployment pattern.
    detail: >
      In your terminal (with .venv active):
        export DEVTOOL_SECRET=mysecret

      Add to Settings:
        agentlog_secret: str | None = None

      Print it from main.py. Set it, run. Unset it (unset DEVTOOL_SECRET), run again.
      Confirm None is the value when unset.

      This is exactly how Docker containers and CI systems inject secrets —
      no .env file, just environment variables passed at container startup.
```

---

## Done When

```yaml
done_when:
  - settings loads correctly from .env — types coerced, no manual os.environ calls
  - FileStorage saves RunRegistry and loads it back with full round-trip integrity
  - You've opened runs.json on disk and read the raw JSON
  - You can navigate the filesystem with pathlib — mkdir, exists, read_text, write_text, unlink
  - You understand the with statement and can explain why it exists
  - You understand the difference between env vars in .env vs the OS shell
  - RunRegistry data survives a process restart
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  pathlib: "Path.Combine() + File.ReadAllText() + Directory.CreateDirectory() — unified"
  BaseSettings: "IConfiguration with typed options — but zero boilerplate"
  json.dumps/loads: "JsonSerializer.Serialize/Deserialize — but working on dicts"
  with open(): "using (var f = File.Open(...)) — same guarantee, different syntax"
  .env file: "Like appsettings.Development.json — local overrides, gitignored"
```

---

## Open Questions

```yaml
open_questions:
  - What happens if .env doesn't exist when Settings() is instantiated?
    Does it error or silently use defaults? Test it.
  - How would you handle config that works in both dev (.env) and prod (real env vars)?
    Does BaseSettings handle this automatically?
  - What is tempfile.NamedTemporaryFile() and when would you use it in tests?
    (Hint: Stage 05 uses tmp_path instead — understand why)
  - What is the difference between Path.read_text() and open() + f.read()?
    Is one better? Are there cases where open() is necessary?
```
