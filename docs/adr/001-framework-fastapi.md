# ADR-001 — Use FastAPI as the HTTP framework

**Status:** Accepted
**Date:** 2026-05-20
**Deciders:** Miguel Lopez

---

## Context

Mini AvatarOps is a backend API service that exposes chat, memory, and
evaluation endpoints. The primary users are a React/Next.js frontend and
automated evaluation harnesses. The service must support async I/O because
every request touches at least one external network call (Anthropic API,
ChromaDB, SQLite). The project is a portfolio piece, so the framework choice
is also a signal about engineering judgment.

Candidates evaluated: FastAPI, Flask, Django REST Framework, Litestar.

## Decision

We will use **FastAPI** as the HTTP framework.

FastAPI is async-native, enforces request/response schemas via Pydantic v2
(which we already require for data modeling), generates an OpenAPI spec
automatically, and has a large, stable ecosystem. Its explicit dependency
injection system maps cleanly onto the vertical-slice architecture we have
chosen, making it easy to swap implementations (e.g., the LLM client) without
touching route handlers.

## Alternatives Considered

| Option | Pros | Cons | Ruled out because |
|--------|------|------|-------------------|
| Flask | Minimal, widely known | Sync by default; no schema enforcement; verbose DI | Async story is a bolt-on; Pydantic integration is manual |
| Django REST Framework | Batteries included; ORM; admin | Heavyweight; sync-first; large learning surface | Overkill for a monolith that owns its own schema layer |
| Litestar | Async-native; strong typing; OpenAPI | Smaller ecosystem; less hiring-signal familiarity | Ecosystem risk for a portfolio project that must be read by unfamiliar reviewers |

## Consequences

**Positive:**
- Request validation is free — Pydantic models double as API contracts.
- OpenAPI/Swagger UI is auto-generated, providing live documentation with no extra work.
- Async handlers compose naturally with `httpx` (Anthropic SDK) and ChromaDB async client.
- Aligns with the current industry hiring signal for Python API services.

**Negative / trade-offs:**
- FastAPI's dependency injection can feel implicit to readers not already familiar with it; requires documentation discipline.
- No built-in ORM; we must wire SQLAlchemy or raw SQLite ourselves.

**Neutral / follow-on work:**
- ADR-002 captures the API design conventions that sit on top of this choice.
- A `lifespan` context manager will handle startup/shutdown (DB pool, vector store client).
