# Milestone R7 — Cloud-Aware Planner Eligibility & Explicit Moment Addition

Status: Phases 0–2 implemented (iOS); STOP at device acceptance. Phase 3 (explicit add-new-Moment during Revise) separately gated.
Date: 2026-10-20
Commit: `7757fe4` (iOS). No backend/worker change — the frozen Edit Planner/validator are byte-identical.

## Problem (P0)

The TMB Story has 14 Moments, but only 4 reached the Edit Planner. A directive naming a real
Moment ("end with Moment 2", id `9D36385A`) failed `directive_ref_ineligible` → `invalid_after_repair`.

Root cause (evidence): `RenderSupport.build()` built the planner `moments[]` from
`MomentResolver.resolve()`, which **hard-skips any Moment whose original file is not on disk**
(`fileExists`). On this device only 4 of the 14 TMB Moments had local bytes; the other 10 were
cloud-only (evicted / R7-restored shells) — including Moment 2, whose Asset Understanding **exists
server-side**. So a satisfiable, correctly-authored directive was blocked by a device-local byte gate
the planner never needed. `StoryCreationReadiness.evaluate` was already cloud-aware (`onDisk ||
restorable`); the request builder was not — the two disagreed.

Attribution: **B — pre-existing planner-input coupling exposed by R7 directive sync.** The Edit
Planner itself is unchanged; R7-C's cross-device directive sync made the cloud-only directive always
present, surfacing the latent local-byte eligibility coupling.

## Phase 0 — Contract audit

The planner `moments[]` payload is `{id, duration_seconds, useful_segments, summary, evidence}` and
needs **zero original bytes**:
- `duration_seconds` — `AssetModel.durationSeconds` (cloud metadata, carried in the synced Story doc).
- `useful_segments` / `summary` / `evidence` — the MomentUnderstanding JSON, hydrated from the server
  by `UnderstandingSync.restore` (metadata batch-get + content-sha reuse gate — **no bytes, no Bedrock**).

Original bytes are required only at **render** time, where R6 (`resolveEnsuringLocal`) already restores
them. "Removing fileExists is not automatically safe" (owner): confirmed — the render-time file gate is
kept; only the **planner-eligibility** gate is decoupled from local bytes.

## Phase 1 — Directive safeguard

Before Create/Revise submit, `restoreReferencedCloudOnlyMoments` now **hydrates server-side AU** for
every directive-referenced Moment missing a local AU (`UnderstandingSync.restore`, bounded to referenced
ids, no Bedrock), then best-effort pre-restores bytes. `directiveEligibility` is cloud-aware: a Moment is
eligible once it has AU + duration + a renderable source. A reference that genuinely cannot be satisfied
(no AU and not restorable, or bytes gone) is preserved and surfaced as an honest pending/recovery state —
never silently stripped, and a directive whose Moment is absent from `moments[]` is never submitted.

## Phase 2 — Cloud-aware eligibility (shared boundary)

New shared policy in `MomentResolver`:
- `plannerDuration(for:)` — local probe if on disk, else `AssetModel.durationSeconds`; 0 = unknown.
- `isRenderableSource(_:)` — bytes local **OR** cloud-verified + `cloudMediaId` (restorable via R6).

A Moment is **planner-eligible** iff: completed AU **AND** `plannerDuration > 0` **AND**
`isRenderableSource`. Applied in all four coupled builders — `build`, `usableNumberedMoments`
(Customize "Moment N" numbering), `momentsPayload` (Duration Planner), `directiveEligibility` — so they
agree with each other and with the planner. Local-only Moments behave exactly as before. A fully
cloud-only Story is plannable (the `noUsableMoments` guard now checks the planner set, not the local set).

Pre-render restore: after the plan completes, the Moments the plan **actually selected** have their
originals restored via R6 before the render resolver/validate — in both the fresh (`run`) and resume
(`renderResumedPlan`) paths. A Moment that cannot be restored is reported by `validate()` as an
unresolved asset (honest failure), never a silent skip.

## Phase 3 — Explicit Moment additions during Revise (separately gated)

This fix already lets a **revise reference any Story Moment with synced AU** (not just those in the base
plan), because the eligible set is the whole cloud-aware Story set. "Add Moment N / replace / remove" via
directives therefore resolves against the full set. A deterministic *add-new-segment* revision transform
(beyond reorder/prune) would touch `apply_deterministic_revision` semantics — if that is required it is a
versioned proposal (STOP gate), not implemented here.

## Boundary assessment

Strictly additive, iOS-only. No change to EditPlan/2 schema, frozen prompts, or `validate_plan` semantics.
Backend `apps/api/tests/test_edit_planner.py`: 125 passed (contract intact).

## Tests

`CloudAwareEligibilityTests` (in-memory SwiftData, no Bedrock/network): cloud-only Moment with AU is
eligible; a directive referencing a cloud-only Moment is eligible and in the payload; Device-A/Device-B
equivalent eligible sets; missing AU → honest exclusion (not invented); bytes-gone Moment with AU → not
eligible (cannot render); fully-cloud-only Story is plannable; Customize numbering includes cloud-only
Moments with `localAvailable=false`. Device Xcode build is the authoritative compile gate (sandbox blocks
the SwiftData macro plugin), so these run on device.

## Deployment & acceptance

**iOS-only — no backend/worker redeploy.** Rebuild the app on device, then using the TMB Story:
- Confirm planner-eligible set includes all 14 Moments when their AU/duration are valid.
- Confirm "end with Moment 2" no longer fails due to device-local availability.
- Confirm the rendered video uses the requested Moment (R6 restores its bytes before render).
- Confirm normal-speed/native-audio behavior is preserved.

## Remaining limitations

- A referenced Moment with no synced AU still blocks honestly until AU hydrates (expected).
- Deterministic *add-new-segment* revision (vs reorder/prune) is Phase 3, gated.
