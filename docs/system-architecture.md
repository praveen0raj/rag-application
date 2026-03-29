# System Architecture — RAG Enterprise Engine

## 1. High-Level System Overview

```mermaid
graph TB
    subgraph Clients["Client Layer"]
        WebUI["Web UI / Dashboard"]
        API_Client["API Clients"]
        CLI["CLI Tools"]
    end

    subgraph Gateway["API Gateway"]
        FastAPI["FastAPI Server<br/>:8000"]
        Auth["Auth Middleware"]
        RateLimit["Rate Limiter"]
    end

    subgraph Core["Core Application Layer"]
        Ingest["Ingestion Pipeline"]
        Retrieval["Retrieval Pipeline"]
        Generation["Generation Pipeline"]
        Evaluation["Evaluation Pipeline"]
    end

    subgraph DataStores["Data & Storage Layer"]
        Weaviate["Weaviate<br/>Hybrid Vector Store<br/>:8080"]
        DocStore["Document Storage<br/>/data/documents"]
        GoldenDS["Golden Dataset<br/>golden_dataset.json"]
    end

    subgraph ExternalServices["External Services"]
        Ollama["Ollama<br/>Embedding Server<br/>:11434"]
        Anthropic["Anthropic API<br/>Claude LLM"]
        Cohere["Cohere API<br/>Reranker"]
    end

    subgraph CICD["CI/CD & Observability"]
        GHA["GitHub Actions"]
        QualityGate["RAGAS Quality Gate"]
        GHCR["GitHub Container Registry"]
    end

    Clients --> Gateway
    FastAPI --> Ingest
    FastAPI --> Retrieval
    FastAPI --> Evaluation

    Ingest --> Ollama
    Ingest --> Weaviate
    Ingest --> DocStore

    Retrieval --> Weaviate
    Retrieval --> Cohere
    Retrieval --> Generation
    Generation --> Anthropic

    Evaluation --> GoldenDS
    Evaluation --> Retrieval

    GHA --> QualityGate
    QualityGate --> Evaluation
    GHA --> GHCR
```

## 2. Document Ingestion Flow

```mermaid
flowchart LR
    subgraph Input["Document Sources"]
        PDF["PDF Files"]
        MD["Markdown"]
        TXT["Plain Text"]
        HTML["HTML / Web Pages"]
    end

    subgraph Processing["Ingestion Pipeline"]
        Loader["Document Loader<br/>(LangChain)"]
        Chunker["Text Chunker<br/>500-800 tokens<br/>100 token overlap"]
        Embedder["Embedding Generator<br/>Ollama nomic-embed-text<br/>768 dimensions"]
    end

    subgraph Storage["Vector Store"]
        Weaviate["Weaviate<br/>• Document chunks<br/>• Vector embeddings<br/>• BM25 index<br/>• Metadata"]
    end

    Input --> Loader --> Chunker --> Embedder --> Weaviate
```

## 3. Query & Retrieval Flow

```mermaid
flowchart TB
    User["User Query"] --> API["POST /query"]

    subgraph RetrievalPipeline["Retrieval Pipeline"]
        direction TB
        HybridSearch["Hybrid Search<br/>━━━━━━━━━━━━<br/>BM25 (keyword) + Vector (semantic)<br/>alpha = 0.5 | top_k = 20"]
        Rerank["Cross-Encoder Reranking<br/>━━━━━━━━━━━━<br/>Cohere Rerank API<br/>top_n = 5"]
        ContextBuild["Context Builder<br/>━━━━━━━━━━━━<br/>Assemble ranked chunks<br/>with source metadata"]
    end

    subgraph GenerationPipeline["Generation Pipeline"]
        direction TB
        Prompt["Citation-Enforcing Prompt<br/>━━━━━━━━━━━━<br/>System prompt requires<br/>source references"]
        LLM["Claude LLM<br/>━━━━━━━━━━━━<br/>Generate answer<br/>with inline citations"]
        Validate["Response Validation<br/>━━━━━━━━━━━━<br/>Verify citations present<br/>Flag unsupported claims"]
    end

    API --> HybridSearch
    HybridSearch -->|"20 candidates"| Rerank
    Rerank -->|"5 best chunks"| ContextBuild
    ContextBuild --> Prompt
    Prompt --> LLM
    LLM --> Validate
    Validate --> Response["Cited Answer + Sources"]

    Weaviate["Weaviate"] -.-> HybridSearch
    Cohere["Cohere API"] -.-> Rerank
    Anthropic["Anthropic API"] -.-> LLM
```

## 4. Evaluation & Quality Gate Flow

```mermaid
flowchart LR
    subgraph Trigger["Triggers"]
        PR["Pull Request<br/>to main"]
        Manual["POST /evaluate"]
    end

    subgraph EvalPipeline["Evaluation Pipeline"]
        Dataset["Golden Dataset<br/>Q&A pairs with<br/>ground truth"]
        Runner["RAGAS Evaluator<br/>Run queries against<br/>retrieval + generation"]
        Metrics["Compute Metrics<br/>• Faithfulness<br/>• Answer Relevancy<br/>• Context Precision"]
    end

    subgraph Gate["Quality Gate"]
        Check{"Thresholds Met?<br/>Faithful ≥ 0.8<br/>Relevancy ≥ 0.7<br/>Precision ≥ 0.7"}
        Pass["✅ Merge Allowed"]
        Fail["❌ Merge Blocked"]
    end

    Trigger --> Dataset --> Runner --> Metrics --> Check
    Check -->|"Yes"| Pass
    Check -->|"No"| Fail
```

## 5. Infrastructure & Deployment

```mermaid
graph TB
    subgraph K8s["Kubernetes Cluster — namespace: rag-app"]
        subgraph AppTier["Application Tier"]
            App1["rag-app Pod 1"]
            App2["rag-app Pod 2"]
        end

        subgraph DataTier["Data Tier (StatefulSets)"]
            WeaviatePod["Weaviate Pod<br/>PVC: 10Gi"]
            OllamaPod["Ollama Pod<br/>PVC: 10Gi"]
        end

        subgraph Networking["Networking"]
            Ingress["Nginx Ingress<br/>rag-app.local"]
            AppSvc["app Service<br/>ClusterIP :80"]
            WeaviateSvc["weaviate Service<br/>ClusterIP :8080"]
            OllamaSvc["ollama Service<br/>ClusterIP :11434"]
        end

        ConfigMap["ConfigMap<br/>rag-app-config"]
        Secret["Secret<br/>rag-app-secrets"]
    end

    subgraph External["External"]
        GHCR["GHCR<br/>Container Images"]
        AnthropicAPI["Anthropic API"]
        CohereAPI["Cohere API"]
    end

    Internet["Internet"] --> Ingress
    Ingress --> AppSvc
    AppSvc --> App1 & App2
    App1 & App2 --> WeaviateSvc --> WeaviatePod
    App1 & App2 --> OllamaSvc --> OllamaPod
    App1 & App2 --> AnthropicAPI
    App1 & App2 --> CohereAPI
    ConfigMap -.-> App1 & App2
    Secret -.-> App1 & App2
    GHCR -.->|"pull image"| App1 & App2
```

## 6. CI/CD Pipeline

```mermaid
flowchart LR
    subgraph CI["CI Pipeline"]
        Lint["Ruff Lint<br/>+ Format Check"]
        TypeCheck["mypy<br/>Type Check"]
        Unit["Unit Tests<br/>≥ 80% coverage"]
        Integration["Integration Tests<br/>Weaviate service"]
        Build["Docker Build<br/>+ Cache"]
    end

    subgraph EvalGate["Eval Quality Gate"]
        Ingest["Ingest Test Docs"]
        Eval["RAGAS Evaluation"]
        Threshold["Check Thresholds"]
    end

    subgraph Deploy["Deploy Pipeline"]
        Push["Push to GHCR"]
        K8sDeploy["kubectl apply"]
        Rollout["Rollout Status"]
    end

    Push_Main["Push to main"] --> CI
    PR["Pull Request"] --> CI
    PR --> EvalGate

    Lint --> TypeCheck --> Unit --> Integration --> Build
    Ingest --> Eval --> Threshold

    Build -->|"main only"| Push --> K8sDeploy --> Rollout
```

## 7. Component Communication Matrix

| Source | Destination | Protocol | Port | Purpose |
|--------|------------|----------|------|---------|
| Client | FastAPI | HTTP/REST | 8000 | API requests |
| FastAPI | Weaviate | HTTP | 8080 | Vector search & storage |
| FastAPI | Weaviate | gRPC | 50051 | Batch operations |
| FastAPI | Ollama | HTTP | 11434 | Embedding generation |
| FastAPI | Anthropic | HTTPS | 443 | LLM generation |
| FastAPI | Cohere | HTTPS | 443 | Reranking |
| GitHub Actions | GHCR | HTTPS | 443 | Image push/pull |
| Ingress | App Service | HTTP | 80 | Traffic routing |

## 8. Data Flow Summary

```
Documents → Load → Chunk → Embed (Ollama) → Store (Weaviate)
                                                    ↓
User Query → Hybrid Search (BM25 + Vector) → Rerank (Cohere) → Generate (Claude) → Cited Answer
                                                                                        ↓
                                                              Evaluate (RAGAS) → Quality Gate → Deploy
```
