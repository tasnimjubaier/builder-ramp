# Stage 02 — Build a Custom MCP Server

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — extends Stage 01 setup
**Depends on:** Stage 01 (protocol understood) + Agentic Ramp (your LangGraph tools exist)

Now you build one. The MCP Python SDK makes this straightforward — you define tools as decorated functions, the SDK handles the JSON-RPC layer. The interesting work is in tool design: descriptions that make the LLM call them correctly, input schemas that validate sensibly, error responses that are readable.

---

## What You're Building

An MCP server called `agent-tools-server` that exposes three tools:
1. `run_agent(prompt, thread_id?)` — submits a prompt to your LangGraph agent and returns the result
2. `get_run_status(run_id)` — checks the status of a submitted run
3. `search_docs(query)` — calls your RAG pipeline's hybrid_search and returns top results

Connect it to Claude Desktop and test all three tools through Claude.

---

## Concepts This Stage Teaches

```yaml
concepts:
  MCP Python SDK (@mcp.tool):
    what: >
      from mcp.server import FastMCP
      mcp = FastMCP("my-server")

      @mcp.tool()
      def my_tool(param: str) -> str:
          \"\"\"Description the LLM reads to decide when to call this.\"\"\"
          return "result"
    why: >
      The decorator registers the function as an MCP tool. The function signature
      generates the JSON Schema automatically (same as @tool in LangChain).
      The docstring is the tool description — write it for the LLM, not for a human developer.

  Input schema design:
    what: >
      The types in your function signature become the JSON Schema the LLM receives.
      str → "type": "string". int → "type": "integer". Optional[str] = None → optional field.
    why: >
      The LLM uses this schema to construct its tool call arguments.
      Overly broad schemas (everything is str) lead to LLM guessing.
      Typed, descriptive parameter names and types guide the LLM to pass correct inputs.

  Error handling in MCP tools:
    what: >
      MCP tools should return errors as structured strings, not raise exceptions.
      raise Exception("something failed") propagates as an MCP protocol error.
      return "Error: something failed — reason: ..." gives the LLM readable context.
    why: >
      The LLM reads the tool result. An exception stack trace is noise.
      A structured error message lets the LLM reason about the failure and decide what to do next.

  Async MCP tools:
    what: >
      @mcp.tool() supports async def — the server runs an asyncio event loop.
    why: >
      Your LangGraph agent is async. Your RAG pipeline uses async HTTP clients.
      Async tools are the natural fit.
```

---

## Setup

```yaml
dependencies:
  python:
    - mcp[cli]    # MCP Python SDK with CLI

install: "pip install 'mcp[cli]'"
note: "Requires Python 3.10+"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Create server.py with FastMCP"
    detail: >
      from mcp.server.fastmcp import FastMCP

      mcp = FastMCP(
          "agent-tools-server",
          description="Tools for running LangGraph agents and searching documentation",
      )

  2:
    task: "Add the run_agent tool"
    detail: >
      import asyncio
      from langchain_core.messages import HumanMessage
      from your_agent import build_agent   # your agentic ramp agent

      _agent = None

      def get_agent():
          global _agent
          if _agent is None:
              _agent = build_agent()
          return _agent

      @mcp.tool()
      async def run_agent(prompt: str, thread_id: str = "") -> str:
          \"\"\"Run the LangGraph agent with the given prompt and return its response.
          Use this tool when the user wants to execute an AI agent task.
          Optionally provide a thread_id to continue an existing conversation.\"\"\"

          try:
              agent = get_agent()
              config = {}
              if thread_id:
                  config = {"configurable": {"thread_id": thread_id}}

              result = agent.invoke(
                  {"messages": [HumanMessage(content=prompt)]},
                  config=config,
              )
              return result["messages"][-1].content

          except Exception as e:
              return f"Error running agent: {str(e)}"

  3:
    task: "Add the search_docs tool"
    detail: >
      from your_rag_pipeline import hybrid_search   # from RAG ramp

      @mcp.tool()
      async def search_docs(query: str, top_k: int = 3) -> str:
          \"\"\"Search the documentation corpus for information relevant to the query.
          Returns the top matching text chunks with relevance scores.
          Use this when you need factual information from the documentation.\"\"\"

          try:
              results = hybrid_search(query, top_k=top_k)
              if not results:
                  return "No relevant documents found."

              formatted = []
              for i, r in enumerate(results, 1):
                  score = r.get("rrf_score", r.get("score", 0))
                  formatted.append(f"[{i}] (score: {score:.3f})\n{r['text']}")
              return "\n\n---\n\n".join(formatted)

          except Exception as e:
              return f"Error searching docs: {str(e)}"

  4:
    task: "Run the server and test it directly"
    detail: >
      mcp dev server.py

      This starts the MCP Inspector — a web UI at http://localhost:5173 that
      lets you call tools directly without Claude Desktop. Test each tool here first.

      In the Inspector:
        - Go to "Tools" tab
        - Click run_agent
        - Enter a prompt and call it
        - Verify the response

  5:
    task: "Connect to Claude Desktop"
    detail: >
      Add to claude_desktop_config.json:

      {
        "mcpServers": {
          "filesystem": { ... },
          "agent-tools": {
            "command": "python",
            "args": ["/absolute/path/to/server.py"],
            "env": {
              "OPENAI_API_KEY": "your-key"
            }
          }
        }
      }

      Restart Claude Desktop. In a new chat, ask:
        "Use the agent to write a haiku about Python"
        "Search the docs for information about LangGraph checkpointing"

      Watch Claude call your tools. Verify the results are correct.

  6:
    task: "Improve tool descriptions based on LLM behavior"
    detail: >
      Run 5 different prompts through Claude and observe:
        - Does Claude call the right tool for each prompt?
        - Does it pass correct arguments?
        - Are there cases where it calls the wrong tool or passes bad arguments?

      If yes, improve the docstring. The description is your interface contract with the LLM.
      Iterate until all 5 prompts produce correct tool calls.
```

---

## Done When

```yaml
done_when:
  - MCP Inspector shows all tools and they work correctly
  - Claude Desktop connects and calls tools in response to natural language
  - You have iterated on at least one tool description based on observed LLM behavior
  - Error cases (bad input, agent failure) return readable error strings, not exceptions
  - You can explain how the function signature becomes the JSON Schema the LLM sees
```

---

## Open Questions

```yaml
open_questions:
  - How would you add authentication to an MCP server — only allow specific clients?
  - What is the difference between an MCP tool and an MCP resource?
    Could search_docs be implemented as a resource instead of a tool?
  - How would you deploy this MCP server so it's accessible remotely (not just locally)?
    (Hint: SSE transport + a public URL)
  - What happens when a tool takes 30 seconds to run? Does MCP have a timeout mechanism?
```
