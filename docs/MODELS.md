# Models

OpsPulse runs fully locally by default. You need no API keys and pay nothing per query. The assignment allows "OpenAI, Anthropic, or local Ollama SDKs"; the specific models below are this project's choices, not spec requirements.

## Overview

| Role | Model | Runs on | Links |
|---|---|---|---|
| LLM: answer generation, query rewriting, contextual enrichment, eval judge | `gemma4:latest` (~9.6 GB) | Ollama (Metal GPU) | [Ollama](https://ollama.com/library/gemma4) · [Google Gemma](https://ai.google.dev/gemma) |
| Embeddings | `bge-m3` (1024-dim) | Ollama | [Ollama](https://ollama.com/library/bge-m3) · [Hugging Face](https://huggingface.co/BAAI/bge-m3) |
| Reranker (cross-encoder) | `BAAI/bge-reranker-v2-m3` | sentence-transformers on `mps` | [Hugging Face](https://huggingface.co/BAAI/bge-reranker-v2-m3) |

Every model call goes through one module (`opspulse/llm.py`), so changing a model only means changing an environment variable:

| Variable | Default | Purpose |
|---|---|---|
| `LLM_BACKEND` | `ollama` | `ollama`, `openai` or `anthropic`. The hosted backends are optional and need their API key in `.env` |
| `LLM_MODEL` | `gemma4:latest` | Main LLM |
| `LLM_MODEL_CI` | a smaller model | Used in CI, because GitHub runners have no GPU |
| `EMBED_MODEL` | `bge-m3` | Embedding model |

Changing any of these is a pipeline change, so run the eval CLI and log the before/after metrics. Changing the embedding model also requires re-ingesting the corpus.

## LLM: Gemma 4

A single instruct model handles every text-generation job in the pipeline. In RAG, the model mostly has to read the retrieved chunks and answer from them; it doesn't need to know the answer itself. That lets a model small enough to run on a laptop do the job well.

- **Why Gemma 4:** it's an official Google model that was already downloaded on the target machine. It's strong at following instructions and producing structured output for its size.
- **Structured output:** Ollama's `format=<JSON schema>` constrains the output, and Pydantic validates it. Turn **thinking mode off** for structured calls: when it's on, Gemma 4 can put the JSON in the reasoning field instead of the content. Keep schemas flat, because small models are reported to break on deeply nested schemas.
- **Use the official build** (`gemma4:latest`), not community fine-tunes.
- **RAM:** at ~9.6 GB this is the largest item on a 16 GB Mac. See [Memory budget](#memory-budget).
- **Fallback:** [`qwen3:8b`](https://ollama.com/library/qwen3) (~5 GB), if the Mac starts swapping.

## Embeddings: bge-m3

Chat models like Gemma aren't trained to produce search vectors, so retrieval needs a separate, dedicated embedding model.

- **Retrieval-focused:** it's one of the most-used open-weight embedding models, from the same lab (BAAI) as the reranker.
- **8,192-token context:** chunks are up to 600 tokens plus a contextual header, so they fit with plenty of room.
- **No prompt prefixes needed**, unlike some alternatives.
- **Small:** about 1.2 GB, so it fits next to the LLM in 16 GB.
- **1024 dimensions:** the Postgres column is `vector(1024)` with an HNSW index.

| Alternative | Status | Notes |
|---|---|---|
| [`mxbai-embed-large`](https://ollama.com/library/mxbai-embed-large) | Rejected | Its 512-token limit would silently truncate chunks |
| [`qwen3-embedding:0.6b`](https://ollama.com/library/qwen3-embedding) | Candidate to benchmark | Scores higher on benchmarks and can output 1024 dimensions, so no schema change |
| [`embeddinggemma`](https://ollama.com/library/embeddinggemma) | Fallback if RAM is tight | Smaller, but 768-dim and a 2,048-token limit, and it needs query/document prefixes added in `embed()` |

## Reranker: bge-reranker-v2-m3

A cross-encoder scores the query and each candidate chunk *together*, which captures detail that a single embedding vector loses. It reranks the top 25 hybrid-search candidates down to the top 5 that pass the confidence threshold. If none pass, the system refuses with "Insufficient evidence in the knowledge base".

- The assignment names it as an option (the other is Cohere Rerank, which is a paid API).
- It runs locally through sentence-transformers on the Apple GPU (`mps`). The weights (~2 GB) download on first use.

## Memory budget

The target machine is an M4 with 16 GB of RAM.

| Component | Approx. RAM |
|---|---|
| Gemma 4 | ~9.6 GB |
| Reranker | ~2 GB |
| bge-m3 | ~1.2 GB |
| Postgres (Docker) | < 1 GB |

This is tight. Cap Docker Desktop's memory at ~4 GB (Settings → Resources → Memory) and close other heavy apps during eval runs. If the Mac swaps, switch `LLM_MODEL` to `qwen3:8b` and benchmark the change.

## Setup

```bash
brew install ollama
ollama pull gemma4
ollama pull bge-m3
```

The reranker downloads automatically the first time the pipeline runs.
