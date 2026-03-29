# Product Requirements Document: Production-Grade RAG System

**Project Name:** Ask My Docs — Production RAG Application
**Created:** 2026-03-29
**Status:** Draft
**Version:** 1.0

---

## 1. Overview

Build a domain-specific "Ask My Docs" system that enables users to ask natural language questions against a corpus of ingested documents and receive accurate, cited answers. The system uses hybrid retrieval (BM25 + vector search), cross-encoder reranking, citation enforcement, and a CI-gated evaluation pipeline.

This is not a demo chatbot — it is a production-grade enterprise RAG application designed for reliability, accuracy, and measurable quality.

---

## 2. Goals and Objectives

| Goal | Description |
|------|-------------|
| **Accurate Retrieval** | Retrieve the most relevant document chunks using hybrid search (BM25 + semantic vector search) |
| **High Precision** | Improve retrieval precision with cross-encoder reranking |
| **Trustworthy Answers** | Enforce source citations in every generated answer — no unsupported claims |
| **Measurable Quality** | Evaluate system quality with RAGAS metrics and gate deployments on quality thresholds |
| **Production Ready** | Deliver a deployable system with proper API design, error handling, and CI/CD integration |

---

## 3. Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Language | Python 3.11+ | Industry standard for ML/AI applications |
| API Framework | FastAPI | Async-first, auto-generated OpenAPI docs, high performance |
| Orchestration | LangChain | Proven RAG integrations, document loaders, retrieval chains |
| Vector Store | Weaviate | Native hybrid search (BM25 + vector), production-grade, scalable |
| Embeddings | Ollama (`nomic-embed-text`) | Free, local, 768 dimensions, no API dependency |
| LLM | Anthropic Claude | Strong instruction following, citation compliance, low hallucination |
| Reranker | Cohere Rerank API | Best-in-class cross-encoder reranking, free tier available |
| Evaluation | RAGAS | Standard framework for RAG evaluation metrics |
| CI/CD | GitHub Actions | Quality-gated pipeline with automated evaluation |

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        FastAPI Server                        │
│                                                             │
│   ┌───────────┐     ┌────────────────┐    ┌──────────────┐  │
│   │  /ingest   │     │    /query      │    │  /evaluate   │  │
│   └─────┬─────┘     └───────┬────────┘    └──────┬───────┘  │
│         │                   │                    │           │
│         ▼                   ▼                    ▼           │
│   ┌───────────┐     ┌────────────────┐    ┌──────────────┐  │
│   │ Document  │     │   Retrieval    │    │  Evaluation  │  │
│   │ Pipeline  │     │   Pipeline     │    │  Pipeline    │  │
│   │           │     │                │    │  (RAGAS)     │  │
│   │ • Load    │     │ • Hybrid Search│    │              │  │
│   │ • Chunk   │     │   (BM25+Vec)  │    │ • Faithful.  │  │
│   │ • Embed   │     │ • Rerank      │    │ • Relevance  │  │
│   │ • Store   │     │   (Cohere)    │    │ • Precision  │  │
│   │           │     │ • Generate    │    │              │  │
│   └─────┬─────┘     │   (Claude)    │    └──────────────┘  │
│         │           │ • Cite Sources│                       │
│         │           └───────┬───────┘                       │
│         ▼                   ▼                               │
│   ┌─────────────────────────────────┐                       │
│   │           Weaviate              │                       │
│   │     (Hybrid Vector Store)       │                       │
│   └─────────────────────────────────┘                       │
│                                                             │
│   ┌──────────┐   ┌──────────┐   ┌────────────────┐         │
│   │  Ollama  │   │  Cohere  │   │   Anthropic    │         │
│   │ (Embeds) │   │ (Rerank) │   │   (Claude)     │         │
│   └──────────┘   └──────────┘   └────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Project Phases

### Phase 1: Fundamental RAG (MVP)

Build the core document ingestion and retrieval pipeline.

#### 5.1.1 Document Ingestion Pipeline

| Requirement | Details |
|-------------|---------|
| **FR-001** | System MUST ingest PDF documents using LangChain PDF loaders |
| **FR-002** | System MUST ingest Markdown files (.md) |
| **FR-003** | System MUST ingest plain text files (.txt) |
| **FR-004** | System MUST ingest web pages / HTML content |
| **FR-005** | System MUST chunk documents into 500–800 token segments with 100-token overlap |
| **FR-006** | System MUST generate embeddings using Ollama `nomic-embed-text` model |
| **FR-007** | System MUST store document chunks and embeddings in Weaviate |
| **FR-008** | System MUST preserve source metadata (filename, page number, chunk index) with each chunk |

#### 5.1.2 Basic Retrieval and Generation

| Requirement | Details |
|-------------|---------|
| **FR-009** | System MUST accept natural language queries via a REST API (`POST /query`) |
| **FR-010** | System MUST perform vector similarity search in Weaviate to retrieve top-K relevant chunks |
| **FR-011** | System MUST pass retrieved chunks as context to Claude for answer generation |
| **FR-012** | System MUST return the generated answer along with source document references |

#### 5.1.3 API Endpoints (Phase 1)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ingest` | POST | Upload and ingest documents (PDF, MD, TXT, HTML) |
| `/query` | POST | Submit a question and receive an answer with sources |
| `/health` | GET | Health check for the service and dependencies |

#### 5.1.4 Deliverables

- [ ] Document loaders for PDF, Markdown, plain text, HTML
- [ ] Chunking strategy with configurable token size and overlap
- [ ] Ollama embedding integration
- [ ] Weaviate schema setup and document storage
- [ ] Basic retrieval chain with Claude generation
- [ ] FastAPI server with `/ingest`, `/query`, `/health` endpoints
- [ ] Docker Compose for local dev (Weaviate + Ollama)

---

### Phase 2: Production Quality

Upgrade retrieval precision and enforce answer trustworthiness.

#### 5.2.1 Hybrid Retrieval

| Requirement | Details |
|-------------|---------|
| **FR-013** | System MUST perform hybrid search combining BM25 keyword search and vector semantic search via Weaviate's native hybrid query |
| **FR-014** | System MUST support a configurable `alpha` parameter to control the balance between BM25 (alpha=0) and vector (alpha=1) search |
| **FR-015** | System MUST retrieve top-20 candidate chunks from hybrid search before reranking |

#### 5.2.2 Cross-Encoder Reranking

| Requirement | Details |
|-------------|---------|
| **FR-016** | System MUST rerank retrieved chunks using the Cohere Rerank API |
| **FR-017** | System MUST select the top-5 chunks after reranking to pass as context to the LLM |
| **FR-018** | System MUST fall back to unreranked results if the Cohere API is unavailable |

#### 5.2.3 Citation Enforcement

| Requirement | Details |
|-------------|---------|
| **FR-019** | System MUST instruct Claude to cite specific source documents for every factual claim in the answer |
| **FR-020** | Citations MUST include the source document name and chunk reference (e.g., `[Source: report.pdf, p.3]`) |
| **FR-021** | System MUST instruct Claude to respond with "I don't have enough information to answer this" when retrieved context is insufficient |
| **FR-022** | System MUST return citation metadata in the API response as structured data alongside the answer text |

#### 5.2.4 Deliverables

- [ ] Weaviate hybrid search integration with configurable alpha
- [ ] Cohere Rerank integration with fallback handling
- [ ] Citation-enforcing prompt template
- [ ] Structured API response with answer + citations array
- [ ] Integration tests for retrieval pipeline

---

### Phase 3: Shipability (Evaluation and CI/CD)

Build the evaluation framework and quality-gated deployment pipeline.

#### 5.3.1 Golden Evaluation Dataset

| Requirement | Details |
|-------------|---------|
| **FR-023** | Project MUST include a golden evaluation dataset of 50–200 question-answer pairs |
| **FR-024** | Each evaluation pair MUST include: question, expected answer, and source document references |
| **FR-025** | Evaluation dataset MUST be stored as a versioned JSON file in the repository |

#### 5.3.2 Offline Evaluation with RAGAS

| Requirement | Details |
|-------------|---------|
| **FR-026** | System MUST include an evaluation script that runs the golden dataset through the RAG pipeline |
| **FR-027** | Evaluation MUST measure **faithfulness** — are answers supported by the retrieved context? |
| **FR-028** | Evaluation MUST measure **answer relevancy** — do answers address the question asked? |
| **FR-029** | Evaluation MUST measure **context precision** — are the retrieved chunks relevant to the question? |
| **FR-030** | Evaluation script MUST output a summary report with per-metric scores |

#### 5.3.3 CI/CD Quality Gate

| Requirement | Details |
|-------------|---------|
| **FR-031** | GitHub Actions workflow MUST run the evaluation script on every pull request |
| **FR-032** | Pipeline MUST fail if any RAGAS metric drops below a configurable threshold |
| **FR-033** | Default quality thresholds: faithfulness ≥ 0.8, answer relevancy ≥ 0.7, context precision ≥ 0.7 |
| **FR-034** | Evaluation results MUST be posted as a PR comment for visibility |

#### 5.3.4 API Endpoints (Phase 3)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/evaluate` | POST | Trigger evaluation run against the golden dataset |
| `/evaluate/results` | GET | Retrieve the latest evaluation results |

#### 5.3.5 Deliverables

- [ ] Golden dataset (JSON) with 50+ Q&A pairs
- [ ] RAGAS evaluation script with configurable metrics
- [ ] Evaluation summary report generation
- [ ] GitHub Actions workflow with quality gate
- [ ] PR comment integration for evaluation results
- [ ] `/evaluate` and `/evaluate/results` API endpoints

---

## 6. Data Flow

### 6.1 Ingestion Flow

```
Document (PDF/MD/TXT/HTML)
    │
    ▼
LangChain Document Loader
    │
    ▼
Text Chunking (500-800 tokens, 100-token overlap)
    │
    ▼
Ollama Embedding (nomic-embed-text, 768 dims)
    │
    ▼
Weaviate Storage (vector + text + metadata)
```

### 6.2 Query Flow

```
User Question
    │
    ▼
Weaviate Hybrid Search (BM25 + Vector, alpha configurable)
    │
    ▼
Top-20 Candidate Chunks
    │
    ▼
Cohere Rerank → Top-5 Chunks
    │
    ▼
Claude Generation (with citation-enforcing prompt)
    │
    ▼
Structured Response { answer, citations[] }
```

### 6.3 Evaluation Flow

```
Golden Dataset (Q&A pairs)
    │
    ▼
Run each question through Query Pipeline
    │
    ▼
RAGAS Scoring (faithfulness, relevancy, precision)
    │
    ▼
Quality Gate Check (pass/fail against thresholds)
    │
    ▼
Report + PR Comment
```

---

## 7. Project Structure

```
rag-application/
├── src/
│   ├── api/
│   │   ├── __init__.py
│   │   ├── main.py              # FastAPI app entry point
│   │   ├── routes/
│   │   │   ├── ingest.py        # /ingest endpoint
│   │   │   ├── query.py         # /query endpoint
│   │   │   ├── evaluate.py      # /evaluate endpoints
│   │   │   └── health.py        # /health endpoint
│   │   └── models/
│   │       ├── requests.py      # Request schemas
│   │       └── responses.py     # Response schemas
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── loader.py            # Document loaders (PDF, MD, TXT, HTML)
│   │   ├── chunker.py           # Text chunking logic
│   │   └── embedder.py          # Ollama embedding integration
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── hybrid_search.py     # Weaviate hybrid search
│   │   ├── reranker.py          # Cohere rerank integration
│   │   └── chain.py             # LangChain retrieval chain
│   ├── generation/
│   │   ├── __init__.py
│   │   ├── prompts.py           # Citation-enforcing prompt templates
│   │   └── generator.py         # Claude generation with citations
│   ├── evaluation/
│   │   ├── __init__.py
│   │   ├── dataset.py           # Golden dataset loader
│   │   ├── evaluator.py         # RAGAS evaluation runner
│   │   └── reporter.py          # Evaluation report generator
│   └── config/
│       ├── __init__.py
│       └── settings.py          # App configuration (env vars, defaults)
├── data/
│   ├── documents/               # Source documents for ingestion
│   └── golden_dataset.json      # Evaluation Q&A pairs
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── .github/
│   └── workflows/
│       └── eval-gate.yml        # CI/CD evaluation pipeline
├── docker-compose.yml           # Weaviate + Ollama local dev
├── Dockerfile
├── pyproject.toml
├── requirements.txt
└── README.md
```

---

## 8. API Specification

### 8.1 POST /ingest

**Request:**
```json
{
  "source_type": "pdf | markdown | text | html | url",
  "file": "<uploaded file>",
  "url": "https://example.com/page",
  "metadata": {
    "category": "engineering",
    "tags": ["api", "docs"]
  }
}
```

**Response:**
```json
{
  "status": "success",
  "document_id": "doc_abc123",
  "chunks_created": 42,
  "message": "Document ingested successfully"
}
```

### 8.2 POST /query

**Request:**
```json
{
  "question": "How do I reset my password?",
  "top_k": 5,
  "alpha": 0.5
}
```

**Response:**
```json
{
  "answer": "To reset your password, navigate to Settings > Security and click 'Reset Password'. You will receive a confirmation email within 5 minutes.",
  "citations": [
    {
      "source": "user-guide.pdf",
      "page": 12,
      "chunk_id": "chunk_045",
      "relevance_score": 0.94,
      "text_excerpt": "Navigate to Settings > Security to manage your password..."
    },
    {
      "source": "faq.md",
      "chunk_id": "chunk_112",
      "relevance_score": 0.87,
      "text_excerpt": "Password reset emails are delivered within 5 minutes..."
    }
  ],
  "metadata": {
    "retrieval_method": "hybrid",
    "chunks_retrieved": 20,
    "chunks_after_rerank": 5,
    "model": "claude-sonnet-4-6"
  }
}
```

### 8.3 POST /evaluate

**Request:**
```json
{
  "dataset_path": "data/golden_dataset.json",
  "metrics": ["faithfulness", "answer_relevancy", "context_precision"]
}
```

**Response:**
```json
{
  "status": "completed",
  "results": {
    "faithfulness": 0.85,
    "answer_relevancy": 0.78,
    "context_precision": 0.82
  },
  "thresholds": {
    "faithfulness": 0.8,
    "answer_relevancy": 0.7,
    "context_precision": 0.7
  },
  "passed": true,
  "evaluated_at": "2026-03-29T15:30:00Z",
  "total_questions": 100
}
```

### 8.4 GET /health

**Response:**
```json
{
  "status": "healthy",
  "dependencies": {
    "weaviate": "connected",
    "ollama": "connected",
    "cohere": "reachable",
    "anthropic": "reachable"
  }
}
```

---

## 9. Configuration

All configuration via environment variables with sensible defaults:

| Variable | Default | Description |
|----------|---------|-------------|
| `WEAVIATE_URL` | `http://localhost:8080` | Weaviate instance URL |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API base URL |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | Embedding model name |
| `ANTHROPIC_API_KEY` | — (required) | Anthropic API key for Claude |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-6` | Claude model to use |
| `COHERE_API_KEY` | — (required) | Cohere API key for reranking |
| `CHUNK_SIZE` | `600` | Target chunk size in tokens |
| `CHUNK_OVERLAP` | `100` | Overlap between chunks in tokens |
| `HYBRID_ALPHA` | `0.5` | Balance between BM25 (0) and vector (1) search |
| `RETRIEVAL_TOP_K` | `20` | Number of chunks from hybrid search |
| `RERANK_TOP_N` | `5` | Number of chunks after reranking |
| `EVAL_FAITHFULNESS_THRESHOLD` | `0.8` | Minimum faithfulness score |
| `EVAL_RELEVANCY_THRESHOLD` | `0.7` | Minimum answer relevancy score |
| `EVAL_PRECISION_THRESHOLD` | `0.7` | Minimum context precision score |

---

## 10. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **NFR-001**: Query response time | < 5 seconds for 95th percentile |
| **NFR-002**: Document ingestion | Handle files up to 100MB |
| **NFR-003**: Concurrent queries | Support 10+ concurrent users |
| **NFR-004**: Uptime | 99% availability for API endpoints |
| **NFR-005**: Data privacy | Documents and queries never sent to services beyond configured LLM/reranker APIs |
| **NFR-006**: Observability | Structured logging for all pipeline stages |

---

## 11. Success Criteria

| Criteria | Metric |
|----------|--------|
| **SC-001** | RAGAS faithfulness score ≥ 0.8 on golden dataset |
| **SC-002** | RAGAS answer relevancy score ≥ 0.7 on golden dataset |
| **SC-003** | RAGAS context precision score ≥ 0.7 on golden dataset |
| **SC-004** | End-to-end query latency < 5 seconds (p95) |
| **SC-005** | CI pipeline blocks merges when quality thresholds are not met |
| **SC-006** | Every answer includes at least one source citation |
| **SC-007** | System correctly responds "I don't have enough information" for out-of-scope questions |

---

## 12. Assumptions

- Users have Docker installed for running Weaviate and Ollama locally
- Anthropic API key and Cohere API key are available
- Source documents are in English
- Document corpus size is < 10,000 documents for initial deployment
- GitHub is used for version control and CI/CD

---

## 13. Out of Scope (v1)

- User authentication and authorization
- Multi-tenant document isolation
- Streaming responses
- Conversation memory / multi-turn chat
- Fine-tuning embedding or LLM models
- Deployment to cloud infrastructure (Kubernetes, AWS, etc.)
- Web UI / frontend application

---

## 14. References

- [LangChain RAG Tutorial](https://docs.langchain.com/oss/python/langchain/rag)
- [Cohere Rerank Guide](https://docs.cohere.com/docs/rerank-guide)
- [RAGAS Documentation](https://docs.ragas.io/)
- [Weaviate Hybrid Search](https://weaviate.io/developers/weaviate/search/hybrid)
- [Ollama Embedding Models](https://ollama.com/library/nomic-embed-text)
