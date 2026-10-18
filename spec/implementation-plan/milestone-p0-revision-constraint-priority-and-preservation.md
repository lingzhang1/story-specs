# P0 — Revision Constraint Priority & Preservation-Safe Fallback

**Status:** IMPLEMENTED (backend + iOS) + automated tests, DEPLOYED to dev, awaiting device acceptance. STOP at device gate.
**Owner decision date:** 2026-10-18
**Branch:** `fix/p0-recover-jobs-explore-latency` (commit `e657a17`; nothing pushed/public)
**Frozen contracts:** `edit_plan/5.2` SYSTEM prompt + `EditPlanV2` output contract — **UNTOUCHED** (all changes additive, request-only).

## Problem

On device, a pure pacing revise ("shorten each clip to ~5s and keep the highlights") produced an
entirely **new clip set** instead of re-timing the selected version's clips. Two architectural root
causes, verified in code (job `d25b9f63`: `model_calls=1`, clips re-picked):

1. **AI-recommended duration was a hard validation constraint.** `validate_plan` checks total output
   against `creative_intent.target_duration_seconds` with an asymmetric -20%/+50% tolerance —
   identically for every operation, with no knowledge of whether the target was a user requirement or
   an AI recommendation, and no knowledge it was a timing-only revise. A correctly-shortened retime
   fell below the AI's recommended length -> rejected.
2. **Silent fallthrough to unrestricted replanning.** When the deterministic retime/structural fast
   path was rejected, the engine proceeded to `_call(_build_user(inputs))` — a full from-scratch model
   plan with no preservation constraint — which re-picked clips. This applied to **all** targeted
   revises, not just the 5s case.

## Constraint hierarchy (owner-approved)

| Tier | Constraint | Policy |
|---|---|---|
| 1 | Renderer feasibility (speed in [0.5, 2.0], source ranges, useful-segment coverage, transitions) | **Mandatory — never relaxed** |
| 2 | Explicit user requirements (custom exact duration, named Moment directives) | **Authoritative**, subject to feasibility |
| 3 | Targeted-revision preservation (clip identities, order, source ranges, soundtrack, native audio, transitions, lineage) | Preserved by default |
| 4 | Soft user preferences ("~5s", "faster", "more energetic") | Optimized as closely as feasible |
| 5 | AI recommendations (inferred duration, pacing, breadth) | **Must NOT override a user's revise intent** |

## Duration provenance (numeric source != editorial mode)

Per owner correction: Highlights / Full Story are explicit **editorial preferences**, but their
**numeric** duration is AI-recommended. Provenance tracks the number, not the mode.

- **New additive request field:** `revision_context.duration_target_source in {user_exact, ai_recommended}`.
  Request-only; **not** an `EditPlan/2` change. Default (absent/unknown) = `user_exact` (strict, safe).
- **iOS** (`StoryCreationCoordinator.composeRevisionContext`) derives it from the persisted
  `StoryModel.editorialMode`: `custom -> user_exact`, `highlights|full -> ai_recommended`. A value the
  base context already carries (a newer explicit instruction) is never clobbered.

## Operation-scoped validation (no global weakening)

- `_undershoot_tolerance_for(inputs)` returns the strict default `DURATION_UNDERSHOOT_TOLERANCE` for
  initial creation and for any revise with a `user_exact` target; it returns **None (no lower bound)**
  only for a **timing-only revise** whose target is `ai_recommended`. Overshoot, speed, source-range,
  and feasibility checks are untouched.
- `apply_deterministic_retime` no longer lifts shot durations toward the old target for an
  `ai_recommended` target (lets the video shorten to honor the pacing); it keeps the `min_total` lift
  **only** for `user_exact` (honor the explicit length while preserving clips).

## Preservation-safe fallback boundary

- A **targeted revise** = `mode==revise` + a `base_plan` + **not** `allow_broad_replan`. Such a revise
  never becomes an unrestricted from-scratch plan free to re-pick every clip. The only from-scratch
  replans allowed: initial creation, Explore (`mode==explore`), and a revise that **explicitly**
  authorizes broad redesign (`revision_context.allow_broad_replan == True`).
- **Preservation gate** (`_preservation_violation`): after the model is consulted and the plan is
  valid + feasible, the accepted plan's clip set (segment `asset_id`s, case-insensitive) is compared to
  the base. A timing/soft revise may not add or drop clips; a directive revise may only touch clips its
  directives name. A silent re-pick is rejected with a clear limitation — *"could not be applied without
  changing parts of your video you did not ask to change ... your current version was kept unchanged"* —
  and the user's existing version is preserved. Never fabricates success, never substitutes a different
  video.
- This increment enforces the **boundary + the check** (owner: do not build every repair algorithm
  now). Richer bounded preservation-repairs are a later increment.

## End-to-end provenance (iOS -> API -> SQS -> Lambda)

`revision_context` is a free-form dict on `EditPlanRequest`, passed unfiltered by the API and the worker
into engine inputs, so `duration_target_source` survives intact. The worker's `rev_diag` now logs
`dur_src` for live CloudWatch verification.

## Tests

- **Python (+9, suite 115 pass):** undershoot relaxed for `ai_recommended` timing revise (0 model
  calls, clips preserved, total shortened, no lift); `user_exact` keeps the strict lower bound (lift,
  clips preserved); initial-creation undershoot unchanged; preservation gate rejects a model clip
  re-pick (clear limitation) / allows a preserving revise; `allow_broad_replan` bypasses; provenance
  defaults to `user_exact`.
- **Swift (+5):** `editorialMode -> duration_target_source` wire mapping (custom/highlights/full/none)
  and no-clobber of a base-context value.

## Deploy

CodeBuild `story-rb-caseg-build` -> ECR -> App Runner `start-deployment` (API) + CDK worker-asset deploy;
health 200. Verify on device via `worker_plan_diag` `rev_diag.revision_path` + `rev_diag.dur_src`.

## Acceptance scenarios (device, owner STOP)

1. "Shorten each clip to ~5s" with inferred (Highlights/Full) duration -> `revision_path=deterministic_retime`,
   `model_calls=0`, `dur_src=ai_recommended`, same clips, total shortened (not lengthened).
2. Same with explicit custom duration -> clips preserved, total held near the exact length.
3. Remove one Moment, allow shorten. 4. Reorder Moments. 5. Replace only the opening.
6. Change soundtrack only. 7. Faster beat-aware cuts. 8. Unsupported color adjustment.
9. Unsupported visual effect. 10. Infeasible source range. 11. Mixed supported/unsupported.
12. Broad creative redesign (authorized). 13. Revision of an older selected version.
14. Recovery after app restart.
Per scenario verify: user-intent satisfaction, preservation of unrelated edits, duration-policy
behavior, renderer executability, no unintended replan, model-call count, latency, correct
`parentVersionId`, previous version unchanged.

## Remaining limitations / deferred

- **Phase 5 unsupported-operation handling** (color correction, photo adjustments, unsupported speed
  effects, reframing): explain the limitation, preserve the version, never fabricate/substitute — NOT
  yet built; separate increment per owner's "do not implement every repair this increment."
- Richer bounded preservation-safe repairs (localized feasibility correction beyond relax-and-
  revalidate) deferred.
- The transition-vocabulary drift (backend {cut,crossfade,fade_in,fade_out,dip_to_black} vs iOS
  {cut,fade,dissolve}) documented elsewhere remains.
