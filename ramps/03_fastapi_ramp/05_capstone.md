# Stage 05 — Capstone: LangGraph Agent Behind FastAPI

**Status:** not started
**Estimated time:** 3–4 days
**Type:** additive — draws on all four previous stages
**Depends on:** Stages 01–04 (FastAPI patterns) + Agentic Ramp Stages 01–06 (LangGraph)

This is the synthesis. Everything in this ramp — routing, Pydantic models, async, dependency injection, background tasks, error handling, Docker, Compose — comes together here to wrap a real LangGraph agent in a deployable HTTP API.

This is also your first genuinely deployable OSS artifact. After this stage you have something you can push to GitHub, add a Railway deploy button to, and link in a job application.

---

## What You're Building

`agent-api` — a FastAPI service that exposes a LangGraph multi-node agent over HTTP:

```
POST   /runs              submit a prompt, get a run_id back immediately (202)
GET    /runs/{run_id}     poll for status + result
GET    /runs/{run_id}/stream   stream the agent's output as SSE
DELETE /runs/{run_id}     cancel a run
GET    /health            health check
```

The agent runs in the background. State is persisted in Redis. The service is containerized with Docker Compose (API + Redis). A GitHub Actions workflow builds and tests it on every push.

---

## Architecture

```
Client
  │
  POST /runs
  │
  ▼
FastAPI (api service)
  │  - validates request (Pydantic)
  │  - creates run record in Redis
  │  - enqueues background task
  │  - returns run_id (202)
  │
  ├── BackgroundTask: run_agent(run_id, prompt)
  │     │
  │     ▼
  │   LangGraph agent (from agentic ramp)
  │     - multi-node graph
  │     - tool calls
  │     - writes result back to Redis on completion
  │
  └── GET /runs/{run_id}
        │
        ▼
      Redis: load run record, return current status + result
```

---

## Project Structure

```
agent-api/
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .github/
│   └── workflows/
│       └── ci.yml
├── requirements.txt
├── main.py              # FastAPI app, routes
├── agent.py             # LangGraph agent definition
├── store.py             # Redis job store (from Stage 04 pattern)
├── dependencies.py      # Depends() functions (LLM client, settings, store)
├── models.py            # Pydantic request/response models
├── errors.py            # Custom exception classes + handlers
└── tests/
    └── test_runs.py
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Port your agentic ramp agent into agent.py"
    detail: >
      Take the Stage 06 supervisor agent (or Stage 02 tool-calling agent if simpler).
      Wrap it in a function:

        def build_agent() -> CompiledGraph:
            ...
            return graph.compile()

      Keep it simple — the point is the integration, not the agent's intelligence.
      Two nodes and one tool is enough.

  2:
    task: "Define models.py"
    detail: >
      class RunRequest(BaseModel):
          prompt: str
          thread_id: str | None = None   # optional — auto-generated if not provided

      class RunStatus(str, Enum):
          pending = "pending"
          running = "running"
          done = "done"
          failed = "failed"
          cancelled = "cancelled"

      class Run(BaseModel):
          id: str
          thread_id: str
          status: RunStatus
          prompt: str
          result: str | None = None
          error: str | None = None
          created_at: str

      class RunAccepted(BaseModel):
          run_id: str
          thread_id: str
          status: str = "pending"

  3:
    task: "Build store.py (Redis-backed run store)"
    detail: >
      Exactly the save_job/load_job pattern from Stage 04, adapted for Run objects.
      Add a cancel flag: redis_client.set(f"run:{run_id}:cancelled", "1", ex=3600)

      Functions needed:
        save_run(run: Run) -> None
        load_run(run_id: str) -> Run | None
        is_cancelled(run_id: str) -> bool

  4:
    task: "Build dependencies.py"
    detail: >
      from agent import build_agent
      from store import RunStore

      _agent = None

      def get_agent():
          global _agent
          if _agent is None:
              _agent = build_agent()
          return _agent

      def get_store() -> RunStore:
          return RunStore(redis_client)

      def get_settings() -> Settings:
          return settings

      Lazy initialization for the agent — it's built once on first request,
      not at import time (import-time LLM client creation can fail silently).

  5:
    task: "Build the background runner"
    detail: >
      from langchain_core.messages import HumanMessage

      def run_agent_background(run_id: str, prompt: str, thread_id: str) -> None:
          store = RunStore(redis_client)
          run = store.load_run(run_id)
          run.status = RunStatus.running
          store.save_run(run)

          try:
              if store.is_cancelled(run_id):
                  run.status = RunStatus.cancelled
                  store.save_run(run)
                  return

              agent = get_agent()
              result = agent.invoke(
                  {"messages": [HumanMessage(content=prompt)]},
                  config={"configurable": {"thread_id": thread_id}},
              )
              final_message = result["messages"][-1].content

              run.status = RunStatus.done
              run.result = final_message
          except Exception as e:
              run.status = RunStatus.failed
              run.error = str(e)

          store.save_run(run)

  6:
    task: "Build main.py routes"
    detail: >
      @app.post("/runs", response_model=RunAccepted, status_code=202)
      async def create_run(
          body: RunRequest,
          background_tasks: BackgroundTasks,
          store: RunStore = Depends(get_store),
      ) -> RunAccepted:
          run_id = str(uuid.uuid4())
          thread_id = body.thread_id or str(uuid.uuid4())
          run = Run(id=run_id, thread_id=thread_id, status=RunStatus.pending,
                    prompt=body.prompt, created_at=datetime.utcnow().isoformat())
          store.save_run(run)
          background_tasks.add_task(run_agent_background, run_id, body.prompt, thread_id)
          return RunAccepted(run_id=run_id, thread_id=thread_id)

      @app.get("/runs/{run_id}", response_model=Run)
      async def get_run(run_id: str, store: RunStore = Depends(get_store)) -> Run:
          run = store.load_run(run_id)
          if not run:
              raise RunNotFoundError(run_id)
          return run

      @app.delete("/runs/{run_id}", status_code=204)
      async def cancel_run(run_id: str, store: RunStore = Depends(get_store)) -> None:
          run = store.load_run(run_id)
          if not run:
              raise RunNotFoundError(run_id)
          redis_client.set(f"run:{run_id}:cancelled", "1", ex=3600)

      @app.get("/health")
      async def health() -> dict:
          return {"status": "ok"}

  7:
    task: "Add SSE streaming to GET /runs/{run_id}/stream"
    detail: >
      from fastapi.responses import StreamingResponse
      import asyncio

      @app.get("/runs/{run_id}/stream")
      async def stream_run(run_id: str, store: RunStore = Depends(get_store)):
          async def event_generator():
              while True:
                  run = store.load_run(run_id)
                  if not run:
                      yield f"data: {json.dumps({'error': 'not found'})}\n\n"
                      return
                  yield f"data: {run.model_dump_json()}\n\n"
                  if run.status in (RunStatus.done, RunStatus.failed, RunStatus.cancelled):
                      return
                  await asyncio.sleep(0.5)

          return StreamingResponse(event_generator(), media_type="text/event-stream")

      Test with:
        curl -N http://localhost:8000/runs/{run_id}/stream

  8:
    task: "Write tests/test_runs.py"
    detail: >
      Use TestClient + dependency_overrides to mock the agent:

        def mock_agent():
            class FakeAgent:
                def invoke(self, input, config=None):
                    return {"messages": [AIMessage(content="mocked response")]}
            return FakeAgent()

        app.dependency_overrides[get_agent] = mock_agent

      Tests:
        - POST /runs returns 202 with run_id
        - GET /runs/{id} returns the run
        - GET /runs/nonexistent returns 404 with correct error shape
        - DELETE /runs/{id} sets cancelled flag
        - GET /health returns {"status": "ok"}

  9:
    task: "Containerize and deploy Compose stack"
    detail: >
      services:
        api:
          build: .
          ports: ["8000:8000"]
          environment:
            - REDIS_HOST=redis
            - OPENAI_API_KEY=${OPENAI_API_KEY}
          depends_on: [redis]
        redis:
          image: redis:7-alpine

      docker compose up --build
      Submit a run, poll for it, stream it.

  10:
    task: "Write a clean README.md"
    detail: >
      - What it is (2 sentences)
      - Architecture diagram (text/ASCII is fine)
      - Quickstart: docker compose up + curl examples
      - Endpoints table
      - Deploy to Railway button (optional but impressive)

      This README is the artifact. A hiring manager who sees this repo
      understands in 60 seconds what you built and that it actually runs.
```

---

## Done When

```yaml
done_when:
  - docker compose up starts everything and all 5 endpoints work
  - A submitted run completes in the background and the result is retrievable
  - Cancellation prevents a pending run from executing
  - SSE streaming endpoint delivers status updates in real time
  - Tests pass with mocked agent (no real LLM calls in CI)
  - GitHub Actions CI is green
  - README is clean enough to link in a job application
  - You can describe every design decision — why background tasks, why Redis,
    why dependency injection for the agent, why SSE for streaming
```

---

## After the Capstone

```yaml
what_this_unlocks:
  - This repo is the shell for every future OSS project
  - AgentTrace, AgentEval, and AgentResilience all get deployed behind exactly
    this pattern — FastAPI wrapper, background runner, Redis state, Compose stack
  - The SSE streaming endpoint is the pattern for real-time agent output in any UI
  - You now have a working, deployable, testable AI backend that is not a tutorial

graduation_test:
  - Can you explain every file in the project without looking at your notes?
  - Can you add a new endpoint (POST /runs/{id}/retry) from memory?
  - If a hiring manager asks "how would you deploy this to production?",
    can you answer without hesitation?
```
