# Stage 02 — Docker Compose

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — extend Stage 01 service with dependencies
**Depends on:** Stage 01 (Dockerfile working, image builds)

A real service doesn't run alone — it needs a database, a cache, a message queue. Running these manually with docker run commands for each one is fragile and doesn't represent how teams work. Docker Compose defines all services in one file, wires them together, and starts them with one command. This stage teaches Compose by adding Redis to the Stage 01 service and making them talk to each other.

---

## What You're Building

Extend the Stage 01 FastAPI service to use Redis for job state storage (same as FastAPI ramp Stage 04). Write a `docker-compose.yml` that runs both services together. After `docker compose up`, the API is reachable, Redis is running, and job state survives container restarts.

---

## Concepts This Stage Teaches

```yaml
concepts:
  docker-compose.yml structure:
    what: >
      A YAML file with a top-level services: key. Each service has:
        image: (use a pre-built image) or build: (build from a Dockerfile)
        ports: (bind host:container ports)
        environment: (set env vars)
        depends_on: (start order dependency)
        volumes: (persist data)
    why: >
      This is the standard local dev environment for any multi-service stack.
      The RAG ramp's Compose file has API + Qdrant + Redis.
      The AgentTrace dev stack has API + Jaeger. Every team uses some version of this.

  Container networking:
    what: >
      Services in the same Compose file are on a shared virtual network.
      They can reach each other by service name — not localhost, not an IP address.
      If your Redis service is named "redis", connect to redis://redis:6379.
    why: >
      This is the most common Compose mistake. localhost inside a container refers
      to that container's loopback — not the host machine, not other containers.
      Service name DNS is how containers find each other.

  depends_on:
    what: >
      depends_on: [redis] — the api service won't start until redis container is created.
    why: >
      Without it, api might start before redis is ready and fail to connect.
      Note: depends_on only waits for the container to start, not for the service
      inside it to be ready. For that you need a health check.

  healthcheck:
    what: >
      A command Compose runs periodically inside a container.
      If it fails, the container is marked unhealthy.
      depends_on: condition: service_healthy waits for the health check to pass.
    why: >
      Redis is ready almost immediately. But databases (Postgres, Qdrant) take a few
      seconds to initialize. Without a health check, api starts before the DB is ready
      and crashes on the first connection attempt.

  Named volumes:
    what: >
      volumes:
        redis_data:
      service:
        volumes:
          - redis_data:/data
      Named volumes are managed by Docker and persist across container restarts.
    why: >
      Without a volume, Redis data is stored inside the container's writable layer.
      When you run docker compose down, the container is deleted and all data is lost.
      With a named volume, data survives down/up cycles.

  docker compose commands:
    what: >
      docker compose up -d          — start all services in background
      docker compose down           — stop and remove containers (keeps volumes)
      docker compose down -v        — stop, remove containers AND volumes
      docker compose logs api       — stream logs from the api service
      docker compose ps             — show status of all services
      docker compose exec api bash  — open a shell inside a running container
      docker compose build          — rebuild images without starting
    why: >
      These are the daily commands. Memorize them — you'll type them dozens of times per day.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Update the service to use Redis"
    detail: >
      If coming from FastAPI ramp Stage 04, you already have Redis integration.
      If not, add minimal Redis support:

      pip install redis
      Add "redis" to requirements.txt

      In main.py, replace the in-memory job dict with Redis:

        import redis
        import os

        redis_client = redis.Redis(
            host=os.getenv("REDIS_HOST", "localhost"),
            port=int(os.getenv("REDIS_PORT", 6379)),
            decode_responses=True,
        )

        # Replace: jobs: dict = {}
        # With: redis_client.set(f"job:{job_id}", json.dumps(job_data))
        #        redis_client.get(f"job:{job_id}")

      Verify it still works locally (with Redis running via docker run redis).

  2:
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
            redis:
              condition: service_healthy
          restart: on-failure

        redis:
          image: redis:7-alpine
          ports:
            - "6379:6379"
          volumes:
            - redis_data:/data
          healthcheck:
            test: ["CMD", "redis-cli", "ping"]
            interval: 5s
            timeout: 3s
            retries: 5

      volumes:
        redis_data:

      Key things to notice:
        - REDIS_HOST=redis — the service name, not localhost
        - depends_on with condition: service_healthy — waits for Redis ping to succeed
        - restart: on-failure — restarts the api if it crashes on startup

  3:
    task: "Start and test"
    detail: >
      docker compose up --build

      Wait until you see: "Application startup complete." from uvicorn.

      In another terminal:
        # Health check
        curl http://localhost:8000/health

        # Submit a job
        curl -X POST http://localhost:8000/jobs \
          -H "Content-Type: application/json" \
          -d '{"prompt": "Hello from Compose"}'
        # → {"job_id": "abc-123", "status": "pending"}

        # Poll the job
        curl http://localhost:8000/jobs/abc-123

  4:
    task: "Prove data survives a restart"
    detail: >
      Submit a job. Note its job_id.

      Restart only the api container (not Redis):
        docker compose restart api

      Poll the job_id again. The data is still there — Redis kept it.

      Now stop everything and bring it back up:
        docker compose down
        docker compose up -d

      Poll the job_id again. Still there — the named volume persisted Redis data.

      Then destroy everything including volumes:
        docker compose down -v

      Bring it back up and poll. Now the job is gone — the volume was deleted.
      This is the difference between down and down -v.

  5:
    task: "Read the logs"
    detail: >
      docker compose logs -f api     # follow api logs
      docker compose logs redis      # one-time redis logs

      Submit a job and watch the logs as it processes.
      Find the request log line uvicorn emits — it shows method, path, status code.

      docker compose logs --since 5m  # last 5 minutes of all logs

  6:
    task: "Open a shell inside a running container"
    detail: >
      docker compose exec api bash

      Inside the container:
        ls                     # see the project files
        env | grep REDIS       # see the env vars
        python -c "import redis; print('redis ok')"
        redis-cli -h redis ping   # ping Redis from inside the api container

      exit to leave.

      This is the most useful debugging tool — when something doesn't work,
      get inside the container and poke around.

  7:
    task: "Override config for different environments"
    detail: >
      Create docker-compose.override.yml (Compose merges it automatically in development):

      services:
        api:
          environment:
            - APP_ENV=development
            - LOG_LEVEL=DEBUG
          volumes:
            - .:/app   # mount source code — changes take effect without rebuilding

      docker compose up --build   # override is applied automatically

      The volume mount (- .:/app) means you can edit main.py and the running container
      sees the change immediately (if uvicorn --reload is in CMD).

      Create docker-compose.prod.yml for production overrides:
      docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

## Multi-Service Debugging Exercises

```yaml
exercises:
  - task: "Change REDIS_HOST=redis to REDIS_HOST=localhost in docker-compose.yml"
    detail: >
      Bring the stack up. Submit a job. What error appears in the api logs?
      (docker compose logs api)
      Why does it fail? Fix it back to REDIS_HOST=redis.

  - task: "Remove the healthcheck from redis and set condition: service_healthy"
    detail: >
      Bring the stack up. Does it start? What error?
      This shows what happens when you use service_healthy without a healthcheck defined.

  - task: "Add a third service"
    detail: >
      Add a simple adminer (DB admin UI) or redisinsight service to docker-compose.yml:

        redisinsight:
          image: redis/redisinsight:latest
          ports:
            - "5540:5540"

      docker compose up -d
      Open http://localhost:5540 — connect to host "redis", port 6379.
      Browse the job keys you've stored.
      This is how you inspect your data without writing code.
```

---

## Done When

```yaml
done_when:
  - docker compose up starts both services and the API works
  - Jobs submitted to the API are stored in Redis and retrievable
  - Job data survives docker compose restart api
  - Job data is wiped by docker compose down -v (you understand why)
  - You can read live logs with docker compose logs -f
  - You can open a shell inside the running api container
  - You can explain container networking — why service name, not localhost
  - You understand the difference between depends_on and a health check
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between docker compose up and docker compose up --build?
    When do you need --build?
  - What does restart: always do vs restart: on-failure?
    When would you use each in production vs development?
  - How do you run database migrations (e.g., Alembic) in a Compose stack?
    Should they run in the same container as the app or a separate container?
  - What is the difference between a bind mount (- .:/app) and a named volume?
    When would you use each?
```
