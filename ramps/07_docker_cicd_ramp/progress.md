# Docker + CI/CD Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 4
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_dockerfile:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1–2 days"
    type: additive start
    artifact: "Dockerfile (naive + multi-stage), .dockerignore, image size comparison confirmed"
    notes: ""

  02_compose:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    type: additive
    artifact: "docker-compose.yml (API + Redis), healthcheck, named volume, data persistence confirmed"
    notes: ""

  03_github_actions:
    status: not_started
    started: ""
    completed: ""
    estimate: "2 days"
    type: additive
    artifact: "ci.yml (test + docker build), docker.yml (build + push to GHCR), green CI on GitHub"
    notes: ""

  04_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "3–4 days"
    type: additive
    artifact: "full pipeline.yml (test → build → push → deploy), live Railway URL, CI badge in README"
    notes: ""
```

---

## Prereqs

```yaml
prereqs:
  docker_desktop: "installed and running — docker --version should work"
  fastapi_ramp:   "capstone (or stage 03 job queue) used as the base service"
  github_repo:    "required for stage 03 onwards"
  railway_account: "required for capstone deployment — railway.app"

unlocks:
  fastapi_ramp_capstone: "stage 04 Docker is prerequisite — now understood deeply"
  rag_ramp_capstone:     "three-service Compose stack (API + Qdrant + Redis)"
  agentic_ramp:          "AgentTrace dev stack uses Compose + Jaeger"
  all_oss_capstones:     "every OSS capstone ships with a Dockerfile and CI workflow"
```

---

## Notes

<!-- freeform — add blockers, surprises, things to revisit -->
