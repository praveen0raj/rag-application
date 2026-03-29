# E-001: Document Ingestion Pipeline — Q&A

---

## Question 1: Token Counting Method for Chunking

FR-005 specifies 500–800 "token" segments. Which tokenizer should we use to measure token boundaries?

1. **tiktoken (cl100k_base)** (Recommendation: ⭐︎| Industry standard, matches OpenAI/Claude token counting closely, deterministic)
2. **Character-based approximation** (~4 chars per token) (Recommendation: ⚪︎| Simpler, faster, but less accurate at boundaries)
3. **LangChain RecursiveCharacterTextSplitter with token counting** (Recommendation: ⚪︎| Built-in LangChain support, uses tiktoken under the hood)
4. Skip / Consider later

Answer: 3

Reason (optional - fill in if you want to record the decision rationale or want AI to consider your intent):

---

## Question 2: Large File Handling Strategy

Documents up to 100MB may be uploaded. Large PDFs could take minutes to process. How should the API handle this?

1. **Synchronous processing** (Recommendation: ⭐︎| Simpler for MVP; set a generous request timeout; iterate later)
2. **Asynchronous with job ID** (Recommendation: ⚪︎| Better UX for large files, but adds complexity — background task queue, status polling endpoint)
3. **Synchronous with a file size limit (e.g. 20MB), async for larger** (Recommendation: △| Hybrid approach adds two code paths)
4. Skip / Consider later

Answer: 3

Reason (optional):

---

## Question 3: Duplicate / Re-ingestion Handling

If the same document is uploaded again, what should happen?

1. **Reject with error** (Recommendation: ⚪︎| Safest, prevents accidental duplicates; user must explicitly delete first)
2. **Overwrite — delete old chunks, re-ingest** (Recommendation: ⭐︎| Most intuitive for users updating documents; matched by document filename or ID)
3. **Allow duplicates** (Recommendation: △| Simplest to implement but pollutes the vector store over time)
4. Skip / Consider later

Answer: 2

Reason (optional):

---

## Question 4: Error Handling for Partial Failures

If embedding fails for one chunk mid-document (e.g., Ollama timeout), should we roll back all stored chunks for that document?

1. **Full rollback — all or nothing** (Recommendation: ⭐︎| Ensures data consistency; no partially indexed documents)
2. **Partial ingestion — store what succeeded, report failures** (Recommendation: ⚪︎| Faster for large documents, but leaves incomplete data in Weaviate)
3. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 5: Page Number Metadata for Non-PDF Formats

FR-008 requires storing page numbers, but Markdown, TXT, and HTML don't have pages. What should we store?

1. **null / None** (Recommendation: ⭐︎| Honest — no page concept exists; downstream consumers handle it)
2. **Always 1** (Recommendation: △| Misleading for multi-section documents)
3. **Section/heading number for MD/HTML, line range for TXT** (Recommendation: ⚪︎| More informative but adds parsing complexity)
4. Skip / Consider later

Answer: 3

Reason (optional):

---

## Question 6: Batch Embedding Performance

Embedding chunks one-by-one via Ollama can be slow for large documents. Should we implement batch embedding?

1. **Batch embedding from the start** (Recommendation: ⭐︎| Ollama supports batch; significant performance gain for documents with 50+ chunks)
2. **Single-chunk embedding for MVP, batch later** (Recommendation: ⚪︎| Simpler initial implementation; optimize when performance is measured)
3. Skip / Consider later

Answer: 1

Reason (optional):

---

# Review Findings (Additional Questions)

## Question 7: Missing FRs for Async Large File Processing

The Decision Records state "sync with file size limit (~20MB), async with job ID for larger files," but no functional requirements capture the async path. This means the async behavior has no spec to implement against.

1. **Add FRs now** (Recommendation: ⭐︎| Add FR-API-005: sync processing for files ≤20MB; FR-API-006: async processing with job ID for files >20MB; FR-API-007: GET /ingest/status/{job_id} endpoint for polling)
2. **Defer async to a separate PBI** (Recommendation: ⚪︎| Keep MVP sync-only with a 20MB limit; add async as a follow-up PBI within this EPIC)
3. **Remove async from Decision Records, keep sync-only** (Recommendation: △| Simplest but contradicts the earlier decision and limits 100MB support)
4. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 8: Missing FRs for Re-ingestion Overwrite Behavior

UC-5 describes overwrite (detect existing → delete old chunks → re-ingest), but no functional requirement covers this. Without an FR, the detection mechanism and deletion behavior are unspecified.

1. **Add FR-009: Detect duplicate documents by filename and delete existing chunks before re-ingestion** (Recommendation: ⭐︎| Matches UC-5 and Decision Records; makes the overwrite contract explicit)
2. **Add FR-009 with document_id-based detection instead of filename** (Recommendation: ⚪︎| More robust if filenames change, but requires the caller to track document_id)
3. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 9: Document Identity Mechanism

The requirements use "filename," "document_id," and "filename/identifier" interchangeably when discussing duplicate detection and re-ingestion. Which is the canonical document identity?

1. **Filename is the identity** (Recommendation: ⚪︎| Simple, intuitive for users; risk of collision if different folders have same-named files)
2. **System-generated document_id (UUID) is the identity; filename is metadata** (Recommendation: ⭐︎| No collisions; re-ingestion requires passing the original document_id)
3. **Composite key: filename + optional user-provided namespace** (Recommendation: ⚪︎| Handles same-named files from different sources; more complex API)
4. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 10: Missing Ingestion Throughput NFR

The PRD specifies "Document ingestion: ≥ 100 docs/min" as a performance requirement, but this is not captured in the NFR section of the requirements analysis.

1. **Add NFR-PERF-001: Ingestion throughput ≥ 100 documents/minute for average-sized documents (< 5MB)** (Recommendation: ⭐︎| Aligns with PRD; provides a measurable target)
2. **Note it as aspirational, not blocking for MVP** (Recommendation: ⚪︎| Avoids over-engineering the first implementation)
3. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 11: FR-008 Metadata Wording vs Decision

FR-008 says "page number (where applicable)" but the Decision Records specify richer metadata: "section/heading number for MD/HTML, line range for TXT." Should FR-008 be updated to reflect the decided granularity?

1. **Update FR-008 to specify: page number for PDF, section/heading for MD/HTML, line range for TXT** (Recommendation: ⭐︎| Aligns the FR with the decision; makes expectations clear for implementation)
2. **Keep FR-008 generic, add detail in acceptance criteria later** (Recommendation: ⚪︎| Simpler FR, detail deferred)
3. Skip / Consider later

Answer: 1

Reason (optional):

---

## Question 12: Out of Scope Wording Contradiction

"Document deletion or update-in-place (except as part of overwrite on re-ingestion)" in Out of Scope is confusing — the overwrite IS a deletion followed by re-ingestion, which is IN scope per UC-5 and Decision Records.

1. **Reword to: "Standalone document deletion API (DELETE endpoint) — not part of this EPIC"** (Recommendation: ⭐︎| Clarifies that re-ingestion overwrite is in scope, but a dedicated delete API is not)
2. **Remove the line entirely** (Recommendation: ⚪︎| Avoids confusion, but loses the explicit exclusion of a delete endpoint)
3. Skip / Consider later

Answer: 1

Reason (optional):

---
