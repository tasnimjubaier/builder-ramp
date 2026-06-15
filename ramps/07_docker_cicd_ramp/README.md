# Docker + CI/CD Ramp — Learning Sequence

Docker and CI/CD are infrastructure skills, not application skills — but they're blocking requirements for every OSS project you ship. Without them, your code only runs on your machine. With them, it runs anywhere and automatically tests and deploys itself on every push.

This ramp is additive. One service grows across all 4 stages. The capstone is a complete CI pipeline that builds, tests, and deploys one of your OSS projects.

---

## Structure

```yaml
stages:
  01_dockerfile:          "Dockerfile for a Python service. Multi-stage build. Additive start."
  02_compose:             "Multi-service Compose stack. API + dependencies together."
  03_github_actions:      "Test → build → push image pipeline. Triggered on push."
  04_capstone:            "Full CI/CD pipeline for agent-api or rag-api. Deployed to Railway."
```

---

## What This Ramp Unlocks

```yaml
unlocks:
  fastapi_ramp_capstone:  "Stage 04 (Docker) is prerequisite — now you understand it deeply"
  rag_ramp_capstone:      "Three-service Compose stack (API + Qdrant + Redis)"
  agentic_ramp:           "AgentTrace dev stack uses Compose + Jaeger"
  all_oss_projects:       "Every OSS capstone ships with a Dockerfile and CI workflow"
```

---

## Completion Criteria

```yaml
done_when:
  - You can write a multi-stage Dockerfile from memory
  - You can write a docker-compose.yml for a multi-service stack from memory
  - You understand container networking — why localhost doesn't work between containers
  - You have a GitHub Actions workflow that runs tests, builds an image, and pushes it
  - You have one OSS project deployed and accessible via a public URL
```

---

## Approach

```yaml
rules:
  - Additive — one service grows through all 4 stages
  - Use the FastAPI job queue service from the FastAPI ramp as the base service
  - If FastAPI ramp isn't done yet, build a minimal echo API as the base
  - Every stage produces something you can run with one command
  - Secrets never go in Dockerfiles or docker-compose.yml — always env vars or .env files
```
