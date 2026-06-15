# ramp-lab

Hands-on project ramps for software engineers. Each ramp is a structured build sequence: real concepts, real artifacts, no hand-holding. You read enough to understand what to build, then you build it.

Not tutorials. Not katas. Project-level practice with a clear done state.

---

## Ramps

```yaml
ramps:
  01_python_ramp:
    title: "Python as a Professional Language"
    stages: 8
    type: additive       # one codebase grows through all stages
    capstone: "devtool — installable Typer CLI"
    unlocks: [02, 03, 04, 06]
    prereq: "basic Python syntax (LeetCode level)"
    summary: >
      LeetCode Python gets you syntax. This ramp gets you professional Python — the kind
      used inside frameworks and production systems. venvs, packages, OOP, Pydantic,
      async, packaging. One codebase grows through all 8 stages.

  02_agentic_ramp:
    title: "Agentic AI Systems with LangGraph"
    stages: 10
    type: additive
    capstone: "mini-agenteval — publishable OSS trajectory assertion library"
    prereq: "01_python_ramp stages 01–06"
    summary: >
      Builds the full agentic stack from a single LangGraph node to production-grade
      multi-agent systems. Raw LLM API → tools → state routing → callbacks → checkpointing
      → multi-agent → evaluation → observability → resilience.

  03_fastapi_ramp:
    title: "FastAPI for AI Backends"
    stages: 5
    type: mixed          # stages 01–04 standalone, stage 05 additive
    capstone: "LangGraph agent deployed behind FastAPI"
    prereq: "01_python_ramp stages 01–06"
    summary: >
      The standard Python API framework in every AI stack. Typed endpoints, async handlers,
      dependency injection, background tasks, Docker. Capstone deploys a LangGraph agent
      behind a streaming FastAPI endpoint.

  04_rag_ramp:
    title: "RAG Pipeline Engineering"
    stages: 6
    type: additive
    capstone: "Production RAG pipeline with RAGAS eval scores, deployed in Docker Compose"
    prereq: "01_python_ramp stages 01–05"
    summary: >
      Retrieval-Augmented Generation from first vector store setup to hybrid search and
      RAGAS evaluation. One real document corpus, one growing codebase. Ties into the
      agentic ramp via a LangGraph retriever node.

  05_typescript_ramp:
    title: "TypeScript for Python Developers"
    stages: 5
    type: mixed          # stages 01–04 standalone, stage 05 additive
    capstone: "TypeScript SDK wrapping your FastAPI agent API"
    prereq: "none — standalone, but 01_python_ramp concepts 01–03 help"
    summary: >
      The specific TypeScript subset that appears in AI engineer JDs: typed APIs, async
      patterns, SDK writing. Not "learn JavaScript" — the type system, Fastify, zod,
      and publishing. Uses Bun as runtime.

  06_mcp_ramp:
    title: "Model Context Protocol"
    stages: 3
    type: additive
    capstone: "MCP server exposing mini-agenteval trajectory capture as a tool"
    prereq: "01_python_ramp stages 01–06, 02_agentic_ramp"
    summary: >
      MCP is now a requirement in agentic AI engineer JDs. Understand the protocol,
      run an existing server, build your own. Ties directly into your existing agentic
      work — the tools already exist, MCP is the deployment target.

  07_docker_cicd_ramp:
    title: "Docker and CI/CD"
    stages: 4
    type: additive
    capstone: "Full CI/CD pipeline for agent-api or rag-api, deployed to Railway"
    prereq: "03_fastapi_ramp recommended as base service"
    summary: >
      Infrastructure skills that are blocking requirements for every OSS project you ship.
      Multi-stage Dockerfiles, Compose stacks, GitHub Actions pipelines. One service
      grows through all 4 stages. Unlocks all other capstone deployments.

  08_cpp_hft_ramp:
    title: "C++ for HFT Systems"
    stages: 7
    type: additive
    capstone: "Order-book backtester with pluggable strategy interface"
    prereq: "C++ syntax basics, basic data structures"
    summary: >
      Targets professional-grade modern C++ and HFT system knowledge: modern C++17/20,
      memory and latency, order book data structures, exchange mechanics, strategy
      interfaces, and backtesting. One trading engine skeleton grows through all 7 stages.
```

---

## Stage Structure

Every ramp follows this pattern. Inside each ramp folder:

```
ramps/NN_ramp_name/
  README.md           # overview, stage index, completion criteria, approach rules
  progress.md         # per-stage status tracker — update as you go
  NN_stage_name.md    # one file per stage — concepts, build steps, done-when
```

Each stage file has:
- **Concepts** — what you're learning and why it matters, in YAML
- **Build Steps** — numbered, with exact code where needed
- **Deliberate Breaks** — things to break on purpose to cement understanding
- **Done When** — concrete checklist, not "you understand it"
- **Open Questions** — things to carry forward into the next stage

---

## Solutions

Solutions live in `solutions/<ramp-name>/` on the `solutions` branch. The structure mirrors `ramps/` — each ramp folder gets a `solution/` directory alongside the spec.

```
solutions/
  01_python_ramp/
    devtool/            # the project built across all 8 stages
  02_agentic_ramp/
    raw_llm.py          # stage 01 prereq
    agent.py            # evolves through all 10 stages
    mini_agenteval/     # capstone package
  ...
```

Solutions are reference implementations — check them after solving, not before.

---

## Ramp Dependencies

```
01_python_ramp  ──►  02_agentic_ramp  ──►  06_mcp_ramp
             │
             └────►  03_fastapi_ramp  ──►  07_docker_cicd_ramp
             │
             └────►  04_rag_ramp

05_typescript_ramp  (standalone, loosely ties to 03)

08_cpp_hft_ramp     (fully independent track)
```

Start with `01_python_ramp` unless you're on the C++ track or have existing Python foundations.

---

## How to Use

1. Pick a ramp. Read its `README.md` to understand scope and prereqs.
2. Open `progress.md` — update status as you go.
3. Work through stages in order. Each stage has explicit build steps.
4. Every stage produces runnable code. If you can't run it, it's not done.
5. Break things deliberately. Each stage has a section for this.
6. Check the solution only after your own implementation works.

---

## Contributing

Phase 1 is curator-only. See [CONTRIBUTING.md](CONTRIBUTING.md) for the Phase 2 community model.
