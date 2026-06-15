# Agentic Ramp — End-to-End Learning Sequence

This is a deliberate skills ramp for building real agentic AI systems from scratch. Not a course. Not reading material. A sequence of small builds that each teach one layer of the stack by forcing you to actually run something, break it, and fix it.

The goal is to reach the point where the technical deep dives in `04_oss_intel/02_oss_ideas/07_ideas_v0.4.md` read as familiar, not foreign. Each stage leaves behind a working artifact — code that runs — not notes.

---

## Structure

```yaml
stages:
  01_first_agent:       "Single-node LangGraph agent. One LLM call. Make it run."
  02_tools_and_state:   "Add tools and structured state. Understand the execution loop."
  03_multi_node_graphs:  "Multi-node graphs with conditional edges. State routing."
  04_callbacks:         "Hook into the callback system. See every event that fires."
  05_checkpointing:     "Persistence across turns. Human-in-the-loop interrupts."
  06_multi_agent:       "Supervisor + subagent pattern. Agent delegation."
  07_evaluation:        "Evaluate agent trajectories. Assertions on behavior."
  08_observability:     "OpenTelemetry spans. Trace the execution topology."
  09_resilience:        "Retries, fallback, cost budgets. Handle what goes wrong."
  10_capstone:          "Mini version of AgentEval. Tie all layers together."
```

---

## What This Ramp Now Covers

```yaml
coverage:
  llm_api_raw:         "Stage 01 — raw OpenAI API calls, streaming, token counting, cost"
  prompt_engineering:  "Stage 03 — system prompts, few-shot, format constraints, temperature discipline"
  langsmith:           "Stage 04 — connected and operational, traces visible in UI"
  langfuse:            "Stage 06 — self-hosted, multi-agent traces, scoring workflow"
  langgraph:           "Stages 01–10 — full stack from first node to production resilience"
  opentelemetry:       "Stage 08 — custom spans, Jaeger, gen_ai.* conventions"
  agent_eval:          "Stages 07+10 — trajectory assertions, golden runs, pytest CI gate"
```

---

## Completion Criteria

You are done with the ramp when you can:

```yaml
done_when:
  - Call the raw OpenAI API, handle streaming, and calculate cost from token counts
  - Write a system prompt and iterate it until routing behavior is reliable at temperature=0
  - Build a LangGraph multi-node graph with tools and conditional edges from memory
  - Hook a custom callback handler into any graph and explain every event that fires
  - Connect both LangSmith and Langfuse and explain the tradeoff between them
  - Write a trajectory assertion that passes or fails based on what tools were called
  - Emit an OpenTelemetry span from inside a LangGraph node
  - Wrap a tool with exponential backoff and a cost budget guard
  - Read the AgentTrace / AgentEval / AgentResilience deep dives and recognize every concept
```

---

## Approach

```yaml
rules:
  - Build first, read docs second — break it first, then look up why
  - Every stage has one artifact: working code that runs
  - No stage takes more than 3–4 days — if it does, scope is too wide
  - Each stage deliberately introduces the concepts the next stage depends on
  - Use the cheapest model (gpt-4o-mini or gemini-flash) to keep costs near zero
```
