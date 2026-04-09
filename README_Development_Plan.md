# Development Plan — Enterprise-Document-RAG

A focused punch list to take this repo from a scaffold to the
architecture in `Docs/Design/architecture.drawio`. Each block builds
out one tab of that diagram so the diagram and the code stay in sync.

Legend: `[ ]` not started · `[~]` in progress · `[x]` done

## Block 1 — Project skeleton

- [ ] **T1.1** `pyproject.toml` with FastAPI, pydantic-settings,
       pymongo, langchain, structlog, pytest, pytest-asyncio,
       httpx, mongomock as the dev extras.
- [ ] **T1.2** `src/enterprise_document_rag/` package layout —
       `ingestion/`, `query/`, `storage/`, `guardrails/`, `config.py`,
       `logging.py`, `main.py`.
- [ ] **T1.3** `tests/` mirror the same tree.
- [ ] **T1.4** `pytest -q` green on an empty `test_smoke.py`.

## Block 2 — Storage (Tab 4 in the diagram)

- [ ] **T2.1** `storage/schemas.py` — Pydantic v2 models for
       `Document`, `Chunk`, `Embedding`, `AnswerLog`. PHI-tagged
       fields use `Field(json_schema_extra={"phi": True})`.
- [ ] **T2.2** `storage/indexes.py` — declarative index list:
       `(tenant_id, document_id)`, `(chunk_id)` unique,
       `acl_tags` multi-key, `(tenant_id, ts)` for `answer_logs`.
- [ ] **T2.3** `storage/mongo.py` — `MongoStore` wrapping
       `pymongo.MongoClient`. Tenant filter enforced inside the
       class. `mongomock`-friendly constructor for tests.
- [ ] **T2.4** `tests/storage/test_mongo.py` — round-trip tests
       against `mongomock`.

## Block 3 — Ingestion pipeline (Tab 2)

- [ ] **T3.1** `ingestion/docai.py` — thin wrapper around the
       Document AI SDK with a `FakeDocAIClient` for tests.
- [ ] **T3.2** `ingestion/chunker.py` — section-aware chunker with
       token budget + overlap. Pure function, fully tested.
- [ ] **T3.3** `ingestion/embedder.py` — `Embedder` Protocol +
       `FakeEmbedder` for tests. Pluggable real implementation.
- [ ] **T3.4** `ingestion/pipeline.py` — composes Doc AI →
       chunker → embedder → `MongoStore.save_document` +
       `save_chunks` + `save_embeddings` + audit row.
- [ ] **T3.5** `tests/ingestion/test_pipeline.py` — happy path
       end-to-end against the fakes.

## Block 4 — Query / `/ask` flow (Tab 3)

- [ ] **T4.1** `query/retriever.py` — `Retriever` that calls Mongo
       `$vectorSearch` (or aggregation fallback for `mongomock`)
       and applies the ACL filter inside the query.
- [ ] **T4.2** `query/rerank.py` — score-based dedupe + rerank.
- [ ] **T4.3** `query/prompt.py` — grounded prompt builder
       (system rules + cited chunks + question).
- [ ] **T4.4** `query/chain.py` — LangChain RAG chain that calls
       the retriever, rerank, prompt builder, LLM, then hands the
       response to the guardrail layer.
- [ ] **T4.5** `tests/query/test_chain.py` — fake LLM, asserts
       citations land and the answer references the cited chunks.

## Block 5 — PII / PHI guardrails (Tab 5)

- [ ] **T5.1** `guardrails/citation.py` — refuses any answer that
       does not cite a `chunk_id`.
- [ ] **T5.2** `guardrails/scrubber.py` — regex + dictionary +
       entity NER PII / PHI scrubber.
- [ ] **T5.3** `guardrails/acl.py` — drop chunks the caller lacks
       `acl_tags` for.
- [ ] **T5.4** `tests/guardrails/test_*.py` — happy path, refusal
       path, scrub path, ACL drop.

## Block 6 — FastAPI surface (Tab 1)

- [ ] **T6.1** `payer_api/app.py` — `create_app()` factory;
       mounts `/healthz`, `/version`, `/ingest`, `/ask`.
- [ ] **T6.2** `tests/api/test_app.py` — `httpx.AsyncClient` smoke
       tests for `/healthz` and one round-trip ingest + ask.

## Block 7 — Observability + CI

- [ ] **T7.1** `logging.py` with a PHI-aware structlog processor
       (mirrors the one in Member-Event-Stream-Agent).
- [ ] **T7.2** `.github/workflows/ci.yml` — pytest on every push
       and PR to `main` against Python 3.13 with the dev extras.
- [ ] **T7.3** `pytest -q` green locally and in CI.

## Definition of done

1. `pytest -q` green locally and in GitHub Actions.
2. `uvicorn enterprise_document_rag.main:app` starts cleanly with
   the in-memory backend.
3. `POST /ingest` against a sample PDF and `POST /ask` against the
   sample question both 200, with citations on the response.
4. Every block lands as its own commit.

## Status

Scaffold + architecture diagram + README walkthrough committed.
Block 1 is the next iteration.
