# RAG Ramp — Learning Sequence

RAG (Retrieval-Augmented Generation) is a tier-1 universal requirement — every AI engineer role expects it. The concept is simple: instead of relying on the LLM's training data, you retrieve relevant documents at query time and give them to the LLM as context. The engineering is in the retrieval quality — chunking, embedding, indexing, hybrid search, reranking.

This ramp is additive. Each stage extends the same codebase and corpus. The capstone is a production-grade RAG pipeline with RAGAS evaluation scores.

---

## Structure

```yaml
stages:
  01_vector_store:       "First Qdrant setup + basic similarity search. Additive start."
  02_chunking:           "Chunking strategies — fixed, semantic, recursive. Same corpus."
  03_embeddings:         "Embedding models, embedding a real document corpus."
  04_hybrid_retrieval:   "BM25 + vector hybrid search + reranking."
  05_rag_in_langgraph:   "RAG as a LangGraph node. Ties to agentic ramp."
  06_capstone:           "Production RAG pipeline with RAGAS eval scores. Deployable."
```

---

## Completion Criteria

```yaml
done_when:
  - You can embed a corpus and retrieve relevant documents from Qdrant
  - You can explain why chunking strategy matters and pick the right one for a use case
  - You can run hybrid BM25 + vector search and explain when it outperforms vector-only
  - You can plug a retriever into a LangGraph node as a tool
  - You have RAGAS faithfulness and answer_relevancy scores for your pipeline
  - The full pipeline runs behind a FastAPI endpoint in Docker Compose
```

---

## Approach

```yaml
corpus: >
  Use a real document set throughout — not toy strings.
  Suggested: the LangGraph documentation pages (crawl or download as text files).
  Same corpus across all stages so improvements in retrieval quality are measurable.

rules:
  - Each stage extends the same codebase — don't start fresh after Stage 01
  - Use Qdrant running in Docker from Stage 01 onwards
  - Measure retrieval quality at each stage — not just "does it return something"
  - Use gpt-4o-mini for generation — cost near zero
```
