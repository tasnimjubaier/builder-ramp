# Stage 02 — Classes and OOP

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — extend agentlog/ from Stage 01
**Depends on:** Stage 01 (project structure, imports working)

---

## The Story So Far

You have a project skeleton. Now agentlog needs to actually represent something — the core entity it tracks: an **agent run**.

A run has an ID, a prompt, a status, and eventually a result. It can be pending, running, done, or failed. You need to create one, update it, store multiple of them, and eventually print them cleanly. That's a data structure with behavior — a class.

By the end of this stage, `agentlog/models.py` has three classes that work together: `Run`, `RunRegistry`, and `StreamingRun`. Every concept in this stage comes from a real need in the project, not from a concept list.

---

## Why this stage exists

Python OOP looks different from C#. No interfaces, no access modifiers enforced at runtime, no explicit getters/setters. But the concepts map directly — you just express them differently. This stage teaches Python classes the way they're actually used in frameworks like LangGraph, FastAPI, and Pydantic — not the textbook way.

---

## Concepts This Stage Teaches

```yaml
concepts:
  __init__ and self:
    what: >
      __init__ is the constructor. self is the instance — it's always the first parameter
      of any instance method, passed automatically by Python.
    csharp_equivalent: "public MyClass(string name) { this.name = name; }"
    python: >
      class Run:
          def __init__(self, name: str):
              self.name = name

  Instance vs class vs static methods:
    what: >
      Instance method: def method(self) — operates on the instance
      Class method:    @classmethod def method(cls) — operates on the class itself
      Static method:   @staticmethod def method() — no access to instance or class
    why: >
      @classmethod is used for alternative constructors — Run.from_dict(data).
      @staticmethod is used for utility functions that logically belong to the class
      but don't need self. You'll see both patterns constantly in LangGraph internals.

  dataclasses:
    what: >
      @dataclass auto-generates __init__, __repr__, and __eq__ from field annotations.
      No need to write self.x = x for every field.
    why: >
      Most data-holding classes in Python use @dataclass or Pydantic (Stage 03).
      Writing __init__ by hand is verbose and error-prone for simple data containers.

  Properties:
    what: >
      @property turns a method into a readable attribute.
      @x.setter adds a write side.
    csharp_equivalent: "public string Name { get; set; }"
    why: >
      A run's result and status are linked — setting a result should also update status.
      A property setter enforces that coupling without exposing the internals.

  Dunder methods (magic methods):
    what: >
      Special methods with double underscores: __str__, __repr__, __eq__, __len__, __contains__
      Python calls these automatically in response to built-in operations.
    why: >
      __repr__ is what you see when you print a Run in the REPL — essential for debugging.
      __eq__ controls == — critical for test assertions.
      __len__ and __contains__ on RunRegistry make it feel like a native collection.
      LangGraph's Message classes define all of these.

  Inheritance:
    what: >
      class StreamingRun(Run): — StreamingRun gets all of Run's methods and attributes.
      super().__init__(...) calls the parent constructor.
    why: >
      A streaming run IS a run — same ID, prompt, status — but it also accumulates
      chunks as they arrive. Inheritance models that relationship directly.
      You'll see this in Pydantic and LangGraph base classes.

  __slots__:
    what: >
      class Run: __slots__ = ["id", "status"] — restricts instance attributes to the named list.
    why: >
      You'll see this in performance-sensitive library code. Knowing it exists prevents
      confusion when you try to add an attribute to a slotted class and get AttributeError.
```

---

## Build Steps

Each step builds on the previous. You're not learning features in isolation — you're building `models.py` piece by piece until it does everything agentlog needs.

```yaml
steps:
  1:
    task: "Write a plain Run class"
    feature: "agentlog — models.py: Run class with __init__, __repr__, and __eq__"
    why: >
      agentlog tracks runs. A run has an ID, a prompt, a status, and eventually a result.
      Start with the simplest version — a plain class with an __init__ and a __repr__
      so you can print it cleanly while developing.
    detail: >
      # agentlog/models.py

      class Run:
          def __init__(self, run_id: str, prompt: str):
              self.run_id = run_id
              self.prompt = prompt
              self.status = "pending"
              self._result: str | None = None   # _ prefix = convention for "private"

          def __repr__(self) -> str:
              return f"Run(id={self.run_id!r}, status={self.status!r})"

          def __eq__(self, other: object) -> bool:
              if not isinstance(other, Run):
                  return NotImplemented
              return self.run_id == other.run_id

      Test in main.py:
        from agentlog.models import Run
        r = Run("run-001", "hello world")
        print(r)             # Run(id='run-001', status='pending')
        r2 = Run("run-001", "different prompt")
        print(r == r2)       # True — same run_id, __eq__ controls this

  2:
    task: "Add a property to link result and status"
    feature: "agentlog — models.py: Run.result property setter that auto-updates status"
    why: >
      When a run finishes, two things change together: the result gets set and the
      status flips to "done". If those are separate assignments, callers can forget
      one. A property setter enforces the coupling — set the result, status updates
      automatically.
    detail: >
      Add to Run:

          @property
          def result(self) -> str | None:
              return self._result

          @result.setter
          def result(self, value: str) -> None:
              self._result = value
              self.status = "done"

          def mark_failed(self, error: str) -> None:
              self.status = "failed"
              self._result = f"ERROR: {error}"

      Test:
        r = Run("run-001", "hello world")
        r.result = "done!"   # setter fires — status becomes "done"
        print(r)             # Run(id='run-001', status='done')

  3:
    task: "Convert Run to a dataclass"
    feature: "agentlog — models.py: Run as @dataclass with auto-generated init, repr, eq"
    why: >
      Run is a data-holding class. Writing self.x = x for every field is noise.
      @dataclass generates __init__, __repr__, and __eq__ automatically.
      Notice what disappears from your code — and what you get for free.
      This is the pattern agentlog will use from Stage 03 onwards with Pydantic.
    detail: >
      from dataclasses import dataclass, field
      from datetime import datetime

      @dataclass
      class Run:
          run_id: str
          prompt: str
          status: str = "pending"
          result: str | None = None
          created_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())

          def mark_done(self, result: str) -> None:
              self.result = result
              self.status = "done"

          def mark_failed(self, error: str) -> None:
              self.result = f"ERROR: {error}"
              self.status = "failed"

      Note: dataclass generates __repr__ and __eq__ automatically.
      The hand-written __repr__ and __eq__ from step 1 are now gone — replaced for free.

      Test:
        r1 = Run("r1", "prompt one")
        r2 = Run("r1", "prompt two")
        print(r1 == r2)   # True — dataclass __eq__ compares ALL fields by default
        # Compare with your hand-written __eq__ — which behavior do you want?
        # This is a real design decision you'll make in Stage 03 with Pydantic.

  4:
    task: "Add a classmethod alternative constructor"
    feature: "agentlog — models.py: Run.from_dict() classmethod for loading from raw data"
    why: >
      agentlog will eventually load runs from a JSON file. The data comes in as a dict,
      not as keyword arguments. Run.from_dict(data) is the clean interface for that —
      one call, handles all the field mapping. This is the alternative constructor pattern.
    detail: >
      Add to Run:

          @classmethod
          def from_dict(cls, data: dict) -> "Run":
              return cls(
                  run_id=data["run_id"],
                  prompt=data["prompt"],
                  status=data.get("status", "pending"),
                  result=data.get("result"),
              )

      Test:
        r = Run.from_dict({"run_id": "abc", "prompt": "test"})
        print(r)

      The "Run" string in the return type is a forward reference — a class can't
      reference itself in its own type hint until Python 3.12 with from __future__ import annotations.
      Using a string avoids the circular definition. You'll see this in framework code constantly.

  5:
    task: "Build RunRegistry to hold multiple runs"
    feature: "agentlog — models.py: RunRegistry with O(1) lookup, filtering, and collection dunders"
    why: >
      agentlog needs to track more than one run at a time — list them, filter by status,
      look one up by ID. A plain list won't do: you want O(1) lookup by ID and clean
      iteration. RunRegistry wraps a dict and exposes exactly the interface agentlog needs.
      The dunder methods make it feel like a native Python collection.
    detail: >
      class RunRegistry:
          def __init__(self) -> None:
              self._runs: dict[str, Run] = {}

          def add(self, run: Run) -> None:
              self._runs[run.run_id] = run

          def get(self, run_id: str) -> Run | None:
              return self._runs.get(run_id)

          def all(self) -> list[Run]:
              return list(self._runs.values())

          def pending(self) -> list[Run]:
              return [r for r in self._runs.values() if r.status == "pending"]

          def __len__(self) -> int:
              return len(self._runs)

          def __contains__(self, run_id: str) -> bool:
              return run_id in self._runs

          def __repr__(self) -> str:
              return f"RunRegistry({len(self)} runs)"

      Test:
        registry = RunRegistry()
        registry.add(Run("r1", "prompt one"))
        registry.add(Run("r2", "prompt two"))
        print(len(registry))        # 2 — uses __len__
        print("r1" in registry)     # True — uses __contains__
        print(registry)             # RunRegistry(2 runs) — uses __repr__
        print(registry.pending())   # both runs, status is "pending"

  6:
    task: "Add StreamingRun as a subclass"
    feature: "agentlog — models.py: StreamingRun subclass for chunk-by-chunk LLM streaming"
    why: >
      Some LLM APIs stream responses — tokens arrive one at a time. A StreamingRun
      IS a Run (same ID, prompt, status, registry behavior) but it also accumulates
      chunks. Inheritance models this: StreamingRun gets everything Run has, and adds
      chunk-handling on top. You don't duplicate Run's logic — you extend it.
    detail: >
      @dataclass
      class StreamingRun(Run):
          chunk_count: int = 0

          def add_chunk(self, chunk: str) -> None:
              self.chunk_count += 1
              self.result = (self.result or "") + chunk
              self.status = "streaming"

          def finalize(self) -> None:
              self.status = "done"

      Test:
        sr = StreamingRun("sr-001", "stream this")
        sr.add_chunk("Hello ")
        sr.add_chunk("world")
        sr.finalize()
        print(sr.result)             # "Hello world"
        print(sr.chunk_count)        # 2
        print(isinstance(sr, Run))   # True — it IS a Run

        # Add it to the registry — it works because it's a Run
        registry.add(sr)
        print(len(registry))         # 3

  7:
    task: "Wire it all together in main.py"
    feature: "agentlog — models.py: integration smoke test — all three model classes working as a system"
    why: >
      Confirm the full model layer works as a system before moving to Stage 03.
      This is the state of agentlog's data layer at the end of this stage.
    detail: >
      from agentlog.models import Run, RunRegistry, StreamingRun

      registry = RunRegistry()
      registry.add(Run("r1", "first run"))
      registry.add(Run("r2", "second run"))
      registry.add(StreamingRun("r3", "streaming run"))

      registry.get("r1").mark_done("result one")
      registry.get("r2").mark_failed("timeout")

      sr = registry.get("r3")
      sr.add_chunk("partial ")
      sr.add_chunk("result")
      sr.finalize()

      for run in registry.all():
          print(run)
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Create two Run instances with the same run_id. Are they equal (==)?
    Why — which method controls this? How does the dataclass __eq__ differ
    from the hand-written one you wrote in step 1?
  - Forget self on a method definition. What error do you get?
  - Try adding a new attribute to a Run after creation — does it work?
    Now add __slots__ = () to the dataclass and try again. What happens?
  - Call super().__init__() in StreamingRun without passing the required fields.
    What error? Fix it by passing them correctly.
  - Add a StreamingRun to the registry, then call registry.pending().
    Does it show up? Why or why not? Fix mark_done to use finalize() instead.
```

---

## Done When

```yaml
done_when:
  - Run, RunRegistry, and StreamingRun all work as described
  - __repr__, __eq__, __len__, __contains__ all behave correctly
  - registry.add(StreamingRun(...)) works because StreamingRun is a Run
  - You can add a new method to any class without looking up syntax
  - You can explain @property and why it exists vs a plain attribute
  - You understand @classmethod vs @staticmethod vs instance method
  - You can read a class definition from LangGraph source and understand every part
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  __init__: "Constructor — but always named __init__, never the class name"
  self: "this — but explicit, always the first parameter"
  @property: "C# property with get/set — but declared differently"
  __repr__: "ToString() override"
  __eq__: "Equals() override — but also affects =="
  @dataclass: "C# record type — auto-generates constructor, equality, repr"
  @classmethod: "Static factory method that receives the class as first arg"
  _private: "Convention only — no enforcement at runtime. __double_underscore name-mangles."
  inheritance: "Same concept, same rules — single inheritance by default"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between __repr__ and __str__? When does Python call each?
  - What is multiple inheritance in Python? When would you use it?
    (Hint: mixins — you'll see this in LangGraph's BaseCallbackHandler)
  - What does dataclass(frozen=True) do? When would you use it?
  - What is the MRO (Method Resolution Order) and why does it matter?
  - The dataclass __eq__ compares all fields. Your hand-written __eq__ compared only run_id.
    Which is correct for agentlog? What are the tradeoffs? You'll make this decision
    explicitly when you switch to Pydantic in Stage 03.
```
