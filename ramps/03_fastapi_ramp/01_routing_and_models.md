# Stage 01 — Routing and Pydantic Models

**Status:** not started
**Estimated time:** 1 day
**Type:** standalone — fresh project, nothing carries over
**Depends on:** basic Python — nothing else

The foundation. FastAPI's entire value comes from two things: declaring routes as Python functions, and using Pydantic models to define what goes in and what comes out. Everything else in the framework is built on top of these two primitives. This stage makes both feel natural.

---

## What You're Building

A self-contained `main.py` — a toy "task manager" API with five endpoints:

```
POST   /tasks          create a task
GET    /tasks          list all tasks
GET    /tasks/{id}     get one task by id
PATCH  /tasks/{id}     update task status
DELETE /tasks/{id}     delete a task
```

No database. Tasks live in a plain Python dict in memory. The point is the routing and model patterns, not persistence.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Route decorators:
    what: >
      @app.get("/path"), @app.post("/path"), etc. Each decorator registers a function
      as the handler for that HTTP method + path. The function's return value becomes
      the response body — FastAPI serializes it to JSON automatically.
    why: >
      This is the entire FastAPI programming model. Every endpoint in your OSS projects
      is one of these decorators. Understanding what the decorator does (and doesn't do)
      means you're never confused by why a handler isn't being called.

  Path parameters:
    what: >
      /tasks/{id} — the curly brace part is extracted and passed as a function argument.
      FastAPI validates its type automatically based on the type hint.
    why: >
      Every REST API uses these. GET /tasks/42 → task_id: int = 42 in your function.
      The type validation is free — pass a string where an int is expected, FastAPI
      returns a 422 with a clear error before your code runs.

  Pydantic request models:
    what: >
      A class inheriting from pydantic.BaseModel with typed fields. When you declare
      a function parameter with this type, FastAPI reads the request body, validates it
      against the schema, and hands you a typed Python object. Invalid requests get
      a 422 automatically.
    why: >
      This is the pattern every AI backend uses for structured input. Your agent
      endpoints will have request models like AgentRequest(prompt: str, thread_id: str).
      Building the habit here means you never write manual request parsing again.

  Pydantic response models:
    what: >
      The response_model= parameter on a route decorator. FastAPI will validate and
      serialize the return value to match this schema — extra fields are stripped,
      missing required fields raise an error.
    why: >
      Response models are your API contract. They document what callers can expect
      and prevent you from accidentally leaking internal fields (e.g., passwords,
      internal IDs) in responses.

  HTTPException:
    what: >
      raise HTTPException(status_code=404, detail="Task not found")
      FastAPI catches this and returns the appropriate HTTP error response.
    why: >
      Clean error handling without try/except in every route. You raise, FastAPI handles.
      This is the idiomatic pattern — not returning error dicts manually.

  Automatic docs:
    what: FastAPI generates interactive API docs at /docs (Swagger) and /redoc automatically.
    why: >
      Run your service, go to http://localhost:8000/docs, and you can test every endpoint
      in the browser. This is the fastest feedback loop for API development.
```

---

## Setup

```yaml
dependencies:
  - fastapi
  - uvicorn[standard]

install: "pip install fastapi uvicorn[standard]"
run: "uvicorn main:app --reload"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define your Pydantic models"
    detail: >
      from pydantic import BaseModel
      from enum import Enum

      class TaskStatus(str, Enum):
          pending = "pending"
          done = "done"

      class TaskCreate(BaseModel):
          title: str
          description: str = ""     # optional, defaults to empty string

      class TaskUpdate(BaseModel):
          status: TaskStatus

      class Task(BaseModel):
          id: int
          title: str
          description: str
          status: TaskStatus = TaskStatus.pending

  2:
    task: "Create the app and in-memory store"
    detail: >
      from fastapi import FastAPI, HTTPException

      app = FastAPI(title="Task Manager", version="0.1.0")

      # in-memory store
      tasks: dict[int, Task] = {}
      next_id: int = 1

  3:
    task: "Implement POST /tasks"
    detail: >
      @app.post("/tasks", response_model=Task, status_code=201)
      def create_task(body: TaskCreate) -> Task:
          global next_id
          task = Task(id=next_id, title=body.title, description=body.description)
          tasks[next_id] = task
          next_id += 1
          return task

  4:
    task: "Implement GET /tasks and GET /tasks/{task_id}"
    detail: >
      @app.get("/tasks", response_model=list[Task])
      def list_tasks() -> list[Task]:
          return list(tasks.values())

      @app.get("/tasks/{task_id}", response_model=Task)
      def get_task(task_id: int) -> Task:
          if task_id not in tasks:
              raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
          return tasks[task_id]

  5:
    task: "Implement PATCH /tasks/{task_id} and DELETE /tasks/{task_id}"
    detail: >
      @app.patch("/tasks/{task_id}", response_model=Task)
      def update_task(task_id: int, body: TaskUpdate) -> Task:
          if task_id not in tasks:
              raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
          tasks[task_id].status = body.status
          return tasks[task_id]

      @app.delete("/tasks/{task_id}", status_code=204)
      def delete_task(task_id: int) -> None:
          if task_id not in tasks:
              raise HTTPException(status_code=404, detail=f"Task {task_id} not found")
          del tasks[task_id]

  6:
    task: "Run it and explore /docs"
    detail: >
      uvicorn main:app --reload

      Go to http://localhost:8000/docs
      Create a task, list all tasks, update its status, delete it.
      Do all of this from the Swagger UI — no curl needed yet.

  7:
    task: "Add query parameters to GET /tasks"
    detail: >
      Add optional filtering by status:

      @app.get("/tasks", response_model=list[Task])
      def list_tasks(status: TaskStatus | None = None) -> list[Task]:
          result = list(tasks.values())
          if status:
              result = [t for t in result if t.status == status]
          return result

      Query params are just function arguments with defaults. No decorator annotation needed.
      Test: GET /tasks?status=pending
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - POST /tasks with a missing required field (title). Read the 422 response body carefully.
    This is FastAPI's validation error format — you will see it in production.
  - Return a plain dict from a route instead of a Task model. Does it still work?
    What does the response_model= parameter actually enforce?
  - Raise HTTPException with status_code=200. Is that valid? What does the client see?
  - Add a field to Task that isn't in the response_model. Does it appear in the response?
    This demonstrates what response_model filtering does.
```

---

## curl Reference

```bash
# create
curl -X POST http://localhost:8000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn FastAPI", "description": "Stage 01"}'

# list
curl http://localhost:8000/tasks

# get one
curl http://localhost:8000/tasks/1

# update
curl -X PATCH http://localhost:8000/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{"status": "done"}'

# delete
curl -X DELETE http://localhost:8000/tasks/1
```

---

## Done When

```yaml
done_when:
  - All five endpoints work and return correct responses
  - You've tested validation errors (422) and not-found errors (404) deliberately
  - You can explain what response_model= does and why it matters
  - You can add a new endpoint from memory — route decorator, Pydantic model, HTTPException
  - You've broken it in at least 2 ways and understand why each one fails
```

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between status_code on the decorator vs inside the function?
  - Can a single endpoint handle multiple HTTP methods? How?
  - What happens if you declare two routes with the same path and method?
  - What is the difference between BaseModel and dataclasses for FastAPI? When would you use each?
```
