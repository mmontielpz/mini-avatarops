# Mini AvatarOps — System Architecture

## Purpose

Mini AvatarOps is a production-grade AI avatar chat backend. A client sends a
message to an avatar; the system retrieves relevant persona documents, injects
them with conversation history into a prompt, calls a large language model, and
returns a response. Every call is traced, costed, and logged. Evaluation runs
automatically in CI.

---

## Component Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Clients                                    │
│          (React/Next.js frontend, curl, eval harness)               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │  HTTPS / JSON
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        FastAPI App                                  │
│                                                                     │
│  ┌──────────────┐   ┌───────────────┐   ┌──────────────────────┐  │
│  │  /api/chat   │   │  /api/memory  │   │  /api/eval  (admin)  │  │
│  └──────┬───────┘   └──────┬────────┘   └──────────┬───────────┘  │
│         │                  │                        │               │
│         └──────────────────┼────────────────────────┘               │
│                            │                                        │
│                            ▼                                        │
│              ┌─────────────────────────┐                           │
│              │      Chat Service       │                           │
│              │  (orchestrates a turn)  │                           │
│              └────┬──────────┬─────────┘                           │
│                   │          │                                      │
│          ┌────────┘          └────────────┐                        │
│          ▼                               ▼                         │
│  ┌───────────────┐            ┌────────────────────┐               │
│  │  RAG Service  │            │  Memory Service    │               │
│  │  (retrieve    │            │  (sliding window + │               │
│  │   documents)  │            │   summarization)   │               │
│  └──────┬────────┘            └────────┬───────────┘               │
│         │                              │                            │
│  ┌──────┴────────┐            ┌────────┴───────────┐               │
│  │  VectorStore  │            │   SQLite / Postgres │              │
│  │  (ChromaDB)   │            │   (sessions, turns) │              │
│  └───────────────┘            └────────────────────┘               │
│                                                                     │
│              ┌─────────────────────────┐                           │
│              │      LLM Client         │                           │
│              │  (AnthropicClient impl) │                           │
│              └─────────────┬───────────┘                           │
│                            │                                        │
└────────────────────────────┼────────────────────────────────────────┘
                             │  HTTPS
                             ▼
              ┌──────────────────────────┐
              │    Anthropic API         │
              │  Sonnet (chat)           │
              │  Haiku  (summarize/eval) │
              └──────────────────────────┘

              ┌──────────────────────────┐
              │  Observability           │
              │  structlog → stdout      │
              │  JSON trace per LLM call │
              │  (cost, latency, tokens) │
              └──────────────────────────┘
```

---

## Data Flow — Single Chat Turn

1. **Client** `POST /api/chat` with `{avatar_id, session_id, message}`.
2. **Chat Service** loads the session's memory from SQLite (summary + last N turns).
3. **RAG Service** embeds the user message and queries ChromaDB for the top-k
   persona documents matching the avatar.
4. **Chat Service** assembles the prompt: system prompt template + persona
   documents + memory summary + raw recent turns + user message.
5. **LLM Client** calls Anthropic Sonnet. The call is wrapped in an
   observability decorator that logs tokens, cost, and latency.
6. The response is appended to the session in SQLite.
7. If the session now exceeds the sliding-window threshold, the Memory Service
   calls Haiku to compress the oldest turns into a summary.
8. The response text is returned to the client.

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| HTTP framework | FastAPI | Async, Pydantic v2 schema enforcement, auto OpenAPI |
| Data validation | Pydantic v2 | Request/response models and internal domain types |
| LLM provider | Anthropic API | Sonnet for chat; Haiku for summarization and eval judges |
| Vector store | ChromaDB | Local dev; migration path to pgvector (see ADR-003) |
| Relational DB | SQLite → Postgres | SQLite in dev; same schema, Postgres in prod |
| Logging | structlog | Structured JSON logs; one trace record per LLM call |
| Testing | pytest + pytest-asyncio | Unit tests use FakeLLMClient; integration tests hit SQLite |
| Evaluation | Custom LLM-as-judge | Haiku judge; datasets in `app/eval/datasets/` |
| Containerization | Docker + Docker Compose | Single `compose up` starts full local stack |
| CI/CD | GitHub Actions | Lint → test → eval on every PR |

---

## Key Design Principles

### 1. Every LLM call is an observable event
Each call through `LLMClient.complete()` emits a structured log record
containing: prompt version, model, input tokens, output tokens, estimated cost,
and wall-clock latency. No call is invisible.

### 2. Prompts are versioned artifacts
Prompt templates live in `app/prompts/v1/` as Markdown files with YAML
frontmatter. They are loaded at startup, not interpolated inline in code.
Changing a prompt is a file change, which means it shows up in git diff,
triggers eval in CI, and can be rolled back independently of code.

### 3. Interfaces before implementations
The `LLMClient` protocol and `VectorStore` protocol are defined before any
concrete implementation is written. All service code depends on the protocol.
This makes every layer unit-testable without network calls and makes provider
swaps a single-file change.

### 4. Evaluation is a first-class concern
The eval harness is not an afterthought. It runs in CI, its judge prompt is
versioned, and its threshold gates merges. Degrading response quality is a
build failure.

### 5. Vertical slices, not horizontal layers
Each feature (chat, memory, RAG, eval) is a self-contained vertical slice with
its own service, repository, and tests. This allows each slice to be developed
and deployed independently, and prevents cross-feature coupling through a
shared "utils" layer.

---

## Directory Map

```
app/
  api/              HTTP route handlers
  core/             Protocols, config, base types
  services/         Business logic (chat, memory, rag, llm)
  agents/           Multi-step agent orchestration (Phase 2+)
  agents/tools/     Tool implementations for agents
  prompts/v1/       Versioned prompt templates (Markdown)
  models/           Pydantic domain models
  db/               SQLAlchemy models + migrations
  db/repositories/  Data access layer
  vector_store/     VectorStore protocol + ChromaDB impl
  eval/             Evaluation harness
  eval/judges/      Judge prompt runners
  eval/datasets/    JSONL evaluation datasets
  eval/reports/     CI-generated score reports
  observability/    Structlog config, cost tracker, trace emitter

tests/
  unit/             No network, no disk (FakeLLMClient, in-memory SQLite)
  integration/      Real SQLite, real ChromaDB, FakeLLMClient
  e2e/              Full stack, real Anthropic API (manual/nightly only)

docs/
  architecture.md   (this file)
  adr/              Architectural Decision Records
  prompt-log.md     Task-by-task prompt and outcome log

infra/              Docker Compose, Dockerfile, future Terraform
scripts/            One-shot maintenance and migration scripts
.github/workflows/  CI pipeline definitions
```
