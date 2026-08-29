# Enterprise Agentic RAG Platform

> Production-grade Corrective RAG system — LangGraph state machine, hybrid FAISS+BM25 retrieval, cross-encoder reranking, SSE streaming, RAGAS evaluation CI, Docker deployment. Runs fully offline at ₹0 cost.

![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=flat&logo=python&logoColor=white)
![LangGraph](https://img.shields.io/badge/LangGraph-0.2-FF6B35?style=flat)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?style=flat&logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=flat&logo=docker&logoColor=white)
![RAGAs](https://img.shields.io/badge/Evaluated-RAGAs-4CAF50?style=flat)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat)

---

## What This Builds

Upload any document (PDF, DOCX, Markdown, HTML, TXT) and ask questions in plain English.
A LangGraph agent grades retrieved chunks, rewrites weak queries, falls back to web search,
and streams the final cited answer — zero paid API dependencies required.

---

## Architecture

```
User → Streamlit UI (port 8501)
              ↓
       FastAPI Backend (port 8000)
       [API Key Auth · Rate Limiting · SSE Streaming]
              ↓
       LangGraph Corrective-RAG Agent
       ┌────────────────────────────────┐
       │  1. Hybrid Retrieval           │
       │     FAISS dense search         │
       │     + BM25 keyword search      │
       │     → RRF fusion               │
       │                                │
       │  2. Cross-encoder Rerank       │
       │     bge-reranker-v2-m3         │
       │                                │
       │  3. Relevance Grader (LLM)     │
       │      ↙              ↘         │
       │   Pass              Fail       │
       │     ↓           Query Rewrite  │
       │  Generate         → retry ×2  │
       │                   → DuckDuckGo │
       └────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent | LangGraph 0.2 | Stateful Corrective-RAG graph with conditional routing |
| LLM (local) | Ollama — Mistral 7B / Llama3 | Fully offline, ₹0 cost |
| LLM (cloud) | Groq llama-3.1-8b-instant | Free-tier cloud fallback, faster on CPU |
| Embeddings | BAAI/bge-m3 (sentence-transformers) | Multilingual, 1024-dim |
| Reranker | bge-reranker-v2-m3 | Cross-encoder 2-stage precision boost |
| Vector DB | FAISS IndexFlatIP | CPU-friendly, no infrastructure needed |
| Keyword | BM25Okapi + rank-bm25 | Sparse retrieval complement |
| Fusion | Reciprocal Rank Fusion (RRF) | Combines dense + sparse rankings |
| API | FastAPI + uvicorn | Async, production-grade |
| UI | Streamlit | Interactive chat interface |
| Evaluation | RAGAS | Faithfulness, Answer Relevancy, Context Precision/Recall |
| Web fallback | DuckDuckGo Search | No API key, no cost |
| Caching | diskcache | Disk-backed query cache |
| Observability | Loguru | Structured logging with request tracing |
| Containers | Docker + docker-compose | Non-root user, CIS-compliant |
| CI/CD | GitHub Actions | Automated RAGAS quality gate on every push |

---

## Key Features

**Retrieval & Agent**
- Corrective RAG loop — grades retrieved chunks → rewrites query → retries (max 2×) → DuckDuckGo web fallback
- 3-stage retrieval: FAISS dense + BM25 sparse → RRF fusion → cross-encoder rerank
- Conversation memory via LangGraph MemorySaver (thread-level checkpointing)
- Token-aware context truncation before LLM call

**API Security**
- API key authentication on all routes (`X-API-Key` header)
- CORS restricted to configured origins — not wildcard
- File upload: magic-byte validation + 50 MB cap + path-traversal guard
- Rate limiting via slowapi (20 queries/min, 5 ingests/hr per IP)
- Background ingestion — returns 202 immediately, never blocks

**Quality & Observability**
- RAGAS evaluation CI — pipeline fails if faithfulness < 0.70
- LangSmith tracing — zero code change, activated via env vars
- SSE streaming endpoint (`/query/stream`)
- Docker deployment — non-root user, CIS-compliant image

---

## Quick Start (No Docker)

```bash
# Step 1 — Install Ollama from https://ollama.ai
ollama pull mistral

# Step 2 — Clone and set up
git clone https://github.com/rashmisontakke/Enterprise-Agentic-RAG-Platform.git
cd Enterprise-Agentic-RAG-Platform

python -m venv .venv
.venv\Scripts\activate        # Windows
pip install -r requirements.txt

# Step 3 — Configure
cp .env.example .env
# Edit .env — set API_KEYS to any string

# Step 4 — Add documents to data/ folder then ingest
python -m ingestion.ingest

# Step 5 — Run (two terminals)
uvicorn app.api:app --reload --port 8000
streamlit run app/main.py
# Open: http://localhost:8501
```

> **Groq option:** Set `GROQ_API_KEY` in `.env` to use Groq free-tier `llama-3.1-8b-instant` — faster on CPU-only machines.

---

## API Endpoints

All routes require `X-API-Key` header.

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Liveness probe |
| POST | `/query` | Ask a question (sync) |
| POST | `/query/stream` | SSE streaming answer |
| POST | `/ingest` | Upload and index a document (background) |
| GET | `/ingest/{job_id}` | Poll ingestion status |
| GET | `/metrics` | Latest RAGAS evaluation scores |

---

## RAGAS Evaluation

```bash
python -m evaluation.evaluate
```

CI pipeline runs on every push to `main` and fails automatically if `faithfulness < 0.70`.

---

## Docker Deployment

```bash
docker compose up --build
docker exec -it ollama ollama pull mistral
```

- API: `http://localhost:8000`
- UI: `http://localhost:8501`
- Swagger: `http://localhost:8000/docs`

---

## Project Structure

```
├── app/
│   ├── agent.py          # LangGraph Corrective-RAG state machine
│   ├── api.py            # FastAPI backend (auth, rate limit, SSE)
│   └── main.py           # Streamlit chat UI
├── embeddings/
│   └── embedder.py       # bge-m3 embedder + bge-reranker-v2-m3
├── ingestion/
│   └── ingest.py         # Multi-format doc loader + chunker
├── vectorstore/
│   └── store.py          # FAISS + BM25 hybrid search + RRF fusion
├── evaluation/
│   └── evaluate.py       # RAGAS scoring suite
├── .github/workflows/
│   └── evaluate.yml      # CI quality gate
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── requirements.txt
```

---

## Author

**Rashmi Sontakke** - AI-ML Enginner | Force Motors EV R&D Division  
📍 Pune, India | Open to Remote / Relocation  
📧 rashmisontakke91@gmail.com  
🔗 [LinkedIn](https://linkedin.com/in/rashmi-sontakke) · [GitHub](https://github.com/rashmisontakke)

---

## License

MIT
