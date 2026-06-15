# Stage 02 — Async Handlers and Dependency Injection

**Status:** not started
**Estimated time:** 1–2 days
**Type:** standalone — fresh project
**Depends on:** Stage 01 concepts (routes, Pydantic models)

Two patterns that appear in every serious FastAPI codebase. Async lets your service handle many requests without blocking while waiting for I/O (LLM calls, DB queries, HTTP requests). Dependency injection gives you a clean way to share resources — an LLM client, a config object, an API key — across routes without global variables.

---

## What You're Building

A toy "AI snippet generator" API — endpoints that call an LLM and return results. Two versions of each endpoint: sync (blocking) and async (non-blocking). A shared LLM client injected via dependency, not imported as a global.

This is a realistic approximation of what every AI backend looks like.

---

## Concepts This Stage Teaches

```yaml
concepts:
  async def vs def in FastAPI:
    what: >
      In FastAPI, async def routes run on the event loop — they yield control while
      waiting for I/O (a network call, a file read) so other requests can be served.
      Plain def routes run in a thread pool — FastAPI runs them in a separate thread
      so they don't block the event loop.
    why: >
      LLM API calls are I/O — they wait for the network. An async route that awaits
      the LLM call can serve hundreds of concurrent requests efficiently. A sync route
      doing the same thing ties up a thread for the full LLM latency. Every AI backend
      with real traffic uses async routes for LLM calls.
    important: >
      The rule: if you call async libraries (httpx, openai, langchain's async methods),
      use async def. If you call sync libraries (requests, some DB drivers), use def.
      Calling sync blocking code inside async def is the #1 FastAPI performance mistake.

  Dependency injection with Depends():
    what: >
      A function that returns a resource, declared as a parameter with Depends().
      FastAPI calls the dependency function, gets the result, and passes it to your route.
      Dependencies can themselves have dependencies — they compose.
    why: >
      Without DI, shared resources (LLM client, settings, DB session) become module-level
      globals or get re-created on every request. DI gives you: one place to configure
      the resource, clean testability (swap the real client for a mock in tests),
      and lifecycle management (open/close DB connections per request).

  Dependency with yield (lifespan):
    what: >
      A dependency function that uses yield instead of return. Code before yield runs
      on setup (get a DB connection), code after yield runs on teardown (close it).
    why: >
      This is how you manage resources that need cleanup — database connections,
      HTTP client sessions, file handles. The pattern also applies to FastAPI's
      lifespan context manager for app-level startup/shutdown.

  Settings via pydantic-settings:
    what: >
      A BaseSettings class that reads from environment variables automatically.
      Declare a field, set the env var, BaseSettings finds it.
    why: >
      Every production service needs config from env vars (API keys, model names,
      base URLs). pydantic-settings is the standard way to do this in FastAPI stacks.
      It validates and types your config the same way Pydantic validates request bodies.
```

---

## Setup

```yaml
dependencies:
  - fastapi
  - uvicorn[standard]
  - openai              # or google-generativeai
  - pydantic-settings
  - python-dotenv

install: "pip install fastapi uvicorn[standard] openai pydantic-settings python-dotenv"
```

`.env` file:
```
OPENAI_API_KEY=your-key
MODEL_NAME=gpt-4o-mini
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define Settings with pydantic-settings"
    detail: >
      from pydantic_settings import BaseSettings

      class Settings(BaseSettings):
          openai_api_key: str
          model_name: str = "gpt-4o-mini"

          class Config:
              env_file = ".env"

      # module-level singleton — created once
      settings = Settings()

  2:
    task: "Build a dependency that provides an LLM client"
    detail: >
      from openai import AsyncOpenAI
      from fastapi import Depends

      def get_llm_client() -> AsyncOpenAI:
          return AsyncOpenAI(api_key=settings.openai_api_key)

      # FastAPI will call get_llm_client() and inject the result
      # Use in route: client: AsyncOpenAI = Depends(get_llm_client)

  3:
    task: "Build a settings dependency"
    detail: >
      def get_settings() -> Settings:
          return settings

      # Any route can inject settings without importing the module-level object directly
      # This makes the route testable — swap settings in tests via dependency override

  4:
    task: "Build async route: POST /generate"
    detail: >
      from fastapi import FastAPI
      from pydantic import BaseModel

      app = FastAPI()

      class GenerateRequest(BaseModel):
          prompt: str
          max_tokens: int = 200

      class GenerateResponse(BaseModel):
          result: str
          model: str

      @app.post("/generate", response_model=GenerateResponse)
      async def generate(
          body: GenerateRequest,
          client: AsyncOpenAI = Depends(get_llm_client),
          cfg: Settings = Depends(get_settings),
      ) -> GenerateResponse:
          response = await client.chat.completions.create(
              model=cfg.model_name,
              messages=[{"role": "user", "content": body.prompt}],
              max_tokens=body.max_tokens,
          )
          return GenerateResponse(
              result=response.choices[0].message.content,
              model=cfg.model_name,
          )

  5:
    task: "Add a dependency with yield for a 'session' resource"
    detail: >
      Build a fake session counter to demonstrate yield-based dependencies:

      from typing import Generator

      def get_request_session() -> Generator[dict, None, None]:
          session = {"request_id": str(uuid.uuid4()), "cost": 0.0}
          print(f"Session opened: {session['request_id']}")
          yield session
          print(f"Session closed: {session['request_id']} | cost: {session['cost']}")

      Update /generate to accept session: dict = Depends(get_request_session)
      and set session["cost"] = 0.001 inside the handler.

      Watch the startup and teardown print statements fire on every request.

  6:
    task: "Add a sync route for comparison"
    detail: >
      Build POST /generate-sync that does the same thing but with the sync OpenAI
      client and a plain def (not async def).

      from openai import OpenAI

      def get_sync_client() -> OpenAI:
          return OpenAI(api_key=settings.openai_api_key)

      @app.post("/generate-sync", response_model=GenerateResponse)
      def generate_sync(
          body: GenerateRequest,
          client: OpenAI = Depends(get_sync_client),
      ) -> GenerateResponse:
          ...

      Both work. The difference matters under load — not visible in single-request testing.

  7:
    task: "Write a dependency override in a test"
    detail: >
      from fastapi.testclient import TestClient

      # Override the real LLM client with a mock
      def mock_llm_client():
          class MockClient:
              class chat:
                  class completions:
                      @staticmethod
                      async def create(**kwargs):
                          class MockResponse:
                              choices = [type("c", (), {"message": type("m", (), {"content": "mock result"})()})()]
                          return MockResponse()
          return MockClient()

      app.dependency_overrides[get_llm_client] = mock_llm_client

      client = TestClient(app)
      response = client.post("/generate", json={"prompt": "hello"})
      assert response.json()["result"] == "mock result"

      app.dependency_overrides.clear()   # reset after test

      This pattern — dependency_overrides — is how you test FastAPI routes without
      hitting real APIs. You will use it constantly.
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Call a synchronous blocking function (time.sleep(2)) inside an async def route.
    Make 3 concurrent requests and time how long total they take.
    Then move it to a def route. Time again. Understand the difference.
  - Forget to await an async LLM call inside an async def route.
    What does Python return? What error do you see?
  - Remove Depends() and import the settings object directly at module level instead.
    Does it still work? Now try to override it in a test. What breaks?
```

---

## Done When

```yaml
done_when:
  - /generate works and returns an LLM response
  - You can explain the difference between async def and def in one sentence
  - You can add a new dependency and inject it into a route from memory
  - You've written a dependency override in a test and confirmed it works
  - You understand why yield-based dependencies exist
```

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What happens if you Depends() on a function that itself Depends() on another function?
    Is this recursive? What's the limit?
  - Can two routes share a dependency instance within the same request? Across requests?
  - What is the difference between Depends() at the route level and at the router level?
  - What is FastAPI's lifespan context manager and how does it differ from yield dependencies?
```
