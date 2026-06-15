# Stage 04 — Files, Config, and Environment

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — extend devtool/ with config loading
**Depends on:** Stage 03 (Pydantic models)

Every real Python program reads configuration from somewhere — environment variables, .env files, YAML/JSON config files. Every program that persists data reads and writes files. This stage builds those skills using the tools that professional Python code actually uses: `pathlib` for paths, `os.environ` for environment variables, `python-dotenv` for .env files, and `json`/`yaml` for structured data.

---

## What You're Building

Add `devtool/config.py` — a settings module that loads configuration from environment variables and a .env file. Add `devtool/storage.py` — a simple file-based storage layer that saves and loads RunRegistry data as JSON.

---

## Concepts This Stage Teaches

```yaml
concepts:
  pathlib.Path:
    what: >
      An object-oriented interface to file paths. Path("/tmp/data") / "runs.json"
      creates a path. p.exists(), p.read_text(), p.write_text(), p.mkdir(parents=True).
    why: >
      The old way (os.path.join, open()) is fragile and verbose. pathlib is the modern
      standard — every professional Python codebase uses it. You'll use it in every
      project to locate config files, read inputs, write outputs.
    csharp_equivalent: "Path.Combine() + File.ReadAllText() + Directory.CreateDirectory()"

  os.environ:
    what: >
      A dict-like object. os.environ["KEY"] raises KeyError if not set.
      os.environ.get("KEY", "default") returns a default safely.
    why: >
      Environment variables are how you inject secrets and config into a running process
      without hardcoding them. Every cloud deployment and Docker container uses this.

  python-dotenv:
    what: >
      load_dotenv() reads a .env file and sets its contents as environment variables.
      Must be called before any os.environ reads that depend on .env values.
    why: >
      In local development you don't want to set env vars manually every time.
      .env file stores them. load_dotenv() loads them. .gitignore keeps them out of git.

  pydantic-settings BaseSettings:
    what: >
      A Pydantic model that reads fields from environment variables automatically.
      class Settings(BaseSettings): openai_key: str — reads OPENAI_KEY from env.
    why: >
      Combines Pydantic validation with environment variable loading. Type-safe config
      with zero extra code. Used in FastAPI, LangGraph projects, and your OSS capstones.
      This is the production pattern — not bare os.environ reads.

  json module:
    what: >
      json.dumps(obj) → JSON string
      json.loads(string) → Python object
      json.dump(obj, file) → write to file object
      json.load(file) → read from file object
    why: >
      JSON is the universal data exchange format. Pydantic's model_dump_json() uses it,
      Redis stores it, HTTP APIs return it. You need to serialize/deserialize it manually
      when working outside of Pydantic.

  yaml (PyYAML):
    what: >
      yaml.safe_load(string) → Python dict/list
      yaml.dump(obj) → YAML string
    why: >
      Config files are often YAML (docker-compose.yml, GitHub Actions workflows,
      LangGraph config). You need to read and write them.

  Context managers (with statement):
    what: >
      with open("file.txt") as f: f.read()
      The with block guarantees f.close() is called even if an exception occurs.
    why: >
      Every file operation should use with. Without it, file handles leak on exceptions.
      Context managers appear everywhere in Python — database connections, locks, HTTP sessions.
      Understanding them makes library code readable.
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
    task: "Create a .env file"
    detail: >
      # devtool/.env
      OPENAI_API_KEY=sk-fake-key-for-now
      APP_ENV=development
      LOG_LEVEL=INFO
      MAX_RUNS=100

      Add .env to .gitignore (already there from Stage 01).

  2:
    task: "Write devtool/config.py with BaseSettings"
    detail: >
      # devtool/config.py

      from pydantic_settings import BaseSettings
      from pydantic import Field

      class Settings(BaseSettings):
          openai_api_key: str = Field(description="OpenAI API key")
          app_env: str = "development"
          log_level: str = "INFO"
          max_runs: int = 100

          model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

      # Module-level singleton — created once on import
      settings = Settings()

      Test in main.py:
        from devtool.config import settings
        print(settings.app_env)        # "development"
        print(settings.max_runs)       # 100 (int, not string — Pydantic coerced it)
        print(settings.openai_api_key) # "sk-fake-key-for-now"

  3:
    task: "Write devtool/storage.py with pathlib"
    detail: >
      # devtool/storage.py

      import json
      from pathlib import Path
      from devtool.models import Run, RunRegistry

      class FileStorage:
          def __init__(self, data_dir: str | Path = ".devtool_data") -> None:
              self.data_dir = Path(data_dir)
              self.data_dir.mkdir(parents=True, exist_ok=True)
              self._runs_file = self.data_dir / "runs.json"

          def save(self, registry: RunRegistry) -> None:
              data = [r.model_dump() for r in registry.all()]
              self._runs_file.write_text(json.dumps(data, indent=2))

          def load(self) -> RunRegistry:
              registry = RunRegistry()
              if not self._runs_file.exists():
                  return registry
              raw = json.loads(self._runs_file.read_text())
              registry.load_all(raw)
              return registry

          def clear(self) -> None:
              if self._runs_file.exists():
                  self._runs_file.unlink()   # delete the file

          @property
          def size_bytes(self) -> int:
              if not self._runs_file.exists():
                  return 0
              return self._runs_file.stat().st_size

  4:
    task: "Test persistence round-trip"
    detail: >
      from devtool.storage import FileStorage
      from devtool.models import Run, RunRegistry

      storage = FileStorage()
      registry = RunRegistry()
      registry.add(Run(run_id="r1", prompt="first"))
      registry.add(Run(run_id="r2", prompt="second"))
      registry.get("r1").mark_done("result one")

      # Save
      storage.save(registry)
      print(f"Saved {storage.size_bytes} bytes")

      # Load back into a fresh registry
      loaded = storage.load()
      print(len(loaded))                  # 2
      print(loaded.get("r1").status)      # "done"
      print(loaded.get("r1").result)      # "result one"

      # Confirm the .devtool_data/runs.json file exists and read it
      from pathlib import Path
      print(Path(".devtool_data/runs.json").read_text())

  5:
    task: "Add YAML config file support"
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
        storage.save_config({"model": "gpt-4o-mini", "temperature": 0, "tags": ["test", "dev"]})
        config = storage.load_config()
        print(config["model"])    # "gpt-4o-mini"
        print(config["tags"])     # ["test", "dev"]

  6:
    task: "Use context manager for file ops"
    detail: >
      Rewrite save() to use the with statement explicitly:

      def save(self, registry: RunRegistry) -> None:
          data = [r.model_dump() for r in registry.all()]
          with open(self._runs_file, "w") as f:
              json.dump(data, f, indent=2)

      This is equivalent to write_text but more explicit about the file handle lifecycle.
      Both are correct — knowing both patterns means you can read any Python codebase.

  7:
    task: "Load a real env var from the OS"
    detail: >
      In your terminal (with .venv active):
        export DEVTOOL_SECRET=mysecret

      Add to Settings:
        devtool_secret: str | None = None

      Print it from main.py. Set it, run. Unset it (unset DEVTOOL_SECRET), run again.
      Confirm that None is the value when it's not set.
      This is exactly how Docker containers and CI systems inject secrets.
```

---

## Done When

```yaml
done_when:
  - Settings loads correctly from .env without any manual os.environ calls
  - FileStorage saves and loads RunRegistry with full round-trip integrity
  - You can navigate the filesystem with pathlib — mkdir, exists, read_text, write_text, unlink
  - You can serialize to/from JSON and YAML
  - You understand the with statement and can explain why it exists
  - You understand the difference between env vars set in .env vs the OS shell
```

---

## Open Questions

```yaml
open_questions:
  - What happens if .env doesn't exist when Settings() is instantiated?
    Does it error or silently use defaults?
  - What is the difference between Path.read_text() and open() + f.read()?
    Is one better? Are there cases where open() is necessary?
  - How would you handle a config file that needs to work in both
    development (reads .env) and production (reads real env vars, no .env file)?
  - What is tempfile.NamedTemporaryFile() and when would you use it in tests?
```
