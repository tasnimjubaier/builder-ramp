# Stage 03 — Type Hints and Pydantic

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — extend devtool/models.py
**Depends on:** Stage 02 (classes, dataclasses)

Python's type hints don't enforce anything at runtime — they're documentation that tools like your editor, mypy, and Pydantic read. Pydantic takes type hints seriously: it uses them to validate data at runtime, coerce types, and generate JSON schemas. FastAPI, LangGraph, and every serious AI library is built on Pydantic. This stage makes type hints second nature and Pydantic familiar.

---

## What You're Building

Rewrite `devtool/models.py` to use Pydantic BaseModel instead of dataclasses. Add validation, serialization, and schema generation. By the end, the models are what you'd actually use in a FastAPI or LangGraph project.

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
      what to pass and what you get back. Writing them makes your editor catch mistakes.

  Pydantic BaseModel:
    what: >
      A class that validates, coerces, and serializes data based on type hints.
      Pass a dict with wrong types — it either coerces them or raises a ValidationError.
      Pass a string where an int is expected — it tries int("42") = 42. Passes "abc" — ValidationError.
    why: >
      This is the foundation of FastAPI (request/response models), LangGraph (state schemas),
      and your OSS projects. If you're comfortable with Pydantic, you're comfortable in every
      framework that uses it — which is almost all of them.

  Field() and validators:
    what: >
      Field() annotates a model field with metadata: default, description, constraints.
      @field_validator adds custom validation logic to a specific field.
    why: >
      Field(description="...") generates API documentation in FastAPI automatically.
      Validators let you enforce business rules (status must be in a known set) at the model level.

  model_dump() and model_validate():
    what: >
      model_dump() → converts a Pydantic model to a plain dict
      model_validate(dict) → creates a model from a dict, with validation
      model_dump_json() → serialize to JSON string
      model_validate_json(str) → deserialize from JSON string
    why: >
      This is how you move data between your Python code and the outside world
      (HTTP responses, Redis storage, files). You'll use these constantly.

  Literal and Enum types:
    what: >
      from typing import Literal
      status: Literal["pending", "running", "done", "failed"]

      from enum import Enum
      class Status(str, Enum):
          pending = "pending"
          done = "done"
    why: >
      Constraining a string field to a known set of values. Pydantic validates this automatically.
      LangGraph state schemas use Literal types for routing fields.

  Nested models:
    what: >
      A Pydantic model field can itself be a Pydantic model.
      class RunResult(BaseModel): content: str; tokens: int
      class Run(BaseModel): result: RunResult | None = None
    why: >
      Real data is nested. LangGraph messages are nested Pydantic models.
      Pydantic validates the entire nested structure in one call.

  model_json_schema():
    what: >
      Run.model_json_schema() → returns a JSON Schema dict describing the model.
    why: >
      FastAPI uses this to generate OpenAPI docs. LangGraph uses this for tool input schemas.
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
    detail: >
      # devtool/models.py

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
              # Pydantic models are mutable by default
              self.result = result
              self.status = "done"

          def mark_failed(self, error: str) -> None:
              self.error = error
              self.status = "failed"

      Test:
        r = Run(run_id="r1", prompt="hello")
        print(r)                         # pretty Pydantic repr
        print(r.model_dump())            # plain dict
        print(r.model_dump_json())       # JSON string

  2:
    task: "Test Pydantic's validation"
    detail: >
      from pydantic import ValidationError

      # This should raise a ValidationError
      try:
          bad = Run(run_id="r1", prompt="hello", status="invalid_status")
      except ValidationError as e:
          print(e)   # read the error carefully — Pydantic error messages are precise

      # This should work — Pydantic coerces compatible types
      r = Run(run_id=123, prompt="hello")   # run_id expects str, gets int
      print(type(r.run_id))                 # str — Pydantic coerced it
      print(r.run_id)                       # "123"

  3:
    task: "Add a nested model"
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

      Test:
        r = Run(run_id="r1", prompt="hello")
        r.usage.prompt_tokens = 50
        r.usage.completion_tokens = 100
        print(r.usage.total)   # 150
        print(r.model_dump())  # usage is nested dict

  4:
    task: "Add a field validator"
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
              return v.strip()   # also strips whitespace as a side effect

      Test:
        try:
            bad = Run(run_id="r1", prompt="   ")
        except ValidationError as e:
            print(e)

        good = Run(run_id="r2", prompt="  hello  ")
        print(repr(good.prompt))   # "hello" — stripped by validator

  5:
    task: "Use model_validate() and serialization round-trip"
    detail: >
      r = Run(run_id="r1", prompt="test")
      r.mark_done("result text")

      # Serialize to dict
      data = r.model_dump()
      print(data)

      # Deserialize back
      r2 = Run.model_validate(data)
      print(r2)
      print(r == r2)   # True — Pydantic generates __eq__ comparing all fields

      # JSON round-trip
      json_str = r.model_dump_json()
      r3 = Run.model_validate_json(json_str)
      print(r == r3)   # True

  6:
    task: "Inspect the JSON schema"
    detail: >
      import json
      schema = Run.model_json_schema()
      print(json.dumps(schema, indent=2))

      Read the output carefully:
        - Which fields are required vs optional?
        - What does the "status" field look like in the schema?
        - Where does the TokenUsage nested schema appear?

      This is exactly what FastAPI generates for its /docs page.

  7:
    task: "Rewrite RunRegistry to store Pydantic models"
    detail: >
      Update RunRegistry from Stage 02 to work with the new Pydantic Run.
      Key change: serialization is now free — registry can dump all runs to JSON:

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
  - Pass a dict with extra fields to Run.model_validate(). Does Pydantic accept it?
    Now add model_config = ConfigDict(extra="forbid") to Run. Try again.
  - Set status = None on a created Run. Does Python allow it? What does Pydantic do?
  - Nest another Run inside Run (circular). Does Pydantic handle this?
  - Call r.model_dump(exclude={"created_at"}) — what does this do?
    This is how you exclude sensitive fields from serialized output.
```

---

## Done When

```yaml
done_when:
  - Run is a Pydantic BaseModel with validation, nested model, and a field validator
  - You've done a full JSON serialization round-trip and confirmed equality
  - You've read the JSON schema output and can explain each field
  - You understand what ValidationError is and how to catch it
  - You can add a new Pydantic model from memory — fields, types, defaults, validator
  - You can explain the difference between a dataclass and a Pydantic model
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
    Are they always the same? When are they different?
  - What does model_config = ConfigDict(frozen=True) do?
    When would you want an immutable Pydantic model?
  - How does Pydantic handle datetime fields — does it serialize them as strings or numbers?
  - What is a Pydantic discriminated union and when would you use it?
    (Hint: you'll see this in LangGraph message types)
```
