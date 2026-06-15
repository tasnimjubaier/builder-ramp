# Stage 03 — Embedding Models

**Status:** not started
**Estimated time:** 1 day
**Type:** additive — extends Stage 01–02 codebase
**Depends on:** Stage 02 (chunking pipeline built)

You've been using OpenAI's embedding model as a black box. This stage opens it. Different embedding models produce vectors of different dimensions, trained on different data, with different quality/cost tradeoffs. Choosing the wrong one for your use case degrades retrieval quality before you've written a single line of retrieval logic.

---

## What You're Building

Extend the ingestion pipeline to support multiple embedding models. Re-embed your corpus with two alternatives to OpenAI and compare retrieval quality across all three. Build a simple benchmark: 10 queries with known good answers, score each model on how often the correct chunk is in the top 3 results.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Embedding model dimensions:
    what: >
      Each model outputs a fixed-length vector. OpenAI text-embedding-3-small: 1536 dims.
      text-embedding-3-large: 3072 dims. all-MiniLM-L6-v2 (local): 384 dims.
      Your Qdrant collection must be created with the correct dimension for the model you use.
    why: >
      Dimension mismatch is a common production error — switching embedding models requires
      re-creating the collection and re-embedding the entire corpus. You cannot mix vectors
      from different models in the same collection.

  Matryoshka embeddings:
    what: >
      OpenAI's text-embedding-3 models support truncation — you can request fewer dimensions
      than the full output (e.g., 256 instead of 1536) and quality degrades gracefully.
    why: >
      Lower-dimensional vectors are cheaper to store and faster to search. For high-volume
      systems where cost matters, truncating to 256 dims can cut storage 6x with modest
      quality loss. Knowing this exists is a production signal.

  Local embedding models:
    what: >
      Models that run on your machine via HuggingFace sentence-transformers.
      No API call, no cost, lower latency. Quality varies by model.
    why: >
      Some production systems (privacy constraints, cost at scale, offline requirements)
      cannot use OpenAI. Knowing how to swap in a local model is an expected skill.
      Also useful for testing without burning API credits.

  Hit rate as a retrieval metric:
    what: >
      Given a question with a known correct document, what % of the time does
      the correct document appear in the top-k results? hit_rate@3 = correct in top 3.
    why: >
      This is how you measure retrieval quality before connecting an LLM.
      If hit_rate@3 is 60%, adding a better LLM won't fix it — the LLM never
      sees the right document. Fix retrieval first.
```

---

## Setup

```yaml
new_dependencies:
  - sentence-transformers   # local embedding models
  - torch                   # required by sentence-transformers

install: "pip install sentence-transformers torch"
note: "torch is large (~2GB). Install once, it's cached."
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Abstract the embedding function"
    detail: >
      Refactor embed() from Stage 01 into a class so you can swap models:

      from abc import ABC, abstractmethod

      class EmbeddingModel(ABC):
          @property
          @abstractmethod
          def dimensions(self) -> int: ...

          @abstractmethod
          def embed(self, texts: list[str]) -> list[list[float]]: ...

  2:
    task: "Implement OpenAI embedding model"
    detail: >
      class OpenAIEmbedder(EmbeddingModel):
          def __init__(self, model: str = "text-embedding-3-small", dimensions: int = 1536):
              self._model = model
              self._dimensions = dimensions
              self._client = OpenAI()

          @property
          def dimensions(self) -> int:
              return self._dimensions

          def embed(self, texts: list[str]) -> list[list[float]]:
              kwargs = {"model": self._model, "input": texts}
              if self._dimensions != 1536:
                  kwargs["dimensions"] = self._dimensions   # Matryoshka truncation
              response = self._client.embeddings.create(**kwargs)
              return [item.embedding for item in response.data]

  3:
    task: "Implement local sentence-transformers model"
    detail: >
      from sentence_transformers import SentenceTransformer

      class LocalEmbedder(EmbeddingModel):
          def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
              self._model = SentenceTransformer(model_name)

          @property
          def dimensions(self) -> int:
              return self._model.get_sentence_embedding_dimension()

          def embed(self, texts: list[str]) -> list[list[float]]:
              return self._model.encode(texts).tolist()

  4:
    task: "Re-ingest corpus with each model into separate collections"
    detail: >
      models = {
          "openai_1536":  OpenAIEmbedder("text-embedding-3-small", 1536),
          "openai_256":   OpenAIEmbedder("text-embedding-3-small", 256),
          "local_minilm": LocalEmbedder("all-MiniLM-L6-v2"),
      }

      # Use the best chunking strategy from Stage 02 (probably recursive)
      chunks = recursive_chunks  # from Stage 02

      for name, model in models.items():
          ingest_chunks(chunks, collection_name=f"docs_{name}", embedder=model)

      Update ingest_chunks() to accept an embedder parameter.

  5:
    task: "Build a benchmark"
    detail: >
      Define 10 question + expected_chunk_id pairs. The expected_chunk_id is the
      index of the chunk you know contains the correct answer (find it manually):

      BENCHMARK = [
          {"query": "How does LangGraph handle state persistence?",
           "expected_chunk_ids": [4, 5]},   # chunks that mention checkpointing
          {"query": "What is the entry point of a StateGraph?",
           "expected_chunk_ids": [12]},
          # ... 8 more
      ]

      def hit_rate(model_name: str, benchmark: list[dict], k: int = 3) -> float:
          hits = 0
          for item in benchmark:
              results = search(item["query"], collection_name=f"docs_{model_name}", top_k=k)
              returned_ids = [r["chunk_index"] for r in results]
              if any(eid in returned_ids for eid in item["expected_chunk_ids"]):
                  hits += 1
          return hits / len(benchmark)

      for name in models:
          rate = hit_rate(name, BENCHMARK)
          print(f"{name}: hit_rate@3 = {rate:.0%}")

  6:
    task: "Compare results and document findings"
    detail: >
      Print the hit_rate@3 for each model. Add a comment block at the top of your script:

      # Benchmark results (langgraph_docs.txt, recursive chunks, chunk_size=500)
      # openai_1536:  hit_rate@3 = X%
      # openai_256:   hit_rate@3 = X%
      # local_minilm: hit_rate@3 = X%

      Which model will you use for the capstone and why?
      (It won't always be the most expensive one.)
```

---

## Done When

```yaml
done_when:
  - Three collections exist in Qdrant with different embedding models
  - Benchmark runs and produces hit_rate@3 for each model
  - You can explain the dimension difference between the three models
  - You understand Matryoshka truncation and when you'd use it
  - You have chosen a model for the capstone and written one sentence justifying why
  - You can swap the embedding model in your pipeline by changing one line
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between sentence similarity models (all-MiniLM) and
    asymmetric retrieval models (multi-qa-MiniLM)? When does the asymmetry matter?
  - How would you embed images alongside text? What model would you use?
    (Hint: CLIP — multi-modal embeddings)
  - At what corpus size does it become worth switching from OpenAI to a local model
    for cost reasons? Work out the math for your corpus.
  - What is "embedding drift" — and why is it a production concern when OpenAI
    updates their embedding models?
```
