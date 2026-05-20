# ADR-003 — Use ChromaDB for local dev; plan migration path to pgvector

**Status:** Accepted
**Date:** 2026-05-20
**Deciders:** Miguel Lopez

---

## Context

Mini AvatarOps implements Retrieval-Augmented Generation (RAG): user messages
are matched against a knowledge base of avatar persona documents, and the
top-k results are injected into the system prompt. This requires a vector
store capable of approximate nearest-neighbor search over dense embeddings.

Constraints:
- Local dev must work with zero external services (single `docker compose up`).
- Production must be viable on a small cloud instance (cost-sensitive portfolio demo).
- The abstraction story from ADR-002 applies here too: service code must not
  be coupled to a specific vector store SDK.

## Decision

We will use **ChromaDB** for local development and the initial deployed demo.

ChromaDB runs in-process (no separate server required in dev), supports
persistent SQLite-backed storage, and has a clean Python client. It is
sufficient for the scale of this portfolio project (< 10k documents).

We will wrap ChromaDB behind a `VectorStore` protocol (mirror of the
`LLMClient` pattern from ADR-002) so that the migration path to pgvector is a
single implementation swap.

## Alternatives Considered

| Option | Pros | Cons | Ruled out because |
|--------|------|------|-------------------|
| pgvector (Postgres extension) | Production-grade; same DB as relational data; no extra service | Requires Postgres in dev; more complex local setup | Overkill for Phase 0; adds infra complexity before the feature is built |
| Pinecone / Weaviate (managed) | Fully managed; no ops | External paid dependency; not reproducible locally without credentials | Portfolio demo must run fully offline |
| Qdrant | Fast; good Python client | Separate Docker service required even in dev | Adds compose complexity before we need the performance |
| FAISS | Extremely fast; battle-tested | No persistence layer; no metadata filtering; low-level API | Missing features we need (metadata, persistence) without adding a wrapper layer anyway |

## Migration Path to pgvector

When the project graduates beyond portfolio demo scale:

1. Add `pgvector` extension to the Postgres Compose service.
2. Implement `PgVectorStore` conforming to the `VectorStore` protocol.
3. Write a one-shot migration script in `scripts/` to re-embed and load documents.
4. Swap the DI binding. No service code changes required.

## Consequences

**Positive:**
- Zero-dependency local dev: `chroma.PersistentClient(path="./data/chroma")` just works.
- The `VectorStore` protocol enforced now means pgvector migration is low-risk.
- ChromaDB supports metadata filtering, which we will use to scope retrieval by avatar persona.

**Negative / trade-offs:**
- ChromaDB is not production-hardened at scale; we are knowingly accepting this for Phase 0.
- Embedding model choice is deferred; ChromaDB defaults to `all-MiniLM-L6-v2` via sentence-transformers, which adds a heavyweight dependency. We may call the Anthropic embeddings endpoint instead to avoid it.

**Neutral / follow-on work:**
- A `VectorStore` protocol and ChromaDB implementation are Phase 1 deliverables.
- Embedding model selection should be its own ADR before Phase 1 implementation begins.
