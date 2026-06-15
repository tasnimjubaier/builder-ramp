# Stage 10 — Capstone: Mini AgentEval

**Status:** not started
**Estimated time:** 4–5 days
**Depends on:** All previous stages

This is the synthesis. You've built every primitive — state graphs, tools, callbacks, checkpointing, multi-agent systems, trajectory capture, OTel spans, and resilience decorators. The capstone ties them together into a minimal but real, pip-installable, pytest-integrated trajectory assertion library.

This is not a tutorial build. This is the first OSS artifact you can put on GitHub and point a hiring manager to.

---

## What You're Building

`mini-agenteval` — a Python package with:
1. `Trajectory` — structured record of an agent run (nodes, tools, edges, cost)
2. `TrajectoryCallbackHandler` — captures trajectory from any LangGraph graph
3. An assertion DSL — `expect(run).to_call_tool("search").before("summarize")`
4. A pytest fixture — `langgraph_run(graph, input)` → `Trajectory`
5. A golden run CLI — `mini-agenteval capture <test_name>`
6. A GitHub Actions workflow that runs your pytest trajectory tests on every push

The scope is intentionally narrower than AgentEval. Two or three assertions, one CI workflow, clean README. Done means shipped and runnable — not feature-complete.

---

## What This Stage Is Really About

```yaml
purpose:
  - Consolidate everything from stages 01–09 into one coherent codebase
  - Learn how to structure a Python library (not a script)
  - Learn how to package and distribute (pyproject.toml, TestPyPI)
  - Build your first real OSS artifact with a README that tells a story
  - Practice writing code at library quality — public API, docstrings, typed signatures
```

---

## Package Structure

```
mini-agenteval/
├── pyproject.toml
├── README.md
├── mini_agenteval/
│   ├── __init__.py
│   ├── trajectory.py       # Trajectory + TrajectoryCallbackHandler
│   ├── assertions.py       # expect() DSL + AssertionResult
│   ├── fixtures.py         # pytest fixture: langgraph_run()
│   └── cli.py              # mini-agenteval capture command
├── tests/
│   ├── test_trajectory.py
│   └── test_assertions.py
└── examples/
    └── example_agent.py    # minimal LangGraph agent for demo
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Set up pyproject.toml"
    detail: >
      [build-system]
      requires = ["hatchling"]
      build-backend = "hatchling.build"

      [project]
      name = "mini-agenteval"
      version = "0.1.0"
      dependencies = ["langgraph", "langchain-core", "pydantic"]

      [project.scripts]
      mini-agenteval = "mini_agenteval.cli:app"

  2:
    task: "Port Trajectory from Stage 07 into trajectory.py"
    detail: >
      Clean it up to library quality:
      - Use Pydantic BaseModel instead of dataclasses (serialization is cleaner)
      - Add type hints to every method
      - Add docstrings to the public API
      - Add Trajectory.save() and Trajectory.load() methods
      - Port TrajectoryCallbackHandler in the same file

  3:
    task: "Build the expect() DSL in assertions.py"
    detail: >
      class TrajectoryExpectation:
          def __init__(self, trajectory: Trajectory):
              self._trajectory = trajectory
              self._pending_tool: str | None = None

          def to_call_tool(self, name: str) -> "TrajectoryExpectation":
              names = [tc.tool_name for tc in self._trajectory.tool_calls]
              assert name in names, \
                  f"Expected tool '{name}' to be called. Tools called: {names}"
              self._pending_tool = name
              return self   # return self for chaining

          def before(self, other: str) -> "TrajectoryExpectation":
              assert self._pending_tool, "Call .to_call_tool() before .before()"
              names = [tc.tool_name for tc in self._trajectory.tool_calls]
              a, b = self._pending_tool, other
              assert a in names and b in names, \
                  f"Both '{a}' and '{b}' must be called for ordering assertion"
              assert names.index(a) < names.index(b), \
                  f"Expected '{a}' before '{b}'. Actual order: {names}"
              return self

          def not_to_call_tool(self, name: str) -> "TrajectoryExpectation":
              names = [tc.tool_name for tc in self._trajectory.tool_calls]
              assert name not in names, \
                  f"Expected '{name}' NOT to be called, but it was."
              return self

          def not_to_loop_more_than(self, n: int) -> "TrajectoryExpectation":
              from collections import Counter
              counts = Counter(self._trajectory.nodes_visited)
              offenders = {node: count for node, count in counts.items() if count > n}
              assert not offenders, \
                  f"Nodes exceeded loop limit of {n}: {offenders}"
              return self

          def to_terminate_at_node(self, name: str) -> "TrajectoryExpectation":
              assert self._trajectory.terminal_node == name, \
                  f"Expected termination at '{name}'. Got: '{self._trajectory.terminal_node}'"
              return self

      def expect(trajectory: Trajectory) -> TrajectoryExpectation:
          return TrajectoryExpectation(trajectory)

  4:
    task: "Build the pytest fixture in fixtures.py"
    detail: >
      import pytest
      from langchain_core.messages import HumanMessage
      from .trajectory import Trajectory, TrajectoryCallbackHandler

      @pytest.fixture
      def langgraph_run():
          def _run(graph, input_text: str) -> Trajectory:
              trajectory = Trajectory()
              handler = TrajectoryCallbackHandler(trajectory)
              graph.invoke(
                  {"messages": [HumanMessage(content=input_text)]},
                  config={"callbacks": [handler]}
              )
              return trajectory
          return _run

      Usage in tests:
        def test_something(langgraph_run):
            t = langgraph_run(my_agent, "Research Python history")
            expect(t).to_call_tool("mock_search")

  5:
    task: "Build the capture CLI in cli.py"
    detail: >
      import typer
      app = typer.Typer()

      @app.command()
      def capture(test_name: str, input_text: str):
          \"\"\"Capture a golden run for regression testing.\"\"\"
          # imports your example agent, runs it, saves trajectory
          from .trajectory import Trajectory, TrajectoryCallbackHandler
          from examples.example_agent import build_agent
          from langchain_core.messages import HumanMessage

          trajectory = Trajectory()
          handler = TrajectoryCallbackHandler(trajectory)
          agent = build_agent()
          agent.invoke(
              {"messages": [HumanMessage(content=input_text)]},
              config={"callbacks": [handler]}
          )
          path = f"golden_runs/{test_name}.json"
          trajectory.save(path)
          print(f"Golden run saved to {path}")

  6:
    task: "Write tests in tests/"
    detail: >
      test_trajectory.py: unit tests for Trajectory — test save/load round-trip,
        test that ToolCall records are populated correctly

      test_assertions.py: test each assertion method:
        - test that to_call_tool passes when the tool was called
        - test that to_call_tool fails with the right message when it wasn't
        - test that before() passes and fails correctly
        - test that not_to_loop_more_than catches loops

  7:
    task: "Write a clean README.md"
    detail: >
      Section 1: what it does (2 sentences)
      Section 2: installation (pip install)
      Section 3: quickstart — a complete example, 15 lines of code
      Section 4: available assertions with code samples
      Section 5: CI gate — the GitHub Actions workflow YAML

      The README is part of the artifact. A hiring manager who finds this repo
      decides in 30 seconds whether to read further. Make those 30 seconds count.

  8:
    task: "Set up GitHub Actions CI"
    detail: >
      .github/workflows/test.yml

      on: [push, pull_request]
      jobs:
        test:
          runs-on: ubuntu-latest
          steps:
            - uses: actions/checkout@v4
            - uses: actions/setup-python@v5
              with:
                python-version: "3.12"
            - run: pip install -e ".[dev]"
            - run: pytest tests/ -v

  9:
    task: "Publish to TestPyPI"
    detail: >
      python -m build
      twine upload --repository testpypi dist/*

      Then install from TestPyPI in a fresh virtualenv and run the quickstart example.
      If it works from TestPyPI, it's a real artifact.
```

---

## Done When

```yaml
done_when:
  - pip install mini-agenteval works from TestPyPI
  - The quickstart in the README runs without modification
  - pytest tests/ passes with at least 10 test cases covering all assertions
  - GitHub Actions CI is green on push
  - golden_runs/example.json exists and the capture CLI works
  - README is clean enough that you'd share the link in a job application
  - You can describe every design decision in the codebase — why Pydantic
    over dataclasses, why the fixture pattern, why the fluent DSL
```

---

## After the Capstone

```yaml
next:
  - This codebase is your proof of concept for AgentEval
  - The gap between mini-agenteval and AgentEval is mostly scope: more assertions,
    RAGAS integration, proper CI action, better error messages
  - You now have the vocabulary and finger memory to read the AgentEval deep dive
    and start building the real thing
  - Same path applies to AgentTrace: you've built the callback-to-span mapping by hand,
    now the extension to proper OTel semantic conventions is a design iteration, not a leap

graduation_test:
  - Read the AgentTrace technical deep dive in 07_ideas_v0.4.md
  - For every sentence, identify which stage of this ramp taught you the relevant concept
  - If you can do this for every sentence, you are ready to build AgentTrace
```
