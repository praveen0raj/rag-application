# E-001: Document Ingestion Pipeline

**Status:** In Progress
**Phase:** Phase 1 — Fundamental RAG (MVP)
**Priority:** P0
**Total Estimate:** 44 story points

## Summary

Ingest domain-specific documents (PDF, Markdown, plain text, HTML), chunk them into token-aware segments, embed them using Ollama, and store them in Weaviate for retrieval. Includes the POST /ingest API endpoint, re-ingestion overwrite, async processing for large files, and structured logging.

## PBI List

| PBI | Name | Est. | Priority | Dependencies | Status |
|-----|------|------|----------|-------------|--------|
| e001-001 | project-scaffold-and-infra | 5 | P0 | None | Not Started |
| e001-002 | weaviate-schema-and-storage | 5 | P0 | e001-001 | Not Started |
| e001-003 | chunking-engine | 3 | P0 | e001-001 | Not Started |
| e001-004 | embedding-service | 3 | P0 | e001-001 | Not Started |
| e001-005 | document-loaders | 5 | P0 | e001-001 | Not Started |
| e001-006 | ingest-api-endpoint | 5 | P0 | e001-002, e001-003, e001-004, e001-005 | Not Started |
| e001-007 | reingestion-overwrite | 3 | P0 | e001-006 | Not Started |
| e001-008 | async-large-file-ingestion | 8 | P1 | e001-006 | Not Started |
| e001-009 | structured-logging | 2 | P1 | e001-006 | Not Started |
| e001-010 | performance-optimization | 5 | P2 | e001-006, e001-009 | Not Started |

## Dependency Graph

```
e001-001 (scaffold)
  ├── e001-002 (weaviate storage)
  ├── e001-003 (chunking)
  ├── e001-004 (embedding)
  └── e001-005 (document loaders)
          │
          ▼
      e001-006 (ingest API) ← depends on 002, 003, 004, 005
          ├── e001-007 (re-ingestion)
          ├── e001-008 (async large files) [P1]
          └── e001-009 (structured logging) [P1]
                  │
                  ▼
              e001-010 (performance) [P2] ← depends on 006, 009
```

## Sprint Composition

| Sprint | PBIs | Points | Focus |
|--------|------|--------|-------|
| Sprint 1 | e001-001, e001-003, e001-004 | 11 | Scaffold + independent pipeline modules |
| Sprint 2 | e001-002, e001-005 | 10 | Storage layer + document loaders |
| Sprint 3 | e001-006, e001-007 | 8 | Full ingest endpoint + re-ingestion |
| Sprint 4 | e001-008, e001-009 | 10 | Async processing + logging |
| Sprint 5 | e001-010 | 5 | Performance optimization |
