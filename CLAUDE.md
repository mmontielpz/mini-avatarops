# Mini AvatarOps — Claude Code Operating Manual

## Project Purpose
Production-grade AI avatar chat backend for portfolio demonstration.
Demonstrates: RAG, memory, prompt engineering, evaluation, observability, cost tracking, CI/CD.

## Architecture Principles
- Backend-first, API-driven (FastAPI monolith)
- Vertical slices with full production discipline at each phase
- Prompts are versioned artifacts, not strings
- Evaluation is a first-class concern, not an afterthought
- Every LLM call is traced, costed, and logged

## Operating Rules for Claude Code
1. One task per session — do not expand scope beyond the current task
2. Never modify files outside the task scope without flagging it first
3. Every task must pass: `make lint && make test` before declaring done
4. If an architectural decision is required, draft an ADR before writing code
5. Every completed task must output a standard summary (see below)
6. Use feature branches — never commit directly to main or develop
7. Commit messages follow Conventional Commits: feat:, fix:, test:, docs:, chore:, eval:
8. Never hardcode secrets or API keys

## Standard Task Summary Format
At the end of every task, output:
- Files changed
- Tests added (with names)
- Commands run and results
- Risks introduced
- Deferred work
- Next recommended task

## Current Phase
Phase 0 — Discovery and Architecture

## Tech Stack
- Python 3.11+
- FastAPI
- Pydantic v2
- structlog
- pytest
- SQLite (dev) → Postgres path
- ChromaDB (vector store — TBD, see ADR-003)
- Anthropic API (claude-sonnet-4-20250514 for system, haiku for eval judges)
- Docker + Docker Compose
- GitHub Actions

## Branching Strategy
main ← protected
develop ← integration branch
feature/*, fix/*, chore/*, eval/*, docs/* ← working branches

## Key Directories (to be created)
app/, docs/adr/, tests/, infra/, scripts/, .github/workflows/

## Prompt Log
All tasks are logged in docs/prompt-log.md
