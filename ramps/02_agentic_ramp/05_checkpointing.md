# Stage 05 — Checkpointing and Human-in-the-Loop

**Status:** not started
**Estimated time:** 2–3 days
**Depends on:** Stage 04 (you understand state flow and the callback system)

Stateless agents are toys. Real agents run across multiple turns, pause for human approval, and resume exactly where they left off. LangGraph's checkpointing system makes this possible. This stage teaches the persistence layer — and the human-in-the-loop (HITL) pattern that appears in both GuardRail and AgentResilience.

---

## What You're Building

An agent that:
1. Drafts an action plan based on a user prompt
2. **Pauses** before executing — shows the plan to the human and waits for approval
3. Resumes after the human approves (or aborts if they reject)
4. Executes the plan and completes

The pause and resume work across separate `invoke()` calls — the graph literally stops mid-execution and its state is saved to a database.

---

## Concepts This Stage Teaches

```yaml
concepts:
  MemorySaver:
    what: >
      An in-memory checkpointer. Every time a node finishes, LangGraph saves the full
      state to the checkpointer keyed by thread_id. On the next invoke() with the same
      thread_id, the graph resumes from exactly where it stopped.
    why: >
      This is what makes multi-turn conversations work. The thread_id is the session ID.
      Same thread_id = same "conversation". Without checkpointing, every invoke() starts
      from scratch.

  interrupt_before:
    what: >
      A compile()-time setting that tells LangGraph to pause the graph before a specific
      node runs. The invoke() call returns early — the node hasn't run yet.
    why: >
      This is the HITL mechanism GuardRail uses. When a safety rule triggers, the graph
      is interrupted before the next node executes. A human reviews. The graph resumes
      or aborts based on their decision. The AgentTrace hitl_pause span records exactly
      this event.

  thread_id:
    what: >
      A string identifier passed in the config to every invoke() call.
      config={"configurable": {"thread_id": "session-abc"}}
    why: >
      This is how you tell LangGraph "this invoke() is a continuation of a previous run,
      not a new one." Same thread_id = resume; new thread_id = fresh start.

  get_state() and update_state():
    what: >
      get_state(config) reads the current saved state for a thread.
      update_state(config, values) patches the state before resuming — you can modify
      state between an interrupt and a resume.
    why: >
      This is how human approval injects feedback into the graph. The human rejects
      the plan → you call update_state() to add a "rejected" flag → invoke() resumes
      → the next node reads the flag and aborts.

  None input on resume:
    what: >
      When resuming an interrupted graph, you call invoke(None, config=...).
      Passing None tells LangGraph "continue from checkpoint, don't inject new input."
    why: >
      A common source of confusion — people pass the original input again and get
      unexpected behavior. The correct pattern is invoke(None, config).
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Set up MemorySaver and compile with interrupt_before"
    detail: >
      from langgraph.checkpoint.memory import MemorySaver

      memory = MemorySaver()

      app = graph.compile(
          checkpointer=memory,
          interrupt_before=["execute_node"]  # pause before this node runs
      )

  2:
    task: "Define the state schema with an approval field"
    detail: >
      class PlanState(TypedDict):
          messages: Annotated[list, add_messages]
          plan: str           # set by planner_node
          approved: bool      # set by human via update_state()
          result: str         # set by execute_node

  3:
    task: "Build planner_node and execute_node"
    detail: >
      planner_node: calls LLM to draft a 3-step plan, writes it to state["plan"]
      execute_node: reads state["plan"], "executes" each step (mock — just print them),
                    writes a completion message to state["result"]

  4:
    task: "First invoke — run up to the interrupt"
    detail: >
      thread_config = {"configurable": {"thread_id": "approval-demo-1"}}

      result = app.invoke(
          {"messages": [HumanMessage(content="Plan a system migration for me")]},
          config=thread_config
      )

      # At this point, planner_node has run but execute_node has NOT
      # result["plan"] contains the plan

      current_state = app.get_state(thread_config)
      print("Plan:", current_state.values["plan"])
      print("Next node to run:", current_state.next)  # should be ["execute_node"]

  5:
    task: "Human approval — inject a decision"
    detail: >
      # Simulate human approving:
      app.update_state(thread_config, {"approved": True})

      # Simulate human rejecting:
      # app.update_state(thread_config, {"approved": False})

  6:
    task: "Resume by invoking with None"
    detail: >
      final_result = app.invoke(None, config=thread_config)
      print("Result:", final_result["result"])

  7:
    task: "Add a guard in execute_node"
    detail: >
      def execute_node(state):
          if not state.get("approved", False):
              return {"result": "Execution aborted — plan was not approved."}
          # ... rest of execution
```

---

## Multi-Turn Conversation Exercise

```yaml
exercise:
  task: "Build a 3-turn conversation with memory"
  detail: >
    Build a simple chat agent with MemorySaver. No interrupt_before — just checkpointing.

    First invoke: "My name is Alex."
    Second invoke (same thread_id): "What is my name?"

    The second invoke should return "Alex" — because the first message is still in
    the checkpointed state.

    Then do the same with a DIFFERENT thread_id on the second invoke.
    The second invoke should NOT know the name — because it's a new thread.

  why: >
    This is the clearest demonstration of what thread_id does. The moment you see
    one thread remember and the other forget, the checkpointing model clicks.
```

---

## Done When

```yaml
done_when:
  - The plan-and-approve graph pauses, shows the plan, accepts approval, and resumes
  - You can demonstrate that rejection aborts execution (approved=False path)
  - The multi-turn exercise works — same thread_id remembers, different thread_id forgets
  - You can call get_state() and explain every field it returns
  - You understand what invoke(None, config) does and why None is correct for resume
  - You can explain the HITL pattern in GuardRail's deep dive using the vocabulary
    from this stage: interrupt, checkpoint, update_state, resume
```

---

## What This Unlocks

Stage 06 builds multi-agent systems — a supervisor dispatching to subagents. The supervisor pattern is just checkpointing + conditional routing at a higher level. You need both stages before that makes sense.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What happens if you call invoke() a third time after a completed graph — does it restart or error?
  - Can you use SQLite as a checkpointer instead of MemorySaver? Find the LangGraph docs for this.
  - What does current_state.next return when the graph has fully completed?
  - Can you interrupt_before multiple nodes? What order do the interrupts fire in?
  - What happens to the checkpoint if a node raises an exception mid-run?
```
