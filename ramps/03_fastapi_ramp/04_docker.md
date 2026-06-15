# Stage 04 — Docker and Compose

**Status:** not started
**Estimated time:** 1–2 days
**Type:** standalone — containerize a FastAPI service from scratch
**Depends on:** Stage 01–03 concepts (you have a working FastAPI service to containerize)

A service that only runs on your machine isn't a service. Docker is how you ship it — the same image runs on your laptop, a CI server, and a cloud VM without any "it worked on my machine" issues. This stage containerizes a FastAPI service and wires it together with a second service (Redis) using Compose. After this, every OSS project you build is deployable.

---

## What You're Building

Take the Stage 03 job queue service and make it production-closer:
1. A `Dockerfile` that builds a lean Python image for the FastAPI service
2. Replace the in-memory job dict with Redis (so job state survives restarts)
3. A `docker-compose.yml` that runs both the API and Redis together
4. A GitHub Actions workflow that builds and tests the image on every push

One `docker compose up` → both services running, jobs persisting across restarts.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Dockerfile:
    what: >
      A text file of instructions that build a Docker image layer by layer.
      FROM (base image) → WORKDIR → COPY → RUN (install deps) → CMD (start command).
    why: >
      Every OSS project you ship needs a Dockerfile. The capstone (Stage 05) deploys
      to Railway using one. Every JD that mentions Docker expects you to own this file.

  Multi-stage builds:
    what: >
      Two FROM instructions in one Dockerfile. The first stage (builder) installs
      dependencies. The second stage (runtime) copies only what's needed from the builder.
      The final image doesn't contain pip, compilers, or build artifacts.
    why: >
      A naive Python Docker image is 800MB+. A multi-stage image is 100–200MB.
      Smaller images pull faster, start faster, and have fewer security vulnerabilities.
      This is the standard pattern — single-stage Dockerfiles are a red flag in code review.

  .dockerignore:
    what: Like .gitignore but for Docker. Files listed here are excluded from the build context.
    why: >
      Without it, Docker sends your entire project directory (including .git, __pycache__,
      .env, node_modules) to the daemon on every build. With it, builds are fast and
      your secrets don't accidentally end up in the image.

  Docker Compose:
    what: >
      A YAML file (docker-compose.yml) that defines multiple services, their images,
      environment variables, ports, and how they connect to each other.
      docker compose up starts all of them.
    why: >
      The agentic ramp's Jaeger dev stack uses Compose. AgentTrace's README says
      "docker compose up gives you a working trace viewer in two minutes." Every
      multi-service local dev environment uses this. You need to be able to write it
      and read it without hesitation.

  Service networking in Compose:
    what: >
      Services in the same Compose file can reach each other by service name.
      If your Redis service is named "redis", your FastAPI service connects to
      redis://redis:6379 — not redis://localhost:6379.
    why: >
      This trips everyone up the first time. localhost inside a container refers to
      that container, not your host machine. Service names are the DNS within Compose's
      virtual network.

  Environment variables in containers:
    what: >
      Pass secrets and config via environment: in docker-compose.yml or --env-file.
      Never bake secrets into the image.
    why: >
      The .env file on your machine doesn't exist inside the container. You must
      explicitly pass values in. This is also the production pattern — secrets come
      from the environment, not from files committed to the repo.
```

---

## Setup

```yaml
requirements:
  - Docker Desktop installed and running
  - The Stage 03 job queue service (or a fresh FastAPI service with one endpoint)

new_dependencies:
  - redis   # Python client: pip install redis
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write the .dockerignore"
    detail: >
      Create .dockerignore in the project root:

      __pycache__
      *.pyc
      *.pyo
      .env
      .git
      .gitignore
      .venv
      venv
      dist
      build
      *.egg-info

      Do this before the Dockerfile — it affects what Docker can see.

  2:
    task: "Write the Dockerfile (multi-stage)"
    detail: >
      # ── builder stage ──────────────────────────────────────────────
      FROM python:3.12-slim AS builder

      WORKDIR /app
      COPY requirements.txt .
      RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

      # ── runtime stage ──────────────────────────────────────────────
      FROM python:3.12-slim AS runtime

      WORKDIR /app
      COPY --from=builder /install /usr/local
      COPY . .

      EXPOSE 8000
      CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

      Key details:
      - python:3.12-slim is the minimal official Python image — no compilers, no extras
      - --host 0.0.0.0 is required — without it the server binds to localhost inside the
        container, which is unreachable from outside
      - --prefix=/install copies deps to a separate path, making the COPY clean

  3:
    task: "Replace in-memory job dict with Redis"
    detail: >
      import redis
      import json

      redis_client = redis.Redis(
          host=os.getenv("REDIS_HOST", "localhost"),
          port=int(os.getenv("REDIS_PORT", 6379)),
          decode_responses=True,
      )

      def save_job(job: Job) -> None:
          redis_client.set(f"job:{job.id}", job.model_dump_json(), ex=3600)

      def load_job(job_id: str) -> Job | None:
          data = redis_client.get(f"job:{job_id}")
          if not data:
              return None
          return Job.model_validate_json(data)

      Update create_job and get_job to use save_job/load_job instead of the dict.
      Update run_job to load the job from Redis, update it, and save it back.

  4:
    task: "Write docker-compose.yml"
    detail: >
      services:
        api:
          build: .
          ports:
            - "8000:8000"
          environment:
            - REDIS_HOST=redis
            - REDIS_PORT=6379
          depends_on:
            - redis

        redis:
          image: redis:7-alpine
          ports:
            - "6379:6379"

      Note REDIS_HOST=redis — the service name, not localhost.

  5:
    task: "Build and run"
    detail: >
      docker compose up --build

      Test:
        curl -X POST http://localhost:8000/jobs \
          -H "Content-Type: application/json" \
          -d '{"prompt": "Hello from Docker"}'

      Then restart the compose stack (docker compose restart api) and poll the job ID again.
      The job should still be there — it survived the restart because Redis persisted it.
      Compare this to Stage 03 where the in-memory dict was wiped on restart.

  6:
    task: "Write a GitHub Actions workflow"
    detail: >
      Create .github/workflows/docker.yml:

      name: Docker Build and Test

      on: [push, pull_request]

      jobs:
        build:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4

            - name: Build Docker image
              run: docker build -t fastapi-jobs:test .

            - name: Start services
              run: docker compose up -d

            - name: Wait for API
              run: sleep 5

            - name: Test health endpoint
              run: |
                curl --fail http://localhost:8000/docs || exit 1

            - name: Teardown
              run: docker compose down

      Add GET /health → {"status": "ok"} to your FastAPI app first.
      The CI workflow hits this endpoint to confirm the container started correctly.
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Remove --host 0.0.0.0 from the CMD. Run docker compose up.
    Try to curl localhost:8000. What happens and why?
  - Change REDIS_HOST=redis to REDIS_HOST=localhost in docker-compose.yml.
    Try to submit a job. What error do you get inside the container logs?
    (docker compose logs api to see them)
  - Add a secret (OPENAI_API_KEY=hardcoded-value) directly in the Dockerfile ENV instruction.
    Build the image. Run docker inspect <image-id>. Can you see the key?
    This is why you never put secrets in Dockerfiles.
  - Remove depends_on from the api service. Start compose. What race condition can occur?
```

---

## Done When

```yaml
done_when:
  - docker compose up starts both services and the API is reachable
  - Jobs survive an API container restart (Redis persistence confirmed)
  - The GitHub Actions workflow builds and passes
  - You can explain multi-stage builds and why they matter
  - You can write a docker-compose.yml with two services and correct networking from memory
  - You know never to put secrets in a Dockerfile and why
```

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between EXPOSE in a Dockerfile and ports: in docker-compose.yml?
  - How do you pass secrets into a container without putting them in docker-compose.yml
    (which might be committed)? Look up Docker secrets and env_file.
  - What does depends_on actually guarantee? Does it wait for the service to be ready,
    or just for the container to start?
  - How would you run multiple API workers (e.g., 4 uvicorn processes) in Compose?
    What breaks with the in-memory store when you do?
```
