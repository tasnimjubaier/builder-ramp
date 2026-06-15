# Stage 02 — Typed HTTP Client with Zod

**Status:** not started
**Estimated time:** 1 day
**Type:** standalone
**Depends on:** Stage 01 (TypeScript basics)

Every SDK you write in TypeScript makes HTTP calls and needs to validate that the response matches what you expect. TypeScript's type system only exists at compile time — it can't validate data that arrives at runtime from an external API. Zod bridges that gap: it validates and types incoming JSON at runtime, so your TypeScript types and your actual data stay in sync.

---

## What You're Building

A typed HTTP client module that:
1. Makes GET and POST requests with typed request/response bodies
2. Validates responses with zod schemas at runtime
3. Handles errors with typed error classes
4. Retries on transient failures (408, 429, 5xx)

This is the foundation of the SDK in Stage 05.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Zod:
    what: >
      A TypeScript-first schema declaration and runtime validation library.
      You define a schema, zod validates an unknown value against it,
      and if it passes, TypeScript knows the exact type of the result.
    why: >
      fetch() returns Promise<Response> — the JSON body is typed as any.
      Zod turns that any into a typed, validated object. If the API changes
      its response shape, zod catches it at runtime instead of causing a
      silent type mismatch deeper in your code.

  z.infer<typeof Schema>:
    what: >
      TypeScript utility that extracts the TypeScript type from a zod schema.
      const RunSchema = z.object({ id: z.string(), status: z.string() });
      type Run = z.infer<typeof RunSchema>;   // { id: string; status: string }
    why: >
      Write the schema once, get the TypeScript type for free.
      Single source of truth — schema and type are always in sync.

  Response validation pattern:
    what: >
      const raw = await response.json();
      const result = RunSchema.safeParse(raw);
      if (!result.success) throw new ValidationError(result.error);
      return result.data;   // typed as Run
    why: >
      safeParse returns { success: true, data: T } | { success: false, error: ZodError }
      — you handle validation failure explicitly. This is cleaner than try/catch on parse().

  Typed error classes:
    what: >
      class ApiError extends Error {
          constructor(public statusCode: number, message: string) { super(message); }
      }
    why: >
      Catching Error gives you no type information. Typed error classes let callers
      do if (e instanceof ApiError) e.statusCode — same pattern as Python's custom exceptions.
```

---

## Setup

```yaml
dependencies:
  bun: "bun add zod"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define zod schemas for your FastAPI Run models"
    detail: >
      import { z } from "zod";

      export const RunStatusSchema = z.enum(["pending", "running", "done", "failed", "cancelled"]);

      export const RunSchema = z.object({
          id: z.string(),
          thread_id: z.string(),
          status: RunStatusSchema,
          prompt: z.string(),
          result: z.string().nullable().optional(),
          error: z.string().nullable().optional(),
          created_at: z.string(),
      });

      export const RunAcceptedSchema = z.object({
          run_id: z.string(),
          thread_id: z.string(),
          status: z.string(),
      });

      export type Run = z.infer<typeof RunSchema>;
      export type RunAccepted = z.infer<typeof RunAcceptedSchema>;

  2:
    task: "Define typed error classes"
    detail: >
      export class ApiError extends Error {
          constructor(
              public statusCode: number,
              public detail: string,
              public errorCode?: string,
          ) {
              super(`API Error ${statusCode}: ${detail}`);
              this.name = "ApiError";
          }
      }

      export class ValidationError extends Error {
          constructor(public issues: z.ZodIssue[]) {
              super(`Validation failed: ${issues.map(i => i.message).join(", ")}`);
              this.name = "ValidationError";
          }
      }

      export class NetworkError extends Error {
          constructor(message: string, public cause?: unknown) {
              super(message);
              this.name = "NetworkError";
          }
      }

  3:
    task: "Build the base HTTP client"
    detail: >
      type HttpMethod = "GET" | "POST" | "DELETE" | "PATCH";

      interface RequestOptions {
          method?: HttpMethod;
          body?: unknown;
          headers?: Record<string, string>;
          retries?: number;
      }

      export async function request<T>(
          url: string,
          schema: z.ZodType<T>,
          options: RequestOptions = {},
      ): Promise<T> {
          const { method = "GET", body, headers = {}, retries = 3 } = options;

          const fetchOptions: RequestInit = {
              method,
              headers: { "Content-Type": "application/json", ...headers },
              body: body ? JSON.stringify(body) : undefined,
          };

          let lastError: unknown;
          for (let attempt = 1; attempt <= retries; attempt++) {
              try {
                  const response = await fetch(url, fetchOptions);

                  if (!response.ok) {
                      const errorBody = await response.json().catch(() => ({}));
                      throw new ApiError(
                          response.status,
                          errorBody.detail ?? response.statusText,
                          errorBody.error,
                      );
                  }

                  const raw = await response.json();
                  const parsed = schema.safeParse(raw);
                  if (!parsed.success) {
                      throw new ValidationError(parsed.error.issues);
                  }
                  return parsed.data;

              } catch (e) {
                  lastError = e;
                  if (e instanceof ApiError && ![408, 429, 500, 502, 503].includes(e.statusCode)) {
                      throw e;   // non-retryable
                  }
                  if (attempt < retries) {
                      await new Promise(resolve => setTimeout(resolve, 2 ** attempt * 100));
                  }
              }
          }
          throw lastError;
      }

  4:
    task: "Write typed convenience methods"
    detail: >
      export const http = {
          get: <T>(url: string, schema: z.ZodType<T>) =>
              request(url, schema, { method: "GET" }),

          post: <T>(url: string, schema: z.ZodType<T>, body: unknown) =>
              request(url, schema, { method: "POST", body }),

          delete: (url: string) =>
              request(url, z.void(), { method: "DELETE" }),
      };

  5:
    task: "Test against a real endpoint"
    detail: >
      Start your FastAPI ramp capstone (docker compose up).
      Then test:

      import { http, RunAcceptedSchema, RunSchema } from "./client";

      const accepted = await http.post(
          "http://localhost:8000/runs",
          RunAcceptedSchema,
          { prompt: "Hello from TypeScript" },
      );
      console.log("Run accepted:", accepted.run_id);

      // Poll for result
      let run = await http.get(`http://localhost:8000/runs/${accepted.run_id}`, RunSchema);
      while (run.status === "pending" || run.status === "running") {
          await new Promise(r => setTimeout(r, 1000));
          run = await http.get(`http://localhost:8000/runs/${accepted.run_id}`, RunSchema);
      }
      console.log("Result:", run.result);

  6:
    task: "Test error handling"
    detail: >
      import { ApiError, ValidationError } from "./client";

      try {
          await http.get("http://localhost:8000/runs/nonexistent", RunSchema);
      } catch (e) {
          if (e instanceof ApiError) {
              console.log(`Got expected error: ${e.statusCode} ${e.errorCode}`);
          }
      }

      // Test validation error: make the schema stricter than the response
      const StrictSchema = RunSchema.extend({ nonexistent_field: z.string() });
      try {
          await http.get(`http://localhost:8000/runs/${accepted.run_id}`, StrictSchema);
      } catch (e) {
          if (e instanceof ValidationError) {
              console.log("Validation failed as expected:", e.issues[0].message);
          }
      }
```

---

## Done When

```yaml
done_when:
  - request() makes typed HTTP calls with zod validation
  - Retry logic triggers on 429/5xx and stops on 404
  - ApiError and ValidationError are distinct, catchable, and typed
  - The test against the FastAPI endpoint works end-to-end
  - You can explain z.infer<typeof Schema> and why it matters
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between z.parse() and z.safeParse()?
    When would you use each?
  - How does zod handle extra fields in a response object that aren't in your schema?
    What does .strict() do?
  - What is zod's z.discriminatedUnion() and when is it better than z.union()?
  - How would you handle paginated responses — schemas that vary based on a "next_page" field?
```
