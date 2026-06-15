# TypeScript Ramp — Learning Sequence

TypeScript appears in ~30% of the JDs you scraped and is required for Founding AI Engineer roles. The goal here is not "learn JavaScript" — it's the specific subset the JDs need: typed APIs, async patterns, and SDK writing. You're a Python developer learning just enough TypeScript to build typed HTTP clients, simple API servers, and SDK wrappers.

Stages 01–04 are standalone. Stage 05 (capstone) pulls them together into a typed SDK for your FastAPI agent API.

---

## Structure

```yaml
stages:
  01_ts_for_python_devs:   "TypeScript fundamentals from a Python perspective. Standalone."
  02_typed_http_client:    "Typed fetch + zod validation. Standalone."
  03_fastify_api:          "Simple Fastify API with typed request/response. Standalone."
  04_llm_sdk_wrapper:      "Typed wrapper around an LLM API. Standalone."
  05_capstone:             "TypeScript SDK for your FastAPI agent API. One real artifact."
```

---

## Completion Criteria

```yaml
done_when:
  - You can write typed interfaces and functions without looking up syntax
  - You understand async/await in TypeScript and how it differs from Python's asyncio
  - You can write a Fastify route with typed request body and response
  - You can validate external API responses with zod
  - You have published a typed SDK that wraps your FastAPI agent endpoint
```

---

## Approach

```yaml
rules:
  - Treat TypeScript as "typed JavaScript" — you already know the programming concepts
  - Focus on the type system, not on JavaScript quirks
  - Every stage produces a runnable script or server — not just type exercises
  - Use Bun as the runtime (faster than Node, built-in TypeScript support, no tsconfig needed for basics)
  - Install: curl -fsSL https://bun.sh/install | bash
```
