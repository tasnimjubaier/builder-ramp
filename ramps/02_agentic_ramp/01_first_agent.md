# Stage 01 — First Agent

**Status:** not started
**Estimated time:** 1–2 days
**Depends on:** nothing — this is the entry point

The only goal here is to make something run. A single LangGraph node that calls an LLM and returns an answer. No tools, no state schema, no conditional logic. Just enough to understand what LangGraph actually is and how it differs from calling an LLM directly.

---

## What You're Building

A Python script that:
1. Creates a LangGraph `StateGraph`
2. Defines one node — a function that calls an LLM
3. Compiles the graph
4. Invokes it with a message
5. Prints the response

That's it. The output is a working script that runs in under 10 seconds.

---

## Concepts This Stage Teaches

```yaml
concepts:
  StateGraph:
    what: The core LangGraph object. A graph where nodes are Python functions and edges connect them.
    why: Everything in LangGraph is a StateGraph. You cannot build anything without this.

  State:
    what: A typed dict (or Pydantic model) that flows through every node in the graph.
    why: Nodes don't talk to each other directly — they read from and write to shared state.
      Understanding this is the foundation for multi-node routing later.

  Node:
    what: A Python function that takes state as input and returns a state update as output.
    why: All logic in LangGraph lives in nodes. Every other concept (tools, memory, routing)
      is built on top of nodes.

  compile():
    what: Turns the graph definition into a runnable object.
    why: LangGraph separates definition from execution. compile() is the boundary.

  invoke():
    what: Runs the graph synchronously with a given input and returns the final state.
    why: The simplest way to execute a graph — you will use this constantly for testing.

  MessagesState:
    what: LangGraph's built-in state type — a list of LangChain messages.
    why: Almost every real agent uses this. Learn it here before you need it for tools.
```

---

## Setup

```yaml
dependencies:
  - langgraph
  - langchain-openai   # or langchain-google-genai for Gemini
  - openai             # raw API client — needed for the prerequisite exercise below
  - python-dotenv

env:
  OPENAI_API_KEY: "your key"   # or GOOGLE_API_KEY for Gemini

model_recommendation: "gpt-4o-mini or gemini-1.5-flash — cheapest capable models, cost is near zero"
```

Install:
```
pip install langgraph langchain-openai openai python-dotenv
```

---

## Prerequisite: Raw LLM API (Do This Before LangGraph)

Before using LangGraph's LLM wrappers, call the OpenAI API directly once. This grounds your understanding — LangGraph's `ChatOpenAI` is a convenience layer on top of exactly what you're about to write.

```yaml
exercise:
  task: "Call the OpenAI API directly in a plain Python file — no LangGraph, no LangChain"
  file: "raw_llm.py"

  steps:
    1:
      task: "Basic chat completion"
      detail: >
        from openai import OpenAI

        client = OpenAI()   # reads OPENAI_API_KEY from env automatically

        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[
                {"role": "system", "content": "You are a concise assistant."},
                {"role": "user",   "content": "What is Python in one sentence?"},
            ],
            temperature=0,
        )

        print(response.choices[0].message.content)

        # Inspect the full response object — read every field
        print(response.model)
        print(response.usage.prompt_tokens)
        print(response.usage.completion_tokens)
        print(response.usage.total_tokens)

    2:
      task: "Streaming response"
      detail: >
        # Streaming: get chunks as they arrive instead of waiting for full response
        stream = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": "Count from 1 to 10 slowly."}],
            stream=True,
        )

        for chunk in stream:
            delta = chunk.choices[0].delta.content
            if delta:
                print(delta, end="", flush=True)
        print()   # newline at end

        # What is the type of chunk? What fields does it have?
        # When is delta None vs a string?

    3:
      task: "Token counting and cost estimation"
      detail: >
        # Run 3 prompts of different lengths. Record token counts each time.
        prompts = [
            "Hi",
            "Explain what a large language model is in 50 words.",
            "Write a detailed technical explanation of how transformer attention works, "
            "covering the query-key-value mechanism, multi-head attention, and why "
            "positional encoding is necessary.",
        ]

        for prompt in prompts:
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=[{"role": "user", "content": prompt}],
            )
            usage = response.usage
            cost = (usage.prompt_tokens * 0.00015 + usage.completion_tokens * 0.0006) / 1000
            print(f"prompt_tokens={usage.prompt_tokens:4d}  "
                  f"completion_tokens={usage.completion_tokens:4d}  "
                  f"cost=${cost:.6f}")

        # Rule of thumb: ~4 characters per token for English text.
        # Verify this against your actual token counts.

    4:
      task: "Multi-turn conversation (message history)"
      detail: >
        # The API has no memory — you must send the full history every time
        messages = [{"role": "system", "content": "You are a helpful assistant."}]

        def chat(user_input: str) -> str:
            messages.append({"role": "user", "content": user_input})
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=messages,
            )
            reply = response.choices[0].message.content
            messages.append({"role": "assistant", "content": reply})
            return reply

        print(chat("My name is Alex."))
        print(chat("What is my name?"))   # should remember "Alex"
        print(f"Total messages in history: {len(messages)}")

        # Key insight: the LLM doesn't "remember" — you send the history each time.
        # LangGraph's MessagesState is just a structured way to manage this list.

    5:
      task: "Rate limit and error handling"
      detail: >
        from openai import RateLimitError, APIError
        import time

        def call_with_retry(prompt: str, max_retries: int = 3) -> str:
            for attempt in range(1, max_retries + 1):
                try:
                    response = client.chat.completions.create(
                        model="gpt-4o-mini",
                        messages=[{"role": "user", "content": prompt}],
                    )
                    return response.choices[0].message.content
                except RateLimitError:
                    wait = 2 ** attempt
                    print(f"Rate limited. Waiting {wait}s (attempt {attempt}/{max_retries})")
                    time.sleep(wait)
                except APIError as e:
                    print(f"API error: {e.status_code} — {e.message}")
                    raise
            raise RuntimeError("Max retries exceeded")

        # You won't actually hit a rate limit at this scale — but understanding the
        # exception types means you know what LangGraph wrappers are catching for you.

  done_when:
    - You have called the raw API and seen a response
    - You understand what response.usage contains and how to estimate cost
    - You have seen streaming chunks arrive one by one
    - You understand that the LLM has no memory — the message list is the memory
    - You can explain what ChatOpenAI in LangChain is doing under the hood
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create a file agent.py"
    detail: "Start with an empty file — don't copy from docs yet"

  2:
    task: "Import and define a State class"
    detail: >
      from langgraph.graph import StateGraph, MessagesState
      Use MessagesState as your state for now — it gives you a messages list out of the box.

  3:
    task: "Define a node function called llm_node"
    detail: >
      It takes state (a dict with a 'messages' key) and returns a dict with an updated messages key.
      Inside it, call your LLM and append the response to the messages list.

  4:
    task: "Build and compile the graph"
    detail: >
      graph = StateGraph(MessagesState)
      graph.add_node("llm", llm_node)
      graph.set_entry_point("llm")
      graph.set_finish_point("llm")
      app = graph.compile()

  5:
    task: "Invoke the graph"
    detail: >
      result = app.invoke({"messages": [HumanMessage(content="What is 2 + 2?")]})
      print(result["messages"][-1].content)

  6:
    task: "Run it"
    detail: "python agent.py — it should print '4' or equivalent"
```

---

## What to Break On Purpose

Once it works, do this:

```yaml
deliberate_breaks:
  - Remove set_entry_point() and run it. Read the error. Understand why it fails.
  - Return a string instead of a dict from llm_node. Read the error. Understand state typing.
  - Add a second node but don't connect it. Try to invoke. What happens?
  - Call app.invoke() without the messages key. Read the error.
```

Breaking things deliberately is how you build the mental model. Each error teaches you a constraint you won't forget.

---

## Done When

```yaml
done_when:
  - The script runs and returns a valid LLM response
  - You can explain in plain language what StateGraph, node, and invoke() do
  - You have broken it in at least 2 ways and understand why each one fails
  - You can rebuild it from a blank file without looking at your previous version
```

---

## What This Unlocks

Stage 02 adds tools and a real state schema. To understand why tools work the way they do in LangGraph, you need the mental model of state-as-shared-dict that this stage builds. Without this, tool call mechanics look like magic.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between invoke() and stream()?
  - What does MessagesState actually contain — what keys, what types?
  - What happens to state between node calls — is it copied or mutated?
```
