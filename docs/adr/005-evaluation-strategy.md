# ADR-005 — LLM-as-judge evaluation strategy

**Status:** Accepted
**Date:** 2026-05-20
**Deciders:** Miguel Lopez

---

## Context

Mini AvatarOps must demonstrate that changes to prompts, memory strategy, or
retrieval do not silently degrade response quality. Traditional unit tests
cannot assess whether a response is "in character" or "factually grounded."
Human evaluation does not scale for rapid iteration.

We need an automated evaluation framework that:
- Runs on every PR that touches prompts or retrieval logic.
- Produces a numeric score and a human-readable rationale.
- Is cheap enough to run in CI without significant cost.
- Is transparent enough that its own failures can be debugged.

## Decision

We will use **LLM-as-judge** evaluation with Claude Haiku as the judge model.

Each evaluation run:
1. Loads a dataset of (input, expected_behavior) pairs from `app/eval/datasets/`.
2. Calls the system under test (the full chat pipeline) to produce a response.
3. Passes `(input, response, rubric)` to a Haiku judge prompt.
4. The judge returns a score (1–5) and a one-sentence rationale.
5. Results are written to `app/eval/reports/` as JSON.
6. CI fails if mean score drops below a threshold (initial: 3.5 / 5).

Metrics evaluated per dataset:
- **Persona consistency** — does the response sound like the defined avatar?
- **Groundedness** — are factual claims supported by retrieved documents?
- **Refusal quality** — does the avatar decline out-of-scope requests gracefully?
- **Coherence** — is the response grammatically and logically sound?

## Alternatives Considered

| Option | Pros | Cons | Ruled out because |
|--------|------|------|-------------------|
| Human evaluation only | Gold standard | Doesn't scale; can't run in CI | Not automatable |
| Rule-based / regex checks | Deterministic; fast | Cannot assess quality, tone, or persona | Too brittle for natural language outputs |
| Embedding similarity (vs. gold response) | Cheap; fast | Penalizes valid paraphrases; rewards surface similarity | Poor signal for "is this in character?" |
| Opus as judge | Higher judgment quality | 15–20× more expensive than Haiku per call | Cost would make CI runs prohibitive |
| External eval frameworks (Ragas, DeepEval) | Feature-rich | Opaque internals; version churn; our rubrics are domain-specific | We need full control over judge prompts to evaluate them separately |

## Known Limitations

- LLM-as-judge has **positional bias** (prefers responses placed first) and
  **self-serving bias** (Claude may prefer Claude-style responses). We mitigate
  by: single-response scoring (not head-to-head), and using a different model
  family for the judge when possible.
- Judge scores are **not calibrated** across runs unless the judge prompt is
  frozen. The judge prompt is therefore a versioned artifact in `app/prompts/v1/`.
- A judge model update can shift scores without any change to the system under
  test. We pin the Haiku model version in config.

## Consequences

**Positive:**
- Evaluation runs fully automatically in CI; regressions are caught before merge.
- JSON reports are human-readable and link each score to a rationale string.
- Using Haiku keeps per-eval-run cost under $0.05 for a 50-example dataset.
- The evaluation harness is itself testable: we can write unit tests for the
  judge prompt using known-good and known-bad fixture responses.

**Negative / trade-offs:**
- Evaluation validity depends on dataset quality; garbage in, garbage out.
- Threshold (3.5/5) is initially arbitrary; it must be calibrated against a
  human-labeled baseline in Phase 1.

**Neutral / follow-on work:**
- Phase 1: build the `eval` CLI (`make eval`) and the first dataset for persona consistency.
- The judge prompt lives in `app/prompts/v1/judge_persona.md` and is versioned.
- A cost-tracking wrapper around the judge calls feeds into `make cost-report`.
