# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

OpsPulse is a multi-tenant engineering runbook and incident triage engine: a production-grade RAG system over Markdown docs, post-mortem PDFs and JSON/CSV incident tickets, with RBAC, strict citations and automated evaluation. The repo is greenfield. `ASSIGNMENT.pdf` is the spec and the source of truth; re-read it when in doubt.

## Hard rules (non-negotiable)

1. **No black-box chains.** Do not use `RetrievalQA.from_chain_type()`, `VectorStoreIndex.from_documents()` or similar LangChain/LlamaIndex abstractions. Ingestion, chunking, SQL/vector queries, reranking and prompt construction are written explicitly with raw SDKs, so every SQL query and prompt payload can be inspected.
2. **Permissions live in the query, never in the prompt.** Every document has an `access_role` of `public`, `engineer` or `admin`. Filter on it in the SQL `WHERE` clause of *both* the vector and the keyword search, before ranking.
3. **No fixed-character chunking.** Split by document structure: headings and paragraphs. Code blocks and tables stay intact. The only exception is the naive-baseline ablation variant (see below), which exists solely to be benchmarked against.
4. **Structured output with verifiable citations.** The LLM returns a Pydantic-validated schema: `answer`, `citations` (a list of `{chunk_id, source_file}`) and `confidence_score`. Every factual claim cites a retrieved `chunk_id`, and a code-side validator rejects any cited `chunk_id` that was not among the chunks sent in that request.
5. **Deterministic refusal.** If the top reranker score is below the threshold, or the context does not contain the answer, return exactly `"Insufficient evidence in the knowledge base"`. Do not let the LLM guess.
6. **No pipeline tweaks without a benchmark run.** Once the baseline exists, any change to chunk sizes, prompts or embedding model requires running the eval CLI and logging before/after metrics. (Project extension: the same applies to thresholds and retrieval logic.)

## Stack

The spec leaves most tooling open (e.g. pgvector *or* Qdrant, Cohere Rerank *or* a local cross-encoder, Langfuse *or* Arize Phoenix, RAGAS *or* Promptfoo). The choices below are this project's picks, not spec requirements.

- Python 3.12, `uv`, `ruff`, `mypy`, `pytest`
- FastAPI (async); ingest endpoints enqueue a job and return immediately
- PostgreSQL 16 + pgvector via Docker Compose, accessed with `psycopg` 3 and **raw SQL (no ORM)**
- Procrastinate (Postgres-backed job queue) for async ingestion
- Docling for converting PDFs to Markdown; custom converters for JSON/CSV tickets
- `tiktoken` for token counting in the chunker
- **Fully local models by default, no API keys.** Ollama runs natively on macOS (not in Docker, so it gets the Metal GPU) and is called through the official `ollama` Python SDK. No LangChain/LlamaIndex wrappers.
- One local instruct model handles answer generation, query rewriting, contextual enrichment and the eval judge. The default is `gemma4:latest` (~9.6 GB; the official build, not community fine-tunes). It is configurable via `LLM_MODEL`; `LLM_MODEL_CI` sets a smaller model for CI.
- Embeddings: `bge-m3` via Ollama (1024-dim, so the column is `vector(1024)`), configurable via `EMBED_MODEL`. Changing the embedding model means re-ingesting and an eval run (rule 6). Why `bge-m3`: it is one of the two most-used open-weight embedding models (with Qwen3-Embedding), its 8K-token context fits 600-token chunks plus the context header, and it is small enough to share 16 GB of RAM with the LLM. Rejected: `mxbai-embed-large` (512-token limit truncates chunks). Candidates to benchmark later: `qwen3-embedding:0.6b` (can output 1024 dimensions, so no schema change; scores higher on benchmarks) and `embeddinggemma` (smaller, the fallback if RAM swaps; 768-dim and needs query/document prompt prefixes added in `embed()`).
- `BAAI/bge-reranker-v2-m3` cross-encoder, run locally via sentence-transformers on the `mps` device
- Structured output: Ollama's `format=<JSON schema>` (generated from the Pydantic model), validated with Pydantic, plus our own small retry loop on schema or citation failure. No Instructor. Disable thinking mode for structured calls: with Gemma 4 thinking on, the JSON can land in the reasoning field instead of the content. Keep the schema flat (small models are reported to break on schemas nested 3+ levels deep).
- **Thin model layer:** every LLM and embedding call goes through one module (e.g. `opspulse/llm.py` with `complete()`, `complete_structured()`, `embed()`). Pipeline code never imports `ollama` directly, so swapping the backend (e.g. for the judge only) is a one-file change. The layer supports `LLM_BACKEND=ollama|openai|anthropic` (default `ollama`). Hosted backends are optional and read their API key from `.env` only when selected, so the default setup still needs no keys. This mirrors how production teams swap providers; benchmark any non-default backend before reporting its numbers (rule 6).
- RAGAS-style metrics in a custom eval CLI; Langfuse (self-hosted) plus a structured JSON trace log for observability

Known caveat: Postgres `ts_rank_cd` full-text ranking is not true BM25. Document this trade-off, or switch to ParadeDB `pg_search` if the benchmark shows it matters.

## Architecture

### Ingestion pipeline (async)
upload → SHA-256 of the content → **no-op if the hash is unchanged** → enqueue → Docling/converter to Markdown → heading-aware chunker → contextual enrichment → embed + build tsvector → **atomically replace** the document's old chunks in one transaction.

- Chunker: splits on `#`/`##`/`###`; chunks are **250–600 tokens** with **10–15% token overlap** between sequential chunks in the same section; code blocks are never split.
- Chunk metadata (JSONB): heading hierarchy (H1 → H2 → H3), `source_file`, `updated_at`, `access_role`.
- Contextual Retrieval: send the whole document (placed first in the prompt so its KV cache is reused; see "Prompt caching" below) plus the chunk to the local LLM, which writes a 1–2 sentence context header explaining which document and system the chunk belongs to. That header is prepended to the chunk before embedding *and* before keyword indexing.
- Chunk table columns: raw text, contextualized text, JSONB metadata (GIN index), vector (HNSW index), tsvector (the GIN index on it is our pick; the spec only requires the column).

### Query pipeline
chat history + message → **query rewriter** (standalone, search-optimized query) → **vector (cosine) top-25** and **full-text top-25**, both RBAC-filtered in SQL → **RRF** merge with `score(d) = Σ 1/(60 + rank_m(d))` → **cross-encoder rerank** of the top-25 → keep the **top-5 above the threshold**, otherwise refuse → prompt with chunks wrapped in `<document id="chunk_42" source="runbook.md">…</document>` tags (untrusted data kept separate from instructions) → structured output → citation validator → trace.

- The retrieval service takes `(query, user_role)`.
- Candidate count: the spec's comparison table says "Top-30 candidates", but its detailed steps say top-25 per retriever and top-25 merged into the reranker. We follow the detailed steps (25).
- The citation validator checks every cited `chunk_id` against the (up to) 5 chunks actually sent in that request.
- Every query logs a trace with: original query, rewritten query, retrieved chunk IDs + RRF scores, reranker scores, prompt/completion tokens, cost in USD, and per-stage latency (retrieval ms, rerank ms, LLM TTFT).

### Spec gaps / project decisions
- **Role visibility:** the spec doesn't say whether roles are hierarchical. We treat them as hierarchical (`admin` ⊇ `engineer` ⊇ `public`), enforced in SQL as `access_role = ANY(<roles allowed for user_role>)`.
- **"Multi-tenant":** this appears only in the spec's title. The spec's details only define RBAC roles, so a `tenant_id` is not required unless we decide to add one.
- **Architecture diagram:** the spec mentions an "interactive architecture diagram below", but the PDF doesn't include one. The README diagram is ours to design.
- **Local models:** we use Ollama instead of hosted APIs. The spec explicitly allows "local Ollama SDKs".
- **Prompt caching:** the spec's "prompt caching to keep costs tiny" assumes a paid API. Locally the API cost is $0. The enrichment prompt puts the full document first and the chunk last, so Ollama can reuse the KV cache for that prefix across chunks of the same document. The README documents the trade-off (slower ingestion than a hosted API).
- **"Cost in USD":** traces record prompt/completion token counts and API cost = $0. The README cost profile also shows an *estimated equivalent* hosted-API cost per 1,000 queries, derived from token counts, so the "retrieve 25 → rerank to 5 vs. stuff 20 chunks" token-savings comparison still holds.
- **Judge reliability:** a small local judge is noisy. If eval scores look unstable, only the judge may be pointed at a stronger backend through the model layer. Every eval run records which judge model produced it.
- **CI eval:** GitHub runners have no GPU, so CI uses `LLM_MODEL_CI` (a smaller model) and/or a subset of the golden set. The full 30-question run is done locally and its results are committed.

## Build order and evaluation

The steps below follow the spec's roadmap numbering.

1. **Corpus + eval harness first.** Build a corpus of 20–30 technical docs: Kubernetes/Postgres runbooks and realistic synthetic post-mortems with specific error codes (e.g. `ERR-4523`), ticket IDs (e.g. `INC-8842`), CLI flags and architecture tables. Write a 30-case golden dataset JSON with `question`, `expected_answer`, `expected_source_docs` and `user_role`. Include exact-code queries (e.g. "OOMKilled exit code 137"), multi-hop questions across two files, and unanswerable questions (to test refusal). Adding RBAC test cases is a project extension. The eval CLI runs the pipeline over all 30 questions and scores **Context Recall**, **Context Precision** and **Faithfulness** with an LLM-as-judge. It must also run as an automated CI evaluation suite, not just locally.
2. **Async ingestion + structure-aware/contextual chunking.**
3. **Hybrid search + RBAC.**
4. **Query rewriting + cross-encoder reranking.** Then run the eval comparing vector-only against hybrid + reranking, and record the Context Precision/Recall gain in the README.
5. **Constrained generation, citations + observability.**

The README is structured around engineering trade-offs and measured metrics. It must include an architecture diagram of both pipelines (async ingestion and multi-stage query), a **4-row ablation table** of real 30-question eval scores, and a latency/cost profile: **p50/p95 latency + average cost per 1,000 queries**, including the input-token savings of retrieving 25 and reranking to 5 versus stuffing 20 raw chunks into the prompt. The ablation rows are:
1. naive baseline (fixed chunks, vector only)
2. + structure-aware/contextual chunking
3. + hybrid RRF
4. + reranking

So the naive baseline must stay runnable as a configurable pipeline variant.

The README also has a short "Why these choices" section covering the stack trade-offs: `bge-m3` vs. Qwen3-Embedding vs. hosted embeddings, local vs. hosted LLM (and the `LLM_BACKEND` switch), self-hosted vs. Cohere reranking, and `ts_rank_cd` vs. true BM25.

## Prerequisites

Target machine: Apple Silicon Mac (M4, 16 GB RAM).

- Docker Desktop, running (Postgres 16 + pgvector; later Langfuse)
- `uv` (`brew install uv`), which installs Python 3.12. Do not use the system Python 3.9.
- Ollama (`brew install ollama`), then `ollama pull gemma4` and `ollama pull bge-m3`
- About 10 GB free disk space for models, Docker images and packages. The reranker and Docling weights download on first use.
- RAM is tight on 16 GB: Gemma 4 (~9.6 GB) + reranker (~2 GB) + Postgres. Cap Docker Desktop's memory at ~4 GB and close other heavy apps during eval runs. If it swaps, fall back to a smaller `LLM_MODEL` (e.g. `qwen3:8b`, ~5 GB) and benchmark the switch.
- No API keys for the default setup. `.env` holds `DATABASE_URL`, `OLLAMA_HOST`, `LLM_BACKEND`, `LLM_MODEL` and `EMBED_MODEL`, plus `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` only if a hosted backend is selected.

## Commands

No code scaffold exists yet. Add build, run, test, single-test and eval commands here once `pyproject.toml` and `docker-compose.yml` exist.
