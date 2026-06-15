# Stage 06 — Async Python

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — add agentlog/runner.py
**Depends on:** Stage 05 (errors and testing in place)

---

## The Story So Far

agentlog has validated models, persistent storage, and a test suite. The one thing it can't do yet is actually run prompts. That's what `runner.py` does — it takes a Run, sends the prompt to an LLM, and saves the result.

Here's the problem: LLM calls are slow. 1–3 seconds each. If a user queues 5 runs, running them sequentially takes 5–15 seconds. Async solves this — run all 5 concurrently, finish in the time it takes one. That's not an optimization, it's a requirement for a usable tool.

This stage builds the async runner and, in doing so, teaches you the mental model behind every async framework you'll use: FastAPI, LangGraph, the MCP SDK.

---

## Why this stage exists

Async is the most conceptually distinct thing in this ramp — it requires a mental model shift, not just new syntax. Once you have the model, async code is obvious. Without it, async looks like magic that sometimes works and sometimes blocks for no apparent reason. FastAPI and LangGraph are async from top to bottom. You need this stage to use them correctly.

---

## Concepts This Stage Teaches

```yaml
concepts:
  The event loop mental model:
    what: >
      Python runs one thread. The event loop manages tasks.
      When a task hits an await, it pauses and the event loop runs another task.
      When the awaited thing finishes, the task resumes.
      No parallelism — cooperative multitasking. Tasks take turns at await points.
    why: >
      This mental model explains every async behavior.
      A blocking function inside async def freezes everything because it never yields.
      asyncio.gather() runs things concurrently because it lets multiple tasks
      hit await at the same time.

  async def and await:
    what: >
      async def makes a function a coroutine — calling it returns a coroutine object,
      it does NOT run the function body.
      await runs the coroutine and waits for it to finish.
      You can only await inside an async def.
    why: >
      The most common confusion: calling an async function without await does nothing.
      You get a coroutine object sitting there, not a result.
      You'll make this mistake once — seeing the output is the lesson.

  asyncio.run():
    what: >
      asyncio.run(my_coroutine()) — starts the event loop, runs the coroutine, stops it.
      The entry point into async code from synchronous code.
    why: >
      FastAPI handles this for you in route handlers.
      In scripts and tests, you call it yourself.

  asyncio.gather():
    what: >
      await asyncio.gather(coro1(), coro2(), coro3())
      Runs all three concurrently. Returns results in input order.
    why: >
      Three LLM calls at 1s each: sequential = 3s, gather = ~1s.
      This is the entire value proposition of async for agentlog.

  asyncio.sleep() vs time.sleep():
    what: >
      asyncio.sleep(1) — yields control to the event loop while waiting
      time.sleep(1)    — blocks the entire thread
    why: >
      time.sleep() inside async def freezes the event loop for the full duration.
      No other tasks can run. The #1 async performance mistake.

  asyncio.create_task():
    what: >
      task = asyncio.create_task(coro())
      Creates a task that starts immediately on the next event loop tick.
    why: >
      gather() waits for all inputs together. create_task() fires and forgets —
      useful for background work. FastAPI BackgroundTasks use this pattern.

  async with and async for:
    what: >
      async with resource: — async context manager (httpx.AsyncClient)
      async for item in aiter: — iterate over an async generator
    why: >
      LangGraph's astream_events() returns an async generator — iterated with async for.
      You'll use both constantly in the agentic ramp.
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
    task: "See the coroutine-without-await mistake on purpose"
    why: >
      This is the mistake every async beginner makes. See it now in a controlled
      setting so you recognize it immediately in real code.
    detail: >
      # agentlog/runner.py — start here

      import asyncio

      async def greet_async(name: str) -> str:
          await asyncio.sleep(0.1)
          return f"Hello, {name}!"

      # DO THIS first — observe the output
      result = greet_async("world")    # no await
      print(result)                    # <coroutine object greet_async at 0x...>
      print(type(result))              # <class 'coroutine'>

      # Now correct it
      async def main():
          result = await greet_async("world")
          print(result)   # "Hello, world!"

      asyncio.run(main())

      The coroutine object is the function call doing nothing.
      await is what actually runs it.

  2:
    task: "Prove gather() is faster than sequential"
    why: >
      agentlog submits multiple runs concurrently. You need to see the timing
      difference with your own eyes. This is the moment async stops being abstract.
    detail: >
      import asyncio, time

      async def simulate_llm_call(prompt: str, delay: float) -> str:
          print(f"Starting: {prompt}")
          await asyncio.sleep(delay)
          print(f"Done: {prompt}")
          return f"Result for: {prompt}"

      async def run_sequential():
          start = time.time()
          r1 = await simulate_llm_call("prompt 1", 1.0)
          r2 = await simulate_llm_call("prompt 2", 1.0)
          r3 = await simulate_llm_call("prompt 3", 1.0)
          print(f"Sequential: {time.time() - start:.1f}s")   # ~3.0s

      async def run_concurrent():
          start = time.time()
          results = await asyncio.gather(
              simulate_llm_call("prompt 1", 1.0),
              simulate_llm_call("prompt 2", 1.0),
              simulate_llm_call("prompt 3", 1.0),
          )
          print(f"Concurrent: {time.time() - start:.1f}s")   # ~1.0s

      asyncio.run(run_sequential())
      asyncio.run(run_concurrent())

      Read the timing. The concurrent version takes ~1s, not ~3s.
      This is why agentlog uses async for its runner.

  3:
    task: "Build the async RunRunner"
    why: >
      This is the core of agentlog's execution layer. execute() runs one prompt —
      updates the run's status, simulates the LLM call, saves the result.
      execute_batch() runs many concurrently using gather().
      The LLM call is simulated with asyncio.sleep() for now —
      Stage 08 replaces it with a real API call.
    detail: >
      # agentlog/runner.py (full version)

      import asyncio
      from agentlog.models import Run, RunRegistry
      from agentlog.storage import FileStorage

      class RunRunner:
          def __init__(self, storage: FileStorage) -> None:
              self.storage = storage

          async def execute(self, run: Run) -> Run:
              """Execute a single run — simulates async LLM call."""
              run.status = "running"

              try:
                  # Real call in Stage 08: result = await llm_client.complete(run.prompt)
                  await asyncio.sleep(0.5)
                  run.mark_done(f"Processed: {run.prompt[:50]}")
              except Exception as e:
                  run.mark_failed(str(e))

              return run

          async def execute_batch(self, runs: list[Run]) -> list[Run]:
              """Execute multiple runs concurrently."""
              return await asyncio.gather(*[self.execute(run) for run in runs])

      Test in main.py:
        import asyncio
        from agentlog.models import Run
        from agentlog.runner import RunRunner
        from agentlog.storage import FileStorage

        async def main():
            storage = FileStorage()
            runner = RunRunner(storage)

            runs = [Run(run_id=f"r{i}", prompt=f"prompt {i}") for i in range(3)]
            results = await runner.execute_batch(runs)
            for r in results:
                print(r)

        asyncio.run(main())

  4:
    task: "Use async with for an HTTP client"
    why: >
      Stage 08 replaces simulate_llm_call with a real HTTP request to the OpenAI API.
      httpx.AsyncClient is how you make async HTTP calls in Python.
      The async with ensures the connection pool closes properly.
      Practice the pattern now so it's familiar when you need it.
    detail: >
      import httpx

      async def fetch_url(url: str) -> str:
          async with httpx.AsyncClient() as client:
              response = await client.get(url)
              return response.text

      text = asyncio.run(fetch_url("https://httpbin.org/get"))
      print(text[:200])

      The async with is the async equivalent of the with statement you used in Stage 04.
      It guarantees the HTTP connection pool closes even if an exception occurs.

  5:
    task: "Prove that blocking inside async kills concurrency"
    why: >
      Understanding what goes wrong is as important as understanding what goes right.
      This exercise teaches the rule you must never break: never call blocking
      functions inside async def.
    detail: >
      async def blocking_mistake():
          import time
          print("Running 3 tasks with time.sleep (WRONG)...")
          start = time.time()
          await asyncio.gather(
              _blocking_task("task 1"),
              _blocking_task("task 2"),
              _blocking_task("task 3"),
          )
          print(f"Took: {time.time() - start:.1f}s")   # ~3s — sequential!

      async def _blocking_task(name: str):
          import time
          time.sleep(1)   # blocks the entire event loop
          print(f"{name} done")

      asyncio.run(blocking_mistake())

      Time it: ~3s, not ~1s. All three tasks ran sequentially despite gather().
      Fix: replace time.sleep(1) with await asyncio.sleep(1). Time it again. ~1s.

      The rule: never use time.sleep, requests.get, or any blocking I/O inside async def.
      Always use the async equivalent.

  6:
    task: "Write async tests with pytest-asyncio"
    why: >
      RunRunner is async — you can't test it with plain pytest.
      pytest-asyncio lets you write async test functions.
      Add asyncio_mode = "auto" to pyproject.toml so you don't need
      @pytest.mark.asyncio on every test.
    detail: >
      Add to pyproject.toml:
        [tool.pytest.ini_options]
        asyncio_mode = "auto"

      Create tests/test_runner.py:

        import pytest, time
        from agentlog.models import Run
        from agentlog.runner import RunRunner
        from agentlog.storage import FileStorage

        @pytest.fixture
        def runner(tmp_path):
            storage = FileStorage(data_dir=tmp_path / "data")
            return RunRunner(storage)

        async def test_execute_single_run(runner):
            run = Run(run_id="r1", prompt="test")
            result = await runner.execute(run)
            assert result.status == "done"
            assert "Processed" in result.result

        async def test_execute_batch(runner):
            runs = [Run(run_id=f"r{i}", prompt=f"prompt {i}") for i in range(5)]
            results = await runner.execute_batch(runs)
            assert len(results) == 5
            assert all(r.status == "done" for r in results)

        async def test_batch_is_faster_than_sequential(runner):
            runs = [Run(run_id=f"r{i}", prompt=f"p{i}") for i in range(3)]
            start = time.time()
            await runner.execute_batch(runs)
            elapsed = time.time() - start
            # Each run sleeps 0.5s — concurrent should be ~0.5s not ~1.5s
            assert elapsed < 1.2

      pytest tests/ -v — all tests should pass.
```

---

## Done When

```yaml
done_when:
  - You can write an async function, call it with await, run it with asyncio.run()
  - You've seen the coroutine-object-without-await output and understand why it happens
  - execute_batch() on 3 runs completes in ~0.5s not ~1.5s — confirmed with timing
  - The blocking mistake exercise runs in ~3s; fixed version runs in ~1s
  - All runner tests pass with pytest-asyncio
  - You can use async with httpx.AsyncClient() to make a real HTTP call
  - You can explain why time.sleep() inside async def is wrong
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  async_def: "async Task<T> MethodAsync() — same semantics, different syntax"
  await: "await — identical keyword, identical purpose"
  asyncio.run(): "Task.Run().GetAwaiter().GetResult() at the top level"
  asyncio.gather(): "Task.WhenAll(task1, task2, task3) — runs all, returns all results"
  asyncio.create_task(): "Task.Run() — fires a task without waiting"
  async_with: "async using — identical concept"
  event_loop: >
    C# uses a thread pool under Task.Run — blocking one thread doesn't freeze others.
    Python asyncio uses a single thread — blocking anywhere freezes everything.
    This is the critical difference.
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between asyncio.gather() and asyncio.wait()?
    When would you use wait() instead?
  - What happens if one coroutine in gather() raises an exception?
    Does gather() cancel the others?
  - What is an asyncio.Queue and when would you use it instead of gather()?
  - What is asyncio.Semaphore and when would you need it?
    (Hint: rate-limiting LLM API calls — you'll hit this in the agentic ramp)
```
