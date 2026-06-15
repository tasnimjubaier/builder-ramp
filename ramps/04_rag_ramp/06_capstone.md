# Stage 06 — Capstone: Production RAG Pipeline with RAGAS Eval

**Status:** not started
**Estimated time:** 3–4 days
**Type:** additive — ties all RAG stages + FastAPI ramp capstone together
**Depends on:** RAG Stages 01–05 + FastAPI Ramp (for deployment)

Everything built in this ramp — Qdrant, chunking, embeddings, hybrid retrieval, reranking, LangGraph integration — gets wrapped in a deployable API with eval scores that prove it works. This is the artifact. RAGAS scores in the README mean a hiring manager can see the pipeline's quality, not just its existence.

---

## What You're Building

`rag-api` — a deployable RAG service:

```
POST /ask              submit a question, get a grounded answer + retrieved sources
GET  /ask/{run_id}     poll for async answer (same pattern as FastAPI capstone)
POST /eval             run RAGAS evaluation against a test dataset, return scores
GET  /health           health check
```

Deployed via Docker Compose: FastAPI + Qdrant + Redis.
RAGAS scores (faithfulness ≥ 0.7, answer_relevancy ≥ 0.7) in the README.
GitHub Actions CI runs eval on every push and fails if scores drop below threshold.

---

## Architecture

```
Client
  │
  POST /ask
  │
  ▼
FastAPI
  │  - validates request
  │  - enqueues background job (Redis)
  │  - returns run_id (202)
  │
  BackgroundTask: run_rag_agent(run_id, question)
  │
  ▼
LangGraph RAG Agent (Stage 05)
  │  - retrieve tool → hybrid_search → Qdrant
  │  - grounded generation → OpenAI
  │  - writes answer + sources to Redis
  │
GET /ask/{run_id} → Redis → return answer + sources

POST /eval
  │
  ▼
RAGAS evaluation
  │  - runs test dataset through pipeline
  │  - scores faithfulness, answer_relevancy, context_precision
  │  - returns scores JSON
```

---

## Build Steps

```yaml
steps:
  1:
    task: "Define models"
    detail: >
      class AskRequest(BaseModel):
          question: str
          thread_id: str | None = None

      class Source(BaseModel):
          text: str
          score: float

      class AskResult(BaseModel):
          run_id: str
          status: str
          question: str
          answer: str | None = None
          sources: list[Source] = []
          error: str | None = None

      class EvalRequest(BaseModel):
          dataset_path: str = "eval_dataset.json"

      class EvalResult(BaseModel):
          faithfulness: float
          answer_relevancy: float
          context_precision: float
          sample_count: int

  2:
    task: "Track sources through the pipeline"
    detail: >
      Extend the RAG agent state to capture which chunks were retrieved:

      class RAGState(TypedDict):
          messages: Annotated[list, add_messages]
          retrieved_sources: list[dict]   # populated by the retrieve tool

      Update the retrieve tool to append to retrieved_sources in state:
        return {"messages": [...], "retrieved_sources": results}

      The final AskResult includes sources so the caller can see what the answer
      was grounded in. This is the "show your work" feature — essential for trust.

  3:
    task: "Build the background runner"
    detail: >
      def run_rag_background(run_id: str, question: str, thread_id: str) -> None:
          store.update_status(run_id, "running")
          try:
              result = rag_app.invoke(
                  {"messages": [HumanMessage(content=question)],
                   "retrieved_sources": []},
                  config={"configurable": {"thread_id": thread_id}},
              )
              answer = result["messages"][-1].content
              sources = [Source(text=s["text"], score=s.get("rrf_score", 0.0))
                         for s in result["retrieved_sources"][:3]]
              store.save_result(run_id, answer=answer, sources=sources, status="done")
          except Exception as e:
              store.save_result(run_id, error=str(e), status="failed")

  4:
    task: "Build POST /eval endpoint"
    detail: >
      import json
      from ragas import evaluate
      from ragas.metrics import faithfulness, answer_relevancy, context_precision
      from datasets import Dataset

      @app.post("/eval", response_model=EvalResult)
      async def run_eval(body: EvalRequest) -> EvalResult:
          with open(body.dataset_path) as f:
              raw = json.load(f)

          # Run each question through the pipeline synchronously
          eval_rows = []
          for item in raw:
              result = rag_app.invoke(
                  {"messages": [HumanMessage(content=item["question"])],
                   "retrieved_sources": []},
              )
              answer = result["messages"][-1].content
              contexts = [s["text"] for s in result["retrieved_sources"][:3]]
              eval_rows.append({
                  "question": item["question"],
                  "answer": answer,
                  "contexts": contexts,
                  "ground_truth": item["ground_truth"],
              })

          dataset = Dataset.from_list(eval_rows)
          scores = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision])
          return EvalResult(
              faithfulness=round(scores["faithfulness"], 3),
              answer_relevancy=round(scores["answer_relevancy"], 3),
              context_precision=round(scores["context_precision"], 3),
              sample_count=len(eval_rows),
          )

  5:
    task: "Create eval_dataset.json"
    detail: >
      Write 10 question + ground_truth pairs based on your corpus.
      Ground truth = the correct factual answer from the document.

      [
        {
          "question": "What does the MemorySaver checkpointer store?",
          "ground_truth": "MemorySaver stores LangGraph state in memory, keyed by thread_id."
        },
        ...
      ]

      This dataset is the eval contract. Scores below threshold = CI failure.

  6:
    task: "Docker Compose with Qdrant"
    detail: >
      services:
        api:
          build: .
          ports: ["8000:8000"]
          environment:
            - QDRANT_HOST=qdrant
            - REDIS_HOST=redis
            - OPENAI_API_KEY=${OPENAI_API_KEY}
          depends_on: [qdrant, redis]

        qdrant:
          image: qdrant/qdrant
          ports: ["6333:6333"]
          volumes:
            - qdrant_data:/qdrant/storage

        redis:
          image: redis:7-alpine

      volumes:
        qdrant_data:

      The qdrant_data volume persists vectors across restarts — no re-ingestion needed.

  7:
    task: "Add ingestion endpoint or startup script"
    detail: >
      Add a startup event that checks if the collection exists and ingests if not:

      @asynccontextmanager
      async def lifespan(app: FastAPI):
          if not collection_exists("docs"):
              ingest_corpus()   # embed and upsert the corpus
          yield

      app = FastAPI(lifespan=lifespan)

      This means docker compose up → corpus ingested → API ready. No manual ingestion step.

  8:
    task: "GitHub Actions CI with eval gate"
    detail: >
      - name: Run RAGAS eval
        run: |
          docker compose up -d
          sleep 10
          SCORES=$(curl -s -X POST http://localhost:8000/eval \
            -H "Content-Type: application/json" \
            -d '{"dataset_path": "eval_dataset.json"}')
          echo "Eval scores: $SCORES"
          FAITHFULNESS=$(echo $SCORES | python3 -c "import sys,json; print(json.load(sys.stdin)['faithfulness'])")
          python3 -c "assert float('$FAITHFULNESS') >= 0.7, 'Faithfulness below threshold'"

  9:
    task: "Write the README"
    detail: >
      Include:
        - What it is (2 sentences)
        - RAGAS scores (faithfulness / answer_relevancy / context_precision) as a table
        - Architecture diagram
        - Quickstart: docker compose up + curl examples
        - Eval endpoint: how to run it yourself

      The RAGAS scores table is the proof. It's what makes this a real artifact
      rather than another PDF chatbot.
```

---

## Done When

```yaml
done_when:
  - docker compose up starts API + Qdrant + Redis
  - POST /ask returns grounded answers with sources
  - POST /eval returns RAGAS scores ≥ 0.7 for faithfulness and answer_relevancy
  - GitHub Actions CI runs eval and fails on score regression
  - README shows actual RAGAS scores
  - You can explain every design decision — why async for /ask, why hybrid retrieval,
    why RAGAS faithfulness as the primary metric

graduation_test: >
  Read the AgentMemory deep dive in 05_ideas_v0.3.md.
  For every sentence in the technical_deep_dive section, identify which stage of
  this ramp taught you the relevant concept. If you can do this, you are ready
  to build AgentMemory.
```
