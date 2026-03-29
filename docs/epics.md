# Epics: Production-Grade RAG System

**Project:** Ask My Docs — Production RAG Application
**Created:** 2026-03-29
**Source:** [PRD](PRD.md)

---

## Epic Overview

| Epic | Phase | Title | Priority | Status |
|------|-------|-------|----------|--------|
| E-001 | Phase 1 | Document Ingestion Pipeline | P0 | Not Started |
| E-002 | Phase 1 | Basic Retrieval & Generation | P0 | Not Started |
| E-003 | Phase 1 | API Foundation & Infrastructure | P0 | Not Started |
| E-004 | Phase 2 | Hybrid Retrieval | P1 | Not Started |
| E-005 | Phase 2 | Cross-Encoder Reranking | P1 | Not Started |
| E-006 | Phase 2 | Citation Enforcement | P1 | Not Started |
| E-007 | Phase 3 | Golden Evaluation Dataset | P2 | Not Started |
| E-008 | Phase 3 | RAGAS Evaluation Pipeline | P2 | Not Started |
| E-009 | Phase 3 | CI/CD Quality Gate | P2 | Not Started |

---

## Phase 1: Fundamental RAG (MVP)

### E-001: Document Ingestion Pipeline

**Goal:** Ingest domain-specific documents, chunk them, embed them, and store them in Weaviate for retrieval.

**User Story:** As a system administrator, I want to upload documents (PDF, Markdown, plain text, HTML) so that the system can index them and make them searchable.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-001 | System MUST ingest PDF documents using LangChain PDF loaders |
| FR-002 | System MUST ingest Markdown files (.md) |
| FR-003 | System MUST ingest plain text files (.txt) |
| FR-004 | System MUST ingest web pages / HTML content |
| FR-005 | System MUST chunk documents into 500–800 token segments with 100-token overlap |
| FR-006 | System MUST generate embeddings using Ollama `nomic-embed-text` model |
| FR-007 | System MUST store document chunks and embeddings in Weaviate |
| FR-008 | System MUST preserve source metadata (filename, page number, chunk index) with each chunk |

**Data Flow:**
```
Document (PDF/MD/TXT/HTML)
    → LangChain Document Loader
    → Text Chunking (500-800 tokens, 100-token overlap)
    → Ollama Embedding (nomic-embed-text, 768 dims)
    → Weaviate Storage (vector + text + metadata)
```

**Acceptance Criteria:**
- [ ] PDF documents are parsed and chunked correctly (including multi-page PDFs)
- [ ] Markdown, plain text, and HTML files are ingested without data loss
- [ ] Chunk sizes stay within 500–800 token range
- [ ] Each chunk retains source metadata (filename, page, chunk index)
- [ ] Embeddings are generated via Ollama and stored in Weaviate
- [ ] Duplicate document ingestion is handled gracefully

**Key Files:**
- `src/ingestion/loader.py` — Document loaders
- `src/ingestion/chunker.py` — Text chunking logic
- `src/ingestion/embedder.py` — Ollama embedding integration

**Dependencies:** None (foundational epic)

---

### E-002: Basic Retrieval & Generation

**Goal:** Accept user questions, retrieve relevant chunks from Weaviate, and generate answers using Claude.

**User Story:** As a user, I want to ask a natural language question and receive an accurate answer drawn from the ingested documents.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-009 | System MUST accept natural language queries via a REST API (`POST /query`) |
| FR-010 | System MUST perform vector similarity search in Weaviate to retrieve top-K relevant chunks |
| FR-011 | System MUST pass retrieved chunks as context to Claude for answer generation |
| FR-012 | System MUST return the generated answer along with source document references |

**Data Flow:**
```
User Question
    → Weaviate Vector Search (top-K chunks)
    → Claude Generation (question + context)
    → Response { answer, sources[] }
```

**Acceptance Criteria:**
- [ ] User can submit a question via `POST /query` and receive a relevant answer
- [ ] Retrieved chunks are semantically relevant to the question
- [ ] Claude generates coherent answers grounded in the retrieved context
- [ ] Source document references are included in the response
- [ ] System handles questions with no relevant documents gracefully

**Key Files:**
- `src/retrieval/hybrid_search.py` — Weaviate search (vector-only in Phase 1)
- `src/retrieval/chain.py` — LangChain retrieval chain
- `src/generation/generator.py` — Claude generation
- `src/generation/prompts.py` — Prompt templates

**Dependencies:** E-001 (documents must be ingested first)

---

### E-003: API Foundation & Infrastructure

**Goal:** Set up the FastAPI server, Docker Compose environment, project configuration, and health checks.

**User Story:** As a developer, I want a working local development environment with all dependencies (Weaviate, Ollama) running via Docker Compose so I can develop and test the system.

**Requirements:**

| ID | Requirement |
|----|-------------|
| — | FastAPI application with `/ingest`, `/query`, `/health` endpoints |
| — | Docker Compose with Weaviate and Ollama services |
| — | Environment-based configuration via `settings.py` |
| — | Request/response Pydantic models |
| — | Structured logging for all pipeline stages |

**API Endpoints:**

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ingest` | POST | Upload and ingest documents |
| `/query` | POST | Submit a question and receive an answer |
| `/health` | GET | Health check for service and dependencies |

**Acceptance Criteria:**
- [ ] `docker-compose up` starts Weaviate and Ollama with no manual steps
- [ ] FastAPI server starts and serves all Phase 1 endpoints
- [ ] `/health` reports connectivity status for Weaviate, Ollama, Cohere, Anthropic
- [ ] All configuration is driven by environment variables with sensible defaults
- [ ] Request validation returns clear error messages for malformed input

**Key Files:**
- `src/api/main.py` — FastAPI entry point
- `src/api/routes/` — Route handlers
- `src/api/models/` — Pydantic request/response schemas
- `src/config/settings.py` — Configuration
- `docker-compose.yml` — Local dev environment
- `pyproject.toml` / `requirements.txt` — Dependencies

**Dependencies:** None (foundational epic, developed in parallel with E-001)

---

## Phase 2: Production Quality

### E-004: Hybrid Retrieval

**Goal:** Upgrade retrieval from vector-only to hybrid search combining BM25 keyword matching and vector semantic search via Weaviate's native hybrid query.

**User Story:** As a user, I want the system to find relevant documents even when my question uses different terminology than the source documents, AND when I use exact keywords or phrases.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-013 | System MUST perform hybrid search combining BM25 keyword search and vector semantic search via Weaviate's native hybrid query |
| FR-014 | System MUST support a configurable `alpha` parameter to control balance between BM25 (alpha=0) and vector (alpha=1) search |
| FR-015 | System MUST retrieve top-20 candidate chunks from hybrid search before reranking |

**How It Works:**
```
User Question
    │
    ├──→ BM25 Keyword Search (exact term matching)
    │
    ├──→ Vector Semantic Search (meaning-based matching)
    │
    └──→ Weaviate Hybrid Fusion (alpha-weighted combination)
            │
            ▼
      Top-20 Candidate Chunks
```

- `alpha = 0.0` → pure BM25 (keyword only)
- `alpha = 0.5` → equal blend (recommended default)
- `alpha = 1.0` → pure vector (semantic only)

**Acceptance Criteria:**
- [ ] Hybrid search returns results that combine keyword and semantic relevance
- [ ] `alpha` parameter is configurable per query and via environment variable
- [ ] Queries with exact terms (e.g., error codes, product names) return better results than vector-only search
- [ ] Queries with paraphrased language still return semantically relevant results
- [ ] Top-20 candidates are returned for downstream reranking

**Key Files:**
- `src/retrieval/hybrid_search.py` — Upgrade from vector-only to hybrid

**Dependencies:** E-002 (basic retrieval must exist first)

---

### E-005: Cross-Encoder Reranking

**Goal:** Improve retrieval precision by reranking hybrid search results with the Cohere Rerank cross-encoder.

**User Story:** As a user, I want the most precisely relevant document chunks used to generate my answer, not just the broadly similar ones.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-016 | System MUST rerank retrieved chunks using the Cohere Rerank API |
| FR-017 | System MUST select the top-5 chunks after reranking to pass as context to the LLM |
| FR-018 | System MUST fall back to unreranked results if the Cohere API is unavailable |

**How It Works:**
```
Top-20 Chunks (from hybrid search)
    │
    ▼
Cohere Rerank API
(cross-encoder scores each chunk against the original query)
    │
    ▼
Top-5 Chunks (highest relevance scores)
    │
    ▼
Passed to Claude for generation
```

**Why cross-encoder reranking matters:**
- Hybrid search casts a wide net (top-20) — fast but imprecise
- Cross-encoder evaluates each chunk **jointly with the query** — slower but much more accurate
- This two-stage approach balances speed and precision

**Acceptance Criteria:**
- [ ] Cohere Rerank is called with the query and top-20 chunks
- [ ] Only top-5 reranked chunks are passed to Claude
- [ ] Reranking measurably improves answer quality (testable via RAGAS)
- [ ] System degrades gracefully when Cohere API is unavailable (uses unreranked top-5)
- [ ] Rerank latency is logged for observability

**Key Files:**
- `src/retrieval/reranker.py` — Cohere rerank integration

**Dependencies:** E-004 (hybrid retrieval provides the candidates to rerank)

---

### E-006: Citation Enforcement

**Goal:** Force the LLM to cite specific source documents for every factual claim, and refuse to answer when context is insufficient.

**User Story:** As a user, I want to trust the system's answers by seeing exactly which documents support each claim, so I can verify the information myself.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-019 | System MUST instruct Claude to cite specific source documents for every factual claim in the answer |
| FR-020 | Citations MUST include the source document name and chunk reference (e.g., `[Source: report.pdf, p.3]`) |
| FR-021 | System MUST instruct Claude to respond with "I don't have enough information to answer this" when retrieved context is insufficient |
| FR-022 | System MUST return citation metadata in the API response as structured data alongside the answer text |

**Response Format:**
```json
{
  "answer": "To reset your password, navigate to Settings > Security...",
  "citations": [
    {
      "source": "user-guide.pdf",
      "page": 12,
      "chunk_id": "chunk_045",
      "relevance_score": 0.94,
      "text_excerpt": "Navigate to Settings > Security..."
    }
  ]
}
```

**Acceptance Criteria:**
- [ ] Every factual claim in the answer has a corresponding citation
- [ ] Citations include source filename, page/chunk reference, and text excerpt
- [ ] Out-of-scope questions receive "I don't have enough information" response
- [ ] Citation metadata is returned as a structured JSON array (not just inline text)
- [ ] Prompt template enforces citation behavior consistently

**Key Files:**
- `src/generation/prompts.py` — Citation-enforcing prompt template
- `src/generation/generator.py` — Citation parsing and structured output
- `src/api/models/responses.py` — Citation response schema

**Dependencies:** E-005 (reranked chunks are the input for citation-grounded generation)

---

## Phase 3: Shipability

### E-007: Golden Evaluation Dataset

**Goal:** Create a curated dataset of question-answer pairs that serves as the ground truth for evaluating RAG system quality.

**User Story:** As a developer, I want a versioned set of expected Q&A pairs so I can measure whether changes to the system improve or degrade answer quality.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-023 | Project MUST include a golden evaluation dataset of 50–200 question-answer pairs |
| FR-024 | Each evaluation pair MUST include: question, expected answer, and source document references |
| FR-025 | Evaluation dataset MUST be stored as a versioned JSON file in the repository |

**Dataset Format:**
```json
[
  {
    "id": "eval_001",
    "question": "How do I reset my password?",
    "expected_answer": "Navigate to Settings > Security and click 'Reset Password'.",
    "source_documents": ["user-guide.pdf"],
    "source_pages": [12],
    "category": "account-management",
    "difficulty": "easy"
  }
]
```

**Acceptance Criteria:**
- [ ] Dataset contains at least 50 Q&A pairs
- [ ] Each entry includes question, expected answer, and source references
- [ ] Dataset covers a range of question types (factual, procedural, comparison)
- [ ] Dataset includes edge cases (out-of-scope questions, ambiguous questions)
- [ ] Dataset is stored at `data/golden_dataset.json` and versioned in git

**Key Files:**
- `data/golden_dataset.json` — The evaluation dataset
- `src/evaluation/dataset.py` — Dataset loader and validator

**Dependencies:** E-001 (documents must be ingested to create meaningful Q&A pairs)

---

### E-008: RAGAS Evaluation Pipeline

**Goal:** Build an automated evaluation script that runs the golden dataset through the RAG pipeline and scores quality using RAGAS metrics.

**User Story:** As a developer, I want to run a single command that evaluates my RAG system's quality and tells me exactly where it's strong and where it's weak.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-026 | System MUST include an evaluation script that runs the golden dataset through the RAG pipeline |
| FR-027 | Evaluation MUST measure **faithfulness** — are answers supported by the retrieved context? |
| FR-028 | Evaluation MUST measure **answer relevancy** — do answers address the question asked? |
| FR-029 | Evaluation MUST measure **context precision** — are the retrieved chunks relevant to the question? |
| FR-030 | Evaluation script MUST output a summary report with per-metric scores |

**Metrics Explained:**

| Metric | What It Measures | Threshold |
|--------|-----------------|-----------|
| Faithfulness | Are claims in the answer supported by the retrieved context? | ≥ 0.8 |
| Answer Relevancy | Does the answer actually address the user's question? | ≥ 0.7 |
| Context Precision | Are the retrieved chunks relevant to the question? | ≥ 0.7 |

**Evaluation Report Output:**
```
=== RAG Evaluation Report ===
Date: 2026-03-29T15:30:00Z
Dataset: data/golden_dataset.json (100 questions)

Metrics:
  Faithfulness:      0.85  ✅ (threshold: 0.80)
  Answer Relevancy:  0.78  ✅ (threshold: 0.70)
  Context Precision:  0.82  ✅ (threshold: 0.70)

Overall: PASSED
```

**Acceptance Criteria:**
- [ ] Evaluation script runs all golden dataset questions through the full RAG pipeline
- [ ] RAGAS faithfulness, answer relevancy, and context precision are computed
- [ ] Summary report is generated with per-metric scores and pass/fail status
- [ ] Individual question scores are available for debugging weak spots
- [ ] Script can be run via CLI: `python -m src.evaluation.evaluator`

**Key Files:**
- `src/evaluation/evaluator.py` — RAGAS evaluation runner
- `src/evaluation/reporter.py` — Report generation

**Dependencies:** E-007 (golden dataset must exist), E-006 (full pipeline must be operational)

---

### E-009: CI/CD Quality Gate

**Goal:** Integrate the evaluation pipeline into GitHub Actions so that pull requests are automatically blocked if RAG quality drops below thresholds.

**User Story:** As a team lead, I want the CI pipeline to automatically reject changes that degrade answer quality, so we never ship a regression.

**Requirements:**

| ID | Requirement |
|----|-------------|
| FR-031 | GitHub Actions workflow MUST run the evaluation script on every pull request |
| FR-032 | Pipeline MUST fail if any RAGAS metric drops below a configurable threshold |
| FR-033 | Default quality thresholds: faithfulness ≥ 0.8, answer relevancy ≥ 0.7, context precision ≥ 0.7 |
| FR-034 | Evaluation results MUST be posted as a PR comment for visibility |

**CI/CD Flow:**
```
Pull Request Opened
    │
    ▼
GitHub Actions Triggered
    │
    ▼
Start Weaviate + Ollama (Docker services)
    │
    ▼
Ingest test documents
    │
    ▼
Run RAGAS evaluation against golden dataset
    │
    ▼
Check scores against thresholds
    │
    ├── All metrics pass → ✅ PR check passes
    │
    └── Any metric fails → ❌ PR check fails + comment posted
```

**PR Comment Format:**
```markdown
## 🔍 RAG Evaluation Results

| Metric | Score | Threshold | Status |
|--------|-------|-----------|--------|
| Faithfulness | 0.85 | 0.80 | ✅ Pass |
| Answer Relevancy | 0.78 | 0.70 | ✅ Pass |
| Context Precision | 0.82 | 0.70 | ✅ Pass |

**Overall: ✅ Passed**
```

**Acceptance Criteria:**
- [ ] GitHub Actions workflow runs on every PR to `main`
- [ ] Workflow starts Weaviate and Ollama as service containers
- [ ] Evaluation script runs and scores are checked against thresholds
- [ ] PR is blocked (check fails) if any metric is below threshold
- [ ] Evaluation results are posted as a PR comment
- [ ] Thresholds are configurable via environment variables

**Key Files:**
- `.github/workflows/eval-gate.yml` — CI/CD pipeline
- `src/evaluation/evaluator.py` — Called by the pipeline

**Dependencies:** E-008 (evaluation pipeline must be built first)

---

## Epic Dependency Graph

```
Phase 1 (MVP):
  E-003 (API & Infra) ──┐
                         ├──→ E-002 (Basic Retrieval)
  E-001 (Ingestion) ────┘

Phase 2 (Production Quality):
  E-002 ──→ E-004 (Hybrid Retrieval)
  E-004 ──→ E-005 (Reranking)
  E-005 ──→ E-006 (Citations)

Phase 3 (Shipability):
  E-001 ──→ E-007 (Golden Dataset)
  E-006 + E-007 ──→ E-008 (RAGAS Evaluation)
  E-008 ──→ E-009 (CI/CD Gate)
```

---

## Configuration Reference

| Variable | Default | Used By |
|----------|---------|---------|
| `WEAVIATE_URL` | `http://localhost:8080` | E-001, E-002, E-004 |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | E-001 |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | E-001 |
| `ANTHROPIC_API_KEY` | — (required) | E-002, E-006 |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-6` | E-002, E-006 |
| `COHERE_API_KEY` | — (required) | E-005 |
| `CHUNK_SIZE` | `600` | E-001 |
| `CHUNK_OVERLAP` | `100` | E-001 |
| `HYBRID_ALPHA` | `0.5` | E-004 |
| `RETRIEVAL_TOP_K` | `20` | E-004 |
| `RERANK_TOP_N` | `5` | E-005 |
| `EVAL_FAITHFULNESS_THRESHOLD` | `0.8` | E-008, E-009 |
| `EVAL_RELEVANCY_THRESHOLD` | `0.7` | E-008, E-009 |
| `EVAL_PRECISION_THRESHOLD` | `0.7` | E-008, E-009 |
