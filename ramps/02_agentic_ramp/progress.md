# Agentic Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 10
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_first_agent:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "raw_llm.py (OpenAI API prereq) + agent.py (single-node LangGraph graph)"
    notes: ""

  02_tools_and_state:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "two-node graph with @tool, ToolNode, tools_condition, custom AgentState"
    notes: ""

  03_multi_node_graphs:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "multi-node graph with conditional edges and state routing"
    notes: ""

  04_callbacks:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "custom callback handler, LangSmith connected, traces visible"
    notes: ""

  05_checkpointing:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "MemorySaver persistence, multi-turn conversation, human-in-the-loop interrupt"
    notes: ""

  06_multi_agent:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "supervisor + subagent graph, Langfuse self-hosted with multi-agent traces"
    notes: ""

  07_evaluation:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "trajectory assertions on tool calls, pytest CI gate for agent behavior"
    notes: ""

  08_observability:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "OpenTelemetry spans from inside a LangGraph node, Jaeger traces"
    notes: ""

  09_resilience:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "exponential backoff wrapper, cost budget guard on tools"
    notes: ""

  10_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "3–4 days"
    artifact: "mini AgentEval — evaluates agent trajectories end-to-end, publishable OSS repo"
    notes: ""
```

---

## Prereqs

```yaml
prereqs:
  python_ramp: "stages 01–06 required before starting"
```

---

## Notes

<!-- freeform — add blockers, surprises, things to revisit -->
