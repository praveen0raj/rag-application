# Requirements Analysis: E-001 Document Ingestion Pipeline

## Abstract

The Document Ingestion Pipeline is the foundational component of the "Ask My Docs" production RAG system, responsible for transforming raw documents into searchable, embedded vector representations stored in Weaviate. The system must accept four document formats (PDF, Markdown, plain text, and HTML/web pages), parse them using LangChain document loaders, and split them into 500-800 token chunks with 100-token overlap to preserve context across boundaries. Each chunk is embedded using the Ollama nomic-embed-text model (768 dimensions) and stored in Weaviate along with the original text and source metadata (filename, location info per format, chunk index). Without this pipeline, no downstream retrieval or answer generation is possible, making it the critical-path dependency for the entire RAG system. The pipeline is exposed via a `POST /ingest` FastAPI endpoint that accepts file uploads and URLs, returning a structured response with document ID and chunk count. Completion criteria include successful ingestion of all four document types, correct chunking with overlap, verified embedding generation, and metadata preservation in Weaviate.

## Problem Statement

The RAG system cannot answer questions without a corpus of ingested, chunked, and embedded documents stored in a vector database. Currently no ingestion infrastructure exists -- the project has a PRD and epic structure but no source code. This pipeline must be built first to unblock all downstream retrieval, generation, and evaluation work (Phases 1-3 of the PRD).

## Goals and Value

- **Enable document-based Q&A**: Transform static documents into a queryable knowledge base, enabling users to get answers from their own document corpus
- **Support diverse document formats**: Accept PDF, Markdown, plain text, and HTML to cover the most common enterprise document types without requiring manual format conversion
- **Preserve provenance**: Maintain source metadata through the entire pipeline so that downstream citation features (Phase 2) can reference exact document locations
- **Establish embedding infrastructure**: Set up the Ollama embedding integration and Weaviate schema that will be reused by the retrieval pipeline
- **Configurable chunking**: Allow token size and overlap to be tuned without code changes, enabling optimization based on evaluation results in Phase 3

## Stakeholders

| Stakeholder | Role/Concerns |
|------------|---------------|
| End Users (document owners) | Need to upload documents in various formats and have them become searchable; care about ingestion reliability and format support |
| End Users (question askers) | Depend on ingestion quality for accurate retrieval; indirectly affected by chunk quality and metadata completeness |
| Backend Engineers | Implement and maintain the ingestion pipeline; concerned with code modularity, LangChain integration patterns, and testability |
| DevOps/Infrastructure | Manage Weaviate and Ollama services; concerned with Docker Compose setup, resource usage, and service health |
| Product Owner | Needs the pipeline delivered as a prerequisite for the MVP; concerned with scope and timeline |
| Retrieval Pipeline (downstream system) | Depends on correctly embedded and indexed documents with consistent schema in Weaviate |

## Use Cases

### UC-1: Ingest a PDF Document

- **Actor**: Document owner (via API client or future UI)
- **Trigger**: User has a PDF document they want to make searchable in the RAG system
- **Purpose**: Upload and process a PDF so its content becomes queryable
- **Basic Flow**:
  1. User sends a `POST /ingest` request with the PDF file and `source_type: "pdf"`
  2. System validates the file type and size
  3. LangChain PDF loader extracts text content, preserving page boundaries
  4. Chunker splits extracted text into 500-800 token segments with 100-token overlap
  5. Each chunk is assigned metadata: filename, page number, chunk index
  6. Ollama generates a 768-dimension embedding for each chunk
  7. Chunks, embeddings, and metadata are stored in Weaviate
  8. System returns a response with document ID and total chunks created
- **Success Criteria**: All pages of the PDF are extracted, chunked, embedded, and stored with correct page-level metadata
- **Current Workaround**: None -- no ingestion system exists

### UC-2: Ingest a Markdown File

- **Actor**: Document owner (via API client)
- **Trigger**: User has a Markdown document (e.g., internal wiki page, README) to ingest
- **Purpose**: Make Markdown content searchable
- **Basic Flow**:
  1. User sends a `POST /ingest` request with the .md file and `source_type: "markdown"`
  2. System validates the file
  3. LangChain Markdown loader parses the content
  4. Chunker splits text into configured token segments with overlap
  5. Metadata assigned: filename, chunk index (page number not applicable for Markdown)
  6. Embeddings generated and stored in Weaviate with text and metadata
  7. System returns success response
- **Success Criteria**: Markdown content is correctly parsed (headers, lists, code blocks handled), chunked, and stored
- **Current Workaround**: None

### UC-3: Ingest a Plain Text File

- **Actor**: Document owner (via API client)
- **Trigger**: User has a .txt file to make searchable
- **Purpose**: Ingest unstructured text content
- **Basic Flow**:
  1. User sends a `POST /ingest` request with the .txt file and `source_type: "text"`
  2. System validates the file
  3. LangChain text loader reads the content
  4. Chunker splits into token segments with overlap
  5. Metadata assigned: filename, chunk index
  6. Embeddings generated and stored in Weaviate
  7. System returns success response
- **Success Criteria**: Text content is fully ingested without data loss
- **Current Workaround**: None

### UC-4: Ingest a Web Page / HTML Content

- **Actor**: Document owner (via API client)
- **Trigger**: User wants to make a web page's content searchable
- **Purpose**: Ingest content from a URL or uploaded HTML file
- **Basic Flow**:
  1. User sends a `POST /ingest` request with either a URL or HTML file, `source_type: "html"` or `source_type: "url"`
  2. System fetches the URL content (if URL provided) or reads the uploaded HTML
  3. LangChain HTML loader extracts text content, stripping navigation, scripts, and boilerplate
  4. Chunker splits extracted text into token segments with overlap
  5. Metadata assigned: source URL or filename, chunk index
  6. Embeddings generated and stored in Weaviate
  7. System returns success response
- **Success Criteria**: Meaningful text content is extracted from HTML (not tags/scripts), chunked, and stored with source URL preserved
- **Current Workaround**: None

### UC-5: Re-ingest an Updated Document

- **Actor**: Document owner
- **Trigger**: A previously ingested document has been updated and needs to be re-indexed
- **Purpose**: Ensure the latest version of a document is reflected in the knowledge base
- **Basic Flow**:
  1. User sends a `POST /ingest` request with the updated document
  2. System detects an existing document with the same filename/identifier
  3. System deletes all existing chunks for that document from Weaviate
  4. System processes the document through the standard pipeline (load, chunk, embed, store)
  5. System returns success response with new chunk count
- **Success Criteria**: Old chunks are removed, updated document content is available for retrieval, no stale data remains
- **Current Workaround**: None

### UC-6: Ingest a Large File (> 20MB) Asynchronously

- **Actor**: Document owner (via API client)
- **Trigger**: User uploads a document larger than 20MB (e.g., a 50MB technical manual PDF)
- **Purpose**: Ingest large documents without request timeouts
- **Basic Flow**:
  1. User sends a `POST /ingest` request with a file > 20MB
  2. System validates the file and detects it exceeds the sync threshold
  3. System returns immediately with a `job_id` and status `"processing"`
  4. System processes the document asynchronously (load, chunk, embed, store)
  5. User polls `GET /ingest/status/{job_id}` to check progress
  6. When complete, status returns `"completed"` with document_id and chunks_created
- **Success Criteria**: Large file is fully ingested; user can track progress via polling; no request timeout occurs
- **Current Workaround**: None

## Functional Requirements

### Document Loading

- [ ] **FR-001**: Ingest PDF documents using LangChain PDF loaders, extracting text while preserving page boundary information
- [ ] **FR-002**: Ingest Markdown files (.md) using LangChain Markdown loaders
- [ ] **FR-003**: Ingest plain text files (.txt) using LangChain text loaders
- [ ] **FR-004**: Ingest web pages / HTML content using LangChain HTML or URL loaders
- [ ] **FR-004a**: Support both file upload and URL-based ingestion for HTML content (PRD API spec defines both `file` and `url` fields)

### Document Chunking

- [ ] **FR-005**: Chunk documents into 500-800 token segments with 100-token overlap
- [ ] **FR-005a**: Chunk size and overlap must be configurable via environment variables (`CHUNK_SIZE` default 600, `CHUNK_OVERLAP` default 100)
- [ ] **FR-005b**: Chunking must use LangChain `RecursiveCharacterTextSplitter` with token counting (tiktoken) to ensure accurate token counts

### Embedding Generation

- [ ] **FR-006**: Generate embeddings using the Ollama `nomic-embed-text` model (768 dimensions)
- [ ] **FR-006a**: Ollama base URL must be configurable via the `OLLAMA_BASE_URL` environment variable (default: `http://localhost:11434`)
- [ ] **FR-006b**: Embedding model name must be configurable via the `OLLAMA_EMBED_MODEL` environment variable (default: `nomic-embed-text`)

### Storage

- [ ] **FR-007**: Store document chunks and their embeddings in Weaviate
- [ ] **FR-007a**: Weaviate URL must be configurable via the `WEAVIATE_URL` environment variable (default: `http://localhost:8080`)
- [ ] **FR-007b**: Weaviate schema must be created/verified on application startup or first ingestion

### Metadata Preservation

- [ ] **FR-008**: Preserve source metadata with each chunk: filename, location info (page number for PDF, section/heading number for MD/HTML, line range for TXT), and chunk index
- [ ] **FR-008a**: Support optional user-provided metadata (category, tags) as defined in the PRD API spec

### API Endpoint

- [ ] **FR-API-001**: Expose a `POST /ingest` endpoint that accepts file uploads and URL inputs
- [ ] **FR-API-002**: Return a structured JSON response containing: status, document_id, chunks_created, and message
- [ ] **FR-API-003**: Validate input: reject unsupported file types with a clear error message
- [ ] **FR-API-004**: Assign a unique `document_id` to each ingested document
- [ ] **FR-API-005**: Process files ≤ 20MB synchronously, returning the result in the same request
- [ ] **FR-API-006**: Process files > 20MB asynchronously, returning a `job_id` immediately
- [ ] **FR-API-007**: Expose a `GET /ingest/status/{job_id}` endpoint for polling async ingestion progress

### Re-ingestion

- [ ] **FR-009**: Detect previously ingested documents by filename; on re-ingestion, delete all existing chunks for that document from Weaviate before processing the new version

## Non-Functional Requirements/Constraints

- [ ] **NFR-002** (from PRD): Handle files up to 100MB for document ingestion
- [ ] **NFR-006** (from PRD): Structured logging for all pipeline stages (loading, chunking, embedding, storing)
- [ ] **NFR-TECH-001**: Python 3.11+ required
- [ ] **NFR-TECH-002**: FastAPI framework for the API layer
- [ ] **NFR-TECH-003**: LangChain for document loading and chunking orchestration
- [ ] **NFR-TECH-004**: Weaviate as the vector store (no alternatives for v1)
- [ ] **NFR-TECH-005**: Ollama `nomic-embed-text` for embeddings (no external API dependency for embeddings)
- [ ] **NFR-TECH-006**: Docker Compose must provide local development setup for Weaviate and Ollama
- [ ] **NFR-ERR-001**: Ingestion failures for individual documents must not crash the service; return appropriate error responses
- [ ] **NFR-ERR-002**: If Ollama is unreachable, the `/ingest` endpoint must return a meaningful error indicating the embedding service is unavailable
- [ ] **NFR-ERR-003**: If Weaviate is unreachable, the `/ingest` endpoint must return a meaningful error indicating the vector store is unavailable
- [ ] **NFR-PERF-001**: Ingestion throughput must be ≥ 100 documents/minute for average-sized documents (< 5MB)

## Current State

There is no existing ingestion system or source code. The project repository contains:

- A PRD (`docs/PRD.md`) defining the full system architecture across three phases
- An epic directory structure (`epics/e001_document-ingestion-pipeline/`) with a skeleton `overview.md`
- No `src/` directory, no Python code, no configuration files, no Docker Compose setup

The entire ingestion pipeline must be built from scratch, including the project structure, dependency management, FastAPI application, and Docker Compose environment.

## Scope

### In Scope

- LangChain-based document loaders for PDF, Markdown, plain text, and HTML/URL
- Token-aware text chunking with configurable size (500-800 tokens) and overlap (100 tokens)
- Ollama `nomic-embed-text` embedding integration
- Weaviate schema creation and document/chunk storage
- `POST /ingest` API endpoint with file upload and URL support
- `GET /health` endpoint (shared infrastructure, but needed for ingestion service validation)
- Source metadata preservation (filename, location info per format, chunk index)
- Re-ingestion overwrite: detect by filename, delete old chunks, re-process
- Async ingestion for large files (> 20MB) with job status polling (`GET /ingest/status/{job_id}`)
- Environment-variable-based configuration for all external service URLs and chunking parameters
- Docker Compose setup for Weaviate and Ollama local development
- Batch embedding via Ollama for performance
- Project skeleton: `pyproject.toml` or `requirements.txt`, directory structure per PRD Section 7

### Out of Scope

- Retrieval pipeline (`/query` endpoint) -- covered in a separate epic
- LLM integration (Claude) -- Phase 1 retrieval/generation epic
- Hybrid search (BM25 + vector) -- Phase 2
- Cross-encoder reranking (Cohere) -- Phase 2
- Citation enforcement -- Phase 2
- Evaluation pipeline (RAGAS) -- Phase 3
- CI/CD quality gate -- Phase 3
- User authentication and authorization
- Multi-tenant document isolation
- Streaming responses
- Web UI / frontend
- Batch ingestion of multiple documents in a single request (PRD API spec shows single-document ingestion)
- Standalone document deletion API (DELETE endpoint) — re-ingestion overwrite is in scope, but a dedicated delete endpoint is not

## Decision Records

| Question | Decision | Rationale |
|----------|----------|-----------|
| Token counting method | LangChain `RecursiveCharacterTextSplitter` with token counting (uses tiktoken under the hood) | Built-in LangChain support, consistent with the orchestration framework |
| Large file handling | Synchronous with file size limit (~20MB), async with job ID for larger files | Balances MVP simplicity with production requirements for 100MB files |
| Duplicate re-ingestion | Overwrite — delete old chunks, then re-ingest | Most intuitive UX; matched by document filename or ID |
| Partial failure handling | Full rollback — all or nothing | Ensures data consistency; no partially indexed documents in Weaviate |
| Page number for non-PDFs | Section/heading number for MD/HTML, line range for TXT | More informative metadata for downstream citation; worth the parsing complexity |
| Batch embedding | Batch embedding from the start | Ollama supports batch; significant performance gain for documents with 50+ chunks |
| Async large file processing | Add FRs: sync ≤ 20MB, async > 20MB with job_id, status polling endpoint | Captures the async path decided earlier as explicit requirements |
| Re-ingestion overwrite FR | Add FR-009: detect by filename, delete old chunks before re-ingestion | Makes the overwrite contract explicit and implementable |
| Document identity | Filename is the canonical document identity | Simple, intuitive for users; re-ingestion detection based on filename |
| Ingestion throughput | Add NFR-PERF-001: ≥ 100 docs/min for < 5MB docs | Aligns with PRD performance requirement |
| FR-008 metadata granularity | Update to specify: page number (PDF), section/heading (MD/HTML), line range (TXT) | Aligns FR with the earlier metadata decision |
| Out of Scope wording | Reword to: "Standalone document deletion API (DELETE endpoint)" | Clarifies overwrite is in scope, dedicated delete is not |

## Unresolved Items/Risks

| Item | Details | Impact |
|------|---------|--------|
| Weaviate schema design | The PRD does not specify the exact Weaviate class name, property names, or index configuration. These design decisions need to be made during implementation. | Low |
| HTML content extraction quality | FR-004 requires HTML ingestion but web pages vary dramatically in structure. The choice of LangChain HTML loader and content extraction strategy will affect quality. | Medium |
| Ollama model availability | The pipeline depends on `nomic-embed-text` being pulled/available in the local Ollama instance. No guidance on automatic model pulling or startup verification. | Low |

## Priority Assessment

| Requirement | Priority | Rationale |
|------------|----------|-----------|
| FR-001: PDF ingestion | Must | Explicitly required by PRD; PDF is the most common enterprise document format |
| FR-002: Markdown ingestion | Must | Explicitly required by PRD |
| FR-003: Plain text ingestion | Must | Explicitly required by PRD |
| FR-004: HTML/URL ingestion | Must | Explicitly required by PRD |
| FR-005: Token-based chunking (500-800 tokens, 100 overlap) | Must | Explicitly required by PRD; directly impacts retrieval quality |
| FR-005a: Configurable chunk size/overlap | Must | PRD defines environment variables for this; required for Phase 3 optimization |
| FR-005b: Token-aware splitting (LangChain + tiktoken) | Must | Decided: use LangChain RecursiveCharacterTextSplitter with token counting per Decision Records |
| FR-006: Ollama nomic-embed-text embeddings | Must | Explicitly required by PRD; no alternative embedding provider specified |
| FR-007: Weaviate storage | Must | Explicitly required by PRD; Weaviate is the designated vector store |
| FR-007b: Schema auto-creation | Should | Not explicitly in PRD but necessary for developer experience; could be manual initially |
| FR-008: Source metadata preservation | Must | Explicitly required by PRD; critical for downstream citation features |
| FR-008a: Optional user metadata | Could | Defined in PRD API spec but not in the FR list; enhances categorization but not core functionality |
| FR-API-001: POST /ingest endpoint | Must | Explicitly required by PRD; the entry point for all ingestion |
| FR-API-002: Structured response | Must | Defined in PRD API spec; needed for client integration |
| FR-API-003: Input validation | Should | Good practice; not explicitly in PRD FRs but implied by production-grade quality goals |
| FR-API-004: Unique document_id | Should | Defined in PRD response spec; needed for document tracking |
| NFR-002: 100MB file support | Should | PRD requirement but may require async processing design; can start with smaller files |
| NFR-006: Structured logging | Should | PRD requirement; important for production operations but not blocking for MVP |
| Docker Compose setup | Must | Listed as Phase 1 deliverable; required for local development of Weaviate and Ollama |
| GET /health endpoint | Should | Listed in PRD Phase 1 endpoints; useful for validating service dependencies |
| FR-API-005: Sync processing ≤ 20MB | Must | Default path; needed for MVP functionality |
| FR-API-006: Async processing > 20MB | Must | Required to support 100MB files per NFR-002 without timeouts |
| FR-API-007: GET /ingest/status/{job_id} | Must | Required companion to async processing |
| FR-009: Re-ingestion overwrite by filename | Must | Decided: overwrite strategy; prevents stale data |
| NFR-PERF-001: ≥ 100 docs/min throughput | Should | PRD performance requirement; important but can be optimized iteratively |
