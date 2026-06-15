# Stage 05 — Errors and Testing with pytest

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — add agentlog/errors.py and tests/
**Depends on:** Stage 04 (models, config, storage all built)

---

## The Story So Far

agentlog can create runs, validate them, save them, and load them back. But what happens when something goes wrong? Right now, `RunRegistry.get()` returns `None` for a missing run — and whoever calls it has to remember to check. `FileStorage.load()` will crash with a raw `json.JSONDecodeError` if the file is corrupted. These aren't edge cases — they're things that will happen the first time you run agentlog against real data.

This stage does two things: adds a proper error hierarchy so failures are explicit and informative, and adds a test suite so you know the code actually works before you wire it to a CLI and an LLM.

---

## Why this stage exists

Two things separate scripts from professional code: deliberate error handling and automated tests. Python's exception system lets you define hierarchies — catch a broad class or a specific one. pytest is the standard test runner — cleaner than unittest, with fixtures and parametrize built in. Every capstone in every ramp depends on pytest. You can't build reliable async agents without knowing how to test them.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Exception hierarchy:
    what: >
      Python exceptions are classes. Custom ones inherit from Exception.
      You can create a hierarchy: DevToolError → RunError → RunNotFoundError.
      Catching DevToolError catches all subclasses.
    why: >
      Custom exceptions let callers handle specific failures differently.
      raise RunNotFoundError("r1") is more informative than raise ValueError("not found").
      FastAPI's HTTPException is just a custom exception class — understanding this
      demystifies how error handlers work.

  raise, raise from, re-raise:
    what: >
      raise MyError("message")           — raise a new exception
      raise MyError("msg") from original — chain to the original cause
      raise                              — re-raise inside an except block
    why: >
      raise ... from ... preserves the original error in the traceback.
      This is how you wrap a low-level json.JSONDecodeError in a meaningful
      StorageCorruptedError without losing the root cause.

  try / except / else / finally:
    what: >
      try: risky code
      except SomeError as e: handle it
      else: runs only if no exception was raised
      finally: always runs — cleanup
    why: >
      else is the "happy path" that only runs when no exception occurred.
      finally is how you close resources when you can't use with.

  pytest basics:
    what: >
      Functions starting with test_ in files starting with test_ are collected and run.
      assert statement — pytest rewrites it to show exactly what failed.
      pytest.raises(ExceptionType) — assert that code raises a specific exception.
    why: >
      This is the testing pattern used in every OSS project you'll read or contribute to.
      The assert rewriting is why pytest's output is so clear — actual vs expected, not just "False".

  pytest fixtures:
    what: >
      A function decorated with @pytest.fixture that provides a resource to tests.
      Tests declare the fixture as a parameter — pytest injects it automatically.
    why: >
      Fixtures replace setUp/tearDown. They're more composable and explicit.
      A storage fixture creates a fresh FileStorage in a temp dir.
      Every test that needs storage gets a clean one without any setup code.

  pytest.mark.parametrize:
    what: >
      Run the same test with multiple inputs:
      @pytest.mark.parametrize("status", ["pending", "running", "done", "failed"])
    why: >
      One test function, four test cases. Better than copy-pasting the same assertion.
      Used in every serious test suite.

  tmp_path fixture:
    what: >
      A built-in pytest fixture — provides a temporary directory cleaned up after each test.
    why: >
      FileStorage tests must not write to real disk locations or interfere with each other.
      tmp_path gives each test its own isolated throwaway directory.
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
    task: "Write agentlog/errors.py"
    why: >
      RunRegistry.get() currently returns None for missing runs. FileStorage.load()
      crashes with a raw JSONDecodeError on corrupt data. Both need to communicate
      failures explicitly. Define the hierarchy now — one base class for everything
      agentlog raises, specific subclasses for specific situations.
    detail: >
      # agentlog/errors.py

      class DevToolError(Exception):
          """Base exception for all agentlog errors."""
          pass

      class RunError(DevToolError):
          """Errors related to a specific Run."""
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
    why: >
      The CLI (Stage 08) needs to catch RunNotFoundError and return a clean error
      message. The runner (Stage 06) needs to catch StorageCorruptedError and
      handle it without crashing. Neither of those works if the errors are generic.
      Wire the hierarchy in now.
    detail: >
      In models.py — update RunRegistry:

        from agentlog.errors import RunNotFoundError, RunAlreadyExistsError

        def get(self, run_id: str) -> Run:          # return type: Run, not Run | None
            if run_id not in self._runs:
                raise RunNotFoundError(run_id)
            return self._runs[run_id]

        def add(self, run: Run) -> None:
            if run.run_id in self._runs:
                raise RunAlreadyExistsError(run.run_id)
            self._runs[run.run_id] = run

      In storage.py — wrap json.loads:

        from agentlog.errors import StorageCorruptedError

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
    task: "Create tests/ and conftest.py"
    why: >
      conftest.py is pytest's shared fixture file — anything defined here is available
      to every test file automatically. The fixtures here represent agentlog's core
      objects in their most common test states.
    detail: >
      mkdir tests
      touch tests/__init__.py tests/conftest.py

      # tests/conftest.py

      import pytest
      from agentlog.models import Run, RunRegistry

      @pytest.fixture
      def sample_run() -> Run:
          return Run(run_id="test-001", prompt="test prompt")

      @pytest.fixture
      def populated_registry() -> RunRegistry:
          registry = RunRegistry()
          registry.add(Run(run_id="r1", prompt="first run"))
          registry.add(Run(run_id="r2", prompt="second run"))
          return registry

  4:
    task: "Write tests/test_models.py"
    why: >
      Run and RunRegistry are the core of agentlog. Every other layer (storage, runner,
      CLI) depends on them behaving correctly. Test the behavior you care about:
      status transitions, validation, registry operations, error cases.
    detail: >
      import pytest
      from pydantic import ValidationError
      from agentlog.models import Run, RunRegistry
      from agentlog.errors import RunNotFoundError, RunAlreadyExistsError

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
          assert r.prompt == "hello"

      def test_run_empty_prompt_raises():
          with pytest.raises(ValidationError):
              Run(run_id="r1", prompt="   ")

      def test_registry_get(populated_registry):
          run = populated_registry.get("r1")
          assert run.prompt == "first run"

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
      def test_valid_run_statuses(status):
          r = Run(run_id="r1", prompt="test", status=status)
          assert r.status == status

      def test_invalid_run_status_raises():
          with pytest.raises(ValidationError):
              Run(run_id="r1", prompt="test", status="blah")

  5:
    task: "Write tests/test_storage.py"
    why: >
      FileStorage is the only place agentlog touches the filesystem. Test it in
      isolation using tmp_path — each test gets a clean directory, nothing
      bleeds between tests, nothing touches your real .agentlog_data folder.
    detail: >
      import pytest
      from agentlog.models import Run, RunRegistry
      from agentlog.storage import FileStorage
      from agentlog.errors import StorageCorruptedError

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

      def test_load_empty_returns_empty_registry(storage):
          registry = storage.load()
          assert len(registry) == 0

      def test_corrupted_storage_raises(storage, tmp_path):
          (tmp_path / "data").mkdir(parents=True, exist_ok=True)
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

      def test_size_bytes_empty(storage):
          assert storage.size_bytes == 0

      def test_size_bytes_after_save(storage):
          registry = RunRegistry()
          registry.add(Run(run_id="r1", prompt="test"))
          storage.save(registry)
          assert storage.size_bytes > 0

  6:
    task: "Run pytest and read the output"
    why: >
      You've been running main.py manually to verify things work. From this stage on,
      the test suite does that verification — faster, reproducible, and it catches
      regressions when you change code in Stage 06 and 07.
    detail: >
      pytest tests/ -v

      The -v flag shows each test individually with pass/fail.
      Read every line. Understand the format.

      Run with coverage:
        pytest tests/ --cov=agentlog --cov-report=term-missing

      The coverage report shows which lines in your code are NOT covered.
      If coverage on models.py or storage.py is below 80%, add tests until it's not.

  7:
    task: "Write one deliberately failing test and read pytest's output"
    why: >
      You need to know what failure looks like — not just green passes.
      pytest's assertion rewriting shows actual vs expected values, not just "AssertionError".
      Read this output carefully. You'll see it constantly.
    detail: >
      def test_intentional_failure():
          r = Run(run_id="r1", prompt="test")
          assert r.status == "done"   # will fail — status is "pending"

      Run pytest. Read:
        AssertionError: assert 'pending' == 'done'

      This is pytest's assertion rewriting. Much more useful than a bare AssertionError.
      Delete the test after reading the output.
```

---

## Done When

```yaml
done_when:
  - pytest tests/ -v shows all tests passing
  - RunRegistry.get() raises RunNotFoundError — never returns None
  - FileStorage.load() raises StorageCorruptedError on invalid JSON
  - You have parametrized tests for all four valid run statuses
  - You understand @pytest.fixture and can write a new one from memory
  - Coverage on agentlog/models.py and agentlog/storage.py is above 80%
  - You can explain the exception hierarchy and add a new exception type in 30 seconds
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  custom_exceptions: "Same pattern — inherit from Exception, override constructor"
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
  - What does conftest.py do exactly — where does pytest look for it?
    What happens if you have a conftest.py in both tests/ and the project root?
  - How do you mock a function in pytest? (Hint: unittest.mock.patch or pytest-mock)
    You'll need this in Stage 08 to fake LLM calls in CLI tests.
  - What is the difference between scope="function" and scope="session" on a fixture?
    When would session-scoped make sense?
```
