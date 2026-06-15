# Stage 06 — Multi-Agent Systems

**Status:** not started
**Estimated time:** 3–4 days
**Depends on:** Stage 05 (you understand state, routing, tools, and checkpointing)

This is where the architecture scales. A single agent is a loop. A multi-agent system is a network — a supervisor decides which specialist agent handles a task, delegates to it, collects the result, and decides what to do next. This pattern is in every senior agentic JD and is the foundation for A2A-Mesh, AgentTrace's multi-agent trace propagation, and AgentResilience's cross-agent fallback.

---

## What You're Building

A supervisor-worker multi-agent system:
- **Supervisor agent** — receives the user request, decides which worker to dispatch to, collects results, decides when the task is complete
- **Researcher worker** — handles research/lookup tasks (mock web search tool)
- **Writer worker** — handles writing/summarization tasks (LLM with writing prompt)

The supervisor can dispatch to either worker, collect results, and dispatch again if needed before returning a final answer.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Supervisor pattern:
    what: >
      A node (the supervisor) that reads state and decides which subagent to call next.
      The subagent runs, writes its result to state, and returns control to the supervisor.
      The supervisor decides: dispatch to another agent, or finish?
    why: >
      This is the most common multi-agent architecture. Every JD that mentions
      "multi-agent orchestration" expects you to know this pattern. AgentTrace's
      multi-agent span propagation is specifically about tracing this delegation chain.

  next field in state:
    what: >
      A conventional state field (often called "next") that the supervisor writes to
      indicate which agent should run next. The conditional edge reads this field.
    why: >
      This is the routing mechanism that replaces the simple string routing from
      Stage 03. The supervisor node is just writing to state["next"] and the
      conditional edge routes based on it.

  Subgraphs:
    what: >
      Each worker can itself be a compiled LangGraph graph, invoked as a node
      inside the parent graph. The parent graph's state schema must be compatible
      with (or a superset of) the subgraph's state schema.
    why: >
      This is what makes multi-agent systems composable — each agent is an
      independently testable, independently deployable unit. AgentTrace propagates
      trace context across subgraph boundaries using this mechanism.

  Compile-time vs runtime workers:
    what: >
      Workers can be defined as Python functions (simple), as ToolNodes wrapping
      LLM+tools (medium), or as compiled subgraphs (complex).
    why: >
      Start with function workers. Upgrade one to a compiled subgraph in the exercise.
      The upgrade path teaches you where the complexity boundary sits.

  Handoff messages:
    what: >
      Special messages that signal "I am done, return to supervisor with this result."
      Often implemented as a tool call (the supervisor calls a "route_to" tool that
      returns the next destination).
    why: >
      This is the A2A handoff pattern — the supervisor and workers communicate via
      structured messages, not direct function calls.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the shared supervisor state"
    detail: >
      class SupervisorState(TypedDict):
          messages: Annotated[list, add_messages]
          next: str              # which agent to dispatch to
          researcher_result: str  # filled by researcher
          writer_result: str      # filled by writer
          final_answer: str       # filled by supervisor on completion

  2:
    task: "Build the supervisor node"
    detail: >
      The supervisor LLM gets a system prompt like:
      "You coordinate a research and writing team. Given the conversation so far,
       decide whether to dispatch to 'researcher', 'writer', or output 'FINISH'.
       Reply with only one word."

      The node calls the LLM, parses the response, and writes to state["next"].

  3:
    task: "Build researcher_node (simple function worker)"
    detail: >
      def researcher_node(state):
          # mock: pretend to search and return a result
          last_message = state["messages"][-1].content
          result = f"Research result for '{last_message}': [mock data about topic]"
          return {
              "researcher_result": result,
              "messages": [AIMessage(content=result, name="researcher")]
          }

  4:
    task: "Build writer_node (LLM worker)"
    detail: >
      def writer_node(state):
          research = state.get("researcher_result", "")
          # call LLM with writing prompt + research result
          # write to state["writer_result"]

  5:
    task: "Wire supervisor conditional edge"
    detail: >
      def route_from_supervisor(state):
          return state["next"]   # "researcher", "writer", or "FINISH"

      graph.add_conditional_edges("supervisor", route_from_supervisor, {
          "researcher": "researcher",
          "writer": "writer",
          "FINISH": END
      })

      # Workers always return to supervisor
      graph.add_edge("researcher", "supervisor")
      graph.add_edge("writer", "supervisor")

  6:
    task: "Test with a task that requires both workers"
    detail: >
      Ask: "Research the history of the internet and write a 2-sentence summary."
      Expected flow: supervisor → researcher → supervisor → writer → supervisor → FINISH

      Print state["next"] at each supervisor call to confirm the routing.

  7:
    task: "Add a max_dispatch_count guard"
    detail: >
      Add dispatch_count: int to state. Increment it in supervisor_node.
      If dispatch_count > 5, force state["next"] = "FINISH" regardless of LLM output.
      This is the loop-bound guard — essential for production agents.
```

---

## Subgraph Exercise

```yaml
exercise:
  task: "Upgrade researcher_node to a compiled subgraph"
  detail: >
    Build a separate ResearchGraph with its own state:
      - query_node: extracts the research query from the message
      - search_node: runs the mock search tool
      - summarize_node: summarizes the search result

    Compile it as research_app = ResearchGraph.compile()

    Replace the researcher_node function in the supervisor graph with a wrapper
    that invokes research_app and writes its output back to SupervisorState.

    This is the subgraph pattern. The supervisor graph doesn't know or care what
    happens inside research_app — it just calls it and gets a result.

  why: >
    This is the exact architecture AgentTrace instruments when it talks about
    "span context propagation across agent hops" and "full delegation chain as
    a single navigable trace tree." Each subgraph invocation is one hop.
```

---

## Langfuse Integration

After building the multi-agent system, add Langfuse as the observability layer. Unlike LangSmith (SaaS-only), Langfuse is open-source and self-hostable — a real differentiator for EU startup roles where data sovereignty matters. And since you understand callbacks from Stage 04, the integration is a 15-minute addition, not a new concept.

```yaml
langfuse_setup:
  why_here: >
    The multi-agent system is the most complex thing you've built so far —
    a supervisor dispatching to subagents across multiple LLM calls. This is
    where observability earns its keep. A single trace should show the full
    supervisor → researcher → supervisor → writer → supervisor chain. Langfuse
    makes that visible in one UI.

  steps:
    1:
      task: "Run Langfuse locally with Docker Compose"
      detail: >
        # Add to your docker-compose.yml or create a new one:

        services:
          langfuse-server:
            image: langfuse/langfuse:latest
            depends_on:
              - langfuse-db
            ports:
              - "3000:3000"
            environment:
              DATABASE_URL: postgresql://langfuse:langfuse@langfuse-db:5432/langfuse
              NEXTAUTH_SECRET: your-random-secret-here
              NEXTAUTH_URL: http://localhost:3000
              SALT: your-random-salt-here

          langfuse-db:
            image: postgres:15-alpine
            environment:
              POSTGRES_USER: langfuse
              POSTGRES_PASSWORD: langfuse
              POSTGRES_DB: langfuse
            volumes:
              - langfuse_db:/var/lib/postgresql/data

        volumes:
          langfuse_db:

        docker compose up -d
        Open http://localhost:3000 → sign up (local only, any credentials)
        Create a project → get the Public Key and Secret Key

    2:
      task: "Connect your multi-agent system to Langfuse"
      detail: >
        pip install langfuse

        Add to .env:
          LANGFUSE_PUBLIC_KEY=pk-lf-your-key
          LANGFUSE_SECRET_KEY=sk-lf-your-key
          LANGFUSE_HOST=http://localhost:3000

        Add to your agent code:
          from langfuse.callback import CallbackHandler as LangfuseHandler

          langfuse_handler = LangfuseHandler()

          result = app.invoke(
              input,
              config={"callbacks": [langfuse_handler]},
          )

        Run the multi-agent task. Open Langfuse at localhost:3000.

    3:
      task: "Explore the Langfuse trace for your multi-agent run"
      detail: >
        Find your trace. You should see:
          Supervisor (LLM call) → "researcher"
          Researcher (function) → result
          Supervisor (LLM call) → "writer"
          Writer (LLM call) → result
          Supervisor (LLM call) → "FINISH"

        For each LLM call, check:
          - Input messages (what the LLM received)
          - Output message (what it returned)
          - Token counts and latency

        This is the full delegation chain visible as one trace.
        Compare it to manually reading state["next"] in your debug prints.

    4:
      task: "Add a score to a trace"
      detail: >
        from langfuse import Langfuse

        lf = Langfuse()

        # After the run completes, score the final answer quality
        trace_id = langfuse_handler.get_trace_id()
        lf.score(
            trace_id=trace_id,
            name="answer_quality",
            value=0.9,   # 0.0 to 1.0
            comment="Factually correct, well summarized",
        )

        Find the score in Langfuse UI. This is the eval annotation workflow —
        human scores on traces that can be filtered, aggregated, and compared.

    5:
      task: "Compare LangSmith vs Langfuse — pick your position"
      detail: >
        Run the same multi-agent task with both:
          config={"callbacks": [langfuse_handler]}   # Langfuse
          # LangSmith active via env vars simultaneously

        Side-by-side comparison:
          LangSmith:   Better LangGraph-native rendering, easier UI, SaaS only
          Langfuse:    Self-hostable, open-source, EU data residency, scoring built-in

        For interview contexts:
          "I use LangSmith for its LangGraph integration and Langfuse for
           production environments where data sovereignty is required."
          This answer demonstrates you know both, understand the tradeoff,
          and can make an informed choice for a given context.
```

---

## Done When

```yaml
done_when:
  - Supervisor correctly dispatches to researcher and writer and terminates
  - The dispatch_count guard prevents infinite loops
  - You have upgraded researcher to a compiled subgraph
  - You can draw the delegation chain on paper from a single invoke() call
  - You can explain the "next field in state" routing pattern in plain language
  - You can describe how trace context would propagate across the supervisor → subgraph boundary
    (even without implementing it yet — the concept is enough for now)
```

---

## What This Unlocks

Stage 07 adds evaluation — asserting what the agent did, not just what it returned. After building a multi-agent system you have real behavior to evaluate, which makes the AgentEval architecture concrete rather than theoretical.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What happens if the supervisor LLM outputs a value not in the conditional edge mapping?
  - Can a worker call the supervisor back directly, or must it always return to a fixed node?
  - How does checkpointing work with subgraphs — is each subgraph's state saved separately?
  - What is the LangGraph "send" API and how does it differ from the supervisor pattern?
    (Research this — it's the parallel dispatch pattern.)
```
