# ADR-002 — Abstract the LLM client behind an interface

**Status:** Accepted
**Date:** 2026-05-20
**Deciders:** Miguel Lopez

---

## Context

Every chat and evaluation feature in Mini AvatarOps calls a large language
model. The initial target is Anthropic Claude (Sonnet for production calls,
Haiku for evaluation judges). However:

1. The Anthropic SDK API surface changes between major versions.
2. Automated tests must not make real network calls — they need a fake that
   returns deterministic responses.
3. A future phase may want to try an OpenAI or local (Ollama) model to
   benchmark quality vs. cost without rewriting every call site.

Calling `anthropic.Anthropic().messages.create(...)` directly in service
code couples business logic to a specific SDK and makes all three problems
harder to solve.

## Decision

We will define a **`LLMClient` abstract base class** (or `Protocol`) in
`app/core/llm_client.py` with a single async method:

```python
async def complete(self, request: LLMRequest) -> LLMResponse: ...
```

All service code calls this interface. The concrete `AnthropicClient`
implementation lives in `app/services/llm/anthropic_client.py` and is
injected via FastAPI's dependency injection. Tests inject a `FakeLLMClient`
that returns canned responses.

## Alternatives Considered

| Option | Pros | Cons | Ruled out because |
|--------|------|------|-------------------|
| Call Anthropic SDK directly | Zero boilerplate | Tight coupling; untestable without network; hard to swap | Violates the "testable at every layer" principle |
| Use LangChain / LiteLLM as the abstraction | Multi-provider out of the box | Heavy dependency; opaque internals; version churn; evaluation becomes a black box | Adds complexity we don't control; evaluation discipline requires knowing exactly what is sent |
| Abstract at the HTTP level (raw `httpx`) | Minimal dependency | Reimplements retry, streaming, token counting | More work with no benefit over a thin wrapper |

## Consequences

**Positive:**
- Unit tests for all service logic run without network calls and in milliseconds.
- Swapping Anthropic for another provider is a one-file change plus a DI re-wire.
- `LLMRequest` / `LLMResponse` types give us a canonical place to attach tracing, cost metadata, and prompt version tags.

**Negative / trade-offs:**
- One extra layer of indirection. New contributors must learn the interface before they can trace a call end-to-end.
- Streaming responses require extending the interface; the initial version will buffer the full response.

**Neutral / follow-on work:**
- `LLMRequest` must carry a `prompt_version` field to support the prompt versioning strategy (see `app/prompts/`).
- The observability layer (ADR or task TBD) will hook into `LLMResponse` to emit cost and latency metrics.
