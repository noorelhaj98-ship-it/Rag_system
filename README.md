# RAG System — Hybrid Search + LLM Question Answering

A production-oriented Retrieval-Augmented Generation (RAG) system built during an internship at **RealSoft**. The system ingests PDF and DOCX documents, stores them in a hybrid vector database, and serves answers through a FastAPI backend with a lightweight web UI.

---

## Tech Stack

- **Python 3.10+** · **FastAPI** — REST API with rate limiting and RBAC
- **Qdrant** — Hybrid vector store (dense + sparse BM25 vectors)
- **DeepSeek** — LLM for answer generation
- **AraGemma** — Arabic-capable embedding model
- **RRF (Reciprocal Rank Fusion)** — Fusion strategy for hybrid retrieval
- **Cross-Encoder** — Reranker (ms-marco-MiniLM-L-6-v2, with keyword-overlap fallback)

---

## Architecture

```
Document (PDF / DOCX)
        │
        ▼
┌──────────────────┐
│  Document Parser │  pdf_parser / docx_parser
│  + Text Chunker  │  chunk_size=1200, overlap=150
└────────┬─────────┘
         │  chunks.jsonl
         ▼
┌──────────────────┐
│  Vector Store    │  vector_store/pipeline.py
│  Ingestion       │  AraGemma embeddings (768-dim dense)
│                  │  BM25 sparse vectors
│                  │  → Qdrant hybrid collection
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│                FastAPI Server                │
│  POST /ask                                   │
│   ├─ Embed query (dense + sparse)            │
│   ├─ Hybrid retrieval (RRF fusion)           │
│   ├─ Cross-encoder reranking                 │
│   ├─ Context compression                     │
│   └─ DeepSeek LLM → answer + citations       │
│                                              │
│  GET  /          → Web UI (index.html)       │
│  GET  /history   → Conversation history      │
│  POST /clear     → Clear session             │
│  GET  /admin/rbac → RBAC status (admin only) │
└──────────────────────────────────────────────┘
```

---

## Project Structure

```
rag/
├── app/                        # FastAPI application
│   ├── api/routes.py           # All API endpoints
│   ├── core/
│   │   ├── conversation.py     # Multi-turn conversation manager
│   │   ├── logging.py          # Structured JSONL logging
│   │   └── security.py         # RBAC + API key authentication
│   ├── services/
│   │   ├── embedding.py        # Embedding service client
│   │   ├── llm.py              # DeepSeek LLM client
│   │   ├── reranker.py         # Cross-encoder reranker
│   │   └── retrieval.py        # Hybrid retrieval (dense + BM25)
│   ├── config.py               # Pydantic settings (env-driven)
│   └── models.py               # Request/response Pydantic models
│
├── pdf_parser/                 # PDF → chunks pipeline
│   ├── extractor.py            # Text extraction
│   ├── text_cleaner.py         # Cleaning and normalization
│   ├── chunk_builder.py        # Chunking logic
│   └── cli.py                  # CLI entry point
│
├── docx_parser/                # DOCX → chunks pipeline
│   ├── extractor.py
│   ├── chunk_builder.py
│   └── cli.py
│
├── vector_store/               # Qdrant ingestion pipeline
│   ├── pipeline.py             # Orchestrates embed + upload
│   ├── embedder.py             # Batch embedding client
│   ├── uploader.py             # Qdrant upsert logic
│   ├── bm25.py                 # BM25 sparse vector builder
│   └── loader.py               # JSONL chunk loader
│
├── eval_runner/                # Retrieval evaluation framework
│   ├── runner.py               # Orchestrates eval loop
│   ├── evaluator.py            # Top-K hit metrics
│   ├── reporter.py             # Results reporting
│   └── rag_client.py           # Calls the live /ask endpoint
│
├── common/                     # Shared utilities
│   ├── splitter.py             # Text splitting logic
│   └── exporters.py            # JSONL / CSV exporters
│
├── ingest_pdf.py               # Entry point: chunk a PDF
├── ingest_docx.py              # Entry point: chunk a DOCX
├── server.py                   # Entry point: start the API server
├── index.html                  # Web UI
├── system_prompt.yaml          # Configurable LLM system prompt
├── eval_questions.json         # Evaluation question set
└── .env.example                # Environment variable template
```

---

## Getting Started

### 1. Clone & install

```bash
git clone https://github.com/noorelhaj98-ship-it/rag.git
cd rag
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp .env.example .env
# Edit .env with your Qdrant URL, DeepSeek API key, and embedding service URL
```

### 3. Ingest a document

```bash
# Chunk a PDF
python ingest_pdf.py --source your_document.pdf

# Chunk a DOCX
python ingest_docx.py --source your_document.docx

# Upload chunks to Qdrant
python -m vector_store.pipeline
```

### 4. Run the server

```bash
uvicorn server:app --reload --port 8000
```

Open `http://localhost:8000` in your browser to use the web UI.

---

## API Endpoints

| Method | Endpoint       | Description                          | Auth      |
|--------|----------------|--------------------------------------|-----------|
| POST   | `/ask`         | Ask a question, get an answer        | optional  |
| GET    | `/history`     | Retrieve conversation history        | optional  |
| POST   | `/clear`       | Clear conversation session           | optional  |
| GET    | `/api`         | API info and available endpoints     | none      |
| GET    | `/admin/rbac`  | RBAC status and user roles           | admin     |

**Example request:**
```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What products does the company offer?"}'
```

**Example response:**
```json
{
  "answer": "The company offers three main product lines: ...",
  "sources": [
    { "source_file": "document.pdf", "page_number": 3, "chunk_id": "abc123" }
  ]
}
```

---

## Evaluation

The `eval_runner` module evaluates retrieval quality against a labeled question set.

```bash
python -m eval_runner.runner
```

Metrics reported:
- **Top-K hit rate** — % of questions where the correct chunk was retrieved
- **Correct page rate** — % of questions where the correct page was retrieved
- **Category breakdown** — hit rate per question category

---

## Key Design Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Vector search | Hybrid (dense + BM25) | Better recall than dense-only, especially for keyword-heavy queries |
| Fusion | RRF (k=60) | Simple, effective, no score normalization needed |
| Reranker | Cross-encoder with fallback | Improves precision; keyword fallback works without GPU |
| Chunking | 1200 chars / 150 overlap | Balances context window and retrieval granularity |
| Auth | RBAC via API keys | Simple role-based access without OAuth overhead |

---

## Internship Context

This system was developed during an internship at **RealSoft** as a practical exploration of RAG architectures. The goal was to build a working, evaluable system over a real company knowledge base — covering the full pipeline from document ingestion to answer generation with source citations.

---

## License

MIT
