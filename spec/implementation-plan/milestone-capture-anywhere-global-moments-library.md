# Milestone — Capture Anywhere + Global Moments Library: Architecture Gate

Status: **AUDIT + ARCHITECTURE PROPOSAL — awaiting owner approval. STOP before changing persistent models. Nothing pushed/deployed.**
Owner: lingzhang · Date: 2026-10-13 · Method: code-grounded (file:line verified against `apps/ios/Story/Story/Data/Persistence/Models.swift`).
Supersedes the "Create Now (C)" ephemeral-Story design in `milestone-mvp-reconciliation-createnow-versions-nlrevision.md` with a simpler, standalone-capture + global-library model.

---

## 1. Existing code-grounded architecture

**Ownership is a strict Story-owned tree, all `.cascade`:**
```
StoryModel
  ├─(.cascade, inverse MomentModel.story)──> [MomentModel]      (Models.swift:121)
  │                                              └─(.cascade)──> AssetModel  (media file)  (Models.swift:209)
  └─(.cascade, inverse GeneratedOutputModel.story)──> [GeneratedOutputModel] (Models.swift:124)
```
- **`MomentModel.story: StoryModel?` is SINGLE-VALUED** (`Models.swift:203`) — a Moment belongs to exactly ONE Story. No sharing possible today.
- **Deleting a Story deletes its Moments AND their media files** (cascade → MomentModel → AssetModel). This is exactly the behavior the owner wants to eliminate ("do not silently delete footage").
- **`AssetModel`** owns the real media as a RELATIVE path under Documents (`relativePath`, e.g. `media/<uuid>.mov`), resolved only via `MediaStore`. Bytes never in SwiftData. (`Models.swift:171+`)
- **`MomentUnderstandingModel` is ALREADY GLOBAL** — keyed by `@Attribute(.unique) momentId`, upsert-per-Moment, stores the `moment_understanding/1` wire JSON (`Models.swift:400-410`). It is NOT Story-scoped. ← major reuse win.
- **`GeneratedOutputModel`** persists each render (relative path under `rendered/`, `editPlanJSON`, version lineage `versionId/parentVersionId/operationType`), additive/migration-safe (`Models.swift:250+`).
- **Shoot-a-Moment** (`SceneScanFlowView.persistMoment`) captures → writes AssetModel → attaches MomentModel to the current Story → enqueues AU. Entry is only from inside a Story (`StoryHomeView`).
- **Home** "Create now" + "Use existing photos" cards are `enabled:false` (`HomeView.swift:55,97`); only "Start a Story" is wired.
- Store schema list + migration-failure recreate fallback in `StoryApp.swift`.

**Migration risk headline:** the single-valued `MomentModel.story` + the two `.cascade` rules are the load-bearing change. Flipping to many-to-many and nullify-not-cascade is a RELATIONSHIP shape change — SwiftData lightweight migration handles added/optional/defaulted properties, but a to-one → to-many relationship inversion + delete-rule change is a heavier migration that needs explicit handling and device testing (the sandbox cannot run it; owner's device is the gate).

## 2. Proposed domain model

**Make the Moment independent; associate Stories many-to-many; nullify instead of cascade.**
```
MomentModel (independent identity + media owner)
  • id (unique), caption, sourceRaw, createdAt
  • asset: AssetModel?  (KEEP .cascade — the Moment owns its media; deleting the MOMENT deletes its file)
  • stories: [StoryModel]   ← NEW many-to-many (replaces single `story`)
StoryModel
  • moments: [MomentModel]  (deleteRule .nullify, inverse MomentModel.stories)  ← was .cascade
  • generatedOutputs: [GeneratedOutputModel]  (KEEP .cascade — outputs are Story-owned)
MomentUnderstandingModel  — UNCHANGED (already global, keyed by momentId)
```
- **Independent Moment identity + media ownership:** media lifetime follows the MOMENT, not the Story. Deleting a Story nullifies the association, never the media.
- **Reuse across Stories:** many-to-many `MomentModel.stories` ⇄ `StoryModel.moments`.
- **Global vs Story-specific AU:** the raw `MomentUnderstandingModel` stays GLOBAL (one per Moment, unchanged). Any Story-specific *interpretation* (how a Moment reads under this Story's intent) is DERIVED at plan time from the global AU + the Story's CreativeIntent — NOT a stored per-(Story,Moment) record. Keeps the model simple and avoids N×M storage.
- **Safe deletion:** delete Story → nullify links, keep Moments+media+AU. Delete Moment (explicit, from library) → cascade its AssetModel (its own media) + its MomentUnderstanding; and nullify it out of any Stories (warn if it's in use).
- **Backward-compatible migration:** every existing Story-scoped Moment becomes a global Moment associated with that one Story (1-element `stories`). Media + outputs preserved.

## 3. SwiftData migration strategy
- This is NOT a lightweight (additive/defaulted) migration — it changes a relationship arity (to-one→to-many) and two delete rules. Options, safest first:
  1. **Additive bridge (preferred, lowest risk):** ADD `MomentModel.stories: [StoryModel]` (new, defaulted empty) and ADD `StoryModel.moments2` many-to-many, KEEP the old `story`/`moments` for one release; a one-time boot migration copies each `moment.story` into `moment.stories`, then reads switch to the new relationship. Change the Story→Moment delete rule to `.nullify`. The old to-one stays as a nullable shadow until a later cleanup release. This keeps SwiftData in lightweight-migration territory for the ADD and isolates the delete-rule change.
  2. **Explicit VersionedSchema + MigrationPlan:** model V1→V2 with a custom stage that rebuilds the relationship. Cleaner end-state, but SwiftData custom migrations are fragile and CANNOT be dry-run in this sandbox — device-only verification, higher risk.
- Keep the existing `StoryApp.swift` migration-failure recreate-store fallback as the backstop, but a recreate would be data loss, so the bridge approach (which avoids a failing migration) is strongly preferred.
- **Owner's device is the migration gate** — no CI can prove it.

## 4. Media ownership / deletion policy
- **Media is owned by the Moment** (`MomentModel.asset`, `.cascade`), stored as a relative path under `Documents/media/`. Resolve only via `MediaStore`.
- **Deleting a Story:** nullify the Story↔Moment links, delete the Story's `generatedOutputs` (Story-owned rendered files), **never touch Moment media or AU.** ← satisfies "do not silently delete footage."
- **Deleting a Moment (explicit library action only):** delete its AssetModel file + its MomentUnderstanding; if it is associated with any Story, warn/confirm first (it will vanish from those Stories' material sets, though already-rendered outputs are unaffected since they're separate files).
- **Orphan safety:** a Moment with empty `stories` is valid (library-only, not yet used). No auto-delete of unused Moments.

## 5. UX changes
- **Home:** replace the two disabled cards. Add **Capture** (opens camera immediately, standalone) and **Moments** (library). Remove "Use existing photos" as a Home milestone; existing-media import stays a shared action inside Capture/Library and Story building.
- **Capture Anywhere:** reuse the EXISTING Shoot-a-Moment flow (`SceneScanFlowView`), NOT a new capture engine. Default: camera opens immediately; optional AI shot idea; save Moment to the GLOBAL library; AU runs async. If launched from inside a Story, also associate with that Story.
- **Moments Library:** chronological list, thumbnail preview, multi-select, safe delete, "Create Story from selected." No albums/search/indexing for MVP.
- **Create from Moments:** selected Moments → the EXISTING creation pipeline (CreativeIntent, Duration Intelligence, Storytelling/Visual-first, MusicPlan, async Edit Planning, Media Engine, version persistence). No media duplication, no second Edit Planner path.

## 6. Implementation complexity
- **High-risk / must-get-right:** the relationship + delete-rule migration (§3) — the only genuinely hard part; device-gated.
- **Medium:** decoupling `persistMoment` from a required Story (make the Story association optional); the Moments Library screen (new UI, but standard list/multi-select); Create-from-Moments wiring (builds a Story from a selected Moment set then runs the existing coordinator).
- **Low / pure reuse:** Capture flow (existing), AU (already global), creation pipeline (unchanged), rendering (unchanged).

## 7. MVP scope & risks
- **In scope:** independent Moments, many-to-many association, Capture from Home, Moments Library (browse/preview/multi-select/delete/create), Create-from-Moments via the existing pipeline, nullify-not-cascade deletion, backward-compatible migration.
- **Out of scope (MVP):** albums, advanced search, large-library indexing, per-(Story,Moment) stored AU interpretation.
- **Risks:** (1) the relationship migration is the single biggest risk — a bad migration = data loss; mitigate with the additive-bridge approach + device gate + keep the recreate-store backstop. (2) Changing the Story delete rule must be verified to NOT delete shared media (acceptance test 4). (3) Thumbnail/preview performance on a growing library (bounded by MVP's no-indexing scope). (4) An in-use Moment deleted from the library needs a clear warning path.

## 8. Acceptance tests (owner device-gated)
1. Capture from Home with NO existing Story → Moment lands in the global library, AU runs.
2. Capture from inside a Story → Moment in library AND associated with that Story.
3. Add the SAME Moment to two Stories → both reference it; one media file, no duplication.
4. Delete one Story → its outputs gone, but the shared Moment + its media + AU survive and the other Story is intact.
5. Select library Moments → create a new Story through the existing pipeline → renders.
6. Close/reopen app → all Moments + associations persist.
7. AU + rendering complete with NO regression vs today.
8. Migration: existing Story+GeneratedOutput+Moment records survive the upgrade; each old Story-scoped Moment becomes a global Moment linked to that Story; media + outputs preserved.
