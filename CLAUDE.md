# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

OpsPulse is a multi-tenant engineering runbook and incident triage engine: a production-grade RAG system over Markdown docs, post-mortem PDFs and JSON/CSV incident tickets, with RBAC, strict citations and automated evaluation. The repo is greenfield. `ASSIGNMENT.pdf` is the spec and the source of truth; re-read it when in doubt.

## Hard rules (non-negotiable)

1. **No black-box chains.** Do not use `RetrievalQA.from_chain_type()`, `VectorStoreIndex.from_documents()` or similar LangChain/LlamaIndex abstractions. Ingestion, chunking, SQL/vector queries, reranking and prompt construction are written explicitly with raw SDKs, so every SQL query and prompt payload can be inspected.
2. **Permissions live in the query, never in the prompt.** Every document has an `access_role` of `public`, `engineer` or `admin`. Filter on it in the SQL `WHERE` clause of *both* the vector and the keyword search, before ranking.
3. **No fixed-character chunking.** Split by document structure: headings and paragraphs. Code blocks and tables stay intact.
4. **Structured output with verifiable citations.** The LLM returns a Pydantic-validated schema: `answer`, `citations` (a list of `{chunk_id, source_file}`) and `confidence_score`. Every factual claim cites a retrieved `chunk_id`, and a code-side validator rejects any cited `chunk_id` that was not among the chunks sent in that request.
5. **Deterministic refusal.** If the top reranker score is below the threshold, or the context does not contain the answer, return exactly `"Insufficient evidence in the knowledge base"`. Do not let the LLM guess.
6. **No pipeline tweaks without a benchmark run.** Once the baseline exists, any change to chunk sizes, prompts, embedding model, thresholds or retrieval logic requires running the eval CLI and logging before/after metrics.

## Stack

- Python 3.12, `uv`, `ruff`, `mypy`, `pytest`
- FastAPI (async); ingest endpoints enqueue a job and return immediately
- PostgreSQL 16 + pgvector via Docker Compose, accessed with `psycopg` 3 and **raw SQL (no ORM)**
- Procrastinate (Postgres-backed job queue) for async ingestion
- Docling for converting PDFs to Markdown; custom converters for JSON/CSV tickets
- `tiktoken` for token counting in the chunker
- Anthropic SDK: Haiku for contextual enrichment (with prompt caching) and query rewriting, Sonnet for answer generation
- OpenAI `text-embedding-3-small` for embeddings (Anthropic has no embeddings API)
- `BAAI/bge-reranker-v2-m3` cross-encoder, run locally via sentence-transformers
- Pydantic + Instructor (or native tool-use JSON schema) for structured outputs with retries
- RAGAS-style metrics in a custom eval CLI; Langfuse (self-hosted) plus a structured JSON trace log for observability

Known caveat: Postgres `ts_rank_cd` full-text ranking is not true BM25. Document this trade-off, or switch to ParadeDB `pg_search` if the benchmark shows it matters.

## Architecture

### Ingestion pipeline (async)
upload → SHA-256 of the content → **no-op if the hash is unchanged** → enqueue → Docling/converter to Markdown → heading-aware chunker → contextual enrichment → embed + build tsvector → **atomically replace** the document's old chunks in one transaction.

- Chunker: splits on `#`/`##`/`###`; chunks are **250–600 tokens** with **10–15% token overlap** between sequential chunks in the same section; code blocks are never split.
- Chunk metadata (JSONB): heading hierarchy (H1 → H2 → H3), `source_file`, `updated_at`, `access_role`.
- Contextual Retrieval: send the whole document (prompt-cached) plus the chunk to a cheap LLM, which writes a 1–2 sentence context header. That header is prepended to the chunk before embedding *and* before keyword indexing.
- Chunk table columns: raw text, contextualized text, JSONB metadata (GIN index), vector (HNSW index), tsvector (GIN index).

### Query pipeline
chat history + message → **query rewriter** (standalone, search-optimized query) → **vector top-25** and **full-text top-25**, both RBAC-filtered in SQL → **RRF** merge with `score(d) = Σ 1/(60 + rank_m(d))` → **cross-encoder rerank** of the top-25 → keep the **top-5 above the threshold**, otherwise refuse → prompt with chunks wrapped in `<document id="chunk_42" source="runbook.md">…</document>` tags (untrusted data kept separate from instructions) → structured output → citation validator → trace.

- Every query logs a trace with: original query, rewritten query, retrieved chunk IDs + RRF scores, reranker scores, prompt/completion tokens, cost in USD, and per-stage latency (retrieval ms, rerank ms, LLM TTFT).

## Build order and evaluation

1. **Eval harness first.** Build a corpus of 20–30 technical docs (Kubernetes/Postgres runbooks, synthetic post-mortems with error codes such as `ERR-4523` and `INC-8842`, CLI flags and tables). Write a 30-case golden dataset JSON with `question`, `expected_answer`, `expected_source_docs` and `user_role`. Include exact-code queries (e.g. "OOMKilled exit code 137"), multi-hop questions across two files, unanswerable questions (to test refusal) and RBAC cases.
2. The eval CLI scores **Context Recall**, **Context Precision** and **Faithfulness** with an LLM-as-judge.
3. Then build, in order: async ingestion + chunking → hybrid search + RBAC → query rewriting + reranking → constrained generation + observability.

The README must include an architecture diagram of both pipelines, a **4-row ablation table**, and **p50/p95 latency + average cost per 1,000 queries**. The ablation rows are:
1. naive baseline (fixed chunks, vector only)
2. + structure-aware/contextual chunking
3. + hybrid RRF
4. + reranking

So the naive baseline must stay runnable as a configurable pipeline variant.

## Commands

No code scaffold exists yet. Add build, run, test, single-test and eval commands here once `pyproject.toml` and `docker-compose.yml` exist.
