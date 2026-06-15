# Stage 06 — Async Python

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — add async capabilities to devtool/
**Depends on:** Stage 05 (you can run pytest, models work)

Async is the most conceptually distinct thing in this ramp — it requires a mental model shift, not just new syntax. Once you have the model, every async framework (FastAPI, LangGraph, MCP) becomes obvious. Without it, async code looks like magic that sometimes works and sometimes blocks for no apparent reason.

---

## What You're Building

Add `devtool/runner.py` — an async job runner that executes runs concurrently. Add async tests using `pytest-asyncio`. By the end you'll understand the event loop, coroutines, and concurrent execution — the foundations of everything in the other ramps.

---

## Concepts This Stage Teaches

```yaml
concepts:
  The event loop mental model:
    what: >
      Python runs one thread. The event loop is a manager that runs tasks.
      When a task hits an await, it pauses and the event loop runs another task.
      When the awaited thing (network call, sleep, file read) finishes, the task resumes.
      No parallelism — just cooperative multitasking. Tasks take turns.
    why: >
      This mental model explains every async behavior. Why does calling a sync
      blocking function inside async def freeze everything? Because it doesn't await —
      it holds the event loop hostage. Why does asyncio.gather() run things concurrently?
      Because it lets multiple tasks await at the same time.

  async def and await:
    what: >
      async def makes a function a coroutine — calling it returns a coroutine object,
      it does NOT run the function.
      await runs the coroutine and waits for it to finish.
      You can only await inside an async def.
    why: >
      This is the most common confusion. greet() where greet is async def doesn't run it.
      await greet() does. Forgetting await gives you a coroutine object, not a result.

  asyncio.run():
    what: >
      The entry point into async code from synchronous code.
      asyncio.run(my_coroutine()) — starts the event loop, runs the coroutine, stops the loop.
    why: >
      You can't await from a regular function. asyncio.run() is the bridge.
      FastAPI handles this for you — you never call asyncio.run() in a FastAPI handler.
      But in scripts and tests you call it.

  asyncio.gather():
    what: >
      await asyncio.gather(coro1(), coro2(), coro3())
      Runs all three concurrently — each one can make progress while others await.
      Returns a list of results in the same order as the inputs.
    why: >
      This is where async pays off. Three LLM calls that each take 2 seconds take 2 seconds
      total with gather(), not 6 seconds sequentially. You'll use this in agent pipelines.

  asyncio.sleep() vs time.sleep():
    what: >
      asyncio.sleep(1) — async sleep, yields control to the event loop while waiting
      time.sleep(1)    — sync sleep, blocks the entire thread for 1 second
    why: >
      time.sleep() inside async def freezes the event loop for the full duration —
      no other tasks can run. This is the #1 async performance mistake.
      asyncio.sleep(0) is a common pattern to explicitly yield control with no delay.

  asyncio.create_task():
    what: >
      task = asyncio.create_task(coro())
      Creates a task that starts running immediately (on the next event loop tick).
      Returns a Task object you can await later.
    why: >
      gather() waits for all coroutines together. create_task() fires and forgets —
      useful when you want something to run in the background without blocking the current flow.
      FastAPI's BackgroundTasks use this pattern internally.

  async with and async for:
    what: >
      async with resource: — async context manager (for async-capable resources)
      async for item in aiter: — iterate over an async generator
    why: >
      HTTP clients (httpx.AsyncClient) use async with.
      LangGraph's astream_events() returns an async generator — you iterate with async for.
      You'll use both constantly.
```

---

## Setup

```bash
pip install pytest-asyncio httpx
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write your first coroutine and run it"
    detail: >
      # devtool/runner.py

      import asyncio

      async def greet_async(name: str) -> str:
          await asyncio.sleep(0.1)   # simulate async work
          return f"Hello, {name}!"

      async def main():
          result = await greet_async("world")
          print(result)

      if __name__ == "__main__":
          asyncio.run(main())

      Run: python devtool/runner.py

      Now add this — DON'T await — and observe:
        result = greet_async("world")   # no await
        print(result)                   # <coroutine object greet_async ...>
        print(type(result))             # <class 'coroutine'>

      This is the most important thing to see early: calling an async function
      without await gives you a coroutine object, not the result.

  2:
    task: "Run tasks concurrently with gather()"
    detail: >
      async def simulate_llm_call(prompt: str, delay: float) -> str:
          print(f"Starting: {prompt}")
          await asyncio.sleep(delay)   # simulate network latency
          print(f"Done: {prompt}")
          return f"Result for: {prompt}"

      async def run_sequential():
          import time
          start = time.time()
          r1 = await simulate_llm_call("prompt 1", 1.0)
          r2 = await simulate_llm_call("prompt 2", 1.0)
          r3 = await simulate_llm_call("prompt 3", 1.0)
          print(f"Sequential: {time.time() - start:.1f}s")   # ~3.0s
          return [r1, r2, r3]

      async def run_concurrent():
          import time
          start = time.time()
          results = await asyncio.gather(
              simulate_llm_call("prompt 1", 1.0),
              simulate_llm_call("prompt 2", 1.0),
              simulate_llm_call("prompt 3", 1.0),
          )
          print(f"Concurrent: {time.time() - start:.1f}s")   # ~1.0s
          return results

      asyncio.run(run_sequential())
      asyncio.run(run_concurrent())

      Read the timing output. The concurrent version takes ~1s not ~3s.
      This is the entire value proposition of async — but ONLY for I/O-bound work.

  3:
    task: "Build an async RunRunner"
    detail: >
      import asyncio
      from devtool.models import Run, RunRegistry
      from devtool.storage import FileStorage

      class RunRunner:
          def __init__(self, storage: FileStorage) -> None:
              self.storage = storage
              self._registry = RunRegistry()

          async def execute(self, run: Run) -> Run:
              """Execute a single run — simulates async LLM call."""
              self._registry.add(run)
              run.status = "running"

              try:
                  # Simulate an async operation (real: await llm_client.complete(...))
                  await asyncio.sleep(0.5)
                  run.mark_done(f"Processed: {run.prompt[:30]}")
              except Exception as e:
                  run.mark_failed(str(e))

              self.storage.save(self._registry)
              return run

          async def execute_batch(self, runs: list[Run]) -> list[Run]:
              """Execute multiple runs concurrently."""
              return await asyncio.gather(*[self.execute(run) for run in runs])

  4:
    task: "Use async with for a resource"
    detail: >
      import httpx

      async def fetch_url(url: str) -> str:
          async with httpx.AsyncClient() as client:
              response = await client.get(url)
              return response.text

      # Test
      text = asyncio.run(fetch_url("https://httpbin.org/get"))
      print(text[:200])

      The async with ensures the HTTP connection pool is closed properly.
      This is the httpx pattern you'll use in the RAG and agentic ramps.

  5:
    task: "Write async tests with pytest-asyncio"
    detail: >
      Create tests/test_runner.py:

      import pytest
      import asyncio
      from devtool.models import Run
      from devtool.runner import RunRunner
      from devtool.storage import FileStorage

      @pytest.fixture
      def runner(tmp_path):
          storage = FileStorage(data_dir=tmp_path / "data")
          return RunRunner(storage)

      @pytest.mark.asyncio
      async def test_execute_single_run(runner):
          run = Run(run_id="r1", prompt="test")
          result = await runner.execute(run)
          assert result.status == "done"
          assert "Processed" in result.result

      @pytest.mark.asyncio
      async def test_execute_batch(runner):
          runs = [Run(run_id=f"r{i}", prompt=f"prompt {i}") for i in range(5)]
          results = await runner.execute_batch(runs)
          assert len(results) == 5
          assert all(r.status == "done" for r in results)

      @pytest.mark.asyncio
      async def test_batch_is_faster_than_sequential(runner):
          import time
          runs = [Run(run_id=f"r{i}", prompt=f"prompt {i}") for i in range(3)]

          start = time.time()
          await runner.execute_batch(runs)
          concurrent_time = time.time() - start

          # Each run sleeps 0.5s — concurrent should be ~0.5s, sequential ~1.5s
          assert concurrent_time < 1.2   # well under 3x sequential

      Add to pytest.ini or pyproject.toml:
        [tool.pytest.ini_options]
        asyncio_mode = "auto"
      
      Or add @pytest.mark.asyncio to each async test.

  6:
    task: "Understand what blocking in async means"
    detail: >
      Add this to runner.py and test it:

      async def blocking_mistake():
          import time
          print("Starting 3 tasks...")
          await asyncio.gather(
              _blocking_task("task 1"),
              _blocking_task("task 2"),
              _blocking_task("task 3"),
          )

      async def _blocking_task(name: str):
          import time
          print(f"{name}: start")
          time.sleep(1)   # WRONG — sync sleep blocks entire event loop
          print(f"{name}: done")

      asyncio.run(blocking_mistake())
      Time it. It takes ~3s, not ~1s. All three tasks run sequentially despite gather().

      Fix: replace time.sleep(1) with await asyncio.sleep(1).
      Time it again. Now ~1s.

      This exercise teaches the rule: never call blocking functions inside async def.
      time.sleep, requests.get, open() with a slow disk — all block the event loop.
```

---

## Done When

```yaml
done_when:
  - You can write an async function, call it with await, and run it with asyncio.run()
  - You can explain why calling async def without await gives a coroutine object
  - asyncio.gather() with 3 tasks runs in ~1x time, not 3x — confirmed with timing
  - RunRunner.execute_batch() works and tests pass
  - You understand why time.sleep() inside async def is wrong
  - pytest async tests run with @pytest.mark.asyncio
  - You can use async with httpx.AsyncClient() to make a real HTTP call
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  async_def: "async Task<T> MethodAsync() — same semantics, different syntax"
  await: "await — identical keyword, identical purpose"
  asyncio.run(): "Task.Run(() => ...).GetAwaiter().GetResult() at the top level"
  asyncio.gather(): "Task.WhenAll(task1, task2, task3) — runs all, returns all results"
  asyncio.create_task(): "Task.Run() — fires a task without waiting"
  async_with: "async using — identical concept"
  event_loop: >
    C# uses a thread pool under Task.Run. Python's asyncio uses a single thread.
    The key difference: blocking in C# only blocks one thread from the pool.
    Blocking in Python asyncio blocks the ENTIRE program.
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between asyncio.gather() and asyncio.wait()?
    When would you use wait() instead?
  - What happens if one of the coroutines in asyncio.gather() raises an exception?
    Does gather() cancel the other coroutines?
  - What is an asyncio.Queue and when would you use it instead of gather()?
  - What is the difference between asyncio.run() and loop.run_until_complete()?
    Why is asyncio.run() preferred in modern Python?
```
