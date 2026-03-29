# RAG Enterprise Engine

A production-grade Retrieval Augmented Generation (RAG) system that enables users to ask natural language questions against a corpus of ingested documents and receive accurate, cited answers.

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11+ |
| API Framework | FastAPI |
| Orchestration | LangChain |
| Vector Store | Weaviate (hybrid BM25 + vector search) |
| Embeddings | Ollama (`nomic-embed-text`) |
| LLM | Anthropic Claude |
| Reranker | Cohere Rerank API |
| Evaluation | RAGAS |
| CI/CD | GitHub Actions |

## Architecture

```
User → FastAPI → Retrieval Pipeline → Weaviate (Hybrid Search)
                      ↓
              Cohere (Rerank) → Claude (Generate + Cite) → Response
```

**Key capabilities:**
- Hybrid retrieval (BM25 + semantic vector search)
- Cross-encoder reranking for precision
- Citation enforcement — every answer includes source references
- RAGAS quality-gated CI/CD pipeline

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/ingest` | Upload and ingest documents (PDF, MD, TXT, HTML) |
| POST | `/query` | Submit a question, receive an answer with citations |
| POST | `/evaluate` | Run RAGAS evaluation against golden dataset |
| GET | `/evaluate/results` | Retrieve latest evaluation results |
| GET | `/health` | Health check |

## Getting Started

### Prerequisites

- Docker and Docker Compose
- API keys: [Anthropic](https://console.anthropic.com/), [Cohere](https://dashboard.cohere.com/)

### Local Development

```bash
# 1. Clone the repo
git clone git@github.com:praveen0raj/rag-enterprise-engine.git
cd rag-enterprise-engine

# 2. Configure environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY and COHERE_API_KEY

# 3. Start all services
docker-compose up

# 4. Pull the embedding model (first time only)
docker-compose exec ollama ollama pull nomic-embed-text
```

The API will be available at `http://localhost:8000`.

### Kubernetes Deployment

```bash
# Create namespace and apply all manifests
kubectl apply -f k8s/

# Update secrets with your API keys
kubectl create secret generic rag-app-secrets \
  --from-literal=ANTHROPIC_API_KEY=your-key \
  --from-literal=COHERE_API_KEY=your-key \
  -n rag-app --dry-run=client -o yaml | kubectl apply -f -
```

## Project Structure

```
rag-enterprise-engine/
├── src/
│   ├── api/              # FastAPI endpoints
│   ├── ingestion/        # Document loading, chunking, embedding
│   ├── retrieval/        # Hybrid search, reranking, chain
│   ├── generation/       # Claude generation with citations
│   ├── evaluation/       # RAGAS evaluation pipeline
│   └── config/           # Settings and configuration
├── data/
│   ├── documents/        # Source documents for ingestion
│   └── golden_dataset.json
├── tests/
│   ├── unit/
│   └── integration/
├── k8s/                  # Kubernetes manifests
├── .github/workflows/    # CI, eval gate, deploy pipelines
├── Dockerfile
├── docker-compose.yml
└── requirements.txt
```

## CI/CD Pipelines

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **CI** | Push/PR to main | Lint, type check, unit tests (80% coverage), integration tests, Docker build |
| **Eval Gate** | PR touching `src/` | RAGAS evaluation — blocks merge if quality thresholds fail |
| **Deploy** | Push to main | Build image, push to GHCR, deploy to Kubernetes |

### Quality Thresholds

| Metric | Threshold |
|--------|-----------|
| RAGAS Faithfulness | >= 0.8 |
| RAGAS Answer Relevancy | >= 0.7 |
| RAGAS Context Precision | >= 0.7 |

## Configuration

All configuration is via environment variables. See [`.env.example`](.env.example) for the full list.

## License

MIT
