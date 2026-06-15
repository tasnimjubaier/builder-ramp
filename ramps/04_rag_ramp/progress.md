# RAG Ramp — Progress

```yaml
status: not_started
started: ""
completed: ""
total_stages: 6
stages_done: 0
```

---

## Stages

```yaml
stages:
  01_vector_store:
    status: not_started   # [ not_started | in_progress | complete ]
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "Qdrant collection with 20 embedded docs, similarity search working"
    notes: ""

  02_chunking:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "three Qdrant collections (fixed / recursive / semantic), retrieval comparison table"
    notes: ""

  03_embeddings:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "real document corpus embedded, embedding model comparison"
    notes: ""

  04_hybrid_retrieval:
    status: not_started
    started: ""
    completed: ""
    estimate: "2 days"
    artifact: "BM25 + vector hybrid search with reranking pipeline"
    notes: ""

  05_rag_in_langgraph:
    status: not_started
    started: ""
    completed: ""
    estimate: "1–2 days"
    artifact: "retriever as a LangGraph tool node, full RAG graph running"
    notes: ""

  06_capstone:
    status: not_started
    started: ""
    completed: ""
    estimate: "3–4 days"
    artifact: "production RAG pipeline with RAGAS faithfulness + answer_relevancy scores, FastAPI + Docker Compose"
    notes: ""
```

---

## Prereqs

```yaml
prereqs:
  python_ramp: "stages 01–05 required"
  docker: "Qdrant must be running before stage 01"
  corpus: "LangGraph docs pages — crawl or download as text files"

docker_start: "docker run -d --name qdrant -p 6333:6333 qdrant/qdrant"
docker_verify: "curl http://localhost:6333/healthz"
```

---

## Notes

<!-- freeform — add blockers, surprises, retrieval quality observations -->
