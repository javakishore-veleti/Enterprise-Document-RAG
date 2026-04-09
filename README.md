# Enterprise-Document-RAG

A production-shaped retrieval-augmented generation service for regulated,
document-heavy enterprise workflows. Documents flow through GCP Document
AI, land in MongoDB as chunks + embeddings, and a LangChain RAG `/ask`
endpoint answers grounded questions with citations and a response-path
PII / PHI guardrail layer.

This README is a direct walkthrough of `Docs/Design/architecture.drawio`.
Each section maps one-to-one to a tab in that file.

## Business domain

Regulated industries (healthcare payers and providers, clinical research,
financial services, legal) generate large volumes of long-form documents
that downstream staff have to read, cite, and act on: care plans, lab
reports, claim packets, trial protocols, compliance memos, contracts.
Two recurring problems:

1. The information someone needs is scattered across hundreds of pages
   no one has time to read in full.
2. Any answer pulled out of those documents has to be **citable**
   (which page, which document, which version) and **safe** to share
   (no PHI, PII, or out-of-scope fields leaking through to the wrong
   reader).

This service is the answer to those two problems framed as one product:
ingest the documents once, and let staff ask grounded, cited questions
over them with redaction enforced on the egress path.

## Business features

| Feature | What the user gets |
|---|---|
| Document upload | A `POST /ingest` endpoint that accepts a file + metadata, runs it through Document AI, chunks the result, embeds the chunks, and stores everything with an audit trail. |
| Grounded Q&A | A `POST /ask` endpoint that returns an answer **plus** the chunk IDs it cited, so the reader can click through to the source. |
| Refusal on no-context | If retrieval finds nothing relevant, the service refuses to answer instead of hallucinating. |
| ACL-aware retrieval | Chunks carry `acl_tags`; users only see citations they're allowed to see. |
| PII / PHI redaction | Response-path scrubber strips identifiers (names, MRNs, emails, phone numbers, etc.) before the answer leaves the service. |
| Immutable answer log | Every Q&A is appended to `answer_logs` with the cited chunks, model, and guardrails applied — auditable after the fact. |

## Architecture (mapped to `Docs/Design/architecture.drawio`)

### Tab 1 — System Overview

The five moving parts of the service and how a single document moves
from a source system to a queryable knowledge base:

- **Source documents** → **GCP Document AI** (OCR / form parsing /
  field extraction)
- → **Ingestion service** (FastAPI) chunks, embeds, stores
- → **MongoDB** holds `documents`, `chunks`, `embeddings`, `answer_logs`
- A separate **Query service** (FastAPI) runs the LangChain RAG chain
  against the same MongoDB
- **LLM provider** is called with a grounded prompt; the response goes
  through the **PII / PHI guardrails** before reaching the user

### Tab 2 — Ingestion Pipeline

The ingestion path in detail: `POST /ingest` → Document AI processor →
chunker (by section, with token budget and overlap) → embedder →
MongoDB write. An **audit log** entry is written from the Document AI
step so every uploaded document has a `(document_id, processor,
checksum, uploader, ts)` row before any analytical processing happens.

### Tab 3 — `/ask` Query Flow

The full query path: question embed → top-k retrieval against MongoDB
(`$vectorSearch` or aggregation depending on deployment) → rerank /
dedupe / ACL filter → grounded prompt build (system rules + cited
chunks + question) → LLM call via LangChain → **response guardrails** →
`answer_logs` append → 200 OK with `{answer, citations, log_id}`.

### Tab 4 — Data Model

Four MongoDB collections and how they reference each other:

- `documents` — one row per upload (tenant_id, source_uri, processor,
  mime_type, checksum, uploaded_at, uploaded_by)
- `chunks` — `document_id`, ordinal, text, page/section, token_count,
  `acl_tags[]`
- `embeddings` — `chunk_id`, model, dim, vector
- `answer_logs` — immutable Q&A records that cite `chunk_ids[]`

### Tab 5 — PII / PHI Guardrails

Decision diagram for the response path:

1. **Citation check** — does every claim cite a `chunk_id`? No →
   refuse and log "no grounded answer".
2. **PII / PHI scrubber** — regex + dictionary + entity NER over the
   response text.
3. **ACL enforcement** — drop any chunk the caller lacks `acl_tags`
   for, even if retrieval surfaced it.
4. Final response carries the answer + cited chunks + applied
   guardrail tags. `answer_logs` gets an immutable append.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| HTTP | FastAPI | async, Pydantic-typed request/response, OpenAPI for free |
| OCR / form extraction | GCP Document AI | covers general OCR, forms processor, and custom processors per document type |
| Storage | MongoDB | flexible document shape per processor; native `$vectorSearch` for retrieval; aggregation pipelines for ACL + rerank |
| RAG orchestration | LangChain | retriever / prompt / LLM chaining; pluggable embedders and chat models |
| Embeddings | text-embedding model (provider-agnostic via LangChain) | swap to Vertex AI / OpenAI / sentence-transformers per deployment |
| Settings | pydantic-settings | typed env-driven config, single source of truth |
| Logging | structlog | JSON output, PHI-aware processor (see tab 5) |
| Tests | Pytest + MagicMock | offline-first; mock Document AI and LLM at the boundary |

## Why this exists

This is a portfolio bridge repo authored to map a previous engagement's
problem (large-volume document workflows in a regulated environment)
into the JD stack of a current opportunity (FastAPI, PyMongo aggregation,
GCP Document AI, RAG, LangChain, prompt grounding, Pytest + MagicMock).
The architecture diagram is the authoritative source — this README walks
through it for readers who want the prose version.

## Status

Scaffolded. The architecture diagram is committed; module implementation
is the next iteration. See `Docs/Design/architecture.drawio` for the
target architecture.
