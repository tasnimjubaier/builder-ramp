# Contributing to ramp-lab

---

## Phase 1 — Curator Only

All ramps are currently authored and reviewed by the repo owner. This ensures spec quality and scaffold consistency before the community contribution model opens.

If you find an issue with an existing ramp — unclear requirement, broken scaffold, incorrect acceptance criteria — open a bug report issue.

---

## Phase 2 — Community Contributions (Coming)

Phase 2 opens when the first 10 ramps are published and the spec format is stable.

The process:

1. Open a **ramp proposal issue** using the proposal template
2. Wait for approval before building — this avoids wasted work
3. Once approved, build the ramp following the format exactly
4. Submit a PR to `main` with the spec and starter only
5. Submit a separate PR to `solutions` with the reference implementation

---

## Ramp Format Requirements

Every ramp must have exactly these files:

```
ramps/ramp-NN-slug/
  README.md     # the spec
  starter/      # the scaffold
  hints.md      # optional — nudges only
```

### README.md must contain

- **Context** — why this skill matters, what real problem it maps to
- **Goal** — one sentence, what you are building
- **Requirements** — numbered list, precise and testable
- **Starter** — what the scaffold gives you and what it doesn't
- **Acceptance Criteria** — how you know you're done
- **Resources** — docs links only, no tutorials or walkthroughs

### starter/ must include

- Folder structure
- Dependency files (`requirements.txt`, `go.mod`, `package.json`, etc.)
- Config files wired up (`.env.example`, `docker-compose.yml` if needed)
- Entry point with function signatures and docstrings — no implementation
- Test file stubs with test names — no test logic

### starter/ must not include

- Any implementation logic
- Filled-in test assertions
- Working solution in comments

### hints.md

Optional. One or two nudges per stuck point. Written as questions or directions, never as code.

---

## Contribution Gates

PRs that fail any of these are rejected:

- Ramp does not follow the spec format
- Starter is missing or doesn't install/run
- Requirements are ambiguous or untestable
- Solution code appears on `main`
- Hints contain code

---

## Branch Model

- `main` — ramps only. No solution code. Ever.
- `solutions` — reference implementations, mirroring `ramps/` structure
- Your fork — where your learner branch lives. Don't open learner branches against upstream.
