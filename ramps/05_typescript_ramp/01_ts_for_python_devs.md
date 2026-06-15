# Stage 01 — TypeScript for Python Developers

**Status:** not started
**Estimated time:** 1–2 days
**Type:** standalone
**Depends on:** Python familiarity — nothing else

TypeScript is JavaScript with a type system bolted on. As a Python developer you already know variables, functions, classes, loops, and async/await conceptually. What's new is the syntax and the type system. This stage maps every Python concept you know to its TypeScript equivalent and highlights the real differences — not the superficial ones.

---

## What You're Building

A single `learn.ts` file that works through every concept below with runnable examples. Not a project — a structured scratchpad. Run it with `bun learn.ts` and iterate.

---

## Concept Map: Python → TypeScript

```yaml
concepts:
  Variables:
    python: "x = 5  /  x: int = 5"
    typescript: "let x = 5  /  let x: number = 5  /  const x = 5 (immutable)"
    key_difference: >
      TypeScript has const (block-scoped immutable), let (block-scoped mutable),
      and var (function-scoped, avoid). Python has no const — use ALL_CAPS by convention.
      TypeScript infers types — you don't always need to annotate.

  Functions:
    python: "def greet(name: str) -> str: return f'Hello {name}'"
    typescript: "function greet(name: string): string { return `Hello ${name}`; }"
    also: "const greet = (name: string): string => `Hello ${name}`;"
    key_difference: >
      Arrow functions (=>) are the preferred modern style.
      Template literals use backticks, not f-strings.
      Return type comes after the parameter list with a colon.

  Types and interfaces:
    python: "from dataclasses import dataclass  /  from pydantic import BaseModel"
    typescript: "interface  /  type  /  class"
    detail: >
      interface User { id: number; name: string; email?: string; }
      // The ? makes email optional — equivalent to email: str | None = None in Python

      type Status = "pending" | "done" | "failed"
      // Union of string literals — equivalent to Python's Literal["pending", "done", "failed"]

  Arrays and objects:
    python: "list[str]  /  dict[str, Any]"
    typescript: "string[]  /  Record<string, unknown>  /  { key: ValueType }"
    key_difference: >
      TypeScript objects are not dicts — they have typed keys and values defined
      at the type level. Record<string, T> is the closest equivalent to dict[str, T].

  Async/await:
    python: >
      async def fetch() -> str:
          result = await some_coroutine()
          return result
    typescript: >
      async function fetch(): Promise<string> {
          const result = await somePromise();
          return result;
      }
    key_difference: >
      TypeScript async functions return Promise<T>, not a coroutine.
      await works the same way but you must be inside an async function.
      There is no event loop to manage — the runtime handles it.
      Top-level await works in Bun and modern Node.

  Error handling:
    python: "try / except ExceptionType as e"
    typescript: "try / catch (e) { if (e instanceof Error) ... }"
    key_difference: >
      TypeScript catch gives you unknown type, not a typed exception.
      You must narrow the type: if (e instanceof MyError) ...
      There is no equivalent to Python's multiple except clauses in one try block.

  Null handling:
    python: "Optional[str]  /  str | None"
    typescript: "string | null  /  string | undefined  /  string?"
    key_difference: >
      TypeScript has BOTH null and undefined — they are different.
      undefined = variable declared but not assigned.
      null = explicitly set to "no value".
      Use optional chaining: user?.email?.toLowerCase() — returns undefined if any step is null.
      Use nullish coalescing: value ?? "default" — returns "default" if value is null/undefined.

  Classes:
    python: "class Dog: def __init__(self, name: str): self.name = name"
    typescript: >
      class Dog {
          constructor(public name: string) {}   // 'public' auto-creates the property
          bark(): string { return `${this.name} says woof`; }
      }
    key_difference: >
      Access modifiers: public (default), private, protected — enforced at compile time.
      Constructor parameter shorthand with public/private/readonly creates and assigns in one line.

  Generics:
    python: "def first(items: list[T]) -> T"
    typescript: "function first<T>(items: T[]): T { return items[0]; }"
    key_difference: Syntax differs but concept is identical.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Install Bun and create learn.ts"
    detail: >
      curl -fsSL https://bun.sh/install | bash
      touch learn.ts
      bun learn.ts   # runs TypeScript directly, no compilation step

  2:
    task: "Work through variables, functions, and types"
    detail: >
      Write each example from the concept map above.
      Run after each section: bun learn.ts
      Fix any type errors — read them carefully, they're precise.

  3:
    task: "Build a typed data structure"
    detail: >
      Model a Run (from the FastAPI ramp) in TypeScript:

      interface Run {
          id: string;
          threadId: string;
          status: "pending" | "running" | "done" | "failed";
          prompt: string;
          result?: string;   // optional
          error?: string;
          createdAt: string;
      }

      Write a function that creates a Run and another that formats it for display.
      Note: TypeScript uses camelCase by convention (threadId not thread_id).

  4:
    task: "Write an async function with error handling"
    detail: >
      async function fetchWithTimeout(url: string, timeoutMs: number): Promise<string> {
          const controller = new AbortController();
          const timeout = setTimeout(() => controller.abort(), timeoutMs);

          try {
              const response = await fetch(url, { signal: controller.signal });
              if (!response.ok) {
                  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
              }
              return await response.text();
          } catch (e) {
              if (e instanceof Error && e.name === "AbortError") {
                  throw new Error(`Request timed out after ${timeoutMs}ms`);
              }
              throw e;
          } finally {
              clearTimeout(timeout);
          }
      }

      Call it against https://httpbin.org/get and print the result.

  5:
    task: "Understand the module system"
    detail: >
      Create utils.ts:
        export function formatRun(run: Run): string { ... }
        export type { Run };

      Import in learn.ts:
        import { formatRun, type Run } from "./utils";

      Note: import type { Run } is a type-only import — it disappears at runtime.
      This is important for understanding SDK design (Stage 04).
```

---

## Done When

```yaml
done_when:
  - You can write a typed function from memory without looking up syntax
  - You understand the difference between interface and type
  - You understand null vs undefined and optional chaining
  - You can write an async function with try/catch and proper type narrowing
  - You can export and import types and functions across files
  - You can read a TypeScript error message and understand what it's saying
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between interface and type in TypeScript?
    When does the choice matter?
  - What does unknown mean vs any? Why is unknown safer?
  - What is a discriminated union and how does TypeScript narrow types with it?
    (Hint: if (run.status === "done") — TypeScript knows run.result is defined here)
  - What is the difference between CommonJS (require()) and ES modules (import/export)?
    Which does Bun use by default?
```
