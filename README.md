# AGENT 1
# Knowledge Ingestion Agent (RAG — Intake & Processing)

An n8n-based document ingestion pipeline that takes a raw file (PDF/TXT/DOCX) via webhook, deduplicates it, extracts and chunks its text with section-awareness, embeds each chunk with Google Gemini, and stores the result in Qdrant for downstream retrieval. Built as the ingestion half of a two-agent RAG system (paired with the Knowledge Retrieval Agent).

## Why this exists

Most "RAG demo" projects hardcode a single PDF loader and call it done. This pipeline is built to survive real input: duplicate uploads, unsupported file types, partial failures mid-processing, and the need to track every document's status end-to-end — not just "did the embedding call succeed."

## Architecture

```
Webhook (file upload)
   │
   ▼
[Workflow 1: Knowledge Intake]
   Validate request → Generate document metadata (UUID, hash placeholder)
   → Write file to disk → Create Airtable tracking record
   → Call Workflow 2 (sub-workflow execution)
   → On failure: delete the Airtable record (rollback, no orphaned entries)
   │
   ▼
[Workflow 2: Document Processing]
   Hash file (Crypto) → Search Airtable for existing hash (duplicate check)
   → If duplicate: update status, stop
   → If new: route by file extension (Switch)
   → Extract text (PDF / TXT parser)
   → Normalize text + calculate statistics
   → Enterprise Chunking (section/heading-aware, see below)
   → Generate embeddings (Gemini gemini-embedding-001)
   → Build Qdrant payload → Store in Qdrant
   → Update Airtable status at every stage (queued → extracting → chunking → embedding → complete/failed)
```

Both workflows report status back to Airtable at each stage, so a document's processing state is always inspectable without reading logs.

## Key engineering decisions

**Deduplication by content hash, not filename.** Every uploaded file is hashed (Crypto node) and checked against existing records before processing. This means the same document uploaded twice — even under a different filename — is caught and skipped rather than creating duplicate vector entries.

**Section-aware chunking, not fixed-size splitting.** The chunker (`CHUNK_SIZE: 1000`, `OVERLAP: 200` characters, `MIN_CHUNK_SIZE: 100`) detects headings via four independent signals — Markdown headers, numbered sections, ALL-CAPS lines, and bullet structure — and keeps chunk boundaries aligned to document structure where possible, rather than cutting mid-sentence at a fixed character count. Each chunk carries `section_title` and `section_level` metadata forward into the vector store, which the retrieval side later uses for citation and context assembly.

**Explicit rollback on failure.** If document processing fails after the Airtable tracking record is created, the Intake workflow deletes that record rather than leaving an orphaned "queued" entry that never resolves. This is a deliberate compensating-transaction pattern, not just a try/catch.

**Retry + graceful degradation on external calls.** The Gemini embedding call and Qdrant write both use `retryOnFail` with `maxTries: 2` and `continueErrorOutput`, so a transient API blip doesn't silently drop a document — it fails into an explicit error path that updates Airtable status instead.

## Tech stack
- **Orchestration:** n8n (custom JS code nodes for business logic)
- **Extraction:** n8n native PDF/TXT parsers
- **Embeddings:** Google Gemini (`gemini-embedding-001`)
- **Vector store:** Qdrant (self-hosted)
- **State tracking:** Airtable
- **Auth:** n8n credential store (no inline API keys)

## Known limitations
- DOCX extraction is routed for but not yet implemented as a working extractor node — currently only PDF and TXT are handled end-to-end.
- Chunking config (`CHUNK_SIZE`, `OVERLAP`) is a fixed constant, not tuned per document type — a 2-page policy doc and a 200-page manual get the same chunk size today.
- No automatic reprocessing/backfill path if the chunking or embedding logic changes — existing vectors would need a manual re-ingestion run.

## Setup requirements
- n8n instance with `n8n-nodes-base` and file extraction nodes enabled
- Airtable base with a documents table (see `Generate Document Metadata` node for expected schema)
- Google Gemini API credential (stored in n8n credential manager — do not hardcode)
- Running Qdrant instance with a `knowledge_base` collection

## Combined Workflows Picture
<img width="1680" height="1335" alt="RAG Knowledge Ingestion Agent All Workflows" src="https://github.com/user-attachments/assets/c93f3c1a-b8b3-4288-8371-b4e59e6c5965" />

# AGENT 2
# Knowledge Retrieval Agent (RAG Runtime)

An n8n-based retrieval-augmented generation runtime that takes a user query, retrieves and ranks relevant chunks from Qdrant, builds a token-budgeted context, and generates a citation-validated answer using Gemini. Built as the query-time half of a two-agent RAG system (paired with the Knowledge Ingestion Agent).

## Why this exists

Naive RAG is "embed the query, top-k search, stuff into a prompt." This runtime instead treats retrieval as a pipeline with explicit quality gates: similarity thresholds that can legitimately return "no answer," diversity enforcement so one document can't dominate the context, complexity-aware token budgeting, and post-hoc citation validation to catch hallucinated sources before they reach the user.

## Architecture

```
Webhook (query)
   │
   ▼
[Workflow 1: Query Processing]
   Validate request → Normalize query → Generate request metadata
   → Embed query (Gemini gemini-embedding-001)
   → Qdrant vector search
   → Similarity Threshold Filter (MIN_SCORE = 0.50)
      → status: SUCCESS | NO_MATCH | EMPTY_COLLECTION
   → Call Workflow 2 (sub-workflow execution)
   │
   ▼
[Workflow 2: Context Optimization Engine]
   Retrieval Validation & Metadata Filtering
   → Composite Ranking & Diversity Engine (weighted scoring + per-document cap)
   → Query Complexity Classification (SIMPLE / MEDIUM / COMPLEX) → chunk & token budget
   → Dedup, Merge, Compress, Budget Enforcement & Ordering
   → Prompt Builder
   → Basic LLM Chain (Gemini, with model fallback — see below) + Structured Output Parser
   → Citation Validation & Confidence Scoring (post-processing)
   → Response Formatting
   → Logging & Analytics
```

## Key engineering decisions

**Similarity threshold as a real decision point, not a formality.** `MIN_SCORE = 0.50` filters Qdrant results before they're treated as usable evidence. If nothing clears the bar, the pipeline returns an explicit `NO_MATCH` status rather than forcing the LLM to generate an answer from weak or irrelevant context — this is the difference between a system that admits uncertainty and one that hallucinates confidently.

**Composite ranking is honest about what data actually exists.** The ranking engine is documented in-code as weighing only `similarity` (0.70 weight) plus signals genuinely present in the schema — fields like `document_authority_score` and `business_priority_tier` that the architecture calls for are explicitly disabled (weight = 0) with a comment explaining why, rather than fed placeholder or fabricated values. This is a deliberate "don't fake it" engineering standard applied consistently across the codebase, and it's the kind of design discipline worth pointing to directly in an interview.

**Query complexity classification drives token budget, not a fixed context window.** A rule-based heuristic (comparison markers, multi-part conjunctions, query length) classifies each query as SIMPLE/MEDIUM/COMPLEX and allocates chunk/token budget accordingly — a one-line factual question and a multi-part comparison question don't get the same amount of context stuffed into the prompt. The code explicitly notes this is a heuristic, not true intent classification, and flags it as a known limitation rather than overselling it.

**Model fallback, not redundant dead wiring.** Two Gemini models (`gemini-3-flash-preview` and `gemini-2.5-flash`) are both connected to the same `ai_languageModel` input on the LLM Chain node — n8n's built-in fallback mechanism, where the second model is used automatically if the first fails. This is a deliberate reliability choice for handling preview-model instability, not leftover experimentation.

**Citation validation without a second LLM call.** After generation, every citation in the structured output is checked against the actual `source_map` of retrieved chunks to catch invented/hallucinated sources. Confidence is computed from retrieval-time signals (top chunk score, tier composition) already in hand — not from a second LLM self-report or a second verification call — because self-reported LLM confidence is unreliable and a second call would double latency and cost for marginal benefit. This is a cost/accuracy tradeoff made explicitly, not by default.

## Tech stack
- **Orchestration:** n8n (custom JS code nodes + LangChain nodes)
- **Embeddings:** Google Gemini (`gemini-embedding-001`)
- **Vector store:** Qdrant (self-hosted)
- **Generation:** Gemini (`gemini-3-flash-preview` primary, `gemini-2.5-flash` fallback), via n8n's Basic LLM Chain + Structured Output Parser
- **Auth:** n8n credential store (no inline API keys)

## Known limitations
- Query complexity classification is heuristic-based (keyword/structure matching), not a trained classifier or LLM-based intent detection — it will misclassify some edge cases by design tradeoff, documented in-code.
- Ranking weights for document authority and business priority are present in the architecture but disabled because the upstream schema (Ingestion Agent) doesn't yet populate those fields — closing this gap requires adding `document_authority_score` and `business_priority_tier` to document metadata at ingestion time.
- Structured-output field paths for the LLM Chain node are version-sensitive (noted directly in the Citation Validation node) and should be re-verified against the execution panel after any n8n or LangChain node version upgrade.

## Setup requirements
- n8n instance with `@n8n/n8n-nodes-langchain` package installed
- Google Gemini API credential (stored in n8n credential manager)
- Running Qdrant instance with a `knowledge_base` collection populated by the Knowledge Ingestion Agent
## Combined Workflows Picture
<img width="1692" height="1239" alt="Knowledge Retrieval Agent (RAG Runtime) All Workflows" src="https://github.com/user-attachments/assets/71548eff-71e6-4bdb-8620-5b0cf03376f1" />


