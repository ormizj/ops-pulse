# OpsPulse

**A multi-tenant engineering runbook and incident triage engine.**

OpsPulse is an internal knowledge assistant for engineering teams. It ingests Markdown runbooks, post-mortem PDFs and JSON/CSV incident tickets, and answers developer questions with strict inline citations, role-based access control and measured evaluation metrics.

> **Status:** early development. The architecture diagram, ablation benchmark and latency/cost profile will be added here as each stage lands and is measured.

## Why it exists

Most "chat with your docs" demos fail in production. OpsPulse is built to address the specific ways naive RAG breaks:

| Problem | Naive RAG | OpsPulse |
|---|---|---|
| Ingestion | Synchronous upload blocks the request | Background job queue, idempotent through content hashing |
| Chunking | Fixed-size splits break tables and code | Structure-aware splitting by headings, plus contextual enrichment |
| Exact matches | Vector search misses `ERR-4523` or `INC-8842` | Hybrid vector + full-text search, merged with Reciprocal Rank Fusion |
| Precision | Top-5 cosine results go straight to the LLM | Top-25 candidates, reranked by a cross-encoder down to 5 |
| Permissions | "Don't show admin data" written into the prompt | Role filtering in the SQL `WHERE` clause, before ranking |
| Hallucinations | Free-form answers | Schema-validated output with citations checked in code, and refusal when evidence is weak |
| Quality | Manual "vibe checks" | Automated evaluation on a 30-question golden dataset, also run in CI |

## How it works

- **Ingestion pipeline:** upload, content hash check, background job, conversion to Markdown, heading-aware chunking, contextual enrichment, then embeddings plus a full-text index, replacing the document's old chunks in one transaction.
- **Query pipeline:** query rewriting, RBAC-filtered vector and full-text search, RRF fusion, cross-encoder reranking, structured generation with citations, a citation validator, and a trace log for every request.

## Stack

- **Python 3.12**, FastAPI, raw SQL with `psycopg` (no ORM, no LangChain/LlamaIndex)
- **PostgreSQL 16 + pgvector**, with Procrastinate as the job queue
- **Local models via Ollama:** no API keys needed by default. See [docs/MODELS.md](docs/MODELS.md) for the models, links and the reasoning behind each choice.
- **Docling** for converting PDFs to Markdown
- **Langfuse** and structured JSON logs for observability

## Prerequisites

- Apple Silicon Mac with 16 GB of RAM or more
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [uv](https://docs.astral.sh/uv/), which installs Python 3.12
- [Ollama](https://ollama.com), with the models pulled as described in [docs/MODELS.md](docs/MODELS.md#setup)

## Documentation

| Document | Contents |
|---|---|
| [docs/MODELS.md](docs/MODELS.md) | The models used, where each one runs, links to their model cards, the reasons for each choice, and the memory budget |
