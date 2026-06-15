# Stage 03 — Multi-Node Graphs and Conditional Routing

**Status:** not started
**Estimated time:** 2–3 days
**Depends on:** Stage 02 (you understand the tool call cycle and state schema)

Real agents don't just loop between one LLM and one tool set. They route — a router decides which specialist to send the task to, a validator checks if the output is good enough before the agent terminates, a planner decides the next step. This stage builds that routing logic and teaches you how graphs model decision-making.

---

## What You're Building

A three-node research agent:
1. **router_node** — classifies the user's question into a category (math / general knowledge / weather)
2. **specialist_node** — runs the right logic based on the category (mock tools per category)
3. **validator_node** — checks if the specialist's answer is complete; if not, routes back to specialist

The key artifact: a graph where the flow changes based on the content of state — not a fixed sequence.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Conditional edges:
    what: >
      Instead of a fixed edge (node A always goes to node B), a conditional edge is
      a function that reads the current state and returns the name of the next node.
      This is how routing works in every real agent.
    why: >
      AgentTrace records edge_traversal spans specifically because conditional edges
      are where the interesting decisions happen. AgentEval's assertions like
      expect(run).to_enter_branch("approved_path") are assertions on conditional edges.
      You cannot build either without deeply understanding this.

  add_conditional_edges():
    what: >
      Takes a source node and a routing function. The routing function returns
      a string (or END) that determines which node runs next.
    why: This is the API you will use for all routing logic going forward.

  END sentinel:
    what: A LangGraph constant. When a routing function returns END, the graph terminates.
    why: >
      Knowing when to stop is a real design decision. An agent that never returns END
      loops forever. Understanding how to model termination conditions is essential.

  State as routing signal:
    what: >
      Nodes write fields to state specifically so that routing functions can read them
      and make decisions. E.g., a node writes state["category"] = "math" and the
      router reads that field.
    why: >
      This is the pattern every multi-agent system uses. The supervisor reads a
      "next_agent" field from state. The validator reads a "quality_score" field.
      State is the communication channel between nodes — not return values.

  Graph visualization:
    what: app.get_graph().draw_mermaid_png() — renders the graph as a diagram.
    why: >
      Use this every time you build a new graph. Seeing the node/edge structure
      visually makes it immediately obvious if your routing logic is wired correctly.
      Also useful for explaining your architecture in portfolio write-ups.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define a richer state schema"
    detail: >
      class ResearchState(TypedDict):
          messages: Annotated[list, add_messages]
          category: str          # set by router_node
          answer: str            # set by specialist_node
          quality_ok: bool       # set by validator_node
          attempt_count: int     # track retries

  2:
    task: "Build router_node"
    detail: >
      A node that reads the last HumanMessage, calls the LLM to classify it
      (you can use a simple prompt: "Classify this question as math, weather,
      or general. Reply with only one word."), and writes the result to
      state["category"].

      Return {"category": classification_result}

  3:
    task: "Build specialist_node"
    detail: >
      A node that reads state["category"] and runs the appropriate logic.
      Math: call a calculator tool.
      Weather: call a mock weather tool.
      General: call LLM directly with no tools.
      Write the answer to state["answer"].

  4:
    task: "Build validator_node"
    detail: >
      A node that reads state["answer"] and asks the LLM to evaluate it:
      "Is this a complete and correct answer? Reply yes or no."
      Write True or False to state["quality_ok"].

  5:
    task: "Wire the conditional edges"
    detail: >
      def route_from_router(state):
          return state["category"]   # returns "math", "weather", or "general"

      def route_from_validator(state):
          if state["quality_ok"] or state["attempt_count"] >= 2:
              return END
          return "specialist"   # retry

      graph.add_conditional_edges("router", route_from_router, {
          "math": "specialist",
          "weather": "specialist",
          "general": "specialist"
      })
      graph.add_conditional_edges("validator", route_from_validator)

  6:
    task: "Visualize the graph"
    detail: >
      from IPython.display import Image
      Image(app.get_graph().draw_mermaid_png())

      Or save it: open("graph.png", "wb").write(app.get_graph().draw_mermaid_png())
      Open it. The routing structure should be visually obvious.

  7:
    task: "Test all three paths"
    detail: >
      Ask a math question, a weather question, and a general question.
      Confirm state["category"] is set correctly each time.
      Confirm the validator runs after the specialist.
```

---

## Prompt Engineering Section

The router_node and validator_node in this stage both depend on prompts you write. This is the first place in the ramp where prompt quality directly determines whether the graph routes correctly. Use this stage to build the habit of deliberate prompt design.

```yaml
prompt_engineering:
  core_principle: >
    A prompt is an interface contract between you and the LLM.
    Write it for the model, not for a human. Be explicit about format, constraints, and edge cases.

  router_prompt_iteration:
    task: "Iterate on the router_node system prompt until routing is reliable"
    starting_prompt: >
      "Classify this question as math, weather, or general. Reply with only one word."
    problems_you_will_hit:
      - LLM replies "Math" (capitalized) instead of "math" — conditional edge fails
      - LLM adds punctuation: "math." — conditional edge fails
      - Ambiguous question ("What is 100 degrees?") routes inconsistently
    fixes:
      - Add: "Your reply must be lowercase with no punctuation."
      - Add: "If the question could fit multiple categories, choose the most specific one."
      - Add few-shot examples to anchor the format:
          "Q: What is 2+2? → math
           Q: Is it raining in Paris? → weather
           Q: Who invented the telephone? → general"
    measure: >
      Run 10 test questions. If routing is wrong for any, fix the prompt and re-run.
      You are done when 10/10 route correctly at temperature=0.

  validator_prompt_iteration:
    task: "Write a validator prompt that gives reliable yes/no output"
    starting_prompt: >
      "Is this a complete and correct answer? Reply yes or no."
    problems_you_will_hit:
      - LLM replies "Yes, this is correct" — your parser breaks
      - LLM qualifies: "Mostly yes, but..." — ambiguous
    fix: >
      "Evaluate the following answer. Reply with exactly one word: yes or no.
       Do not add any other text."
    lesson: >
      Structured output prompts require explicit format constraints.
      "Reply with one word" is not enough — "Reply with exactly one word: yes or no" is.
      This pattern applies everywhere: any time you need parseable LLM output,
      spell out the exact format and valid values.

  temperature_discipline:
    what: >
      Use temperature=0 for any node that needs deterministic, parseable output.
      (router_node, validator_node, any node that writes to a routing field)
      Use temperature=0.7–1.0 for creative/generative nodes.
      (writer_node, summarizer_node, any node where variety is acceptable)
    why: >
      A routing node that outputs "Math" 9 times and "MATH" once at temperature=1
      causes a conditional edge failure that's hard to debug.
      Temperature=0 makes routing tests reliable.

  few_shot_examples:
    what: >
      Include 2–3 example input/output pairs inside the system prompt.
    when_to_use: >
      When the LLM's zero-shot behavior is inconsistent.
      When the output format is unusual or strict.
      When you need the LLM to follow a specific reasoning pattern.
    example: >
      system_prompt = """
      You classify questions into categories.

      Examples:
      Q: What is 15% of 200? → math
      Q: What's the weather like in Tokyo? → weather
      Q: Who wrote Hamlet? → general

      Reply with exactly one word: math, weather, or general.
      """

  prompt_versioning_habit:
    what: >
      Store your prompts as constants at the top of the file, not inline strings.
    why: >
      When you change a prompt, you change a constant — it's visible in git diff.
      When you have 10 nodes each with inline prompts, iteration becomes impossible to track.
    example: >
      # At the top of agent.py — not inside the node function

      ROUTER_SYSTEM_PROMPT = """
      You classify questions into categories.
      ...
      """

      VALIDATOR_SYSTEM_PROMPT = """
      You evaluate the quality of answers.
      ...
      """
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Make validator_node always set quality_ok=False and remove the attempt_count guard.
    What happens? How does LangGraph handle an infinite loop?
  - Add a fourth category "unknown" to router_node but don't add it to the
    conditional edge mapping. What error do you get?
  - What happens if router_node returns None for state["category"]?
  - Remove the "specialist" → "validator" edge. Does the graph compile? Does it run?
  - Set temperature=1 on the router LLM. Run the same question 5 times.
    How often does the routing change? This demonstrates why temperature=0 is required
    for routing nodes.
```

---

## Deliberate Design Exercise

```yaml
exercise:
  task: "Model a simple approval workflow as a LangGraph graph"
  spec: >
    Build a graph that:
    1. Drafts a short email based on a user prompt
    2. Routes to an approval_node that checks if the email is professional
    3. If yes → terminates and returns the email
    4. If no → routes back to the draft node with a "improve it" instruction

    No LLM calls required in approval_node — use simple string checks (does it
    contain profanity? is it longer than 10 words?). The point is the routing,
    not the intelligence.

  why: >
    This is the HITL (human-in-the-loop) pattern in simplified form. GuardRail
    and AgentResilience both use variants of this routing structure. Modeling it
    yourself first makes the deep dives readable.
```

---

## Done When

```yaml
done_when:
  - The three-node research agent routes correctly based on question type
  - You have visualized the graph and can read the diagram
  - You can add a new routing branch (new category) without breaking existing paths
  - You can explain conditional edges in plain language without looking at docs
  - You've built the approval workflow exercise
  - You can describe the difference between a fixed edge and a conditional edge
    and when you'd use each
```

---

## What This Unlocks

Stage 04 hooks the callback system — you'll instrument this exact graph to see every event fire in real time. Without the multi-node mental model, the callback event sequence is just noise.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - Can a conditional edge return a list of next nodes (parallel execution)?
  - What is the difference between add_edge() and add_conditional_edges()
    when the condition always returns the same value?
  - What happens to state fields that a node doesn't update — are they preserved?
  - Can you call app.invoke() on a graph that has no END — one that always loops?
```
