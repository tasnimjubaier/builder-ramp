# Stage 05 — RAG Inside a LangGraph Node

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — connects RAG ramp to agentic ramp
**Depends on:** Stage 04 (hybrid retrieval built) + Agentic Ramp Stage 02 (tool-calling agent)

Retrieval doesn't run in isolation in production — it runs inside an agent. The agent decides when to retrieve, what to query, and whether to retrieve again if the first result wasn't good enough. This stage wires your hybrid retrieval pipeline into a LangGraph tool-calling agent.

---

## What You're Building

A LangGraph agent with two tools:
1. `retrieve(query)` — calls your hybrid_search from Stage 04, returns top 3 chunks
2. `generate_answer(context, question)` — takes retrieved context and generates a final answer

The agent decides when to call retrieve, can call it multiple times with refined queries, and uses the results to produce a grounded answer. This is the agentic RAG pattern — not just one retrieval call, but iterative retrieval.

---

## Concepts This Stage Teaches

```yaml
concepts:
  RAG as a tool:
    what: >
      Instead of hardcoding retrieval into a pipeline (retrieve → generate, always),
      expose retrieval as a tool the LLM can choose to call. The LLM can then decide
      when to retrieve, what to query, and whether to iterate.
    why: >
      Simple RAG (always retrieve once) is brittle — if the first retrieval misses,
      the LLM generates a wrong or hallucinated answer. Agentic RAG lets the LLM
      recognize insufficient context and retrieve again with a better query.
      This is the pattern in AgentMemory, GuardRail, and any serious RAG system.

  Query rewriting:
    what: >
      The LLM reformulates the user's question into a better retrieval query before calling
      retrieve(). "How do I use the add_messages thing?" becomes "add_messages reducer
      LangGraph state annotation usage."
    why: >
      User questions are often vague or conversational. Retrieval queries should be
      dense with keywords. The LLM is good at this translation — letting it happen
      naturally inside tool call arguments is the simplest way to implement it.

  Grounded generation:
    what: >
      The LLM only answers from the retrieved context — it is instructed not to use
      its training data. The system prompt says "Only answer based on the provided context."
    why: >
      This is what reduces hallucination in RAG. Without this instruction, the LLM
      mixes retrieved context with training data and the blend is undetectable.
      RAGAS's faithfulness metric measures exactly this — are all claims in the answer
      grounded in the retrieved context?

  Multi-turn RAG:
    what: >
      Using LangGraph checkpointing (from Agentic Ramp Stage 05), a RAG agent can
      maintain conversation history across multiple questions. Later questions can
      reference earlier answers. The retrieval adapts to the conversation context.
    why: >
      Single-turn RAG answers one question. Multi-turn RAG is a knowledge assistant
      — the full value proposition of most AI product use cases.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Wrap hybrid_search as a LangGraph tool"
    detail: >
      from langchain_core.tools import tool

      @tool
      def retrieve(query: str) -> str:
          """Search the documentation for relevant information about the given query.
          Use this tool whenever you need factual information to answer a question.
          Reformulate the query to be dense with keywords relevant to the topic."""

          results = hybrid_search(query, top_k=3)
          if not results:
              return "No relevant documents found."

          formatted = []
          for i, r in enumerate(results, 1):
              formatted.append(f"[{i}] {r['text']}")
          return "\n\n".join(formatted)

      The docstring is the tool description the LLM reads — make it specific
      about when to use the tool and how to format the query.

  2:
    task: "Build the RAG agent graph"
    detail: >
      from langchain_openai import ChatOpenAI
      from langgraph.prebuilt import ToolNode, tools_condition
      from langgraph.graph import StateGraph, MessagesState

      llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
      llm_with_tools = llm.bind_tools([retrieve])

      SYSTEM_PROMPT = """You are a documentation assistant. Answer questions using ONLY
      the information retrieved via the retrieve tool. If you cannot find sufficient
      information after 2 retrieval attempts, say so explicitly. Do not use your training data."""

      def llm_node(state: MessagesState):
          messages = [SystemMessage(content=SYSTEM_PROMPT)] + state["messages"]
          response = llm_with_tools.invoke(messages)
          return {"messages": [response]}

      tool_node = ToolNode([retrieve])

      graph = StateGraph(MessagesState)
      graph.add_node("llm", llm_node)
      graph.add_node("tools", tool_node)
      graph.set_entry_point("llm")
      graph.add_conditional_edges("llm", tools_condition)
      graph.add_edge("tools", "llm")

      from langgraph.checkpoint.memory import MemorySaver
      app = graph.compile(checkpointer=MemorySaver())

  3:
    task: "Test single-turn RAG"
    detail: >
      from langchain_core.messages import HumanMessage

      config = {"configurable": {"thread_id": "rag-test-1"}}
      result = app.invoke(
          {"messages": [HumanMessage(content="How does LangGraph handle state persistence?")]},
          config=config,
      )

      # Print full message history to see: HumanMessage → AIMessage(tool_call) → ToolMessage(results) → AIMessage(answer)
      for msg in result["messages"]:
          print(f"{type(msg).__name__}: {str(msg.content)[:120]}")

  4:
    task: "Test multi-turn RAG"
    detail: >
      # Same thread_id — conversation continues from above
      result2 = app.invoke(
          {"messages": [HumanMessage(content="And how does the MemorySaver fit into that?")]},
          config=config,
      )

      The agent should retrieve again and connect the answer to the previous context.
      The second question ("And how does...") is deliberately vague — the agent should
      use conversation context to form a good retrieval query.

  5:
    task: "Observe iterative retrieval"
    detail: >
      Ask a question that requires multiple retrieval calls:
      "Compare how checkpointing works in LangGraph vs how tools work — what's similar?"

      Print the message sequence. Count how many retrieve tool calls the agent made.
      Did it retrieve once with a broad query or twice with focused queries?

      If it only retrieved once, modify the system prompt to encourage multiple
      targeted queries before answering.
```

---

## Done When

```yaml
done_when:
  - The RAG agent retrieves relevant chunks and generates grounded answers
  - Multi-turn conversation maintains context across questions
  - You can observe the full message sequence (HumanMessage → tool_call → ToolMessage → answer)
  - You've seen the agent make multiple retrieval calls on a complex question
  - You understand what "grounded generation" means and how the system prompt enforces it
  - You can explain why this is called "agentic RAG" vs "naive RAG"
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between "retrieve then read" (naive RAG) and "read then retrieve"
    (flipped RAG, where the LLM generates a hypothetical answer first, then retrieves similar docs)?
    When would you use each?
  - How would you prevent the agent from calling retrieve in an infinite loop?
    (Hint: loop bound from Agentic Ramp Stage 03)
  - What is "self-RAG" and how does it differ from the simple retrieve-then-answer pattern?
  - If the user asks a question the corpus doesn't cover, what should the agent do?
    How would you test this behavior?
```
