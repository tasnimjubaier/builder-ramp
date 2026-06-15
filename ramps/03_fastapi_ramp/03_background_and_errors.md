# Stage 03 — Background Tasks and Structured Error Handling

**Status:** not started
**Estimated time:** 1 day
**Type:** standalone — fresh project
**Depends on:** Stage 01–02 concepts (routes, models, async)

Two patterns every production AI API needs. Background tasks let you kick off work — logging, sending a webhook, writing to a DB — after the HTTP response has already been sent, without making the caller wait. Structured error handling makes sure every failure your API can produce is a typed, predictable response rather than a raw 500.

---

## What You're Building

A toy "agent job queue" API. The caller submits a job, gets an immediate `202 Accepted` response with a job ID, and the job runs in the background. The caller can poll a second endpoint to check job status. Errors (job not found, invalid input, unexpected failures) all return structured JSON with consistent shape.

This is a realistic pattern for AI backends — LLM calls are too slow to block an HTTP response.

---

## Concepts This Stage Teaches

```yaml
concepts:
  BackgroundTasks:
    what: >
      A FastAPI utility injected into a route. You call background_tasks.add_task(fn, *args)
      before returning the response. FastAPI runs fn(*args) after the response is sent —
      the caller doesn't wait for it.
    why: >
      LLM calls take 2–30 seconds. Making a caller wait for a full agent run on a
      synchronous endpoint is bad UX. The pattern: accept the job, return a job ID
      immediately, run the agent in background, let the caller poll for results.
      This is the architecture the AgentEval CI runner uses.

  In-memory job store (and its limits):
    what: >
      A plain dict mapping job_id → status + result. Background task writes to it
      when done. Poll endpoint reads from it.
    why: >
      This works for a single-process service. It breaks the moment you run more
      than one worker (two processes don't share memory). Understanding this limit
      is why production systems use Redis or a DB for job state. You'll encounter
      this in the Docker stage when you scale to multiple workers.

  Custom exception handlers:
    what: >
      @app.exception_handler(SomeExceptionType) — a decorator that registers a function
      to handle a specific exception type anywhere in the app. Returns a JSONResponse.
    why: >
      Without this, an unhandled exception produces a raw 500 with whatever Python
      decides to stringify. With it, every known failure mode returns a structured
      JSON body with consistent fields (error, detail, code). Callers can reliably
      parse errors.

  Custom exception classes:
    what: >
      Python exceptions with extra fields (error_code, detail) that your exception
      handlers read and include in the response.
    why: >
      This is the pattern that separates a production API from a tutorial.
      raise JobNotFoundError(job_id=42) → {"error": "JOB_NOT_FOUND", "detail": "Job 42 not found"}
      is a contract. raise HTTPException(404, "not found") is ad hoc.

  Request validation error customization:
    what: >
      Override FastAPI's default 422 handler to return your own error shape.
    why: >
      FastAPI's default 422 body is verbose and nested. Production APIs typically
      flatten it. Customizing it teaches you where the default handler lives and
      how to replace it.
```

---

## Setup

```yaml
dependencies:
  - fastapi
  - uvicorn[standard]

install: "pip install fastapi uvicorn[standard]"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define custom exception classes"
    detail: >
      class AppError(Exception):
          """Base class for all application errors."""
          def __init__(self, error_code: str, detail: str, status_code: int = 400):
              self.error_code = error_code
              self.detail = detail
              self.status_code = status_code

      class JobNotFoundError(AppError):
          def __init__(self, job_id: str):
              super().__init__(
                  error_code="JOB_NOT_FOUND",
                  detail=f"Job '{job_id}' does not exist",
                  status_code=404,
              )

      class InvalidJobInputError(AppError):
          def __init__(self, reason: str):
              super().__init__(
                  error_code="INVALID_INPUT",
                  detail=reason,
                  status_code=422,
              )

  2:
    task: "Define the error response model and register exception handler"
    detail: >
      from fastapi import FastAPI, Request
      from fastapi.responses import JSONResponse
      from pydantic import BaseModel

      class ErrorResponse(BaseModel):
          error: str
          detail: str

      app = FastAPI()

      @app.exception_handler(AppError)
      async def app_error_handler(request: Request, exc: AppError) -> JSONResponse:
          return JSONResponse(
              status_code=exc.status_code,
              content={"error": exc.error_code, "detail": exc.detail},
          )

  3:
    task: "Define the job state store and models"
    detail: >
      import uuid
      from enum import Enum

      class JobStatus(str, Enum):
          pending = "pending"
          running = "running"
          done = "done"
          failed = "failed"

      class Job(BaseModel):
          id: str
          status: JobStatus
          prompt: str
          result: str | None = None
          error: str | None = None

      jobs: dict[str, Job] = {}

  4:
    task: "Build POST /jobs — accept and enqueue"
    detail: >
      from fastapi import BackgroundTasks

      class JobRequest(BaseModel):
          prompt: str

      class JobAccepted(BaseModel):
          job_id: str
          status: str = "pending"

      def run_job(job_id: str, prompt: str) -> None:
          """Runs in background after response is sent."""
          import time
          jobs[job_id].status = JobStatus.running
          time.sleep(2)   # simulate slow work
          jobs[job_id].status = JobStatus.done
          jobs[job_id].result = f"Processed: {prompt[:50]}"

      @app.post("/jobs", response_model=JobAccepted, status_code=202)
      async def create_job(
          body: JobRequest,
          background_tasks: BackgroundTasks,
      ) -> JobAccepted:
          if not body.prompt.strip():
              raise InvalidJobInputError("Prompt cannot be empty")

          job_id = str(uuid.uuid4())
          jobs[job_id] = Job(id=job_id, status=JobStatus.pending, prompt=body.prompt)
          background_tasks.add_task(run_job, job_id, body.prompt)
          return JobAccepted(job_id=job_id)

  5:
    task: "Build GET /jobs/{job_id} — poll for status"
    detail: >
      @app.get("/jobs/{job_id}", response_model=Job)
      async def get_job(job_id: str) -> Job:
          if job_id not in jobs:
              raise JobNotFoundError(job_id)
          return jobs[job_id]

  6:
    task: "Add a catch-all handler for unexpected exceptions"
    detail: >
      @app.exception_handler(Exception)
      async def generic_error_handler(request: Request, exc: Exception) -> JSONResponse:
          return JSONResponse(
              status_code=500,
              content={"error": "INTERNAL_ERROR", "detail": "An unexpected error occurred"},
          )

      Now deliberately raise a plain RuntimeError inside a route and confirm it
      returns {"error": "INTERNAL_ERROR"} instead of a raw 500 traceback.

  7:
    task: "Override FastAPI's default 422 validation error handler"
    detail: >
      from fastapi.exceptions import RequestValidationError

      @app.exception_handler(RequestValidationError)
      async def validation_error_handler(request: Request, exc: RequestValidationError):
          first_error = exc.errors()[0]
          return JSONResponse(
              status_code=422,
              content={
                  "error": "VALIDATION_ERROR",
                  "detail": f"{first_error['loc'][-1]}: {first_error['msg']}",
              },
          )

      Test by sending a POST /jobs with no body. Compare the response
      to what FastAPI returned before you added this handler.
```

---

## Test the Background Pattern

```bash
# submit a job
curl -X POST http://localhost:8000/jobs \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Write a haiku about Python"}'
# → {"job_id": "abc-123", "status": "pending"}

# immediately poll (should be "running" or "pending")
curl http://localhost:8000/jobs/abc-123
# → {"id": "abc-123", "status": "running", ...}

# poll again after 3 seconds
curl http://localhost:8000/jobs/abc-123
# → {"id": "abc-123", "status": "done", "result": "Processed: ..."}

# test error handling
curl http://localhost:8000/jobs/nonexistent-id
# → {"error": "JOB_NOT_FOUND", "detail": "Job 'nonexistent-id' does not exist"}
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Raise a JobNotFoundError inside run_job (the background function).
    Does the exception handler catch it? What happens to unhandled exceptions in background tasks?
    (Hint: they don't propagate to the client — this is a real production gotcha.)
  - Make run_job update a job that doesn't exist in the dict mid-run.
    What error do you get and where does it appear?
  - Submit 5 jobs simultaneously. Do they all run? Do they run in sequence or parallel?
    (BackgroundTasks are sequential within one request but concurrent across requests.)
```

---

## Done When

```yaml
done_when:
  - POST /jobs returns 202 immediately while the job runs in background
  - GET /jobs/{id} correctly shows pending → running → done progression
  - JobNotFoundError, InvalidJobInputError, and plain RuntimeError all return
    structured JSON with the right status codes
  - You've overridden the 422 handler and understand what it changed
  - You understand the key limitation: background tasks die with the process —
    there's no persistence across restarts (and can explain why Redis fixes this)
```

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between BackgroundTasks and asyncio.create_task()?
    When would you use each in FastAPI?
  - What happens to running background tasks when the server shuts down?
  - Can a background task itself raise a BackgroundTasks and queue further work?
  - How would you add a job timeout — cancel a job if it runs longer than 30 seconds?
```
