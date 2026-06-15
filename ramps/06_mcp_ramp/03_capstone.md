# Stage 03 — Capstone: MCP Server for mini-agenteval

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — connects MCP to your OSS work
**Depends on:** Stage 02 + Agentic Ramp Stage 10 (mini-agenteval built)

This capstone turns mini-agenteval into an MCP server. Any MCP-compatible client — Claude Desktop, Cursor, your own LangGraph agent — can now call trajectory capture, run assertions, and retrieve golden run comparisons as tools. This is the "MCP as a deployment target for your OSS tools" pattern.

It's also a portfolio signal: you've taken a Python library you wrote and exposed it via MCP — demonstrating both the library and the protocol in one repo.

---

## What You're Building

`mini-agenteval-mcp` — an MCP server that exposes:
1. `capture_trajectory(agent_name, prompt)` — runs an agent, captures its trajectory, returns a summary
2. `assert_trajectory(trajectory_json, assertion)` — runs a single assertion on a trajectory
3. `compare_to_golden(agent_name, prompt, golden_path)` — runs the agent and compares to a saved golden run
4. `list_golden_runs()` — lists saved golden runs available for comparison

---

## Build Steps

```yaml
steps:
  1:
    task: "Create mcp_server.py in the mini-agenteval repo"
    detail: >
      from mcp.server.fastmcp import FastMCP
      from mini_agenteval import Trajectory, TrajectoryCallbackHandler
      from mini_agenteval.assertions import expect
      import json

      mcp = FastMCP(
          "mini-agenteval",
          description="Trajectory capture and assertion tools for LangGraph agents",
      )

  2:
    task: "Add capture_trajectory tool"
    detail: >
      @mcp.tool()
      async def capture_trajectory(agent_name: str, prompt: str) -> str:
          \"\"\"Run a LangGraph agent and capture its execution trajectory.
          Returns a JSON summary of nodes visited, tools called, and termination.
          Use this to inspect what an agent actually did during a run.

          Args:
              agent_name: Name of the registered agent to run ('research', 'rag', 'supervisor')
              prompt: The input prompt to send to the agent
          \"\"\"
          try:
              agent = get_agent(agent_name)   # registry of your ramp agents
              trajectory = Trajectory()
              handler = TrajectoryCallbackHandler(trajectory)
              from langchain_core.messages import HumanMessage
              agent.invoke(
                  {"messages": [HumanMessage(content=prompt)]},
                  config={"callbacks": [handler]},
              )
              return json.dumps({
                  "nodes_visited": trajectory.nodes_visited,
                  "tool_calls": [tc.tool_name for tc in trajectory.tool_calls],
                  "terminal_node": trajectory.terminal_node,
                  "tool_call_count": len(trajectory.tool_calls),
              }, indent=2)
          except Exception as e:
              return f"Error capturing trajectory: {str(e)}"

  3:
    task: "Add assert_trajectory tool"
    detail: >
      @mcp.tool()
      async def assert_trajectory(trajectory_json: str, assertion_type: str, value: str) -> str:
          \"\"\"Run a single assertion on a captured trajectory.
          Returns 'PASS' with details or 'FAIL' with the reason.

          Args:
              trajectory_json: JSON string from capture_trajectory
              assertion_type: One of: 'called_tool', 'not_called_tool',
                              'visited_node', 'max_tool_calls', 'terminal_node'
              value: The value to check against (tool name, node name, or count as string)
          \"\"\"
          try:
              data = json.loads(trajectory_json)
              t = Trajectory(
                  nodes_visited=data["nodes_visited"],
                  tool_calls=[type("TC", (), {"tool_name": n})() for n in data["tool_calls"]],
                  terminal_node=data["terminal_node"],
              )
              e = expect(t)
              if assertion_type == "called_tool":
                  e.to_call_tool(value)
              elif assertion_type == "not_called_tool":
                  e.not_to_call_tool(value)
              elif assertion_type == "visited_node":
                  e.assert_visited_node(value)
              elif assertion_type == "max_tool_calls":
                  e.not_to_loop_more_than(int(value))
              elif assertion_type == "terminal_node":
                  e.to_terminate_at_node(value)
              else:
                  return f"Unknown assertion type: {assertion_type}"
              return f"PASS: {assertion_type}({value})"
          except AssertionError as ae:
              return f"FAIL: {str(ae)}"
          except Exception as e:
              return f"Error: {str(e)}"

  4:
    task: "Add list_golden_runs and compare_to_golden tools"
    detail: >
      import os

      @mcp.tool()
      async def list_golden_runs() -> str:
          \"\"\"List all saved golden runs available for comparison.
          Returns names and file paths of saved trajectories.\"\"\"
          golden_dir = "golden_runs"
          if not os.path.exists(golden_dir):
              return "No golden runs directory found. Run capture_trajectory first."
          files = [f for f in os.listdir(golden_dir) if f.endswith(".json")]
          if not files:
              return "No golden runs saved yet."
          return "\n".join(files)

      @mcp.tool()
      async def compare_to_golden(agent_name: str, prompt: str, golden_name: str) -> str:
          \"\"\"Run an agent and compare its trajectory to a saved golden run.
          Returns a diff showing what changed between the golden and current run.

          Args:
              agent_name: Name of the agent to run
              prompt: Input prompt (should match the golden run's prompt)
              golden_name: Filename of the golden run (from list_golden_runs)
          \"\"\"
          try:
              # Capture current run
              current_json = await capture_trajectory(agent_name, prompt)
              current = json.loads(current_json)

              # Load golden run
              golden_path = os.path.join("golden_runs", golden_name)
              with open(golden_path) as f:
                  golden = json.load(f)

              # Diff
              diffs = []
              if current["nodes_visited"] != golden.get("nodes_visited"):
                  diffs.append(f"nodes_visited changed:\n  golden:  {golden.get('nodes_visited')}\n  current: {current['nodes_visited']}")
              if current["tool_calls"] != golden.get("tool_calls"):
                  diffs.append(f"tool_calls changed:\n  golden:  {golden.get('tool_calls')}\n  current: {current['tool_calls']}")
              if current["terminal_node"] != golden.get("terminal_node"):
                  diffs.append(f"terminal_node changed:\n  golden:  {golden.get('terminal_node')}\n  current: {current['terminal_node']}")

              if not diffs:
                  return "PASS: Trajectory matches golden run exactly."
              return "DIFF:\n" + "\n\n".join(diffs)

          except FileNotFoundError:
              return f"Golden run '{golden_name}' not found. Use list_golden_runs to see available runs."
          except Exception as e:
              return f"Error: {str(e)}"

  5:
    task: "Test through Claude Desktop"
    detail: >
      Add to claude_desktop_config.json:
      {
        "mcpServers": {
          "mini-agenteval": {
            "command": "python",
            "args": ["/path/to/mini-agenteval/mcp_server.py"],
            "env": { "OPENAI_API_KEY": "your-key" }
          }
        }
      }

      In Claude Desktop, try:
        "Capture the trajectory of the research agent with prompt 'What is LangGraph?'"
        "Assert that the trajectory called the 'mock_search' tool"
        "List all golden runs"
        "Compare the research agent's current run to golden_run_001.json"

      Iterate on tool descriptions until all four work correctly.

  6:
    task: "Write the README"
    detail: >
      - What it is: mini-agenteval exposed as MCP tools
      - Installation: how to add to Claude Desktop config
      - Tools reference: all 4 tools with parameter descriptions
      - Example Claude conversation showing all tools in use

      Link this repo from the mini-agenteval README.
      Two repos that reference each other signal ecosystem thinking, not isolated demos.
```

---

## Done When

```yaml
done_when:
  - All 4 tools work through Claude Desktop
  - compare_to_golden correctly identifies when a trajectory has changed
  - README is clean and linkable
  - You can explain the architectural decision: why MCP over a REST API for this use case
  - You can connect any MCP-compatible client to this server, not just Claude Desktop

graduation_test: >
  Read the MCP section in your consolidated_skills.md JD scrape.
  Can you describe — from experience, not docs — exactly what MCP is, what you built
  with it, and what problem it solves? If yes, you have the answer to the MCP
  interview question at any agentic AI engineer role.
```
