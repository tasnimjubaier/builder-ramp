# MCP Ramp — Learning Sequence

MCP (Model Context Protocol) is now required in multiple agentic AI engineer JDs. It's a standardized protocol for exposing tools to LLMs — any MCP-compatible client (Claude Desktop, Cursor, your own agent) can connect to any MCP server and use its tools. The engineering is narrow: understand the protocol, run an existing server, build your own.

This ramp is additive. Three stages, same MCP server growing in capability.

---

## Structure

```yaml
stages:
  01_run_and_understand:  "Run an existing MCP server. Understand the protocol message flow."
  02_build_mcp_server:    "Build a custom MCP server exposing your LangGraph agent tools."
  03_capstone:            "MCP server for mini-agenteval — expose trajectory capture as an MCP tool."
```

---

## Completion Criteria

```yaml
done_when:
  - You can explain the MCP protocol in one paragraph (client, server, tools, resources)
  - You have built a custom MCP server that exposes at least 2 tools
  - Your MCP server connects to Claude Desktop and the tools work
  - You understand the difference between MCP tools, resources, and prompts
  - You have a publicly linked MCP server repo
```

---

## Approach

```yaml
rules:
  - Use the Python MCP SDK (mcp) — it's the simplest path given your Python stack
  - Start by running someone else's server before building your own
  - Treat MCP as a deployment target, not a new framework — your tools already exist
  - The capstone connects MCP to your existing OSS work (mini-agenteval)
```
