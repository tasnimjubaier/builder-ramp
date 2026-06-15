# Stage 02 — Chunking Strategies

**Status:** not started
**Estimated time:** 1–2 days
**Type:** additive — extends Stage 01 codebase and corpus
**Depends on:** Stage 01 (Qdrant running, embed() function built)

Chunking is where most RAG pipelines break in production. Embed a whole document as one vector and the embedding averages across all topics — retrieval quality collapses. Chunk too small and you lose context — the retrieved fragment is meaningless without its surroundings. This stage makes chunking concrete by letting you measure the quality difference between strategies on the same corpus.

---

## What You're Building

Replace the 20 hand-written snippets from Stage 01 with a real document — the LangGraph README or documentation page (a few thousand words). Ingest it using three chunking strategies, measure which one retrieves better, and understand why.

Same Qdrant instance. Three separate collections — one per strategy. Same queries across all three. Compare scores.

---

## Concepts This Stage Teaches

```yaml
concepts:
  Fixed-size chunking:
    what: Split text every N characters (or tokens), with an optional overlap.
    why: >
      Simplest to implement. Works for uniform-density text (legal docs, transcripts).
      Breaks badly when a sentence or concept spans a chunk boundary — the meaning
      is split and both halves retrieve poorly. Overlap mitigates this partially.
    when_to_use: Baseline. When speed matters more than quality.

  Recursive character splitting:
    what: >
      LangChain's RecursiveCharacterTextSplitter. Tries to split on paragraph breaks
      first, then sentences, then words, then characters — in priority order.
      Respects natural text boundaries rather than forcing splits at fixed positions.
    why: >
      Better than fixed-size for most prose. Sentences and paragraphs stay intact.
      The "recursive" part means it tries successively smaller separators until
      the chunk fits within the target size.
    when_to_use: Default choice for most prose documents.

  Semantic chunking:
    what: >
      Embed consecutive sentences. When the cosine similarity between adjacent
      sentence embeddings drops sharply, it's a topic boundary — split there.
    why: >
      Groups sentences that are about the same topic regardless of their position.
      More expensive (requires embedding every sentence) but produces the most
      topically coherent chunks. Good for long mixed-topic documents.
    when_to_use: Long documents with multiple distinct topics. When quality matters more than cost.

  Chunk overlap:
    what: >
      Re-include the last N characters of the previous chunk at the start of the next.
    why: >
      Ensures that concepts split across a boundary appear in at least one complete chunk.
      Overlap of 10–20% of chunk size is a common default.

  Chunk size tradeoff:
    what: Small chunks → precise retrieval, lost context. Large chunks → rich context, noisy retrieval.
    why: >
      The right chunk size depends on what the LLM needs. A factual Q&A system wants
      precise 200-token chunks. A summarization system wants rich 800-token chunks.
      There is no universal answer — you measure.
```

---

## Setup

```yaml
new_dependencies:
  - langchain-text-splitters
  - langchain-experimental   # for SemanticChunker

install: "pip install langchain-text-splitters langchain-experimental"

corpus:
  download: >
    curl -o langgraph_docs.txt https://raw.githubusercontent.com/langchain-ai/langgraph/main/README.md
    Or copy any ~3000-word technical document you have locally.
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Load the document"
    detail: >
      with open("langgraph_docs.txt") as f:
          full_text = f.read()

      print(f"Document length: {len(full_text)} characters")
      print(f"Approximate tokens: {len(full_text) // 4}")

  2:
    task: "Fixed-size chunking"
    detail: >
      from langchain_text_splitters import CharacterTextSplitter

      fixed_splitter = CharacterTextSplitter(
          chunk_size=500,
          chunk_overlap=50,
          separator="\n",
      )
      fixed_chunks = fixed_splitter.split_text(full_text)
      print(f"Fixed-size: {len(fixed_chunks)} chunks, "
            f"avg length: {sum(len(c) for c in fixed_chunks) // len(fixed_chunks)} chars")

  3:
    task: "Recursive character splitting"
    detail: >
      from langchain_text_splitters import RecursiveCharacterTextSplitter

      recursive_splitter = RecursiveCharacterTextSplitter(
          chunk_size=500,
          chunk_overlap=50,
      )
      recursive_chunks = recursive_splitter.split_text(full_text)
      print(f"Recursive: {len(recursive_chunks)} chunks, "
            f"avg length: {sum(len(c) for c in recursive_chunks) // len(recursive_chunks)} chars")

      # Print 3 chunks from each strategy side by side — do they respect sentence boundaries?
      for i in range(3):
          print(f"\n--- Fixed chunk {i} ---\n{fixed_chunks[i]}")
          print(f"\n--- Recursive chunk {i} ---\n{recursive_chunks[i]}")

  4:
    task: "Semantic chunking"
    detail: >
      from langchain_experimental.text_splitter import SemanticChunker
      from langchain_openai import OpenAIEmbeddings

      embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

      semantic_splitter = SemanticChunker(
          embeddings,
          breakpoint_threshold_type="percentile",
          breakpoint_threshold_amount=95,   # split on top 5% similarity drops
      )
      semantic_chunks = semantic_splitter.split_text(full_text)
      print(f"Semantic: {len(semantic_chunks)} chunks, "
            f"avg length: {sum(len(c) for c in semantic_chunks) // len(semantic_chunks)} chars")

  5:
    task: "Ingest all three into separate Qdrant collections"
    detail: >
      def ingest_chunks(chunks: list[str], collection_name: str) -> None:
          # Create or recreate collection
          client.recreate_collection(
              collection_name=collection_name,
              vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
          )
          # Embed in batches of 20 (API rate limit safety)
          all_vectors = []
          for i in range(0, len(chunks), 20):
              batch = chunks[i:i+20]
              all_vectors.extend(embed(batch))

          points = [
              PointStruct(id=i, vector=v, payload={"text": c, "chunk_index": i})
              for i, (c, v) in enumerate(zip(chunks, all_vectors))
          ]
          client.upsert(collection_name=collection_name, points=points)
          print(f"Ingested {len(points)} chunks into '{collection_name}'")

      ingest_chunks(fixed_chunks, "docs_fixed")
      ingest_chunks(recursive_chunks, "docs_recursive")
      ingest_chunks(semantic_chunks, "docs_semantic")

  6:
    task: "Compare retrieval quality across strategies"
    detail: >
      TEST_QUERIES = [
          "How does LangGraph handle state persistence?",
          "What is the difference between nodes and edges?",
          "How do I add a tool to a LangGraph agent?",
          "What is the entry point of a StateGraph?",
      ]

      for query in TEST_QUERIES:
          print(f"\n=== Query: {query} ===")
          for collection in ["docs_fixed", "docs_recursive", "docs_semantic"]:
              results = search(query, collection_name=collection, top_k=1)
              print(f"[{collection}] score={results[0]['score']:.4f} | {results[0]['text'][:100]}")

      For each query, which strategy returned the most relevant top result?
      Which had the highest score? Note patterns — don't just look at one query.
```

---

## Done When

```yaml
done_when:
  - All three collections are populated and queryable
  - You have run at least 4 queries across all three strategies and compared results
  - You can explain in plain language why recursive splitting is the default choice
  - You understand what chunk overlap does and can reason about the right overlap %
  - You have identified at least one query where semantic chunking outperforms recursive
    and can explain why
  - You know what chunk size you'll use for the capstone and why
```

---

## Open Questions

```yaml
open_questions:
  - What chunking strategy would you use for code files? For PDFs with tables?
    For a transcript with speaker turns?
  - How does chunk overlap affect the total number of vectors in the collection?
    Is storing overlapping content wasteful or necessary?
  - What is the "lost in the middle" problem in LLM context windows, and how
    does it affect how you order retrieved chunks before passing them to the LLM?
  - LlamaIndex has a SentenceWindowNodeParser that stores the full surrounding
    paragraph as metadata but embeds only the sentence. How does this differ from
    semantic chunking and when would you use it?
```
