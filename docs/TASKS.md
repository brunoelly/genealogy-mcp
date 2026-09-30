# Genealogy MCP: Task List

This is the working checklist for building the system described in [SPECIFICATION.md](SPECIFICATION.md). Work top to bottom. Each task lists what to do, the files it touches, and how you know it is done. Section numbers in parentheses, such as (§5), point to the specification.

Docker runs as Docker Engine inside WSL2 Ubuntu; Docker Desktop is not used. Every `docker` command runs in an Ubuntu shell from the repository root. For a clone at `C:\Projects\genealogy-mcp`, that is:

```bash
cd /mnt/c/Projects/genealogy-mcp
```

From a Windows terminal (Git Bash or PowerShell), you can also prefix any command with `wsl`, for example `wsl docker compose ps`, when the current directory is the repository root. Git runs from either side; `.gitattributes` keeps line endings as LF.

Status legend: `[ ]` to do, `[x]` done.

---

## Phase 0: Prerequisites

### 0.1 Check Docker Engine in WSL2 Ubuntu

- [x] Ubuntu runs as a WSL 2 distribution (`wsl -l -v` shows `VERSION 2`).
- [x] Docker Engine and the Compose plugin are installed inside Ubuntu (Engine 27 or newer, Compose v2.20 or newer).
- [x] systemd is enabled in `/etc/wsl.conf` (`[boot] systemd=true`), and `systemctl is-enabled docker` reports `enabled`, so the daemon starts with the distribution.
- [x] The Ubuntu user is in the `docker` group, so `docker` works without `sudo`.
- [x] Check that it works:

  ```bash
  docker version
  docker compose version
  docker run --rm hello-world
  ```

- [x] Resources: WSL has at least 3.5 GB of RAM and 20 GB of free disk (check with `free -h` and `df -h /`). That covers the Compose limits (512 MB Postgres + 2 GB app). By default WSL gets half of the host's RAM.
- [ ] Optional, only if the first image build or embedding run is killed for lack of memory: create `%UserProfile%\.wslconfig` with the content below, then run `wsl --shutdown` from Windows (this stops all running containers).

  ```ini
  [wsl2]
  memory=4GB
  swap=4GB
  ```

**Done when** all three commands succeed inside Ubuntu.

### 0.2 Initialize the repository

- [x] Create the Git repository on branch `main` and connect it to `origin`.
- [x] Create the empty folders: `src/genealogy_mcp`, `tests/unit`, `tests/integration`, `tests/fixtures`, `evals`, `corpus`, `alembic/versions`.
- [x] Add a `.gitkeep` to `corpus/`, `tests/fixtures/`, `evals/`, and `alembic/versions/` so those folders exist in fresh clones. The contents of `corpus/` stay ignored.
- [x] Add `.gitattributes` with `* text=auto eol=lf`, so files mounted into Linux containers never get Windows line endings, and mark PDFs and images as binary.
- [x] Add `.gitignore` early, so sample PDFs, `.env`, and private evaluation files can never be committed by accident. Task 1.1 extends it.
- [x] Add `.editorconfig` so every editor uses UTF-8, LF, and the same indentation.
- [x] Open the folder in Cursor as its own workspace.

**Done when** `git status` works and the folder tree matches §4.

Note on file location: a clone on the Windows drive is reached from Ubuntu through `/mnt/c/...`. Bind mounts from `/mnt/c` are slower than the Linux filesystem, but only `src/`, `tests/`, `alembic/`, `evals/`, and `corpus/` are mounted. The Python environment, the model cache, and the Postgres data live inside the image or in named volumes on the Linux side. That is fast enough for this project. If file access in containers ever becomes a bottleneck, move the repository to `~/projects/genealogy-mcp` in Ubuntu and open it in Cursor through the WSL remote connection.

### 0.3 Collect test PDFs

- [ ] Choose 5–20 PDFs for the local corpus. Include:
  - [ ] at least one born-digital PDF (text can be selected),
  - [ ] at least one scanned PDF (text cannot be selected), to exercise OCR,
  - [ ] at least one non-English document, if you have one.
- [ ] Copy them into `C:\Projects\genealogy-mcp\corpus\`. Never commit them.
- [ ] Check which files have a text layer (runs in a throwaway container, nothing installed):

  ```bash
  docker run --rm -v "$PWD/corpus":/corpus:ro python:3.12-slim bash -c \
    "pip install -q pymupdf && python -c \"
  import pymupdf, pathlib
  for p in sorted(pathlib.Path('/corpus').glob('*.pdf')):
      d = pymupdf.open(p)
      chars = sum(len(pg.get_text().strip()) for pg in d)
      print(f'{p.name}: {d.page_count} pages, {chars} text chars ->', 'text layer' if chars > 40 * d.page_count else 'needs OCR')
  \""
  ```

**Done when** `corpus/` contains the sample set, the check above shows at least one `text layer` file and one `needs OCR` file, and `git status` does not list the PDFs.

---



## Phase 1: Cited search over a few PDFs



### 1.1 Repository hygiene files (§4, §12)

- [x] `.gitignore`: Python artifacts (`.venv/`, `__pycache__/`, `*.py[cod]`, `*.egg-info/`, `build/`, `dist/`), tool caches and coverage reports, `.env` and `.env.*` (except `.env.example`), `corpus/*` (except `.gitkeep`), private eval files, database dumps, and editor/OS files.
- [x] `.dockerignore` as an allow-list: exclude everything (`*`), then re-include only `pyproject.toml`, `uv.lock`, `README.md`, `src/`, `alembic/`, and `alembic.ini`, and drop `**/__pycache__` and `**/*.py[cod]`. Tests, docs, evals, and the corpus reach the dev container through bind mounts, so secrets and PDFs can never enter an image layer.
- [x] `.env.example`, with placeholders and no real secrets:
  ```text
  POSTGRES_USER=genealogy
  POSTGRES_PASSWORD=change-me
  POSTGRES_DB=genealogy
  DATABASE_URL=postgresql+psycopg://genealogy:change-me@postgres:5432/genealogy
  MCP_TRANSPORT=streamable-http
  HOST=0.0.0.0
  PORT=8000
  CORPUS_ROOT=/corpus
  BLOB_BACKEND=local
  EMBEDDING_MODEL=intfloat/multilingual-e5-small
  EMBEDDING_DIM=384
  EMBEDDING_BATCH_SIZE=16
  SENTENCE_TRANSFORMERS_HOME=/models
  OCR_LANGUAGES=por+eng
  OCR_MIN_CHARS=40
  SEARCH_MAX_LIMIT=10
  SEARCH_MAX_CHARS=4000
  LOG_LEVEL=INFO
  ```

- [x] Copy it to `.env` and set a real password there. One way, which fills every `change-me` with the same random value:

  ```bash
  sed "s/change-me/$(openssl rand -hex 24)/g" .env.example > .env
  ```

**Done when** `git status` shows `.env.example` but not `.env` or the PDFs, and a throwaway build shows that the Docker context contains only the allow-listed files:

```bash
printf 'FROM busybox\nCOPY . /ctx\nRUN find /ctx | sort\n' | docker build --no-cache --progress=plain -f - .
```

### 1.2 `pyproject.toml` (§4 Python standards, Dependencies)

- [ ] Project metadata: name `genealogy-mcp`, `requires-python = ">=3.12"`, `src` layout.
- [ ] Runtime dependencies: `mcp>=2,<3`, `uvicorn`, `pydantic`, `pydantic-settings`, `sqlalchemy>=2`, `psycopg[binary]>=3`, `pgvector`, `alembic`, `pymupdf`, `pytesseract`, `sentence-transformers`.
- [ ] A dev dependency group with `pytest`, `ruff`, and `mypy`.
- [ ] Point `torch` at the PyTorch CPU index through `[tool.uv.sources]` and `[[tool.uv.index]]`, so the CUDA wheels are never pulled.
- [ ] Tool configuration:
  - [ ] `[tool.ruff]`: `line-length = 100`, `target-version = "py312"`, and lint rules `E`, `F`, `I`, `B`, `UP`, `SIM`.
  - [ ] `[tool.mypy]`: `strict = true`, `files = ["src"]`, plus a `pgvector` / `pymupdf` ignore-missing-imports override if those packages ship no type hints.
  - [ ] `[tool.pytest.ini_options]`: `testpaths = ["tests"]`, plus the marker `integration: needs the Compose database`.
- [ ] `[project.scripts]`: `genealogy-mcp = "genealogy_mcp.server:main"`.

**Done when** the file parses. The lock file is created in task 1.4.

### 1.3 `Dockerfile` (§5)

- [ ] `base` stage:
  - [ ] `FROM python:3.12-slim`
  - [ ] `apt-get install --no-install-recommends tesseract-ocr tesseract-ocr-por tesseract-ocr-eng`, then clean the apt lists.
  - [ ] `COPY --from=ghcr.io/astral-sh/uv:<pinned version> /uv /uvx /bin/`
  - [ ] Environment: `PYTHONDONTWRITEBYTECODE=1`, `PYTHONUNBUFFERED=1`, `UV_LINK_MODE=copy`, `UV_PROJECT_ENVIRONMENT=/opt/venv`, and `/opt/venv/bin` on `PATH`.
  - [ ] `WORKDIR /app`
- [ ] `dev` stage:
  - [ ] Copy `pyproject.toml` and `uv.lock`, then `uv sync --frozen --no-install-project` (all groups). Dependencies install in their own layer, so later code edits do not reinstall them.
  - [ ] Source is bind-mounted by Compose, not copied.
  - [ ] Command: `uv run --no-sync python -m genealogy_mcp`.
- [ ] `runtime` stage:
  - [ ] `uv sync --frozen --no-dev`, then copy `src/`, `alembic/`, and `alembic.ini`.
  - [ ] Create user `app` with uid 10001 and switch to it with `USER app`.
  - [ ] `EXPOSE 8000`.
  - [ ] `HEALTHCHECK` calling `http://127.0.0.1:8000/health` with a Python one-liner, so curl is not needed.
  - [ ] `CMD ["python", "-m", "genealogy_mcp"]`.

**Done when** `docker build --target dev -t genealogy-mcp:dev .` succeeds. It fails until the lock file exists, so do task 1.4 first if necessary.

### 1.4 Create the lock file without a host Python

- [ ] Generate `uv.lock` with a throwaway uv container:
  ```bash
  docker run --rm -v "$(pwd)":/app -w /app ghcr.io/astral-sh/uv:python3.12-bookworm-slim uv lock
  ```

- [ ] Commit `uv.lock`.
- [ ] Whenever you add a dependency later, use `docker compose run --rm app uv add <package>`.

**Done when** `uv.lock` exists and contains CPU builds of `torch` (no `nvidia-`* packages).

### 1.5 `compose.yaml` (§5)

- [ ] `postgres` service:
  - [ ] `image: pgvector/pgvector:pg16`
  - [ ] `env_file: .env`
  - [ ] `volumes: [pgdata:/var/lib/postgresql/data]`
  - [ ] `ports: ["127.0.0.1:5432:5432"]`
  - [ ] `mem_limit: 512m`
  - [ ] healthcheck: `pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB`, interval 5s, 10 retries.
- [ ] `app` service:
  - [ ] `build: { context: ., target: dev }`
  - [ ] `env_file: .env`
  - [ ] `depends_on: { postgres: { condition: service_healthy } }`
  - [ ] `ports: ["127.0.0.1:8000:8000"]`
  - [ ] Volumes: `./src:/app/src`, `./tests:/app/tests`, `./alembic:/app/alembic`, `./alembic.ini:/app/alembic.ini`, `./evals:/app/evals`, `./corpus:/corpus:ro`, `model-cache:/models`.
  - [ ] `mem_limit: 2g`
- [ ] Top-level `volumes:` with `pgdata` and `model-cache`.

**Done when** `docker compose config` prints the merged file without errors, and `docker compose up -d postgres` reports the service as healthy in `docker compose ps`.

### 1.6 Package skeleton, settings, and logging (§4)

- [ ] `src/genealogy_mcp/__init__.py` with `__version__ = "0.1.0"`.
- [ ] `config.py`:
  - [ ] A `Settings(BaseSettings)` class with every variable from `.env.example`, typed. Settings are immutable (`frozen=True`).
  - [ ] Validators: `MCP_TRANSPORT` must be `stdio` or `streamable-http`; `EMBEDDING_DIM` must be positive; `CORPUS_ROOT` must exist when the server starts.
  - [ ] `get_settings()` wrapped in `functools.lru_cache`. Tests clear the cache.
- [ ] `logging_setup.py`:
  - [ ] A JSON formatter that writes `timestamp`, `level`, `logger`, `message`, and any `extra` fields.
  - [ ] A single handler on stderr. Nothing writes to stdout.
- [ ] `paths.py`:
  - [ ] `resolve_inside_root(root: Path, relative: str) -> Path`. It rejects absolute paths, drive letters, and anything that resolves outside `root`.
- [ ] Unit tests in `tests/unit/test_paths.py`, `tests/unit/test_config.py`:
  - [ ] `..`, `/etc/passwd`, `C:\x`, and `sub/../../x` are all rejected.
  - [ ] A valid nested path is accepted.
  - [ ] An invalid transport raises an error.

**Done when** `docker compose run --rm app pytest tests/unit` passes.

### 1.7 Database models and session (§6)

- [ ] `db/session.py`: create the engine from `DATABASE_URL` with `pool_pre_ping=True`; provide a `sessionmaker` and a `session_scope()` context manager that commits or rolls back.
- [ ] `db/models.py` with SQLAlchemy 2.0 `Mapped[...]` models:
  - [ ] `Source`: `id` UUID primary key, `filename`, `sha256` (unique), `byte_size`, `page_count`, `language` (nullable), `ingested_at`, `status` (enum `pending|ready|failed`), `error` (nullable).
  - [ ] `Page`: `id`, `source_id` foreign key with cascade delete, `page_number`, `text`, `extraction` (enum `text_layer|ocr`), `ocr_confidence` (nullable). Unique on `(source_id, page_number)`.
  - [ ] `Chunk`: `id`, `source_id` foreign key, `page_start`, `page_end`, `chunk_index`, `content`, `content_hash`, `token_count`, `embedding` `Vector(EMBEDDING_DIM)` nullable, and `tsv` as a computed column `to_tsvector('simple', content)`. Unique on `(source_id, content_hash)`.
  - [ ] `QueryLog`: `id`, `created_at`, `query`, `chunk_ids` (UUID array), `latency_ms`.

**Done when** `docker compose run --rm app python -c "import genealogy_mcp.db.models"` runs without errors.

### 1.8 Alembic migrations (§6)

- [ ] `alembic.ini` with `script_location = alembic`. The URL is not stored in this file.
- [ ] `alembic/env.py` reads `DATABASE_URL` from `Settings` and sets `target_metadata` to the models' metadata.
- [ ] Generate the first revision:
  ```bash
  docker compose run --rm app alembic revision --autogenerate -m "initial schema"
  ```

- [ ] Edit the generated file by hand:
  - [ ] First line of `upgrade()`: `op.execute("CREATE EXTENSION IF NOT EXISTS vector")`.
  - [ ] Add a GIN index on `chunks.tsv`.
  - [ ] Add an HNSW index: `CREATE INDEX ix_chunks_embedding ON chunks USING hnsw (embedding vector_cosine_ops)`.
  - [ ] `downgrade()` drops the indexes and tables in reverse order.
- [ ] Apply it:
  ```bash
  docker compose run --rm app alembic upgrade head
  ```

**Done when** `docker compose exec postgres psql -U genealogy -d genealogy -c "\dt"` lists `sources`, `pages`, `chunks`, `query_log`, and `alembic_version`, and `\dx` lists the `vector` extension.

### 1.9 Blob storage interface (§14)

- [ ] `storage/blob.py`:
  - [ ] A `BlobStore` Protocol with `open(key) -> BinaryIO` and `put(key, data) -> None`.
  - [ ] `LocalVolumeStore(root)`, which reads files relative to `CORPUS_ROOT` through `resolve_inside_root`. In phase 1 the key is the relative filename. `put` raises `NotImplementedError`, because the corpus is mounted read-only.
  - [ ] `get_blob_store(settings)`, which returns the store for `BLOB_BACKEND` (only `local` for now).

**Done when** a unit test opens a fixture file through `LocalVolumeStore` and an escaping path raises an error.

### 1.10 PDF text extraction and OCR (§7)

- [ ] `ingest/extract.py`:
  - [ ] `extract_pages(stream) -> list[PageText]`, where `PageText` is a dataclass with `page_number`, `text`, `extraction`, and `ocr_confidence`.
  - [ ] Uses PyMuPDF `page.get_text("text")`.
- [ ] `ingest/ocr.py`:
  - [ ] `ocr_page(page) -> tuple[str, float]`: render the page at 300 DPI with `page.get_pixmap(dpi=300)`, run `pytesseract.image_to_data` with `OCR_LANGUAGES`, and return the text and mean confidence.
- [ ] In `extract_pages`, call `ocr_page` only when the stripped text is shorter than `OCR_MIN_CHARS`.
- [ ] Fixtures in `tests/fixtures/`: generate a two-page synthetic text PDF, and a one-page image-only PDF, with a small script (`tests/fixtures/make_fixtures.py`) that uses PyMuPDF. No real family data.
- [ ] Unit tests:
  - [ ] The text PDF returns `text_layer` for every page.
  - [ ] The image PDF takes the OCR branch (mock `pytesseract` in the unit test).

**Done when** the unit tests pass, and one manual run against your scanned sample PDF prints readable text:

```bash
docker compose run --rm app python -c "from genealogy_mcp.ingest.extract import extract_pages; ..."
```



### 1.11 Chunking (§8)

- [ ] `ingest/chunk.py`:
  - [ ] `count_tokens(text)`, using the embedding model's tokenizer (cached) so counts match the model.
  - [ ] `chunk_pages(pages, filename) -> list[ChunkDraft]`:
    - [ ] One chunk per page when the page is under 800 tokens.
    - [ ] Otherwise, windows of about 600 tokens with 80 tokens of overlap, split on paragraph or sentence boundaries when possible.
    - [ ] Chunks never cross a page boundary.
    - [ ] `content` starts with the header `source: {filename} | page: {n}`.
    - [ ] `content_hash` is the SHA-256 of `content`.
  - [ ] Skip empty pages.
- [ ] Unit tests:
  - [ ] A short page produces exactly one chunk.
  - [ ] A long synthetic page produces several chunks with overlap, all with the same page number.
  - [ ] No chunk spans two pages.
  - [ ] The same input gives the same hashes.

**Done when** the chunk unit tests pass.

### 1.12 Embeddings (§4, §9)

- [ ] `rag/embeddings.py`:
  - [ ] `get_model()`, cached, which loads `SentenceTransformer(EMBEDDING_MODEL, device="cpu")` with weights stored under `/models`.
  - [ ] At load time, check that the model's output dimension equals `EMBEDDING_DIM`, and fail with a clear message if it does not.
  - [ ] `embed_passages(texts) -> list[list[float]]`: add the `passage:`  prefix, batch by `EMBEDDING_BATCH_SIZE`, `normalize_embeddings=True`.
  - [ ] `embed_query(text) -> list[float]`: add the `query:`  prefix.
- [ ] Download the model once, into the volume:
  ```bash
  docker compose run --rm app python -c "from genealogy_mcp.rag.embeddings import get_model; get_model()"
  ```

- [ ] Unit test with a fake model object: prefixes are applied and batching splits correctly.

**Done when** the download command finishes, and running it again starts in seconds without downloading.

### 1.13 Ingest pipeline and CLI (§7)

- [ ] `ingest/pipeline.py`, `ingest_source(relative_path) -> IngestResult`:
  1. [ ] Resolve and open the file through the blob store.
  2. [ ] Compute SHA-256 while reading.
  3. [ ] If a `Source` with that hash exists with status `ready`, return it unchanged.
  4. [ ] Create or reuse the `Source` row with status `pending`.
  5. [ ] Extract pages and upsert `Page` rows.
  6. [ ] Chunk, and insert only chunks whose `(source_id, content_hash)` is new.
  7. [ ] Embed chunks where `embedding IS NULL`, in batches, committing after each batch so a crash resumes where it stopped.
  8. [ ] Set status `ready`, `page_count`, and `ingested_at`.
  9. [ ] On any exception, set status `failed` and a short `error` (exception type and message, no document text), log it, and re-raise.
- [ ] `IngestResult`: `source_id`, `status`, `page_count`, `chunk_count`.
- [ ] `ingest/__main__.py`: `python -m genealogy_mcp.ingest [folder]` ingests every `*.pdf` in the folder (default `CORPUS_ROOT`) and prints one JSON line per file to stderr.
- [ ] Log per-file start, finish, duration, page count, OCR page count, and chunk count.

**Done when**:

```bash
docker compose run --rm app python -m genealogy_mcp.ingest
```

marks every sample PDF `ready`, and running it a second time adds no rows. Check with:

```bash
docker compose exec postgres psql -U genealogy -d genealogy -c "select status, count(*) from sources group by status; select count(*) from chunks;"
```



### 1.14 Hybrid retrieval (§9)

- [ ] `rag/retrieve.py`, `search(query, limit=5, source_id=None) -> list[ChunkHit]`:
  - [ ] Clamp `limit` to `1..SEARCH_MAX_LIMIT`.
  - [ ] Full-text candidates: `tsv @@ websearch_to_tsquery('simple', :q)`, ordered by `ts_rank_cd`, top 20.
  - [ ] Vector candidates: `ORDER BY embedding <=> :qvec`, top 20.
  - [ ] Apply the optional `source_id` filter to both queries.
  - [ ] Reciprocal rank fusion: `score = Σ 1 / (60 + rank)` over both lists. Return the top `limit` hits.
  - [ ] Join `sources` to get `filename`. Truncate `content` to `SEARCH_MAX_CHARS`.
  - [ ] Write one `QueryLog` row with the query, returned chunk ids, and latency.
- [ ] `get_chunk(chunk_id) -> ChunkWithNeighbors`: the chunk, plus the previous and next chunks from the same source that share its page.
- [ ] Pydantic models `ChunkHit` and `ChunkWithNeighbors` (§9 field list).
- [ ] Unit test for the fusion function with hand-made rankings.

**Done when** a manual query returns cited hits:

```bash
docker compose run --rm app python -c "from genealogy_mcp.rag.retrieve import search; [print(h.filename, h.page_start, round(h.score,4)) for h in search('birth of <a name in your corpus>')]"
```



### 1.15 MCP server and tools (§3, §10)

- [ ] `tools/documents.py`:
  - [ ] `list_sources()`, read-only: id, filename, status, page count.
  - [ ] `ingest_pdf(filename: str)`, not read-only: calls `ingest_source` and returns `IngestResult`. Runs as a normal `def` so the SDK moves it to a worker thread.
- [ ] `tools/search.py`:
  - [ ] `search_sources(query: str, limit: int = 5, source_id: str | None = None)`, read-only.
  - [ ] `get_chunk(chunk_id: str)`, read-only.
- [ ] Each tool has a docstring the model can act on, typed arguments, a Pydantic structured result, and `ToolAnnotations` (`read_only_hint` set correctly).
- [ ] Each tool logs its name, duration, and status, never the returned text.
- [ ] `server.py`:
  - [ ] `build_server()`: create `MCPServer("genealogy-mcp", instructions=<text from §10>)` and register both tool modules.
  - [ ] `GET /health` custom route: runs `SELECT 1` and returns `{"status": "ok"}`, or HTTP 503 if the database fails.
  - [ ] `main()`: load settings, configure logging, then either `server.run(transport="stdio")` or build the Streamable HTTP app and serve it with uvicorn on `HOST:PORT`.
- [ ] `__main__.py` calls `main()`.
- [ ] Start the stack:
  ```bash
  docker compose up -d
  curl http://127.0.0.1:8000/health
  ```

**Done when** `/health` returns `{"status":"ok"}` and `docker compose logs app` shows JSON lines only.

### 1.16 Connect Cursor (§5)

- [ ] Create `.cursor/mcp.json` in the repository:
  ```json
  {
    "mcpServers": {
      "genealogy-mcp": { "url": "http://127.0.0.1:8000/mcp" }
    }
  }
  ```

- [ ] In Cursor Settings, open MCP and confirm `genealogy-mcp` is green with four tools.
- [ ] In a chat, ask "Which sources are loaded?" and confirm Cursor calls `list_sources`.
- [ ] Ask a question whose answer you know is in a specific PDF page.

**Done when** Cursor's answer cites the correct filename and page.

### 1.17 Integration test (§13)

- [ ] `tests/integration/conftest.py`: a fixture that creates a temporary schema or database, runs `alembic upgrade head`, and drops it afterwards.
- [ ] `tests/integration/test_retrieval.py`, marked `@pytest.mark.integration`:
  - [ ] Ingest two synthetic fixtures: a birth certificate and an unrelated text.
  - [ ] Assert that a birth query ranks the birth fixture first.
  - [ ] Assert that ingesting the same fixture again does not change the row counts.
- [ ] Commands:
  ```bash
  docker compose run --rm app pytest -m "not integration"
  docker compose run --rm app pytest
  ```

**Done when** both commands pass.

### 1.18 Golden question set (§13)

- [ ] `evals/questions.yaml`, about 10 entries over your real sample PDFs:
  ```yaml
  - question: "When was Maria Silva born?"
    expected_filename: "certidao_maria.pdf"
    expected_page: 1
  ```

- [ ] `src/genealogy_mcp/evals.py`: `python -m genealogy_mcp.evals` runs each question through `search`, checks whether the expected filename and page are in the top 5, and prints the hit rate plus the misses.
- [ ] Record the baseline hit rate in a short `evals/RESULTS.md`, with the date, model, and chunk settings.

**Done when** the baseline is recorded. Because this file mentions real names, keep `questions.yaml` and `RESULTS.md` out of git (add them to `.gitignore`) or use neutral wording.

### 1.19 Lint, types, and README

- [ ] `docker compose run --rm app ruff format .`
- [ ] `docker compose run --rm app ruff check .`
- [ ] `docker compose run --rm app mypy src`
- [ ] `README.md`: purpose, prerequisites (Docker only), the quick start commands from §5, how to add PDFs, how to connect Cursor, and a link to `docs/SPECIFICATION.md`.
- [ ] Build the runtime image once to make sure it works: `docker build --target runtime -t genealogy-mcp:0.1.0 .`

**Done when** all three checks pass and the runtime image builds.

### Phase 1 exit criteria

- [ ] `docker compose up -d` starts both services from a clean clone plus `.env`.
- [ ] All sample PDFs are `ready`. A second ingest adds no rows.
- [ ] Cursor answers a corpus question with the correct filename and page.
- [ ] Unit and integration tests pass. Ruff and mypy are clean.
- [ ] A baseline golden-set hit rate is recorded.

---



## Phase 1.5: Tuning, optional

Change one thing at a time and rerun `python -m genealogy_mcp.evals` after each change.

- [ ] Try page-chunk thresholds of 500 and 1000 tokens instead of 800.
- [ ] Try the full-text configuration for the corpus language instead of `simple`, and compare name queries.
- [ ] Try `BAAI/bge-small-en-v1.5` if the corpus is all English. This needs a new migration for the vector dimension and a full re-embed.
- [ ] Try raising the candidate pool from 20 to 50 before fusion.
- [ ] Keep a change only if the hit rate improves. Record each result in `evals/RESULTS.md`.

---



## Phase 2: FamilySearch reads



### 2.1 API access

- [ ] Register a FamilySearch developer application and get sandbox (integration) credentials.
- [ ] Read the API terms and rate limits. Write the limits into `docs/SPECIFICATION.md`.
- [ ] Add to `.env.example` (placeholders only): `FAMILYSEARCH_CLIENT_ID`, `FAMILYSEARCH_REDIRECT_URI`, `FAMILYSEARCH_BASE_URL`, `FAMILYSEARCH_ACCESS_TOKEN`, `FAMILYSEARCH_REFRESH_TOKEN`, `FAMILYSEARCH_CACHE_TTL_HOURS`.

**Done when** a sandbox token works in a manual request.

### 2.2 Client and cache

- [ ] Add the `httpx` dependency.
- [ ] `connectors/familysearch.py`:
  - [ ] An async `FamilySearchClient` with a timeout, a retry with backoff on 429 and 5xx that honors `Retry-After`, and token refresh when a request returns 401.
  - [ ] `search_person(given, surname, birth_year=None, place=None)`
  - [ ] `get_person(person_id)`
- [ ] Alembic migration for `fs_cache`: `request_hash` primary key, `response_json` (JSONB), `fetched_at`.
- [ ] Read-through cache: return the cached row while it is younger than the TTL.
- [ ] Unit tests with `httpx.MockTransport`: success, 429 then success, 401 then refresh, and a cache hit.

**Done when** the tests pass and a sandbox call is cached on its second run.

### 2.3 Tools

- [ ] `tools/familysearch.py` with the async tools `familysearch_search_person` and `familysearch_get_person`, both read-only with `open_world_hint=True`.
- [ ] Update the server instructions: use FamilySearch only when the loaded documents do not answer, and cite FamilySearch person ids.
- [ ] Do not log tokens or response bodies.

**Done when** Cursor can find a sandbox person and cite the id.

### 2.4 Claim validation tool

- [ ] `tools/validate.py`, `compare_claim(claim: str, chunk_id: str)`:
  - [ ] Load the chunk and return it together with the claim in a structured result, so Cursor's model judges it with the evidence in view.
  - [ ] Add a deterministic pre-check: report which names and four-digit years in the claim literally appear in the chunk.
  - [ ] Result: `supported | contradicted | absent | needs_review`, the matched terms, and the chunk citation.
- [ ] Unit tests for the literal pre-check.

**Done when** Cursor uses `compare_claim` to confirm or reject a fact against a cited page.

### Phase 2 exit criteria

- [ ] FamilySearch read tools work against the sandbox with caching and rate-limit handling.
- [ ] `compare_claim` works on corpus chunks.
- [ ] No write tools and no scraping exist.

---



## Phase 3: Structured facts and optional remote hosting



### 3.1 Genealogy schema

- [ ] Alembic migration for:
  - [ ] `persons`: id, given names, surname, sex, notes.
  - [ ] `events`: id, type (birth, baptism, marriage, death, burial, residence), date text, normalized date range, place text.
  - [ ] `event_participants`: event, person, role (principal, father, mother, spouse, witness).
  - [ ] `relationships`: person A, person B, type.
  - [ ] `citations`: links any fact to `chunk_id`, `source_id`, and page, with a confidence value and status `proposed | confirmed | rejected`.
- [ ] Read-only tools: `find_person`, `get_person_facts` (each fact returned with its citations).

**Done when** hand-inserted test rows come back through the tools with citations.

### 3.2 Batch extraction job (§11)

- [ ] Add `langgraph` and an LLM client for an OpenAI-compatible API.
- [ ] Settings: `LLM_BASE_URL`, `LLM_MODEL`, `LLM_API_KEY` (optional).
- [ ] `jobs/extract_facts.py`, a LangGraph graph:
  - [ ] Node `propose`: for each chunk of a source, ask the model for structured facts (a Pydantic schema) with the chunk citation.
  - [ ] Node `verify`: run the `compare_claim` logic for each fact. Keep only `supported` facts, stored as `proposed` citations.
  - [ ] Checkpoint state in Postgres so an interrupted job resumes.
- [ ] CLI: `docker compose run --rm app python -m genealogy_mcp.jobs run --source <id>`.
- [ ] A review tool `review_citation(citation_id, decision)` to confirm or reject a proposed fact. This is the only write tool, and it requires an explicit id.

**Done when** the job extracts facts from one sample PDF and every stored fact points at the correct page.

### 3.3 Optional local chat model

- [ ] Only if the golden-set hit rate is already good.
- [ ] Add an `ollama` service to `compose.yaml` with a memory limit and a volume for the models.
- [ ] Pull one small model and set `LLM_BASE_URL=http://ollama:11434/v1`.
- [ ] Compare the job's output against the Cursor-backed or hosted model on the same source.

**Done when** the job runs end to end on the local model, or you decide not to keep it.

### 3.4 Remote object storage (§14)

- [ ] Create a Cloudflare R2 bucket (or Supabase Storage) and an access key restricted to that bucket.
- [ ] Add `boto3` and implement `S3CompatibleStore` with the key set to the file's `sha256`.
- [ ] Settings: `BLOB_BACKEND=s3`, `S3_ENDPOINT_URL`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`.
- [ ] Upload command: `python -m genealogy_mcp.storage upload corpus/`.
- [ ] Add a `blob_key` column to `sources` by migration.

**Done when** ingest reads a PDF from the bucket and search results are unchanged.

### 3.5 Remote Postgres (§14)

- [ ] Create a Supabase or Neon free project and enable the `vector` extension.
- [ ] Check the current free quotas and the idle pause or suspend rules. Write them down in the specification.
- [ ] Put the remote `DATABASE_URL` (with `sslmode=require`) in `.env` only.
- [ ] `docker compose run --rm app alembic upgrade head` against the remote database.
- [ ] Re-ingest from the laptop so the embeddings are computed locally and written remotely.
- [ ] Run the golden set against the remote database.

**Done when** the hit rate matches the local baseline and the local Postgres container can be stopped.

### Phase 3 exit criteria

- [ ] Structured facts exist only with page citations, and each can be confirmed or rejected.
- [ ] If remote hosting was chosen, the app runs with only `DATABASE_URL` and `BLOB_BACKEND` changed.
- [ ] Nothing in the setup incurs a bill: quotas are checked and billing alerts are set wherever a card is on file.

---



## Ongoing habits

- [ ] Run `ruff`, `mypy`, and the unit tests before every commit.
- [ ] Rerun the golden set after any change to chunking, retrieval, or the model.
- [ ] Never commit `.env`, `corpus/`, or real family names in test data.
- [ ] Back up the local database before risky migrations:
  ```bash
  docker compose exec postgres pg_dump -U genealogy genealogy > backup.sql
  ```

- [ ] Update `docs/SPECIFICATION.md` whenever a decision in §1 changes.