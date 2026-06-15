# Stage 01 — First Vector Store

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive start — this codebase grows through all 6 stages
**Depends on:** basic Python, Docker running

Vectors are just numbers. A vector store is a database optimized for finding vectors that are numerically close to a query vector. "Close" in vector space means "semantically similar" in text. This stage strips away all the RAG abstraction and shows you the raw mechanics: embed some text, store the vectors, query by similarity, get results back.

---

## What You're Building

A Python script that:
1. Starts Qdrant locally via Docker
2. Creates a collection (think: a table for vectors)
3. Embeds 20 short text snippets using OpenAI's embedding API
4. Stores them in Qdrant
5. Queries with a natural language question and returns the top 3 most similar results

No LLM generation yet. Just retrieval. Understanding retrieval in isolation is what makes the full RAG pipeline make sense.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Embeddings:
    what: >
      A function that converts text into a fixed-length list of floats (a vector).
      Semantically similar texts produce vectors that are close together in space.
      "king" and "queen" are closer to each other than "king" and "bicycle".
    why: >
      This is the foundation of every RAG system. Without embeddings there is no
      semantic search — only keyword matching. Understanding that an embedding is
      just a list of floats (not magic) demystifies everything built on top of it.

  Cosine similarity:
    what: >
      A measure of how close two vectors are — 1.0 means identical direction,
      0.0 means unrelated, -1.0 means opposite. Qdrant uses this (or dot product)
      to rank results.
    why: >
      This is what "semantic similarity" actually means in code. When Qdrant returns
      the top 3 results for a query, it's returning the 3 vectors with highest
      cosine similarity to the query vector.

  Qdrant collection:
    what: >
      A named group of vectors with a fixed dimension size and distance metric.
      Equivalent to a table in a relational DB. Every vector in a collection has
      the same length and is compared using the same distance function.
    why: >
      You will create a collection, upsert documents into it, and query it.
      These three operations are 90% of what you do with a vector DB in production.

  Payload:
    what: >
      Metadata attached to each vector in Qdrant — the original text, a document ID,
      a source URL, a timestamp. When Qdrant returns a match, it returns the vector
      score AND the payload.
    why: >
      The vector gets you the score. The payload gets you the actual content to
      return to the user (or feed to the LLM). Without payload, you'd have a score
      but no text.

  Upsert vs insert:
    what: >
      Upsert = insert if not exists, update if exists (matched by ID).
      Qdrant uses upsert as the standard write operation.
    why: >
      You'll re-run your ingestion script multiple times during development.
      Upsert means it's idempotent — running it twice doesn't create duplicates.
```

---

## Setup

```yaml
dependencies:
  python:
    - qdrant-client
    - openai
    - python-dotenv

  docker: "Qdrant runs as a container — no local install needed"

install: "pip install qdrant-client openai python-dotenv"
```

Start Qdrant:
```bash
docker run -d --name qdrant -p 6333:6333 qdrant/qdrant
```

Verify: `curl http://localhost:6333/healthz` → `{"title":"qdrant - vector search engine",...}`

---

## Build Steps

```yaml
steps:
  1:
    task: "Connect to Qdrant and create a collection"
    detail: >
      from qdrant_client import QdrantClient
      from qdrant_client.models import Distance, VectorParams

      client = QdrantClient(host="localhost", port=6333)

      client.recreate_collection(
          collection_name="docs",
          vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
      )
      # size=1536 matches OpenAI's text-embedding-3-small output dimension

  2:
    task: "Define 20 text snippets as your corpus"
    detail: >
      CORPUS = [
          {"id": 1, "text": "LangGraph is a library for building stateful multi-agent applications."},
          {"id": 2, "text": "A StateGraph defines nodes as Python functions that read and write state."},
          {"id": 3, "text": "Nodes in LangGraph are connected by edges, which can be conditional."},
          {"id": 4, "text": "Checkpointing in LangGraph persists state across multiple invocations."},
          {"id": 5, "text": "MemorySaver is an in-memory checkpointer for LangGraph graphs."},
          # ... add 15 more on related topics: tools, RAG, agents, embeddings, etc.
      ]

      Write these yourself — the act of writing them forces you to think about
      what makes a good retrieval corpus.

  3:
    task: "Embed the corpus"
    detail: >
      from openai import OpenAI

      openai_client = OpenAI()

      def embed(texts: list[str]) -> list[list[float]]:
          response = openai_client.embeddings.create(
              model="text-embedding-3-small",
              input=texts,
          )
          return [item.embedding for item in response.data]

      texts = [item["text"] for item in CORPUS]
      vectors = embed(texts)   # one API call for all 20 texts — batch is cheaper

  4:
    task: "Upsert into Qdrant"
    detail: >
      from qdrant_client.models import PointStruct

      points = [
          PointStruct(
              id=item["id"],
              vector=vector,
              payload={"text": item["text"], "doc_id": item["id"]},
          )
          for item, vector in zip(CORPUS, vectors)
      ]

      client.upsert(collection_name="docs", points=points)
      print(f"Upserted {len(points)} points")

  5:
    task: "Query by similarity"
    detail: >
      def search(query: str, top_k: int = 3) -> list[dict]:
          query_vector = embed([query])[0]
          results = client.search(
              collection_name="docs",
              query_vector=query_vector,
              limit=top_k,
          )
          return [
              {"score": round(r.score, 4), "text": r.payload["text"]}
              for r in results
          ]

      # Test
      results = search("How does state persistence work in LangGraph?")
      for r in results:
          print(f"{r['score']:.4f}  {r['text']}")

  6:
    task: "Observe score distribution"
    detail: >
      Run 5 different queries. For each:
        - Print all 3 results with their scores
        - Is the top result actually the most relevant one?
        - How big is the score gap between result 1 and result 3?

      Keep a note of any query where the top result is wrong.
      This is your baseline retrieval quality — you'll improve it in later stages.
```

---

## What to Break On Purpose

```yaml
deliberate_breaks:
  - Query with something completely unrelated ("What is photosynthesis?").
    Qdrant still returns 3 results — it always does. What are the scores?
    This is the "no threshold" problem — raw similarity has no concept of "no match".
  - Create the collection with size=768 but embed with text-embedding-3-small (1536 dims).
    What error do you get? This is the dimension mismatch error you will see in production.
  - Upsert the same corpus twice with different text but the same IDs.
    Query again. Which version is returned? (Upsert overwrites — confirm this.)
  - Try to upsert a point with a string ID instead of int. Does Qdrant accept it?
```

---

## Done When

```yaml
done_when:
  - 20 points are in Qdrant and queries return scored results
  - You can explain cosine similarity in plain language
  - You've noted at least one query where the top result is wrong or surprising
  - You understand what payload is and why it's necessary
  - You can describe the full flow: text → embedding → vector → upsert → query → score + payload
```

---

## Open Questions

```yaml
open_questions:
  - What is the difference between text-embedding-3-small and text-embedding-3-large?
    When would you pay for the larger model?
  - What happens to retrieval quality if you embed very long texts (5000 tokens)?
    Why does this matter for chunking (Stage 02)?
  - What is the difference between cosine similarity and dot product as distance metrics?
    When does it matter which you choose?
  - Qdrant also supports filtering by payload fields (e.g., only search within docs
    where source == "langchain"). How would you use this in a real system?
```
