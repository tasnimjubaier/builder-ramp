# Stage 01 — Dockerfile

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive start — this project grows through all 4 stages
**Depends on:** Docker Desktop installed, a working FastAPI service (from FastAPI ramp) or a minimal echo API

A Dockerfile is a script that builds a reproducible environment. Every instruction creates a layer. The result — a Docker image — is a snapshot of your application plus all its dependencies, guaranteed to run the same everywhere. This stage teaches you to write one correctly, which means understanding layers, caching, multi-stage builds, and why images get bloated.

---

## What You're Building

A Dockerfile for a FastAPI service. Two versions: a naive single-stage version (to understand what's wrong with it), then a production-quality multi-stage version. By the end, `docker run` starts the service and it responds to requests.

If you haven't done the FastAPI ramp yet, use this minimal service as your base:

```python
# main.py
from fastapi import FastAPI
app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/echo")
def echo(body: dict):
    return {"echo": body}
```

```
# requirements.txt
fastapi
uvicorn[standard]
```

---

## Concepts This Stage Teaches

```yaml
concepts:
  Docker image vs container:
    what: >
      Image = the blueprint. A read-only snapshot of a filesystem + metadata.
      Container = a running instance of an image. Like class vs instance.
      docker build → creates an image. docker run → creates a container from it.
    why: >
      People use "image" and "container" interchangeably and get confused.
      The distinction is important: you ship images, you run containers.
      The same image can run as 100 simultaneous containers.

  Layers and caching:
    what: >
      Each Dockerfile instruction (FROM, RUN, COPY) creates a layer.
      Docker caches layers — if a layer's instruction and its inputs haven't changed,
      Docker reuses the cached layer instead of re-running it.
      Once any layer changes, all subsequent layers are re-built.
    why: >
      Layer order is a performance decision. COPY requirements.txt + RUN pip install
      before COPY . . means dependency installation is cached unless requirements.txt
      changes. Flip the order and every code change re-installs all dependencies.
      Slow builds are almost always a layer ordering mistake.

  Multi-stage builds:
    what: >
      Two FROM instructions. Stage 1 (builder) installs build tools and dependencies.
      Stage 2 (runtime) copies only what's needed from stage 1 — no compilers, no pip,
      no build artifacts. The final image only contains the runtime.
    why: >
      A naive Python image is 800MB–1GB. A multi-stage image is 100–200MB.
      Smaller = faster to pull, faster to start, less attack surface.
      This is the standard for production images — single-stage is a red flag.

  .dockerignore:
    what: >
      Like .gitignore. Files listed here are excluded from the build context —
      the set of files Docker sends to the daemon when building.
    why: >
      Without it: every build sends .git/, .venv/, __pycache__, and all test fixtures
      to the daemon. Build context of 500MB instead of 10KB. Builds are slow.
      With it: fast builds, and your .env secrets don't accidentally end up in the image.

  CMD vs ENTRYPOINT:
    what: >
      CMD: default command run when the container starts. Can be overridden with docker run args.
      ENTRYPOINT: the executable. CMD becomes arguments to ENTRYPOINT.
    why: >
      For a web service, CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
      is the right choice — it's the default but you can override it for debugging.
      Using ENTRYPOINT + CMD together is for tools where the executable is fixed.

  Port binding:
    what: >
      EXPOSE 8000 in the Dockerfile documents which port the app listens on — informational only.
      -p 8000:8000 in docker run actually binds host port 8000 to container port 8000.
    why: >
      EXPOSE alone doesn't make the port reachable. You need -p (or ports: in Compose).
      This trips everyone up once.

  --host 0.0.0.0:
    what: >
      Without this, uvicorn binds to 127.0.0.1 (localhost inside the container).
      Requests from outside the container (your browser, curl) can't reach it.
      0.0.0.0 binds to all network interfaces, including the one Docker exposes.
    why: >
      "It works locally but not in Docker" is almost always this problem.
      Always use --host 0.0.0.0 in containerized services.
```

---

## Setup

```yaml
requirements:
  - Docker Desktop installed and running
  - docker --version in terminal returns something
  - A FastAPI service with requirements.txt (or use the minimal echo API above)
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Write .dockerignore first"
    detail: >
      Create .dockerignore in the project root before touching the Dockerfile.
      This is a discipline habit — always write it first.

      .venv/
      __pycache__/
      *.pyc
      *.pyo
      .env
      .git/
      .gitignore
      tests/
      dist/
      build/
      *.egg-info/
      .pytest_cache/
      .coverage

  2:
    task: "Write a naive single-stage Dockerfile"
    detail: >
      # Dockerfile.naive — write this first, understand what's wrong with it

      FROM python:3.12

      WORKDIR /app
      COPY . .
      RUN pip install -r requirements.txt

      EXPOSE 8000
      CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

      Build and measure:
        docker build -f Dockerfile.naive -t myapp:naive .
        docker images myapp   # check the size — will be ~1GB

      Problems with this:
        1. python:3.12 (full) is 1GB+. python:3.12-slim is 130MB.
        2. COPY . . before pip install — code changes bust the pip cache.
        3. No separation between build tools and runtime.
        4. pip installs into the image layer alongside source code.

  3:
    task: "Write the production multi-stage Dockerfile"
    detail: >
      # Dockerfile

      # ── Stage 1: builder ──────────────────────────────────────────
      FROM python:3.12-slim AS builder

      WORKDIR /app

      # Install dependencies into /install prefix (separate from system Python)
      COPY requirements.txt .
      RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

      # ── Stage 2: runtime ─────────────────────────────────────────
      FROM python:3.12-slim AS runtime

      WORKDIR /app

      # Copy installed packages from builder
      COPY --from=builder /install /usr/local

      # Copy application code (after deps — better layer caching)
      COPY . .

      EXPOSE 8000
      CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

  4:
    task: "Build and compare sizes"
    detail: >
      docker build -t myapp:prod .
      docker images myapp

      Compare naive vs prod image size. Should be ~5–8x smaller.

      Run the production image:
        docker run -p 8000:8000 myapp:prod

      In another terminal:
        curl http://localhost:8000/health
        # → {"status": "ok"}

  5:
    task: "Pass environment variables into the container"
    detail: >
      Add a route that reads an env var:
        import os
        @app.get("/env")
        def show_env():
            return {"app_env": os.getenv("APP_ENV", "not set")}

      Run with -e flag:
        docker run -p 8000:8000 -e APP_ENV=production myapp:prod
        curl http://localhost:8000/env
        # → {"app_env": "production"}

      Run with --env-file flag:
        docker run -p 8000:8000 --env-file .env myapp:prod

      Note: .env on your machine is NOT automatically inside the container.
      You must explicitly pass it with --env-file or -e KEY=VALUE.

  6:
    task: "Inspect the image"
    detail: >
      # See what's in the image
      docker run --rm -it myapp:prod /bin/bash
      # Inside: ls, which python, python --version, pip list

      # See the layers
      docker history myapp:prod

      # See full image metadata
      docker inspect myapp:prod | head -100

      Confirm: no .venv/, no __pycache__, no .env file inside the image.
      If any of these appear, fix .dockerignore.
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Remove --host 0.0.0.0 from CMD. Build and run. Try curl http://localhost:8000/health.
    What happens? Why?

  - Swap the layer order: COPY . . before COPY requirements.txt + pip install.
    Make a small change to main.py. Rebuild. Is it slower?
    Restore the correct order. Rebuild after the same change. Is it faster?
    This demonstrates layer caching concretely.

  - Add a secret to the Dockerfile: ENV API_KEY=mysecret
    Build the image. Run: docker inspect myapp:prod | grep API_KEY
    The secret is visible in the image metadata. This is why you never put secrets in Dockerfiles.

  - Try to run two containers binding to the same host port:
    docker run -p 8000:8000 myapp:prod &
    docker run -p 8000:8000 myapp:prod
    What error do you get?
```

---

## Done When

```yaml
done_when:
  - docker build -t myapp:prod . succeeds with no errors
  - curl http://localhost:8000/health returns {"status": "ok"}
  - The production image is significantly smaller than the naive image
  - No .env or .venv appears inside the running container
  - You can explain multi-stage builds in one paragraph without looking at notes
  - You understand why --host 0.0.0.0 is required
  - You can explain layer caching and why order matters
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between RUN, CMD, and ENTRYPOINT?
    Can you use all three in the same Dockerfile? What happens?
  - What does --no-cache-dir do on the pip install command?
    Why does it matter for Docker images?
  - What is a distroless image? When would you use it instead of python:3.12-slim?
  - How do you reduce image size further if your app has compiled C extensions
    (like numpy or psycopg2)?
```
