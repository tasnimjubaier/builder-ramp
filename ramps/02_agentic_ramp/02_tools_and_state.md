# Stage 02 — Tools and State

**Status:** not started
**Estimated time:** 2–3 days
**Depends on:** Stage 01 (you can build a single-node graph and invoke it)

This is where agents become agents. A tool is just a Python function the LLM can decide to call. The LLM doesn't run the function — it outputs a structured request saying "call this function with these arguments", and your code executes it. Understanding this cycle is the single most important concept in agentic AI.

---

## What You're Building

A two-node LangGraph agent that:
1. Receives a user message
2. Decides whether to call a tool (a simple web search mock or calculator)
3. Calls the tool and gets a result back
4. Uses the result to generate a final response

The tool can be fake — a Python function that returns hardcoded data. The behavior of the loop is what matters, not the tool's output.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Tool calling mechanics:
    what: >
      The LLM returns a special response with tool_calls — a list of structured
      requests (tool name + JSON arguments). Your code inspects this, runs the function,
      and feeds the result back as a ToolMessage. The LLM then sees the result and
      continues reasoning.
    why: >
      This is the core of every agentic system. Every OSS project in v0.4 —
      AgentTrace, AgentEval, AgentResilience — instruments or evaluates this
      loop. If you don't understand the cycle, nothing in those deep dives makes sense.

  @tool decorator:
    what: LangChain's decorator that turns a Python function into a tool the LLM can call.
    why: >
      It handles the JSON schema generation from your function signature automatically.
      The LLM needs a schema to know what arguments to pass — @tool generates that.

  ToolNode:
    what: A pre-built LangGraph node that receives tool call requests and executes the tools.
    why: >
      You could write this node yourself — a function that reads tool_calls from state,
      runs the matching function, and returns ToolMessages. ToolNode does this for you.
      Build it manually once (Stage 02 deliberate breaks) then use ToolNode after.

  tools_condition:
    what: >
      A pre-built conditional edge function. Returns "tools" if the last message has
      tool_calls, returns END if it doesn't.
    why: >
      This is the routing logic that makes the agent loop: LLM node → check if tool call
      → if yes, run tools → back to LLM node. Without this edge the graph always terminates
      after the first LLM call.

  Custom State schema:
    what: A TypedDict with typed fields, not just MessagesState.
    why: >
      Real agents carry more than messages — a context variable, a counter, a mode flag.
      Defining a proper state schema is how you model what the agent tracks over its lifetime.
      This is what the agent.* attributes in AgentTrace are reflecting.
```

---

## Setup

Same as Stage 01 — no new dependencies for a mock tool. If you want a real tool:

```yaml
optional_dependencies:
  - langchain-community   # for pre-built tools like DuckDuckGo search, Wikipedia
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define a real state schema (not just MessagesState)"
    detail: >
      from typing import TypedDict, Annotated
      from langgraph.graph.message import add_messages

      class AgentState(TypedDict):
          messages: Annotated[list, add_messages]
          tool_call_count: int   # track how many tool calls have been made

      The Annotated[list, add_messages] is the LangGraph pattern for append-only
      message lists. add_messages is a reducer — it merges rather than overwrites.

  2:
    task: "Define a mock tool with @tool"
    detail: >
      from langchain_core.tools import tool

      @tool
      def get_weather(city: str) -> str:
          \"\"\"Get the current weather for a city.\"\"\"
          return f"It is sunny and 22°C in {city}."

      The docstring is critical — the LLM reads it to decide when to call this tool.

  3:
    task: "Bind the tool to your LLM"
    detail: >
      llm = ChatOpenAI(model="gpt-4o-mini")
      llm_with_tools = llm.bind_tools([get_weather])

      Now the LLM knows about get_weather and can choose to call it.

  4:
    task: "Build a two-node graph: llm_node → ToolNode"
    detail: >
      from langgraph.prebuilt import ToolNode, tools_condition

      tool_node = ToolNode([get_weather])

      graph = StateGraph(AgentState)
      graph.add_node("llm", llm_node)
      graph.add_node("tools", tool_node)
      graph.set_entry_point("llm")
      graph.add_conditional_edges("llm", tools_condition)
      graph.add_edge("tools", "llm")   # after tools, always go back to llm
      app = graph.compile()

  5:
    task: "Invoke with a question that requires the tool"
    detail: >
      result = app.invoke({"messages": [HumanMessage(content="What's the weather in Tokyo?")],
                           "tool_call_count": 0})
      for msg in result["messages"]:
          print(type(msg).__name__, ":", msg.content[:80])

  6:
    task: "Inspect the full message list"
    detail: >
      Print every message in result["messages"] with its type.
      You will see: HumanMessage → AIMessage (with tool_calls) → ToolMessage → AIMessage (final answer).
      This sequence IS the agent execution loop. Memorize it.
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Ask a question that doesn't need the tool ("What is 2 + 2?"). Does it still call the tool?
    What does tools_condition return? What does the message list look like?
  - Remove the docstring from @tool. Does the LLM still call it? Does behavior change?
  - Remove graph.add_edge("tools", "llm"). What happens when the tool runs?
  - Change the tool to raise an Exception. How does LangGraph handle a tool failure?
  - Update tool_call_count in llm_node by reading it from state and returning the incremented value.
    Confirm it increments correctly across multiple tool calls.
```

---

## Manual ToolNode Exercise

Before moving on, write ToolNode yourself once:

```yaml
manual_exercise:
  task: "Replace ToolNode with your own tool_executor_node"
  detail: >
    Write a node function that:
    1. Gets the last message from state
    2. Checks if it has tool_calls
    3. For each tool call, finds the matching function and calls it with the provided args
    4. Returns a list of ToolMessages with the results

    Then swap your version in for ToolNode and confirm the graph still works.
    This is what ToolNode does under the hood — writing it yourself means you'll
    never be confused by what it's doing again.
```

---

## Done When

```yaml
done_when:
  - The graph correctly routes to the tool when needed and skips it when not
  - You can print the full message list and explain what every message type means
  - You've manually implemented ToolNode once
  - You can explain the tool call cycle in plain language: LLM outputs tool_calls → your code runs
    the function → ToolMessage goes back to LLM → LLM continues
  - You can add a second tool and the LLM picks the right one based on the question
```

---

## What This Unlocks

Stage 03 adds multiple nodes and conditional edges — the graph stops being a simple loop and becomes a real decision structure. You need the tool call mental model from this stage to understand why routing decisions matter.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between bind_tools() and passing tools= to the constructor?
  - What happens if the LLM calls a tool that isn't in your tools list?
  - Can a single LLM response request multiple tool calls at once? Test it.
  - What does add_messages do differently from a plain list assignment?
```
