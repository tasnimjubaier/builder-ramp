# Stage 05 — Errors and Testing with pytest

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — add tests/ to devtool/
**Depends on:** Stage 04 (models, config, storage all built)

Two things that separate scripts from professional code: deliberate error handling and automated tests. Python's exception system is richer than C#'s — you can define exception hierarchies, catch multiple types, and add context. pytest is the standard test runner — cleaner than unittest, with fixtures and parametrize built in. Every capstone in every ramp depends on pytest.

---

## What You're Building

Add `devtool/errors.py` — a custom exception hierarchy for devtool. Add `tests/` — a test suite for the models, storage, and config you've built. By the end, `pytest tests/` passes and you have real test coverage on your codebase.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Exception hierarchy:
    what: >
      Python exceptions are classes. You define custom ones by inheriting from Exception.
      You can create a hierarchy: AppError → RunError → RunNotFoundError.
      Catching AppError catches all subclasses.
    why: >
      Custom exceptions let callers catch specific error types and handle them differently.
      raise RunNotFoundError("r1") is more informative than raise ValueError("not found").
      FastAPI's HTTPException is just a custom exception class — understanding this
      demystifies how error handlers work.

  raise, raise from, and re-raise:
    what: >
      raise MyError("message")          — raise a new exception
      raise MyError("msg") from original — chain to the original cause (shows both in traceback)
      raise                             — re-raise the current exception (inside except block)
    why: >
      raise ... from ... preserves the original error context in the traceback.
      This is how you wrap low-level exceptions (json.JSONDecodeError) in meaningful
      application errors (StorageCorruptedError) without losing the root cause.

  try / except / else / finally:
    what: >
      try: risky code
      except SomeError as e: handle it
      else: runs if no exception was raised
      finally: always runs — cleanup code
    why: >
      else is underused but valuable — it's the "happy path" code that only runs
      when no exception occurred, keeping it separate from exception handling.
      finally is how you close resources when you can't use with.

  pytest basics:
    what: >
      Functions starting with test_ in files starting with test_ are collected and run.
      assert statement — pytest rewrites it to show exactly what failed.
      pytest.raises(ExceptionType) — assert that code raises a specific exception.
    why: >
      This is the testing pattern used in every OSS project you'll read or contribute to.
      The assert rewriting is why pytest's output is so clear — you see actual vs expected.

  pytest fixtures:
    what: >
      A function decorated with @pytest.fixture that provides a resource to tests.
      Tests declare the fixture as a parameter — pytest injects it.
    why: >
      Fixtures replace setUp/tearDown. They're more composable and explicit.
      A storage fixture creates a temp FileStorage. Every test that needs storage
      gets a fresh one automatically.

  pytest.mark.parametrize:
    what: >
      Run the same test with multiple inputs:
      @pytest.mark.parametrize("status", ["pending", "running", "done", "failed"])
      def test_valid_status(status): ...
    why: >
      One test function, four test cases. Better than copy-pasting the same test four times.
      Used in every serious test suite.

  tmp_path fixture:
    what: >
      A built-in pytest fixture that provides a temporary directory that's cleaned up after each test.
    why: >
      File-based tests (like FileStorage tests) must not write to real disk locations.
      tmp_path gives you an isolated throwaway directory per test.
```

---

## Setup

```bash
pip install pytest pytest-cov
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write devtool/errors.py"
    detail: >
      # devtool/errors.py

      class DevToolError(Exception):
          """Base exception for all devtool errors."""
          pass

      class RunError(DevToolError):
          """Errors related to Run operations."""
          def __init__(self, run_id: str, message: str) -> None:
              self.run_id = run_id
              super().__init__(f"Run '{run_id}': {message}")

      class RunNotFoundError(RunError):
          def __init__(self, run_id: str) -> None:
              super().__init__(run_id, "not found")

      class RunAlreadyExistsError(RunError):
          def __init__(self, run_id: str) -> None:
              super().__init__(run_id, "already exists")

      class StorageError(DevToolError):
          """Errors related to storage operations."""
          pass

      class StorageCorruptedError(StorageError):
          def __init__(self, path: str, cause: Exception) -> None:
              self.path = path
              super().__init__(f"Storage corrupted at '{path}'") from cause

      class ConfigError(DevToolError):
          """Errors related to configuration."""
          pass

  2:
    task: "Use custom errors in RunRegistry and FileStorage"
    detail: >
      In RunRegistry (models.py):

        from devtool.errors import RunNotFoundError, RunAlreadyExistsError

        def get(self, run_id: str) -> Run:
            if run_id not in self._runs:
                raise RunNotFoundError(run_id)
            return self._runs[run_id]

        def add(self, run: Run) -> None:
            if run.run_id in self._runs:
                raise RunAlreadyExistsError(run.run_id)
            self._runs[run.run_id] = run

      In FileStorage (storage.py), wrap json.loads with StorageCorruptedError:

        def load(self) -> RunRegistry:
            registry = RunRegistry()
            if not self._runs_file.exists():
                return registry
            try:
                raw = json.loads(self._runs_file.read_text())
            except json.JSONDecodeError as e:
                raise StorageCorruptedError(str(self._runs_file), e)
            registry.load_all(raw)
            return registry

  3:
    task: "Create the tests/ directory and conftest.py"
    detail: >
      mkdir tests
      touch tests/__init__.py
      touch tests/conftest.py

      # tests/conftest.py

      import pytest
      from pathlib import Path
      from devtool.models import Run, RunRegistry

      @pytest.fixture
      def sample_run() -> Run:
          return Run(run_id="test-001", prompt="test prompt")

      @pytest.fixture
      def populated_registry() -> RunRegistry:
          registry = RunRegistry()
          registry.add(Run(run_id="r1", prompt="first"))
          registry.add(Run(run_id="r2", prompt="second"))
          return registry

      conftest.py is automatically loaded by pytest — fixtures defined here
      are available to all test files in the directory and below.

  4:
    task: "Write tests/test_models.py"
    detail: >
      import pytest
      from devtool.models import Run, RunRegistry
      from devtool.errors import RunNotFoundError, RunAlreadyExistsError

      def test_run_initial_status(sample_run):
          assert sample_run.status == "pending"
          assert sample_run.result is None

      def test_run_mark_done(sample_run):
          sample_run.mark_done("my result")
          assert sample_run.status == "done"
          assert sample_run.result == "my result"

      def test_run_mark_failed(sample_run):
          sample_run.mark_failed("something broke")
          assert sample_run.status == "failed"
          assert sample_run.error == "something broke"

      def test_run_prompt_stripped():
          r = Run(run_id="r1", prompt="  hello  ")
          assert r.prompt == "hello"   # field_validator strips it

      def test_run_empty_prompt_raises():
          with pytest.raises(Exception):   # Pydantic ValidationError
              Run(run_id="r1", prompt="   ")

      def test_registry_add_and_get(populated_registry):
          run = populated_registry.get("r1")
          assert run.prompt == "first"

      def test_registry_get_not_found(populated_registry):
          with pytest.raises(RunNotFoundError) as exc_info:
              populated_registry.get("nonexistent")
          assert "nonexistent" in str(exc_info.value)

      def test_registry_add_duplicate(populated_registry):
          with pytest.raises(RunAlreadyExistsError):
              populated_registry.add(Run(run_id="r1", prompt="duplicate"))

      def test_registry_len(populated_registry):
          assert len(populated_registry) == 2

      def test_registry_contains(populated_registry):
          assert "r1" in populated_registry
          assert "r999" not in populated_registry

      @pytest.mark.parametrize("status", ["pending", "running", "done", "failed"])
      def test_valid_run_status(status):
          r = Run(run_id="r1", prompt="test", status=status)
          assert r.status == status

  5:
    task: "Write tests/test_storage.py"
    detail: >
      import pytest
      from devtool.models import Run, RunRegistry
      from devtool.storage import FileStorage
      from devtool.errors import StorageCorruptedError

      @pytest.fixture
      def storage(tmp_path):
          return FileStorage(data_dir=tmp_path / "data")

      def test_save_and_load(storage):
          registry = RunRegistry()
          registry.add(Run(run_id="r1", prompt="test"))
          registry.get("r1").mark_done("result")

          storage.save(registry)

          loaded = storage.load()
          assert len(loaded) == 1
          assert loaded.get("r1").status == "done"
          assert loaded.get("r1").result == "result"

      def test_load_empty(storage):
          registry = storage.load()
          assert len(registry) == 0

      def test_corrupted_storage(storage, tmp_path):
          # Write invalid JSON to the storage file
          (tmp_path / "data" / "runs.json").write_text("not valid json {{{")

          with pytest.raises(StorageCorruptedError):
              storage.load()

      def test_clear(storage):
          registry = RunRegistry()
          registry.add(Run(run_id="r1", prompt="test"))
          storage.save(registry)
          storage.clear()

          loaded = storage.load()
          assert len(loaded) == 0

  6:
    task: "Run pytest and read the output"
    detail: >
      pytest tests/ -v

      The -v flag shows each test individually with pass/fail.
      Read the output carefully — understand the format.

      Run with coverage:
        pytest tests/ --cov=devtool --cov-report=term-missing

      The coverage report shows which lines in your code are NOT covered by tests.
      Add a test to cover any obvious gap.

  7:
    task: "Write one deliberately failing test and read pytest's output"
    detail: >
      def test_intentional_failure():
          r = Run(run_id="r1", prompt="test")
          assert r.status == "done"   # will fail — status is "pending"

      Run pytest. Read the assertion error output carefully.
      pytest shows: assert "pending" == "done" with the actual values.
      This is pytest's assertion rewriting — much more useful than "AssertionError".

      Delete the test after reading the output.
```

---

## Done When

```yaml
done_when:
  - pytest tests/ -v shows all tests passing
  - You have tests for RunRegistry, FileStorage, and error cases
  - You understand @pytest.fixture and can write a new one from memory
  - You understand pytest.raises() and have used it for at least 3 error cases
  - You've used @pytest.mark.parametrize at least once
  - You understand the exception hierarchy and can add a new exception type
  - Coverage report shows >80% for devtool/models.py and devtool/storage.py
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  custom_exceptions: "Same pattern — inherit from Exception, add constructor"
  raise_from: "Like throw new AppException(...) { InnerException = e }"
  try/finally: "Identical concept and purpose"
  pytest_fixture: "Like [TestInitialize] or constructor injection in xUnit"
  pytest.raises: "Like Assert.Throws<ExceptionType>(() => ...)"
  parametrize: "Like [Theory] + [InlineData(...)] in xUnit"
  tmp_path: "Like Path.GetTempPath() but scoped and auto-cleaned per test"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between pytest.raises() as a context manager vs as a decorator?
  - What does conftest.py do specifically — where does pytest look for it?
  - How do you mock a function in pytest? (Hint: pytest-mock or unittest.mock.patch)
    You'll need this in the FastAPI ramp for dependency overrides.
  - What is the difference between scope="function" and scope="session" on a fixture?
    When would a session-scoped fixture make sense?
```
