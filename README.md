# Mini AvatarOps

A production-grade AI avatar chat backend built as a portfolio demonstration.
The system shows end-to-end engineering discipline across RAG, conversation memory, prompt versioning, automated evaluation, observability, and CI/CD — not just a working demo, but a maintainable, testable service.

---

## Architecture Summary

- **FastAPI monolith** — async HTTP API with Pydantic v2 schema enforcement and auto-generated OpenAPI docs
- **Retrieval-Augmented Generation** — avatar persona documents embedded in ChromaDB; top-k chunks injected per turn
- **Sliding-window memory** — last N turns verbatim + periodic Haiku summarization; persisted in SQLite
- **Versioned prompts** — all prompt templates live in `app/prompts/v1/` as Markdown files, not inline strings
- **LLM client abstraction** — `LLMClient` protocol decouples service logic from the Anthropic SDK; swappable for tests and future providers
- **LLM-as-judge evaluation** — automated eval harness runs in CI; Haiku judges score persona consistency, groundedness, and refusal quality
- **Full observability** — every Anthropic API call logs model, tokens, cost, and latency via structlog

---

## Tech Stack

| Layer | Technology |
|---|---|
| HTTP framework | FastAPI + Pydantic v2 |
| LLM provider | Anthropic API (Claude Sonnet + Haiku) |
| Vector store | ChromaDB (dev) → pgvector migration path |
| Relational DB | SQLite (dev) → Postgres |
| Logging | structlog (structured JSON) |
| Testing | pytest + pytest-asyncio |
| Evaluation | Custom LLM-as-judge harness |
| Containers | Docker + Docker Compose |
| CI/CD | GitHub Actions |

---

## Setup

> **Phase 1 in progress.** Full setup instructions will be added when the development environment and first runnable endpoint are complete.
>
> For now: see [`docs/architecture.md`](docs/architecture.md) for a full system overview and [`docs/adr/`](docs/adr/) for all architectural decision records.

---

## Documentation

| Document | Description |
|---|---|
| [`docs/architecture.md`](docs/architecture.md) | System overview, ASCII component diagram, data flow, design principles |
| [`docs/adr/001-framework-fastapi.md`](docs/adr/001-framework-fastapi.md) | Why FastAPI |
| [`docs/adr/002-llm-client-abstraction.md`](docs/adr/002-llm-client-abstraction.md) | Why we abstract the LLM client |
| [`docs/adr/003-vector-store-chromadb.md`](docs/adr/003-vector-store-chromadb.md) | ChromaDB for dev; pgvector migration path |
| [`docs/adr/004-memory-strategy.md`](docs/adr/004-memory-strategy.md) | Sliding window + summarization strategy |
| [`docs/adr/005-evaluation-strategy.md`](docs/adr/005-evaluation-strategy.md) | LLM-as-judge evaluation design |

---

## Project Status

| Phase | Status | Description |
|---|---|---|
| Phase 0 | ✅ In progress | Discovery, architecture docs, ADRs |
| Phase 1 | Planned | Dev environment, health endpoint, LLM client, first eval |
| Phase 2 | Planned | RAG pipeline, memory service, chat endpoint |
| Phase 3 | Planned | Agent tools, multi-turn workflows |
| Phase 4 | Planned | CI/CD, Docker, cost dashboard |

---

## Development Conventions

- **Branches:** `feature/*`, `fix/*`, `chore/*`, `eval/*`, `docs/*` — never commit directly to `main` or `develop`
- **Commits:** [Conventional Commits](https://www.conventionalcommits.org/) — `feat:`, `fix:`, `test:`, `docs:`, `chore:`, `eval:`
- **Tasks:** tracked in [`docs/prompt-log.md`](docs/prompt-log.md)
- **ADRs:** any new architectural decision gets an ADR in `docs/adr/` before code is written
