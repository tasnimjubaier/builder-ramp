# Stage 03 — Simple Fastify API

**Status:** not started
**Estimated time:** 1 day
**Type:** standalone
**Depends on:** Stage 01 (TypeScript basics)

Fastify is the standard TypeScript API framework for performance-critical Node.js backends. It's what you'd use if you were building the TypeScript side of a multi-language stack — e.g., a TypeScript gateway in front of Python agents. This stage builds one small API server so the server-side TypeScript pattern feels natural.

---

## What You're Building

A minimal "prompt proxy" API — accepts a prompt via HTTP, calls an LLM, and returns the result. Typed route schemas, typed request/response, error handling. Runnable in one command.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Fastify vs Express:
    what: >
      Fastify has built-in JSON schema validation on routes (using JSON Schema,
      not TypeScript types directly). Faster than Express. TypeScript support
      is first-class. The route schema doubles as documentation.
    why: >
      Express is older and less type-safe by default. Fastify's route typing
      pattern is closer to FastAPI — declare schema, get validation and docs.

  Route typing with TypeScript generics:
    what: >
      fastify.post<{ Body: RequestType; Reply: ResponseType }>("/path", ...)
      The generic parameters tell TypeScript what types request.body and reply have.
    why: >
      Without generics, request.body is any. With them, TypeScript knows exactly
      what fields are available and enforces correct response shape.

  Fastify plugins:
    what: >
      Fastify extends its functionality through plugins (registered with fastify.register()).
      @fastify/cors, @fastify/helmet, etc.
    why: >
      Every real Fastify app uses plugins. Understanding the register() pattern
      is how you add middleware — it's not Express's app.use().
```

---

## Setup

```yaml
dependencies:
  "bun add fastify @fastify/cors openai"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create a Fastify app with typed routes"
    detail: >
      import Fastify from "fastify";
      import cors from "@fastify/cors";

      const app = Fastify({ logger: true });
      await app.register(cors, { origin: true });

      interface PromptBody { prompt: string; maxTokens?: number; }
      interface PromptReply { result: string; model: string; tokensUsed: number; }
      interface ErrorReply  { error: string; detail: string; }

      app.post<{ Body: PromptBody; Reply: PromptReply | ErrorReply }>(
          "/prompt",
          {
              schema: {
                  body: {
                      type: "object",
                      required: ["prompt"],
                      properties: {
                          prompt:    { type: "string", minLength: 1 },
                          maxTokens: { type: "number", minimum: 1, maximum: 2000 },
                      },
                  },
              },
          },
          async (request, reply) => {
              const { prompt, maxTokens = 200 } = request.body;
              // call LLM here (Stage 04 pattern)
              reply.send({ result: `Echo: ${prompt}`, model: "mock", tokensUsed: 0 });
          },
      );

      await app.listen({ port: 3000, host: "0.0.0.0" });
      console.log("Server running on http://localhost:3000");

  2:
    task: "Add a health endpoint and 404 handler"
    detail: >
      app.get("/health", async () => ({ status: "ok", timestamp: new Date().toISOString() }));

      app.setNotFoundHandler(async (request, reply) => {
          reply.status(404).send({ error: "NOT_FOUND", detail: `Route ${request.url} not found` });
      });

      app.setErrorHandler(async (error, request, reply) => {
          app.log.error(error);
          reply.status(500).send({ error: "INTERNAL_ERROR", detail: error.message });
      });

  3:
    task: "Wire in a real LLM call"
    detail: >
      import OpenAI from "openai";

      const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

      // Inside the /prompt handler:
      const response = await openai.chat.completions.create({
          model: "gpt-4o-mini",
          messages: [{ role: "user", content: prompt }],
          max_tokens: maxTokens,
      });

      const content = response.choices[0].message.content ?? "";
      const tokensUsed = response.usage?.total_tokens ?? 0;

      reply.send({ result: content, model: "gpt-4o-mini", tokensUsed });

  4:
    task: "Test with curl"
    detail: >
      curl -X POST http://localhost:3000/prompt \
        -H "Content-Type: application/json" \
        -d '{"prompt": "What is TypeScript?"}'

      # Test validation
      curl -X POST http://localhost:3000/prompt \
        -H "Content-Type: application/json" \
        -d '{}'   # missing required 'prompt' — should return 400
```

---

## Done When

```yaml
done_when:
  - /prompt returns a typed LLM response
  - Missing/invalid body returns a 400 with Fastify's validation error
  - /health returns correctly
  - Error handler returns structured JSON on unexpected errors
  - You can add a new route from memory with typed Body and Reply
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between Fastify's JSON Schema validation and zod?
    Can you use zod with Fastify instead of JSON Schema?
  - What is Fastify's hook system (onRequest, preHandler, onSend)?
    How does it compare to FastAPI's middleware?
  - What does Fastify's logger: true give you and how would you send logs to a
    structured log aggregator (like Datadog or CloudWatch)?
```
