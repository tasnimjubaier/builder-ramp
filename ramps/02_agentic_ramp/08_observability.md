# Stage 08 — Observability with OpenTelemetry

**Status:** not started
**Estimated time:** 3 days
**Depends on:** Stage 04 (callbacks), Stage 07 (trajectory capture)

Every senior agentic JD lists observability. What they actually want is evidence you've instrumented a running system at the decision level — not just printed logs. OpenTelemetry is the industry standard for structured, backend-agnostic tracing. This stage takes the callback events you've been printing since Stage 04 and turns them into proper OTel spans visible in a real trace viewer.

---

## What You're Building

An `OtelCallbackHandler` that:
1. Emits an OpenTelemetry span for every node execution, tool call, and LLM call
2. Maintains parent-child span relationships (node span is parent of tool span)
3. Carries structured attributes (node name, tool name, token counts, duration)
4. Exports to a local Jaeger instance running in Docker

One `docker compose up` → your agent runs → open `localhost:16686` → see the full trace tree.

---

## Concepts This Stage Teaches

```yaml
concepts:
  OpenTelemetry basics:
    what: >
      Three primitives: Tracer (creates spans), Span (a named, timed operation),
      Exporter (sends spans to a backend). You create a span, do work inside it,
      end it. Spans have a parent_id — if you create a span inside another span's
      context, they are linked in the trace tree.
    why: >
      This is all you need to understand to read the AgentTrace deep dive.
      The "OTel gen_ai.* semantic conventions", "span relationships", and
      "trace topology" sections are just this model applied to agents.

  gen_ai.* semantic conventions:
    what: >
      A set of standardized OTel attribute names for LLM calls:
      gen_ai.system (e.g., "openai"), gen_ai.request.model, gen_ai.usage.input_tokens,
      gen_ai.usage.output_tokens. These are the official names — use them.
    why: >
      AgentTrace specifically mentions implementing these conventions.
      Using them means your spans are compatible with any OTel-native dashboard.

  agent.* custom attributes:
    what: >
      AgentTrace extends gen_ai.* with agent-specific attributes:
      agent.node.name, agent.tool.name, agent.tool.success, etc.
    why: >
      gen_ai.* covers LLM calls but not agent decision topology. You will define
      agent.* attributes yourself in this stage — which is exactly what AgentTrace does.
      Building them yourself first means AgentTrace's design makes complete sense.

  Span context propagation:
    what: >
      When you create a span inside another span's context, the child span carries
      the parent's trace_id and a parent_span_id. This links them in the trace tree.
      Across process boundaries, this context is propagated via HTTP headers (traceparent).
    why: >
      This is how AgentTrace links supervisor and subagent spans into a single trace.
      The "multi-agent delegation chain as a single navigable trace tree" is just
      span context propagation across subgraph invocations.

  Jaeger:
    what: An open-source trace visualization backend. Accepts OTel spans via OTLP.
    why: >
      Free, local, runs in one Docker container. The AgentTrace dev stack uses Jaeger
      for exactly this reason. Seeing your spans in Jaeger for the first time makes
      the whole observability story click.
```

---

## Setup

```yaml
dependencies:
  python:
    - opentelemetry-api
    - opentelemetry-sdk
    - opentelemetry-exporter-otlp-proto-grpc

  docker:
    - jaeger: "jaegertracing/all-in-one:latest"
```

Install Python dependencies:
```
pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-grpc
```

Docker Compose file (`docker-compose.yml`):
```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"   # Jaeger UI
      - "4317:4317"     # OTLP gRPC
```

Start Jaeger: `docker compose up -d`
View traces: `http://localhost:16686`

---

## Build Steps

```yaml
steps:
  1:
    task: "Set up the OTel tracer"
    detail: >
      from opentelemetry import trace
      from opentelemetry.sdk.trace import TracerProvider
      from opentelemetry.sdk.trace.export import BatchSpanProcessor
      from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

      provider = TracerProvider()
      exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
      provider.add_span_processor(BatchSpanProcessor(exporter))
      trace.set_tracer_provider(provider)

      tracer = trace.get_tracer("my-agent")

  2:
    task: "Build OtelCallbackHandler"
    detail: >
      class OtelCallbackHandler(BaseCallbackHandler):
          def __init__(self, tracer):
              self.tracer = tracer
              self._spans = {}   # run_id -> span

          def on_chain_start(self, serialized, inputs, *, run_id, parent_run_id=None, **kwargs):
              name = serialized.get("name", "unknown_node")
              parent_span = self._spans.get(parent_run_id)
              ctx = trace.set_span_in_context(parent_span) if parent_span else None
              span = self.tracer.start_span(f"node.{name}", context=ctx)
              span.set_attribute("agent.node.name", name)
              self._spans[run_id] = span

          def on_chain_end(self, outputs, *, run_id, **kwargs):
              span = self._spans.pop(run_id, None)
              if span:
                  span.end()

          def on_tool_start(self, serialized, input_str, *, run_id, parent_run_id=None, **kwargs):
              name = serialized.get("name", "unknown_tool")
              parent_span = self._spans.get(parent_run_id)
              ctx = trace.set_span_in_context(parent_span) if parent_span else None
              span = self.tracer.start_span(f"tool.{name}", context=ctx)
              span.set_attribute("agent.tool.name", name)
              span.set_attribute("agent.tool.input", input_str[:200])
              self._spans[run_id] = span

          def on_tool_end(self, output, *, run_id, **kwargs):
              span = self._spans.pop(run_id, None)
              if span:
                  span.set_attribute("agent.tool.success", True)
                  span.set_attribute("agent.tool.output", str(output)[:200])
                  span.end()

          def on_llm_end(self, response, *, run_id, parent_run_id=None, **kwargs):
              parent_span = self._spans.get(parent_run_id)
              ctx = trace.set_span_in_context(parent_span) if parent_span else None
              with self.tracer.start_as_current_span("llm.call", context=ctx) as span:
                  if response.llm_output:
                      usage = response.llm_output.get("token_usage", {})
                      span.set_attribute("gen_ai.usage.input_tokens",
                                         usage.get("prompt_tokens", 0))
                      span.set_attribute("gen_ai.usage.output_tokens",
                                         usage.get("completion_tokens", 0))

  3:
    task: "Run your Stage 06 agent with OtelCallbackHandler"
    detail: >
      handler = OtelCallbackHandler(tracer)
      result = app.invoke(input, config={"callbacks": [handler]})

  4:
    task: "Open Jaeger and find your trace"
    detail: >
      Go to http://localhost:16686
      Select service "my-agent"
      Click Search
      Find your trace and expand it

      You should see:
        node.supervisor
          llm.call
        node.researcher
          tool.mock_search
        node.writer
          llm.call
        node.supervisor
          llm.call

      This is the trace topology. Each span has timing, attributes, parent-child relationships.

  5:
    task: "Add cost tracking"
    detail: >
      In on_llm_end, compute cost from token counts using a price table:
        PRICES = {"gpt-4o-mini": {"input": 0.00015/1000, "output": 0.0006/1000}}
        cost = input_tokens * price["input"] + output_tokens * price["output"]
        span.set_attribute("gen_ai.usage.total_cost", cost)

      Add a running cost accumulator across the full run and set it on the root span.
      This is the agent.cost.cumulative_usd attribute in AgentTrace.
```

---

## Done When

```yaml
done_when:
  - Traces appear in Jaeger with correct parent-child structure
  - Node spans are parents of tool spans and LLM spans
  - Spans carry gen_ai.* and agent.* attributes
  - Cost is tracked per run and visible as a span attribute
  - You can look at a trace in Jaeger and reconstruct the agent's execution from it
    without looking at the code or logs
  - You can read the AgentTrace technical deep dive and point to the code
    you wrote in this stage for every concept it describes
```

---

## What This Unlocks

Stage 09 adds resilience — retries, fallback, circuit breakers. With observability in place, you can actually see retry behavior in traces, which makes the AgentResilience architecture concrete.

---

## Open Questions to Answer During the Build

```yaml
open_questions:
  - What is the difference between a Span and a SpanContext?
  - What happens to a span that is never ended — does it appear in Jaeger?
  - How would you propagate trace context across an HTTP call to a subagent running
    in a separate process? (Hint: look up W3C traceparent header)
  - What is the difference between BatchSpanProcessor and SimpleSpanProcessor?
    When would you use each?
```
