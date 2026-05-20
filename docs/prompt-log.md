# Prompt Log

All task prompts and outputs are recorded here for reproducibility and audit.

---

## task-001 — Scaffold repository structure

**Date:** 2026-05-20
**Branch:** feature/task-001-repo-scaffold
**Phase:** 0 — Discovery and Architecture

**Objective:** Create the complete folder structure for Mini AvatarOps. Directories and placeholder files only — no implementation code.

**Scope:**
- All app/, tests/, docs/, infra/, scripts/, .github/workflows/ directories with .gitkeep
- Minimal pyproject.toml
- Makefile with stub targets: run, test, lint, eval, cost-report, demo
- This prompt-log entry

**Outcome:** Repository scaffold created. All acceptance criteria met.

---

## task-002 — Architecture docs and ADRs

**Date:** 2026-05-20
**Branch:** feature/task-002-architecture-docs
**Phase:** 0 — Discovery and Architecture

**Objective:** Write all Phase 0 architecture documentation. No code. Docs only.

**Scope:**
- `docs/architecture.md` — system overview, ASCII component diagram, data flow, tech stack, design principles
- `docs/adr/000-adr-template.md` — standard ADR template
- `docs/adr/001-framework-fastapi.md` — FastAPI over Flask/Django/Litestar
- `docs/adr/002-llm-client-abstraction.md` — LLM client protocol pattern
- `docs/adr/003-vector-store-chromadb.md` — ChromaDB for dev; pgvector migration path
- `docs/adr/004-memory-strategy.md` — sliding window + summarization
- `docs/adr/005-evaluation-strategy.md` — LLM-as-judge with Haiku
- `README.md` — project overview presentable to hiring managers
- This prompt-log entry

**Outcome:** All 8 documentation files created. Every ADR has all template sections filled. README links to architecture and ADR index. No code created.
