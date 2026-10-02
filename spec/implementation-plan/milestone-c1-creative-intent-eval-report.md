# Milestone C1 — Real Creative Intent: Evaluation Report

**Date:** 2026-10-01
**Component:** Creative Intent (first real-AI vertical slice)
**Model:** `amazon.nova-lite-v1:0` (Amazon Bedrock, us-east-1) via Converse tool-use
**Prompt version:** `creative_intent/v1`  **Schema version:** `creative_intent.schema/1`
**Harness:** `evals/harness` Layer-0 deterministic + manual semantic review (no LLM judge yet)

---

## Fixture count & contract success

- **14 fixtures** across all 14 required categories (simple/detailed/vague text,
  tone/mood, pacing, subject/person/landscape emphasis, explicit + negative
  constraints, short/longer duration, equivalent-phrasing pair, conflicting/
  ambiguous).
- **Contract success: 14/14** — every output validated against the canonical
  `CreativeIntentContract` (`extra="forbid"`), both before and after the one
  prompt change. 0 schema failures, 0 provider failures during the runs.

## Latency distribution (real model, per resolution)

- min 580 ms · p50 ~724 ms · p90 ~973 ms · max ~1189 ms.
- Comfortably inside the ~1–3s interactive target (design §20) and far under the
  120s App Runner budget. Synchronous transport is appropriate; no async needed.

## Approximate usage / cost

- ~30K input + ~1K output tokens for the full 14-fixture run (~2.2K in / ~75 out
  per call). Nova Lite is Amazon's low-cost tier; a single resolution is a tiny
  fraction of a cent. Cost is a non-issue at this scale; re-measure if the prompt
  grows or the model changes.

## Observed failure categories

No contract failures. Semantic weaknesses found (and the general fix applied):
- **INTENT_INVENTED (over-filling)** — the biggest issue. On the vague
  "make something cool here", v1-initial fabricated `desired_feeling:["cool"]`,
  `emphasis:["cool"]`, `creative_constraints:["authentic"]`; and on "a short
  energetic video of Times Square" it invented `target_duration_seconds:30` +
  `target_format:landscape`. These are exactly the fabrications the fixtures
  guard against.
- **Minor over-read of pace** — "a quick 7-second clip" read "quick" as a tempo.

## Prompt change made (one general change, full-set rerun)

Strengthened the preserve-uncertainty section of `creative_intent/v1` with an
explicit "do NOT over-fill" rule: never echo a vague adjective as a creative tag;
never set `target_duration_seconds`/`target_format`/`pace` unless stated or
clearly implied; a minimal summary-only intent is the correct answer for a vague
request. **Not** tuned per-fixture — one general change, then the whole set rerun.

**Result after the change:**
- vague → `summary` + `raw_user_description` only (invented fields gone). ✅
- Times Square → fabricated duration/format gone; keeps energetic/fast. ✅
- 7-second clip → pace relaxed from `fast` to `moderate`; duration still 7s. ✅
- No regressions elsewhere; still 14/14 contract-valid.

## Representative GOOD outputs

- **Negative constraint** ("epic but not cheesy, not like an advertisement") →
  `desired_feeling:["epic"]`, `creative_constraints:["not cheesy","not like an
  advertisement"]` — preserved both negatives (the hard INTENT_MISREAD case).
- **Landscape/scale** ("make the mountains feel huge and us feel tiny") →
  `emphasis:["scale"]`, constraint preserves the scale contrast; no invented
  mountain/scene facts.
- **Conflicting intent** ("calm but also really energetic") → kept BOTH feelings
  rather than silently picking one — correct uncertainty handling.
- **Durations** — 7s and 30s extracted correctly only when stated.

## Representative WEAK outputs (remaining, acceptable for C1)

- "quick 7-second clip" still nudged `pace:moderate` — "quick" lightly read as
  tempo. Minor; a candidate for a future prompt tweak if evals show it matters.
- Nova occasionally duplicates a feeling into both `desired_feeling` and
  `creative_constraints` (e.g. "serene"/"movie-like"). Harmless redundancy, not a
  contract or correctness issue.

## Failures discovered (process)

- **Account-level provider gate (not a code bug):** Anthropic Claude models on
  this Bedrock account return `ResourceNotFoundException — use case details have
  not been submitted`. That form is a human-only, account-owner action. Switched
  the config default to `amazon.nova-lite-v1:0` (no form gate, on-demand
  invocable), which satisfies the design's "reliable structured output, low
  latency, low/moderate cost" bar. Config-driven, so switching to Claude later is
  a one-env-var change once the form is approved.
- **Claude requires an inference-profile id** (not the plain on-demand id) — noted
  in the gateway for when Anthropic is enabled.

## Deterministic (Layer-0) + unit/endpoint tests

- Full suite: **68 passed** (contracts, API, harness, new engine + endpoint tests).
- Engine unit tests (fake gateway): valid→canonical conversion, request carries
  task/prompt-version/schema, underspecified→minimal (no fabrication), invalid
  output→`StructuredOutputError` (not fabricated), provider error propagates.
- Endpoint tests: 200 canonical intent, 422 body validation, 503 on
  structured-output + provider failures (designed failure path, never a fabricated
  intent).
- Live local HTTP check: `POST /v1/creative-intent` → real Nova → canonical intent,
  with the structured observability log (correlation_id, task, prompt/schema
  version, latency_ms, success; no raw user content).
