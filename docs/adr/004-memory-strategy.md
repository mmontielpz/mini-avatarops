# ADR-004 — Use sliding window + summarization for conversation memory

**Status:** Accepted
**Date:** 2026-05-20
**Deciders:** Miguel Lopez

---

## Context

An avatar chat session accumulates turn history. LLMs have a finite context
window (200k tokens for Claude Sonnet, but we pay per token), so we cannot
pass the full history on every turn. We must decide how to manage what history
the model sees and how to persist memory between sessions.

Three broad strategies exist:
1. **Sliding window** — keep the N most recent turns verbatim.
2. **Summarization** — periodically compress older turns into a prose summary;
   prepend that summary to the recent window.
3. **External memory store** — embed each turn, retrieve the top-k most
   semantically relevant turns at inference time (similar to RAG).

## Decision

We will use **sliding window (last N turns) with periodic summarization** as
the primary memory strategy.

Concretely:
- Keep the last 20 turns verbatim in the context window.
- When the window exceeds 20 turns, call a cheap summarization pass (Haiku)
  to compress the oldest 10 turns into a prose summary.
- Prepend the running summary to the system prompt on every request.
- Persist summary + raw turns in SQLite so sessions survive server restarts.

External (vector) memory retrieval is deferred to a later phase.

## Alternatives Considered

| Option | Pros | Cons | Ruled out because |
|--------|------|------|-------------------|
| Full history (no truncation) | Perfect recall | Cost grows unboundedly; latency increases | Not viable at scale |
| Sliding window only (no summary) | Simple; deterministic | Old context is silently dropped; avatar "forgets" established facts | Persona consistency breaks over long sessions |
| External memory (vector retrieval) | Scales to very long sessions; theoretically best recall | Complex; requires embedding every turn; retrieval quality depends on query formulation | Higher complexity than Phase 0 justifies; defer to Phase 2 |
| Full summarization (no raw turns) | Compact | Lossy; recent turns may be misrepresented; harder to debug | Recent turn verbatim is important for coherent follow-up |

## Consequences

**Positive:**
- Predictable cost: max context = summary tokens + 20 × avg_turn_tokens.
- Summarization is done by Haiku (cheap); the chat model sees a clean, compact history.
- SQLite persistence means sessions survive pod restarts — important for the demo.

**Negative / trade-offs:**
- Summarization is lossy and can introduce hallucinated "memories."
- Summary quality depends on the Haiku prompt; this prompt must be versioned and evaluated like any other.
- 20-turn window is a guess; it should be tunable via config, not hardcoded.

**Neutral / follow-on work:**
- The summarization prompt goes in `app/prompts/v1/summarize_history.md`.
- A compression trigger threshold (20 turns, or N tokens) should be a config value.
- Phase 2: layer in vector retrieval of older turns on top of this baseline.
