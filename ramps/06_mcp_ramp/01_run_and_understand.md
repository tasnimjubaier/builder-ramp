# Stage 01 — Run an MCP Server and Understand the Protocol

**Status:** not started
**Estimated time:** 1 day
**Type:** additive start
**Depends on:** Claude Desktop installed, basic Python

Before building anything, watch MCP work from the outside. Run an existing MCP server, connect it to Claude Desktop, call its tools through Claude, and inspect every message that flows between client and server. After this stage the protocol is not abstract — you've seen it run.

---

## What You're Doing

1. Install and run the official filesystem MCP server
2. Connect it to Claude Desktop
3. Ask Claude to read and write files through the MCP tools
4. Enable MCP logging and read the raw JSON-RPC messages

---

## Concepts This Stage Teaches

```yaml
concepts:
  MCP in one paragraph:
    what: >
      MCP is a client-server protocol where the server exposes tools, resources, and prompts,
      and the client (an LLM host like Claude Desktop or your own agent) discovers and calls them.
      The client sends a JSON-RPC request: "call tool X with args Y". The server runs the tool
      and returns the result. The LLM sees the result as context and continues reasoning.
    why: >
      Once you see this message flow in the logs, every MCP integration is the same pattern.
      Your LangGraph agent becomes an MCP client. Your Python functions become MCP tools.

  Tools vs Resources vs Prompts:
    what: >
      Tools:     functions the LLM can call (like LangChain tools) — have input schema, return output
      Resources: data the LLM can read (files, DB rows, API responses) — URI-addressable, read-only
      Prompts:   reusable prompt templates the LLM can invoke — parameterized system prompts
    why: >
      Most agentic use cases only need Tools. But knowing Resources and Prompts exist means you
      can read any MCP server's documentation and understand what it exposes.

  JSON-RPC over stdio:
    what: >
      MCP communicates over stdio (stdin/stdout) by default — not HTTP.
      The client launches the server as a subprocess and pipes JSON-RPC messages.
      An HTTP/SSE transport also exists for remote servers.
    why: >
      This is why MCP server config in Claude Desktop is a command to run, not a URL.
      The server is a subprocess of the client. Understanding this makes the config
      format obvious rather than mysterious.

  Tool discovery:
    what: >
      On connection, the client sends a tools/list request. The server responds with all
      available tools — name, description, and JSON Schema for the input.
      The LLM reads tool descriptions to decide when to call each one.
    why: >
      The tool description is the interface contract. Same principle as @tool docstrings
      in LangChain — the LLM reads it. Bad descriptions = tools that never get called.
```

---

## Setup

```yaml
requirements:
  - Claude Desktop: download from claude.ai/download
  - uvx: pip install uv  (or brew install uv)
  - The filesystem MCP server: uvx mcp-server-filesystem  (no separate install needed)
```

---

## Steps

```yaml
steps:
  1:
    task: "Configure Claude Desktop to use the filesystem MCP server"
    detail: >
      Open Claude Desktop settings (⌘, on Mac) → Developer → Edit Config.
      This opens claude_desktop_config.json. Add:

      {
        "mcpServers": {
          "filesystem": {
            "command": "uvx",
            "args": ["mcp-server-filesystem", "/tmp/mcp-test"]
          }
        }
      }

      Create the directory first: mkdir -p /tmp/mcp-test

      Restart Claude Desktop. You should see a hammer icon in the chat input — this
      means MCP tools are available.

  2:
    task: "Call MCP tools through Claude"
    detail: >
      In Claude Desktop, ask:
        "List all files in the MCP directory"
        "Create a file called hello.txt with the content 'Hello from MCP'"
        "Read the file hello.txt and tell me what's in it"

      Observe: Claude calls the list_directory, write_file, and read_file tools
      automatically. You can see which tool it called and the result in the chat.

  3:
    task: "Enable MCP logging and read the raw messages"
    detail: >
      MCP logs are written to ~/Library/Logs/Claude/ on Mac.
      Open a new terminal and tail the log:
        tail -f ~/Library/Logs/Claude/mcp*.log

      Make a tool call through Claude and watch the JSON-RPC messages scroll by.
      Find:
        - The tools/list request and response (tool discovery)
        - The tools/call request (tool invocation with arguments)
        - The tools/call response (tool result)

      Copy one complete tools/call exchange into a note. This is the protocol.

  4:
    task: "Read the tools/list response"
    detail: >
      In the log, find the tools/list response. It will look like:
      {
        "result": {
          "tools": [
            {
              "name": "read_file",
              "description": "Read the complete contents of a file...",
              "inputSchema": {
                "type": "object",
                "properties": { "path": { "type": "string" } },
                "required": ["path"]
              }
            },
            ...
          ]
        }
      }

      For each tool in the response, answer:
        - What does it do?
        - What input does it expect?
        - How would the LLM know when to call it?

  5:
    task: "Run the server directly and inspect its behavior"
    detail: >
      uvx mcp-server-filesystem /tmp/mcp-test

      The server starts and waits for JSON-RPC input on stdin.
      It won't do anything until a client connects — but this confirms it runs.

      Now look at the mcp-server-filesystem source code on GitHub:
      github.com/modelcontextprotocol/servers/tree/main/src/filesystem

      Find where it defines the read_file tool. Read the tool definition — name,
      description, inputSchema. This is what you'll write in Stage 02.
```

---

## Done When

```yaml
done_when:
  - Claude Desktop has the filesystem server connected and tools work
  - You have found and read at least one tools/call JSON-RPC exchange in the logs
  - You can explain the MCP protocol in plain language:
    client launches server → tools/list → LLM reads descriptions → tools/call → result
  - You understand the difference between Tools, Resources, and Prompts
  - You can explain why MCP uses stdio by default
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between MCP's stdio transport and SSE transport?
    When would you use each?
  - Can one MCP server expose both tools and resources? What would a resource look like
    in your agent use case?
  - How does the LLM decide which tool to call when multiple tools are available?
    Is this different from LangChain's bind_tools() pattern?
  - What happens if the MCP server crashes mid-tool-call? How does the client handle it?
```
