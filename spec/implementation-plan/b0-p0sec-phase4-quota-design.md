# B0 / P0 Beta Security — Phase 4: AI Quota & Spending Protection (DESIGN)

Status: **DESIGN — awaiting approval. Nothing provisioned.** Reuses existing UserUsage + Bedrock cost
accounting. Out of scope: subscriptions, paid billing, free-tier product pricing (req 14).

## 1. Reuse of existing mechanisms (req 10)

| Existing | Reuse in Phase 4 |
|---|---|
| **UserUsage table** `story-<env>-user-usage` (PK `story_user_id`, SK `quota_period` e.g. "2026-10"/"intro", PAY_PER_REQUEST, PITR, RETAIN) | The durable per-user allowance ledger. Phase 4 writes reservation + settlement items here. Distinct from the ephemeral Phase-3 rate-limit table. |
| **Bedrock cost accounting** (`packages/observability` `estimated_cost_usd`, worker EMF `estimated_cost_usd`, `last_max_tokens_used`) | The per-generation USD estimate that each settlement records against the user's monthly spend. |
| **Job idempotency** `job_id = UUIDv5(operation_id)` + lease fencing | The idempotency key for reserve/settle so a retried submit neither double-reserves nor double-charges. |
| **Phase-3 throttle** (rate, per minute) | Orthogonal: throttle limits REQUEST RATE; quota limits CUMULATIVE AI USAGE/COST per period. Both key on story_user_id. |

## 2. Atomic usage reservation + idempotent settlement (req 11)

Two-phase, reservation-first (never charge after the fact only):

1. **RESERVE (before inference).** On an AI submit/call, atomically reserve an estimated cost/unit against the
   user's period budget: a conditional `UpdateItem` on `(story_user_id, quota_period)` — `ADD reserved :est`
   with `ConditionExpression reserved + settled + :est <= :budget`. Fail → **402-style quota-denied** (we use
   429 or a dedicated quota error, see §6), BEFORE any Bedrock call, so an over-budget user never spends.
   The reservation carries a `reservation_id = UUIDv5(operation_id)` so a retry of the SAME operation finds
   its existing reservation and does NOT reserve again (idempotent, req 11).
2. **SETTLE (after inference).** When the real token usage/cost is known, atomically move the amount from
   `reserved` to `settled` with the ACTUAL `estimated_cost_usd`: `UpdateItem ADD settled :actual, reserved
   :-est` conditioned on the reservation existing and not already settled (`attribute_exists(res#<id>) AND
   res#<id>.state = 'reserved'`). Settling twice is a no-op (the condition fails) — **idempotent settlement**.
3. **RELEASE on terminal failure.** If inference fails/cancels, release the reservation (`ADD reserved :-est`)
   so a failed job does not consume allowance. Tied to the job's terminal-state transition (req 12).

Honesty note (req 11, cross-store): the reservation ledger (UserUsage) and the job store are SEPARATE
DynamoDB tables. We do NOT claim a single atomic transaction across both. Instead: reserve in UserUsage
FIRST (atomic within that item), then create the job; on job-create failure, release the reservation. The
reservation_id = UUIDv5(operation_id) makes the whole flow idempotent and self-healing (a crashed flow
re-runs to the same reservation). A background reconcile sweeps orphaned reservations (reserved but no
live job past a TTL) back to available — the crash-recovery path (req 12).

## 3. Active-job concurrency + crash recovery (req 12) — the deferred Phase-3 item

- **Atomic active-job count:** a counter on `(story_user_id, "active_jobs")` incremented on job submit
  (`ADD active :1` conditioned `active < :max_concurrent`) and decremented on terminal state. Over the cap →
  denied at submit. This is the "5 active jobs" idea from Phase 3, now with real reserve/release semantics.
- **Terminal-state release:** the worker decrements `active` and settles/releases the reservation when a job
  reaches success/failure/cancelled — exactly once, guarded by the job's existing lease + a `settled` flag on
  the job row so a redelivery does not double-decrement.
- **Crash recovery:** the existing reconcile EventBridge rule (already in the api stack) is extended to also
  release reservations + decrement active for jobs stuck past their lease with no progress — so a crashed
  worker/instance cannot permanently leak a reservation or an active-job slot.

## 4. Configurable beta allowances + global safeguards (req 13)

All env-configurable, NOT product pricing (req 14) — operational beta guardrails:

| Control | Proposed beta default | Env |
|---|---|---|
| Per-user monthly AI spend cap | **$5 / user / month** (estimated USD) | `STORY_QUOTA_USER_MONTHLY_USD` |
| Per-user daily AI generations | **40 / day** | `STORY_QUOTA_USER_DAILY_GENS` |
| Max active AI jobs / user | **5** | `STORY_QUOTA_MAX_ACTIVE_JOBS` |
| **Global** monthly Bedrock spend cap (all users) | **$200 / month** (circuit breaker) | `STORY_QUOTA_GLOBAL_MONTHLY_USD` |
| Global concurrent Bedrock invocations | **cap** (protect account-level Bedrock quota) | `STORY_QUOTA_GLOBAL_CONCURRENCY` |

- **Global cost safeguard:** a global counter item (`PK="__global__"`) tracks month-to-date estimated spend;
  when it crosses `STORY_QUOTA_GLOBAL_MONTHLY_USD`, new AI reservations are denied (circuit breaker) with a
  clear error + a `GlobalQuotaTripped` metric/alarm. Protects against a runaway cost event across ALL users,
  independent of per-user caps.
- Thresholds are starting points for observation (like Phase 3), tuned from real beta data before enforcing.

## 5. Observation vs enforcement

Mirror Phase 3: `STORY_QUOTA_MODE=observe|enforce`. Observe emits `QuotaWouldDeny` + usage metrics without
rejecting; enforce denies over-budget. Default `observe` for the beta rollout.

## 6. Denial response + metrics

- Quota denial: a distinct **429** (or 402-style) with `{"error":"quota_exceeded","scope":"user_monthly|user_daily|active_jobs|global","retry_after_seconds":N}`. Distinct from a rate-limit 429 (`error:"rate_limited"`) so clients and logs can tell them apart.
- Low-cardinality metrics (reuse Observability EMF, `op_type` dim): `QuotaReserved`, `QuotaSettled`,
  `QuotaDenied`, `QuotaWouldDeny`, `QuotaReleased`, `GlobalQuotaTripped`, `QuotaStoreError`. NEVER dimensioned
  by story_user_id. Per-user spend is reconstructable from the existing per-generation cost LOG (not a metric
  dimension), consistent with the observability design.
- Store-failure policy (consistent with Phase 3 amendment 1): a UserUsage outage on an EXPENSIVE op fails
  CLOSED (503) — we must not run unbounded inference when we cannot account for it; safe ops unaffected.

## 7. CDK / IAM / cost (to author on approval)

- No new table (reuse `story-<env>-user-usage`); add a TTL only to ephemeral reservation sub-items if used.
- Instance + worker roles already have `dynamodb:*`-scoped grants to the usage table family; confirm
  `UpdateItem/GetItem` on the usage table ARN (likely already granted — verify, add if missing).
- Extend the reconcile rule's Lambda to the orphan-reservation sweep.
- Cost: a couple of extra DynamoDB UpdateItems per AI op + a few EMF metrics ≈ **< $1/month** beyond Phase 3.

## 8. Tests (to write with implementation)

Atomic reserve boundary (concurrent reserves respect the budget); idempotent settle (double-settle no-op);
release-on-failure; active-job cap under concurrency; crash-recovery sweep releases orphans; global circuit
breaker trips at the cap; observe-vs-enforce; store-failure fail-closed on expensive; user isolation; no
high-cardinality metric dimensions; retry does not double-reserve or double-charge.

## STOP — Phase 4 architecture & provisioning approval

Awaiting owner review of: the reserve/settle/release model, the explicit NON-atomic cross-store honesty note,
active-job concurrency + crash recovery, the proposed beta allowances + global circuit breaker, observe
default, and the costs. On approval: implement + tests + authoritative beta diff, STOP again before deploy.
No subscriptions/billing/free-tier pricing. No enforcement until separately approved.
