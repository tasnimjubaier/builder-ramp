# Stage 03 — GitHub Actions CI

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — add .github/workflows/ to the Stage 02 project
**Depends on:** Stage 02 (Compose stack working), GitHub repo with the code

CI (Continuous Integration) means every push automatically runs your tests. A green check on a PR means the tests pass — no manual "did you test this?" A red check means something broke and the code shouldn't be merged. This stage builds that pipeline: push code → tests run → image builds → image pushes to a registry. Automatically, every time.

---

## What You're Building

A GitHub Actions workflow that on every push:
1. Runs the pytest test suite
2. Builds the Docker image
3. Pushes the image to GitHub Container Registry (GHCR)
4. Posts a status check on the commit

A second workflow for pull requests that runs tests only (no push to registry).

---

## Concepts This Stage Teaches

```yaml
concepts:
  GitHub Actions structure:
    what: >
      A YAML file in .github/workflows/.
      on: defines triggers (push, pull_request, schedule, manual).
      jobs: defines parallel or sequential job groups.
      steps: the individual commands inside a job.
      Each step runs: a shell command or uses: an action (reusable script).
    why: >
      GitHub Actions is the standard CI/CD for open source and most startups.
      Every OSS project you contribute to has one of these files.
      Reading and writing them is a baseline expectation for mid+ engineers.

  Runners:
    what: >
      runs-on: ubuntu-latest — the machine that runs your job.
      GitHub provides Ubuntu, macOS, and Windows runners for free (within limits).
      ubuntu-latest is the default for almost everything.
    why: >
      Your CI runs on a fresh VM every time — not your laptop. It has no .env,
      no .venv, no cached packages unless you set that up explicitly.
      Understanding this explains why "it works on my machine" doesn't help in CI.

  Actions (uses:):
    what: >
      Reusable step definitions from the GitHub Marketplace or your own repo.
      actions/checkout@v4 — checks out your code into the runner.
      actions/setup-python@v5 — installs a specific Python version.
      docker/login-action@v3 — logs into a container registry.
    why: >
      Actions are the building blocks. You compose them with your own run: commands.
      Using well-maintained official actions (actions/*, docker/*) is safer than
      writing the equivalent shell commands yourself.

  Secrets:
    what: >
      Encrypted key-value pairs stored in GitHub repo settings (Settings → Secrets).
      Referenced in workflows as ${{ secrets.MY_SECRET }}.
      Never visible in logs — GitHub masks them automatically.
    why: >
      API keys, registry passwords, and deploy tokens must never be in code.
      GitHub Secrets is where they live for CI/CD. You'll use secrets.GITHUB_TOKEN
      (automatically provided) and any custom secrets you create.

  GitHub Container Registry (GHCR):
    what: >
      A Docker image registry built into GitHub. Images are stored at
      ghcr.io/your-username/your-repo:tag.
      Free for public repos, free tier for private repos.
    why: >
      No separate account or service needed — just your GitHub account.
      Images pushed here can be pulled anywhere Docker runs.

  Matrix builds:
    what: >
      strategy: matrix: python-version: [3.11, 3.12]
      Runs the same job for every combination of matrix values.
    why: >
      Test that your package works on multiple Python versions in parallel.
      One job definition, N actual jobs. Common for libraries.

  Caching dependencies:
    what: >
      actions/cache caches a directory (like ~/.cache/pip) between runs.
      If requirements.txt hasn't changed, the cached packages are used.
    why: >
      Installing dependencies from scratch on every run adds 1–3 minutes.
      Caching cuts that to seconds for unchanged dependencies.
```

---

## Setup

```yaml
requirements:
  - Code is in a GitHub repository
  - Repository has a Dockerfile and docker-compose.yml from Stages 01–02
  - Tests exist in tests/ directory (from Python ramp)
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create the workflow directory"
    detail: >
      mkdir -p .github/workflows

  2:
    task: "Write the CI workflow (test only)"
    detail: >
      # .github/workflows/ci.yml

      name: CI

      on:
        push:
          branches: ["main", "develop"]
        pull_request:
          branches: ["main"]

      jobs:
        test:
          runs-on: ubuntu-latest

          steps:
            - name: Checkout code
              uses: actions/checkout@v4

            - name: Set up Python
              uses: actions/setup-python@v5
              with:
                python-version: "3.12"

            - name: Cache pip dependencies
              uses: actions/cache@v4
              with:
                path: ~/.cache/pip
                key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}
                restore-keys: |
                  ${{ runner.os }}-pip-

            - name: Install dependencies
              run: |
                pip install -r requirements.txt
                pip install pytest pytest-asyncio pytest-cov

            - name: Run tests
              run: pytest tests/ -v --cov=. --cov-report=xml

            - name: Upload coverage
              uses: codecov/codecov-action@v4
              with:
                file: ./coverage.xml
              continue-on-error: true   # don't fail if codecov is unavailable

  3:
    task: "Write the Docker build and push workflow"
    detail: >
      # .github/workflows/docker.yml

      name: Docker Build & Push

      on:
        push:
          branches: ["main"]
          tags: ["v*"]   # also trigger on version tags like v1.0.0

      env:
        REGISTRY: ghcr.io
        IMAGE_NAME: ${{ github.repository }}   # e.g. your-username/your-repo

      jobs:
        build-and-push:
          runs-on: ubuntu-latest
          permissions:
            contents: read
            packages: write   # needed to push to GHCR

          steps:
            - name: Checkout code
              uses: actions/checkout@v4

            - name: Log in to GitHub Container Registry
              uses: docker/login-action@v3
              with:
                registry: ${{ env.REGISTRY }}
                username: ${{ github.actor }}
                password: ${{ secrets.GITHUB_TOKEN }}

            - name: Extract metadata (tags, labels)
              id: meta
              uses: docker/metadata-action@v5
              with:
                images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
                tags: |
                  type=ref,event=branch
                  type=semver,pattern={{version}}
                  type=sha,prefix=sha-

            - name: Build and push Docker image
              uses: docker/build-push-action@v5
              with:
                context: .
                push: true
                tags: ${{ steps.meta.outputs.tags }}
                labels: ${{ steps.meta.outputs.labels }}
                cache-from: type=gha
                cache-to: type=gha,mode=max

  4:
    task: "Push to GitHub and watch the workflow run"
    detail: >
      git add .
      git commit -m "add GitHub Actions CI/CD"
      git push origin main

      Go to your GitHub repo → Actions tab.
      Watch the CI workflow run in real time.

      Click on each step to expand it and read the logs.
      Find:
        - Which step takes the longest?
        - What does the pip cache step show on the first run vs second run?
        - What does the test output look like — same as running pytest locally?

  5:
    task: "Make a test fail and observe the check"
    detail: >
      Add a deliberately failing test:
        def test_intentional_failure():
            assert 1 == 2

      Commit and push. Watch the CI run.
      Notice:
        - The Actions tab shows a red X
        - The commit shows a red X next to it
        - The PR (if you create one) shows a failing check

      Revert the failing test. Push again. Confirm the check goes green.
      This is CI working — broken code is blocked from merging.

  6:
    task: "Add a Docker build test to CI"
    detail: >
      Add a job to ci.yml that builds the image but doesn't push it.
      This catches Dockerfile errors before the main branch:

          docker-build:
            runs-on: ubuntu-latest
            steps:
              - uses: actions/checkout@v4

              - name: Build Docker image
                run: docker build -t myapp:test .

              - name: Start services
                run: docker compose up -d

              - name: Wait for services
                run: sleep 10

              - name: Health check
                run: curl --fail http://localhost:8000/health

              - name: Teardown
                run: docker compose down -v

  7:
    task: "Pull your published image"
    detail: >
      After the docker.yml workflow runs successfully:

        docker pull ghcr.io/your-username/your-repo:main
        docker run -p 8000:8000 -e REDIS_HOST=localhost ghcr.io/your-username/your-repo:main

      You've just pulled an image from a public registry and run it with one command.
      This is what "deployable" means — the image exists independently of your machine.
```

---

## Workflow Anatomy Reference

```yaml
# Full annotated workflow structure

name: CI                    # shown in the Actions tab

on:                         # triggers
  push:
    branches: ["main"]
  pull_request:

jobs:                       # parallel job groups
  test:                     # job name (arbitrary)
    runs-on: ubuntu-latest  # runner OS

    steps:                  # sequential steps in this job
      - name: Human-readable step name
        uses: actions/checkout@v4    # use a pre-built action

      - name: Run a command
        run: echo "hello"            # run a shell command

      - name: Multi-line command
        run: |
          echo "line 1"
          echo "line 2"

      - name: Use a secret
        run: echo "key=${{ secrets.MY_KEY }}" # masked in logs

      - name: Use output from a previous step
        id: prev_step          # give the step an id
        run: echo "value=42" >> $GITHUB_OUTPUT

      - name: Read the output
        run: echo "${{ steps.prev_step.outputs.value }}"

      - name: Conditional step
        if: github.ref == 'refs/heads/main'
        run: echo "only on main"
```

---

## Done When

```yaml
done_when:
  - CI workflow runs on push and shows green checks on passing commits
  - A failing test shows a red check — confirmed by deliberately breaking one
  - Docker workflow builds and pushes an image to GHCR on push to main
  - You can pull the published image and run it locally
  - You understand the difference between on: push and on: pull_request triggers
  - You can add a new step to a workflow from memory
  - You understand what secrets.GITHUB_TOKEN is and why it's automatically available
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between a job and a step? Can steps run in parallel?
  - What does needs: [test] in a job definition do?
    How do you make the docker build job only run if tests pass?
  - What is a reusable workflow and when would you use one?
  - What is the difference between environment: production and secrets: in a workflow?
    When would you use environments?
  - How do you debug a failing CI step that only fails in CI but works locally?
    What tools does GitHub Actions provide for this?
```
