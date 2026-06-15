# Stage 04 — The Callback System

**Status:** not started
**Estimated time:** 2 days
**Depends on:** Stage 03 (you have a multi-node graph with tools and conditional edges)

This is the stage where AgentTrace stops being abstract. The callback system is the single integration surface every LangGraph observability, eval, and tracing library hooks into — including LangSmith, LangFuse, and the entire AgentTrace architecture. After this stage you will understand exactly how AgentTrace works under the hood.

---

## What You're Building

A custom `CallbackHandler` attached to the Stage 03 graph that:
1. Prints every event that fires during execution, in order
2. Records a timeline: which node ran, when, what it received and returned
3. Counts total LLM calls and total tool calls
4. Prints a summary at the end

No third-party tracing service. Just Python and print statements.

---

## Concepts This Stage Teaches

```yaml
concepts:
  BaseCallbackHandler:
    what: >
      A LangChain base class with one method per event type. You subclass it,
      override the methods you care about, and pass an instance to invoke().
    why: >
      This is the single hook point that AgentTrace, LangSmith, and every
      observability library uses. Writing your own handler means you understand
      exactly what data is available at each event — which is precisely what
      AgentTrace's span design is based on.

  Event types:
    what: >
      on_chain_start / on_chain_end — graph-level and node-level execution
      on_llm_start / on_llm_end — each LLM API call
      on_tool_start / on_tool_end — each tool invocation
      on_agent_action / on_agent_finish — when using AgentExecutor (older pattern)
    why: >
      AgentTrace maps each of these to a structured OTel span. Knowing which
      event fires when, and what data it carries, is the prerequisite for
      understanding span design.

  run_id:
    what: >
      A UUID passed to every callback event. Every node execution, LLM call, and tool call
      in a single graph run shares a root run_id. Child runs have their own IDs but carry
      the parent's ID too.
    why: >
      This is how AgentTrace stitches together the trace tree — parent/child relationships
      between spans come from run_id and parent_run_id. Writing your own handler and
      printing these IDs makes the trace topology concept concrete.

  Passing callbacks to invoke():
    what: >
      app.invoke(input, config={"callbacks": [MyHandler()]})
    why: >
      This is the invocation pattern all observability libraries document. You will use
      this exact pattern when integrating AgentTrace later.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Subclass BaseCallbackHandler"
    detail: >
      from langchain_core.callbacks import BaseCallbackHandler

      class DebugHandler(BaseCallbackHandler):
          def __init__(self):
              self.timeline = []
              self.llm_call_count = 0
              self.tool_call_count = 0

  2:
    task: "Override on_chain_start and on_chain_end"
    detail: >
      def on_chain_start(self, serialized, inputs, *, run_id, parent_run_id=None, **kwargs):
          name = serialized.get("name", "unknown")
          self.timeline.append(f"[CHAIN START] {name} | run={str(run_id)[:8]}")

      def on_chain_end(self, outputs, *, run_id, **kwargs):
          self.timeline.append(f"[CHAIN END]   run={str(run_id)[:8]}")

  3:
    task: "Override on_llm_start and on_llm_end"
    detail: >
      def on_llm_start(self, serialized, prompts, *, run_id, **kwargs):
          self.llm_call_count += 1
          self.timeline.append(f"[LLM START]   call #{self.llm_call_count}")

      def on_llm_end(self, response, *, run_id, **kwargs):
          usage = response.llm_output.get("token_usage", {}) if response.llm_output else {}
          self.timeline.append(f"[LLM END]     tokens={usage}")

  4:
    task: "Override on_tool_start and on_tool_end"
    detail: >
      def on_tool_start(self, serialized, input_str, *, run_id, **kwargs):
          self.tool_call_count += 1
          name = serialized.get("name", "unknown")
          self.timeline.append(f"[TOOL START]  {name} | input={input_str[:40]}")

      def on_tool_end(self, output, *, run_id, **kwargs):
          self.timeline.append(f"[TOOL END]    output={str(output)[:40]}")

  5:
    task: "Add a summary method"
    detail: >
      def summary(self):
          print("\n=== Execution Timeline ===")
          for entry in self.timeline:
              print(entry)
          print(f"\nLLM calls: {self.llm_call_count}")
          print(f"Tool calls: {self.tool_call_count}")

  6:
    task: "Attach to your Stage 03 graph and run it"
    detail: >
      handler = DebugHandler()
      result = app.invoke(
          {"messages": [HumanMessage(content="What's the weather in Paris?")],
           "category": "", "answer": "", "quality_ok": False, "attempt_count": 0},
          config={"callbacks": [handler]}
      )
      handler.summary()

  7:
    task: "Run it three times with different inputs"
    detail: >
      - A weather question (goes through tool call)
      - A math question (goes through tool call, different tool)
      - A question that triggers the validator retry

      Compare the timelines. How does the event sequence differ?
```

---

## Key Observation Exercise

```yaml
exercise:
  task: "Map the timeline to the graph diagram from Stage 03"
  detail: >
    Take the printed timeline from a single run and draw a table:

    | Event          | Node/Component | run_id (first 8) | parent_run_id |
    |----------------|---------------|------------------|---------------|
    | CHAIN START    | Graph         | abc12345         | None          |
    | CHAIN START    | router_node   | def67890         | abc12345      |
    | LLM START      |               | ghi11111         | def67890      |
    ...

    This table IS the trace tree. Every OTel span in AgentTrace is one row of this table,
    enriched with timing, token counts, and structured attributes.
  why: >
    Once you see this table, the AgentTrace deep dive section on "span relationships,
    trace topology across multi-agent hops" is no longer abstract.
```

---

## Done When

```yaml
done_when:
  - DebugHandler runs and prints a readable timeline for every execution
  - You can name every event type that fires and describe what data it carries
  - You have completed the table-mapping exercise for at least one run
  - You can explain what run_id and parent_run_id represent and how they relate
  - You can write a handler that counts only tool calls to a specific tool by name
  - You understand what data would be needed to turn each event into an OTel span
  - LangSmith is connected, you have seen traces in the UI, and you can explain
    what it adds beyond your DebugHandler
```

---

## LangSmith Integration

Now that you understand what the callback system does, connect LangSmith — the industry-standard tracing tool built on exactly this mechanism. This section takes less than 30 minutes. The goal is to use it as a tool, not re-implement it.

```yaml
langsmith_setup:
  steps:
    1:
      task: "Create a LangSmith account and get an API key"
      detail: >
        Go to smith.langchain.com → sign up (free tier is sufficient)
        Settings → API Keys → Create new key
        Copy the key

    2:
      task: "Set environment variables"
      detail: >
        Add to your .env file:

        LANGCHAIN_TRACING_V2=true
        LANGCHAIN_API_KEY=your-langsmith-key
        LANGCHAIN_PROJECT=agentic-ramp   # project name in LangSmith UI

        That's it. No code changes required.
        LangChain/LangGraph automatically detects these env vars and sends traces.

    3:
      task: "Run your Stage 03 agent with LangSmith active"
      detail: >
        python agent.py   # same command as before, just with the env vars set

        Go to smith.langchain.com → your "agentic-ramp" project
        Find the run you just triggered.

        Explore:
          - The trace tree: expand each node to see inputs/outputs
          - The token usage panel: total tokens, cost estimate
          - The latency waterfall: time per node and LLM call
          - The message list: the full conversation history at each step

    4:
      task: "Compare LangSmith's trace to your DebugHandler output"
      detail: >
        Run the same agent with both your DebugHandler AND LangSmith active:
          config={"callbacks": [debug_handler]}   # your handler
          # LangSmith adds itself automatically via env vars

        Open both outputs side by side.
        Find the same run_id in both. Confirm the event sequence matches.
        Note what LangSmith shows that your handler doesn't:
          - Rendered inputs/outputs (not raw dicts)
          - Token costs
          - Latency per step
          - Link to share the trace

    5:
      task: "Tag a run and add metadata"
      detail: >
        app.invoke(
            input,
            config={
                "callbacks": [debug_handler],
                "tags": ["stage-04", "weather-test"],
                "metadata": {"test_case": "routing-weather", "temperature": 0},
            }
        )

        In LangSmith, filter runs by tag. Find your tagged run.
        This is the production pattern — tag runs by feature, environment, test case.
        Filtering by tag is how you compare "before vs after prompt change" in LangSmith.

langsmith_vs_langfuse:
  what: >
    LangSmith: LangChain's own product. Best integration with LangGraph. SaaS only.
    Langfuse: open-source alternative. Self-hostable. EU-friendly (data sovereignty).
    Both hook into the same callback system — switching between them is an env var change.
  recommendation: >
    Use LangSmith now (fastest to set up, best LangGraph support).
    Learn Langfuse in Stage 06 (multi-agent) — self-hosted is a real differentiator for EU roles.
```

---

## What This Unlocks

Stage 05 adds checkpointing and human-in-the-loop interrupts. Callbacks are the observation layer — checkpoints are the persistence layer. After this stage you'll be ready to understand how AgentTrace emits hitl_pause spans and how GuardRail pauses graphs waiting for approval.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - Are on_chain_start events fired for both the whole graph and each node individually?
    How do you tell them apart from the serialized argument?
  - What data is available in on_llm_end — specifically, where are token counts?
  - Can you attach multiple handlers at once? What's the use case for that?
  - What is the difference between synchronous callbacks (BaseCallbackHandler) and
    async callbacks (AsyncCallbackHandler)?
```
