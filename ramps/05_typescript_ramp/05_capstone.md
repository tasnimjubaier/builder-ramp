# Stage 05 — Capstone: TypeScript SDK for the Agent API

**Status:** not started
**Estimated time:** 2–3 days
**Type:** additive — uses all four previous stages
**Depends on:** Stages 01–04 + FastAPI Ramp Capstone (the API being wrapped)

A Python FastAPI service is only as reachable as the clients built for it. This capstone produces a typed TypeScript SDK that wraps your FastAPI agent API — the same API you built in the FastAPI ramp. Anyone with a TypeScript project can install this SDK and call your agent without writing a single HTTP request manually.

This is also the pattern behind every OSS SDK — LangChain.js, OpenAI TypeScript SDK, Anthropic TypeScript SDK. After this stage you will have written one yourself.

---

## What You're Building

`agent-sdk` — a published npm package:

```typescript
import { AgentClient } from "agent-sdk";

const client = new AgentClient({ baseUrl: "http://localhost:8000" });

// Submit a prompt
const run = await client.runs.create({ prompt: "Research Python history" });

// Poll until done
const result = await client.runs.waitForCompletion(run.runId);
console.log(result.answer);

// Or stream status updates
for await (const update of client.runs.stream(run.runId)) {
    console.log(update.status);
}
```

---

## Package Structure

```
agent-sdk/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # exports everything
│   ├── client.ts         # AgentClient class
│   ├── resources/
│   │   └── runs.ts       # RunsResource — all /runs endpoints
│   ├── types.ts          # zod schemas + inferred types
│   └── http.ts           # base HTTP client (from Stage 02)
├── tests/
│   └── runs.test.ts
└── README.md
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Initialize the package"
    detail: >
      mkdir agent-sdk && cd agent-sdk
      bun init
      bun add zod
      bun add -d @types/node typescript

      package.json:
      {
        "name": "agent-sdk",
        "version": "0.1.0",
        "main": "dist/index.js",
        "types": "dist/index.d.ts",
        "scripts": {
          "build": "tsc",
          "test":  "bun test"
        }
      }

  2:
    task: "Port types.ts from Stage 02"
    detail: >
      Port your zod schemas and inferred types from Stage 02:
      RunSchema, RunAcceptedSchema, RunStatusSchema.
      Add a HealthSchema:
        export const HealthSchema = z.object({ status: z.string() });
        export type Health = z.infer<typeof HealthSchema>;

  3:
    task: "Port http.ts from Stage 02"
    detail: >
      The request() function and typed error classes from Stage 02.
      Make the base URL configurable — it comes from AgentClient, not a module constant.

      export function createHttpClient(baseUrl: string) {
          return {
              get:    <T>(path: string, schema: z.ZodType<T>) =>
                          request(`${baseUrl}${path}`, schema),
              post:   <T>(path: string, schema: z.ZodType<T>, body: unknown) =>
                          request(`${baseUrl}${path}`, schema, { method: "POST", body }),
              delete: (path: string) =>
                          request(`${baseUrl}${path}`, z.void(), { method: "DELETE" }),
          };
      }

  4:
    task: "Build RunsResource"
    detail: >
      import type { ReturnType } from "./http";
      import { RunSchema, RunAcceptedSchema, type Run, type RunAccepted } from "./types";

      export interface CreateRunOptions { prompt: string; threadId?: string; }

      export class RunsResource {
          constructor(private http: ReturnType<typeof createHttpClient>) {}

          async create(options: CreateRunOptions): Promise<RunAccepted> {
              return this.http.post("/runs", RunAcceptedSchema, {
                  prompt: options.prompt,
                  thread_id: options.threadId,
              });
          }

          async get(runId: string): Promise<Run> {
              return this.http.get(`/runs/${runId}`, RunSchema);
          }

          async cancel(runId: string): Promise<void> {
              return this.http.delete(`/runs/${runId}`);
          }

          async waitForCompletion(
              runId: string,
              options: { pollIntervalMs?: number; timeoutMs?: number } = {},
          ): Promise<Run> {
              const { pollIntervalMs = 1000, timeoutMs = 60_000 } = options;
              const deadline = Date.now() + timeoutMs;

              while (Date.now() < deadline) {
                  const run = await this.get(runId);
                  if (run.status === "done" || run.status === "failed" || run.status === "cancelled") {
                      return run;
                  }
                  await new Promise(resolve => setTimeout(resolve, pollIntervalMs));
              }
              throw new Error(`Run ${runId} did not complete within ${timeoutMs}ms`);
          }

          async *stream(runId: string): AsyncGenerator<Run> {
              const response = await fetch(
                  `${this.baseUrl}/runs/${runId}/stream`,
                  { headers: { Accept: "text/event-stream" } },
              );
              const reader = response.body!.getReader();
              const decoder = new TextDecoder();

              while (true) {
                  const { done, value } = await reader.read();
                  if (done) break;
                  const text = decoder.decode(value);
                  for (const line of text.split("\n")) {
                      if (line.startsWith("data: ")) {
                          const data = JSON.parse(line.slice(6));
                          const parsed = RunSchema.safeParse(data);
                          if (parsed.success) yield parsed.data;
                      }
                  }
              }
          }
      }

  5:
    task: "Build AgentClient"
    detail: >
      import { createHttpClient } from "./http";
      import { RunsResource } from "./resources/runs";

      export interface AgentClientOptions {
          baseUrl: string;
          apiKey?: string;   // for future auth
      }

      export class AgentClient {
          public runs: RunsResource;

          constructor(options: AgentClientOptions) {
              const http = createHttpClient(options.baseUrl);
              this.runs = new RunsResource(http);
          }
      }

      export * from "./types";

  6:
    task: "Write tests"
    detail: >
      tests/runs.test.ts using bun:test (built-in, no Jest needed):

      import { describe, test, expect, mock } from "bun:test";
      import { AgentClient } from "../src/index";

      describe("RunsResource", () => {
          test("create returns a run id", async () => {
              // mock fetch to return a fixed response
              global.fetch = mock(() => Promise.resolve(new Response(
                  JSON.stringify({ run_id: "test-123", thread_id: "t-1", status: "pending" }),
                  { status: 202, headers: { "Content-Type": "application/json" } },
              )));

              const client = new AgentClient({ baseUrl: "http://localhost:8000" });
              const result = await client.runs.create({ prompt: "test" });
              expect(result.runId).toBe("test-123");
          });

          test("waitForCompletion times out", async () => {
              global.fetch = mock(() => Promise.resolve(new Response(
                  JSON.stringify({ id: "x", thread_id: "t", status: "running",
                                   prompt: "p", created_at: "" }),
                  { headers: { "Content-Type": "application/json" } },
              )));

              const client = new AgentClient({ baseUrl: "http://localhost:8000" });
              await expect(
                  client.runs.waitForCompletion("x", { timeoutMs: 100, pollIntervalMs: 50 })
              ).rejects.toThrow("did not complete");
          });
      });

      bun test

  7:
    task: "Write the README"
    detail: >
      Installation, quickstart (the code snippet at the top of this doc),
      full API reference table (all methods, parameters, return types).

      The README is the public face of the SDK. A developer should be able
      to start using it without reading source code.

  8:
    task: "Build and publish to npm (or jsr.io)"
    detail: >
      bun run build     # compiles TypeScript to dist/
      npm publish --dry-run   # verify the package contents before publishing

      Publish to jsr.io (Deno/Bun registry) as an alternative:
        bunx jsr publish

      Either works. Having a published package URL to put in the README is the point.
```

---

## Done When

```yaml
done_when:
  - AgentClient.runs.create(), .get(), .waitForCompletion(), and .stream() all work
  - Tests pass without hitting the real API
  - Package builds cleanly (tsc produces dist/ with .d.ts type declarations)
  - README quickstart works when installed fresh
  - Published to npm or jsr.io

graduation_test: >
  Can you write the AgentClient class from memory in 20 minutes?
  Can you explain every design decision — why RunsResource is a separate class,
  why zod validates at runtime, why AsyncGenerator for streaming?
  If yes, you're ready to write the TypeScript client SDK for AgentTrace.
```
