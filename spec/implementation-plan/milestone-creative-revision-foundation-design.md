# Milestone — Creative Revision Foundation: Create Another Version + Natural-Language Editing (DESIGN)

Status: **ARCHITECTURE / FREEZE GATE — awaiting owner approval. No production implementation; nothing pushed/deployed.**
Owner: lingzhang · Date: 2026-10-14 · Method: code-grounded re-audit (direct read, file:line verified this session).
Scope: two milestones designed around ONE shared foundation, implemented + validated SEPARATELY. Supersedes the
Versions + NL-revision sections of `milestone-mvp-reconciliation-createnow-versions-nlrevision.md` (Create Now is
NOT in this milestone). No frozen contract (EditPlan/2) or prompt (edit_plan/5.2) is reinterpreted.

---

## 1. Current code-grounded implementation status

Verified in code this session (not inferred from old specs):

**Create Another Version**
- The button EXISTS and is always available — `StoryResultView.recreate()` → `StoryCreationCoordinator.create()`
  (`apps/ios/Story/Story/Features/Create/StoryResultView.swift`). It is ADDITIVE: `persist()` only ever appends a
  new `GeneratedOutputModel` after a successful render; prior versions are never overwritten.
- Version lineage is ALREADY MODELED — `GeneratedOutputModel` carries `versionId`, `parentVersionId`,
  `operationType` ("original" | "change_music"; "another_version" reserved), `sourceMomentIds`,
  `sourceFingerprint`, and `editPlanJSON` (the EXACT executable plan the renderer consumed), plus
  `soundPreferenceRaw` / `musicAssetRef` (`apps/ios/Story/Story/Data/Persistence/Models.swift`).
- So the substrate (lineage + the parent's exact plan on-device) already exists; what is missing is only the
  version-aware planning INPUT and the lineage WIRING on the re-create path.

**Natural-Language Revision** — does not exist anywhere in shipping code. No endpoint, DTO field, UI control, or
planner input takes a free-text editing directive. The `CreativeDirectionView` free-text box feeds
CreativeIntent at Story START, never `/v1/edit-plan*`. `creative_brief` is plumbed end-to-end but its value is a
hardcoded constant (`StoryMultimodalPlanning.creativeBrief`, `shortform_mvp/1`), not user text. The only
breadth/pace lever today is the structured `editorial_mode` enum (highlights|full|custom).

**Shared foundations that already exist and will be reused as-is**
- Async Edit Planning: `/v1/edit-plan-jobs` (submit 202 → poll), `EditPlanJobClient`, idempotent on
  `operation_id`; the worker runs the SAME planning core as the sync endpoints.
- The deterministic executability gate `validate_plan` (`packages/edit_planner/__init__.py`) — the hard,
  authoritative validator; the model never self-certifies.
- The `creative_brief` USER-message injection seam (`_build_user`) and the `critique → revise → validate`
  backbone (`_build_critique` / `_build_revision`), already in the engine but today only reachable INSIDE one
  `generate()` call and gated to multimodal.
- `StoryCreationCoordinator` 8-state machine; `EditPlanRequestBuilder`; `DurationFeasibility` preflight;
  Media Engine; MusicPlan; AU evidence + `MultimodalVisualBudget` contact sheets; GeneratedOutput persistence.
- `EditPlanRequest` is `model_config = ConfigDict(extra="forbid")` — ANY new wire field is a versioned contract
  change, additive and defaulted so an older client is unchanged.

## 2. Root cause of low version diversity

The plan is a pure, deterministic function of unchanged inputs. Three contributing causes, in priority order:

1. **No version-aware input.** `EditPlanInputs` has no field describing what the prior version did. Identical
   inputs → identical reasoning.
2. **`temperature=0.0`** on every planner call (`RealEditPlannerEngine._call`, both the initial and repair
   turns). Even with other inputs equal, the model is driven to the same argmax plan.
3. **Idempotency short-circuit** on the async path: `/v1/edit-plan-jobs` computes a `payload_fingerprint(body,
   order, critique_revision, approach, …)`; the same `operation_id` with the SAME body reuses the SAME job →
   byte-identical cached plan. The happy path mints a fresh `operation_id` per version (cleared on completion),
   so (1)+(2) dominate; op_id reuse only bites on the poll-timeout recovery path. A version-aware body changes
   the fingerprint, so (3) stops masking a genuine re-plan automatically.

Storytelling mode is worst because it is text-only (no contact-sheet variation) and leans hardest on the
deterministic argmax.

## 3. Shared architecture recommendation

**One foundation, two operations, no new engine.** Add ONE optional structured input to the planning request —
a `revision_context` object — that both milestones populate differently, and inject it into the **USER message
only** (the proven `creative_brief` seam). The frozen `edit_plan/5.2` SYSTEM body is untouched; `validate_plan`
stays the authoritative gate and gains new deterministic predicates for HARD constraints. The
`critique → revise → validate` backbone is lifted out so a "revise THIS committed plan with THIS instruction"
entry point is reachable (today it only runs inside `generate()`).

Minimum additive request fields (all optional, default none ⇒ today's behavior byte-for-byte):
```
revision_context (optional object, USER-message only):
  mode:            "explore" | "revise"            # drives preservation semantics (§ Phase 3)
  prior_plans:     [compact digest]                # for diversity / preservation (client-derived)
  directives:      { include_moment_ids: [...],    # HARD — validated deterministically
                     exclude_moment_ids: [...],
                     opening_moment_id: id|null,
                     ending_moment_id:  id|null,
                     order_pins: [ [id,id], ... ],  # relative order constraints
                     preserve_continuous: [moment_id,...] }
  soft_direction:  str|null                        # SOFT — "more cinematic", "faster", opening/ending vibe
```
Nothing here changes the EditPlan/2 OUTPUT contract. `revision_context` is request-only, bumping the
request/DTO minor; the output stays frozen EditPlan/2.

Why not a separate revision engine: the engine already plans, critiques, revises and validates against the same
schema. A second engine would duplicate the validator and the Media-Engine contract. The required behavior is a
composition over existing components — the gate's bar ("do not introduce a separate revision engine unless
existing components cannot support it") is NOT met.

## 4. Version diversity design (Create Another Version)

- **Evidence, not randomness.** Derive a compact `prior_plans` digest CLIENT-side from the parent chain's
  `editPlanJSON` (already persisted): per prior version — selected `moment_ids`, source ranges, ordering,
  opening + ending moment, shot-duration distribution (count + min/median/max), and the prior creative emphasis
  (sound preference, editorial_mode). Inject it in the USER message with a bounded objective: *"produce a
  meaningfully DIFFERENT yet valid edit from the prior version(s) below — different opening, different ordering
  or source emphasis — WITHOUT dropping required material or padding."* No shuffle, no templates.
- **Determinism lever.** Keep `temperature=0.0` as the baseline (prior-plan context is the primary lever); allow
  an OPTIONAL small temperature bump (e.g. 0.4) for the `mode:"explore"` op ONLY, flag-gated, measured in eval
  before it is a default.
- **Preserve (never varied away):** user duration (CreativeIntent target), sound preferences, any HARD
  directives, continuous-performance spans, source feasibility (validator unchanged), and ALL prior
  GeneratedOutputs (append-only).
- **Lineage wiring:** the re-create path sets `operationType="another_version"`,
  `parentVersionId = source.versionId`, and passes the derived `prior_plans`. Mints a fresh `operation_id` (so
  the idempotency fingerprint differs and a real re-plan happens).
- **Deterministic diversity diagnostics** (computed in eval + optionally logged): Jaccard distance of the
  selected moment-id set vs parent; opening-moment-changed (bool); ordering edit distance (Kendall tau on the
  shared moments); source-range IoU per shared moment; shot-count delta; shot-duration-distribution delta. A
  version that scores ~0 on all of these is a "near-identical" failure.
- **Human quality eval:** side-by-side parent vs new version on the four fixtures — is the new one genuinely a
  different, still-good edit (not merely shuffled, not degraded)?

## 5. Natural-language revision design (V1)

Reuse the `critique → revise → validate` backbone, seeded with the EXISTING committed plan (`base_plan` from the
source version's `editPlanJSON`) plus the parsed directive. Two directive classes, separated explicitly:

- **HARD constraints** → a STRUCTURED `directives` object parsed from the user's NL, enforced by NEW
  deterministic `validate_plan` predicates: required moment present; forbidden moment absent; opening/ending
  `asset_id` pinned; relative order respected; a `preserve_continuous` span kept as one unbroken segment. A
  revision that violates a hard predicate is REJECTED → one bounded repair → honest failure; it is NEVER
  silently ignored. Covers: must-include a Moment, exclude a Moment, change opening, change ending, reorder
  specified Moments, preserve a continuous performance.
- **SOFT preferences** → carried as text in the revise call (same seam as `soft_direction`): "more cinematic",
  "faster/slower pacing", "increase/decrease coverage" (nudges `editorial_mode` + text), opening/ending *vibe*.
  Gated only by `validate_plan` (no hard predicate). Nearly free — mostly wiring.
- **Source-reference resolution.** Map a user phrase ("the summit shot", "where she spins") to a Moment id using
  EXISTING AU evidence (moment summaries, `useful_segments`, events) and, when visual-first is on, the
  multimodal contact sheets. Deterministic candidate ranking; the model proposes, the resolver confirms against
  real Moment ids (never invents footage).
- **Ambiguous references → thumbnail clarification UX.** When resolution is low-confidence or multiple Moments
  match, present a small thumbnail picker ("which one did you mean?") rather than guessing. Uses the existing
  `MomentThumbnailCache`; no new media pipeline.
- **Infeasible / conflicting directives are surfaced**, never dropped: e.g. "exclude the only Moment that can
  open" or "include X and exclude X" → an honest, specific message; the user adjusts.

## 6. Preservation semantics (Explore vs Revise)

Two distinct operations, both append-only (each creates a NEW GeneratedOutput; previous versions preserved):

| | **Explore** (`mode:"explore"`) | **Revise** (`mode:"revise"`) |
|---|---|---|
| Intent | Seek meaningful editorial diversity | Apply requested changes |
| Prior plan | Used as an "avoid being identical" reference | Used as the BASE to preserve |
| Latitude | Substantial changes allowed | Change the smallest set needed; preserve unaffected edits where feasible |
| Temperature | baseline 0.0, optional bounded bump | 0.0 (determinism is desirable for a targeted edit) |
| Directives | none required | hard + soft directives applied + validated |

**User-owned invariants that survive BOTH operations** (never silently replaced): CreativeIntent target
duration; sound preference; editing approach; any HARD directive the user has stated; the user-authored Story
title/description. AI-generated recommendations (StoryGuide, shot suggestions, the plan itself) MAY change as
evidence changes — the distinction mirrors the Capture-Anywhere Phase-4 user-intent rule already in the codebase.

## 7. Contract and prompt implications

- Additive request-only field `revision_context` on `EditPlanRequest` + `EditPlanInputs` + the
  `/v1/edit-plan-jobs` form. Default none ⇒ behavior unchanged ⇒ older clients unaffected. Bump the REQUEST/DTO
  minor; the EditPlan/2 OUTPUT contract and the `edit_plan/5.2` SYSTEM body stay FROZEN (all new text enters via
  the USER message, exactly like `creative_brief`).
- New deterministic predicates added to `validate_plan` for the HARD directive class (include/exclude/opening/
  ending/order/continuous). These are ADDITIVE checks that only run when a `directives` object is present.
- `GeneratedOutputModel.operationType` vocabulary extended with `another_version` and `edit_ai` (both already
  reserved); lineage model already migration-safe (additive).
- A versioned prompt proposal is required ONLY if evaluation shows the USER-message injection is insufficient
  and the SYSTEM body must change — explicitly deferred to a separate freeze gate, not assumed here.

## 8. Evaluation and acceptance criteria

Fixtures (reuse cached AU evidence; bounded model spend; NO new multimodal research program): **Girl with
Dress, Hiking, Guitar, Minimal footage.**

- **Create Another Version:** regenerating a version yields measurably different output vs parent on the §4
  diagnostics (not identical) AND passes human side-by-side quality (different, still good, not degraded, not
  shuffled). Prior version preserved; user duration + sound preference + hard directives honored. Minimal-footage
  fixture must NOT be forced to fabricate diversity it cannot support (honest "material too thin to vary").
- **NL Revision V1:** per directive — directive satisfaction (the intended change happened); source grounding
  (resolved to the right real Moment); preservation of unaffected edits (Revise mode); executability
  (`validate_plan` passes); audio continuity (native-audio + continuous performance preserved). HARD directives
  deterministically validated; infeasible/conflicting directives surfaced, never silently dropped.
- Spend: cap model calls, reuse cached evidence, no broad sweep.

## 9. Implementation sequence (after the gate)

1. **Shared foundation:** `revision_context` request field (additive) + lift `_build_revision` into a reachable
   "revise base_plan with context" entry point + new `validate_plan` hard predicates (behind presence of
   `directives`). No behavior change when the field is absent.
2. **Create Another Version (A):** client-derived `prior_plans` digest + `operationType`/`parentVersionId`
   wiring + fresh op_id; `mode:"explore"`. Diversity diagnostics in eval. Device-gate on girl + hiking.
3. **NL Revision V1 (B):** soft preferences first (wiring), then hard directives (structured parse + predicates),
   then source-reference resolution + thumbnail clarification UX. Device-gate per directive.

Each ships and is validated separately; (2) and (3) both depend only on (1).

## 10. MVP vs post-MVP boundaries

- **MVP:** shared `revision_context` foundation; Create Another Version with prior-plan diversity;
  NL Revision soft preferences + hard include/exclude/opening/ending/order.
- **Post-MVP:** `preserve_continuous` tied to AU action spans (depends on `CreativeRefinementV2b`
  device-acceptance); temperature-tuned variation as a default; multi-segment-per-Moment diversity;
  conversational multi-turn revision.
- **OUT OF SCOPE (owner):** new editing models, manual timeline editing, generative effects, large-library
  indexing, Create Now (separate milestone).

## Updated spec references

- Supersedes the Versions (§A) + NL-revision (§B) design of
  `milestone-mvp-reconciliation-createnow-versions-nlrevision.md`; Create Now (§C) remains in that doc, out of
  this milestone.
- Relates: `milestone-me-music-library-change-music-versioning.md` (lineage substrate),
  `milestone-async-edit-planning-and-prod-rollout.md` (async foundation), `12-edit-flow.md` /
  `milestone-v2a/v2b-creative-editing-*.md` (deterministic cut refinement, distinct from NL directives).
- **STOP at owner architecture/freeze gate.** No implementation, no contract/prompt change, nothing pushed.
