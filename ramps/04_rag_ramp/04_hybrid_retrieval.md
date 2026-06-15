# Stage 04 — Hybrid Retrieval and Reranking

**Status:** not started
**Estimated time:** 2 days
**Type:** additive — extends Stage 01–03 codebase
**Depends on:** Stage 03 (embedding model abstraction built, benchmark exists)

Vector search is good at semantic similarity but bad at exact matches. If a user asks "What does the `add_messages` reducer do?", vector search might return vaguely relevant chunks about state. BM25 (keyword search) finds the exact term `add_messages` reliably. Hybrid retrieval combines both. Reranking then re-scores the combined results with a more powerful model. This combination is what production RAG uses.

---

## What You're Building

Extend the pipeline to run both vector search and BM25 on every query, merge the results using Reciprocal Rank Fusion (RRF), and optionally rerank with a cross-encoder. Re-run your Stage 03 benchmark and compare hit_rate@3. The improvement should be measurable.

---

## Concepts This Stage Teaches

```yaml
concepts:
  BM25:
    what: >
      A classical keyword-based ranking algorithm. Scores documents by term frequency
      and inverse document frequency — how often the query terms appear in the document
      relative to how common they are across the corpus. No vectors involved.
    why: >
      Pure vector search misses exact matches. If the query contains a specific function
      name, library name, or technical term, BM25 finds it reliably. The combination
      covers both semantic similarity AND lexical overlap.

  Reciprocal Rank Fusion (RRF):
    what: >
      A score fusion formula. Each candidate gets a score of 1/(rank + k) from each
      retrieval method (k is usually 60). Sum the scores across methods. Sort by total score.
    why: >
      Simple, parameter-light, and empirically effective at combining ranked lists.
      No need to normalize scores from different systems — only rank position matters.
      This is the standard hybrid fusion method — Qdrant's built-in hybrid search uses it.

  Cross-encoder reranking:
    what: >
      A model that takes (query, document) pairs and outputs a relevance score.
      Unlike embedding models that encode query and document independently,
      a cross-encoder sees both together — it can model interactions between them.
    why: >
      More accurate than cosine similarity but too slow to run on the full corpus.
      The pattern: use fast vector/BM25 to get top 20 candidates, then use
      cross-encoder to rerank them to top 3. Quality of top-3 improves significantly.

  Two-stage retrieval:
    what: >
      Stage 1 (recall): fast, approximate retrieval — get top 20–50 candidates cheaply.
      Stage 2 (precision): slow, accurate reranking — score the candidates precisely.
    why: >
      This is the production pattern. You can't run a cross-encoder over 10,000 chunks
      per query — latency would be seconds. But you can run it over 20 candidates in
      under 100ms. This architecture lets you have both speed and quality.
```

---

## Setup

```yaml
new_dependencies:
  - rank_bm25              # BM25 implementation
  - sentence-transformers  # cross-encoder reranker (already installed from Stage 03)

install: "pip install rank_bm25"
note: "cross-encoder/ms-marco-MiniLM-L-6-v2 is the standard reranker model — downloaded on first use"
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Build BM25 index over your chunks"
    detail: >
      from rank_bm25 import BM25Okapi

      def tokenize(text: str) -> list[str]:
          return text.lower().split()

      # chunks from Stage 02 (recursive, whichever performed best)
      tokenized_chunks = [tokenize(c) for c in recursive_chunks]
      bm25_index = BM25Okapi(tokenized_chunks)

      def bm25_search(query: str, top_k: int = 10) -> list[dict]:
          tokenized_query = tokenize(query)
          scores = bm25_index.get_scores(tokenized_query)
          top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:top_k]
          return [
              {"chunk_index": i, "score": scores[i], "text": recursive_chunks[i]}
              for i in top_indices
          ]

  2:
    task: "Implement Reciprocal Rank Fusion"
    detail: >
      def reciprocal_rank_fusion(
          ranked_lists: list[list[dict]],
          id_key: str = "chunk_index",
          k: int = 60,
      ) -> list[dict]:
          scores: dict[int, float] = {}
          docs: dict[int, dict] = {}

          for ranked_list in ranked_lists:
              for rank, item in enumerate(ranked_list):
                  doc_id = item[id_key]
                  scores[doc_id] = scores.get(doc_id, 0) + 1 / (rank + k)
                  docs[doc_id] = item

          sorted_ids = sorted(scores, key=lambda x: scores[x], reverse=True)
          return [{"chunk_index": i, "rrf_score": scores[i], "text": docs[i]["text"]}
                  for i in sorted_ids]

  3:
    task: "Build hybrid search function"
    detail: >
      def hybrid_search(query: str, top_k: int = 3) -> list[dict]:
          # Stage 1: get candidates from both retrievers
          vector_results = search(query, collection_name="docs_openai_1536", top_k=10)
          bm25_results   = bm25_search(query, top_k=10)

          # Fuse rankings
          fused = reciprocal_rank_fusion([vector_results, bm25_results])

          return fused[:top_k]

  4:
    task: "Add cross-encoder reranking"
    detail: >
      from sentence_transformers import CrossEncoder

      reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

      def reranked_search(query: str, top_k: int = 3, candidate_k: int = 20) -> list[dict]:
          # Stage 1: hybrid retrieval — get top 20 candidates
          candidates = hybrid_search(query, top_k=candidate_k)

          # Stage 2: cross-encoder reranking
          pairs = [(query, c["text"]) for c in candidates]
          rerank_scores = reranker.predict(pairs)

          reranked = sorted(
              zip(candidates, rerank_scores),
              key=lambda x: x[1],
              reverse=True,
          )
          return [
              {**item, "rerank_score": float(score)}
              for item, score in reranked[:top_k]
          ]

  5:
    task: "Re-run Stage 03 benchmark with all three retrieval strategies"
    detail: >
      def hit_rate_fn(search_fn, benchmark, k=3):
          hits = 0
          for item in benchmark:
              results = search_fn(item["query"])[:k]
              returned_ids = [r["chunk_index"] for r in results]
              if any(eid in returned_ids for eid in item["expected_chunk_ids"]):
                  hits += 1
          return hits / len(benchmark)

      vector_rate  = hit_rate_fn(lambda q: search(q, "docs_openai_1536", top_k=3), BENCHMARK)
      hybrid_rate  = hit_rate_fn(lambda q: hybrid_search(q, top_k=3), BENCHMARK)
      reranked_rate = hit_rate_fn(lambda q: reranked_search(q, top_k=3), BENCHMARK)

      print(f"Vector only:  {vector_rate:.0%}")
      print(f"Hybrid:       {hybrid_rate:.0%}")
      print(f"Hybrid+rerank:{reranked_rate:.0%}")

  6:
    task: "Identify where each strategy wins and loses"
    detail: >
      For each query in your benchmark, print which strategy returned the correct
      chunk in position 1 vs 2 vs 3 vs not at all.

      Find:
        - A query where hybrid beats vector-only (likely: exact technical term)
        - A query where reranking promotes a result that hybrid buried (likely: semantically close but lexically dissimilar)
        - A query where all three perform the same (likely: clear, unambiguous semantic query)
```

---

## Done When

```yaml
done_when:
  - Hybrid search runs and produces fused results from BM25 + vector
  - Reranking further improves results on at least some queries
  - Benchmark shows measurable improvement from vector → hybrid → hybrid+rerank
  - You can explain RRF to someone without them needing to look it up
  - You can describe the two-stage retrieval architecture and its latency tradeoffs
  - You know which strategy you'll use for the capstone and why
```

---

## Open Questions

```yaml
open_questions:
  - Qdrant has native sparse vector support for BM25 hybrid search (using SPLADE).
    How does this differ from the BM25-outside-Qdrant approach you built here?
    When would you use the native approach vs the external one?
  - What is the difference between a bi-encoder (embedding model) and a cross-encoder
    (reranker) architecturally? Why can't you just use a cross-encoder for the full corpus?
  - What is "contextual retrieval" (Anthropic's approach from 2024)?
    How does it differ from standard chunking + embedding?
  - At what corpus size does maintaining a BM25 index in memory become impractical?
    What would you use instead?
```
