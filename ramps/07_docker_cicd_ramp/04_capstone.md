# Stage 04 — Capstone: Full CI/CD Pipeline with Deployment

**Status:** not started
**Estimated time:** 3–4 days
**Type:** additive — applies everything to a real OSS project from another ramp
**Depends on:** Stages 01–03 + one completed OSS project (agent-api or rag-api)

This is the point where Docker and CI/CD stop being a separate thing you learn and become infrastructure you own. Take one of your OSS projects — the FastAPI capstone (`agent-api`) or the RAG capstone (`rag-api`) — and give it a complete production-ready CI/CD pipeline: test → build → push → deploy. One `git push` triggers everything. The deployed URL goes in the README.

---

## What You're Building

A complete pipeline for `agent-api` (or `rag-api`):

```
git push main
    │
    ▼
GitHub Actions CI
    │
    ├── Job 1: test
    │     pytest tests/ → must pass
    │
    ├── Job 2: docker-build (needs: test)
    │     docker build → must succeed
    │     docker compose up → health check → must pass
    │
    └── Job 3: deploy (needs: docker-build, on main only)
          push image to GHCR
          deploy to Railway (or Render)
          smoke test the live URL
```

The deployed service is publicly accessible. You can curl it. The URL goes in the README. This is what "shipped" means.

---

## Choose Your Target

```yaml
recommended: agent-api
  why: >
    Simpler Compose stack (API + Redis).
    Easier to deploy — Railway handles Redis as a managed service.
    The CI/CD pipeline is the full story here, not the application.

alternative: rag-api
  why: >
    More complex but more impressive (3-service stack with Qdrant).
    Harder to deploy — Qdrant needs persistent storage, which Railway supports
    via volumes but requires more config.
    Good choice if you want a more complete portfolio artifact.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Verify the project is production-ready"
    detail: >
      Before adding CI/CD, confirm:
        - Dockerfile builds successfully: docker build -t agent-api:test .
        - docker compose up starts all services
        - curl http://localhost:8000/health returns {"status": "ok"}
        - pytest tests/ passes with at least 5 tests
        - .dockerignore excludes .env, .venv, __pycache__, tests/

      Fix any of these before proceeding. CI will expose every gap.

  2:
    task: "Write the full CI/CD workflow"
    detail: >
      # .github/workflows/pipeline.yml

      name: Pipeline

      on:
        push:
          branches: ["main"]
        pull_request:
          branches: ["main"]

      env:
        REGISTRY: ghcr.io
        IMAGE_NAME: ${{ github.repository }}

      jobs:
        # ── Job 1: Run tests ──────────────────────────────────────
        test:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4

            - uses: actions/setup-python@v5
              with:
                python-version: "3.12"

            - name: Cache dependencies
              uses: actions/cache@v4
              with:
                path: ~/.cache/pip
                key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}

            - name: Install dependencies
              run: pip install -r requirements.txt pytest pytest-asyncio pytest-cov

            - name: Run tests (with mocked dependencies)
              run: pytest tests/ -v
              env:
                OPENAI_API_KEY: "sk-fake-key-for-tests"
                REDIS_HOST: "localhost"

        # ── Job 2: Build and smoke test ───────────────────────────
        build:
          needs: [test]   # only run if tests pass
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4

            - name: Build Docker image
              run: docker build -t ${{ env.IMAGE_NAME }}:test .

            - name: Start Compose stack
              run: docker compose up -d
              env:
                OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}

            - name: Wait for health
              run: |
                for i in $(seq 1 10); do
                  curl -sf http://localhost:8000/health && echo "Ready" && break
                  echo "Waiting... ($i/10)"
                  sleep 3
                done

            - name: Smoke test POST /runs
              run: |
                RESPONSE=$(curl -sf -X POST http://localhost:8000/runs \
                  -H "Content-Type: application/json" \
                  -d '{"prompt": "CI smoke test"}')
                echo "Response: $RESPONSE"
                echo $RESPONSE | python3 -c "import sys,json; d=json.load(sys.stdin); assert 'run_id' in d"

            - name: Teardown
              if: always()   # run even if previous steps failed
              run: docker compose down -v

        # ── Job 3: Push image to GHCR ─────────────────────────────
        push-image:
          needs: [build]
          if: github.ref == 'refs/heads/main'   # only on main branch
          runs-on: ubuntu-latest
          permissions:
            contents: read
            packages: write
          outputs:
            image-tag: ${{ steps.meta.outputs.version }}

          steps:
            - uses: actions/checkout@v4

            - uses: docker/login-action@v3
              with:
                registry: ${{ env.REGISTRY }}
                username: ${{ github.actor }}
                password: ${{ secrets.GITHUB_TOKEN }}

            - id: meta
              uses: docker/metadata-action@v5
              with:
                images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
                tags: |
                  type=sha,prefix=sha-
                  type=raw,value=latest,enable={{is_default_branch}}

            - uses: docker/build-push-action@v5
              with:
                context: .
                push: true
                tags: ${{ steps.meta.outputs.tags }}
                cache-from: type=gha
                cache-to: type=gha,mode=max

        # ── Job 4: Deploy to Railway ──────────────────────────────
        deploy:
          needs: [push-image]
          if: github.ref == 'refs/heads/main'
          runs-on: ubuntu-latest
          environment: production   # requires manual approval in GitHub settings

          steps:
            - name: Deploy to Railway
              run: |
                curl -X POST \
                  "https://backboard.railway.app/graphql/v2" \
                  -H "Authorization: Bearer ${{ secrets.RAILWAY_TOKEN }}" \
                  -H "Content-Type: application/json" \
                  -d '{"query": "mutation { deploymentTrigger(input: { serviceId: \"${{ secrets.RAILWAY_SERVICE_ID }}\" }) { id } }"}'

            - name: Wait for deployment
              run: sleep 30

            - name: Smoke test production
              run: |
                curl --fail ${{ secrets.RAILWAY_URL }}/health
                echo "Production deployment verified"

  3:
    task: "Set up Railway deployment"
    detail: >
      Railway is the simplest way to deploy Docker-based services.

      1. Sign up at railway.app (free tier is sufficient)
      2. Create a new project → Deploy from GitHub repo
      3. Railway auto-detects the Dockerfile and builds it
      4. Add a Redis plugin to the project (managed Redis — no config needed)
      5. Set environment variables in Railway dashboard:
           OPENAI_API_KEY = your-real-key
           REDIS_HOST = (Railway sets this automatically for the Redis plugin)
      6. Note the service URL (e.g., https://agent-api-abc123.railway.app)
      7. Get your Railway API token: Account Settings → API Tokens
      8. Add these to GitHub Secrets:
           RAILWAY_TOKEN = your-railway-token
           RAILWAY_SERVICE_ID = the service ID from Railway dashboard
           RAILWAY_URL = https://your-app.railway.app
           OPENAI_API_KEY = your-openai-key

  4:
    task: "Set up the production environment in GitHub"
    detail: >
      The deploy job uses environment: production.
      This enables:
        - Manual approval gates (optional but professional)
        - Environment-specific secrets
        - Deployment history in the GitHub UI

      Configure it:
      1. Go to repo Settings → Environments → New environment: "production"
      2. Optionally enable "Required reviewers" — someone must approve before deploy
      3. Add environment secrets (RAILWAY_TOKEN, etc.) at the environment level
         instead of repo level for better access control

  5:
    task: "Trigger the full pipeline"
    detail: >
      git push origin main

      Watch the pipeline in the Actions tab:
        - test job runs (green or red)
        - build job runs (only if test passed)
        - push-image job runs (only if build passed, only on main)
        - deploy job runs (only if push-image passed, only on main)

      Wait for the deploy job to complete.
      Curl the Railway URL:
        curl https://your-app.railway.app/health
        # → {"status": "ok"}

      Your service is live on the public internet with a full CI/CD pipeline.

  6:
    task: "Make a change and watch it deploy automatically"
    detail: >
      Add a new endpoint:
        @app.get("/version")
        def get_version():
            return {"version": "0.1.0", "environment": os.getenv("APP_ENV", "unknown")}

      Write a test for it.
      Commit and push.

      Watch:
        - Tests run (including your new test)
        - Image builds with the new endpoint
        - Image pushes to GHCR
        - Railway deploys the new image
        - Curl /version on the live URL — it's there

      This is the full CI/CD cycle. Code → PR → review → merge → auto-deploy.

  7:
    task: "Update the README with the live URL and pipeline badge"
    detail: >
      Add to README.md:

        ## Status
        [![CI](https://github.com/your-username/agent-api/actions/workflows/pipeline.yml/badge.svg)](https://github.com/your-username/agent-api/actions)

        ## Live Demo
        API is deployed at: https://your-app.railway.app

        | Endpoint | Method | Description |
        |----------|--------|-------------|
        | /health  | GET    | Health check |
        | /runs    | POST   | Submit a run |
        | /runs/{id} | GET  | Get run status |

      The CI badge is real-time — green when CI passes, red when it fails.
      This is the signal that tells a hiring manager: this person ships real code.
```

---

## Alternative: Deploy to Render

```yaml
render_alternative:
  why: "Simpler than Railway for first deployment — no API token needed for basic deploys"
  steps:
    - "Sign up at render.com"
    - "New → Web Service → Connect GitHub repo"
    - "Render auto-detects Dockerfile"
    - "Add Redis via Render's managed Redis (free tier)"
    - "Set env vars in Render dashboard"
    - "Auto-deploy is enabled by default on push to main"
    - "Get the URL from Render dashboard"
  note: >
    Render's free tier spins down after 15 minutes of inactivity.
    The first request after spin-down takes 30+ seconds.
    Acceptable for a portfolio demo, not for production.
```

---

## Done When

```yaml
done_when:
  - Full pipeline runs: test → build → push → deploy — all green
  - The deployed service responds to curl at a public URL
  - A code change goes from git push to live deployment automatically
  - The README has a CI badge and the live URL
  - You can explain every job in the pipeline workflow without looking at notes
  - You know where secrets live and why they're not in the workflow file

graduation_test: >
  Can you recreate this pipeline from scratch for a new project?
  Given a FastAPI service with tests and a Dockerfile, can you write the full
  pipeline.yml from memory in under 30 minutes?

  If yes — Docker and CI/CD are no longer prerequisites you're missing.
  Every OSS capstone in the other ramps can now be fully deployed.
```
