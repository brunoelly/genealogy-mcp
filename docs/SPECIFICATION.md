# Genealogy MCP + RAG Specification

| | |
| --- | --- |
| Version | 0.1 |
| Runtime | Python 3.12, MCP Python SDK v2, Docker Compose |
| MCP client | Cursor IDE |
| Container runtime | Docker Engine on WSL2 (Ubuntu) |

This document specifies a small genealogical research assistant: a Python MCP server backed by Postgres and pgvector. Everything runs in Docker. Nothing besides Docker Engine needs to be installed.

## 1. Decisions

Key technical decisions and the reasoning behind each one.

| Topic | Decision | Why |
| --- | --- | --- |
| Local test corpus | 5–20 PDFs in one Postgres database | Enough to debug chunking and retrieval. Small enough to re-ingest in minutes on a laptop. |
| Vector store | Postgres 16 + pgvector in Docker from day one | Documents, chunks, embeddings, and later person/event rows live in one database with metadata filters. The same schema moves to a managed Postgres later. |
| FAISS | Out of scope | A second index would be thrown away at the first migration. |
| What runs in Docker | Two services: `app` and `postgres` | Postgres is the dependency worth isolating. The app image owns PDF extraction and embeddings. |
| OCR | A library call inside the app image, only for pages with no text layer | A separate OCR container sits idle on born-digital PDFs and adds a network hop. |
| Agent runtime in v1 | Cursor itself | Cursor already has the model and calls MCP tools. A second agent framework would duplicate the host. |
| Agent framework later | LangGraph, one package, inside this codebase | Use it when a job must run unattended. Do not adopt LangChain agents, AutoGen, and CrewAI together. |
| LLM for answers | The model Cursor already uses | The MCP server exposes tools. It does not embed a chat model. |
| Embeddings | Local multilingual model on CPU, inside the app image | Retrieval stays free and offline. Document language is often not English. |
| Original files | Local volume now; object storage later | The database stores text, metadata, and vectors. Object storage stores the PDF bytes. |
| Google Drive as the RAG store | Out of scope | Drive has no vector search and no transactional ingest. Acceptable only as a personal drop folder that a human copies into the corpus. |
| AWS free tier as the next host | Defer | The useful AWS allowances expire after 12 months, and a forgotten RDS instance, public IPv4 address, or NAT gateway bills after that. |
| Managed free step, if local Docker is no longer enough | Supabase (Postgres + pgvector + file storage) or Neon (Postgres + pgvector) plus Cloudflare R2 for PDFs | Both match this schema. Quotas change; confirm them before relying on them. Local Docker remains the zero-cost default. |
| Secrets locally | `.env` loaded by Compose, never committed | Docker Swarm secrets and Kubernetes secrets matter when there is an orchestrator. Compose on one laptop does not have one. |
| Host installs | Docker Engine in WSL2 Ubuntu only | Python, uv, Postgres, Tesseract, and the embedding weights live in images and volumes. Tests, linting, and migrations also run in containers. |

## 2. Goal

Answer genealogical questions from a small, cited document set, with a path to FamilySearch later.

A good answer names the person, the claim, and the source page it came from. A chunk of similar text with no citation is a failed answer.

### In scope for the first usable version

- Ingest a folder of PDFs into Postgres.
- Extract text, OCR only when a page has no text layer.
- Chunk by page, embed locally, store vectors in pgvector.
- Retrieve with hybrid search: metadata filter, full text, then vector similarity.
- Expose retrieval and ingest status as MCP tools that Cursor can call.
- Run the stack with `docker compose up`.

### Out of scope until a later phase

- A local chat model (Llama, Ollama, or similar). That is the workload that fills a laptop, not Postgres or a few PDFs.
- FamilySearch writes, tree merge, or GEDCOM import.
- A general web scraper.
- Multi-agent crews.
- Publishing the database or the MCP port beyond `127.0.0.1`.

## 3. How the pieces fit

```text
Cursor IDE  (model + conversation)
    |
    |  Streamable HTTP  http://127.0.0.1:8000/mcp
    v
app container (genealogy_mcp)
    |-- tools: list_sources, ingest_pdf, search_sources, get_chunk
    |-- ingest pipeline
    |-- local embedding model (CPU)
    |
    |  internal Docker network
    v
postgres + pgvector
    sources, pages, chunks, embeddings, query log
```

Cursor is the agent. The server is a set of tools plus a database. When Cursor needs a fact, it calls `search_sources`. The tool returns chunks with source file, page, and score. Cursor writes the answer and keeps the citation.

The container uses Streamable HTTP because Cursor cannot cleanly own the stdin of a process inside Docker. The server also supports stdio (`MCP_TRANSPORT=stdio`) for a future host-launched setup, but that is not the default path.

## 4. Repository layout

```text
genealogy-mcp/
  pyproject.toml
  uv.lock
  Dockerfile               # multi-stage: dev target and runtime target
  compose.yaml
  .env.example
  .gitignore
  .dockerignore
  alembic.ini
  alembic/
    env.py
    versions/
  src/genealogy_mcp/
    __init__.py
    __main__.py            # python -m genealogy_mcp
    server.py              # MCPServer, /health, tool registration
    config.py              # frozen settings from the environment
    logging_setup.py       # JSON logs on stderr
    paths.py               # corpus-root path guard
    db/
      models.py            # SQLAlchemy 2.0 models
      session.py           # engine and session factory
    ingest/
      pipeline.py          # idempotent ingest of one PDF
      extract.py           # text layer via PyMuPDF
      ocr.py               # Tesseract fallback per page
      chunk.py             # page-aware chunks
    rag/
      embeddings.py        # local model, batched
      retrieve.py          # hybrid search
    storage/
      blob.py              # BlobStore protocol + LocalVolumeStore
    tools/
      documents.py         # list_sources, ingest_pdf
      search.py            # search_sources, get_chunk
    connectors/
      familysearch.py      # phase 2, interface only in phase 1
  tests/
    unit/
    integration/           # needs the Compose database
    fixtures/              # tiny synthetic PDFs, no real family data
  evals/
    questions.yaml
  corpus/                  # gitignored sample PDFs
  docs/
    SPECIFICATION.md
```

### Python standards

- Python 3.12, `src` layout, a single `pyproject.toml`.
- `uv` manages dependencies and the lock file, inside the dev container. Commit `uv.lock`.
- `ruff` for lint and format. `mypy` in strict mode on `src/`. `pytest` for tests. All three run through `docker compose run`.
- Settings are a frozen dataclass (or `pydantic-settings`) loaded from the environment and validated at startup: transport, database URL, corpus root, embedding model name, vector dimension. The process fails fast on a bad value.
- JSON logs go to stderr. In stdio mode stdout is the protocol stream, so nothing else may print there.
- SQLAlchemy 2.0 typed mappings. Alembic owns the schema. No hand-written schema changes against a running database.
- Pydantic models at the MCP tool boundary, for arguments and structured results.
- Read tools use `ToolAnnotations(read_only_hint=True)`.
- Ingest is the one mutating tool in phase 1. It takes a file name relative to the corpus root and refuses absolute paths and any path that resolves outside that root (`paths.py`).
- Type hints on public functions. No network access in unit tests.

### Dependencies

| Concern | Library |
| --- | --- |
| MCP server | `mcp>=2,<3` |
| HTTP server | `uvicorn` |
| Settings | `pydantic`, `pydantic-settings` |
| Database | `sqlalchemy>=2`, `psycopg[binary]>=3`, `pgvector`, `alembic` |
| PDF text | `pymupdf` |
| OCR fallback | `pytesseract` plus the `tesseract-ocr`, `tesseract-ocr-por`, and `tesseract-ocr-eng` apt packages in the image |
| Embeddings | `sentence-transformers` with a pinned model, CPU-only PyTorch wheel |
| Dev | `pytest`, `ruff`, `mypy` |

Install the CPU-only PyTorch wheel. The default wheel pulls CUDA libraries and makes the image several gigabytes larger.

Pin the embedding model name and its output dimension in settings. Changing the model is a migration: new column dimension, then re-embed every chunk.

Default model: `intfloat/multilingual-e5-small` (384 dimensions). It covers Portuguese, Spanish, English, and other languages common in family records, and it runs on CPU for a few dozen PDFs. If every sample PDF is English, `BAAI/bge-small-en-v1.5` is a fine substitute. Record the choice in `.env`. Do not download a multi-gigabyte model for this phase.

Cache the model weights in a named Docker volume (`SENTENCE_TRANSFORMERS_HOME=/models`) so recreating the container does not download them again.

## 5. Docker

### Dockerfile

One multi-stage `Dockerfile`:

- `base`: `python:3.12-slim`, apt install Tesseract and its language packs, copy `uv` from the official `ghcr.io/astral-sh/uv` image.
- `dev`: `uv sync --frozen` including dev dependencies. Source is bind-mounted, not copied, so edits apply without a rebuild.
- `runtime`: `uv sync --frozen --no-dev`, copy `src/`, create a non-root user (for example uid 10001), `USER` that user, `HEALTHCHECK` against `GET /health`, `CMD ["python", "-m", "genealogy_mcp"]`.

Set `PYTHONDONTWRITEBYTECODE=1` and `PYTHONUNBUFFERED=1`. Keep `.dockerignore` strict: exclude `.venv`, `corpus/`, `.env`, caches, and `.git`.

### compose.yaml

```text
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB (from .env)
    volumes: pgdata:/var/lib/postgresql/data
    ports: "127.0.0.1:5432:5432"
    mem_limit: 512m
    healthcheck: pg_isready

  app:
    build: { context: ., target: dev }
    env_file: .env
    depends_on: { postgres: { condition: service_healthy } }
    ports: "127.0.0.1:8000:8000"
    volumes:
      - ./src:/app/src
      - ./tests:/app/tests
      - ./alembic:/app/alembic
      - ./corpus:/corpus:ro
      - model-cache:/models
    mem_limit: 2g

volumes:
  pgdata:
  model-cache:
```

Practices:

- One user-defined bridge network. The app reaches Postgres as hostname `postgres`. There is no host Postgres install.
- Publish ports on `127.0.0.1` only. The Postgres port is published only so a database client in the IDE can inspect tables. Remove it if you do not need that.
- Postgres memory cap at 512 MB. That is enough for this corpus and protects the laptop.
- App memory cap at 2 GB because the embedding model is the heavy part. Raise it only if the process is killed during the first embed.
- Corpus mounted read-only. The container cannot rewrite sample PDFs.
- Named volume `pgdata` so a restart keeps the index.
- `.env` holds the database password and future tokens. Commit only `.env.example`.
- Logs are one JSON object per event on stderr. Do not log chunk text, PDF bytes, or tokens.
- Tesseract runs in-process for empty pages. There is no `ocr-service`.

The first `docker compose up --build` is slow because it installs the embedding stack and downloads the model once. Later starts reuse the image and the volume.

### Commands (host has only Docker)

Run these in an Ubuntu (WSL2) shell. The repository is at `/mnt/c/Projects/genealogy-mcp` from Ubuntu. Ports published on `127.0.0.1` inside WSL2 are forwarded to Windows `localhost`, so Cursor reaches the server at `http://127.0.0.1:8000/mcp`.

```bash
cd /mnt/c/Projects/genealogy-mcp
cp .env.example .env
# put 5–20 PDFs in corpus/
docker compose up --build -d
docker compose run --rm app alembic upgrade head
docker compose run --rm app pytest
docker compose run --rm app ruff check .
docker compose run --rm app mypy src
docker compose logs -f app
```

For editor support without a local Python install, open the repository in a Dev Container that uses the `dev` target of the same `Dockerfile`. It is optional. The commands above are enough.

### Cursor MCP config

`.cursor/mcp.json` in this repository:

```json
{
  "mcpServers": {
    "genealogy-mcp": {
      "url": "http://127.0.0.1:8000/mcp"
    }
  }
}
```

## 6. Data model

Hybrid storage: relational rows for anything you might filter, vectors for wording.

### `sources`

| Column | Notes |
| --- | --- |
| `id` | UUID |
| `filename` | name inside the corpus root |
| `sha256` | idempotency key, unique |
| `byte_size` | |
| `page_count` | |
| `language` | optional, set by the operator or left null |
| `ingested_at` | |
| `status` | `pending`, `ready`, `failed` |
| `error` | short failure reason, no document text |

### `pages`

| Column | Notes |
| --- | --- |
| `id` | UUID |
| `source_id` | FK |
| `page_number` | 1-based |
| `text` | extracted or OCR text |
| `extraction` | `text_layer` or `ocr` |
| `ocr_confidence` | null when the text layer was used |

### `chunks`

| Column | Notes |
| --- | --- |
| `id` | UUID |
| `source_id` | FK |
| `page_start`, `page_end` | |
| `chunk_index` | order inside the source |
| `content` | the text that was embedded |
| `content_hash` | skip unchanged chunks on re-ingest |
| `token_count` | approximate |
| `embedding` | `vector(384)`, null until embedded |
| `tsv` | `tsvector` generated from `content`, GIN index |

Index `embedding` with HNSW (`vector_cosine_ops`). With fewer than a few thousand chunks an exact scan is also fine; create the index anyway so the query shape does not change later.

The first Alembic migration runs `CREATE EXTENSION IF NOT EXISTS vector`.

### `query_log`

Store the question, the chunk ids returned, and the latency. Do not store the model's final answer. This table is how you tell whether a chunk-size change helped.

Persons, events, and relationships are a phase 3 schema (a thin GEDCOM-like set: person, event, relationship, citation). They are not required to answer "what does this document say?". Add them when you want the assistant to query a tree, not only documents.

## 7. Ingestion

One function: `ingest_source(relative_path) -> source_id`. It is safe to run twice.

1. Resolve the path inside the corpus root. Reject absolute paths and `..`.
2. Hash the file. If `sha256` already exists and status is `ready`, return the existing id.
3. Open with PyMuPDF. For each page, read the text layer.
4. If a page's text is below a small threshold (default 40 characters), OCR that page only, with Tesseract languages `por+eng`.
5. Build chunks (section 8) and insert rows whose `content_hash` is new.
6. Embed missing chunks in batches (default 16).
7. Mark the source `ready`, or `failed` with a short error.

The MCP tool `ingest_pdf` calls this function and returns `{source_id, status, page_count, chunk_count}`. `list_sources` returns id, filename, status, and page count so Cursor can see what is loaded.

A CLI entry point runs the same function for a whole folder: `docker compose run --rm app python -m genealogy_mcp.ingest /corpus`.

Callers cannot pass an arbitrary filesystem path. The corpus root is the allowlist.

## 8. Chunking

Genealogical PDFs are usually one record per page: a certificate, a register row, a letter page. Blind 500–1000 token windows split a record in half and glue two families together.

Default strategy:

- One chunk per page when the page is under 800 tokens.
- If a page is longer, split it into windows of about 600 tokens with 80 tokens of overlap, and keep `page_start` / `page_end` on every piece.
- Never cross a page boundary in phase 1. Add cross-page overlap only if the golden questions show answers sitting on page breaks.
- Prepend a header line to the embedded text: `source: {filename} | page: {n}`. It improves retrieval. The citation returned to Cursor still comes from the structured columns, not from parsing that header.

A fixed 500–1000 token window is used only inside a long page, never as the primary splitter.

## 9. Retrieval

`search_sources(query, limit=5, source_id=None) -> list[ChunkHit]`

`ChunkHit` contains `chunk_id`, `source_id`, `filename`, `page_start`, `page_end`, `content`, and `score`.

Steps:

1. Optional filter on `source_id`.
2. Full-text search on `chunks.tsv` with `websearch_to_tsquery`. Use the `simple` configuration by default so surnames are not stemmed away.
3. Embed the query with the same model and the model's query prefix (`query: ` for E5). Documents are embedded with the `passage: ` prefix at ingest time.
4. Vector search: cosine distance on `embedding`, same filter.
5. Fuse the two ranked lists with reciprocal rank fusion and return the top `limit` hits.

Names, dates, and places need exact matches, which full text finds. Vector search finds paraphrases ("birth record" versus "certidão de nascimento"). Either list alone is a weaker system.

`get_chunk(chunk_id)` returns one chunk and its neighbors on the same page so Cursor can read the surrounding lines without a second search.

Cap `limit` at 10. Cap returned characters per chunk (default 4000) so a tool result cannot flood the context window.

## 10. MCP tools (phase 1)

| Tool | Read-only | Purpose |
| --- | --- | --- |
| `list_sources` | yes | What has been ingested and whether it is ready |
| `ingest_pdf` | no | Ingest one filename from the corpus root |
| `search_sources` | yes | Hybrid search, cited chunks |
| `get_chunk` | yes | One chunk plus page neighbors |

Server instructions to the model:

> Search ingested genealogical PDFs before answering a question about their contents. Every factual claim must cite filename and page. If search returns nothing, say that the loaded documents do not contain the answer.

`GET /health` checks that the process is up and that `SELECT 1` against Postgres succeeds. The model never calls it.

## 11. Agents

### Phase 1

No agent framework. Cursor calls the four tools. That is the orchestration.

### Phase 2, still one process

Add FamilySearch as another tool group, behind an interface:

- `familysearch_search_person`
- `familysearch_get_person`

The OAuth2 client id, token, and refresh token come from the environment. Cache raw responses in an `fs_cache` table keyed by request hash, with a TTL, so reruns do not hammer the API. Honor rate limits. The official FamilySearch API is the only FamilySearch client; scraping the website is out of scope.

Validation is a tool, not a persona: `compare_claim(claim, chunk_id)` re-reads the chunk and returns whether the claim is supported, contradicted, or absent. Cursor can call it. A separate "Validation Agent" process is unnecessary at this size.

### Phase 3, unattended jobs

When you want a batch job ("extract every birth from these 20 PDFs and write citation rows") to run without a chat window, add a LangGraph graph in `genealogy_mcp.jobs` with two nodes:

1. Retrieve and propose structured facts.
2. Check each fact against the cited chunk.

Persist job state in Postgres. Run it as a command, not inside a tool call: `docker compose run --rm app python -m genealogy_mcp.jobs run`. Do not add CrewAI or AutoGen beside it.

A local open-source chat model is optional in this phase, through an OpenAI-compatible base URL in settings (`LLM_BASE_URL`). Ollama would then be a third Compose service. Leave it out until retrieval quality on the golden set is already good. A stronger model will not fix bad chunks.

## 12. Security and privacy

Family documents identify living people. Treat the corpus and the database as private.

- `corpus/` and `.env` are gitignored and excluded from the Docker build context.
- Test fixtures are synthetic. No real family documents in the repository.
- Tool logs contain ids, filenames, durations, and status. They omit page text and tokens.
- The database password is in `.env` locally, and in the host's secret store if the app ever leaves the laptop.
- Postgres and the MCP port bind to localhost.
- The app container runs as a non-root user and mounts the corpus read-only.
- FamilySearch tokens are environment variables, rotated by the operator, never written into the repository or a log.
- The ingest tool cannot read outside the corpus root.
- Phase 1 has no delete tool. Add delete later as its own explicit tool.

## 13. Tests and a golden set

Automated:

- Path guard tests: absolute paths, `..`, and drive letters are rejected.
- Chunker tests on a tiny fixture PDF with a text layer, and one fixture page that forces the OCR branch (Tesseract mocked in unit tests).
- Re-ingest test: the same bytes produce the same `source_id` and no duplicate chunks.
- Retrieval integration test against the Compose database, marked `integration`: ingest two short fixtures and assert the birth fixture outranks the unrelated one for a birth query.

Run unit tests with `docker compose run --rm app pytest -m "not integration"` and everything with `docker compose run --rm app pytest`.

Manual, and worth more than another framework:

Write `evals/questions.yaml` with about 10 questions over the real sample PDFs. Each item has the question, the expected filename, and the expected page. After every chunking or model change, run the questions through `search_sources` and record how often the expected page is in the top 5. Change one variable at a time.

## 14. Later hosting, still low cost

Move only when the laptop is no longer the right place. The application change is two settings: `DATABASE_URL` and `BLOB_BACKEND`.

| Stage | Database | PDF bytes | Cost shape |
| --- | --- | --- | --- |
| Now | Postgres + pgvector in Docker | `./corpus` volume | Zero |
| First remote step | Supabase free project with pgvector enabled, or Neon free Postgres | Supabase Storage or Cloudflare R2 | Free quotas. Supabase pauses an idle free project. Neon suspends idle compute. R2 has a free storage allowance and no egress fees. Confirm current quotas before depending on them. |
| When you choose to pay | Any managed Postgres that offers pgvector | S3 or R2 | The schema does not change |

Object storage holds the original PDF under a key equal to its `sha256`. Postgres holds the key, the text, and the vectors. The ingest pipeline reads through an interface:

```python
class BlobStore(Protocol):
    def open(self, key: str) -> BinaryIO: ...
    def put(self, key: str, data: bytes) -> None: ...
```

Implementations: `LocalVolumeStore` now, `S3CompatibleStore` later (R2 and Supabase Storage both speak the S3 API). Retrieval code does not change.

Google Drive can remain a personal archive. It is not a backend for this system. Copy files into `corpus/` or the blob store when you want them searchable.

AWS is a reasonable paid host later: the same image runs on ECS Fargate or App Runner with RDS Postgres and S3. It is a poor free learning host. The 12-month database and object-storage allowances end, and unrelated resources (public IPv4 addresses, NAT gateways, idle RDS) are what create a bill. If you use AWS during learning, set a billing alarm and delete the stack when you stop.

Do not run the embedding model on a free cloud VM unless you have measured it. Ingest can stay on the laptop, writing to the remote database. Embedding one query sentence is cheap and can run in a hosted app later.

## 15. Delivery phases

### Phase 1: cited search over a few PDFs

- Repository scaffold, Dockerfile, `compose.yaml`, `.env.example`.
- Schema and first Alembic migration.
- Ingest, page-aware chunks, local embeddings, hybrid search.
- Four MCP tools.
- Golden question file and one retrieval integration test.
- Cursor connected over Streamable HTTP.

Done when Cursor answers a question about a loaded PDF with the right filename and page, and a second ingest of the same file does not duplicate rows.

### Phase 2: FamilySearch reads

- OAuth2 client, two read tools, response cache, rate limit.
- `compare_claim` tool.
- Still no scraper and no write tools.

### Phase 3: structured facts and optional remote hosting

- Person, event, relationship, and citation tables.
- One LangGraph batch job that proposes facts and checks them against chunks.
- `S3CompatibleStore` and a remote Postgres if you leave the laptop.

## 16. Design principles

- One database from the start, so the test setup is the real setup at a smaller size.
- Two containers, not four. OCR and agents are code paths until load forces a split.
- Development, tests, lint, and migrations run in containers, so the host needs only Docker.
- Cursor remains the orchestrator. Frameworks are deferred and limited to one.
- The server does not host a chat model. It hosts tools, ingestion, and embeddings.
- Retrieval is hybrid and page-aware, because genealogy is full of exact names and one-page records.
- Every answer path carries filename and page.
- Embeddings are local, CPU-only, and multilingual, so the learning loop does not depend on a paid API and does not assume English.
- Re-ingest is idempotent, and a 10-question golden set decides whether a tuning change worked.
- Remote hosting is a connection-string change plus a blob-store interface. Drive stays a drop folder. AWS waits until you intend to pay.
- Secrets stay in an untracked `.env`. Database and MCP ports stay on localhost.
