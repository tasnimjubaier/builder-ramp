# Stage 03 — Type Hints and Pydantic

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — rewrite agentlog/models.py
**Depends on:** Stage 02 (classes, dataclasses, RunRegistry working)

---

## The Story So Far

You have `Run`, `RunRegistry`, and `StreamingRun` as dataclasses. They work, but they have no validation. You can create a Run with `status="blah"` and nobody complains. You can pass a dict with missing fields to `from_dict()` and get a KeyError at runtime. When agentlog eventually talks to an LLM API and parses its response, unvalidated data will cause subtle bugs that are hard to trace.

This stage rewrites `models.py` to use Pydantic. Pydantic takes type hints seriously: it validates data at runtime, coerces compatible types, and tells you exactly what's wrong when something doesn't fit. FastAPI, LangGraph, and every serious AI library is built on it. After this stage, Run is what you'd actually use in a production project.

---

## Why this stage exists

Python's type hints don't enforce anything by default — they're documentation that your editor and tools read. Pydantic takes those same hints and enforces them at runtime. The payoff: when an LLM returns malformed output, you get a clear ValidationError pointing to the exact field, not a mysterious AttributeError three function calls later.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Type hint syntax:
    what: >
      str, int, float, bool          — primitives
      list[str], dict[str, int]      — generics
      str | None                     — union (Python 3.10+)
      Optional[str]                  — same as str | None (older style)
      tuple[str, int]                — fixed-length tuple
      Any                            — opt out of type checking
      Callable[[str], int]           — a function taking str, returning int
    why: >
      Every library function you'll call has type hints. Reading them tells you
      what to pass and what you get back. Writing them makes your editor catch mistakes
      before you run the code.

  Pydantic BaseModel:
    what: >
      A class that validates, coerces, and serializes data based on type hints.
      Pass wrong types — Pydantic either coerces them or raises a ValidationError.
      Pass a string where an int is expected — it tries int("42") = 42.
      Pass "abc" — ValidationError with a clear message.
    why: >
      This is the foundation of FastAPI (request/response models), LangGraph (state schemas),
      and your OSS projects. Comfortable with Pydantic means comfortable in every
      framework that uses it — which is almost all of them.

  Field() and validators:
    what: >
      Field() annotates a model field with metadata: default, description, constraints.
      @field_validator adds custom validation logic to a specific field.
    why: >
      Field(description="...") generates API documentation in FastAPI automatically.
      Validators enforce business rules (prompt can't be empty) at the model level —
      not scattered across call sites.

  model_dump() and model_validate():
    what: >
      model_dump() → converts a Pydantic model to a plain dict
      model_validate(dict) → creates a model from a dict, with validation
      model_dump_json() → serialize to JSON string
      model_validate_json(str) → deserialize from JSON string
    why: >
      This is how agentlog's FileStorage (Stage 04) will persist runs to disk.
      You'll use these constantly — HTTP responses, Redis, files, LLM tool calls.

  Literal types:
    what: >
      from typing import Literal
      status: Literal["pending", "running", "done", "failed"]
    why: >
      Constraining a string field to a known set of values. Pydantic validates this
      automatically — pass "blah" and you get a ValidationError immediately.
      LangGraph state schemas use Literal types for routing fields.

  Nested models:
    what: >
      A Pydantic model field can itself be a Pydantic model.
      class TokenUsage(BaseModel): ...
      class Run(BaseModel): usage: TokenUsage = ...
    why: >
      LLM APIs return token counts alongside results. Nesting them as a separate
      model keeps Run clean and makes the usage data independently usable.
      Pydantic validates the entire nested structure in one call.

  model_json_schema():
    what: >
      Run.model_json_schema() → returns a JSON Schema dict describing the model.
    why: >
      FastAPI uses this to generate OpenAPI docs automatically.
      LangGraph uses this for tool input schemas.
      Understanding it means you can read any auto-generated API spec.
```

---

## Setup

```bash
pip install pydantic
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Rewrite Run as a Pydantic BaseModel"
    feature: "agentlog — models.py: Run as Pydantic BaseModel with Literal status validation and JSON serialization"
    why: >
      The dataclass Run has no validation. A Pydantic Run validates status,
      rejects empty prompts, and serializes to JSON for free. The interface
      stays the same — mark_done(), mark_failed() — so nothing that uses
      Run needs to change.
    detail: >
      # agentlog/models.py

      from pydantic import BaseModel, Field
      from typing import Literal
      from datetime import datetime

      class Run(BaseModel):
          run_id: str
          prompt: str
          status: Literal["pending", "running", "done", "failed"] = "pending"
          result: str | None = None
          error: str | None = None
          created_at: str = Field(default_factory=lambda: datetime.utcnow().isoformat())

          def mark_done(self, result: str) -> None:
              self.result = result
              self.status = "done"

          def mark_failed(self, error: str) -> None:
              self.error = error
              self.status = "failed"

      Test:
        r = Run(run_id="r1", prompt="hello")
        print(r)                         # clean Pydantic repr
        print(r.model_dump())            # plain dict — this is what FileStorage will use
        print(r.model_dump_json())       # JSON string

  2:
    task: "Test Pydantic's validation — make it reject bad data"
    feature: "agentlog — models.py: Run validation error handling and type coercion behavior"
    why: >
      The whole point of switching from dataclass to Pydantic is that bad data
      gets caught at construction, not three steps later. Verify this works
      before building on top of it.
    detail: >
      from pydantic import ValidationError

      # Status must be one of the Literal values
      try:
          bad = Run(run_id="r1", prompt="hello", status="blah")
      except ValidationError as e:
          print(e)   # read this carefully — Pydantic error messages are precise

      # Pydantic coerces compatible types
      r = Run(run_id=123, prompt="hello")   # run_id expects str, gets int
      print(type(r.run_id))                 # str — Pydantic coerced it
      print(r.run_id)                       # "123"

      This coercion behavior matters when parsing LLM API responses — the API
      might return an integer where you expected a string. Pydantic handles it.

  3:
    task: "Add a field validator to clean prompt input"
    feature: "agentlog — models.py: Run.prompt field validator — reject empty/whitespace, strip on entry"
    why: >
      A run with an empty or whitespace-only prompt is a bug — no LLM call
      will produce a meaningful result. Enforce this at the model level so
      the rest of agentlog never has to check for it.
    detail: >
      from pydantic import field_validator

      class Run(BaseModel):
          run_id: str
          prompt: str
          ...

          @field_validator("prompt")
          @classmethod
          def prompt_not_empty(cls, v: str) -> str:
              if not v.strip():
                  raise ValueError("prompt cannot be empty or whitespace")
              return v.strip()   # strip whitespace as a side effect

      Test:
        try:
            bad = Run(run_id="r1", prompt="   ")
        except ValidationError as e:
            print(e)

        good = Run(run_id="r2", prompt="  hello  ")
        print(repr(good.prompt))   # "hello" — stripped by validator

  4:
    task: "Add a nested TokenUsage model"
    feature: "agentlog — models.py: TokenUsage nested model on Run for tracking LLM token costs"
    why: >
      LLM APIs return token counts with every response. Storing them on Run
      lets agentlog track cost and usage over time. A nested model keeps Run
      clean — usage is its own thing with its own fields — and Pydantic
      validates both levels together.
    detail: >
      class TokenUsage(BaseModel):
          prompt_tokens: int = 0
          completion_tokens: int = 0

          @property
          def total(self) -> int:
              return self.prompt_tokens + self.completion_tokens

      class Run(BaseModel):
          run_id: str
          prompt: str
          status: Literal["pending", "running", "done", "failed"] = "pending"
          result: str | None = None
          error: str | None = None
          usage: TokenUsage = Field(default_factory=TokenUsage)
          created_at: str = Field(default_factory=lambda: datetime.utcnow().isoformat())

          ...validators and methods unchanged...

      Test:
        r = Run(run_id="r1", prompt="hello")
        r.usage.prompt_tokens = 50
        r.usage.completion_tokens = 100
        print(r.usage.total)   # 150
        print(r.model_dump())  # usage is a nested dict

  5:
    task: "Serialization round-trip — this is how FileStorage will work"
    feature: "agentlog — models.py: Run JSON round-trip (model_dump / model_validate) that FileStorage depends on"
    why: >
      Stage 04 builds FileStorage, which saves runs to disk as JSON and loads them back.
      That means: Run → dict → JSON string → file → JSON string → dict → Run.
      Verify now that this round-trip is lossless, before building the storage layer on top of it.
    detail: >
      r = Run(run_id="r1", prompt="test")
      r.mark_done("result text")
      r.usage.prompt_tokens = 30

      # Serialize
      data = r.model_dump()
      json_str = r.model_dump_json()

      # Deserialize
      r2 = Run.model_validate(data)
      r3 = Run.model_validate_json(json_str)

      print(r == r2)   # True
      print(r == r3)   # True

      # This is exactly what FileStorage will do in Stage 04:
      # save: json.dumps(registry.dump_all())
      # load: Run.model_validate(item) for item in raw

  6:
    task: "Inspect the JSON schema"
    feature: "agentlog — models.py: Run.model_json_schema() — the schema FastAPI and LangGraph tools use"
    why: >
      FastAPI generates its /docs page from this schema automatically.
      LangGraph uses it for tool input validation.
      Reading it now means you're never confused by auto-generated API specs later.
    detail: >
      import json
      schema = Run.model_json_schema()
      print(json.dumps(schema, indent=2))

      Read the output:
        - Which fields are required vs optional?
        - What does the "status" field look like? (It should show the Literal values)
        - Where does TokenUsage appear?

  7:
    task: "Update RunRegistry to use Pydantic Run"
    feature: "agentlog — models.py: RunRegistry.dump_all() and load_all() for FileStorage integration"
    why: >
      RunRegistry stores and retrieves Run instances. With Pydantic Run, it
      also gets free serialization — dump_all() returns a list of dicts ready
      for JSON storage. Stage 04's FileStorage will call exactly this method.
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

          def dump_all(self) -> list[dict]:
              return [r.model_dump() for r in self._runs.values()]

          def load_all(self, data: list[dict]) -> None:
              for item in data:
                  run = Run.model_validate(item)
                  self._runs[run.run_id] = run
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Pass a dict with extra unknown fields to Run.model_validate(). Does Pydantic accept it?
    Now add model_config = ConfigDict(extra="forbid") to Run. Try again.
    This is how you harden a model against unexpected API response fields.
  - Set status = None on a created Run. Does Python allow it? Does Pydantic?
  - Call r.model_dump(exclude={"created_at"}) — what does this produce?
    This is how you exclude sensitive fields (like API keys) from serialized output.
  - Do the full round-trip with a Run that has non-ASCII characters in the prompt.
    Does JSON serialization handle it correctly?
```

---

## Done When

```yaml
done_when:
  - Run is a Pydantic BaseModel with status validation, prompt validator, and nested TokenUsage
  - Full JSON round-trip works and produces equal objects
  - RunRegistry.dump_all() and load_all() work correctly — this is what Stage 04 depends on
  - You understand what ValidationError is and how to catch it
  - You can add a new Pydantic model from memory — fields, types, defaults, validator
  - You can explain the difference between a dataclass and a Pydantic BaseModel
```

---

## C#/.NET Comparison

```yaml
if_you_know_csharp:
  BaseModel: "Like a C# record or DTO with built-in JSON serialization and validation"
  field_validator: "Like FluentValidation rules or DataAnnotation attributes"
  model_dump(): "Like JsonSerializer.Serialize() but returns a dict, not a string"
  model_validate(): "Like JsonSerializer.Deserialize<T>() but from a dict"
  ValidationError: "Like a collection of validation exceptions thrown together"
  Literal["a","b"]: "Like an enum constraint on a string property"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between model_dump() and __dict__ on a Pydantic model?
  - What does model_config = ConfigDict(frozen=True) do?
    When would you want an immutable Run?
  - How does Pydantic handle datetime fields — strings or numbers?
  - What is a Pydantic discriminated union and when would you use it?
    (Hint: you'll see this in LangGraph message types — HumanMessage vs AIMessage)
```
