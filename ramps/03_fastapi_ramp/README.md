# FastAPI Ramp — Learning Sequence

FastAPI is the standard Python API framework in every AI stack. You need it to deploy agents, expose tools over HTTP, and build the backends that your OSS projects run behind. This ramp gets you there in 5 stages.

Stages 01–04 are standalone — each is a fresh, self-contained project. Stage 05 (capstone) pulls everything together into one deployed FastAPI service wrapping a LangGraph agent.

---

## Structure

```yaml
stages:
  01_routing_and_models:  "First endpoints + Pydantic request/response models. Standalone."
  02_async_and_di:        "Async handlers + dependency injection. Standalone."
  03_background_and_errors: "Background tasks + structured error handling. Standalone."
  04_docker:              "Containerize a FastAPI service. Dockerfile + Compose. Standalone."
  05_capstone:            "Deploy a LangGraph agent behind FastAPI. Additive — uses all four."
```

---

## Completion Criteria

```yaml
done_when:
  - You can build a typed FastAPI service from scratch without looking at docs
  - You understand async def vs def and when each applies in FastAPI
  - You can inject shared resources (LLM client, DB connection) via dependencies
  - You can run a background task that doesn't block the HTTP response
  - You can containerize a FastAPI service and run it with Docker Compose
  - You have a deployed endpoint that accepts a prompt and streams back an agent response
```

---

## Approach

```yaml
rules:
  - Stages 01–04 are isolated — start each one fresh, no shared codebase
  - Stage 05 draws on all four — don't start it until you've done all previous stages
  - Every stage ends with a running service you can curl
  - Use gpt-4o-mini or gemini-flash where LLM calls appear — cost near zero
  - Treat FastAPI as infrastructure, not the product — the agent is always the point
```
