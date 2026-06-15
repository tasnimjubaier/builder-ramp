# Stage 09 — Resilience: Retries, Fallback, and Cost Budgets

**Status:** not started
**Estimated time:** 3 days
**Depends on:** Stage 08 (observability in place — you can see retry behavior in traces)

Production agents fail. LLM APIs rate-limit, tools throw exceptions, subagents time out, cost explodes. This stage builds the fault tolerance layer that AgentResilience is based on — retry with backoff, fallback routing, and cost-bounded execution. After this stage you will have written the core primitives that AgentResilience ships as a library.

---

## What You're Building

A decorator library (your own, hand-rolled) with three primitives:
1. `@retry_with_backoff` — retry a function on failure with exponential backoff, abort when a cost budget is hit
2. `@fallback_to` — if the primary function fails, call a backup function instead
3. A `CostLedger` — a context-local accumulator that tracks cumulative token spend across a retry chain

Apply all three to nodes in your Stage 06 agent. Run the agent with failures injected. Watch the retry/fallback behavior in Jaeger traces from Stage 08.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Exponential backoff:
    what: >
      Wait 2^attempt seconds between retries. First retry after 1s, second after 2s,
      third after 4s, etc. Cap the wait at a configurable maximum.
    why: >
      Naive retries hammer a rate-limited API and make things worse. Backoff gives
      the provider time to recover. Every production system that calls external APIs uses this.

  Cost-bounded retry:
    what: >
      Before each retry, check a running cost total. If the next attempt would exceed
      a cost budget (e.g., $0.50), abort the retry chain instead of making the call.
    why: >
      This is the IP of AgentResilience's retry primitive. A simple max_attempts guard
      doesn't prevent runaway cost — an agent retrying a complex task 3 times could
      spend 3x the expected budget. Cost bounding is the pattern that separates
      someone who's run agents in production from someone who hasn't.

  Fallback routing:
    what: >
      If function A fails (after all retries), call function B with the same arguments
      and return its result. B is typically cheaper, faster, or more reliable.
    why: >
      Common pattern: fall back from GPT-4o to GPT-4o-mini on failure. Or fall back
      from a web search tool to a cached result. Knowing how to implement this as
      a generic decorator means you can apply it anywhere.

  Circuit breaker:
    what: >
      Track consecutive failures for a given function. After a threshold (e.g., 5),
      stop trying immediately for a cooldown period. After cooldown, try once — if it
      succeeds, reset the counter.
    why: >
      Retries help with transient failures. Circuit breakers prevent cascading failures
      when a dependency is down for longer. They protect the rest of the system from
      spending time on a call that's known to be failing.

  Failure injection:
    what: >
      Deliberately making a function fail in tests — using unittest.mock to replace
      a function with one that raises an exception on the first N calls.
    why: >
      You can't test resilience primitives without controlling when failures happen.
      Failure injection is the testing technique. You built the tool mocking skills
      for this in Stage 02.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Build CostLedger"
    detail: >
      import contextvars

      _cost_ledger: contextvars.ContextVar[float] = contextvars.ContextVar("cost_ledger", default=0.0)

      class CostLedger:
          @staticmethod
          def add(cost_usd: float):
              current = _cost_ledger.get()
              _cost_ledger.set(current + cost_usd)

          @staticmethod
          def get() -> float:
              return _cost_ledger.get()

          @staticmethod
          def reset():
              _cost_ledger.set(0.0)

      Use ContextVar so each async task has its own ledger — they don't interfere.

  2:
    task: "Build retry_with_backoff decorator"
    detail: >
      import asyncio
      import functools
      from typing import Callable, Type

      def retry_with_backoff(
          max_attempts: int = 3,
          backoff_base: float = 1.0,
          backoff_cap: float = 30.0,
          cost_budget: float = None,
          exceptions: tuple[Type[Exception], ...] = (Exception,)
      ):
          def decorator(func: Callable):
              @functools.wraps(func)
              async def wrapper(*args, **kwargs):
                  for attempt in range(1, max_attempts + 1):
                      if cost_budget and CostLedger.get() >= cost_budget:
                          raise CostBudgetExceeded(
                              f"Cost budget ${cost_budget} reached after ${CostLedger.get():.4f} spent"
                          )
                      try:
                          return await func(*args, **kwargs)
                      except exceptions as e:
                          if attempt == max_attempts:
                              raise
                          wait = min(backoff_base * (2 ** (attempt - 1)), backoff_cap)
                          await asyncio.sleep(wait)
              return wrapper
          return decorator

      class CostBudgetExceeded(Exception):
          pass

  3:
    task: "Build fallback_to decorator"
    detail: >
      def fallback_to(backup_func: Callable):
          def decorator(func: Callable):
              @functools.wraps(func)
              async def wrapper(*args, **kwargs):
                  try:
                      return await func(*args, **kwargs)
                  except Exception as primary_error:
                      print(f"Primary failed ({primary_error}), trying fallback")
                      return await backup_func(*args, **kwargs)
              return wrapper
          return decorator

  4:
    task: "Build a simple CircuitBreaker"
    detail: >
      class CircuitBreaker:
          def __init__(self, threshold: int = 5, cooldown: float = 60.0):
              self.threshold = threshold
              self.cooldown = cooldown
              self._failures = 0
              self._opened_at: float | None = None

          def call(self, func):
              @functools.wraps(func)
              async def wrapper(*args, **kwargs):
                  import time
                  if self._opened_at:
                      if time.time() - self._opened_at < self.cooldown:
                          raise RuntimeError("Circuit is open — call blocked")
                      # half-open: try once
                  try:
                      result = await func(*args, **kwargs)
                      self._failures = 0    # success resets
                      self._opened_at = None
                      return result
                  except Exception:
                      self._failures += 1
                      if self._failures >= self.threshold:
                          self._opened_at = time.time()
                      raise
              return wrapper

  5:
    task: "Apply to a node in your Stage 06 agent"
    detail: >
      @retry_with_backoff(max_attempts=3, cost_budget=0.10)
      @fallback_to(cheap_researcher_node)
      async def researcher_node(state):
          ...

      Define cheap_researcher_node as a mock that always succeeds (cheaper/simpler fallback).

  6:
    task: "Inject failures and observe"
    detail: >
      Use unittest.mock.patch to make researcher_node's internal tool fail on the
      first 2 calls and succeed on the 3rd.

      Run the agent. Observe:
        - Did it retry 2 times?
        - Did the final result come from the 3rd attempt?
        - Does the cost ledger show the cost of all 3 attempts?

      Then make it fail all 3 times. Does it route to the fallback?

  7:
    task: "Emit OTel spans for retry events"
    detail: >
      Add span emission to retry_with_backoff:
        - On each retry: emit a span "retry.attempt" with attempt number and exception type
        - On budget exceeded: emit a span "retry.budget_exceeded" with cumulative cost

      These should appear in Jaeger as children of the node span — exactly as
      AgentResilience describes in its technical deep dive.
```

---

## Done When

```yaml
done_when:
  - retry_with_backoff retries on failure and backs off correctly
  - cost_budget enforcement aborts the retry chain before hitting the limit
  - fallback_to routes to the backup function when primary exhausts retries
  - CircuitBreaker opens after threshold failures and resets after cooldown
  - Retry and fallback spans appear in Jaeger as children of the node span
  - You can read the AgentResilience deep dive and point to your code
    for every concept it describes (CostLedger, fallback, circuit breaker state machine)
```

---

## What This Unlocks

Stage 10 (capstone) combines all layers. You've now built every primitive — callback system, trajectory capture, OTel spans, retry/fallback — by hand. The capstone turns that into a minimal version of AgentEval with a real pytest CI gate.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between retry on any Exception vs retry only on specific
    exception types (e.g., RateLimitError)? Why does specificity matter?
  - How do you test that the circuit breaker resets correctly after cooldown?
    (Hint: you need to mock time.time())
  - Can you apply retry_with_backoff and circuit_breaker to the same function?
    What order should they be applied in?
  - What happens to the CostLedger if the agent runs in multiple async tasks
    simultaneously? Is contextvars the right solution?
```
