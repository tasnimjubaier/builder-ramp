# Stage 04 — Typed LLM SDK Wrapper

**Status:** not started
**Estimated time:** 1 day
**Type:** standalone
**Depends on:** Stages 01–02 (TypeScript + zod)

SDK writing is the pattern behind every TypeScript client library — including the OpenAI SDK, Anthropic SDK, and any client SDK you ship with your OSS projects. The pattern: hide HTTP complexity, expose a typed, ergonomic interface, handle errors consistently. This stage builds a thin wrapper around an LLM API to internalize the pattern before you apply it to your own API in Stage 05.

---

## What You're Building

A typed wrapper around the OpenAI chat completions API with:
- A `chat()` method with typed parameters and return value
- A `stream()` method that yields typed chunks
- Configurable retry and timeout
- Usage tracking across calls

This is smaller and more focused than the real OpenAI SDK — the goal is to understand the pattern, not reproduce the full SDK.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Class-based SDK design:
    what: >
      A class that holds configuration (API key, base URL, model) and exposes
      typed methods. The constructor is where you inject config — not module-level globals.
    why: >
      This is the pattern the OpenAI, Anthropic, and LangChain SDKs all use.
      Understanding it means reading any SDK's source code becomes trivial.

  AsyncGenerator for streaming:
    what: >
      async function* stream(): AsyncGenerator<string> { yield chunk; }
      The caller iterates with: for await (const chunk of client.stream(...))
    why: >
      LLM streaming responses arrive as a sequence of chunks over time.
      AsyncGenerator is the TypeScript idiom for this — same concept as
      Python's async for over an async generator. You'll use this in the
      capstone's SSE streaming.

  Builder pattern for request config:
    what: >
      Method chaining to build a request: client.chat("hello").withModel("gpt-4o").withTemperature(0)
    why: >
      Optional and not required — but common in well-designed SDKs.
      Understanding it means you can read and use chained APIs confidently.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define the types"
    detail: >
      interface ChatMessage { role: "user" | "assistant" | "system"; content: string; }
      interface ChatOptions { model?: string; temperature?: number; maxTokens?: number; }
      interface ChatResult  { content: string; model: string; usage: TokenUsage; }
      interface TokenUsage  { promptTokens: number; completionTokens: number; totalTokens: number; }

  2:
    task: "Build the LLMClient class"
    detail: >
      import OpenAI from "openai";

      class LLMClient {
          private openai: OpenAI;
          private defaultModel: string;
          public totalUsage: TokenUsage = { promptTokens: 0, completionTokens: 0, totalTokens: 0 };

          constructor(apiKey: string, defaultModel = "gpt-4o-mini") {
              this.openai = new OpenAI({ apiKey });
              this.defaultModel = defaultModel;
          }

          async chat(messages: ChatMessage[], options: ChatOptions = {}): Promise<ChatResult> {
              const response = await this.openai.chat.completions.create({
                  model: options.model ?? this.defaultModel,
                  messages,
                  temperature: options.temperature,
                  max_tokens: options.maxTokens,
              });

              const usage: TokenUsage = {
                  promptTokens:     response.usage?.prompt_tokens ?? 0,
                  completionTokens: response.usage?.completion_tokens ?? 0,
                  totalTokens:      response.usage?.total_tokens ?? 0,
              };

              // accumulate across calls
              this.totalUsage.promptTokens     += usage.promptTokens;
              this.totalUsage.completionTokens += usage.completionTokens;
              this.totalUsage.totalTokens      += usage.totalTokens;

              return {
                  content: response.choices[0].message.content ?? "",
                  model:   response.model,
                  usage,
              };
          }

          async *stream(messages: ChatMessage[], options: ChatOptions = {}): AsyncGenerator<string> {
              const stream = await this.openai.chat.completions.create({
                  model: options.model ?? this.defaultModel,
                  messages,
                  stream: true,
              });

              for await (const chunk of stream) {
                  const delta = chunk.choices[0]?.delta?.content;
                  if (delta) yield delta;
              }
          }
      }

  3:
    task: "Test both methods"
    detail: >
      const client = new LLMClient(process.env.OPENAI_API_KEY!);

      // Non-streaming
      const result = await client.chat([
          { role: "system", content: "You are a concise assistant." },
          { role: "user",   content: "What is TypeScript in one sentence?" },
      ]);
      console.log("Response:", result.content);
      console.log("Tokens used:", result.usage.totalTokens);

      // Streaming
      process.stdout.write("Streaming: ");
      for await (const chunk of client.stream([
          { role: "user", content: "Count to 5 slowly." },
      ])) {
          process.stdout.write(chunk);
      }
      console.log();

      // Total usage across both calls
      console.log("Total tokens this session:", client.totalUsage.totalTokens);
```

---

## Done When

```yaml
done_when:
  - chat() returns a typed ChatResult with usage data
  - stream() yields string chunks and the caller can iterate them
  - totalUsage accumulates correctly across multiple calls
  - You can explain the AsyncGenerator pattern and when to use it
  - You can describe the class-based SDK design in one paragraph
```

---

## Open Questions

```yaml
open_questions:
  - How would you add rate limiting to LLMClient — a token bucket that ensures
    you don't exceed 10,000 tokens per minute?
  - How would you mock LLMClient in tests so no real API calls are made?
    (Hint: dependency injection — pass a client factory, not the class directly)
  - What is the difference between stream: true and a regular completion call in
    terms of the HTTP connection? (Hint: SSE vs regular JSON response)
  - How would you add multi-provider support — OpenAI and Anthropic behind the same interface?
```
