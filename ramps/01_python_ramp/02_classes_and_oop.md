# Stage 02 — Classes and OOP

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — extend devtool/ from Stage 01
**Depends on:** Stage 01 (project structure, imports working)

Python OOP looks different from C#. No interfaces, no access modifiers enforced at runtime, no explicit getters/setters. But the concepts map directly — you just express them differently. This stage teaches Python classes the way they're actually used in frameworks like LangGraph, FastAPI, and Pydantic — not the textbook way.

---

## What You're Building

Add a `devtool/models.py` with a set of classes that model a simple "run tracker" — something that tracks job runs with status, timing, and results. This is a realistic domain you'll reuse in later stages.

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
    example: >
      from dataclasses import dataclass, field

      @dataclass
      class Run:
          id: str
          status: str = "pending"
          tags: list[str] = field(default_factory=list)

  Properties:
    what: >
      @property turns a method into a readable attribute.
      @x.setter adds a write side.
    csharp_equivalent: "public string Name { get; set; }"
    python: >
      @property
      def display_name(self) -> str:
          return self.name.upper()

  Dunder methods (magic methods):
    what: >
      Special methods with double underscores: __str__, __repr__, __eq__, __len__, __contains__
      Python calls these automatically in response to built-in operations.
    why: >
      __repr__ is what you see when you print an object in the REPL — essential for debugging.
      __eq__ controls == comparison — critical for test assertions.
      __str__ controls str(obj) — what you display to users.
      LangGraph's Message classes define these. Understanding them makes reading framework code easy.

  Inheritance:
    what: >
      class FailedRun(Run): — FailedRun gets all of Run's methods and attributes.
      super().__init__(...) calls the parent constructor.
    why: >
      Used for exception hierarchies (Stage 05), base classes in Pydantic and LangGraph.
      Not overused in modern Python — composition is usually preferred — but you need to read it.

  __slots__:
    what: >
      class Run: __slots__ = ["id", "status"] — restricts instance attributes to the named list.
      Saves memory, prevents accidental attribute creation.
    why: >
      You'll see this in performance-sensitive library code. Knowing it exists prevents confusion
      when you try to add an attribute to a slotted class and get an AttributeError.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write a plain class in models.py"
    detail: >
      # devtool/models.py

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

      Test in main.py:
        from devtool.models import Run
        r = Run("run-001", "hello world")
        print(r)            # uses __repr__
        r.result = "done!"  # uses setter, sets status to "done"
        print(r)

  2:
    task: "Convert to a dataclass"
    detail: >
      Rewrite Run as a dataclass. Notice what disappears:

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
      Print a Run instance — what does the repr look like?
      Compare two Run instances with the same run_id — are they equal?

  3:
    task: "Add a classmethod alternative constructor"
    detail: >
      Add to the Run dataclass:

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

      Note the "Run" string in the return type — forward reference. When a class
      references itself in its own type hint, use a string to avoid circular definition.

  4:
    task: "Build a RunRegistry class that holds multiple Runs"
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
        print(registry)             # uses __repr__

  5:
    task: "Add a subclass"
    detail: >
      @dataclass
      class StreamingRun(Run):
          chunk_count: int = 0

          def add_chunk(self, chunk: str) -> None:
              self.chunk_count += 1
              # append to result
              self.result = (self.result or "") + chunk

      Test:
        sr = StreamingRun("sr-001", "stream this")
        sr.add_chunk("Hello ")
        sr.add_chunk("world")
        print(sr.result)        # "Hello world"
        print(sr.chunk_count)   # 2
        print(isinstance(sr, Run))  # True — it IS a Run

  6:
    task: "Update main.py to use the registry"
    detail: >
      from devtool.models import Run, RunRegistry, StreamingRun

      registry = RunRegistry()
      registry.add(Run("r1", "first run"))
      registry.add(Run("r2", "second run"))
      registry.add(StreamingRun("r3", "streaming run"))

      registry.get("r1").mark_done("result one")
      registry.get("r3").add_chunk("partial ")
      registry.get("r3").add_chunk("result")

      for run in registry.all():
          print(run)
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Create two Run instances with the same run_id. Are they equal (==)?
    Why — which method controls this?
  - Forget self on a method definition. What error do you get?
  - Try to add an attribute that isn't declared in a dataclass after creation.
    Does it work? (It does — dataclasses don't enforce slots by default.)
    Now add __slots__ = () to the dataclass and try again.
  - Call super().__init__() in StreamingRun without passing the required fields.
    What error? Fix it by passing them correctly.
```

---

## Done When

```yaml
done_when:
  - Run, RunRegistry, and StreamingRun all work as described
  - __repr__, __eq__, __len__, __contains__ all behave correctly
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
  - What is the MRO (Method Resolution Order) and why does it matter for inheritance?
```
