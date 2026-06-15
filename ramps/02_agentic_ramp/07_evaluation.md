# Stage 07 — Agent Evaluation

**Status:** not started
**Estimated time:** 3 days
**Depends on:** Stage 06 (you have a multi-agent system with real behavior to evaluate)

Output quality is easy to evaluate — did the final answer match the expected answer? Trajectory quality is harder and more valuable: did the agent take the right path to get there? This stage builds the evaluation layer that AgentEval is based on — trajectory capture, assertion, and a CI-ready pass/fail result.

---

## What You're Building

A hand-rolled evaluation harness for the Stage 06 supervisor agent:
1. A `Trajectory` object that records what happened during a run
2. A set of assertion functions that check the trajectory
3. A pytest test file that runs the agent against golden inputs and asserts the trajectory
4. A `golden_run` capture script that saves a reference run for regression testing

---

## Concepts This Stage Teaches

```yaml
concepts:
  Trajectory:
    what: >
      A structured record of what an agent did during a run: which nodes fired,
      in what order, which tools were called, which branches were taken, and what
      the final output was.
    why: >
      This is the data model AgentEval's assertion DSL operates on.
      expect(run).to_call_tool("search").before("summarize") is just a method
      on a Trajectory object. Building Trajectory from scratch teaches you exactly
      what AgentEval captures and why.

  Trajectory capture via callbacks:
    what: >
      A CallbackHandler (from Stage 04) that records node names, tool names, and
      edge decisions into a Trajectory object as the graph runs.
    why: >
      This is exactly how AgentEval captures trajectories — same callback system,
      just writing to a structured object instead of printing. Stage 04 was the
      prerequisite.

  Golden run:
    what: >
      A saved Trajectory from a run you've manually verified as correct.
      Future runs are compared against the golden run.
    why: >
      This is regression testing for agents. If you change the prompt and the
      agent suddenly calls tools in a different order, the golden run comparison
      catches it — even if the final answer looks correct.

  pytest for agent assertions:
    what: >
      Writing your trajectory assertions as pytest test functions so they
      can run in CI and produce pass/fail results.
    why: >
      AgentEval's CI gate runs pytest under the hood. Understanding how to
      structure agent tests as pytest functions means you can use AgentEval
      immediately — and understand what it's doing.

  Determinism problem:
    what: >
      LLM outputs are non-deterministic. The same input can produce different
      tool call sequences on different runs.
    why: >
      This is the real engineering challenge in agent eval. You will encounter
      it in this stage. The solutions (temperature=0, mock LLM, soft assertions)
      are important to know before building a production eval system.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the Trajectory dataclass"
    detail: >
      from dataclasses import dataclass, field
      import json

      @dataclass
      class ToolCall:
          tool_name: str
          arguments: dict
          result: str

      @dataclass
      class NodeExecution:
          node_name: str
          tool_calls: list[ToolCall] = field(default_factory=list)

      @dataclass
      class Trajectory:
          nodes_visited: list[str] = field(default_factory=list)
          node_executions: list[NodeExecution] = field(default_factory=list)
          tool_calls: list[ToolCall] = field(default_factory=list)
          terminal_node: str = ""
          final_output: str = ""

          def save(self, path: str):
              with open(path, "w") as f:
                  json.dump(self.__dict__, f, default=str, indent=2)

          @classmethod
          def load(cls, path: str) -> "Trajectory":
              with open(path) as f:
                  return cls(**json.load(f))

  2:
    task: "Build TrajectoryCallbackHandler"
    detail: >
      A CallbackHandler (from Stage 04) that populates a Trajectory object.
      on_chain_start: append node name to trajectory.nodes_visited
      on_tool_start: append ToolCall to trajectory.tool_calls
      on_chain_end (for the final graph chain): set trajectory.terminal_node

      trajectory = Trajectory()
      handler = TrajectoryCallbackHandler(trajectory)
      result = app.invoke(input, config={"callbacks": [handler]})

  3:
    task: "Write assertion functions"
    detail: >
      def assert_called_tool(trajectory: Trajectory, tool_name: str):
          names = [tc.tool_name for tc in trajectory.tool_calls]
          assert tool_name in names, f"Expected tool '{tool_name}' to be called. Got: {names}"

      def assert_tool_order(trajectory: Trajectory, first: str, second: str):
          names = [tc.tool_name for tc in trajectory.tool_calls]
          assert names.index(first) < names.index(second), \
              f"Expected '{first}' before '{second}'. Got order: {names}"

      def assert_visited_node(trajectory: Trajectory, node_name: str):
          assert node_name in trajectory.nodes_visited, \
              f"Expected node '{node_name}' to be visited. Got: {trajectory.nodes_visited}"

      def assert_max_tool_calls(trajectory: Trajectory, max_count: int):
          assert len(trajectory.tool_calls) <= max_count, \
              f"Expected at most {max_count} tool calls. Got: {len(trajectory.tool_calls)}"

  4:
    task: "Write a pytest test file"
    detail: >
      # test_agent.py
      import pytest
      from your_agent import app
      from trajectory import Trajectory, TrajectoryCallbackHandler
      from assertions import assert_called_tool, assert_tool_order

      def run_with_trajectory(input_text: str) -> Trajectory:
          trajectory = Trajectory()
          handler = TrajectoryCallbackHandler(trajectory)
          app.invoke(
              {"messages": [HumanMessage(content=input_text)]},
              config={"callbacks": [handler]}
          )
          return trajectory

      def test_research_task_calls_researcher():
          t = run_with_trajectory("Research the history of Python and summarize it.")
          assert_visited_node(t, "researcher")
          assert_visited_node(t, "writer")

      def test_simple_question_does_not_call_researcher():
          t = run_with_trajectory("What is 2 + 2?")
          assert "researcher" not in t.nodes_visited

  5:
    task: "Capture a golden run"
    detail: >
      Run the agent on a known good input at temperature=0.
      Save the trajectory: trajectory.save("golden_runs/research_task.json")
      Write a test that loads the golden run and compares it to a new run.

  6:
    task: "Run pytest"
    detail: >
      pytest test_agent.py -v

      Fix any failures. Pay attention to when tests fail because the agent
      is non-deterministic vs when they fail because the assertion is wrong.
```

---

## Determinism Exercise

```yaml
exercise:
  task: "Run the same test 5 times and record how often it passes"
  detail: >
    Set temperature=1 on your LLM. Run the same pytest test 5 times.
    Record pass rate.

    Then set temperature=0. Run 5 times again.
    Record pass rate.

    Then replace the LLM with a mock that always returns the same output.
    Run 5 times. Record pass rate.

  why: >
    This exercise makes the determinism problem concrete. You will use these
    three strategies (temp=0, mock LLM, soft assertions) in production eval.
    AgentEval's golden run comparison only works reliably with temperature=0
    or mock LLMs — understanding why is essential before using the tool.
```

---

## Done When

```yaml
done_when:
  - Trajectory captures nodes_visited and tool_calls correctly for every run
  - At least 3 pytest tests pass consistently at temperature=0
  - You have saved a golden run and written a regression test against it
  - You have run the determinism exercise and can explain the three strategies
  - You can read the AgentEval assertion DSL examples in v0.4 and understand
    exactly which part of the Trajectory each assertion reads
  - You can explain what "golden run management" means in production
```

---

## What This Unlocks

Stage 08 adds OpenTelemetry — turning the callback events from Stage 04 into structured spans. After building trajectory capture by hand, the AgentTrace span design is the natural next step rather than a leap.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the right granularity for trajectory assertions — exact match (same tools in same order)
    or soft match (required tools present, order flexible)?
  - How do you handle a test agent that calls an LLM inside an assertion? Is that reliable?
  - What's the difference between a trajectory assertion and a unit test on a node function?
  - How would you version a golden run when the agent's behavior legitimately changes?
```
