# genealogy-mcp

An MCP server for genealogical research. It ingests PDF records (certificates, parish registers, letters), indexes them in Postgres with pgvector, and exposes cited search tools to an MCP client such as Cursor. Every answer points back to a file and page.

## Status

Early development. The repository layout and specification are in place; the server is not implemented yet. Progress is tracked in [docs/TASKS.md](docs/TASKS.md).

## Stack

- Python 3.12, [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- Postgres 16 + [pgvector](https://github.com/pgvector/pgvector)
- PyMuPDF for text extraction, Tesseract for scanned pages
- Local multilingual embeddings via sentence-transformers (CPU)
- Docker Compose for everything, including tests and migrations

## Requirements

- Docker Engine with the Compose plugin (developed on WSL2 Ubuntu)

No local Python installation is needed.

## Repository layout

```text
src/genealogy_mcp/   application package
tests/               unit and integration tests
alembic/             database migrations
evals/               retrieval evaluation (question set is kept private)
corpus/              local PDFs, not tracked
docs/                specification and task list
```

## Documentation

- [Specification](docs/SPECIFICATION.md): architecture, data model, retrieval, and hosting decisions
- [Task list](docs/TASKS.md): step-by-step implementation plan

## Privacy

Genealogical documents often identify living people. PDFs in `corpus/`, the `.env` file, and evaluation questions are excluded from version control.
