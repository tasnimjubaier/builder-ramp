# MCP Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 3
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_run_and_understand:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1 day"
    artifact: "filesystem MCP server connected to Claude Desktop, raw JSON-RPC log captured"
    notes: ""

  02_build_mcp_server:
    status: not_started
    started: ""
    completed: ""
    estimate: "2 days"
    artifact: "agent-tools-server with run_agent + search_docs tools, connected to Claude Desktop"
    notes: ""

  03_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "2–3 days"
    artifact: "MCP server exposing trajectory capture from mini-agenteval, public GitHub repo"
    notes: ""
```

---

## Prereqs

```yaml
prereqs:
  python_ramp:   "stages 01–06 required"
  agentic_ramp:  "stage 02 (tools) required for stage 02 of this ramp"
  rag_ramp:      "stage 04 (hybrid search) required for stage 02 of this ramp"
  claude_desktop: "download from claude.ai/download"
  uvx:           "pip install uv  or  brew install uv"
```

---

## Notes

<!-- freeform — add blockers, surprises, Claude tool call observations -->
