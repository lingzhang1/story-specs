# B0 / P0 Beta Security & Cost — Acceptance Evidence & Enforcement Proposal

Status: **MEASURED EVIDENCE — observe mode, nothing enforced. STOP before any enforcement flip.**
Date: 2026-10-22. Account: beta `224033508028`. Deployed API digest `sha256:ceb8787c…` (observe mode).
Scope: Phase 3 throttle + Phase 4 quota acceptance on real beta traffic + bounded synthetic tests. No
gamma/prod, no billing, no R7, no editing changes.

## 1. End-to-end path trace (real beta user `61310227-…`, period 2026-10)

`request → auth(principal) → throttle(ThrottleAllowed) → quota reserve(res#<rid>) → job + active-job slot →
Bedrock physical attempts → settle(actual USD) / release → client result`. Every stage is present and
reconciles in the live ledger + CloudWatch metrics below.

## 2. EMF metrics in CloudWatch — VERIFIED (namespace `Story/Beta`)

Present + populated: `ThrottleAllowed`, `QuotaReserved`, `QuotaSettled`, `GenerationStarted/Completed/Failed`,
`ModelInvocations`, `InputTokens`, `OutputTokens`, `EstimatedModelCostUSD`, `ModelFallbacks`,
`MissingModelPricing`, `QueueWaitMs`, `WorkerProcessingMs`.
- Throttle/quota metrics carry ONLY `op_type=<class>` — **no user id** (low-cardinality confirmed, req 10).
- **No `WouldThrottle` / `QuotaWouldDeny` metrics exist** — at the observed volume nothing ever approached a
  limit. This is the central threshold finding: current limits have large headroom for beta.

Measured over 48h, by class:

| class | ThrottleAllowed | QuotaReserved | QuotaSettled |
|---|---|---|---|
| ai_sync | 18 | 18 | 18 |
| ai_job_submit | 7 | 7 | (worker-settled) |
| music_provider | 8 | 8 | 8 |
| read | 11 | — (not gated) | — |
| writes | 6 | — (not gated) | — |

## 3. Idempotency + active-job slots return to baseline — VERIFIED

Live `story-beta-user-usage` ledger (user `61310227-…`, period 2026-10):
- **33 reservations** = **32 settled + 1 released**; reserve↔settle counts match exactly (ai_sync 18=18,
  music 8=8) → idempotent, no double-count.
- **7 active-job slot markers, ALL `released`**; **`active_jobs = 0`** → slots returned to baseline.
- **`reserved_usd = 0.0000`** → no leaked/stuck reservation.

## 4. Bedrock token/cost ↔ quota ledger reconciliation — VERIFIED

- Ledger `settled_usd = $3.9555` (ACTUAL incurred) vs sum of reservation ESTIMATES `$1.88` — differ BY DESIGN:
  settle records the real Bedrock cost, which exceeds the conservative reserve estimate (correction 8,
  "financial truth wins"). settled ≥ estimate as intended.
- Beta revise edit-plans (worker, opus-4-6): **80,615 input + 4,880 output tokens / 3 invocations =
  `$1.575225` EstimatedModelCostUSD**. The remainder of the $3.9555 is sync ai_sync (creative-intent, guide,
  director, understanding, duration) + coverage jobs + music_provider.
- `ModelFallbacks = 0` on the measured revise path (primary opus-4-6 served all calls).

## 5. Global + per-user aggregates — VERIFIED

`__global__` item mirrors the user exactly: `settled_usd = $3.9555 == $3.9555`; 33 `gres#` markers
(32 settled + 1 released) == 33 `res#`. Global = Σ per-user. Correct.

## 6. PRICE PER USER PER COMPONENT (observed beta)

Actual cost basis: **$3.9555 total / 32 generations = ~$0.1236 actual per generation** for user `61310227`.
Reserve-time ESTIMATES per component (conservative headroom; actual settled afterward):

| Component (class) | Reserve estimate / call | Observed actual basis | Notes |
|---|---|---|---|
| Sync Bedrock inference (`ai_sync`) | **$0.05** | dominant share of the $3.9555 | creative-intent, interpret-revision, scene/asset-understanding, story-coverage(+v2), story-guide, edit-plan(+mm), music-plan, duration, director/* |
| Async AI job submit (`ai_job_submit`) | **$0.15** | revise edit-plan ≈ **$0.525/call** actual (3 calls = $1.575 / 3) | multi-call plan + critique + repair; opus-4-6; settled by the worker |
| Paid music provider (`music_provider`) | **$0.01** | small fixed | Epidemic candidates/search/analysis/soundtrack (provider call, not Bedrock) |
| Metadata writes (`writes`) | $0 | $0 | not quota-gated |
| Reads / polling (`read`) | $0 | $0 | not quota-gated |

Per-user monthly projection at observed intensity (32 gens in the test window): well under $5/user/month.
A heavy beta user doing ~65 generations/month ≈ $8 actual — the basis for the proposed $8 cap.

## 7. Proposed FINAL thresholds (for review; not applied)

Throttle (per-minute, per class):

| class | current | proposed enforce |
|---|---|---|
| ai_sync | 20 | **20** (keep) |
| ai_job_submit | 10 | **10** (keep) |
| music_provider | 20 | **15** (tighten) |
| writes | 60 | **60** (keep) |
| presign | 30 | **30** (keep) |
| read | 120 | **120** (keep) |

Quota:

| control | current | proposed enforce |
|---|---|---|
| per-user monthly USD | $5 | **$8** (~65 gens/mo headroom) |
| per-user daily gens | 40 | **40** (keep) |
| max active jobs | 5 | **5** (keep) |
| global monthly USD | $60 | **$60** (keep — below the $100 budget ALERT) |

## 8. No paid-endpoint bypass — VERIFIED

Every expensive class (`ai_sync`, `music_provider`, `ai_job_submit`) is quota-gated from the single shared
`throttle.ROUTE_CLASS`; a route-coverage test fails CI if a new paid endpoint is left ungated. Live metrics
show reservations ONLY on those classes, none on read/writes.

## 9. Boundary / failure behavior — bounded synthetic tests (no Bedrock load)

29 quota tests + jobstore sweep tests: concurrent reserve-once, per-user + global boundary denials,
transaction-failure recovery, observe-never-denies, settle/release idempotency, slot cap + exactly-once,
orphan recovery, store-error fail-closed. No expensive Bedrock calls used.

## 10. Capacity (confirm before recommending more)

- Worker Lambda: **no reserved concurrency** (unreserved pool, as deployed).
- Account Lambda concurrent-executions quota (L-B99A9384) = **1000** (increase request **CASE_CLOSED/granted**)
  — the fresh-account limit of 10 is resolved.
- Bedrock per-day model-invocation token caps are **non-adjustable** (prior finding) — capacity is a
  model-choice lever (Sonnet has a far larger daily budget than Opus), not a Service-Quotas bump.
- **No capacity increase needed for beta.**

## 11. WAF Count review (72h)

**296 AllowedRequests · 76 CountedRequests · 0 BlockedRequests.** All 76 counts attribute to the
**CommonRuleSet (CRS)** rule; IpReputation / KnownBadInputs / RateLimitPerIp / BodySizeOver2MB counted ZERO.
Given all traffic was first-party app testing, the 76 CRS counts are near-certainly **false positives on
large AI JSON POST bodies** — the exact risk Phase 2 anticipated by launching CRS in Count. (Naming the exact
CRS sub-rule needs WAF full request logging enabled; the metric is conclusive it is CRS-only.)

## 12. Enforcement proposal + rollback

| Flip | Recommendation | Evidence | Rollback |
|---|---|---|---|
| `STORY_THROTTLE_MODE=enforce` | **GO** (low risk) | 0 WouldThrottle; rates far below limits | env → `observe`, redeploy (digest unchanged) |
| `STORY_QUOTA_MODE=enforce` (+ user cap $5→$8) | **GO** | 0 QuotaWouldDeny; $0.12/gen; slots clean; ledger reconciles | env → `observe`, redeploy |
| WAF **rate rule** Count→Block | **GO** | 0 rate-limit counts in 72h | set rule back to Count / disassociate |
| WAF **CRS** Count→Block | **HOLD** | 76 CRS false-positive counts on legit AI bodies | keep Count; exclude/tune the offending CRS sub-rule, re-observe |

## Separate status

- **Epidemic key rotation:** see the rotation runbook — the old key leaked in a pasted terminal output and
  must be rotated in the Epidemic dashboard, then the new value written to the beta Secrets Manager secret
  (`epidemic-mcp-api-key`). The console/dashboard step is the owner's.
- **SES production access:** `ProductionAccessEnabled=false` (sandbox, 200/day). Request submitted; case reply
  to be sent.
- **Bedrock SCP read exceptions:** LIVE in SCP `p-njbqhrap` (ListFoundationModels, GetFoundationModel,
  ListInferenceProfiles, GetUseCaseForModelAccess, etc. all in NotAction). Only the CDK-source backport on
  `feat/r1` remains (live is already correct).

## Appendix — "Create Another Version" infeasible (job `3cff789c…`, root-caused 2026-10-22)

NOT a defect. The job reached the planner (3 Opus calls, ~3 min, 2 repair passes) and the validator rejected
the final plan: `terminal=invalid_after_repair`, `final_error_codes=["directive_order_violated","other"]`,
`directive_categories=["ending","must_include","opening","order_after"]`, `usable_moment_count=18`,
`target_seconds=47.5`. The another-version request carried FOUR hard directives (folded in via the Customize
flow, `directedByCustomize=true`): pinned opening + pinned ending + must-include set + an order_after ordering
constraint. These over-constrain a 47.5s cut over 18 moments — no valid plan satisfies all four, so after
repair it is honestly infeasible and the app correctly routes to "Adjust instructions." Remedy: loosen one
constraint (drop order_after, or unpin opening/ending, or fewer must-include), or use a plain Create Another
Version with no directives. A separate NON-FATAL bug was fixed while investigating: the orphan-reservation
sweep's Scan used the DynamoDB reserved keyword `owner` without an alias (ValidationException, caught/logged,
sweep no-opped) — fixed with an ExpressionAttributeNames alias.
